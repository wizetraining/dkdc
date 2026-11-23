# LAB 11 - Autoscaling : Pods, ressources et nœuds

> **Cluster : GKE** (contexte `cluster-lab-dkdc`) | **Durée : ~50 min**
> **Prérequis :** Lab 10 terminé ; application Countvisit dans `countvisit` (Labs 02 à 08), dossier
> `~/labs/countvisit`.

Au Lab 10, vous avez fixé à la main **combien** chaque Pod réserve, et constaté qu'un Pod trop gros
reste `Pending`. Un cluster de production ne se règle pas à la main : la charge varie dans la
journée. Kubernetes et GKE proposent trois autoscalers complémentaires :

| Autoscaler | Il ajuste... | Il réagit à... | Fourni par |
| --- | --- | --- | --- |
| **HorizontalPodAutoscaler** (HPA) | le **nombre de Pods** d'un Deployment | la consommation (CPU, mémoire, métriques métier) | Kubernetes |
| **VerticalPodAutoscaler** (VPA) | les **requests et limits** des Pods | la consommation passée des conteneurs | projet Kubernetes autoscaler, intégré à GKE |
| **Cluster Autoscaler** | le **nombre de nœuds** d'un node pool | des Pods `Pending` faute de place, des nœuds sous-utilisés | intégré à GKE (présenté en fin de lab) |

Ils s'enchaînent : la charge monte → le HPA ajoute des Pods → plus de place sur les nœuds → les
nouveaux Pods sont `Pending` → le Cluster Autoscaler ajoute un nœud.

```bash
kubectl config set-context --current --namespace=countvisit
cd ~/labs/countvisit
```

## Objectifs

1. Rendre Countvisit élastique avec un **HPA**, et comprendre sa formule et sa stabilisation.
2. Obtenir des **recommandations** de requests avec un **VPA**, puis le laisser les appliquer.
3. Comprendre le rôle du **Cluster Autoscaler**, qui ajuste le nombre de nœuds.
4. Diagnostiquer un HPA `<unknown>` et un conflit entre autoscalers.

