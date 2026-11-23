# LAB 02 - Pods, namespaces et labels

> **Cluster : GKE** (contexte `cluster-lab-dkdc`) | **Durée : ~40 min**
> **Prérequis :** Labs 00 et 01 (alias `k`, variable `$do`).

Le **Pod** est la plus petite unité déployable de Kubernetes. On ne déploie jamais un conteneur
seul : on déploie un Pod, qui contient un ou plusieurs conteneurs partageant la même IP et les
mêmes volumes. Ce lab pose les bases manipulées dans tous les labs suivants, ainsi que la
**méthode de diagnostic** que vous réutiliserez jusqu'à la fin de la formation.

## Objectifs

1. Cloisonner avec les **namespaces** et fixer un namespace par défaut.
2. Créer un **Pod** en mode impératif puis déclaratif, et l'inspecter (`describe`, `logs`, `exec`).
3. Constater le réseau "à plat" de Kubernetes.
4. Organiser et filtrer les objets avec les **labels**, **selectors** et **annotations**.
5. Appliquer la méthode de diagnostic sur trois Pods en erreur.

**Documentation utile :** [Pods](https://kubernetes.io/docs/concepts/workloads/pods/) · [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) · [Labels et selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) · [Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/) · [Débugger un Pod](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)

**Images utilisées :** `public.ecr.aws/wizetraining/nginx:latest` et `public.ecr.aws/wizetraining/busybox:latest`.

```bash
kubectl config use-context cluster-lab-dkdc
```

---

## Partie 1 : Pods et namespaces en mode impératif

L'approche **impérative** consiste à donner des ordres directs en ligne de commande (`run`,
`create`, `delete`). C'est la plus rapide pour tester et pour dépanner.

**1.1 - Lancer un serveur Web**

Lancez un Pod nommé `webserver`, image `public.ecr.aws/wizetraining/nginx:latest`, dans le
namespace `default`. Relevez ensuite :

* son adresse IP ;
* le nœud sur lequel il a été placé ;
* son état et les événements de son démarrage (commande `describe`).

<details><summary>Correction</summary>

```bash
kubectl run webserver --image=public.ecr.aws/wizetraining/nginx:latest
kubectl get pods -o wide          # colonnes IP et NODE
kubectl describe pod webserver    # section Events en bas : Scheduled, Pulling, Pulled, Created, Started
```

La colonne `NODE` est la décision du **scheduler**, la colonne `IP` est attribuée par le **plugin
réseau** (CNI). Ni l'une ni l'autre ne sont choisies par vous.

☁️ **Sur GKE** - le cluster est « VPC natif » : l'IP du Pod est prise dans une plage d'adresses du
réseau VPC de Google Cloud réservée à chaque nœud. Elle est donc routable dans tout le VPC, par
exemple depuis une VM Compute Engine du même réseau.

🏠 **En local** - le plugin réseau (Cilium, Calico, Flannel...) distribue des IP prises dans une
plage interne au cluster, invisible du reste du réseau.

</details>

**1.2 - Isolation par namespace et unicité des noms**

