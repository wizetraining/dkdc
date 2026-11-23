# LAB 03 - Workloads : Deployments et DaemonSet

> **Cluster : GKE** (contexte `cluster-lab-dkdc`) | **Durée : ~55 min**
> **Prérequis :** Lab 02.

Un Pod seul ne se relève pas d'une panne. Le **Deployment** surveille en permanence l'écart entre
l'état désiré (« 3 Pods en v1 ») et l'état réel, et le corrige. Il sait aussi faire évoluer une
application sans coupure, et revenir en arrière.

À partir de ce lab, vous construisez pas à pas une vraie application : **Countvisit**, un compteur
de visites adossé à une base PostgreSQL. Elle vous suivra jusqu'au dernier lab, où vous la
packagerez avec Helm puis la déploierez en GitOps.

## Objectifs

1. Déployer Countvisit et comprendre la chaîne **Deployment → ReplicaSet → Pods**.
2. Observer l'auto-réparation et le scaling.
3. Passer au mode **déclaratif** : un fichier YAML comme source de vérité.
4. Mettre à jour l'application (**rolling update**), diagnostiquer une mise à jour ratée et
   **revenir en arrière**.
5. Déployer un agent sur chaque nœud avec un **DaemonSet**.

**Documentation utile :** [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) · [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) · [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) · [`kubectl rollout`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/)

---

## L'application Countvisit

| | |
| --- | --- |
| **Images** | `public.ecr.aws/wizetraining/webapp-count:v1` et `:v2` |
| **Port d'écoute** | `5000` |
| **Dépendance** | une base PostgreSQL joignable sous le nom d'hôte `postgres`, port `5432` |
| **Variables lues** | `PG_HOST` (défaut `postgres`), `PG_PORT` (`5432`), `PG_USER` (`postgres`), `PG_PASSWORD` (`postgres`), `PG_DB` (`postgres`) |
| **Comportement** | affiche le nom du conteneur qui a répondu, puis tente 5 fois de joindre PostgreSQL. En cas de succès, incrémente et affiche le compteur ; sinon affiche « Echec de connexion à la base de données ». |

> Sans base de données, la page affiche « Echec de connexion » : **c'est normal dans ce lab**.
> La base arrive au Lab 04.

<details><summary>Code de l'application webapp-count:v1</summary>

