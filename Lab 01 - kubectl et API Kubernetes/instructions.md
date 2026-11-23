# LAB 01 - kubectl, contextes et API Kubernetes

> **Cluster : GKE** (contexte `cluster-lab-dkdc`) | **Durée : ~50 min**
> **Prérequis :** Lab 00 (cluster créé, `kubectl` connecté, alias `k`).

`kubectl` n'est qu'un **client** : tout ce qu'il fait, c'est envoyer des requêtes HTTP à l'**API
Server** du cluster. Comprendre cette API (comment on s'y connecte, ce qu'elle contient, comment
elle se décrit elle-même) rend Kubernetes beaucoup moins mystérieux.

## Objectifs

1. Comprendre comment `kubectl` sait à quel cluster parler, et avec quelle identité (kubeconfig,
   contextes, namespace par défaut).
2. Interroger le cluster lui-même : `cluster-info`, versions, état de santé.
3. Voir ce que GKE vous montre (et vous cache) du plan de contrôle.
4. Explorer l'API : groupes, ressources, `kubectl explain`.
5. Appeler l'API REST directement, sans `kubectl`.
6. Extraire précisément une information (`jsonpath`, `custom-columns`, tri).

**Documentation utile :** [Organiser l'accès aux clusters avec kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/) · [Composants de Kubernetes](https://kubernetes.io/docs/concepts/overview/components/) · [L'API Kubernetes](https://kubernetes.io/docs/concepts/overview/kubernetes-api/) · [Aide-mémoire kubectl](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

---

## Partie 1 : Le kubeconfig et les contextes

`kubectl` lit par défaut le fichier `~/.kube/config` (ou celui désigné par la variable `KUBECONFIG`).
Il contient trois listes : des **clusters** (adresse de l'API Server et autorité de certification),
des **users** (comment s'authentifier) et des **contexts** (un cluster + un user + un namespace
optionnel).

**1.1 - Les contextes**

Listez les contextes de votre kubeconfig. Combien en avez-vous, et lequel est actif ?

<details><summary>Correction</summary>

```bash
kubectl config get-contexts
kubectl config current-context        # cluster-lab-dkdc
```

L'astérisque (`*`) marque le contexte actif. Si vous avez déjà travaillé avec d'autres clusters
(Docker Desktop, minikube, un autre cluster GKE...), ils apparaissent aussi : toujours vérifier le
contexte actif avant une commande qui modifie quelque chose.

</details>

**1.2 - Comment kubectl s'authentifie-t-il auprès de GKE ?**

Affichez la configuration du **seul** contexte actif (option `--minify`). Repérez l'adresse de
l'API Server, puis regardez la section `users` : où est le mot de passe ou le certificat ?

<details><summary>Correction</summary>

```bash
kubectl config view --minify
```

```yaml
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://34.xx.xx.xx          # l'API Server de votre cluster
  name: gke_<projet>_<zone>_<cluster>
users:
- name: gke_<projet>_<zone>_<cluster>
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: gke-gcloud-auth-plugin     # appelé à chaque commande kubectl
      provideClusterInfo: true
```

**Il n'y a aucun secret** dans le fichier. À chaque commande, `kubectl` exécute
`gke-gcloud-auth-plugin`, qui obtient un **jeton d'accès Google temporaire** pour votre compte
`gcloud`. L'API Server de GKE vérifie ce jeton auprès de Google : votre identité Kubernetes est
votre **compte Google**.

```bash
kubectl auth whoami          # Username : votre adresse e-mail Google
```

🏠 **En local** - avec un cluster installé par kubeadm, la section `users` contient un **certificat
client et sa clé**, intégrés en base64 (`client-certificate-data`, `client-key-data`). Quiconque
copie le fichier obtient vos droits, jusqu'à l'expiration du certificat.

</details>

**1.3 - Les namespaces et le namespace par défaut**

Lancez les commandes suivantes, et répondez aux questions :

```bash
kubectl run pod-default --image=public.ecr.aws/wizetraining/nginx:latest
kubectl create namespace dev
kubectl run pod-dev -n dev --image=public.ecr.aws/wizetraining/nginx:latest
kubectl get pods
```

* Combien de Pods voyez-vous ? Voyez-vous `pod-dev` ? Pourquoi ?
* Quelles commandes affichent respectivement les Pods de `default`, ceux de `dev`, et ceux de tous
  les namespaces ?

<details><summary>Correction</summary>

`kubectl get pods` n'affiche que `pod-default` : sans option `-n`, `kubectl` travaille dans le
**namespace du contexte**, ici `default`.

```bash
kubectl get pods -n default
kubectl get pods -n dev
kubectl get pods -A                   # --all-namespaces
```

Un **namespace** cloisonne les noms, les droits et les quotas : deux équipes peuvent avoir chacune
un Pod `api` sans conflit, à condition qu'ils soient dans des namespaces différents.

</details>

**1.4 - Un contexte par namespace de travail**

Vous travaillez surtout dans `dev`. Pour ne plus taper `-n dev` :

1. créez un **second contexte** nommé `cluster-lab-dev`, qui utilise le même cluster et le même user que
   `cluster-lab-dkdc`, avec le namespace par défaut `dev` ;
2. basculez dessus et listez les Pods sans option `-n` ;
3. revenez sur `cluster-lab-dkdc`.

Puis montrez une seconde méthode, qui change le namespace du contexte **courant** sans en créer
un nouveau.

<details><summary>Correction</summary>

```bash
CLUSTER=$(kubectl config view --minify -o jsonpath='{.contexts[0].context.cluster}')
USER=$(kubectl config view --minify -o jsonpath='{.contexts[0].context.user}')

kubectl config set-context cluster-lab-dev --cluster="$CLUSTER" --user="$USER" --namespace=dev
kubectl config use-context cluster-lab-dev
kubectl get pods                      # pod-dev
kubectl config use-context cluster-lab-dkdc
```

Seconde méthode, très utilisée au quotidien :

```bash
kubectl config set-context --current --namespace=dev
kubectl get pods                      # pod-dev
kubectl config set-context --current --namespace=default
```

Modifier le fichier `~/.kube/config` à la main fonctionne aussi, mais `kubectl config` évite les
erreurs d'indentation.

🧹 `kubectl delete pod pod-default; kubectl delete namespace dev; kubectl config delete-context cluster-lab-dev`

</details>

---

## Partie 2 : Interroger le cluster

**2.1 - cluster-info**

Affichez les informations de base du cluster. Quelles adresses obtenez-vous ? Comparez l'adresse
de l'API Server avec le **point de terminaison** affiché dans l'onglet **Détails** de votre cluster,
dans la console.

<details><summary>Correction</summary>

```bash
kubectl cluster-info
```

```text
Kubernetes control plane is running at https://34.xx.xx.xx
GLBCDefaultBackend is running at https://34.xx.xx.xx/api/v1/namespaces/kube-system/services/default-http-backend:http/proxy
KubeDNS is running at https://34.xx.xx.xx/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://34.xx.xx.xx/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
```

Même adresse que dans la console : c'est l'**adresse publique** du plan de contrôle géré par
Google. Les autres lignes sont des Services du cluster accessibles via l'API Server (DNS, métriques,
et le *backend par défaut* du contrôleur Ingress propre à GKE, mentionné au Lab 08).

`kubectl cluster-info dump` produit un état complet du cluster (objets, logs) : utile pour joindre
un diagnostic à un ticket de support.

</details>

**2.2 - Versions et santé**

Affichez la version du client `kubectl` et celle du serveur. Puis interrogez les points de santé de
l'API Server, en détail.

<details><summary>Correction</summary>

```bash
kubectl version
kubectl get --raw='/version'
kubectl get --raw='/readyz?verbose' | tail -5
kubectl get --raw='/livez?verbose'  | tail -3
```

`/readyz` liste chaque vérification interne de l'API Server, dont la connexion à etcd. Sur GKE,
c'est le seul regard que vous ayez sur l'état du plan de contrôle : le reste est surveillé par
Google.

La version du client et celle du serveur peuvent différer d'**une** version mineure au plus.

</details>

---

## Partie 3 : Ce que GKE vous montre du plan de contrôle

**3.1 - Les Pods système**

Listez les Pods du namespace `kube-system`. Cherchez `kube-apiserver`, `etcd`, `kube-scheduler`,
`kube-controller-manager` : les trouvez-vous ? Pourquoi ? À quoi servent les Pods que vous voyez ?

<details><summary>Correction</summary>

```bash
kubectl get pods -n kube-system -o wide
kubectl get daemonsets,deployments -n kube-system
```

**Aucun** composant du plan de contrôle n'apparaît : ils tournent dans l'infrastructure de Google,
hors de votre cluster. Vous voyez en revanche les composants que Google installe **sur vos nœuds** :

| Composant | Rôle |
| --- | --- |
| `kube-dns` | le DNS interne du cluster (résolution des noms de Services, Lab 04) |
| `kube-proxy` ou `anetd` | règles réseau des Services sur chaque nœud (`anetd` avec GKE Dataplane V2, basé sur Cilium) |
| `konnectivity-agent` | tunnel entre le plan de contrôle de Google et vos nœuds (pour `kubectl logs`, `exec`...) |
| `metrics-server` | consommation CPU et mémoire (`kubectl top`, autoscaling) |
| `fluentbit-gke` | envoi des logs des conteneurs vers Cloud Logging |
| `gke-metrics-agent` | envoi des métriques vers Cloud Monitoring |
| `pdcsi-node` | pilote de stockage des disques persistants Google (Lab 06) |
| `l7-default-backend` | page renvoyée par les équilibreurs HTTP de Google quand aucune règle ne correspond (Lab 08) |

🏠 **En local** - avec kubeadm, ces quatre composants sont des **static pods** du namespace
`kube-system` (`kube-apiserver-controlplane`, `etcd-controlplane`...). Le kubelet de ce nœud les
démarre à partir de fichiers YAML placés dans `/etc/kubernetes/manifests/`, et vous pouvez lire
leurs options de démarrage avec `kubectl describe pod`.

</details>

**3.2 - Les plages d'adresses du cluster**

Puisque vous ne pouvez pas lire les options de l'API Server dans un Pod, retrouvez avec `kubectl` :

1. la version du plan de contrôle et celle des kubelets des nœuds ;
2. la plage d'adresses attribuée aux Pods de chaque nœud (champ `spec.podCIDR` des nœuds) ;
3. l'adresse IP du Service `kubernetes` du namespace `default` : c'est la **première** adresse de la
   plage réservée aux Services.

Comparez avec la section **Mise en réseau** de l'onglet **Détails** du cluster, dans la console.

<details><summary>Correction</summary>

```bash
kubectl version                                    # Server Version
kubectl get nodes -o custom-columns=NOEUD:.metadata.name,KUBELET:.status.nodeInfo.kubeletVersion,PODCIDR:.spec.podCIDR
kubectl get svc kubernetes -n default              # CLUSTER-IP : 34.118.224.1 par exemple
```

Chaque nœud reçoit un bloc d'adresses (un `/24`) dans lequel ses Pods prennent leur IP ; les Services
ont leur propre plage, distincte. Ces plages sont choisies à la création du cluster.

Sur un service managé, la configuration du plan de contrôle se lit et se modifie **chez le
fournisseur** (la console, ou en ligne de commande `gcloud container clusters describe`), et non
dans des fichiers.

🏠 **En local** - ces plages sont des options de `kubeadm init` (`--pod-network-cidr`,
`--service-cidr`), visibles dans le manifeste du Pod `kube-apiserver`.

</details>

**3.3 - Le kubelet sur un nœud**

Le kubelet, lui, tourne bien sur vos nœuds. Ouvrez un shell **sur un nœud**, sans SSH, avec
`kubectl debug node/<nom-du-nœud>` (image `public.ecr.aws/wizetraining/busybox:latest`), puis
basculez dans le système de fichiers du nœud (`chroot /host`). Affichez l'état du service `kubelet`
et ses options de démarrage. Supprimez ensuite le Pod de débogage.

<details><summary>Correction</summary>

```bash
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl debug node/$NODE -it --image=public.ecr.aws/wizetraining/busybox:latest

# Dans le Pod de débogage :
chroot /host
systemctl status kubelet --no-pager | head -5
ps aux | grep -m1 "[k]ubelet "
exit      # sortir du chroot
exit      # sortir du Pod

kubectl get pods -o name | grep node-debugger | xargs kubectl delete
```

`kubectl debug node/...` crée un Pod privilégié sur le nœud, avec le système de fichiers du nœud
monté sur `/host`. Le kubelet y est un service `systemd`, comme sur n'importe quel nœud Kubernetes.

🏠 **En local** - on se connecte au nœud en SSH (`vagrant ssh worker1`, par exemple) et on lance
directement `systemctl status kubelet`.

</details>

---

## Partie 4 : Explorer l'API

L'API Kubernetes est découpée en **groupes**, chacun versionné :

* le groupe **core** (historique), au chemin `/api/v1` : Pods, Services, ConfigMaps, Secrets... ;
* les **groupes nommés**, au chemin `/apis/<groupe>/<version>` : `apps/v1` (Deployments),
  `networking.k8s.io/v1` (Ingress), `storage.k8s.io/v1` (StorageClasses)...

**4.1 - Groupes et ressources**

Listez les groupes et versions d'API disponibles, puis toutes les ressources. Pour chaque ressource,
`kubectl api-resources` indique son nom court, son groupe et si elle est *namespacée*. Répondez :

* quel est le nom court de `deployments`, de `services`, de `persistentvolumeclaims` ?
* quelle est l'`apiVersion` d'une `StorageClass` et d'un `RoleBinding` ?
* les `nodes` et les `persistentvolumes` sont-ils namespacés ?

<details><summary>Correction</summary>

```bash
kubectl api-versions
kubectl api-resources
kubectl api-resources --namespaced=false
kubectl api-resources | grep -iE "^(deployments|services|persistentvolumeclaims|storageclasses|rolebindings) "
```

* noms courts : `deploy`, `svc`, `pvc` (`kubectl get deploy`) ;
* `storage.k8s.io/v1` et `rbac.authorization.k8s.io/v1` ;
* non : les nœuds et les PersistentVolumes appartiennent au cluster entier, pas à un namespace.

</details>

**4.2 - Les extensions de GKE**

Listez les ressources dont le nom de groupe contient `gke` ou `google`. D'où viennent-elles ?
Qu'est-ce que cela vous apprend sur l'API Kubernetes ?

<details><summary>Correction</summary>

```bash
kubectl api-resources | grep -iE "gke|google"
kubectl get crd | grep -iE "gke|google" | head
```

Vous y trouvez par exemple `managedcertificates.networking.gke.io` (certificats HTTPS gérés par
Google), `frontendconfigs` et `backendconfigs` (réglages des équilibreurs de Google), ou les objets de
supervision de Google.

Ce ne sont pas des ressources natives de Kubernetes : GKE a **étendu l'API** avec des
**CustomResourceDefinitions**. L'API Server les traite comme des objets natifs (`kubectl get`,
`explain`, droits RBAC). Vous créerez vos propres types d'objets au Lab 13.

