# LAB 06 - Stockage persistant sur GKE

> **Cluster : GKE** (contexte `cluster-lab-dkdc`) | **Durée : ~55 min** (+ 15 min de bonus)
> **Prérequis :** Lab 05 terminé. Au Lab 04, vous avez vu le compteur retomber à 1 quand PostgreSQL
> redémarre : le système de fichiers d'un conteneur est éphémère.

Kubernetes sépare la **demande** de stockage, exprimée par l'application, de la **ressource** de
stockage, fournie par l'infrastructure :

* **PersistentVolume (PV)** : un morceau de stockage du cluster, le « disque ». Objet **global**,
  non namespacé.
* **PersistentVolumeClaim (PVC)** : la demande de stockage d'une application, la « réservation ».
  Objet **namespacé**.
* **StorageClass** : un « catalogue » qui crée les PV automatiquement à la demande d'un PVC, via un
  pilote **CSI**. Sur GKE, ce pilote crée des **disques Persistent Disk** de Google Cloud.

## Objectifs

1. Déclarer un volume à la main (PV + PVC), puis créer des disques **à la demande** (StorageClass).
2. Comprendre les **modes d'accès**, la **reclaim policy** et le **binding mode**.
3. Déployer PostgreSQL avec un **StatefulSet** pour que le compteur survive à la perte d'un Pod.
4. Comprendre ce que le StatefulSet fait... et ce qu'il ne fait pas.
5. Diagnostiquer un PVC qui reste `Pending`.
6. *Bonus* : sauvegarder et restaurer un volume par **snapshot**.

