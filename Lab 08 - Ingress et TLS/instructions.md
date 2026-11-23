# LAB 08 - Ingress et TLS

> **Cluster : GKE** (contexte `cluster-lab-dkdc`) | **Durée : ~50 min**
> **Prérequis :** Lab 07 terminé (Service `webapp` dans `countvisit`, dossier `~/labs/countvisit`).

Au Lab 04, l'application était exposée par son propre Service `LoadBalancer` : une IP publique (et un
équilibreur facturé) par application. Avec dix applications, dix IP. Un **Ingress** décrit des
règles de routage HTTP (nom d'hôte, chemin → Service) ; il ne fait **rien** tout seul. C'est
l'**Ingress Controller** (ici NGINX) qui lit ces règles et configure un reverse-proxy, lui-même
exposé par **un seul** Service `LoadBalancer`. On passe de *un LoadBalancer par application* à *un
point d'entrée pour toutes*.

☁️ **Sur GKE** - GKE fournit **son propre** contrôleur Ingress, qui crée un équilibreur HTTP Google
Cloud pour chaque Ingress. Nous ne l'utiliserons pas : nous installons **Ingress NGINX**, un
contrôleur qui tourne dans le cluster et fonctionne de la même façon sur tous les clusters. Le champ
`ingressClassName: nginx` de chaque Ingress désigne explicitement ce contrôleur (Partie 5).

> **Pour information :** le projet Ingress NGINX est en fin de vie (maintenance arrêtée en mars 2026).
> Son successeur désigné par la communauté est la **Gateway API**, présentée en fin de lab. Les
> notions vues ici (hôte, chemin, TLS, backend) s'y retrouvent à l'identique.

## Objectifs

1. Installer un contrôleur Ingress et identifier son point d'entrée.
2. Publier l'application par **nom de domaine** (routage de niveau 7).
3. Router par **chemin**, avec réécriture d'URL.
4. Activer **HTTPS** avec un certificat stocké dans un Secret TLS.
5. Diagnostiquer les pannes typiques : Ingress pris en charge par le mauvais contrôleur, 404, 503.

**Documentation utile :** [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) · [Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) · [Ingress NGINX - rewrite](https://kubernetes.github.io/ingress-nginx/examples/rewrite/) · [Ingress NGINX - TLS](https://kubernetes.github.io/ingress-nginx/user-guide/tls/) · [Gateway API](https://kubernetes.io/docs/concepts/services-networking/gateway/)

---

## Partie 1 : Le point d'entrée

Le contrôleur Ingress NGINX n'est pas installé sur le cluster. Installez-le avec Helm (l'outil sera
étudié en détail au Lab 09) :

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.type=LoadBalancer
```

**1.1 - Le contrôleur et sa classe**

Vérifiez que les Pods du contrôleur sont `Running` dans le namespace `ingress-nginx`. Retrouvez le
nom de l'`IngressClass`, puis l'adresse du contrôleur (le Service de type `LoadBalancer` de ce
namespace). Rangez cette IP dans la variable `INGRESS_IP`.

Le domaine du lab n'existe dans aucun DNS : nous utiliserons le service public **nip.io**, qui
résout tout nom de la forme `<texte>.<IP>.nip.io` vers `<IP>`. Définissez les variables
`HOST_APP=countvisit.<INGRESS_IP>.nip.io` et `HOST_APPS=apps.<INGRESS_IP>.nip.io`, et ajoutez-les à
`~/formation.env`.

<details><summary>Correction</summary>

```bash
kubectl get pods -n ingress-nginx
kubectl get ingressclass                 # NAME : nginx
kubectl get svc -n ingress-nginx         # ingress-nginx-controller   LoadBalancer   ...   EXTERNAL-IP

INGRESS_IP=$(kubectl get svc -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $INGRESS_IP

cat >> ~/formation.env <<EOF
export INGRESS_IP=$INGRESS_IP
export HOST_APP=countvisit.$INGRESS_IP.nip.io
export HOST_APPS=apps.$INGRESS_IP.nip.io
EOF
source ~/formation.env
nslookup $HOST_APP                       # Address : <INGRESS_IP>
```

Un cluster peut héberger plusieurs contrôleurs : l'`IngressClass` permet à chaque règle de
désigner le sien. Le contrôleur intégré de GKE, lui, tourne dans le plan de contrôle géré par
Google : il n'apparaît ni parmi les Pods, ni forcément dans `kubectl get ingressclass`.

🏠 **En local** - le Service `LoadBalancer` du contrôleur n'obtient une IP que si un composant comme
**MetalLB** est installé ; sinon on passe par le NodePort du contrôleur. On utilise souvent un nom
en `.lab.test` déclaré dans le fichier `hosts` du poste, ou `curl --resolve`.

</details>

**1.2 - Un contrôleur sans règle**

Interrogez `http://<INGRESS_IP>/` avec `curl -i`. Que répond-il, et pourquoi ?

<details><summary>Correction</summary>

```bash
curl -i http://$INGRESS_IP/
# HTTP/1.1 404 Not Found ... nginx
```

Le 404 vient du contrôleur lui-même (son *default backend*) : il fonctionne, mais aucune règle ne
correspond encore à la requête.

</details>

**1.3 - Retirer l'exposition directe**

Votre Service `webapp` n'a plus besoin d'être exposé directement : repassez-le en `ClusterIP`
(modifiez `webapp-service.yaml` et appliquez). Pourquoi est-ce préférable ?

<details><summary>Correction</summary>

```bash
# webapp-service.yaml : type: ClusterIP
kubectl apply -f webapp-service.yaml
kubectl get svc webapp            # plus d'EXTERNAL-IP
```

Moins de surface exposée, et un équilibreur Google Cloud de moins (il est supprimé automatiquement) :
seul l'Ingress Controller reste joignable de l'extérieur.

</details>

---

## Partie 2 : Routage par nom de domaine

**2.1 - Écrire l'Ingress**

Écrivez `webapp-ingress.yaml` : un Ingress `webapp` qui envoie l'hôte `$HOST_APP` (remplacez par sa
valeur), chemin `/` (type `Prefix`), vers le Service `webapp`, port 80. N'oubliez pas
`ingressClassName`.

*(Astuce : `kubectl create ingress webapp --class=nginx --rule="$HOST_APP/*=webapp:80" $do`)*

<details><summary>Correction</summary>

```yaml
# webapp-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp
spec:
  ingressClassName: ngimx            # nom de l'ingressclass
  rules:
  - host: countvisit.<INGRESS_IP>.nip.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp
            port:
              number: 80             # le port du SERVICE, pas celui du conteneur
```

`pathType` est obligatoire :

| `pathType` | Comportement |
| --- | --- |
| `Exact` | correspondance stricte (`/foo` ≠ `/foo/`) |
| `Prefix` | correspondance par **segments** : `/foo` couvre `/foo/bar`, pas `/foobar` |
| `ImplementationSpecific` | laissé au contrôleur (expressions régulières NGINX par exemple) |

</details>

**2.2 - Appliquer et vérifier**

Appliquez l'Ingress et vérifiez qu'il a reçu une adresse (colonne `ADDRESS`). Sinon, cherchez
l'erreur.

<details><summary>Correction</summary>

```bash
kubectl apply -f webapp-ingress.yaml
kubectl get ingress webapp            # ADDRESS = <INGRESS_IP> (peut prendre quelques secondes)
kubectl describe ingress webapp       # règles et backends résolus
```

</details>

<details><summary>Correction - Diagnostic</summary>

Si vous avez trouvé, bravo ! Sinon, posez la question, ou lisez la Partie 5 sur le diagnostic des
pannes d'un Ingress.

</details>

**2.3 - Tester**

Ouvrez `http://$HOST_APP` dans votre navigateur, ou interrogez-le avec `curl`. Que répond
`curl http://<INGRESS_IP>/`, sans nom d'hôte ?

<details><summary>Correction</summary>

```bash
curl -s http://$HOST_APP | head
curl -i http://$INGRESS_IP/            # toujours 404
```

Sans en-tête `Host` correspondant, le contrôleur ne trouve aucune règle : le **nom d'hôte** fait
partie de la règle. En production, on crée un enregistrement DNS de type A vers l'IP du contrôleur.

</details>

---

## Partie 3 : Routage par chemin et réécriture

Plusieurs applications peuvent partager un même nom d'hôte, distinguées par le chemin.

**3.1 - Un second site**

Déployez un Deployment `static-site` (image `public.ecr.aws/wizetraining/nginx:latest`, 1 réplica,
port 80) et son Service `ClusterIP` sur le port 80, avec les commandes impératives
`create deployment` et `expose`.

<details><summary>Correction</summary>

```bash
kubectl create deployment static-site --image=public.ecr.aws/wizetraining/nginx:latest --port=80
kubectl expose deployment static-site --port=80
```

</details>

**3.2 - L'Ingress par chemin**

Créez `static-site-ingress.yaml` : un Ingress `static-site` pour l'hôte `$HOST_APPS`, dont le
chemin `/static` (type `Prefix`) mène au Service `static-site`, port 80. Testez
`http://$HOST_APPS/static`.

<details><summary>Correction</summary>

```yaml
# static-site-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: static-site
spec:
  ingressClassName: nginx
  rules:
  - host: apps.<INGRESS_IP>.nip.io
    http:
      paths:
      - path: /static
        pathType: Prefix
        backend:
          service:
            name: static-site
            port:
              number: 80
```

```bash
kubectl apply -f static-site-ingress.yaml
curl -i http://$HOST_APPS/static
# HTTP/1.1 404 Not Found
```

</details>

**3.3 - 🩺 Un 404 qui ne vient pas du contrôleur**

La requête renvoie un 404. Est-ce le contrôleur ou l'application qui répond ? Regardez les logs du
Pod `static-site` : quel chemin reçoit-il ? Corrigez avec l'annotation
`nginx.ingress.kubernetes.io/rewrite-target: /`.

<details><summary>Correction</summary>

```bash
kubectl logs deploy/static-site | tail -3
# "GET /static HTTP/1.1" 404 ...
```

Cette fois, le 404 vient de **l'application** : le contrôleur a bien trouvé la règle et transmis la
requête, mais nginx reçoit `/static`, un chemin qu'il ne connaît pas. L'annotation remplace le
chemin par `/` avant de transmettre :

```yaml
metadata:
  name: static-site
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
```

```bash
kubectl apply -f static-site-ingress.yaml
curl -s http://$HOST_APPS/static | head -4
```

Pour conserver la suite du chemin (`/static/css/a.css` → `/css/a.css`), on utilise des groupes de
capture (`path: /static(/|$)(.*)` et `rewrite-target: /$2`), cf. documentation.

⚠️ Les annotations `nginx.ingress.kubernetes.io/*` sont **propres au contrôleur NGINX** : elles
seraient ignorées par un autre contrôleur (celui de GKE, par exemple). C'est l'une des limites que la
Gateway API corrige.

💡 **Bon réflexe** - deux 404 n'ont pas la même cause : un 404 **sans** trace dans les logs de
l'application vient du contrôleur (aucune règle ne correspond : hôte ou chemin) ; un 404 **présent**
dans ses logs vient de l'application (chemin mal réécrit).

</details>

---

## Partie 4 : HTTPS avec un certificat auto-signé

**4.1 - Le certificat**

Générez avec `openssl req -x509` un certificat auto-signé pour `$HOST_APP` : clé `tls.key`,
certificat `tls.crt`, valable 365 jours, avec le nom d'hôte en *Subject Alternative Name*.

<details><summary>Correction</summary>

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=$HOST_APP" \
  -addext "subjectAltName=DNS:$HOST_APP"
```

</details>

**4.2 - Le Secret TLS**

Créez un Secret de type `tls` nommé `countvisit-tls` à partir de ces deux fichiers. Quelles clés
contient-il ?

<details><summary>Correction</summary>

```bash
kubectl create secret tls countvisit-tls --cert=tls.crt --key=tls.key
kubectl describe secret countvisit-tls      # Type: kubernetes.io/tls ; clés tls.crt et tls.key
```

🔎 **Attention** - un Secret TLS a **obligatoirement** les clés `tls.crt` et `tls.key`, et il doit
être dans le **même namespace** que l'Ingress.

</details>

**4.3 - Activer TLS**

Modifiez l'Ingress `webapp` pour activer TLS sur cet hôte avec ce Secret. Testez en HTTPS
(`curl -k`, ou le navigateur en acceptant l'alerte). Que répond maintenant l'accès en HTTP ?
Observez le certificat présenté (`curl -vk`).

<details><summary>Correction</summary>

```yaml
# webapp-ingress.yaml : ajouter sous spec
  tls:
  - hosts:
    - countvisit.<INGRESS_IP>.nip.io
    secretName: countvisit-tls
```

```bash
kubectl apply -f webapp-ingress.yaml
curl -sk https://$HOST_APP/ | head -5
curl -i http://$HOST_APP/ | head -3
# HTTP/1.1 308 Permanent Redirect → redirection vers https
curl -vk https://$HOST_APP/ 2>&1 | grep -E "subject|issuer"
```

Le navigateur affiche une alerte : le certificat est auto-signé (émetteur = sujet). Le TLS est
**terminé** au contrôleur : le trafic repart en clair vers les Pods, à l'intérieur du cluster. En
production, on automatise l'émission de certificats de confiance avec **cert-manager**
(Let's Encrypt par exemple).

```bash
rm -f tls.key tls.crt         # ne jamais laisser une clé privée dans un dossier de travail
```

</details>

---

## Partie 5 : 🩺 Diagnostic - les pannes d'un Ingress

**5.1 - Un Ingress sans classe**

Créez un Ingress `orphelin` identique à `webapp` mais pour l'hôte `orphelin.<INGRESS_IP>.nip.io`,
**sans** le champ `ingressClassName`. Suivez la colonne `ADDRESS` pendant 3 à 5 minutes, et lisez les
événements de l'objet. Quelle adresse obtient-il ? Est-ce celle de votre contrôleur NGINX ? Qui l'a
pris en charge ?

<details><summary>Correction</summary>

```bash
kubectl create ingress orphelin --rule="orphelin.$INGRESS_IP.nip.io/*=webapp:80"
kubectl get ingress orphelin -w       # ADDRESS vide... puis une AUTRE adresse IP
kubectl describe ingress orphelin | tail -5
# Normal  Sync  ...  loadbalancer-controller  Scheduled for sync
kubectl get ingressclass -o yaml | grep is-default-class      # aucune classe par défaut
kubectl delete ingress orphelin       # supprimez-le vite : c'est un équilibreur facturé
```

Sans `ingressClassName`, NGINX ignore l'Ingress (il n'est pas la classe par défaut). Mais sur GKE,
le **contrôleur intégré de Google** prend en charge les Ingress qui ne désignent aucune classe : il
crée un **équilibreur HTTP Google Cloud**, avec sa propre IP, en quelques minutes. Votre règle est
donc publiée ailleurs que prévu, sans les annotations NGINX (réécriture, redirection HTTPS...).

🏠 **En local** - sans contrôleur par défaut, la colonne `ADDRESS` resterait simplement **vide** :
l'objet est accepté par l'API, mais il ne fait rien, et rien ne le signale.

🔎 **Attention** - un Ingress sans `ADDRESS`, ou avec une adresse inattendue : vérifiez d'abord
`ingressClassName`. L'ancienne annotation `kubernetes.io/ingress.class` est dépréciée.

</details>

**5.2 - Un 503**

Passez le Deployment `webapp` à **0 réplica**, puis interrogez `https://$HOST_APP/`. Quel code HTTP
obtenez-vous ? Pourquoi est-ce différent d'un 404 ? Restaurez 3 réplicas.

<details><summary>Correction</summary>

```bash
kubectl scale deployment webapp --replicas=0
curl -k -i https://$HOST_APP/ | head -1
# HTTP/2 503
kubectl get endpoints webapp          # <none>
kubectl scale deployment webapp --replicas=3
```

| Code | Signification côté Ingress | Où chercher |
| --- | --- | --- |
| **404** (pas de log applicatif) | aucune règle ne correspond (hôte, chemin) | `kubectl get ingress`, en-tête `Host` |
| **404** (log applicatif) | la règle marche, l'application ne connaît pas le chemin | réécriture d'URL |
| **503** | la règle marche, mais le Service n'a **aucun endpoint** | `kubectl get endpoints`, Pods `Ready` (Lab 07) |
| **502** | le backend répond mal ou pas sur le port annoncé | `targetPort`, logs de l'application |

```bash
# Les logs du contrôleur montrent chaque requête et sa destination
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=10
```

</details>

---

## Pour aller plus loin : la Gateway API

La **Gateway API** est le successeur standard de l'Ingress. Elle répartit les responsabilités sur
trois ressources : `GatewayClass` (le type de point d'entrée, fourni par l'infrastructure),
`Gateway` (un point d'entrée concret, géré par l'équipe plateforme) et `HTTPRoute` (les règles d'une
application, gérées par l'équipe applicative). La réécriture d'URL, la redirection HTTPS ou la
répartition pondérée (canari) y sont des **filtres standard**, et non des annotations propres à un
contrôleur. GKE la prend en charge nativement, et la plupart des contrôleurs (NGINX Gateway Fabric,
Envoy Gateway, Istio, Cilium...) l'implémentent.

---

## ✅ État attendu en fin de lab

`https://$HOST_APP` affiche Countvisit via l'Ingress ; `http://$HOST_APPS/static` affiche la page
nginx ; le Service `webapp` est en `ClusterIP` et le Deployment à 3 réplicas ; aucun Ingress
`orphelin` ne subsiste. Conservez `webapp-ingress.yaml`, et ne désinstallez pas Ingress NGINX : il
resservira aux Labs 14 et 15.

---

## Points clés

* Un Ingress est une **règle** : c'est le **contrôleur** désigné par `ingressClassName` qui l'applique.
  Sur GKE, un Ingress **sans classe** est pris en charge par le contrôleur de Google.
* `backend.service.port` désigne le port du **Service**, pas du conteneur.
* `pathType` est obligatoire ; `Prefix` raisonne par segments d'URL.
* Secret TLS : clés `tls.crt` / `tls.key`, dans le namespace de l'Ingress.
* 404 = pas de règle (ou chemin mal réécrit), 503 = pas d'endpoint derrière le Service.
* Les annotations sont propres au contrôleur ; la **Gateway API** les remplace par des filtres
  standard.
