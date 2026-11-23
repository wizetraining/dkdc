# LAB 05 - ConfigMaps, Secrets, Job et CronJob

> **Cluster : GKE** (contexte `cluster-lab-dkdc`) | **Durée : ~55 min**
> **Prérequis :** Lab 04 terminé (`webapp` + `postgres`, namespace `countvisit`, dossier `~/labs/countvisit`).

Une image de conteneur doit être **identique** en recette et en production. Tout ce qui change
d'un environnement à l'autre sort de l'image et des manifestes : la configuration non sensible va
dans un **ConfigMap**, les identifiants dans un **Secret**.

## Objectifs

1. Séparer la **configuration** et les **mots de passe** de l'image et des manifestes.
2. Injecter un ConfigMap en **variables d'environnement**, et un Secret en variables **et** en
   **fichier monté**.
3. Comprendre les limites d'un Secret (base64 n'est pas du chiffrement).
4. Diagnostiquer une application mal configurée à partir de ses logs et de ses événements.
5. Sauvegarder la base avec un **Job**, puis un **CronJob**, qui réutilisent la même configuration.

**Documentation utile :** [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/) · [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) · [Utiliser un Secret dans un Pod](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/) · [Configurer un Pod avec un ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) · [Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/) · [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)

---

## L'application Countvisit « Entreprise »

À partir de ce lab, Countvisit passe dans sa version « Entreprise », conçue pour être configurée
proprement.

| | |
| --- | --- |
| **Image** | `public.ecr.aws/wizetraining/webapp-count-secure:v1` |
| **Port d'écoute** | `5000` |
| **Variables lues** | `PG_HOST` (défaut `postgres`), `PG_PORT` (`5432`), `PG_USER` (`postgres`), `PG_DB` (`postgres`) |
| **Mot de passe** | **lu dans un fichier**, jamais dans une variable : chemin donné par `PG_PASSWORD_FILE`, défaut `/etc/app/secrets/pg_password` |
| **Sondes** | `GET /healthz` (vivant) et `GET /readyz` (prêt) - utilisées au Lab 07 |
| **Si le fichier manque** | écrit un avertissement dans ses logs et tente de se connecter sans mot de passe |

<details><summary>Code de l'application webapp-count-secure:v1</summary>