**Documentation utile :** [Volumes persistants](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) · [StorageClasses](https://kubernetes.io/docs/concepts/storage/storage-classes/) · [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) · [Persistent Disk sur GKE](https://cloud.google.com/kubernetes-engine/docs/concepts/persistent-volumes) · [Pilote CSI Persistent Disk](https://cloud.google.com/kubernetes-engine/docs/how-to/persistent-volumes/gce-pd-csi-driver) · [Snapshots de volumes](https://cloud.google.com/kubernetes-engine/docs/how-to/persistent-volumes/volume-snapshots)

---

## Partie 1 : Un volume déclaré à la main

Vous jouez ici le rôle de l'administrateur qui déclare **à la main** un volume existant, pour le
rendre utilisable par une application. Pour rester simple, ce volume est un **répertoire d'un nœud**
(type `hostPath`).

**1.1 - Le PersistentVolume**

Écrivez `pv-demo.yaml` : un PV `pv-demo` de **1Gi**, mode d'accès `ReadWriteOnce`, politique de
récupération `Retain`, avec `storageClassName: ""` (pas de classe, pour empêcher tout
provisionnement automatique), qui désigne le répertoire `/tmp/pv-demo` du nœud (`hostPath`, type
`DirectoryOrCreate`). Appliquez-le : quel est son statut ?

<details><summary>Correction</summary>

```yaml
# pv-demo.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-demo
spec:
  capacity:
    storage: 1Gi
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  hostPath:
    path: /tmp/pv-demo
    type: DirectoryOrCreate
```

```bash
kubectl apply -f pv-demo.yaml
kubectl get pv pv-demo           # STATUS : Available
```

Un PV est un objet **global** au cluster (non namespacé) : il décrit une ressource de stockage de
l'infrastructure. Le type de stockage (`hostPath`, `nfs`, `csi`...) est la seule partie qui change
d'une infrastructure à l'autre.

</details>

**1.2 - Le PersistentVolumeClaim**

Écrivez `pvc-demo.yaml` : un PVC `pvc-demo` qui réclame **1Gi** en `ReadWriteOnce`, avec la même
`storageClassName` vide, et qui désigne explicitement le PV `pv-demo` (champ `volumeName`). Que
devient le statut du PV ? Et celui du PVC ?

<details><summary>Correction</summary>

```yaml
# pvc-demo.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-demo
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: ""
  volumeName: pv-demo          # liaison explicite à ce PV
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f pvc-demo.yaml
kubectl get pv,pvc                # les deux passent à Bound
```

Les critères de liaison entre un PVC et un PV :

1. même `storageClassName` ;
2. `accessModes` compatibles ;
3. capacité du PV **supérieure ou égale** à la demande ;
4. `volumeName` ou `selector` du PVC, s'il en a un.

La liaison est **exclusive** : un PV lié ne sert plus aucun autre PVC.

</details>

**1.3 - Les modes d'accès**

À l'aide de la documentation, expliquez les quatre modes d'accès. Que faudrait-il pour qu'un même
volume soit monté en écriture par des Pods de plusieurs nœuds ?

<details><summary>Correction</summary>

| Mode | Abrév. | Signification |
| --- | --- | --- |
| `ReadWriteOnce` | RWO | lecture-écriture par **un seul nœud** (plusieurs Pods de ce nœud peuvent le partager) |
| `ReadOnlyMany` | ROX | lecture seule par plusieurs nœuds |
| `ReadWriteMany` | RWX | lecture-écriture par plusieurs nœuds |
| `ReadWriteOncePod` | RWOP | un **seul Pod**, quel que soit le nœud |

Pour du RWX, il faut un stockage de **fichiers partagé** (NFS, CephFS...). Sur GKE, c'est le service
**Filestore**. Les disques persistants classiques (Partie 2) sont RWO.

🔎 **Attention** - `ReadWriteOnce` se raisonne **par nœud**, pas par Pod. Un Deployment à 3 réplicas
sur des nœuds différents ne peut pas partager un PVC RWO : les Pods placés sur un autre nœud restent
bloqués en `ContainerCreating` (`Multi-Attach error`).

</details>

**1.4 - Utiliser le volume, puis changer de nœud**

Créez `pod-pv.yaml` : un Pod `pod-pv` (busybox ECR) qui monte le PVC `pvc-demo` dans `/data` et
exécute `sh -c "echo bonjour-de-\$(hostname) >> /data/out.txt && sleep 3600"`. Sur quel nœud
tourne-t-il ? Que contient `/data/out.txt` ?

Supprimez le Pod, puis recréez-le en l'obligeant à tourner sur un **autre** nœud (champ `nodeName`).
Que contient maintenant `/data/out.txt` ? Pourquoi ?

<details><summary>Correction</summary>

```yaml
# pod-pv.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-pv
spec:
  containers:
  - name: app
    image: public.ecr.aws/wizetraining/busybox:latest
    command: ["sh", "-c", "echo bonjour-de-$(hostname) >> /data/out.txt && sleep 3600"]
    volumeMounts:
    - name: vol
      mountPath: /data
  volumes:
  - name: vol
    persistentVolumeClaim:
      claimName: pvc-demo
```

```bash
kubectl apply -f pod-pv.yaml
kubectl get pod pod-pv -o wide
kubectl exec pod-pv -- cat /data/out.txt              # une ligne

# Recréer le Pod sur un autre nœud
CURRENT=$(kubectl get pod pod-pv -o jsonpath='{.spec.nodeName}')
OTHER=$(kubectl get nodes -o name | sed 's|node/||' | grep -v "$CURRENT" | head -1)
kubectl delete pod pod-pv
sed "s|^spec:|spec:\n  nodeName: $OTHER|" pod-pv.yaml | kubectl apply -f -
kubectl get pod pod-pv -o wide                        # sur l'autre nœud
kubectl exec pod-pv -- cat /data/out.txt              # UNE seule ligne : la nouvelle
```

La première ligne a **disparu** : elle est restée dans `/tmp/pv-demo` **du premier nœud**. Un volume
`hostPath` est un simple répertoire local, il ne suit pas le Pod. C'est pourquoi `hostPath` est à
proscrire pour des données applicatives : on lui préfère un **stockage réseau**, que le cluster
rattache au nœud où tourne le Pod. C'est l'objet de la partie suivante.

🧹 La politique `Retain` conserve le PV après suppression du PVC (statut `Released`) : il faut le
supprimer à la main.

```bash
kubectl delete pod pod-pv && kubectl delete pvc pvc-demo
kubectl get pv pv-demo            # Released
kubectl delete pv pv-demo
```

</details>

---

## Partie 2 : StorageClass et provisionnement dynamique

En pratique, personne ne crée les disques à la main : une **StorageClass** les fabrique à la demande,
dès qu'un PVC en a besoin.

**2.1 - Les StorageClasses de GKE**

Listez les StorageClasses du cluster. Laquelle est la classe **par défaut** ? Pour chacune, relevez
le `PROVISIONER`, la `RECLAIMPOLICY`, le `VOLUMEBINDINGMODE` et le type de disque (champ
`parameters`). Que signifient la politique de récupération et le mode de liaison ?

<details><summary>Correction</summary>

```bash
kubectl get storageclass
kubectl get storageclass -o custom-columns='NOM:.metadata.name,TYPE:.parameters.type,BINDING:.volumeBindingMode,RECLAIM:.reclaimPolicy'
kubectl get csidrivers
```

| Classe | Disque Google Cloud | Usage |
| --- | --- | --- |
| `standard-rwo` **(défaut)** | `pd-balanced` | usage général, bon compromis prix/performances |
| `premium-rwo` | `pd-ssd` | bases de données, faible latence |
| `standard` | `pd-standard` | disque magnétique, économique |

Toutes utilisent le pilote CSI `pd.csi.storage.gke.io`.

* `(default)` vient de l'annotation `storageclass.kubernetes.io/is-default-class: "true"` : un PVC
  **sans** `storageClassName` utilise cette classe.
* `reclaimPolicy` - ce qu'il advient du disque quand le PVC est supprimé :
  * `Delete` : le PV **et le disque Google Cloud** sont supprimés, données comprises ;
  * `Retain` : ils sont conservés pour une récupération manuelle.
* `volumeBindingMode` - quand le disque est créé :
  * `WaitForFirstConsumer` : seulement quand un Pod utilise le PVC, **dans la zone** de ce Pod ;
  * `Immediate` : dès la création du PVC.

🏠 **En local** - aucune classe n'existe par défaut : on installe un pilote de stockage (Longhorn,
Ceph, un serveur NFS...) qui fournit sa propre StorageClass.