</details>

**4.3 - kubectl explain : la documentation de l'API**

Sans ouvrir de navigateur, retrouvez avec `kubectl explain` :

1. à quoi sert le champ `spec.replicas` d'un Deployment, et son type ;
2. les champs disponibles sous `spec.strategy` d'un Deployment ;
3. le nom exact du champ qui fixe le délai avant la première vérification d'une sonde de vie
   (`livenessProbe`) d'un conteneur ;
4. toute l'arborescence des champs d'un Service, d'un coup.

<details><summary>Correction</summary>

```bash
kubectl explain deployment.spec.replicas               # <integer>
kubectl explain deployment.spec.strategy
kubectl explain pod.spec.containers.livenessProbe       # → initialDelaySeconds
kubectl explain service --recursive | less
```

`kubectl explain` interroge le **schéma de votre propre cluster** : la réponse correspond exactement
à sa version, y compris pour les ressources ajoutées par GKE ou par vous-même.

```bash
kubectl explain managedcertificate.spec            # une ressource propre à GKE
```

</details>

**4.4 - Ce que fait vraiment kubectl**

Relancez `kubectl get pods -n kube-system` avec un niveau de verbosité élevé (`-v=6`, puis `-v=8`).
Quelle requête HTTP `kubectl` envoie-t-il ?