```python
import time
import socket
import logging
import json
import os
import psycopg2
from flask import Flask, Response

app = Flask(__name__)

# Configuration des logs pour voir l'activité en temps réel
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(message)s')

# Paramètres de connexion PostgreSQL (Docker Compose résoudra le hostname 'postgres')
PG_HOST = os.getenv('PG_HOST', 'postgres')
PG_PORT = os.getenv('PG_PORT', '5432')
PG_USER = os.getenv('PG_USER', 'postgres')
PG_PASSWORD = os.getenv('PG_PASSWORD', 'postgres')
PG_DB = os.getenv('PG_DB', 'postgres')

def stream_connection_and_count():
    """
    Générateur Server-Sent Events (SSE).
    Tente de se connecter à PostgreSQL, incrémente le compteur, et envoie 
    l'état en temps réel au navigateur ainsi qu'aux logs.
    """
    retries = 5
    while True:
        # 1. On log en console et on notifie le navigateur de la tentative
        logging.info(f"Tentative de connexion à PostgreSQL (Essais restants : {retries})...")
        yield f"data: {json.dumps({'status': 'loading', 'message': f'Recherche de la base de données... ({retries} essais restants)'})}\n\n"
        
        try:
            # 2. Tentative de connexion
            conn = psycopg2.connect(
                host=PG_HOST, port=PG_PORT, user=PG_USER, password=PG_PASSWORD, dbname=PG_DB,
                connect_timeout=2
            )
            conn.autocommit = True
            with conn.cursor() as cur:
                # On s'assure que la table existe
                cur.execute("CREATE TABLE IF NOT EXISTS hits (id serial PRIMARY KEY, count integer);")
                # On insère la ligne si elle n'existe pas, sinon on l'incrémente
                cur.execute("INSERT INTO hits (id, count) VALUES (1, 1) ON CONFLICT (id) DO UPDATE SET count = hits.count + 1 RETURNING count;")
                count = cur.fetchone()[0]
            conn.close()

            # 3. Succès ! On notifie les logs et le navigateur
            logging.info(f"Connexion réussie ! Visiteur n°{count}")
            yield f"data: {json.dumps({'status': 'success', 'count': count, 'message': 'Connecté à la base de données'})}\n\n"
            break

        except psycopg2.OperationalError as exc:
            # 4. Echec de la tentative
            if retries == 0:
                logging.error("Échec critique : Impossible de joindre PostgreSQL.")
                yield f"data: {json.dumps({'status': 'error', 'message': 'Echec de connexion à la base de données'})}\n\n"
                break
            retries -= 1
            # Pause pour éviter de spammer et pour laisser le temps de voir l'animation côté navigateur
            time.sleep(1.5)

@app.route('/stream')
def stream():
    """Route asynchrone qui envoie les événements au navigateur en temps réel."""
    return Response(stream_connection_and_count(), mimetype='text/event-stream')

@app.route('/')
def hello():
    # L'affichage du HTML est IMMÉDIAT. Il n'y a plus de code bloquant ici.
    hostname = socket.gethostname()

    html_response = f"""
    <!DOCTYPE html>
    <html lang="fr">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Compteur de Visites</title>
        <style>
            body {{ font-family: sans-serif; text-align: center; margin-top: 50px; }}
            .container {{ padding: 20px; border: 1px solid #ccc; border-radius: 8px; display: inline-block; }}
            .hostname {{ font-size: 0.9em; color: #555; margin-top: 20px; }}
            .db-status {{ margin-top: 15px; font-size: 1.1em; }}
            /* Effet visuel clignotant pour le fun */
            .blinking {{ animation: blinker 1s linear infinite; color: #ff9800; font-weight: bold; }}
            @keyframes blinker {{ 50% {{ opacity: 0; }} }}
        </style>
    </head>
    <body>
        <div class="container">
            <h1>Bonjour !</h1>
            
            <p>Vous êtes le visiteur numéro <strong id="count-display">...</strong>.</p>
            <div class="hostname">Vous êtes sur le conteneur : {hostname}</div>
            <div class="db-status" id="db-status-display"><span class="blinking">Initialisation de la connexion... 📡</span></div>
        </div>

        <script>
            // Connexion à la route SSE pour recevoir les événements Python en direct
            const source = new EventSource('/stream');
            
            source.onmessage = function(event) {{
                const data = JSON.parse(event.data);
                const statusDisplay = document.getElementById('db-status-display');
                const countDisplay = document.getElementById('count-display');
                
                if (data.status === 'loading') {{
                    statusDisplay.innerHTML = '<span class="blinking">' + data.message + ' 📡</span>';
                }} else if (data.status === 'success') {{
                    statusDisplay.innerHTML = '<strong style="color:green;">' + data.message + '</strong>';
                    countDisplay.innerText = data.count;
                    source.close(); // Le travail est terminé, on ferme la connexion
                }} else if (data.status === 'error') {{
                    statusDisplay.innerHTML = '<span style="color:red;">' + data.message + '</span>';
                    countDisplay.innerText = 'N/A';
                    source.close();
                }}
            }};
        </script>
    </body>
    </html>
    """
    return html_response

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
```

</details>

---

## Partie 1 : Premier Deployment en mode impératif

**1.1 - Préparer le terrain**

Créez le namespace `countvisit` et faites-en votre namespace par défaut (vu au Lab 02).
Toute la suite du fil rouge se déroule dans ce namespace.

<details><summary>Correction</summary>

```bash
kubectl create namespace countvisit
kubectl config set-context --current --namespace=countvisit
```

</details>

**1.2 - Créer le Deployment**

Créez un Deployment `webapp` de **2 réplicas** avec l'image `public.ecr.aws/wizetraining/webapp-count:v1`.
Utilisez l'option `--port` pour déclarer le port 5000 du conteneur.

<details><summary>Correction</summary>

```bash
kubectl create deployment webapp --image=public.ecr.aws/wizetraining/webapp-count:v1 --replicas=2 --port=5000
```

🔎 **Attention** - `kubectl run` ne crée **qu'un Pod**. Pour un Deployment, c'est
`kubectl create deployment`. La confusion entre les deux est l'erreur la plus fréquente.

</details>