```python
import time
import socket
import logging
import json
import os
import psycopg2
from flask import Flask, Response, jsonify, request

# --------------------------------------------------
# Configuration du logging (Format standard Entreprise)
# --------------------------------------------------
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger("app")

app = Flask(__name__)

# --------------------------------------------------
# Configuration PostgreSQL dynamique (12-factor compliant)
# --------------------------------------------------
PG_HOST = os.getenv('PG_HOST', 'postgres')
PG_PORT = os.getenv('PG_PORT', '5432')
PG_USER = os.getenv('PG_USER', 'postgres')
PG_DB = os.getenv('PG_DB', 'postgres')

logger.info(f"Configuration PostgreSQL: {PG_HOST}:{PG_PORT} (User: {PG_USER}, DB: {PG_DB})")

# --------------------------------------------------
# Gestion du mot de passe via Docker/K8s Secret
# --------------------------------------------------
def get_pg_password():
    secret_path = os.getenv("PG_PASSWORD_FILE", "/etc/app/secrets/pg_password")
    try:
        with open(secret_path, 'r') as secret_file:
            password = secret_file.read().strip()
            # On ne logue JAMAIS le mot de passe, juste le succès
            logger.info(f"Mot de passe PostgreSQL chargé avec succès depuis {secret_path}.")
            return password
    except IOError:
        logger.warning(
            f"Secret introuvable dans {secret_path}. "
            "Tentative de connexion sans mot de passe."
        )
        return ""

# --------------------------------------------------
# Sondes Kubernetes (Liveness & Readiness)
# --------------------------------------------------
@app.route('/healthz')
def healthz():
    """Liveness Probe: Vérifie si le conteneur tourne."""
    return jsonify({"status": "alive"}), 200

@app.route('/readyz')
def readyz():
    """
    Readiness Probe: Normalement on teste la DB ici. 
    Mais pour ton lab (où on veut voir l'UI chercher la DB en temps réel), 
    on doit dire à K8s que l'UI est prête à être affichée.
    """
    return jsonify({"status": "ready", "ui": "available"}), 200

# --------------------------------------------------
# Fonction métier : Connexion et incrément (SSE)
# --------------------------------------------------
def stream_connection_and_count():
    """Générateur Server-Sent Events (SSE)."""
    password = get_pg_password()
    retries = 5
    
    while True:
        logger.info(f"Tentative de connexion à PostgreSQL (Essais restants : {retries})...")
        yield f"data: {json.dumps({'status': 'loading', 'message': f'Recherche de la base de données... ({retries} essais restants)'})}\n\n"
        
        try:
            conn = psycopg2.connect(
                host=PG_HOST, port=PG_PORT, user=PG_USER, password=password, dbname=PG_DB,
                connect_timeout=2
            )
            conn.autocommit = True
            with conn.cursor() as cur:
                cur.execute("CREATE TABLE IF NOT EXISTS hits (id serial PRIMARY KEY, count integer);")
                cur.execute("INSERT INTO hits (id, count) VALUES (1, 1) ON CONFLICT (id) DO UPDATE SET count = hits.count + 1 RETURNING count;")
                count = cur.fetchone()[0]
            conn.close()

            logger.info(f"Connexion réussie ! Visiteur n°{count}")
            yield f"data: {json.dumps({'status': 'success', 'count': count, 'message': 'Connecté à PostgreSQL (Sécurisé)'})}\n\n"
            break

        except psycopg2.OperationalError as exc:
            if retries == 0:
                logger.error(f"Échec critique PostgreSQL : {exc}")
                yield f"data: {json.dumps({'status': 'error', 'message': 'Echec de connexion à PostgreSQL'})}\n\n"
                break
            retries -= 1
            time.sleep(1.5)

@app.route('/stream')
def stream():
    """Route asynchrone pour les événements en temps réel."""
    return Response(stream_connection_and_count(), mimetype='text/event-stream')

# --------------------------------------------------
# Route Principale (Chargement immédiat)
# --------------------------------------------------
@app.route('/')
def hello():
    hostname = socket.gethostname()

    html_response = f"""
    <!DOCTYPE html>
    <html lang="fr">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Compteur Sécurisé (PostgreSQL)</title>
        <style>
            body {{ font-family: -apple-system, sans-serif; text-align: center; margin-top: 50px; background: #f8f9fa; }}
            .container {{ padding: 30px; border: 1px solid #e9ecef; border-radius: 12px; display: inline-block; background: #fff; box-shadow: 0 4px 6px rgba(0,0,0,0.05); }}
            .hostname {{ font-size: 0.9em; color: #6c757d; margin-top: 25px; padding: 10px; background: #f8f9fa; border-radius: 6px; }}
            .db-status {{ margin-top: 20px; font-size: 1.1em; }}
            .blinking {{ animation: blinker 1s linear infinite; color: #ff9800; font-weight: bold; }}
            @keyframes blinker {{ 50% {{ opacity: 0; }} }}
        </style>
    </head>
    <body>
        <div class="container">
            <h1>Application d'Entreprise 🚀</h1>
            <p>Vous êtes le visiteur numéro <strong id="count-display" style="font-size: 1.2em;">...</strong>.</p>
            <div class="hostname">Instance : <code>{hostname}</code></div>
            <div class="db-status" id="db-status-display"><span class="blinking">Initialisation de la connexion... 📡</span></div>
        </div>

        <script>
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
                    source.close();
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
    app.run(host="0.0.0.0", port=5000)
```

</details>

---

## Partie 1 : Le problème

Affichez la définition du Deployment `postgres` en YAML et retrouvez le mot de passe de la base.
Que constatez-vous ? Quels sont les problèmes de cette méthode de configuration ?

<details><summary>Correction</summary>

```bash
kubectl get deployment postgres -o yaml | grep -B1 -A1 POSTGRES_PASSWORD
```

Le mot de passe est **en clair** dans le manifeste, donc dans Git, dans l'historique, et visible par
quiconque peut lire le Deployment. De plus, l'application utilise le compte superutilisateur
`postgres`. Ce lab corrige les deux.

</details>

---

## Partie 2 : ConfigMap - la configuration non sensible

**2.1 - Créer le ConfigMap**

Écrivez `webapp-configmap.yaml` : un ConfigMap `webapp-config` avec les clés `PG_HOST=postgres`,
`PG_PORT=5432` et `PG_DB=app_db`. Appliquez-le et vérifiez son contenu.

*(Astuce : `kubectl create configmap ... --from-literal=... $do` génère le YAML.)*