<details><summary>Correction</summary>

```bash
kubectl get pods -n kube-system -v=6 2>&1 | grep -E "GET|POST"
# GET https://34.xx.xx.xx/api/v1/namespaces/kube-system/pods?limit=500 200 OK
kubectl get deployments -n kube-system -v=8 2>&1 | grep -m1 "GET"
# GET https://34.xx.xx.xx/apis/apps/v1/namespaces/kube-system/deployments?limit=500
```

Une commande `kubectl` = une requête HTTP sur l'API : `get` → `GET`, `apply` → `PATCH` ou `POST`,
`delete` → `DELETE`. On retrouve le chemin du groupe core (`/api/v1`) et celui d'un groupe nommé
(`/apis/apps/v1`).

</details>

---

## Partie 5 : Appeler l'API REST sans kubectl

**5.1 - Sans identité**

Récupérez l'adresse de l'API Server, puis interrogez-la avec `curl` sans aucune authentification :
`curl -k https://<adresse>/api/v1/namespaces`. Que répond-elle, et pourquoi ?

<details><summary>Correction</summary>

```bash
API=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
curl -sk $API/api/v1/namespaces
```

```json
{ "kind": "Status", "status": "Failure",
  "message": "namespaces is forbidden: User \"system:anonymous\" cannot list resource \"namespaces\" ...",
  "code": 403 }
```