**1.3 - Qu'a-t-on créé ?**

Listez les Deployments, ReplicaSets et Pods. Combien d'objets cette seule commande a-t-elle créés ?
Comment le nom des Pods est-il construit ?

<details><summary>Correction</summary>

```bash
kubectl get deployment,replicaset,pods
```

```text
deployment.apps/webapp              2/2
replicaset.apps/webapp-7d9f8c5b4    2
pod/webapp-7d9f8c5b4-x8k2p          Running
pod/webapp-7d9f8c5b4-q4n7m          Running
```

Le Deployment crée un **ReplicaSet** (`webapp-<hash>`), qui crée les **Pods**
(`webapp-<hash>-<suffixe>`). Le `<hash>` est calculé à partir du *template* de Pod : changer le
template (une nouvelle image par exemple) produira un **nouveau** ReplicaSet.

Rôles respectifs : le ReplicaSet **maintient le nombre** de Pods ; le Deployment **gère les
versions** (mise à jour, historique, retour arrière).

</details>

**1.4 - Auto-réparation**

Dans un premier terminal, observez les Pods en continu (`-w`). Dans un second terminal, supprimez
l'un des Pods à la main. Que fait le Deployment ?

<details><summary>Correction</summary>

```bash
kubectl get pods -w                     # terminal 1
kubectl delete pod <un-des-pods>        # terminal 2
```

Un nouveau Pod apparaît immédiatement. Le ReplicaSet constate 1 Pod au lieu de 2 et corrige
l'écart : c'est la **boucle de réconciliation**, le principe de fonctionnement de tout Kubernetes.

</details>

**1.5 - Scaling**

Passez à **4 réplicas** avec `kubectl scale`, puis retrouvez la trace de cette opération dans les
événements du Deployment. Revenez à 2.

<details><summary>Correction</summary>

```bash
kubectl scale deployment webapp --replicas=4
kubectl describe deployment webapp | tail -5
# Normal  ScalingReplicaSet  ... Scaled up replica set webapp-7d9f8c5b4 from 2 to 4
kubectl scale deployment webapp --replicas=2
```

</details>

---

## Partie 2 : Passer au mode déclaratif

**2.1 - Générer le manifeste**

Créez le dossier `~/labs/countvisit` : vous y rangerez **tous** les manifestes de l'application
jusqu'à la fin de la formation. Générez-y le YAML du Deployment dans `webapp-deployment.yaml`
(`--dry-run=client -o yaml`), avec 3 réplicas.

<details><summary>Correction</summary>

```bash
mkdir -p ~/labs/countvisit && cd ~/labs/countvisit
kubectl create deployment webapp --image=public.ecr.aws/wizetraining/webapp-count:v1 \
  --replicas=3 --port=5000 $do > webapp-deployment.yaml
```

</details>

**2.2 - Mettre le fichier au propre**

Modifiez le fichier pour obtenir :