<details><summary>Correction</summary>

```bash
kubectl create configmap webapp-config \
  --from-literal=PG_HOST=postgres --from-literal=PG_PORT=5432 --from-literal=PG_DB=app_db \
  $do > webapp-configmap.yaml
```

```yaml
# webapp-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
data:
  PG_HOST: postgres
  PG_PORT: "5432"        # les valeurs d'un ConfigMap sont toujours des chaînes
  PG_DB: app_db
```

```bash
kubectl apply -f webapp-configmap.yaml
kubectl describe configmap webapp-config
```

Les trois façons de créer un ConfigMap en impératif :

| Option | Résultat |
| --- | --- |
| `--from-literal=CLE=valeur` | une clé par option |
| `--from-file=fichier.conf` | **une seule** clé, nommée d'après le fichier, contenant tout le fichier |
| `--from-env-file=fichier.env` | **une clé par ligne** `CLE=valeur` du fichier |

🔎 **Attention** - `--from-file` et `--from-env-file` ne produisent pas du tout le même résultat :
lisez bien l'énoncé.

</details>

**2.2 - Brancher le ConfigMap sur l'application**

Modifiez `webapp-deployment.yaml` pour :

* utiliser l'image `public.ecr.aws/wizetraining/webapp-count-secure:v1` ;
* charger **toutes** les clés du ConfigMap en variables d'environnement (`envFrom`).

Appliquez, puis vérifiez que les variables apparaissent bien dans le conteneur `webapp`.

<details><summary>Correction</summary>

```yaml
# webapp-deployment.yaml (spec.template.spec)
      containers:
      - name: webapp
        image: public.ecr.aws/wizetraining/webapp-count-secure:v1
        ports:
        - containerPort: 5000
        envFrom:
        - configMapRef:
            name: webapp-config
```

```bash
kubectl apply -f webapp-deployment.yaml
kubectl rollout status deployment/webapp
kubectl exec deploy/webapp -- env | grep ^PG_
```

```text
PG_DB=app_db
PG_HOST=postgres
PG_PORT=5432
```

Pour n'injecter **qu'une** clé, sous un autre nom, on utilise `env` + `configMapKeyRef` (vous le
ferez en Partie 4).

</details>

**2.3 - 🩺 Diagnostic : l'application ne se connecte plus**

Rechargez la page : le titre est devenu « Application d'Entreprise », mais la connexion à la base
échoue. **Sans regarder la correction**, trouvez dans les **logs** de l'application les deux
raisons de l'échec.

<details><summary>Correction</summary>

```bash
kubectl logs deploy/webapp | grep -E "Configuration|Secret|Échec"
```

```text
INFO - Configuration PostgreSQL: postgres:5432 (User: postgres, DB: app_db)
WARNING - Secret introuvable dans /etc/app/secrets/pg_password. Tentative de connexion sans mot de passe.
ERROR - Échec critique PostgreSQL : connection to server at "postgres" ... fe_sendauth: no password supplied
```

1. Le fichier de mot de passe n'existe pas : il sera fourni par un Secret en Partie 4.
2. La base `app_db` n'existe pas encore : PostgreSQL ne la créera qu'avec la variable
   `POSTGRES_DB` (Partie 4).

Les pods sont `Running` et `Ready` : **Kubernetes n'a rien à signaler**. Une erreur de configuration
applicative ne se voit que dans les logs de l'application. C'est la raison pour laquelle on lit
toujours les logs, même quand `kubectl get pods` est « vert ».

</details>

---

## Partie 3 : Secret - les identifiants

**3.1 - Le mot de passe sur le poste**

Créez un fichier local `secrets/pg_password.txt` (dans `~/labs/countvisit`) contenant un mot de
passe de votre choix, **sans retour à la ligne final** (option `-n` de `echo`). Ce fichier ne doit
jamais être commité dans Git.

<details><summary>Correction</summary>

```bash
mkdir -p secrets
echo -n "ChangeMe-Postgres123" > secrets/pg_password.txt
echo "secrets/" >> .gitignore
```

Sans `-n`, le fichier se termine par un saut de ligne, qui ferait partie du mot de passe.

</details>

**3.2 - Créer le Secret**

Créez un Secret `pg-secret` de type `kubernetes.io/basic-auth` contenant :

* la clé `username` : `app_user` ;
* la clé `password` : le contenu du fichier précédent (option `--from-file`).

