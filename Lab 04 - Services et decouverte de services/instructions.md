# LAB 04 - Services et découverte de services

> **Cluster : GKE** (contexte `cluster-lab-dkdc`) | **Durée : ~55 min**
> **Prérequis :** Lab 03 terminé (Deployment `webapp`, v1, 3 réplicas, namespace `countvisit` par défaut).
> Vous travaillez dans `~/labs/countvisit`.

Les Pods sont éphémères : leur IP change à chaque recréation, et un rolling update les remplace
tous. Un **Service** fournit une IP et un **nom DNS stables** devant un ensemble de Pods choisis par
**selector de labels**, et répartit le trafic entre eux. Le nom DNS, lui, est servi par le **DNS du
cluster** (kube-dns sur GKE).

## Objectifs

1. Comprendre pourquoi les Pods ont besoin d'un Service (IP stable, répartition, DNS).
2. Utiliser les types `ClusterIP`, `NodePort` et `LoadBalancer`, et voir ce que GKE crée dans
   Google Cloud.
3. Relier Countvisit à **PostgreSQL** grâce au DNS interne.
4. Comprendre la **résolution des noms** dans le cluster.
5. Diagnostiquer un Service sans endpoints, un mauvais port, et un nom qui ne se résout pas.

**Documentation utile :** [Services](https://kubernetes.io/docs/concepts/services-networking/service/) · [DNS pour Services et Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) · [Service LoadBalancer sur GKE](https://cloud.google.com/kubernetes-engine/docs/concepts/service-load-balancer) · [kube-dns sur GKE](https://cloud.google.com/kubernetes-engine/docs/how-to/kube-dns) · [Débugger la résolution DNS](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/) · [Débugger un Service](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)

---

## Partie 1 : Service ClusterIP

**1.1 - Le problème**

Relevez les IP de vos Pods `webapp`. Supprimez-en un : le Pod recréé a-t-il la même IP ?

<details><summary>Correction</summary>

```bash
kubectl get pods -o wide
kubectl delete pod <un-pod>
kubectl get pods -o wide          # le nouveau Pod a une nouvelle IP
```

Aucun client ne peut donc se fier à l'IP d'un Pod.

</details>

**1.2 - Créer le Service**

Écrivez `webapp-service.yaml` : un Service `webapp` de type `ClusterIP`, qui expose le port **80**
et redirige vers le port **5000** des Pods portant le label `app=webapp`. Appliquez-le.

<details><summary>Correction</summary>

```yaml
# webapp-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  type: ClusterIP           # valeur par défaut
  selector:
    app: webapp             # les Pods ciblés
  ports:
  - port: 80                # port du SERVICE
    targetPort: 5000        # port du CONTENEUR
```

```bash
kubectl apply -f webapp-service.yaml
kubectl get service webapp
```

Équivalent impératif : `kubectl expose deployment webapp --port=80 --target-port=5000`.

</details>

**1.3 - Les endpoints**

Quels sont les *endpoints* du Service ? Supprimez un Pod puis réaffichez-les : que constatez-vous ?

<details><summary>Correction</summary>

```bash
kubectl describe service webapp | grep Endpoints        # 3 adresses IP:5000
kubectl get endpointslices -l kubernetes.io/service-name=webapp

kubectl delete pod <un-pod>
kubectl get endpointslices -l kubernetes.io/service-name=webapp   # liste mise à jour
```

La chaîne complète est :

```text
Service --selector--> Pods Ready --> EndpointSlice (liste IP:port) --> règles réseau sur chaque nœud
```

Le Service suit automatiquement les Pods qui correspondent à son selector **et qui sont `Ready`**.
L'**EndpointSlice** est l'objet moderne ; l'ancien objet `Endpoints` reste lisible
(`kubectl get endpoints webapp`).

</details>

**1.4 - Joindre le Service depuis le cluster**

Depuis un Pod temporaire (image busybox ECR, commande `wget -qO-`), interrogez le Service sous les
noms `webapp`, `webapp.countvisit` et `webapp.countvisit.svc.cluster.local`. Recommencez depuis le
namespace `default` : lesquels fonctionnent encore ?

<details><summary>Correction</summary>

```bash
# Depuis countvisit
kubectl run tmp --rm -it --restart=Never --image=public.ecr.aws/wizetraining/busybox:latest \
  -- wget -qO- http://webapp | head -5

# Depuis default
kubectl run tmp --rm -it --restart=Never -n default --image=public.ecr.aws/wizetraining/busybox:latest \
  -- wget -qO- http://webapp.countvisit | head -5
```

Dans le même namespace, `webapp` suffit. Depuis un autre namespace, il faut au minimum
`webapp.countvisit`. Vous comprendrez pourquoi en Partie 4.

💡 **Astuce** - `kubectl run tmp --rm -it --restart=Never --image=... -- <commande>` crée un Pod
jetable, exécute la commande et le supprime. C'est l'outil de test réseau le plus rapide.

</details>

---

## Partie 2 : NodePort et LoadBalancer

`ClusterIP` n'est joignable que depuis l'intérieur du cluster. Pour y accéder depuis l'extérieur :

* **NodePort** : ouvre un même port (plage 30000-32767) sur **tous** les nœuds ;
* **LoadBalancer** : demande une IP externe à l'infrastructure. Sur GKE, c'est un **équilibreur de
  charge réseau Google Cloud** qui est créé automatiquement.

**2.1 - NodePort, depuis le cluster**

Passez le Service `webapp` en `NodePort` (modifiez le YAML et ré-appliquez). Quel port a été
alloué ? Depuis un Pod jetable, interrogez l'**IP interne** d'un nœud sur ce port, puis celle d'un
autre nœud.

<details><summary>Correction</summary>

```bash
# type: NodePort dans webapp-service.yaml
kubectl apply -f webapp-service.yaml
kubectl get service webapp            # PORT(S) : 80:3xxxx/TCP
NODEPORT=$(kubectl get service webapp -o jsonpath='{.spec.ports[0].nodePort}')
kubectl get nodes -o wide             # colonnes INTERNAL-IP et EXTERNAL-IP

NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
kubectl run tmp --rm -it --restart=Never --image=public.ecr.aws/wizetraining/busybox:latest \
  -- wget -qO- http://$NODE_IP:$NODEPORT | head -5
```

Le port est ouvert sur **chaque** nœud, même sur ceux qui n'hébergent aucun Pod `webapp`.

🔎 **Attention** - trois ports à ne pas confondre :
`nodePort` (sur le nœud, 30000-32767) → `port` (sur le Service) → `targetPort` (dans le conteneur).
Pour fixer le `nodePort` vous-même, il faut l'écrire dans le YAML : `kubectl expose` ne le permet pas.

</details>

**2.2 - NodePort, depuis l'extérieur**

Depuis le terminal de Cloud Shell, tentez `http://<EXTERNAL-IP d'un nœud>:<nodePort>` avec
`curl --max-time 5`. Que se passe-t-il ? Pourquoi, alors que le port est bien ouvert sur le nœud ?

<details><summary>Correction</summary>

```bash
NODE_EXT=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}')
curl --max-time 5 http://$NODE_EXT:$NODEPORT          # délai dépassé
```

Kubernetes a bien ouvert le port sur le nœud (la question 2.1 le prouve depuis l'intérieur), mais le
**pare-feu du réseau** qui entoure les machines bloque, par défaut, tout trafic entrant qui n'est pas
explicitement autorisé. Ouvrir la plage des NodePorts demanderait une règle de pare-feu dans Google
Cloud : c'est une tâche d'administration du réseau, hors du périmètre de Kubernetes, que nous ne
ferons pas ici.

Un NodePort expose directement les nœuds : en production on lui préfère un `LoadBalancer` ou un
Ingress (Lab 08), et on n'ouvre jamais la plage des NodePorts à tout Internet.

🏠 **En local** - les nœuds sont sur votre réseau (par exemple des VM VirtualBox en
`192.168.56.x`) : `http://<IP-du-nœud>:<nodePort>` fonctionne directement depuis votre poste.

</details>

**2.3 - LoadBalancer**

Passez le Service en `LoadBalancer`. Que contient la colonne `EXTERNAL-IP`, et combien de temps
faut-il pour qu'elle soit remplie ? Ouvrez l'application sur cette IP, port 80. Rafraîchissez
plusieurs fois la page : que change le nom du conteneur affiché ? Retrouvez l'équilibreur dans la
console Google Cloud.

<details><summary>Correction</summary>

```bash
# type: LoadBalancer dans webapp-service.yaml
kubectl apply -f webapp-service.yaml
kubectl get service webapp -w         # EXTERNAL-IP : <pending> pendant ~1 minute, puis une IP publique
LB_IP=$(kubectl get service webapp -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -s http://$LB_IP | grep conteneur
kubectl describe service webapp | tail -5        # événements : EnsuringLoadBalancer, EnsuredLoadBalancer
```

GKE a créé un **équilibreur de charge réseau** Google Cloud (console : *Services réseau →
Équilibrage de charge*), ainsi que la règle de pare-feu nécessaire : c'est le **contrôleur du
fournisseur cloud** qui répond à la demande exprimée par le Service. Un Service `LoadBalancer` **inclut** un NodePort, qui inclut lui-même une ClusterIP :
chaque type ajoute une couche au précédent.

Le nom du conteneur varie d'un rafraîchissement à l'autre : les requêtes sont réparties entre les
3 Pods. La page affiche toujours « Echec de connexion à la base de données » : il manque PostgreSQL.

Chaque Service `LoadBalancer` crée un équilibreur **facturé** et une IP publique : on en limite le
nombre (Lab 08).

🏠 **En local** - aucun fournisseur cloud ne répond à la demande : `EXTERNAL-IP` reste `<pending>`
indéfiniment, sauf si l'on installe un composant comme **MetalLB**, qui attribue des IP d'une plage
du réseau local.

</details>

> Laissez le Service en `LoadBalancer` : vous en aurez besoin pour voir l'application jusqu'au
> Lab 08.

---

## Partie 3 : PostgreSQL et découverte de services

L'application cherche par défaut un serveur nommé **`postgres`** sur le port 5432, avec
l'utilisateur `postgres` et le mot de passe `postgres` (variables `PG_*`, voir la fiche du Lab 03).

**3.1 - La base de données**

Créez `postgres-deployment.yaml` : un Deployment `postgres`, 1 réplica, image
`public.ecr.aws/wizetraining/postgres:16-alpine`, label `app: postgres`, port 5432.
L'image PostgreSQL **refuse de démarrer** sans la variable d'environnement `POSTGRES_PASSWORD` :
donnez-lui la valeur attendue par l'application. Appliquez, puis vérifiez dans les logs que la
base est prête.

<details><summary>Correction</summary>

```yaml
# postgres-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: public.ecr.aws/wizetraining/postgres:16-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          value: postgres
```

```bash
kubectl apply -f postgres-deployment.yaml
kubectl logs deploy/postgres | tail -3      # "database system is ready to accept connections"
```

</details>

**3.2 - Le Service de la base**

Créez `postgres-service.yaml`. Quel **nom** et quel **type** de Service faut-il pour que
l'application trouve sa base sans aucune configuration ? Pourquoi ce type ?

<details><summary>Correction</summary>

```yaml
# postgres-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres          # c'est ce nom que l'application résout (PG_HOST par défaut)
spec:
  type: ClusterIP         # seule l'application doit joindre la base : aucune exposition externe
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

```bash
kubectl apply -f postgres-service.yaml
```

🔎 **Attention** - le **nom** d'un Service est une donnée fonctionnelle : c'est le nom DNS que les
clients résolvent. Un Service mal nommé donne une application qui ne se connecte pas, sans
**aucune** erreur côté Kubernetes.

</details>

**3.3 - Constater la découverte**

Rechargez la page de l'application plusieurs fois. Consultez les logs d'un Pod `webapp`. Depuis un
Pod `webapp`, quelle IP est résolue pour `postgres` ? Comparez-la à la ClusterIP du Service.

<details><summary>Correction</summary>

La page affiche « Connecté à la base de données » et le compteur s'incrémente.

```bash
kubectl logs deploy/webapp | tail -3                      # "Connexion réussie ! Visiteur n°..."
kubectl exec deploy/webapp -- python -c "import socket; print(socket.gethostbyname('postgres'))"
kubectl get service postgres                              # même IP : la ClusterIP
```

Le nom `postgres` est résolu en **ClusterIP du Service**, et non en IP du Pod : si le Pod
PostgreSQL est recréé, l'application n'a rien à changer.

</details>

**3.4 - Les limites**

Supprimez le Pod PostgreSQL, attendez qu'il soit recréé, rechargez la page. **Que devient le
compteur ? Pourquoi ?**

<details><summary>Correction</summary>

```bash
kubectl delete pod -l app=postgres
kubectl get pods -w
```

Le compteur **repart à 1** : le système de fichiers d'un conteneur est éphémère et les données
disparaissent avec lui. C'est le point de départ du Lab 06 (stockage persistant). Inutile de
redémarrer l'application : elle se reconnecte à chaque chargement de page.

</details>

---

## Partie 4 : La résolution des noms

C'est le DNS du cluster, déployé dans `kube-system`, qui a résolu `postgres` à la Partie 3.

**4.1 - Le format des noms**

Depuis un Pod jetable busybox du namespace `countvisit`, résolvez le Service `postgres` avec la
commande `nslookup`, sous ses quatre formes : `postgres`, `postgres.countvisit`,
`postgres.countvisit.svc` et `postgres.countvisit.svc.cluster.local`. Recommencez avec le nom
court `postgres` depuis le namespace `default`.

<details><summary>Correction</summary>

```bash
kubectl run dnstest --rm -it --restart=Never --image=public.ecr.aws/wizetraining/busybox:latest -- sh

# Dans le Pod :
nslookup postgres
nslookup postgres.countvisit
nslookup postgres.countvisit.svc
nslookup postgres.countvisit.svc.cluster.local
exit

kubectl run dnstest --rm -it --restart=Never -n default --image=public.ecr.aws/wizetraining/busybox:latest \
  -- nslookup postgres          # NXDOMAIN : il n'y a pas de Service postgres dans default
```

Le format complet :

```text
<service>.<namespace>.svc.cluster.local
   │          │         │       └ domaine du cluster
   │          │         └ type d'objet
   │          └ namespace
   └ nom du Service
```

🔎 **Attention** - depuis `default`, le nom court `postgres` cherche `postgres.default`, **pas**
`postgres.countvisit`. Un nom court ne traverse pas les namespaces.

</details>

**4.2 - Comment un Pod sait-il tout cela ?**

Affichez le contenu du fichier `/etc/resolv.conf` d'un Pod jetable. Retrouvez à quoi correspond
l'adresse `nameserver` (indice : un Service du namespace `kube-system`).

<details><summary>Correction</summary>

```bash
kubectl run dnstest --rm -it --restart=Never --image=public.ecr.aws/wizetraining/busybox:latest \
  -- cat /etc/resolv.conf
```

```text
search countvisit.svc.cluster.local svc.cluster.local cluster.local
nameserver 34.118.224.10        # exemple : l'adresse dépend de la plage des Services du cluster
options ndots:5
```

```bash
kubectl get service -n kube-system kube-dns       # CLUSTER-IP : la même adresse
kubectl get service kubernetes -n default         # la 1re adresse de la plage des Services
# Le DNS prend, par convention, la 10e adresse de cette même plage.
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

* `nameserver` : la ClusterIP du Service `kube-dns` ;
* `search` : les suffixes essayés automatiquement. C'est ce qui fait marcher `postgres` tout court,
  et ce qui explique que le premier suffixe soit **le namespace du Pod** ;
* `ndots:5` : tout nom de moins de 5 points est d'abord essayé avec chaque suffixe de `search`.
  Pratique en interne, mais un nom externe comme `www.google.com` génère jusqu'à 4 requêtes
  inutiles avant d'être résolu.

C'est le **kubelet** qui écrit ce fichier dans chaque Pod, à partir de sa propre configuration
(adresse du DNS du cluster et domaine `cluster.local`).

</details>

**4.3 - Le DNS de GKE**

Retrouvez dans `kube-system` le Deployment qui sert le DNS du cluster et le Service vers lequel
pointe le `nameserver` de vos Pods. Combien de réplicas tournent ? Quel autre Deployment ajuste ce
nombre ? Où se configurerait, par exemple, un serveur DNS d'entreprise pour un domaine interne ?

<details><summary>Correction</summary>

```bash
kubectl get deployment -n kube-system -l k8s-app=kube-dns
kubectl get deployment -n kube-system kube-dns-autoscaler
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get service -n kube-system kube-dns
```

Par défaut, GKE utilise **kube-dns**, et non CoreDNS. C'est un composant **géré par Google** :

* `kube-dns-autoscaler` ajuste le nombre de réplicas à la taille du cluster ;
* les requêtes pour les noms hors du cluster sont transmises au serveur DNS de Google Cloud (via le
  serveur de métadonnées du nœud) ;
* pour un domaine d'entreprise (par exemple `corp.elastic.co`), on déclare un *stub domain* dans
  une ConfigMap `kube-dns` du namespace `kube-system`, sans toucher au Deployment ;
* GKE propose aussi **Cloud DNS pour GKE**, qui remplace kube-dns par le service DNS managé de
  Google Cloud.

Ne modifiez **pas** le Deployment `kube-dns` : GKE le réconcilie et annulerait vos changements.

🏠 **En local** - un cluster kubeadm utilise **CoreDNS** (Deployment `coredns`, mêmes labels
`k8s-app=kube-dns` et même nom de Service `kube-dns` pour la compatibilité). Sa configuration est un
fichier `Corefile` stocké dans la ConfigMap `coredns`, que l'administrateur modifie lui-même : par
exemple `forward . 8.8.8.8` pour changer les serveurs DNS externes.

</details>

---

## Partie 5 : 🩺 Diagnostic - Services et DNS

**5.1 - Un Service sans endpoints**

C'est l'une des pannes les plus courantes. Créez le Service suivant, puis tentez de le joindre
depuis un Pod jetable comme en Partie 1.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-broken
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 5000
```

Pourquoi la requête échoue-t-elle ? Quelle commande le montre immédiatement ? Corrigez.

<details><summary>Correction</summary>

```bash
kubectl apply -f webapp-broken.yaml
kubectl describe service webapp-broken | grep Endpoints    # Endpoints: <none>
kubectl get pods --show-labels                             # les Pods portent app=webapp
```

Le selector `app=web-app` ne correspond à aucun Pod : le Service n'a aucune cible.

```bash
kubectl patch service webapp-broken -p '{"spec":{"selector":{"app":"webapp"}}}'
kubectl get endpoints webapp-broken                        # 3 adresses
```

</details>

**5.2 - Des endpoints, mais pas de réponse**

Modifiez le `targetPort` de `webapp-broken` en `8080` et testez de nouveau. Les endpoints sont-ils
toujours présents ? Isolez la couche fautive en testant successivement : l'IP d'un Pod sur le port
5000, la ClusterIP du Service, puis le nom du Service.

<details><summary>Correction</summary>

```bash
kubectl patch service webapp-broken -p '{"spec":{"ports":[{"port":80,"targetPort":8080}]}}'
kubectl get endpoints webapp-broken          # présents, mais sur le port 8080

POD_IP=$(kubectl get pod -l app=webapp -o jsonpath='{.items[0].status.podIP}')
SVC_IP=$(kubectl get service webapp-broken -o jsonpath='{.spec.clusterIP}')

kubectl run tmp --rm -it --restart=Never --image=public.ecr.aws/wizetraining/busybox:latest \
  -- sh -c "wget -qO- -T 3 $POD_IP:5000 | head -3; echo ---; wget -qO- -T 3 $SVC_IP:80; echo ---; wget -qO- -T 3 webapp-broken"
```

L'IP du Pod répond, la ClusterIP non : le problème est entre le Service et le conteneur, c'est-à-dire
le `targetPort`.

| Ce qui marche | Ce qui échoue | Conclusion |
| --- | --- | --- |
| rien, même l'IP du Pod | tout | l'application n'écoute pas, ou pas sur ce port |
| l'IP du Pod | la ClusterIP | selector, `targetPort`, Pods non `Ready`, ou NetworkPolicy |
| la ClusterIP | le nom DNS | DNS du cluster, ou un nom incomplet (mauvais namespace) |
| tout en interne | depuis l'extérieur | NodePort, LoadBalancer, Ingress ou pare-feu |

```bash
kubectl delete service webapp-broken
```

💡 **Bon réflexe** - « le Service ne répond pas » : la **première** commande est
`kubectl get endpoints <service>`. Vide → selector ou Pods non `Ready`. Rempli → port ou application.

</details>

**5.3 - Un nom qui ne se résout pas**

Une équipe déploie un outil de supervision dans le namespace `default` et lui donne l'adresse
`postgres:5432` pour surveiller la base. Simulez-le avec un Pod jetable de `default` qui lance
`nslookup postgres`. Puis, **comme si vous ne connaissiez pas la cause**, menez le diagnostic :
le DNS fonctionne-t-il ? Avec quels suffixes le nom a-t-il été essayé ? Quel nom faut-il donner ?

<details><summary>Correction</summary>

```bash
# 1. Symptôme
kubectl run dnstest -n default --rm -it --restart=Never --image=public.ecr.aws/wizetraining/busybox:latest \
  -- nslookup postgres
# ** server can't find postgres.default.svc.cluster.local: NXDOMAIN

# 2. Le DNS fonctionne-t-il ? Un nom de référence se résout-il ?
kubectl run dnstest -n default --rm -it --restart=Never --image=public.ecr.aws/wizetraining/busybox:latest \
  -- nslookup kubernetes.default.svc.cluster.local          # OK : le DNS répond

# 3. Avec quels suffixes ?
kubectl run dnstest -n default --rm -it --restart=Never --image=public.ecr.aws/wizetraining/busybox:latest \
  -- cat /etc/resolv.conf                                   # search default.svc.cluster.local ...

# 4. Le bon nom
kubectl run dnstest -n default --rm -it --restart=Never --image=public.ecr.aws/wizetraining/busybox:latest \
  -- nslookup postgres.countvisit
```

Le DNS fonctionne : c'est le **nom** qui est incomplet. Depuis `default`, `postgres` est cherché
comme `postgres.default.svc.cluster.local`, qui n'existe pas. Il faut `postgres.countvisit` (ou le
nom complet `postgres.countvisit.svc.cluster.local`).

**L'arbre de diagnostic DNS, dans l'ordre :**

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns          # 1. le DNS tourne-t-il ?
kubectl get endpoints kube-dns -n kube-system                # 2. le Service DNS a-t-il des cibles ?
kubectl exec <pod> -- cat /etc/resolv.conf                   # 3. le client pointe-t-il au bon endroit ?
kubectl run t --rm -it --restart=Never --image=public.ecr.aws/wizetraining/busybox:latest \
  -- nslookup kubernetes.default.svc.cluster.local            # 4. un nom de référence se résout-il ?
kubectl run t --rm -it --restart=Never --image=public.ecr.aws/wizetraining/busybox:latest \
  -- nslookup <service>.<namespace>                           # 5. le nom complet se résout-il ?
```

| Symptôme | Cause probable |
| --- | --- |
| rien ne résout, nulle part | DNS du cluster à terre ou Service `kube-dns` sans endpoints |
| les noms internes résolvent, pas les externes | la résolution externe (DNS des nœuds, ou configuration du DNS du cluster) |
| ça résout depuis certains Pods seulement | une NetworkPolicy bloque le port 53 |
| le nom court échoue, le nom complet marche | mauvais namespace |

</details>

---

## ✅ État attendu en fin de lab

* Service `webapp` en `LoadBalancer`, avec une IP externe ;
* Deployment `postgres` et Service `postgres` (ClusterIP) ;
* la page Countvisit affiche « Connecté à la base de données » et un compteur ;
* vous avez constaté que le compteur ne survit pas à un redémarrage de PostgreSQL.

Fichiers dans `~/labs/countvisit` : `webapp-deployment.yaml`, `webapp-service.yaml`,
`postgres-deployment.yaml`, `postgres-service.yaml`.

---

## Points clés

| Type | Portée | Usage |
| --- | --- | --- |
| `ClusterIP` | interne au cluster | communication entre composants (défaut) |
| `NodePort` | `<IP-nœud>:30000-32767` | accès de test ; sur GKE, nécessite une règle de pare-feu |
| `LoadBalancer` | IP externe | GKE crée un équilibreur réseau Google Cloud, facturé - à réserver aux points d'entrée (Lab 08) |

* Un Service choisit ses Pods par **selector** et ne garde que les Pods **`Ready`**.
* `kubectl get endpoints <svc>` : la première commande de diagnostic d'un Service.
* `port` (Service) ≠ `targetPort` (conteneur) ≠ `nodePort` (nœud).
* `<service>.<namespace>.svc.cluster.local` : le format des noms. Un nom court ne traverse pas les
  namespaces.
* Sur GKE, le DNS du cluster (kube-dns) est **géré par Google** : on ne le modifie pas ; on le
  personnalise au besoin par la ConfigMap `kube-dns` (domaines d'entreprise). En local, c'est
  CoreDNS, configuré par l'administrateur.