</details>

**2.2 - Votre StorageClass**

Créez `sc-ssd.yaml` : une StorageClass `sc-ssd-<prénom>` pour des disques **SSD** (`pd-ssd`), avec le
pilote CSI de GKE, `reclaimPolicy: Delete`, `volumeBindingMode: WaitForFirstConsumer`, et
l'agrandissement de volume autorisé.

<details><summary>Correction</summary>

```yaml
# sc-ssd.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: sc-ssd-alice
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

```bash
kubectl apply -f sc-ssd.yaml
```

D'autres paramètres existent, par exemple `replication-type: regional-pd` pour un disque **répliqué
sur deux zones** (cluster régional) : la donnée survit à la perte d'une zone entière.

</details>

**2.3 - Un PVC dynamique**

Créez `pvc-dynamic.yaml` : un PVC `pvc-dynamic` de **10Gi** utilisant votre StorageClass. Que montre
`kubectl get pvc,pv` ? Si le PVC reste `Pending`, est-ce une panne ?

<details><summary>Correction</summary>

```yaml
# pvc-dynamic.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-dynamic
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: sc-ssd-alice
  resources:
    requests:
      storage: 10Gi
```

```bash
kubectl apply -f pvc-dynamic.yaml
kubectl get pvc,pv
kubectl describe pvc pvc-dynamic | tail -3
# waiting for first consumer to be created before binding
```

`Pending` est **normal** : avec `WaitForFirstConsumer`, aucun disque n'est créé tant qu'aucun Pod
n'utilise le PVC. Votre cluster est **régional** : cela garantit que le disque sera créé dans la
**zone** où le Pod sera planifié.

</details>

**2.4 - Le premier consommateur, puis un agrandissement**

1. Créez un Pod `pod-dyn` identique à `pod-pv` mais qui monte `pvc-dynamic`. Observez le PVC, le PV
   créé automatiquement, et sa section `nodeAffinity`. Le disque apparaît aussi dans la console
   (**Compute Engine → Disques**).
2. Agrandissez le volume à **20Gi**, sans arrêter le Pod, et vérifiez la taille vue **dans** le
   conteneur.
3. Supprimez le Pod, puis recréez-le sur un nœud d'une **autre zone** (`nodeName`). Que se passe-t-il ?
4. Supprimez le Pod puis le PVC : que devient le PV ?

<details><summary>Correction</summary>

```bash
sed 's/pvc-demo/pvc-dynamic/; s/name: pod-pv/name: pod-dyn/' pod-pv.yaml | kubectl apply -f -
kubectl get pvc,pv                                    # PVC Bound, un PV pvc-xxxx créé
PV=$(kubectl get pvc pvc-dynamic -o jsonpath='{.spec.volumeName}')
kubectl get pv $PV -o jsonpath='{.spec.nodeAffinity}'; echo
# ... topology.gke.io/zone In [europe-west1-c]   → le disque n'existe que dans cette zone