*(Le type `basic-auth` impose exactement ces deux clés : l'API refuse le Secret s'il en manque une.)*

<details><summary>Correction</summary>

```bash
kubectl create secret generic pg-secret \
  --type=kubernetes.io/basic-auth \
  --from-literal=username=app_user \
  --from-file=password=./secrets/pg_password.txt
```

`--from-file=password=./fichier` : la partie avant `=` donne le **nom de la clé**, la partie après
le chemin du fichier.

Les types de Secrets à connaître :

| Type | Usage | Création |
| --- | --- | --- |
| `Opaque` | tout usage (défaut) | `kubectl create secret generic` |
| `kubernetes.io/basic-auth` | `username` + `password` | `generic --type=...` ou YAML |
| `kubernetes.io/tls` | certificat + clé (Ingress, Lab 08) | `kubectl create secret tls` |
| `kubernetes.io/dockerconfigjson` | identifiants de registre privé | `kubectl create secret docker-registry` |

</details>

**3.3 - Un Secret est-il secret ?**

Affichez le Secret en YAML. Pouvez-vous retrouver le mot de passe ? Que peut-on en conclure sur la
sécurité d'un Secret ?

<details><summary>Correction</summary>

```bash
kubectl get secret pg-secret -o yaml
kubectl get secret pg-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

Les valeurs sont **encodées en base64, pas chiffrées** : toute personne autorisée à lire le Secret
obtient le mot de passe en clair. La protection réelle repose sur :

1. le **RBAC** : qui a le droit de lire les Secrets (Lab 12) ;
2. le **chiffrement au repos** de la base etcd ;
3. en production, un **gestionnaire externe** (Vault, External Secrets Operator...).

☁️ **Sur GKE** - etcd est chiffré au repos par défaut, par Google. On peut ajouter un second niveau,
le *chiffrement des secrets au niveau de la couche application*, avec une clé Cloud KMS que vous
contrôlez (option à activer sur le cluster). Pour ne plus stocker du tout les secrets
dans le cluster, **Secret Manager** de Google Cloud peut les fournir directement aux Pods
(module complémentaire *Secret Manager* de GKE).

🏠 **En local** - c'est l'administrateur qui active le chiffrement d'etcd, en fournissant à l'API
Server un fichier `EncryptionConfiguration`. Par défaut, les Secrets y sont stockés en clair.

Et surtout : **un Secret ne va jamais dans Git**.

💡 **Astuce** - pour écrire un Secret en YAML sans encoder soi-même, utilisez le champ
`stringData` : Kubernetes encode à la création.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: exemple
type: Opaque
stringData:
  password: en-clair-ici
```

</details>

---

## Partie 4 : Brancher le tout

**4.1 - PostgreSQL sans mot de passe en clair**

Modifiez `postgres-deployment.yaml` :

* `POSTGRES_USER` et `POSTGRES_PASSWORD` viennent des clés `username` et `password` de `pg-secret`
  (`secretKeyRef`) ;
* `POSTGRES_DB` vient de la clé `PG_DB` du ConfigMap `webapp-config` (`configMapKeyRef`).

<details><summary>Correction</summary>

```yaml
# postgres-deployment.yaml (section du conteneur postgres)
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
```

</details>

**4.2 - L'application lit son mot de passe dans un fichier**

Modifiez `webapp-deployment.yaml` :

* `PG_USER` vient de la clé `username` de `pg-secret` ;
* le Secret est **monté comme un volume** dans `/etc/app/secrets`, et **seule** la clé `password`
  doit y apparaître, sous la forme du fichier `pg_password`.

<details><summary>Correction</summary>

```yaml
# webapp-deployment.yaml (spec.template.spec)
      containers:
      - name: webapp
        image: public.ecr.aws/wizetraining/webapp-count-secure:v1
        ports:
        - containerPort: 5000
        envFrom:
        - configMapRef:
            name: webapp-config
        env:
        - name: PG_USER
          valueFrom:
            secretKeyRef:
              name: pg-secret
              key: username
        volumeMounts:
        - name: pg-password
          mountPath: /etc/app/secrets
          readOnly: true
      volumes:
      - name: pg-password
        secret:
          secretName: pg-secret
          items:                 # ne monter QUE la clé password...
          - key: password
            path: pg_password    # ...sous ce nom de fichier
```