1. Créez un namespace `k8s-lab`.
2. Lancez-y un Pod nommé `tools`, image `public.ecr.aws/wizetraining/busybox:latest`, qui exécute
   la commande `sleep 1000` (sans commande, busybox s'arrête immédiatement).
3. Tentez de créer un **second** Pod nommé `tools` dans ce même namespace. Que se passe-t-il ?
   Et dans le namespace `default` ?

<details><summary>Correction</summary>

```bash
kubectl create ns k8s-lab
kubectl run tools --image=public.ecr.aws/wizetraining/busybox:latest -n k8s-lab -- sleep 1000

# Doublon dans le même namespace : échec
kubectl run tools --image=public.ecr.aws/wizetraining/busybox:latest -n k8s-lab -- sleep 1000
# Error from server (AlreadyExists): pods "tools" already exists

# Même nom dans un autre namespace : accepté
kubectl run tools --image=public.ecr.aws/wizetraining/busybox:latest -n default -- sleep 1000
kubectl delete pod tools -n default
```

Le nom d'un objet est unique **par type et par namespace**.

🔎 **Attention** - le `--` sépare les options de `kubectl` de la commande passée au conteneur.
Tout ce qui suit `--` est envoyé tel quel au conteneur.

</details>

**1.3 - Fixer un namespace par défaut**

Faites de `k8s-lab` votre namespace courant, de façon à ne plus avoir à taper `-n k8s-lab`.
Vérifiez avec `kubectl get pods` sans option.

<details><summary>Correction</summary>

```bash
kubectl config set-context --current --namespace=k8s-lab
kubectl config view --minify | grep namespace
kubectl get pods          # affiche "tools"
```

Pensez à revenir à `default` à la fin du lab : une commande lancée dans le mauvais namespace est
l'une des erreurs les plus fréquentes, et les plus déroutantes (« mon Pod a disparu »).

</details>

**1.4 - Communication inter-Pods**

Depuis le Pod `tools` (namespace `k8s-lab`), interrogez en HTTP (`wget`) l'IP du Pod `webserver`
(namespace `default`), puis essayez un `ping`. L'isolation **administrative** par namespace
bloque-t-elle la communication **réseau** ?

<details><summary>Correction</summary>

```bash
IP=$(kubectl get pod webserver -n default -o jsonpath='{.status.podIP}')
kubectl exec -it tools -n k8s-lab -- wget -qO- $IP | head -5
kubectl exec -it tools -n k8s-lab -- ping -c 3 $IP
```

**La requête passe.** Le modèle réseau de Kubernetes impose que tout Pod puisse joindre tout autre
Pod, **sans NAT**, quel que soit son nœud ou son namespace. Le namespace est une frontière
administrative (noms, droits, quotas), **pas** une frontière réseau. Pour cloisonner le réseau,
il faut des **NetworkPolicies** (sur GKE, elles nécessitent l'option *Network Policy* ou GKE
Dataplane V2 sur le cluster).

Si `ping` échoue alors que `wget` fonctionne, ce n'est pas un problème de réseau : l'envoi de
paquets ICMP demande au conteneur la capacité Linux `NET_RAW`, que certaines configurations
retirent par sécurité.

</details>

---

## Partie 2 : Approche déclarative (YAML)

C'est la méthode de production : l'état désiré est décrit dans un fichier, versionné dans Git, et
appliqué par `kubectl apply`.

**2.1 - Générer puis appliquer un manifeste**

Générez, **sans créer le Pod**, le fichier `pod-tools2.yaml` décrivant un Pod `tools2` (image
busybox ECR, commande `sleep 1500`, namespace `k8s-lab`). Lisez-le, puis appliquez-le.

<details><summary>Correction</summary>

```bash
kubectl run tools2 --image=public.ecr.aws/wizetraining/busybox:latest -n k8s-lab $do -- sleep 1500 > pod-tools2.yaml
cat pod-tools2.yaml
kubectl apply -f pod-tools2.yaml
kubectl get pods -n k8s-lab
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: tools2
  name: tools2
  namespace: k8s-lab
spec:
  containers:
  - args:
    - sleep
    - "1500"
    image: public.ecr.aws/wizetraining/busybox:latest
    name: tools2
```

💡 **Astuce** - n'écrivez jamais un manifeste de zéro : `--dry-run=client -o yaml` (la variable
`$do` du Lab 00) produit un squelette valide en une seconde.

</details>

**2.2 - Un Pod est quasi-immuable**

Essayez de modifier la commande du Pod `tools2` (`kubectl edit pod tools2`, changez `1500` en
`2000`). Que répond l'API ? Comment appliquer quand même ce changement ?

<details><summary>Correction</summary>