kubectl patch pvc pvc-dynamic -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
kubectl get pvc pvc-dynamic -w                         # CAPACITY passe à 20Gi
kubectl exec pod-dyn -- df -h /data                    # ~20G, sans redémarrage

# Une autre zone
CURRENT=$(kubectl get pod pod-dyn -o jsonpath='{.spec.nodeName}')
ZONE_CUR=$(kubectl get node $CURRENT -o jsonpath='{.metadata.labels.topology\.kubernetes\.io/zone}')
OTHER=$(kubectl get nodes -l "topology.kubernetes.io/zone!=$ZONE_CUR" -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod pod-dyn
sed "s/pvc-demo/pvc-dynamic/; s/name: pod-pv/name: pod-dyn/; s|^spec:|spec:\n  nodeName: $OTHER|" pod-pv.yaml | kubectl apply -f -
kubectl get pod pod-dyn                                # ContainerCreating... indéfiniment
kubectl describe pod pod-dyn | tail -3
# ... volume node affinity conflict   (ou : disk is in zone X, instance in zone Y)
kubectl delete pod pod-dyn

kubectl delete pvc pvc-dynamic
kubectl get pv                                         # le PV a disparu (et le disque avec lui)
```

Un disque persistant est **zonal** : il peut passer d'un nœud à l'autre **dans sa zone**, mais pas
d'une zone à l'autre. Le PV porte cette contrainte dans sa `nodeAffinity`, et le scheduler la
respecte : sans `nodeName` forcé, il aurait placé le Pod dans la bonne zone. Pour qu'une donnée
survive à la perte d'une zone, il faut la **répliquer** : disques régionaux (`regional-pd`), ou
réplication par l'application elle-même (Lab 13).

🔎 **Attention** - `nodeName` court-circuite le scheduler : c'est pour cela que le Pod a été envoyé
sur un nœud incompatible. N'utilisez `nodeName` que pour des tests.

L'agrandissement se fait **à chaud** (le disque, puis le système de fichiers), grâce à
`allowVolumeExpansion: true`. On ne peut qu'agrandir, jamais réduire.

`reclaimPolicy: Delete` est le défaut des classes de GKE : supprimer le PVC **détruit le disque et
ses données**. Pour un volume précieux, on change la politique du PV :

```bash
kubectl patch pv <nom-du-pv> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

</details>

---

## Partie 3 : 🩺 Diagnostic - un PVC qui reste Pending

Créez le PVC suivant, puis un Pod qui le monte (sur le modèle de `pod-pv`, nommé `pod-broken`).

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-broken
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: standrd-rwo
  resources:
    requests:
      storage: 10Gi
```

**3.1** - Dans quel état sont le PVC et le Pod ? Trouvez le message exact de chacun.

**3.2** - La classe d'un PVC ne peut pas être modifiée après création. Réparez.

<details><summary>Correction</summary>

```bash
kubectl get pvc pvc-broken                 # Pending
kubectl describe pvc pvc-broken | tail -3
# storageclass.storage.k8s.io "standrd-rwo" not found

kubectl get pod pod-broken                 # Pending
kubectl describe pod pod-broken | tail -3
# 0/3 nodes are available: pod has unbound immediate PersistentVolumeClaims
```

Le Pod est `Pending` **à cause** du PVC : on remonte toujours la chaîne Pod → PVC → StorageClass → PV.

```bash
kubectl delete pod pod-broken
kubectl delete pvc pvc-broken
# corriger storageClassName: standard-rwo, puis réappliquer le PVC et le Pod
```

| Message dans `describe pvc` | Cause |
| --- | --- |
| `storageclass ... not found` | nom de classe erroné |
| `waiting for first consumer` | **normal** avec `WaitForFirstConsumer` |
| `no persistent volumes available` | classe vide, aucun PV statique compatible (taille, mode d'accès) |
| `failed to provision volume` | le pilote CSI est en erreur, ou un **quota** Google Cloud est atteint (espace disque, nombre de disques) |

🧹 `kubectl delete pod pod-broken; kubectl delete pvc pvc-broken`

</details>

---

## Partie 4 : PostgreSQL avec un StatefulSet

Un **StatefulSet** est un contrôleur pour applications à état : chaque Pod reçoit une **identité
stable** (`postgres-0`, `postgres-1`...) et **son propre volume**, créé via `volumeClaimTemplates`,
qui le suit même après un redémarrage ou un déplacement.

**4.1 - Remplacer le Deployment**

Supprimez le Deployment `postgres` du Lab 05. **Gardez** le Service `postgres` : il sélectionne
`app=postgres` et pointera vers le nouveau Pod.

<details><summary>Correction</summary>

```bash
kubectl delete deployment postgres
kubectl get service postgres          # toujours là
```

</details>

**4.2 - Écrire le StatefulSet**

Créez `postgres-statefulset.yaml` contenant :

* un Service **headless** `postgres-headless` (`clusterIP: None`, port 5432, selector `app: postgres`) ;
* un StatefulSet `postgres` : 1 réplica, `serviceName: postgres-headless`, mêmes image, variables
  d'environnement (Secret et ConfigMap du Lab 05) et label `app: postgres` que le Deployment ;
* un `volumeClaimTemplates` nommé `data` de **10Gi** (`ReadWriteOnce`, classe par défaut), monté
  dans `/var/lib/postgresql/data` avec `subPath: pgdata` (PostgreSQL refuse un répertoire de
  données qui contient déjà `lost+found`).

<details><summary>Correction</summary>

```yaml
# postgres-statefulset.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None                 # pas d'IP virtuelle : le DNS renvoie directement les Pods
  selector:
    app: postgres
  ports:
  - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless  # DOIT désigner le Service headless
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
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: pg-secret
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: pg-secret
              key: password
        - name: POSTGRES_DB
          valueFrom:
            configMapKeyRef:
              name: webapp-config
              key: PG_DB
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
          subPath: pgdata
  volumeClaimTemplates:           # un PVC créé PAR réplica
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

La taille minimale d'un Persistent Disk est de 10 Go : une demande plus petite serait arrondie.

</details>

**4.3 - Observer**

Appliquez. Quels Pods et quels PVC ont été créés ? Comment s'appelle le PVC ?

<details><summary>Correction</summary>

```bash
kubectl apply -f postgres-statefulset.yaml
kubectl get pods,pvc -l app=postgres
kubectl get pvc
```

```text
pod/postgres-0                          Running     ← nom ordinal stable, pas de hash aléatoire
persistentvolumeclaim/data-postgres-0   Bound       ← <template>-<statefulset>-<ordinal>
```

</details>

**4.4 - Test de persistance**

Rechargez la page de l'application jusqu'à un compteur d'environ 5. Supprimez le Pod `postgres-0`,
attendez sa recréation, rechargez. Le compteur continue-t-il ?

<details><summary>Correction</summary>

```bash
kubectl delete pod postgres-0
kubectl get pods -w
```

Le compteur **reprend où il en était** : Kubernetes a rattaché le même PVC (`data-postgres-0`) au
Pod recréé sous le même nom.

</details>

**4.5 - Supprimer le StatefulSet entier**

Supprimez le StatefulSet (`kubectl delete statefulset postgres`), puis ré-appliquez-le. Le compteur
est-il toujours là ? Pourquoi ?

<details><summary>Correction</summary>

```bash
kubectl delete statefulset postgres
kubectl get pvc                       # data-postgres-0 est toujours là
kubectl apply -f postgres-statefulset.yaml
```

Les données sont intactes : **supprimer un StatefulSet ne supprime pas ses PVC**. C'est une
protection volontaire. Contrepartie : les PVC orphelins doivent être supprimés à la main.

</details>

**4.6 - Le Service headless**

Depuis un Pod `webapp`, résolvez `postgres-0.postgres-headless`. Quelle IP obtenez-vous ? À quoi
sert un Service headless ?

<details><summary>Correction</summary>

```bash
kubectl exec deploy/webapp -- python -c "import socket; print(socket.gethostbyname('postgres-0.postgres-headless'))"
kubectl get pod postgres-0 -o wide      # même IP : celle du Pod, pas une ClusterIP
```

Un Service headless (`clusterIP: None`) n'a pas d'IP virtuelle : le DNS renvoie directement les IP
des Pods, et donne à **chaque** Pod un nom individuel et stable
(`<pod>.<service>.<namespace>.svc.cluster.local`). Indispensable quand les membres d'un cluster de
base de données doivent se joindre un par un (primaire, réplicas).

</details>

---

## Partie 5 : La limite du StatefulSet

**5.1** - Passez le StatefulSet à **3 réplicas** et attendez `postgres-1` et `postgres-2`.
Rechargez la page une dizaine de fois en notant le compteur. Que constatez-vous ?

**5.2** - Expliquez pourquoi, à l'aide de `kubectl get pvc` et de ce que fait le Service `postgres`.

**5.3** - Revenez à 1 réplica et supprimez les PVC devenus inutiles.

<details><summary>Correction</summary>

```bash
kubectl scale statefulset postgres --replicas=3
kubectl get pods -l app=postgres -w
kubectl get pvc                         # 3 PVC distincts
```

**Constat :** le compteur est incohérent (15, puis 1, puis 16, puis 2...).

**Explication :** le Service `postgres` répartit les connexions entre les 3 Pods. Chaque Pod a **son
propre volume, vierge**, et sa propre base : ce sont 3 PostgreSQL indépendants qui s'ignorent. Le
StatefulSet gère l'**infrastructure** (identités stables, volumes dédiés, ordre de démarrage), mais
il ne connaît pas le métier de PostgreSQL : il ne configure ni la réplication, ni le choix du
primaire, ni la bascule en cas de panne. Ce travail est celui d'un **Opérateur** (CloudNativePG par
exemple, que vous déploierez au Lab 13), qui encode cette connaissance métier.

