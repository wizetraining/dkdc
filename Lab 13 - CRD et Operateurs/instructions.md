# LAB 13 - CRD et Opérateurs

> **Cluster : GKE** (contexte `cluster-lab-dkdc`) | **Durée : ~55 min**
> **Prérequis :** Labs 05, 06 et 09 terminés (Secret `pg-secret` de type basic-auth, CronJob
> `pg-backup`, StatefulSet `postgres`, `helm` disponible). Dossier `~/labs/countvisit`.

Le Lab 06 s'est terminé sur un constat : un StatefulSet sait créer trois Pods PostgreSQL avec trois
disques, mais il ne sait pas les **relier** (réplication, primaire, bascule). Ce savoir-faire de DBA
peut être encodé dans un **Opérateur**. Ce lab construit la notion en deux temps : étendre l'API
avec une **CRD**, puis confier PostgreSQL à l'Opérateur **CloudNativePG**.

Vous avez déjà croisé des CRD sans le savoir : `VolumeSnapshot` (Lab 06), `VerticalPodAutoscaler`
(Lab 11), et les ressources propres à GKE vues au Lab 01. GKE en installe des dizaines
(`kubectl get crd | wc -l`).

## Objectifs

1. Comprendre qu'un **Opérateur** = **CRD** (un nouveau type d'objet) + **contrôleur** (la logique).
2. Installer CloudNativePG avec Helm et déployer un cluster PostgreSQL à **3 instances répliquées**.
3. Brancher Countvisit sur le bon Service, provoquer la **panne du primaire** et constater la bascule.