```text
error: pods "tools2" is invalid ... spec: Forbidden: pod updates may not change fields other than
`spec.containers[*].image`, `spec.initContainers[*].image`, `spec.activeDeadlineSeconds`,
`spec.tolerations` (only additions to existing tolerations) ...
```

Sur un Pod existant, seuls quelques champs sont modifiables (l'image notamment). Pour le reste,
il faut **recréer** le Pod :

```bash
kubectl replace --force -f pod-tools2.yaml    # supprime puis recrée
```

C'est précisément pour cette raison qu'on n'exploite pas de Pods "nus" en production, mais des
**Deployments** (Lab 03), qui savent remplacer leurs Pods pour vous.

</details>

---

## Partie 3 : Labels, selectors et annotations

Les **labels** sont des paires clé/valeur attachées aux objets ; les **selectors** les filtrent.
Tout Kubernetes repose dessus : un Service choisit ses Pods par selector, un Deployment reconnaît
ses Pods par selector, une NetworkPolicy désigne ses cibles par selector.

**3.1 - Labelliser un namespace et des Pods**

1. Créez un namespace `dev` et ajoutez-lui le label `formation=k8s`.
2. Écrivez un fichier `nginx-labels.yaml` qui crée un Pod `nginx1` dans `dev`, image nginx ECR,
   avec le label `app=frontend`. Appliquez-le.
3. Ajoutez au Pod `nginx1`, **en ligne de commande**, le label `rel=beta`.
4. Lancez en une seule commande un Pod `nginx2` dans `dev` avec les labels `app=frontend` et
   `rel=stable`.

<details><summary>Correction</summary>

```bash
kubectl create ns dev
kubectl label ns dev formation=k8s
kubectl get ns dev --show-labels
```

```yaml
# nginx-labels.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx1
  namespace: dev
  labels:
    app: frontend
spec:
  containers:
  - name: nginx
    image: public.ecr.aws/wizetraining/nginx:latest
```

```bash
kubectl apply -f nginx-labels.yaml
kubectl label pod nginx1 -n dev rel=beta
kubectl run nginx2 --image=public.ecr.aws/wizetraining/nginx:latest -n dev --labels="app=frontend,rel=stable"
```

Pour **modifier** un label existant, il faut `--overwrite` ; pour le **supprimer**, on suffixe la
clé d'un tiret :

```bash
kubectl label pod nginx1 -n dev rel=alpha --overwrite
kubectl label pod nginx1 -n dev rel-
kubectl label pod nginx1 -n dev rel=beta
```

Notez aussi que chaque namespace porte automatiquement le label
`kubernetes.io/metadata.name=<nom>`.

</details>

**3.2 - Filtrer avec les selectors**

Dans le namespace `dev` :

* affichez les Pods avec les colonnes `app` et `rel` ;
* listez uniquement les Pods `rel=stable` ;
* listez les Pods qui **possèdent** la clé `rel`, puis ceux qui **ne la possèdent pas** ;
* listez les Pods dont `rel` vaut `beta` **ou** `stable`.

<details><summary>Correction</summary>

```bash
kubectl get po -n dev -L app,rel
kubectl get po -n dev -l rel=stable
kubectl get po -n dev -l rel                     # présence de la clé
kubectl get po -n dev -l '!rel'                  # absence : quotes obligatoires pour le shell
kubectl get po -n dev -l 'rel in (beta,stable)'
kubectl get po -n dev -l app=frontend,rel=beta   # la virgule est un ET logique
```

🔎 **Attention** - il n'existe **pas de OU** entre deux clés différentes : `a=1,b=2` signifie
« a=1 **et** b=2 ». Le seul OU possible est `in (...)` sur une même clé.

</details>

**3.3 - Annotations**

Ajoutez au Pod `nginx1` l'annotation `wizetraining.com/equipe-responsable="les K8S dev"`, puis
retrouvez-la. Peut-on filtrer des Pods sur une annotation avec `-l` ?

<details><summary>Correction</summary>

```bash
kubectl annotate pod nginx1 -n dev wizetraining.com/equipe-responsable="les K8S dev"
kubectl describe pod nginx1 -n dev | grep -A2 Annotations
```

Non : une annotation ne se sélectionne pas.

| | Label | Annotation |
| --- | --- | --- |
| Sert à | **sélectionner** des objets | porter des **métadonnées** (contact, outil, configuration d'un contrôleur) |
| Filtrable avec `-l` | oui | non |
| Taille de la valeur | 63 caractères max | jusqu'à 256 Ko |

</details>

**3.4 - Placer un Pod grâce à un label de nœud**

1. Ajoutez le label `disk=ssd` sur **un seul** de vos nœuds.
2. Écrivez `pod-ssd.yaml` : un Pod `pod-ssd` dans `dev`, image nginx ECR, label `app=backend`,
   qui doit **obligatoirement** s'exécuter sur un nœud portant `disk=ssd` (champ `nodeSelector`).
3. Vérifiez sur quel nœud il tourne.

<details><summary>Correction</summary>

```bash
NODE=$(kubectl get nodes -o jsonpath='{.items[1].metadata.name}')
kubectl label node $NODE disk=ssd
kubectl get nodes -L disk
```

```yaml
# pod-ssd.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-ssd
  namespace: dev
  labels:
    app: backend
spec:
  nodeSelector:
    disk: "ssd"
  containers:
  - name: nginx
    image: public.ecr.aws/wizetraining/nginx:latest
```

```bash
kubectl apply -f pod-ssd.yaml
kubectl get pod pod-ssd -n dev -o wide      # NODE = le nœud étiqueté
```

Vous approfondirez le placement (taints, affinités) au Lab 10.

☁️ **Sur GKE** - un label posé à la main sur un nœud est **perdu** si ce nœud est recréé (mise à
jour automatique, réparation, réduction puis agrandissement du node pool). Pour des labels
durables, on les déclare dans la configuration du **node pool** (dans la console, à la création du
pool) : tous les nœuds du pool, présents et futurs, les portent.

🏠 **En local** - les nœuds sont des machines durables : un label posé avec `kubectl label` reste
en place.

</details>

---

## Partie 4 : 🩺 Diagnostic - la méthode

Toute panne applicative se diagnostique dans le **même ordre**. Retenez-le, vous l'appliquerez
dans chacun des labs suivants.

```text
1. kubectl get pods -o wide        → quel est l'ÉTAT ? sur quel nœud ? combien de RESTARTS ?
2. kubectl describe pod <nom>      → section EVENTS, en bas : la cause est là dans 80 % des cas
3. kubectl logs <nom>              → qu'a écrit l'application ?
4. kubectl logs <nom> --previous   → et avant le dernier redémarrage ?
5. kubectl exec -it <nom> -- sh    → entrer voir (fichiers, réseau, variables)
```

Créez les trois Pods suivants dans le namespace `dev`, **sans regarder la correction** :

```bash
kubectl run panne-1 -n dev --image=public.ecr.aws/wizetraining/nginx:latest \
  --overrides='{"spec":{"nodeSelector":{"disk":"nvme"}}}'
kubectl run panne-2 -n dev --image=public.ecr.aws/wizetraining/ngnix:latest
kubectl run panne-3 -n dev --image=public.ecr.aws/wizetraining/busybox:latest
```

**4.1** - Pour chacun des trois Pods : quel est son état, quelle est la cause exacte, et quelle
commande vous l'a révélée ?

**4.2** - Corrigez `panne-2` sans le supprimer (rappel : l'image est l'un des rares champs
modifiables d'un Pod).

**4.3** - Pour `panne-3`, affichez les logs de l'instance **précédente** du conteneur et son code
de sortie. Le conteneur a-t-il vraiment « planté » ?

<details><summary>Correction</summary>

```bash
kubectl get pods -n dev
```

```text
NAME      READY   STATUS             RESTARTS
panne-1   0/1     Pending            0
panne-2   0/1     ImagePullBackOff   0
panne-3   0/1     CrashLoopBackOff   3
```

**`panne-1` - Pending**

```bash
kubectl describe pod panne-1 -n dev | tail -5
# FailedScheduling ... 0/3 nodes are available: ... didn't match Pod's node affinity/selector
```

`Pending` = **le scheduler n'a trouvé aucun nœud**. Aucun nœud ne porte `disk=nvme`.
Les autres causes classiques de `Pending` : ressources insuffisantes, taint non tolérée, volume
non disponible (vous les rencontrerez aux Labs 06 et 10).

**`panne-2` - ImagePullBackOff**

```bash
kubectl describe pod panne-2 -n dev | tail -5
# Failed to pull image "public.ecr.aws/wizetraining/ngnix:latest": ... not found
kubectl set image pod/panne-2 -n dev panne-2=public.ecr.aws/wizetraining/nginx:latest
kubectl get pod panne-2 -n dev -w
```

Faute de frappe dans le nom de l'image (`ngnix`). Les autres causes : tag inexistant, registre
privé sans identifiants, limitation de débit du registre.

**`panne-3` - CrashLoopBackOff**

```bash
kubectl logs panne-3 -n dev --previous
kubectl describe pod panne-3 -n dev | grep -A4 "Last State"
# Reason: Completed   Exit Code: 0
```

Le conteneur n'a **pas planté** : busybox sans commande exécute un shell qui se termine aussitôt,
avec un code de sortie `0`. Mais un Pod a par défaut `restartPolicy: Always` : Kubernetes le
relance en boucle, en espaçant les tentatives (10 s, 20 s, 40 s... jusqu'à 5 min). C'est ce
qu'indique `CrashLoopBackOff`.

| Code de sortie | Signification |
| --- | --- |
| `0` | le conteneur a terminé son travail normalement |
| `1` | erreur applicative |
| `127` | commande introuvable |
| `137` | tué par SIGKILL - souvent un dépassement mémoire (OOMKilled) |
| `143` | arrêt propre par SIGTERM |

🔎 **Attention** - sur un Pod qui redémarre en boucle, `kubectl logs` montre l'instance
**actuelle**, souvent vide car elle vient de démarrer. **`--previous`** est indispensable.

💡 **Astuce** - pour voir d'un coup d'œil ce qui se passe dans un namespace :

```bash
kubectl get events -n dev --sort-by=.lastTimestamp | tail -20
```

</details>

---

## 🧹 Nettoyage

```bash
kubectl delete pod pod-ssd nginx1 nginx2 panne-1 panne-2 panne-3 -n dev
# Suppression de masse par selector (ce qu'il resterait avec app=frontend)
kubectl delete po -n dev -l app=frontend
kubectl delete pod webserver -n default
kubectl label nodes --all disk-
kubectl config set-context --current --namespace=default
```

Conservez les namespaces `dev` et `k8s-lab` : ils ne gênent pas la suite.

---

## Points clés

* Le namespace cloisonne les **noms**, les **droits** et les **quotas**, jamais le **réseau**.
* `kubectl config set-context --current --namespace=<ns>` pour changer de namespace de travail.
* `--dry-run=client -o yaml` pour générer, `kubectl apply -f` pour appliquer.
* Un Pod est quasi-immuable : seule l'image (et quelques champs) se modifie à chaud.
* Les selectors ne connaissent que le **ET** ; le OU n'existe que via `in (...)`.
* Sur GKE, les labels durables de nœuds se déclarent sur le **node pool**.
* Méthode de diagnostic : `get` → **`describe` (Events)** → `logs` → **`logs --previous`** → `exec`.
* `Pending` = pas de nœud. `ImagePullBackOff` = image introuvable. `CrashLoopBackOff` = le
  conteneur sort en boucle : lisez le **code de sortie**.
