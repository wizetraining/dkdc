# LAB 10 - Ordonnancement et ressources

> **Cluster : GKE** (contexte `cluster-lab-dkdc`) | **Durée : ~50 min**
> **Prérequis :** cluster GKE de 3 nœuds (Lab 00).

Le **scheduler** choisit le nœud de chaque Pod en fonction des ressources disponibles et des
contraintes que vous exprimez. Ce lab couvre les deux questions qui décident du placement :
**où** un Pod a le droit d'aller, et **combien** il réserve.

Ce lab est **indépendant** de l'application Countvisit : travaillez dans un namespace dédié `sched`.
Image utilisée : `public.ecr.aws/wizetraining/nginx:latest`. Les noms des nœuds GKE étant longs
(`gke-formation-alice-default-pool-1a2b3c4d-xk9p`), enregistrez-les dans trois variables :

```bash
kubectl create namespace sched
kubectl config set-context --current --namespace=sched

read N1 N2 N3 <<< $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}')
echo "N1=$N1"; echo "N2=$N2"; echo "N3=$N3"
```

Le lab suivant (Lab 11) s'appuiera sur ces notions pour rendre Countvisit **élastique**.

## Objectifs

1. Contrôler **où** tournent les Pods : `nodeSelector`, **taints/tolerations**, **anti-affinité**,
   **node pools** GKE.
2. Dimensionner les Pods : **requests**, **limits**, classes de **QoS**, `OOMKilled`.
3. Encadrer un namespace avec un **LimitRange** et un **ResourceQuota**.
4. Mesurer la consommation réelle avec `kubectl top`.
5. Lire les messages d'un Pod `Pending` et d'un Pod `OOMKilled`.