```bash
kubectl scale statefulset postgres --replicas=1
kubectl delete pvc data-postgres-1 data-postgres-2      # le scale down ne supprime pas les PVC
```

</details>

---

## Partie 6 : Bonus - sauvegarder un volume par snapshot

Les disques Persistent Disk savent prendre des **snapshots** (instantanés), et Kubernetes les pilote
par l'API standard `VolumeSnapshot`.

**6.1 - La classe de snapshot**

Vérifiez que les CRD de snapshots sont présentes, puis créez une `VolumeSnapshotClass` nommée
`pd-snapshot`, pour le pilote CSI de GKE, avec `deletionPolicy: Delete`.

<details><summary>Correction</summary>

```bash
kubectl get crd | grep snapshot.storage.k8s.io
kubectl get volumesnapshotclass
```

```yaml
# snapshotclass.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: pd-snapshot
driver: pd.csi.storage.gke.io
deletionPolicy: Delete
```

```bash
kubectl apply -f snapshotclass.yaml
```

</details>

**6.2 - Prendre un snapshot de la base**

Notez la valeur du compteur, puis créez un `VolumeSnapshot` nommé `postgres-snap-1` du PVC
`data-postgres-0`. Attendez qu'il soit prêt (`READYTOUSE`). Où le retrouvez-vous dans Google Cloud ?