Sans `items`, **chaque clé** du Secret deviendrait un fichier (`username` et `password`).

</details>

**4.3 - Vérifier**

Appliquez les deux fichiers et attendez la fin des rollouts. Vérifiez :

* dans le navigateur : « Connecté à PostgreSQL (Sécurisé) » ;
* dans le Pod `webapp` : le fichier `/etc/app/secrets/pg_password` existe ;
* **aucune** variable d'environnement du Pod ne contient le mot de passe ;
* les logs confirment le chargement du mot de passe.

<details><summary>Correction</summary>

```bash
kubectl apply -f postgres-deployment.yaml -f webapp-deployment.yaml
kubectl rollout status deployment/postgres
kubectl rollout status deployment/webapp

kubectl exec deploy/webapp -- ls -l /etc/app/secrets/
kubectl exec deploy/webapp -- env | grep ^PG_          # PG_HOST, PG_PORT, PG_DB, PG_USER - pas de mot de passe
kubectl logs deploy/webapp | grep "chargé avec succès"
```

Le compteur repart à 1 : PostgreSQL a été recréé, toujours sans volume persistant (Lab 06).

Un mot de passe en **fichier** est préférable à une variable : une variable d'environnement
apparaît dans `kubectl describe`, dans les dumps de processus et parfois dans les traces
d'erreur des applications.

</details>

---

## Partie 5 : 🩺 Diagnostic - CreateContainerConfigError

**5.1 - Une clé qui n'existe pas**