**Documentation utile :** [Affecter des Pods aux nœuds](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) · [Taints et tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) · [Gérer les ressources des conteneurs](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) · [QoS des Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/) · [LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/) · [ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas/) · [Node pools GKE](https://cloud.google.com/kubernetes-engine/docs/concepts/node-pools)

---

## Partie 1 : Placer les Pods

### 1.1 - nodeSelector (révision du Lab 02)

**1.** Simulez une infrastructure hétérogène en labellisant les nœuds : `$N1` → `disk=hdd`,
`$N2` et `$N3` → `disk=ssd`.

**2.** Créez un Pod `pod-ssd` (nginx ECR) qui doit tourner **uniquement** sur un nœud `disk=ssd`
(`nodeSelector`). Sur quel nœud arrive-t-il ? Que se passe-t-il si vous demandez `disk=nvme` ?

<details><summary>Correction</summary>

```bash
kubectl label node $N1 disk=hdd
kubectl label node $N2 $N3 disk=ssd
kubectl get nodes -L disk
```

```yaml
# pod-ssd.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-ssd
spec:
  nodeSelector:
    disk: ssd
  containers:
  - name: nginx
    image: public.ecr.aws/wizetraining/nginx:latest
```

```bash
kubectl apply -f pod-ssd.yaml
kubectl get pod pod-ssd -o wide        # $N2 ou $N3
```

Avec `disk: nvme`, aucun nœud ne correspond : le Pod reste `Pending`, et `kubectl describe pod`
affiche `0/3 nodes are available: ... didn't match Pod's node affinity/selector`.

</details>

### 1.2 - Taints et tolerations

Un **taint** *repousse* les Pods d'un nœud, sauf ceux qui portent la **toleration** correspondante.
C'est le mécanisme qui sert à réserver un nœud (matériel dédié, équipe, maintenance).

**3.** Posez sur `$N1` le taint `team=blue:NoSchedule`. Créez un Pod `pod-hdd` avec un
`nodeSelector` `disk=hdd` (il ne peut donc aller que sur `$N1`). Que se passe-t-il ?
Lisez le message dans `describe`.

<details><summary>Correction</summary>

```bash
kubectl taint nodes $N1 team=blue:NoSchedule
kubectl describe node $N1 | grep -i taint

kubectl run pod-hdd --image=public.ecr.aws/wizetraining/nginx:latest \
  --overrides='{"spec":{"nodeSelector":{"disk":"hdd"}}}'
kubectl get pod pod-hdd                       # Pending
kubectl describe pod pod-hdd | tail -3
# 0/3 nodes are available: 1 node(s) had untolerated taint {team: blue}, ...
```

| Effet | Nouveaux Pods | Pods déjà en place |
| --- | --- | --- |
| `NoSchedule` | refusés | conservés |
| `PreferNoSchedule` | évités si possible | conservés |
| `NoExecute` | refusés | **expulsés** |

</details>

**4.** Ajoutez la toleration adéquate pour que `pod-hdd` soit planifié sur `$N1`.

<details><summary>Correction</summary>

```yaml
# pod-hdd.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-hdd
spec:
  nodeSelector:
    disk: hdd
  tolerations:
  - key: team
    operator: Equal
    value: blue
    effect: NoSchedule
  containers:
  - name: nginx
    image: public.ecr.aws/wizetraining/nginx:latest
```

```bash
kubectl delete pod pod-hdd && kubectl apply -f pod-hdd.yaml
kubectl get pod pod-hdd -o wide        # $N1
```

🔎 **Attention** - une toleration **autorise** un nœud tainté, elle **n'attire pas** le Pod vers lui.
C'est le rôle du `nodeSelector` ou de l'affinité. Pour réserver un nœud à une équipe, il faut donc
le taint **et** un label.

🏠 **En local** - c'est aussi par un taint que le nœud du plan de contrôle reste à l'écart des Pods
applicatifs : kubeadm y pose `node-role.kubernetes.io/control-plane:NoSchedule`. Sur GKE, la
question ne se pose pas : le plan de contrôle n'est pas un nœud du cluster.

</details>

### 1.3 - Anti-affinité de Pods

Pour la haute disponibilité, on ne veut pas que tous les réplicas d'une application tombent avec un
seul nœud.

**5.** Créez un Deployment `web-ha` (nginx ECR, 3 réplicas, label `app: web-ha`) avec une
**podAntiAffinity** de type *required* : deux Pods `app=web-ha` ne peuvent pas partager le même nœud
(`topologyKey: kubernetes.io/hostname`). Où atterrissent les Pods ? Combien sont `Running` alors que
le taint est toujours posé sur `$N1` ? Et à 4 réplicas ?

<details><summary>Correction</summary>

```yaml
# web-ha.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-ha
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-ha
  template:
    metadata:
      labels:
        app: web-ha
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: web-ha
            topologyKey: kubernetes.io/hostname
      containers:
      - name: nginx
        image: public.ecr.aws/wizetraining/nginx:latest
```

```bash
kubectl apply -f web-ha.yaml
kubectl get pods -l app=web-ha -o wide
```

Avec le taint sur `$N1` : 2 Pods `Running` (sur `$N2` et `$N3`), 1 `Pending` : il ne reste
aucun nœud libre, le Deployment n'ayant pas la toleration. À 4 réplicas : 2 `Pending`.

`topologyKey` fixe l'échelle de la dispersion : `kubernetes.io/hostname` = un Pod par **nœud** ;
`topology.kubernetes.io/zone` = un Pod par **zone**.

☁️ **Sur GKE** - GKE renseigne ces labels de topologie sur chaque nœud. Votre cluster est
**régional** : ses 3 nœuds sont dans 3 zones différentes (`kubectl get nodes -L topology.kubernetes.io/zone`).
Ici, un Pod par nœud revient donc à un Pod par zone : l'application survit à la panne d'une zone
entière. Avec plusieurs nœuds par zone, il faudrait `topologyKey: topology.kubernetes.io/zone` pour
obtenir cette garantie.

</details>

**6.** Retirez le taint. Les Pods `Pending` se planifient-ils ? Combien au maximum ? Que changerait
`preferredDuringSchedulingIgnoredDuringExecution` ?

<details><summary>Correction</summary>

```bash
kubectl taint nodes $N1 team=blue:NoSchedule-      # le "-" final retire le taint
kubectl get pods -l app=web-ha -o wide
```

Les Pods `Pending` sont planifiés dès qu'un nœud devient éligible. Mais avec 3 nœuds et une règle
*required*, 3 Pods au maximum peuvent tourner : le 4e reste `Pending`.

☁️ **Sur GKE** - avec l'autoscaling des nœuds (évoqué au Lab 11), ce Pod `Pending` déclencherait
l'ajout d'un **nouveau nœud** : c'est même le signal qu'il attend.

Avec `preferred...`, la dispersion devient un **souhait** : le scheduler essaie de l'honorer, mais
place quand même le Pod si c'est impossible. Aucun Pod ne reste `Pending`. En production, c'est
souvent le meilleur compromis.

🧹 `kubectl delete deployment web-ha; kubectl delete pod pod-ssd pod-hdd; kubectl label nodes --all disk-`

</details>

### 1.4 - Pour aller plus loin : les node pools

*(Lecture seulement : on ne crée pas de node pool dans ce lab.)*

Les labels et taints posés avec `kubectl` sur un nœud GKE sont **fragiles** : quand GKE remplace un
nœud (mise à jour automatique, réparation, redimensionnement), le nouveau nœud n'en porte plus aucun.
La bonne pratique est de les déclarer sur le **node pool**, un groupe de nœuds identiques : GKE les
applique alors à chacun de ses nœuds, présents et futurs.

Un cluster de production combine souvent plusieurs pools : un pool général, un pool à mémoire
élevée, un pool GPU, un pool de VM *Spot* moins chères pour les traitements interruptibles... Pour
réserver un pool `pool-blue` à l'équipe bleue, on le crée avec le label `team=blue` et le taint
`team=blue:NoSchedule` (dans la console, section *Métadonnées* du node pool), et les Pods de l'équipe
déclarent :

```yaml
    spec:
      nodeSelector:
        cloud.google.com/gke-nodepool: pool-blue     # label posé automatiquement par GKE (ou team: blue)
      tolerations:
      - key: team
        operator: Equal
        value: blue
        effect: NoSchedule
```

On retrouve la règle de la question 4 : le **taint** écarte les autres Pods du pool, le
**nodeSelector** attire les Pods de l'équipe vers lui. Il faut les deux.

🏠 **En local** - les nœuds sont des machines que vous gérez : un label ou un taint posé avec
`kubectl` y reste tant que le nœud n'est pas réinstallé. On peut aussi les fixer au démarrage du
kubelet (options `--node-labels` et `--register-with-taints`).

---

## Partie 2 : Requests, limits et QoS

* **Requests** : ce que le conteneur **réserve**. **C'est la seule valeur que le scheduler regarde.**
* **Limits** : le **plafond**. CPU dépassé → le conteneur est **ralenti** (*throttling*).
  Mémoire dépassée → il est **tué** (`OOMKilled`).

**2.1 - Une première déclaration**

Créez un Pod `res-pod` (nginx ECR) avec des requests `cpu: 100m`, `memory: 64Mi` et des limits
`cpu: 200m`, `memory: 128Mi`. Quelle est sa classe de **QoS** (champ `.status.qosClass`) ?

<details><summary>Correction</summary>

```yaml
# res-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: res-pod
spec:
  containers:
  - name: nginx
    image: public.ecr.aws/wizetraining/nginx:latest
    resources:
      requests: { cpu: 100m, memory: 64Mi }
      limits:   { cpu: 200m, memory: 128Mi }
```

```bash
kubectl apply -f res-pod.yaml
kubectl get pod res-pod -o jsonpath='{.status.qosClass}'; echo      # Burstable
```

`100m` = 100 millicores = 0,1 CPU. `64Mi` = 64 mébioctets.

</details>

**2.2 - Les trois classes de QoS**

Quelle QoS aurait un Pod avec **requests = limits** pour le CPU et la mémoire ? Et un Pod sans
aucune section `resources` ? Et un Pod qui ne déclare **que** des limits ? Vérifiez en créant les
Pods `qos-guaranteed`, `qos-besteffort` et `qos-limits-only`.

<details><summary>Correction</summary>

```bash
kubectl run qos-guaranteed --image=public.ecr.aws/wizetraining/nginx:latest \
  --overrides='{"spec":{"containers":[{"name":"c","image":"public.ecr.aws/wizetraining/nginx:latest","resources":{"requests":{"cpu":"100m","memory":"64Mi"},"limits":{"cpu":"100m","memory":"64Mi"}}}]}}'
kubectl run qos-besteffort --image=public.ecr.aws/wizetraining/nginx:latest
kubectl run qos-limits-only --image=public.ecr.aws/wizetraining/nginx:latest \
  --overrides='{"spec":{"containers":[{"name":"c","image":"public.ecr.aws/wizetraining/nginx:latest","resources":{"limits":{"cpu":"100m","memory":"64Mi"}}}]}}'

kubectl get pods -o custom-columns=NOM:.metadata.name,QOS:.status.qosClass
```

| QoS | Condition | Sous pression mémoire sur le nœud |
| --- | --- | --- |
| `Guaranteed` | requests = limits, CPU **et** mémoire, pour **tous** les conteneurs | évincé en dernier |
| `Burstable` | au moins une request ou limit, sans égalité stricte | évincé ensuite |
| `BestEffort` | aucune request ni limit | évincé **en premier** |

🔎 **Attention** - un Pod qui ne déclare **que** des limits est `Guaranteed` : Kubernetes recopie
automatiquement les limits dans les requests.

</details>

**2.3 - 🩺 Un Pod trop gourmand**

Créez un Pod `trop-gros` (nginx ECR) qui **demande** (`requests`) `100` CPU. Que se passe-t-il ?
Quel est le message exact ? Que prouve-t-il sur le rôle des requests ?

<details><summary>Correction</summary>

```bash
kubectl run trop-gros --image=public.ecr.aws/wizetraining/nginx:latest \
  --overrides='{"spec":{"containers":[{"name":"c","image":"public.ecr.aws/wizetraining/nginx:latest","resources":{"requests":{"cpu":"100"}}}]}}'
kubectl get pod trop-gros                    # Pending
kubectl describe pod trop-gros | tail -3
# 0/3 nodes are available: ... 3 Insufficient cpu ...
```

Le scheduler place les Pods selon les **requests** (réservations), **pas** selon la consommation
réelle. Un nœud à 5 % de CPU consommé peut refuser un Pod si ses requests cumulées sont saturées :

```bash
kubectl describe node $N2 | grep -A8 "Allocated resources"
```

☁️ **Sur GKE** - comparez `Capacity` et `Allocatable` dans `kubectl describe node $N2` : GKE
**réserve** une partie du CPU et de la mémoire de chaque nœud pour le système (kubelet, runtime,
système d'exploitation) et pour le seuil d'éviction. Sur une `e2-medium` (2 vCPU partagés, 4 Go),
moins d'**1 vCPU** (environ 940m) et moins de 3 Go restent allouables aux Pods, et les composants de
GKE (DNS, journalisation, métriques...) en réservent déjà une bonne partie. Sur de petits nœuds,
chaque request compte.

**Récapitulatif des causes d'un Pod `Pending`** (message dans `describe pod`) :

| Message | Cause |
| --- | --- |
| `didn't match Pod's node affinity/selector` | aucun nœud ne porte le label demandé |
| `had untolerated taint` | taint sans toleration |
| `Insufficient cpu` / `Insufficient memory` | requests supérieures à la capacité libre |
| `didn't match pod anti-affinity rules` | règle d'anti-affinité impossible à satisfaire |
| `unbound immediate PersistentVolumeClaims` | volume non disponible (Lab 06) |

</details>

**2.4 - 🩺 OOMKilled**

Déployez le Pod ci-dessous (limite de 50Mi, tente d'allouer 100M) et observez-le. Quel état, quel
code de sortie ? Pourquoi la mémoire et le CPU ne se comportent-ils pas de la même façon ?

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: oom-test
spec:
  containers:
  - name: stress
    image: polinux/stress
    resources:
      limits:
        memory: 50Mi
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "100M", "--vm-hang", "1"]
```

*(Cette image n'est pas sur ECR : si elle n'est pas accessible depuis votre cluster, prévenez le
formateur.)*

<details><summary>Correction</summary>

```bash
kubectl apply -f oom-test.yaml
kubectl get pod oom-test -w                   # OOMKilled, puis CrashLoopBackOff
kubectl describe pod oom-test | grep -A4 "Last State"
# Reason: OOMKilled   Exit Code: 137
```

| Ressource | Nature | Dépassement de la limite |
| --- | --- | --- |
| CPU | **compressible** | le conteneur est ralenti, il continue de tourner |
| Mémoire | **incompressible** | le conteneur est tué (SIGKILL, code **137**) |

💡 **Bon réflexe** - `Exit Code: 137` = tué par SIGKILL, presque toujours un dépassement mémoire.
Corrigez la limite, ou l'application.

🧹 `kubectl delete pod res-pod qos-guaranteed qos-besteffort qos-limits-only trop-gros oom-test`

</details>

---

## Partie 3 : LimitRange et ResourceQuota

* **LimitRange** : agit sur **chaque conteneur** à sa création (valeurs par défaut, bornes).
* **ResourceQuota** : agit sur **l'ensemble du namespace** (le « budget » de l'équipe).

Créez le namespace `quota-lab` et faites-en votre namespace courant.

```bash
kubectl create namespace quota-lab
kubectl config set-context --current --namespace=quota-lab
```

**3.1 - LimitRange**

Créez un LimitRange `defaults` qui :

* injecte par défaut `cpu: 100m` / `memory: 256Mi` en **request** et `cpu: 500m` / `memory: 512Mi`
  en **limit** ;
* interdit tout conteneur dépassant `1Gi` de mémoire.

Créez un Pod nginx `no-limits` **sans** section `resources` : quelles valeurs reçoit-il ? Essayez
ensuite un Pod `too-big` avec une limite mémoire de `2Gi`.

<details><summary>Correction</summary>

```yaml
# limitrange.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
spec:
  limits:
  - type: Container
    defaultRequest: { cpu: 100m, memory: 256Mi }
    default:        { cpu: 500m, memory: 512Mi }
    max:            { memory: 1Gi }
```

```bash
kubectl apply -f limitrange.yaml
kubectl run no-limits --image=public.ecr.aws/wizetraining/nginx:latest
kubectl get pod no-limits -o jsonpath='{.spec.containers[0].resources}'; echo   # valeurs injectées

kubectl run too-big --image=public.ecr.aws/wizetraining/nginx:latest \
  --overrides='{"spec":{"containers":[{"name":"c","image":"public.ecr.aws/wizetraining/nginx:latest","resources":{"limits":{"memory":"2Gi"}}}]}}'
# Error from server (Forbidden): ... maximum memory usage per Container is 1Gi, but limit is 2Gi
```

Le refus vient de l'**API Server**, à l'admission : le Pod n'est **jamais créé**. Ce n'est pas un
`Pending`. Un LimitRange ne s'applique qu'aux Pods créés **après** lui.

</details>

**3.2 - ResourceQuota**

Limitez le namespace à `requests.cpu: 500m`, `requests.memory: 512Mi`, `limits.cpu: 1`,
`limits.memory: 1Gi`. Déployez `nginx` en **2 réplicas** avec requests = limits = `200m` CPU /
`128Mi`, puis passez à **5 réplicas**. Combien de Pods tournent ? Où voyez-vous la cause ?

<details><summary>Correction</summary>

```yaml
# quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute
spec:
  hard:
    requests.cpu: 500m
    requests.memory: 512Mi
    limits.cpu: "1"
    limits.memory: 1Gi
```

```bash
kubectl apply -f quota.yaml
kubectl delete pod no-limits      # il consomme déjà une partie du quota

kubectl create deployment nginx --image=public.ecr.aws/wizetraining/nginx:latest --replicas=2 $do > nginx.yaml
# ajouter dans le conteneur :
#   resources:
#     requests: { cpu: 200m, memory: 128Mi }
#     limits:   { cpu: 200m, memory: 128Mi }
kubectl apply -f nginx.yaml
kubectl scale deployment nginx --replicas=5

kubectl get deployment nginx               # 2/5
kubectl get pods                           # seulement 2 Pods : les 3 autres n'existent pas
kubectl describe replicaset -l app=nginx | tail -3
# Error creating: pods "nginx-..." is forbidden: exceeded quota: compute,
#   requested: requests.cpu=200m, used: requests.cpu=400m, limited: requests.cpu=500m
```

🔎 **Attention** - un quota dépassé n'est **pas** un `Pending` : les Pods ne sont **jamais créés**.
L'erreur n'apparaît ni sur le Deployment ni dans `kubectl get pods`, mais dans les **événements du
ReplicaSet**.

</details>

**3.3 - Suivre le quota**

Quelle commande montre la consommation du quota ? Pourquoi un quota sur `requests.cpu` rend-il un
LimitRange quasi indispensable ?

<details><summary>Correction</summary>

```bash
kubectl describe resourcequota compute     # colonnes Used / Hard
```

Dès qu'un ResourceQuota porte sur `requests.cpu` ou `limits.memory`, **tout** Pod du namespace doit
déclarer ces valeurs, sinon il est refusé (`must specify limits.cpu...`). Le LimitRange les injecte
automatiquement pour les Pods qui les oublient : les deux objets vont presque toujours par paire.

</details>

---

## Partie 4 : Mesurer la consommation réelle

Affichez la consommation CPU et mémoire des nœuds, puis les 5 Pods les plus gourmands en mémoire de
tout le cluster. Quel composant fournit ces chiffres ? Comparez la consommation réelle d'un nœud à
ses **réservations** (`describe node`).

<details><summary>Correction</summary>

```bash
kubectl top nodes
kubectl top pods -A --sort-by=memory | head -6
kubectl top pod -n countvisit --containers
kubectl describe node $N2 | grep -A8 "Allocated resources"
```

C'est **metrics-server** (fourni et géré par GKE dans `kube-system`) qui fournit ces chiffres. Il ne garde aucun
historique : c'est un outil d'autoscaling et de diagnostic instantané, pas de supervision. Si
`kubectl top` répond `Metrics API not available`, vérifiez ses Pods dans `kube-system`.

La comparaison montre souvent un écart important : un nœud peut être réservé à 90 % (requests) et
consommé à 10 %. C'est la réservation qui décide du placement.

☁️ **Sur GKE** - pour l'historique, Cloud Monitoring collecte les métriques du cluster (console :
*Kubernetes Engine → Clusters → votre cluster → Observabilité*). La console affiche aussi, par
Deployment, une recommandation de requests calculée sur la consommation passée.

🏠 **En local** - metrics-server s'installe à part (souvent via Helm) ; pour l'historique, on déploie
Prometheus et Grafana, ou Elastic Agent avec l'intégration Kubernetes.

</details>

---

## 🧹 Nettoyage

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace quota-lab sched
kubectl get nodes -L disk                       # plus aucun label disk
kubectl describe nodes | grep -i "taints:"      # plus de taint team=blue
```

---

## Points clés

* `nodeSelector` = placement strict par label. Taint = **repousse** ; toleration = **autorise**
  (n'attire pas).
* Sur GKE, labels et taints durables se déclarent sur le **node pool** : posés à la main avec
  `kubectl`, ils disparaissent quand GKE remplace un nœud.
* `NoSchedule` n'expulse pas les Pods en place ; `NoExecute` si.
* Anti-affinité `required` = un refus de planifier ; `preferred` = un souhait.
* Le scheduler ne regarde que les **requests**. Les **limits** sont appliquées sur le nœud.
* CPU dépassé = ralenti ; mémoire dépassée = **OOMKilled**, code **137**.
* QoS : `Guaranteed` (requests = limits) > `Burstable` > `BestEffort`, évincés dans l'ordre inverse.
* `Pending` : lisez le message `FailedScheduling`, il nomme la cause.
* LimitRange = par conteneur ; ResourceQuota = par namespace. Un quota dépassé empêche la
  **création** du Pod : erreur dans les événements du **ReplicaSet**.
* `kubectl top` nécessite metrics-server (fourni par GKE). `Allocatable` < `Capacity` : GKE réserve
  des ressources pour le système.