<details><summary>Correction</summary>

```yaml
# snapshot.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-snap-1
spec:
  volumeSnapshotClassName: pd-snapshot
  source:
    persistentVolumeClaimName: data-postgres-0
```

```bash
kubectl apply -f snapshot.yaml
kubectl get volumesnapshot -w                       # READYTOUSE : true
gcloud compute snapshots list --filter="name~snapshot-"
```

Un snapshot de disque n'est cohérent que pour ce qui a été écrit sur le disque : pour une base de
données en production, on préfère les sauvegardes de la base elle-même (`pg_dump`, Lab 05) ou un
Opérateur (Lab 13), éventuellement complétées par des snapshots.

</details>

**6.3 - Restaurer dans un nouveau volume**

Créez un PVC `data-restore` de 10Gi **à partir du snapshot** (champ `dataSource`), puis un Pod
busybox qui le monte dans `/data` et liste le contenu de `/data/pgdata`. Retrouvez-vous les fichiers
de PostgreSQL ?

<details><summary>Correction</summary>

```yaml
# pvc-restore.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-restore
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 10Gi
  dataSource:
    apiGroup: snapshot.storage.k8s.io
    kind: VolumeSnapshot
    name: postgres-snap-1
```

```bash
kubectl apply -f pvc-restore.yaml
sed 's/pvc-demo/data-restore/; s/name: pod-pv/name: pod-restore/' pod-pv.yaml | kubectl apply -f -
kubectl exec pod-restore -- ls /data/pgdata | head      # PG_VERSION, base, global...
```