Dans `postgres-deployment.yaml`, remplacez `key: PG_DB` par `key: PG_DATABASE` (une clé qui
n'existe pas dans le ConfigMap) et appliquez. Observez l'état du nouveau Pod, trouvez le message
exact, puis réparez. Le service a-t-il été interrompu ?

<details><summary>Correction</summary>

```bash
kubectl apply -f postgres-deployment.yaml
kubectl get pods -l app=postgres
# postgres-xxxx   0/1   CreateContainerConfigError

kubectl describe pod -l app=postgres | tail -5
# Error: couldn't find key PG_DATABASE in ConfigMap countvisit/webapp-config
```

`CreateContainerConfigError` : le Pod est planifié, l'image est là, mais le kubelet ne peut pas
**construire** le conteneur car une référence de configuration est invalide :

* ConfigMap ou Secret **inexistant** (ou dans un autre namespace) ;
* **clé** absente du ConfigMap ou du Secret.

```bash
# remettre key: PG_DB
kubectl apply -f postgres-deployment.yaml
kubectl rollout status deployment/postgres
```

Comme pour l'image inexistante du Lab 03, l'ancien Pod est resté en place pendant toute la panne :
le rolling update ne retire pas un Pod sain tant que son remplaçant n'est pas prêt.

🔎 **Attention** - un ConfigMap ou un Secret est **namespacé** : un Pod ne peut référencer que ceux
de **son propre** namespace.

</details>

**5.2 - Bonus : les changements de configuration sont-ils pris en compte ?**

Changez `PG_PORT` en `5433` dans le ConfigMap (`kubectl edit`), puis relisez la variable dans un Pod
`webapp` **existant**. La valeur a-t-elle changé ? Qu'en est-il d'un fichier monté depuis un Secret
ou un ConfigMap ? Comment forcer la prise en compte ? Remettez `5432` à la fin.

<details><summary>Correction</summary>

```bash
kubectl edit configmap webapp-config               # PG_PORT: "5433"
kubectl exec deploy/webapp -- printenv PG_PORT     # toujours 5432
```

| Mode de consommation | Mise à jour à chaud |
| --- | --- |
| variable d'environnement (`env`, `envFrom`) | **non** - figée au démarrage du conteneur |
| volume | **oui** - propagée en une minute environ |
| volume monté avec `subPath` | **non** |

```bash
kubectl rollout restart deployment/webapp          # recrée les Pods avec la nouvelle valeur
kubectl exec deploy/webapp -- printenv PG_PORT     # 5433 → l'application ne joint plus la base

kubectl edit configmap webapp-config               # remettre "5432"
kubectl rollout restart deployment/webapp
```

Dans tous les cas, l'application doit savoir relire sa configuration : un fichier mis à jour sur le
disque n'est d'aucune utilité si l'application ne le relit jamais.

</details>

---

## Partie 6 : Sauvegarder la base - Job et CronJob

La base contient désormais de vraies données. L'équipe demande une **sauvegarde** : un traitement
qui s'exécute, se termine, et ne doit **pas** être relancé en boucle comme un Deployment. C'est le
rôle du **Job** (exécution jusqu'à complétion) et du **CronJob** (un Job selon un planning).

La sauvegarde utilise l'outil `pg_dump`, présent dans l'image PostgreSQL. Elle réutilise
**exactement les mêmes** ConfigMap et Secret que l'application : c'est tout l'intérêt de les avoir
sortis des manifestes.

> Dans ce lab, le contenu de la sauvegarde est simplement écrit sur la sortie standard, donc dans
> les logs. En production, il serait écrit sur un volume ou envoyé vers un stockage objet.

**6.1 - Une sauvegarde ponctuelle**

Écrivez `pg-dump-job.yaml` : un Job `pg-dump-manuel` avec :

* l'image `public.ecr.aws/wizetraining/postgres:16-alpine` ;
* les variables `PGHOST` et `PGDATABASE`, issues des clés `PG_HOST` et `PG_DB` du ConfigMap
  `webapp-config`, et `PGUSER`, issue de la clé `username` de `pg-secret` (les outils PostgreSQL
  lisent ces variables automatiquement) ;
* le Secret `pg-secret` monté dans `/etc/app/secrets`, la clé `password` sous le nom `pg_password`,
  comme pour l'application ;
* la commande : `sh -c 'PGPASSWORD=$(cat /etc/app/secrets/pg_password) pg_dump --table=hits && echo SAUVEGARDE-OK'` ;
* au plus **2** nouvelles tentatives en cas d'échec, et une suppression automatique du Job
  **10 minutes** après sa fin.

Appliquez, attendez la fin, lisez le résultat.

<details><summary>Correction</summary>

```yaml
# pg-dump-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pg-dump-manuel
spec:
  backoffLimit: 2                  # 2 nouvelles tentatives au maximum
  ttlSecondsAfterFinished: 600     # nettoyage automatique 10 min après la fin
  template:
    spec:
      restartPolicy: Never         # Never ou OnFailure - JAMAIS Always dans un Job
      containers:
      - name: pg-dump
        image: public.ecr.aws/wizetraining/postgres:16-alpine
        command: ["sh", "-c", "PGPASSWORD=$(cat /etc/app/secrets/pg_password) pg_dump --table=hits && echo SAUVEGARDE-OK"]
        env:
        - name: PGHOST
          valueFrom:
            configMapKeyRef: { name: webapp-config, key: PG_HOST }
        - name: PGDATABASE
          valueFrom:
            configMapKeyRef: { name: webapp-config, key: PG_DB }
        - name: PGUSER
          valueFrom:
            secretKeyRef: { name: pg-secret, key: username }
        volumeMounts:
        - name: pg-password
          mountPath: /etc/app/secrets
          readOnly: true
      volumes:
      - name: pg-password
        secret:
          secretName: pg-secret
          items:
          - key: password
            path: pg_password
```

```bash
kubectl apply -f pg-dump-job.yaml
kubectl get job pg-dump-manuel -w        # COMPLETIONS 1/1
kubectl logs job/pg-dump-manuel | tail -8
# ... COPY public.hits (id, count) FROM stdin; ... SAUVEGARDE-OK
```

🔎 **Attention** - `restartPolicy: Always` est **interdit** dans un Job. Si vous générez le YAML
depuis un Deployment ou un Pod, corrigez ce champ.

💡 **Astuce** - `kubectl create job <nom> --image=<image> $do -- <commande>` produit un squelette
de Job.

</details>

**6.2 - 🩺 Un Job qui échoue**

Copiez le fichier en `pg-dump-ko.yaml`, nommez le Job `pg-dump-ko`, et remplacez la source de
`PGHOST` par la valeur en dur `postgress` (une faute de frappe). Appliquez. Combien de Pods
sont créés ? Quel est l'état final du Job, et pourquoi ? Trouvez le message d'erreur exact.

<details><summary>Correction</summary>

```yaml
        - name: PGHOST
          value: postgress
```

```bash
kubectl apply -f pg-dump-ko.yaml
kubectl get pods -l job-name=pg-dump-ko -w     # 3 Pods en Error : 1 essai + 2 tentatives
kubectl describe job pg-dump-ko | tail -5
# Warning  BackoffLimitExceeded  ... Job has reached the specified backoff limit
kubectl logs -l job-name=pg-dump-ko --tail=2
# pg_dump: error: could not translate host name "postgress" to address
```

Avec `restartPolicy: Never`, chaque échec crée un **nouveau Pod** : les Pods en erreur restent
visibles, avec leurs logs. Avec `OnFailure`, c'est le **même** Pod qui redémarre son conteneur.
Au-delà de `backoffLimit`, le Job passe en `Failed`.

🧹 `kubectl delete job pg-dump-ko`

</details>

**6.3 - Une sauvegarde planifiée**

Transformez la sauvegarde en CronJob `pg-backup`, exécuté **toutes les 2 minutes**, qui :

* ne démarre **jamais** une exécution si la précédente tourne encore ;
* ne conserve que les **3** derniers Jobs réussis et le dernier Job en échec.

Le `jobTemplate.spec` reprend le contenu du `spec` de votre Job. Observez les Jobs créés pendant
4 minutes, puis déclenchez une sauvegarde **à la demande**, sans attendre le prochain créneau.

<details><summary>Correction</summary>

```yaml
# pg-backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: pg-backup
spec:
  schedule: "*/2 * * * *"          # minute heure jour-du-mois mois jour-de-la-semaine
  concurrencyPolicy: Forbid        # Allow (défaut) | Forbid | Replace
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: pg-dump
            # ... exactement le même conteneur et les mêmes volumes que dans pg-dump-job.yaml
```

```bash
kubectl apply -f pg-backup-cronjob.yaml
kubectl get cronjob pg-backup
kubectl get jobs -w                                   # un Job toutes les 2 minutes

kubectl create job --from=cronjob/pg-backup pg-backup-maintenant
kubectl logs job/pg-backup-maintenant | tail -2
```

| `concurrencyPolicy` | Si l'exécution précédente tourne encore |
| --- | --- |
| `Allow` | on lance quand même (défaut) |
| `Forbid` | on saute ce créneau |
| `Replace` | on arrête l'ancienne et on lance la nouvelle |

</details>

**6.4 - Mettre la sauvegarde en pause**

Une maintenance de la base est prévue : suspendez le CronJob **sans le supprimer**. Vérifiez qu'aucun
nouveau Job n'est créé.

<details><summary>Correction</summary>

```bash
kubectl patch cronjob pg-backup -p '{"spec":{"suspend":true}}'
kubectl get cronjob pg-backup          # SUSPEND : True
```

Pour reprendre : `kubectl patch cronjob pg-backup -p '{"spec":{"suspend":false}}'`.

Laissez le CronJob suspendu pour la suite des labs.

</details>

---

## ✅ État attendu en fin de lab

* l'application « Entreprise » fonctionne et affiche « Connecté à PostgreSQL (Sécurisé) » ;
* plus aucun mot de passe en clair dans vos manifestes ;
* `pg-secret` (basic-auth) et `webapp-config` existent ;
* le CronJob `pg-backup` existe et est **suspendu** ;
* fichiers : `webapp-configmap.yaml`, `pg-dump-job.yaml` et `pg-backup-cronjob.yaml` ajoutés,
  `webapp-deployment.yaml` et `postgres-deployment.yaml` mis à jour, `secrets/` exclu de Git.

---

## Points clés

* ConfigMap = configuration non sensible. Secret = identifiants. Les deux sont **namespacés**.
* `envFrom` pour toutes les clés, `env` + `configMapKeyRef` / `secretKeyRef` pour une clé précise.
* Un volume Secret avec `items` ne monte que les clés choisies, sous le nom de fichier choisi.
* Un Secret est **encodé en base64, pas chiffré**. Protection = RBAC + chiffrement d'etcd (par défaut
  sur GKE, avec en option une clé Cloud KMS) + gestionnaire externe (Secret Manager, Vault).
* `stringData` évite d'encoder soi-même.
* Variable d'environnement = figée au démarrage → `kubectl rollout restart`.
* `CreateContainerConfigError` = ConfigMap, Secret ou **clé** introuvable.
* Un Pod `Running` peut être mal configuré : **lisez les logs de l'application**.
* **Job** : `restartPolicy` `Never` ou `OnFailure`, `backoffLimit`, `ttlSecondsAfterFinished`.
  **CronJob** : `schedule`, `concurrencyPolicy`, `suspend`, et
  `kubectl create job --from=cronjob/<nom>` pour une exécution à la demande.