**Documentation utile :** [HorizontalPodAutoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/) · [VPA sur GKE](https://cloud.google.com/kubernetes-engine/docs/concepts/verticalpodautoscaler) · [Cluster Autoscaler GKE](https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-autoscaler) · [Autoscaling sur GKE - bonnes pratiques](https://cloud.google.com/kubernetes-engine/docs/concepts/horizontalpodautoscaler)

---

## Partie 1 : HorizontalPodAutoscaler - plus de Pods

Le **HPA** interroge metrics-server toutes les 15 secondes et ajuste le champ `replicas` du
Deployment pour maintenir une utilisation cible.

**1.1 - Le prérequis : des requests**

Le HPA exprime sa cible en **pourcentage de la request CPU**. Ajoutez au conteneur `webapp` de
`webapp-deployment.yaml` des requests `cpu: 50m`, `memory: 64Mi` et des limits `cpu: 250m`,
`memory: 256Mi`. Appliquez et vérifiez la consommation actuelle des Pods.

<details><summary>Correction</summary>

```yaml
# webapp-deployment.yaml (conteneur webapp)
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 250m
            memory: 256Mi
```

```bash
kubectl apply -f webapp-deployment.yaml
kubectl rollout status deployment/webapp
kubectl top pods -l app=webapp
```

</details>

**1.2 - Créer l'autoscaler**

Créez un HPA pour le Deployment `webapp` : **entre 3 et 6 réplicas**, avec une cible de **50 %**
d'utilisation CPU moyenne. Que signifient ces 50 % en millicores ? Que montre la colonne `TARGETS` ?

<details><summary>Correction</summary>

```yaml
# webapp-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 3
  maxReplicas: 6
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

```bash
kubectl apply -f webapp-hpa.yaml
kubectl get hpa webapp           # TARGETS : cpu: 2%/50%   MINPODS 3   MAXPODS 6   REPLICAS 3
```

Équivalent impératif : `kubectl autoscale deployment webapp --min=3 --max=6 --cpu-percent=50`
(selon la version de kubectl, l'option peut aussi s'écrire `--cpu=50%`).

50 % de la request de 50m = **25 millicores** par Pod en moyenne. Au-delà, le HPA ajoute des
Pods selon la formule :

```text
réplicas souhaités = arrondi supérieur( réplicas actuels × utilisation actuelle / utilisation cible )
```

Exemple : 3 Pods à 120 % → `ceil(3 × 120 / 50)` = 8, plafonné à `maxReplicas` = **6**.

🔎 **Attention** - sans `requests.cpu` sur le conteneur, la colonne `TARGETS` affiche
`<unknown>/50%` et le HPA ne fait **rien** : il ne peut pas calculer de pourcentage.

</details>

**1.3 - Générer de la charge**

Dans un second terminal, surveillez le HPA (`kubectl get hpa webapp -w`). Dans le premier, lancez
**deux** Pods `charge-1` et `charge-2` (busybox ECR) qui interrogent le Service `webapp` en boucle,
sans pause. Observez pendant 2 à 3 minutes : comment évoluent `TARGETS` et `REPLICAS` ? Retrouvez
la décision du HPA dans ses événements.

<details><summary>Correction</summary>

```bash
for i in 1 2; do
  kubectl run charge-$i --image=public.ecr.aws/wizetraining/busybox:latest \
    -- sh -c "while true; do wget -qO- http://webapp > /dev/null; done"
done

kubectl get hpa webapp -w            # TARGETS monte au-dessus de 50 %, REPLICAS passe à 4, 5, 6
kubectl describe hpa webapp | tail -5
# SuccessfulRescale  New size: 6; reason: cpu resource utilization (percentage of request) above target
kubectl get pods -l app=webapp       # jusqu'à 6 Pods
```

La montée en charge est rapide : le HPA réagit dès la mesure suivante.

☁️ **Sur GKE** - le HPA peut aussi suivre des métriques **externes** collectées par Cloud Monitoring
(nombre de messages d'une file Pub/Sub, requêtes par seconde d'un équilibreur...), au lieu du seul
CPU.

🏠 **En local** - le HPA fonctionne de la même façon, à condition d'avoir installé metrics-server ;
pour des métriques métier, on ajoute un adaptateur Prometheus ou **KEDA**.

</details>

**1.4 - Arrêter la charge**

Supprimez les deux Pods de charge. Le nombre de réplicas redescend-il immédiatement ? Pourquoi ?

<details><summary>Correction</summary>

```bash
kubectl delete pod charge-1 charge-2
kubectl get hpa webapp -w            # TARGETS retombe vite, REPLICAS reste à 6... pendant ~5 min
```

Non : par défaut, le HPA applique une **fenêtre de stabilisation de 5 minutes** avant de réduire,
pour éviter le *flapping* (création et suppression incessantes de Pods si la charge oscille).
Ce comportement se règle dans `spec.behavior` :

```yaml
spec:
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60     # défaut : 300
```

Vous n'êtes pas obligé d'attendre la fin de la descente pour passer à la question suivante.

</details>

**1.5 - 🩺 Le HPA et le fichier YAML se disputent**

Pendant que le HPA maintient plus de 3 réplicas (relancez un Pod de charge si nécessaire),
ré-appliquez `webapp-deployment.yaml`, qui contient `replicas: 3`. Observez les Pods et le HPA
pendant une minute. Que se passe-t-il ? Comment éviter ce conflit ?

<details><summary>Correction</summary>

```bash
kubectl apply -f webapp-deployment.yaml
kubectl get pods -l app=webapp -w     # des Pods sont supprimés... puis recréés par le HPA
```

Le fichier impose `replicas: 3`, le HPA impose sa propre valeur : chaque `apply` ramène le
Deployment à 3, puis le HPA le corrige au cycle suivant. Les Pods sont supprimés et recréés
pour rien.

**Règle :** quand un HPA gère un Deployment, on **retire le champ `replicas`** du manifeste. C'est
le HPA qui devient la source de vérité du nombre de Pods.

</details>

🧹 Avant de passer au VPA, supprimez le HPA et les éventuels Pods de charge. **Conservez** les
requests et limits (c'est une bonne pratique) et le champ `replicas: 3` dans le fichier.

```bash
kubectl delete hpa webapp
kubectl delete pod charge-1 charge-2 --ignore-not-found
kubectl apply -f webapp-deployment.yaml           # retour à 3 réplicas
```

---

---

## Partie 2 : VerticalPodAutoscaler - des Pods à la bonne taille

Les requests de `webapp` (`50m`, `64Mi`) ont été choisies au jugé. Trop hautes, elles gaspillent
des nœuds payés ; trop basses, elles exposent aux `OOMKilled` et aux nœuds surchargés. Le **VPA**
observe la consommation réelle des conteneurs et **recommande** (ou applique) des requests adaptées.

**2.1 - Le VPA est-il disponible ?**

Retrouvez le type de ressource `VerticalPodAutoscaler` dans l'API. À quel groupe d'API
appartient-il ? Si vous ne le trouvez pas, que s'est-il passé à la création du cluster ?

<details><summary>Correction</summary>

```bash
kubectl api-resources | grep -i verticalpodautoscaler
# verticalpodautoscalers   vpa   autoscaling.k8s.io/v1   true   VerticalPodAutoscaler
```

Il a été activé à la création du cluster (option *Autoscaling vertical des pods*, Lab 00). S'il
manque, l'option peut être activée après coup dans la console (onglet **Détails** du cluster,
section *Automatisation*) : prévenez le formateur.

Le VPA n'est **pas** dans le cœur de Kubernetes : c'est une **CRD** (Lab 13) et des contrôleurs. Sur
GKE, Google les gère pour vous.

🏠 **En local** - on installe le VPA depuis le dépôt `kubernetes/autoscaler` (trois composants :
*recommender*, *updater*, *admission controller*), après metrics-server.

</details>

**2.2 - Mode recommandation**

Écrivez `webapp-vpa.yaml` : un VPA `webapp` qui cible le Deployment `webapp`, avec
`updatePolicy.updateMode: "Off"` (il calcule, mais ne touche à rien). Appliquez-le, générez un peu de
trafic avec **un** Pod de charge (comme en 1.3), attendez 3 à 5 minutes, puis lisez la
recommandation. Comparez-la aux requests actuelles.

<details><summary>Correction</summary>

```yaml
# webapp-vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: webapp
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  updatePolicy:
    updateMode: "Off"
```

```bash
kubectl apply -f webapp-vpa.yaml
kubectl run charge-1 --image=public.ecr.aws/wizetraining/busybox:latest \
  -- sh -c "while true; do wget -qO- http://webapp > /dev/null; sleep 0.2; done"

kubectl get vpa webapp -w              # PROVIDED : True, colonnes CPU et MEM renseignées
kubectl describe vpa webapp | sed -n '/Recommendation/,$p'
```

```text
Recommendation:
  Container Recommendations:
    Container Name:  webapp
    Lower Bound:     Cpu: 25m    Memory: 262144k
    Target:          Cpu: 35m    Memory: 262144k
    Upper Bound:     Cpu: 210m   Memory: 400Mi
```

*(Les valeurs varient selon la charge.)*

| Champ | Signification |
| --- | --- |
| `Target` | la request recommandée |
| `Lower Bound` / `Upper Bound` | l'intervalle dans lequel le VPA laisse une request en place |
| `Uncapped Target` | la recommandation brute, avant application des bornes `minAllowed` / `maxAllowed` |

Le mode `Off` est le bon point de départ en production : on lit les recommandations, puis on corrige
les manifestes. La console GKE affiche ces mêmes recommandations dans l'onglet de chaque Deployment.

</details>

**2.3 - Le VPA applique ses recommandations**

Passez `updateMode` à `"Recreate"` et ajoutez une `resourcePolicy` qui borne les requests du
conteneur `webapp` entre `cpu: 25m` / `memory: 64Mi` (`minAllowed`) et `cpu: 500m` / `memory: 512Mi`
(`maxAllowed`). Appliquez et observez les Pods pendant 2 à 3 minutes. Quelles requests portent les
nouveaux Pods ? Le fichier `webapp-deployment.yaml` et l'objet Deployment ont-ils changé ?

<details><summary>Correction</summary>

```yaml
# webapp-vpa.yaml (extrait de spec)
  updatePolicy:
    updateMode: "Recreate"
  resourcePolicy:
    containerPolicies:
    - containerName: webapp
      minAllowed: { cpu: 25m, memory: 64Mi }
      maxAllowed: { cpu: 500m, memory: 512Mi }
```

```bash
kubectl apply -f webapp-vpa.yaml
kubectl get pods -l app=webapp -w        # les Pods sont évincés un par un, puis recréés

kubectl get pods -l app=webapp \
  -o custom-columns=POD:.metadata.name,CPU:.spec.containers[0].resources.requests.cpu,MEM:.spec.containers[0].resources.requests.memory
kubectl get pod -l app=webapp -o jsonpath='{.items[0].metadata.annotations}'; echo
# vpaUpdates: Pod resources updated by webapp: container 0: cpu request, memory request...

kubectl get deployment webapp -o jsonpath='{.spec.template.spec.containers[0].resources}'; echo
# toujours 50m / 64Mi
```

Le VPA ne modifie **pas** le Deployment : son *updater* **évince** les Pods dont les requests
s'écartent trop de la recommandation, et son *admission controller* réécrit les requests des Pods
**à leur création**. Les limits sont ajustées dans la même proportion que les requests.

| `updateMode` | Comportement |
| --- | --- |
| `Off` | recommandations seulement |
| `Initial` | requests appliquées à la création des Pods, jamais d'éviction |
| `Recreate` (ou `Auto`) | éviction des Pods mal dimensionnés, puis recréation |
| `InPlaceOrRecreate` | redimensionnement **sans redémarrage** quand c'est possible (versions récentes) |

🔎 **Attention** - le VPA n'évince pas les Pods d'un Deployment à **1 réplica** (par défaut, il en
faut au moins 2) : il ne veut pas couper le service. Avec 1 réplica, les nouvelles valeurs ne
s'appliquent qu'au prochain redémarrage.

</details>

**2.4 - 🩺 HPA et VPA sur le même Deployment ?**

Pourquoi ne faut-il **pas** activer, sur un même Deployment, un HPA sur l'utilisation CPU et un VPA
qui ajuste la request CPU ? Supprimez ensuite le VPA et le Pod de charge, et remettez les Pods aux
valeurs du manifeste.

<details><summary>Correction</summary>

Le HPA calcule son pourcentage **par rapport à la request**. Si le VPA augmente la request, le
pourcentage baisse et le HPA retire des Pods ; s'il la baisse, le HPA en ajoute. Les deux boucles se
répondent et ne se stabilisent pas.

Combinaisons saines :

* HPA sur le CPU + VPA limité à la mémoire (`controlledResources: ["memory"]`) ;
* HPA sur une métrique métier (requêtes par seconde, longueur de file) + VPA ;
* sur GKE, le *multidimensional Pod autoscaling* combine les deux de façon coordonnée.

```bash
kubectl delete vpa webapp
kubectl delete pod charge-1
kubectl rollout restart deployment webapp      # les nouveaux Pods reprennent 50m / 64Mi
```

Supprimer le VPA ne modifie pas les Pods existants : seuls les Pods recréés reprennent les valeurs
du manifeste.

</details>

---

## Pour aller plus loin : l'autoscaling des nœuds

*(Lecture seulement : on ne modifie pas le nombre de nœuds dans ce lab.)*

Le HPA et le VPA créent ou grossissent des Pods ; encore faut-il de la place sur les nœuds. Le
**Cluster Autoscaler** surveille les Pods `Pending` pour cause de ressources insuffisantes, et ajoute
des nœuds au node pool. Inversement, il retire les nœuds sous-utilisés dont les Pods peuvent aller
ailleurs.

Sur GKE, il s'active par node pool, avec un minimum et un maximum de nœuds (dans la console :
onglet **Nœuds** → node pool → **Modifier** → *Activer l'autoscaling*). En ligne de commande :

```bash
# Exemple, à ne pas exécuter pendant la formation
gcloud container clusters update <cluster> --region <région> \
  --node-pool default-pool --enable-autoscaling --min-nodes 1 --max-nodes 2    # par zone
```

Ce que vous observeriez :

| Situation | Signal dans Kubernetes |
| --- | --- |
| des Pods ne trouvent pas de place | événement `FailedScheduling` (`Insufficient cpu`), puis `TriggeredScaleUp` sur le Pod ; un nouveau nœud apparaît dans `kubectl get nodes` en 1 à 2 minutes |
| un Pod demande plus qu'un nœud entier ne peut offrir | événement `NotTriggerScaleUp` : même un nœud neuf ne suffirait pas |
| la charge retombe | après une dizaine de minutes de sous-utilisation, un nœud est drainé puis supprimé |

Un nœud n'est **pas** retiré si l'un de ses Pods ne peut pas être déplacé : Pod sans contrôleur,
PodDisruptionBudget atteint, annotation `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"`...

🔎 **Attention** - comme le scheduler (Lab 10), le Cluster Autoscaler raisonne sur les **requests**,
pas sur la consommation réelle : un nœud saturé de requests mais consommé à 10 % ne sera jamais
retiré.

☁️ **Sur GKE** - en mode **Autopilot**, Google gère les nœuds pour vous : vous ne déclarez que des
Pods et leurs requests, et vous payez ces requests plutôt que les nœuds.

🏠 **En local** - le Cluster Autoscaler doit pouvoir **créer des machines** : il s'appuie sur l'API
d'un fournisseur (cloud, Cluster API, OpenStack...). Sur des VM Vagrant ou des serveurs physiques, la
capacité est fixe. Sur AWS, **Karpenter** est une alternative courante.

---

## ✅ État attendu en fin de lab

* Deployment `webapp` à 3 réplicas, avec requests `50m` / `64Mi` et limits `250m` / `256Mi` ;
* aucun HPA, aucun VPA, aucun Pod de charge.

---

## Points clés

* Trois autoscalers : **HPA** (nombre de Pods), **VPA** (taille des Pods), **Cluster Autoscaler**
  (nombre de nœuds). Ils s'enchaînent.
* HPA : cible en pourcentage de la **request** CPU (sans request : `<unknown>`). Montée rapide,
  descente après 5 min de stabilisation. Avec un HPA, retirez `replicas` du manifeste.
* VPA : commencez en mode `Off` pour lire les recommandations ; en `Recreate`, il évince et
  redimensionne les Pods sans toucher au Deployment. Pas de HPA CPU et de VPA CPU sur le même
  Deployment.
* Cluster Autoscaler : déclenché par des Pods `Pending` pour manque de ressources, borné par un
  minimum et un maximum de nœuds.
* Tous raisonnent sur les **requests** : des requests justes sont la condition d'un autoscaling
  efficace.
* 🏠 Localement, le HPA fonctionne avec metrics-server ; le VPA s'installe à part ; le Cluster
  Autoscaler exige une infrastructure capable de créer des machines.