Le volume restauré contient les fichiers de la base au moment du snapshot. Pour revenir réellement
en arrière, on ferait démarrer PostgreSQL sur ce volume.

🧹

```bash
kubectl delete pod pod-restore; kubectl delete pvc data-restore
kubectl delete volumesnapshot postgres-snap-1         # deletionPolicy Delete : le snapshot Google Cloud est supprimé
```

🏠 **En local** - les snapshots dépendent du pilote de stockage installé : Longhorn ou Ceph les
supportent, un `hostPath` non.

</details>

---

## 🧹 Nettoyage

```bash
kubectl delete sc sc-ssd-alice
kubectl get pv                                     # seul le volume de data-postgres-0 doit rester
```

## ✅ État attendu en fin de lab

StatefulSet `postgres` à **1 réplica** avec le PVC `data-postgres-0` ; Services `postgres` et
`postgres-headless` ; l'application fonctionne et le compteur survit à la suppression de
`postgres-0`. Le fichier `postgres-deployment.yaml` n'est plus utilisé : conservez
`postgres-statefulset.yaml`.

---

## Points clés

* PV = global ; PVC = namespacé. Liaison : même classe, modes d'accès compatibles, capacité ≥ demande.
* `storageClassName: ""` = pas de provisionnement dynamique ; champ absent = classe **par défaut**.
* `ReadWriteOnce` se compte **par nœud**.
* Un `hostPath` attache la donnée à **un nœud** : à proscrire pour des données applicatives. Sur GKE,
  un volume dynamique est un **disque persistant** : il suit le Pod d'un nœud à l'autre, mais **dans
  sa zone** (`nodeAffinity` du PV).
* `reclaimPolicy: Delete` (défaut sur GKE) détruit le disque avec le PVC ; `Retain` le conserve.
* Classes GKE : `standard-rwo` (défaut, `pd-balanced`), `premium-rwo` (`pd-ssd`). Pas de RWX avec
  Persistent Disk : Filestore pour cela.
* `WaitForFirstConsumer` : un PVC `Pending` sans Pod est **normal**.
* Diagnostic : Pod `Pending` → `describe pvc` → StorageClass → pilote CSI.
* `VolumeSnapshot` + `dataSource` : instantané et restauration d'un volume.
* StatefulSet : `serviceName` vers un Service headless, `volumeClaimTemplates` = un PVC par réplica.
  Supprimer le StatefulSet ou réduire ses réplicas **ne supprime pas** les PVC.