L'API Server est joignable, mais vous êtes `system:anonymous` : aucune action n'est permise sans
identité.

</details>

**5.2 - Avec un jeton Google**

Sur GKE, l'API Server accepte les **jetons d'accès Google**. Obtenez-en un avec
`gcloud auth print-access-token`, puis refaites la requête avec l'en-tête
`Authorization: Bearer <jeton>`. Listez ainsi les noms des namespaces.

<details><summary>Correction</summary>

```bash
TOKEN=$(gcloud auth print-access-token)
curl -sk -H "Authorization: Bearer $TOKEN" $API/api/v1/namespaces | grep '"name"'
curl -sk -H "Authorization: Bearer $TOKEN" $API/apis/apps/v1/namespaces/kube-system/deployments | grep -c '"kind": "Deployment"'
```

C'est exactement ce que fait `gke-gcloud-auth-plugin` pour `kubectl` (Partie 1). L'option `-k`
désactive la vérification du certificat du serveur : pratique pour un test, à éviter ailleurs
(on fournirait plutôt l'autorité de certification du kubeconfig avec `--cacert`).

🏠 **En local** - on s'authentifierait avec le certificat client et la clé du kubeconfig
(`curl --cert client.crt --key client.key --cacert ca.crt ...`), ou avec le jeton d'un ServiceAccount.

</details>

**5.3 - Avec kubectl proxy**

Démarrez un proxy local de l'API avec `kubectl proxy --port=8001` (en arrière-plan), puis interrogez
`http://localhost:8001/version` et la liste des Pods de `kube-system`, cette fois **sans** jeton.
Pourquoi cela fonctionne-t-il ? Arrêtez ensuite le proxy.

<details><summary>Correction</summary>

```bash
kubectl proxy --port=8001 &
curl -s http://localhost:8001/version
curl -s http://localhost:8001/api/v1/namespaces/kube-system/pods | grep '"name"' | head -5
kill %1
```

Le proxy écoute en local (sur `127.0.0.1` uniquement) et ajoute **votre** identité à chaque requête
qu'il relaie. Pratique pour explorer l'API ou pour un script, à ne jamais exposer sur une interface
réseau.

</details>

---

## Partie 6 : Extraire précisément une information

**6.1 - Les formats de sortie**

Affichez :

1. le manifeste complet d'un nœud, en YAML ;
2. uniquement les **noms** des nœuds, sur une ligne (`jsonpath`) ;
3. un tableau `NOEUD` / `ZONE` / `TYPE` (`custom-columns`, à partir des labels vus au Lab 00) ;
4. les Pods de `kube-system` triés par date de création ;
5. la liste des images utilisées dans `kube-system`, sans doublon.

<details><summary>Correction</summary>

```bash
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl get node $NODE -o yaml | less

kubectl get nodes -o jsonpath='{.items[*].metadata.name}'; echo

kubectl get nodes -o custom-columns='NOEUD:.metadata.name,ZONE:.metadata.labels.topology\.kubernetes\.io/zone,TYPE:.metadata.labels.node\.kubernetes\.io/instance-type'

kubectl get pods -n kube-system --sort-by=.metadata.creationTimestamp

kubectl get pods -n kube-system -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}' \
  | tr ' ' '\n' | sort -u
```

Dans un `custom-columns` ou un `jsonpath`, les points contenus dans une clé de label doivent être
échappés (`topology\.kubernetes\.io/zone`).

Le format `-o yaml` est aussi le meilleur moyen d'apprendre : il montre **tous** les champs d'un
objet, y compris ceux que Kubernetes a remplis lui-même (section `status`, valeurs par défaut).

</details>

---

## ✅ État attendu en fin de lab

* le contexte courant est `cluster-lab-dkdc`, namespace `default` ;
* le namespace `dev`, les Pods de test et le contexte `cluster-lab-dev` sont supprimés ;
* aucun `kubectl proxy` ne tourne en arrière-plan (`jobs` ne renvoie rien).

---

## Points clés

* Le kubeconfig associe **clusters**, **users** et **contexts**. Sur GKE, il ne contient aucun secret :
  `gke-gcloud-auth-plugin` fournit un jeton Google temporaire à chaque commande.
* `kubectl config set-context --current --namespace=<ns>` change le namespace de travail.
* `kubectl cluster-info`, `kubectl version`, `kubectl get --raw='/readyz?verbose'` : l'état du cluster.
* Sur GKE, le plan de contrôle est **invisible** dans `kube-system` ; sa configuration se lit dans
  la console. Les nœuds, eux, exécutent un kubelet comme partout (`kubectl debug node/...`).
* L'API est découpée en groupes : `/api/v1` (core) et `/apis/<groupe>/<version>`. GKE l'étend avec
  ses propres CRD.
* `kubectl explain` documente chaque champ, à partir du schéma de **votre** cluster.
* `kubectl` n'est qu'un client HTTP (`-v=6`) ; `kubectl proxy` ou un jeton suffisent pour appeler
  l'API directement.