**Documentation utile :** [CustomResourceDefinitions](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) · [Le pattern Opérateur](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) · [CloudNativePG](https://cloudnative-pg.io/documentation/current/) · [CNPG - bootstrap](https://cloudnative-pg.io/documentation/current/bootstrap/) · [ECK - Elastic Cloud on Kubernetes](https://www.elastic.co/docs/deploy-manage/deploy/cloud-on-k8s)

> Les images de l'Opérateur et de PostgreSQL de ce lab viennent du registre de CloudNativePG
> (`ghcr.io`), et non d'ECR.

---

## Partie 1 : Étendre l'API avec une CRD

L'API Server est extensible : une **CustomResourceDefinition** lui apprend un nouveau type d'objet,
qui est ensuite traité comme un objet natif (`kubectl get`, RBAC, labels, `watch`...).

**Scénario :** une plateforme de formation veut gérer des objets `Course`. Groupe
`stable.k8s.wizetraining.io`, version `v1`, portée *namespaced*, avec `spec.title` (texte) et
`spec.durationHours` (entier), nom court `crs`.

**1.1 - Avant la CRD**

Lancez `kubectl get courses`. Quelle erreur obtenez-vous ?

<details><summary>Correction</summary>

```bash
kubectl get courses
# error: the server doesn't have a resource type "courses"
```

</details>

**1.2 - Déclarer le type**

Écrivez `course-crd.yaml` (`apiVersion: apiextensions.k8s.io/v1`, `kind: CustomResourceDefinition`).
Attention : le **nom** de la CRD doit être `<pluriel>.<groupe>`. Appliquez-le, puis retrouvez le
nouveau type dans `kubectl api-resources` et sa documentation dans `kubectl explain`.

<details><summary>Correction</summary>

```yaml
# course-crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: courses.stable.k8s.wizetraining.io     # <pluriel>.<groupe> : obligatoire
spec:
  group: stable.k8s.wizetraining.io
  scope: Namespaced
  names:
    plural: courses
    singular: course
    kind: Course
    shortNames: [crs]
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              title:
                type: string
              durationHours:
                type: integer
```

```bash
kubectl apply -f course-crd.yaml
kubectl api-resources | grep course
kubectl explain course.spec            # la documentation est générée à partir du schéma
```

</details>

**1.3 - Créer et valider un objet**

Créez un objet `Course` nommé `kubernetes-fundamentals` (titre et durée de votre choix), modifiez sa
durée avec `kubectl patch`, puis listez-le avec le nom court. Tentez ensuite une durée en texte
(`"beaucoup"`) : que se passe-t-il ?

<details><summary>Correction</summary>

```yaml
# my-course.yaml
apiVersion: stable.k8s.wizetraining.io/v1
kind: Course
metadata:
  name: kubernetes-fundamentals
spec:
  title: "Les fondamentaux de Kubernetes"
  durationHours: 14
```

```bash
kubectl apply -f my-course.yaml
kubectl patch course kubernetes-fundamentals --type=merge -p '{"spec":{"durationHours": 16}}'
kubectl get crs
kubectl patch course kubernetes-fundamentals --type=merge -p '{"spec":{"durationHours": "beaucoup"}}'
# The Course "kubernetes-fundamentals" is invalid: spec.durationHours: Invalid value: "string": ...must be of type integer
```

Le schéma OpenAPI de la CRD est appliqué par l'API Server : un objet invalide est refusé avant même
d'être stocké.

</details>

**1.4 - Bonus : des colonnes utiles**

`kubectl get crs` n'affiche que `NAME` et `AGE`. Ajoutez à la version `v1` de la CRD des
`additionalPrinterColumns` pour afficher le titre (`.spec.title`) et la durée (`.spec.durationHours`),
réappliquez la CRD, puis relancez `kubectl get crs`.

<details><summary>Correction</summary>

```yaml
# course-crd.yaml : dans spec.versions[0], au même niveau que schema
    additionalPrinterColumns:
    - name: Titre
      type: string
      jsonPath: .spec.title
    - name: Durée
      type: integer
      jsonPath: .spec.durationHours
```

```bash
kubectl apply -f course-crd.yaml
kubectl get crs
# NAME                      TITRE                            DURÉE
# kubernetes-fundamentals   Les fondamentaux de Kubernetes   16
```

C'est ainsi que `kubectl get cluster` (CloudNativePG) ou `kubectl get elasticsearch` (ECK) affichent
l'état de santé et la version : ces colonnes sont déclarées dans leur CRD.

</details>

**1.5 - Et ensuite ?**

Que fait Kubernetes, concrètement, de cet objet `Course` ? Que manque-t-il pour qu'il se passe
quelque chose (créer un namespace par formation, par exemple) ?

<details><summary>Correction</summary>

Kubernetes se contente de **valider et stocker** l'objet dans etcd : ce n'est que de la donnée. Rien
ne se passe tant qu'aucun **contrôleur** ne surveille ces objets pour agir.

```text
Opérateur  =  CRD (le QUOI : un nouveau type d'objet)  +  contrôleur (le COMMENT : la logique métier)
```

🧹 `kubectl delete -f my-course.yaml -f course-crd.yaml`

</details>

---

## Partie 2 : Installer et observer l'Opérateur CloudNativePG

**2.1 - L'installation**

Installez l'Opérateur avec Helm (Lab 09) : ajoutez le dépôt `cnpg`
(`https://cloudnative-pg.github.io/charts`), mettez-le à jour, puis installez le chart
`cnpg/cloudnative-pg` sous le nom de release `cnpg-operator`, dans le namespace `cnpg-system` créé
au passage. Attendez que le Pod de l'Opérateur soit `Running`.

<details><summary>Correction</summary>

```bash
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo update
helm upgrade --install cnpg-operator cnpg/cloudnative-pg \
  --namespace cnpg-system --create-namespace

kubectl get pods -n cnpg-system
```

`helm upgrade --install` installe si la release n'existe pas, et la met à jour sinon : une commande
idempotente, idéale pour l'automatisation.

</details>

**2.2 - Ce que l'Opérateur a apporté**

Listez les CRD apportées par CloudNativePG et les ressources du groupe d'API `postgresql.cnpg.io`.
Quelle ressource allez-vous manipuler pour obtenir un cluster PostgreSQL ? Retrouvez avec
`kubectl explain` le rôle de son champ `spec.instances`.

<details><summary>Correction</summary>

```bash
kubectl get crd | grep cnpg
kubectl api-resources --api-group=postgresql.cnpg.io     # clusters, backups, scheduledbackups, poolers...
kubectl explain clusters.postgresql.cnpg.io.spec.instances
```

La ressource principale est `Cluster`. Le contrôleur (le Pod de `cnpg-system`) surveille les objets
`Cluster` de **tous** les namespaces : pour chacun, il crée et maintient Pods, PVC, Services et
Secrets, et configure la réplication PostgreSQL.

> **Pour les curieux :** c'est exactement le modèle d'**ECK** (Elastic Cloud on Kubernetes) : des CRD
> `Elasticsearch`, `Kibana`, `Agent`... et un Opérateur qui gère leur cycle de vie (déploiement,
> montée de version, ajout de nœuds). [Documentation ECK](https://www.elastic.co/docs/deploy-manage/deploy/cloud-on-k8s)

</details>

---

## Partie 3 : Un cluster PostgreSQL à 3 instances

**3.1 - Décrire le cluster souhaité**

À l'aide de la [documentation de bootstrap](https://cloudnative-pg.io/documentation/current/bootstrap/),
écrivez `~/labs/countvisit/postgres-cluster.yaml` : un `Cluster` nommé `pg-database` dans
`countvisit`, avec :

* 3 instances (1 primaire + 2 réplicas) ;
* l'image `ghcr.io/cloudnative-pg/postgresql:16` ;
* un stockage de 10Gi par instance, dans la classe `standard-rwo` ;
* une initialisation (`bootstrap.initdb`) créant la base `app_db`, propriétaire `app_user`, dont le
  mot de passe est fourni par votre Secret `pg-secret`.

Pourquoi le Secret `pg-secret` du Lab 05 convient-il tel quel ?

<details><summary>Correction</summary>

```yaml
# postgres-cluster.yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-database
  namespace: countvisit
spec:
  instances: 3
  imageName: ghcr.io/cloudnative-pg/postgresql:16
  storage:
    size: 10Gi
    storageClass: standard-rwo
  bootstrap:
    initdb:
      database: app_db
      owner: app_user
      secret:
        name: pg-secret          # type basic-auth : username + password
```

CloudNativePG attend un Secret de type `kubernetes.io/basic-auth` dont le `username` correspond au
propriétaire : c'est précisément ce que vous avez créé au Lab 05.

</details>

**3.2 - Observer l'Opérateur au travail**

Appliquez le manifeste et observez les Pods en continu. Dans quel ordre apparaissent-ils ? Combien de
PVC ont été créés ? Quel est l'état global du cluster (`kubectl get cluster`) ?

<details><summary>Correction</summary>

```bash
cd ~/labs/countvisit
kubectl apply -f postgres-cluster.yaml
kubectl get pods -w -l cnpg.io/cluster=pg-database
# pg-database-1-initdb-xxxx (Job d'initialisation), puis pg-database-1, -2, -3, un par un
kubectl get pvc -l cnpg.io/cluster=pg-database       # 1 PVC par instance
kubectl get cluster pg-database                      # STATUS : Cluster in healthy state, READY 3
```

Les réplicas sont créés **un par un**, chacun cloné depuis le primaire puis maintenu à jour par
réplication en flux.

```bash
kubectl get pods -l cnpg.io/cluster=pg-database -o wide     # un Pod par nœud
kubectl get pods -l cnpg.io/cluster=pg-database   -o custom-columns=POD:.metadata.name,NOEUD:.spec.nodeName
kubectl get nodes -L topology.kubernetes.io/zone            # donc un Pod par zone
```

☁️ **Sur GKE** - CloudNativePG répartit les instances sur des **nœuds différents** (anti-affinité,
Lab 10). Votre cluster étant **régional**, chaque instance est dans une **zone** différente, avec son
propre disque. Comme un disque persistant est **zonal** (Lab 06), c'est la **réplication au niveau de
la base** qui permet de survivre à la perte d'une zone entière.

</details>

**3.3 - Les Services**

Listez les Services générés pour ce cluster. À quoi sert chacun ? Lequel pointe vers quel Pod ?
Retrouvez le Pod primaire grâce au label `cnpg.io/instanceRole`.

<details><summary>Correction</summary>

```bash
kubectl get svc -l cnpg.io/cluster=pg-database
kubectl get pods -L cnpg.io/instanceRole -l cnpg.io/cluster=pg-database
kubectl get endpointslices -l kubernetes.io/service-name=pg-database-rw
```

| Service | Cible | Usage |
| --- | --- | --- |
| `pg-database-rw` | **toujours le primaire** | lectures et écritures |
| `pg-database-ro` | uniquement les réplicas | lectures seules, pour décharger le primaire |
| `pg-database-r` | n'importe quelle instance | lectures tolérant un léger retard |

</details>

---

## Partie 4 : Brancher Countvisit sur le cluster

**4.1 - Quel Service ?**

Quel Service doit devenir le `PG_HOST` de l'application ? Pourquoi pas le nom d'un Pod
(`pg-database-1`) ?

<details><summary>Correction</summary>

`pg-database-rw`. Un Pod peut **changer de rôle** : après une bascule, `pg-database-1` peut devenir
un réplica en lecture seule. Le Service `-rw`, lui, est repointé par l'Opérateur vers le primaire
du moment.

</details>

**4.2 - La bascule de l'application**

Mettez à jour la clé `PG_HOST` du ConfigMap `webapp-config` (fichier `webapp-configmap.yaml`), puis
faites-la prendre en compte par les Pods (rappel du Lab 05 : une variable d'environnement n'est lue
qu'au démarrage). Vérifiez dans le navigateur : que devient le compteur ?

<details><summary>Correction</summary>

```bash
# webapp-configmap.yaml : PG_HOST: pg-database-rw
kubectl apply -f webapp-configmap.yaml
kubectl rollout restart deployment/webapp
kubectl rollout status deployment/webapp
kubectl exec deploy/webapp -- printenv PG_HOST       # pg-database-rw
```

La page affiche « Connecté » et le compteur **repart à 1** : c'est une **nouvelle** base. Migrer les
données de l'ancienne base aurait demandé un `pg_dump` puis une restauration (ou l'import intégré
de CloudNativePG, `bootstrap.initdb.import`).

</details>

**4.3 - Effets de bord et ménage**

La sauvegarde CronJob `pg-backup` du Lab 05 vise-t-elle l'ancienne ou la nouvelle base ? Vérifiez en
déclenchant une sauvegarde à la demande. Puis supprimez l'ancienne base : le StatefulSet `postgres`,
son PVC, et les Services `postgres` et `postgres-headless`.

<details><summary>Correction</summary>

```bash
kubectl create job --from=cronjob/pg-backup backup-cnpg
kubectl logs job/backup-cnpg --follow | tail -1       # SAUVEGARDE-OK
kubectl delete job backup-cnpg
```

La sauvegarde lit son `PGHOST` dans le **même** ConfigMap : elle a suivi la nouvelle base sans
modification. C'est le bénéfice concret d'une configuration centralisée.

```bash
kubectl delete statefulset postgres
kubectl delete service postgres postgres-headless
kubectl delete pvc data-postgres-0          # supprimer le StatefulSet ne supprime pas ses PVC (Lab 06)
kubectl get pv                              # le volume de l'ancienne base a disparu (classe en Delete)
```

</details>

---

## Partie 5 : La panne du primaire

**5.1 - Provoquer la panne**

Rechargez la page jusqu'à un compteur d'environ 10 et identifiez le Pod primaire. Dans un terminal,
surveillez les Pods du cluster (`-w`, avec la colonne `cnpg.io/instanceRole`). Dans un autre,
**supprimez le Pod primaire**, puis continuez de recharger la page.

Qu'observez-vous : durée de l'interruption, Pod promu, valeur du compteur après la bascule ? À quoi
servent les 5 tentatives de connexion de l'application ?

<details><summary>Correction</summary>

```bash
kubectl get pods -w -L cnpg.io/instanceRole -l cnpg.io/cluster=pg-database     # terminal 1
kubectl delete pod -l cnpg.io/instanceRole=primary,cnpg.io/cluster=pg-database  # terminal 2

kubectl get cluster pg-database          # nouveau "Current Primary"
kubectl get endpointslices -l kubernetes.io/service-name=pg-database-rw   # repointé
```

Quelques secondes d'attente (le temps de la promotion), puis le compteur reprend **exactement là où
il était** : un réplica, synchronisé en continu, est promu primaire, et le Service `-rw` est repointé
automatiquement. Les tentatives de reconnexion de l'application absorbent la bascule. L'ancien
primaire revient ensuite, comme **réplica**.

</details>

**5.2 - Le bilan**

Que serait-il arrivé avec le StatefulSet du Lab 06 à 3 réplicas ? Complétez le tableau : réplication,
bascule, sauvegardes, montées de version.

<details><summary>Correction</summary>

| | StatefulSet seul | Opérateur CloudNativePG |
| --- | --- | --- |
| Réplication | aucune (3 bases isolées) | automatique (réplication en flux) |
| Bascule | aucune | automatique (promotion d'un réplica) |
| Sauvegardes | à scripter (Job du Lab 05) | natives (objets `Backup`, `ScheduledBackup`, vers un stockage objet comme Cloud Storage) |
| Montées de version | risque de perte de données | progressives, avec bascule contrôlée |

L'Opérateur encode le savoir-faire d'un DBA dans un contrôleur. Le StatefulSet reste la brique
d'infrastructure ; l'Opérateur ajoute le **métier**.

☁️ **Sur GKE** - une alternative consiste à ne pas héberger la base dans le cluster : **Cloud SQL**
ou **AlloyDB** sont des PostgreSQL gérés par Google. L'application s'y connecte avec le même
`PG_HOST`, et l'identité du Pod peut passer par Workload Identity (Lab 12). Le choix entre base
gérée et Opérateur dépend des exigences de portabilité, de coût et d'exploitation.

</details>

---

## ✅ État attendu en fin de lab

* `kubectl get cluster -n countvisit` affiche `pg-database` avec 3 instances saines ;
* le ConfigMap `webapp-config` pointe vers `pg-database-rw`, et le compteur a survécu à la perte du
  primaire ;
* l'ancien StatefulSet `postgres` et son PVC sont supprimés ;
* le fichier `~/labs/countvisit/postgres-cluster.yaml` décrit la base (il resservira aux Labs 14 et 15).

---

## Points clés

* Une **CRD** ajoute un type d'objet à l'API, avec validation par schéma. Son nom est
  `<pluriel>.<groupe>`. Seule, elle ne fait **rien**.
* **Opérateur** = CRD + contrôleur. Il s'installe souvent avec Helm, et surveille ses objets dans
  tous les namespaces.
* `kubectl api-resources --api-group=<groupe>` et `kubectl explain <ressource>` pour découvrir une
  API inconnue.
* Avec CloudNativePG, l'application vise le Service **`-rw`**, repointé vers le primaire après une
  bascule.
* Un Opérateur répartit les instances sur des nœuds (ou des zones) différents et réplique les
  données : c'est ce qui rend une base résiliente, pas le disque persistant seul.