* `replicas: 3` ;
* le label `app: webapp` sur le Deployment, dans le `selector` **et** dans le `template` ;
* un conteneur nommé **`webapp`** (le nom généré par défaut est `webapp-count`, tiré du nom de
  l'image) ;
* `containerPort: 5000` ;
* supprimez les champs vides générés (`creationTimestamp`, `resources: {}`, `strategy: {}`, `status: {}`).

Quelle règle lie `selector.matchLabels` et `template.metadata.labels` ?

<details><summary>Correction</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp          # doit correspondre au selector ci-dessus
    spec:
      containers:
      - name: webapp
        image: public.ecr.aws/wizetraining/webapp-count:v1
        ports:
        - containerPort: 5000
```

Le `selector.matchLabels` doit **sélectionner** les labels du template : c'est ainsi que le
ReplicaSet reconnaît « ses » Pods. Sinon, l'API refuse l'objet :
`selector does not match template labels`.

🔎 **Attention** - le `selector` d'un Deployment est **immuable** après création. Pour le changer,
il faut supprimer et recréer le Deployment.

</details>

**2.3 - Appliquer le fichier**

Supprimez d'abord le Deployment créé à la main (un objet créé en impératif n'a pas été
« déclaré », et son nom de conteneur diffère), puis appliquez votre fichier.

<details><summary>Correction</summary>

```bash
kubectl delete deployment webapp
kubectl apply -f webapp-deployment.yaml
kubectl get deployment webapp
```

</details>

**2.4 - Le fichier comme source de vérité**

Passez à 5 réplicas **en modifiant le fichier** puis en le ré-appliquant. Quel est l'avantage par
rapport à `kubectl scale` ? Remettez ensuite `replicas: 3`.

<details><summary>Correction</summary>

```bash
# replicas: 5 dans le fichier, puis :
kubectl apply -f webapp-deployment.yaml
# replicas: 3, puis :
kubectl apply -f webapp-deployment.yaml
```

Le fichier reste la **source de vérité** : versionnable dans Git, relisible, rejouable sur un autre
cluster. `kubectl scale` modifie le cluster sans laisser de trace.

🔎 **Attention** - après un `kubectl scale`, le fichier local est désynchronisé : le prochain
`kubectl apply -f` ramènera silencieusement le nombre de réplicas à la valeur du fichier.

</details>

---

## Partie 3 : Mises à jour et retour arrière

**3.1 - La stratégie par défaut**

Quelle est la stratégie de mise à jour appliquée par défaut à votre Deployment ? Quelles valeurs
de `maxSurge` et `maxUnavailable` sont utilisées ? (`describe`, ou
`kubectl explain deployment.spec.strategy`)

<details><summary>Correction</summary>

```bash
kubectl describe deployment webapp | grep -E "StrategyType|RollingUpdateStrategy"
# StrategyType:           RollingUpdate
# RollingUpdateStrategy:  25% max unavailable, 25% max surge
```

* `maxUnavailable` : combien de Pods peuvent manquer pendant la bascule ;
* `maxSurge` : combien de Pods **en plus** du nombre désiré peuvent être créés pendant la bascule.

</details>

**3.2 - Documenter les révisions**

Ajoutez au Deployment l'annotation `kubernetes.io/change-cause: "Version initiale v1"` (dans
`metadata.annotations` du Deployment) et ré-appliquez.

<details><summary>Correction</summary>

```yaml
metadata:
  name: webapp
  labels:
    app: webapp
  annotations:
    kubernetes.io/change-cause: "Version initiale v1"
```

```bash
kubectl apply -f webapp-deployment.yaml
# Équivalent impératif :
kubectl annotate deployment webapp kubernetes.io/change-cause="Version initiale v1" --overwrite
```

L'ancienne option `--record` est dépréciée au profit de cette annotation. C'est elle qui alimente
la colonne `CHANGE-CAUSE` de `kubectl rollout history`.

</details>

**3.3 - Passer en v2**

Passez l'application en **v2** (image `public.ecr.aws/wizetraining/webapp-count:v2`), soit en
modifiant le YAML, soit avec `kubectl set image` (attention au **nom du conteneur**), et mettez à
jour l'annotation `change-cause` avec « Passage en v2 ». Pendant la mise à jour, observez dans un
autre terminal `kubectl get rs -w`, puis lancez `kubectl rollout status`.

Combien de ReplicaSets existent ensuite ? Que sont devenus les Pods v1 ?

<details><summary>Correction</summary>

```bash
kubectl get rs -w                                    # terminal 2

kubectl set image deployment/webapp webapp=public.ecr.aws/wizetraining/webapp-count:v2
kubectl annotate deployment webapp kubernetes.io/change-cause="Passage en v2" --overwrite
kubectl rollout status deployment/webapp
kubectl get rs
```

```text
NAME                DESIRED   CURRENT   READY
webapp-7d9f8c5b4    0         0         0       ← v1, conservé à 0 pour permettre le rollback
webapp-5c6b7d8e9    3         3         3       ← v2
```

Le nouveau ReplicaSet monte progressivement pendant que l'ancien descend, **sans coupure**.
L'ancien est conservé (vide) : c'est lui qui rend le retour arrière instantané. Le nombre de
ReplicaSets conservés est réglé par `spec.revisionHistoryLimit` (10 par défaut).

🔎 **Attention** - `kubectl set image deployment/webapp <CONTENEUR>=<image>` : la clé est le **nom
du conteneur**, pas celui du Deployment. Avec un nom erroné :
`error: unable to find container named "webapp-count"`. Vérifiez-le avec :
`kubectl get deploy webapp -o jsonpath='{.spec.template.spec.containers[*].name}'`

</details>

**3.4 - Historique et rollback**

Affichez l'historique des révisions, le détail de la révision 1, puis revenez à la v1.

<details><summary>Correction</summary>

```bash
kubectl rollout history deployment webapp
kubectl rollout history deployment webapp --revision=1

kubectl rollout undo deployment webapp                  # révision précédente
kubectl rollout undo deployment webapp --to-revision=1  # ou une révision précise
kubectl rollout status deployment webapp
```

Un `undo` ne « remonte » pas l'historique : il crée une **nouvelle** révision qui reprend le
contenu de l'ancienne.

</details>

**3.5 - Bonus : la stratégie `Recreate`**

Passez la stratégie à `Recreate` (`spec.strategy.type: Recreate`, sans bloc `rollingUpdate`),
déployez la v2 et comparez le comportement des Pods avec le rolling update. Quel est le risque ?
Remettez ensuite `RollingUpdate`.

<details><summary>Correction</summary>

```yaml
spec:
  strategy:
    type: Recreate
```

Avec `Recreate`, **tous** les anciens Pods sont supprimés **avant** la création des nouveaux :
coupure de service assurée. À réserver aux applications qui ne supportent pas deux versions
simultanées (schéma de base incompatible, volume `ReadWriteOnce` partagé).

</details>

---

## Partie 4 : 🩺 Diagnostic - une mise à jour ratée

**4.1 - Simuler une erreur humaine**

Déployez l'image `public.ecr.aws/wizetraining/webapp-count:v999`, qui n'existe pas. Puis :

* listez les Pods : combien en voyez-vous, et dans quels états ?
* lancez `kubectl rollout status` avec un délai de 30 secondes : que répond-il ?
* trouvez le message d'erreur exact dans les événements d'un Pod en échec ;
* les anciens Pods v1 ont-ils été arrêtés ? L'application est-elle encore en service ?

<details><summary>Correction</summary>

```bash
kubectl set image deployment/webapp webapp=public.ecr.aws/wizetraining/webapp-count:v999
kubectl get pods
```

```text
webapp-7d9f8c5b4-x8k2p   1/1   Running            ← anciens Pods v1, toujours là
webapp-7d9f8c5b4-q4n7m   1/1   Running
webapp-7d9f8c5b4-z2w9r   1/1   Running
webapp-6f5d4c3b2-k1j8h   0/1   ImagePullBackOff   ← nouveaux Pods, bloqués
```

```bash
kubectl rollout status deployment/webapp --timeout=30s
# error: timed out waiting for the condition
kubectl describe pod <pod-en-erreur> | tail -5
# Failed to pull image ".../webapp-count:v999": ... not found
```

**L'application reste en service.** Avec `maxUnavailable: 25%` sur 3 réplicas, Kubernetes n'a
retiré au plus qu'un Pod v1 ; il ne retirera les autres que lorsque les nouveaux seront `Ready`.
C'est ce qui rend le rolling update sûr.

</details>

**4.2 - Réparer**

Revenez à la version précédente et vérifiez que tout est rentré dans l'ordre.

<details><summary>Correction</summary>

```bash
kubectl rollout undo deployment webapp
kubectl rollout status deployment webapp
kubectl get rs                 # le ReplicaSet v999 est à 0
```

💡 **Bon réflexe** - un Deployment qui « ne passe pas » se diagnostique en trois commandes :

```bash
kubectl rollout status deployment/<nom> --timeout=30s
kubectl get rs -l app=<label>                 # quel ReplicaSet porte les Pods en échec ?
kubectl describe pod <pod-en-echec> | tail    # pourquoi ?
```

</details>

---

## Partie 5 : Un autre contrôleur - le DaemonSet

Un Deployment garantit **N Pods**, placés là où le scheduler le décide. Certains besoins sont
différents : l'équipe d'exploitation veut un **agent qui tourne sur chaque nœud**, et un seul par
nœud. C'est le modèle des collecteurs de logs et des agents de supervision (Elastic Agent,
Filebeat, Fluent Bit...), mais aussi de composants du cluster lui-même. Le contrôleur dédié est le
**DaemonSet**.

**5.1 - Les DaemonSets du cluster**

Listez les DaemonSets de tous les namespaces. Lesquels reconnaissez-vous (Lab 01) ? Pour celui qui
collecte les logs des conteneurs (`fluentbit-gke`), combien de Pods sont désirés, et pourquoi ce
nombre ?

<details><summary>Correction</summary>

```bash
kubectl get daemonsets -A
```

```text
NAMESPACE     NAME                 DESIRED   CURRENT   READY
kube-system   fluentbit-gke        3         3         3
kube-system   gke-metrics-agent    3         3         3
kube-system   pdcsi-node           3         3         3
kube-system   ...                  (réseau, DNS local, selon la configuration du cluster)
```

`DESIRED` = le nombre de nœuds éligibles, ici vos 3 nœuds. La collecte des logs (vers Cloud
Logging), des métriques (vers Cloud Monitoring) ou le pilote des disques doivent être présents sur
**chaque** nœud : c'est exactement le rôle d'un DaemonSet.

Un agent Elastic (Elastic Agent, Filebeat) se déploie de la même façon pour envoyer les logs des
conteneurs vers Elasticsearch.

</details>

**5.2 - Un agent de supervision**

Écrivez `node-agent.yaml` : un DaemonSet `node-agent` dans le namespace `countvisit`, label
`app: node-agent`, image busybox ECR. Toutes les 30 secondes, le conteneur affiche l'heure, le nom
du **nœud** qui l'héberge et la charge de la machine :

```bash
sh -c 'while true; do echo "$(date +%T) noeud=$NODE_NAME charge=$(cat /proc/loadavg)"; sleep 30; done'
```

La variable `NODE_NAME` doit être fournie par Kubernetes à partir du champ `spec.nodeName` du Pod
(*downward API* : `env` + `valueFrom.fieldRef`).

Appliquez. Combien de Pods sont créés, et sur quels nœuds ? Lisez les logs de tous les agents en
une seule commande.

<details><summary>Correction</summary>

💡 **Astuce** - il n'existe **pas** de `kubectl create daemonset`. Générez un Deployment,
puis changez `kind: Deployment` en `kind: DaemonSet` et supprimez `replicas` et `strategy` :

```bash
kubectl create deployment node-agent --image=public.ecr.aws/wizetraining/busybox:latest $do > node-agent.yaml
```

```yaml
# node-agent.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-agent
  labels:
    app: node-agent
spec:
  selector:
    matchLabels:
      app: node-agent
  template:
    metadata:
      labels:
        app: node-agent
    spec:
      containers:
      - name: agent
        image: public.ecr.aws/wizetraining/busybox:latest
        command:
        - sh
        - -c
        - 'while true; do echo "$(date +%T) noeud=$NODE_NAME charge=$(cat /proc/loadavg)"; sleep 30; done'
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName      # downward API : le Pod connaît son nœud
```

```bash
kubectl apply -f node-agent.yaml
kubectl get daemonset node-agent            # DESIRED 3
kubectl get pods -l app=node-agent -o wide  # un Pod par nœud
kubectl logs -l app=node-agent --prefix --tail=2
```

**3 Pods, un par nœud.** Sur GKE, tous les nœuds visibles sont des nœuds de travail : le plan de
contrôle n'en fait pas partie (Lab 01).

Remarquez que la charge affichée est celle du **nœud**, pas du conteneur : `/proc/loadavg` n'est
pas isolé. Vous retrouverez ce phénomène au Lab 10.

</details>

**5.3 - Un nœud qui repousse les Pods**

Un agent de supervision doit couvrir **tous** les nœuds, même ceux qu'on réserve à un usage
particulier. On « réserve » un nœud en lui posant un **taint** : une marque qui repousse les Pods qui
ne la *tolèrent* pas (notion approfondie au Lab 10).

Dans un second terminal, observez les Pods de l'agent (`-w`). Dans le premier, posez sur le premier
nœud le taint `maintenance=oui:NoExecute` (`kubectl taint nodes <nœud> maintenance=oui:NoExecute`).
Que devient le Pod de l'agent sur ce nœud ? Ajoutez ensuite au DaemonSet la *toleration* qui lui
permet de rester sur ce nœud, réappliquez, et constatez. Retirez enfin le taint.

<details><summary>Correction</summary>

```bash
kubectl get pods -l app=node-agent -o wide -w                       # terminal 2

NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')   # terminal 1
kubectl taint nodes $NODE maintenance=oui:NoExecute
kubectl get daemonset node-agent                                    # DESIRED 2
```

L'effet `NoExecute` **expulse** les Pods qui ne le tolèrent pas : le Pod de l'agent quitte le nœud,
et le DaemonSet ne vise plus que 2 nœuds. Le nœud n'est plus supervisé.

```yaml
# node-agent.yaml (spec.template.spec)
      tolerations:
      - key: maintenance
        operator: Exists          # quelle que soit la valeur
        effect: NoExecute
```

```bash
kubectl apply -f node-agent.yaml
kubectl get daemonset node-agent                                    # DESIRED 3
kubectl taint nodes $NODE maintenance=oui:NoExecute-                # le "-" final retire le taint
```

Les DaemonSets système de GKE doivent tourner sur **tous** les nœuds : ils déclarent pour cela une
*toleration* universelle.

```bash
kubectl get daemonset -n kube-system fluentbit-gke -o jsonpath='{.spec.template.spec.tolerations}'; echo
# [{"operator":"Exists"}, ...]   → tolère tous les taints
```

Ajouter un nœud au cluster a le même effet que retirer le taint : l'agent y démarre tout seul, sans
aucune action de votre part.

🏠 **En local** - avec kubeadm, le nœud du plan de contrôle porte le taint
`node-role.kubernetes.io/control-plane:NoSchedule` : on ajoute une toleration sur cette clé pour
superviser aussi ce nœud. Sur GKE, la question ne se pose pas : le plan de contrôle n'est pas l'un de
vos nœuds.

</details>

**5.4 - DaemonSet ou Deployment ?**

Que se passe-t-il si vous supprimez un Pod de l'agent ? Si vous tentez `kubectl scale` sur le
DaemonSet ? Pourquoi un DaemonSet n'a-t-il pas de champ `replicas` ?

<details><summary>Correction</summary>

```bash
kubectl delete pod <un-pod-node-agent>
kubectl get pods -l app=node-agent -o wide   # recréé sur le MÊME nœud
kubectl scale daemonset node-agent --replicas=5
# error: ... cannot be scaled
```

Le nombre de Pods d'un DaemonSet est **dicté par le nombre de nœuds éligibles**, pas par vous.
Ajoutez un nœud au cluster : l'agent y démarre tout seul. Retirez-en un : son Pod disparaît.

| | Deployment | DaemonSet |
| --- | --- | --- |
| Nombre de Pods | `replicas` | un par nœud éligible |
| Placement | décidé par le scheduler | un Pod sur chaque nœud |
| Cas d'usage | applications | agents d'infrastructure (logs, supervision, réseau, stockage) |

🧹 `kubectl delete daemonset node-agent`

</details>

---

## ✅ État attendu en fin de lab

Le Deployment `webapp` tourne en **v1**, **3 réplicas**, stratégie `RollingUpdate`, dans le
namespace `countvisit`. Le fichier `~/labs/countvisit/webapp-deployment.yaml` reflète exactement
cet état (vérifiez l'image et la stratégie avant de passer au Lab 04).

```bash
kubectl get deployment webapp          # 3/3
kubectl apply -f ~/labs/countvisit/webapp-deployment.yaml   # "unchanged" ou "configured"
```

---

## Points clés

* **Deployment → ReplicaSet → Pods** : le Deployment gère les versions, le ReplicaSet le nombre.
* `selector.matchLabels` doit correspondre à `template.metadata.labels`, et il est **immuable**.
* Le fichier YAML est la source de vérité ; `kubectl scale` le désynchronise.
* `kubectl set image deployment/<nom> <conteneur>=<image>` : la clé est le nom du **conteneur**.
* `rollout status` / `history` / `undo` : suivre, consulter et annuler une mise à jour.
* Un rolling update raté ne coupe pas le service : les anciens Pods restent tant que les nouveaux
  ne sont pas `Ready`.
* `Recreate` = coupure de service, à réserver aux cas qui l'exigent.
* **DaemonSet** = un Pod par nœud éligible, pas de `replicas`. Un nœud ajouté au cluster reçoit
  automatiquement son Pod. Pas de `kubectl create daemonset` : générez un Deployment et changez le
  `kind`.
