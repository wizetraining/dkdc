# LAB 14 - Helm 2 : créer la chart de Countvisit

> **Cluster : GKE** (contexte `cluster-lab-dkdc`) | **Durée : ~75 min**
> **Prérequis :** Labs 08, 09 et 13 terminés : namespace `countvisit` avec le Secret `pg-secret`, le
> cluster PostgreSQL `pg-database` (CloudNativePG), Ingress NGINX et l'Ingress `webapp` (Lab 08) ;
> variables `HOST_APP` et `INGRESS_IP` dans `~/formation.env`. Les identifiants du registre Harbor
> fournis par le formateur.

Au Lab 09, vous avez **consommé** un chart écrit par d'autres. Depuis le Lab 02, Countvisit est
déployée avec une demi-douzaine de fichiers YAML appliqués à la main, dans le bon ordre, avec des
valeurs écrites en dur. Dans ce lab, vous **empaquetez** l'application web dans votre propre chart
Helm, en partant de l'échafaudage généré par `helm create` et en le transformant pas à pas.

À la fin, une seule commande déploiera Countvisit, avec sa configuration, son Ingress et son
autoscaling optionnel, et ce chart sera la source du déploiement GitOps du Lab 15.

```text
                       chart countvisit (ce lab)
                 ┌──────────────────────────────────────────┐
 Ingress NGINX ─▶│ Ingress ─▶ Service ─▶ Deployment (Pods)   │ ──▶ pg-database-rw ──▶ PostgreSQL
                 │            ConfigMap   ServiceAccount, HPA │      (CloudNativePG, Lab 13)
                 └──────────────────────────────────────────┘
                     pg-secret : créé hors du chart (Lab 05)
```

La base de données reste **hors** du chart : elle a son propre cycle de vie, géré par l'Opérateur.

## Objectifs

1. Générer un chart avec `helm create` et comprendre sa structure : `Chart.yaml`, `values.yaml`,
   `templates/`, `_helpers.tpl`.
2. Remplacer l'application d'exemple par Countvisit, en paramétrant **dans `values.yaml`**.
3. Écrire des templates : ConfigMap, référence à un **Secret existant**, fonctions `include`,
   `quote`, `required`, `toYaml`.
4. Redémarrer automatiquement les Pods quand la configuration change (annotation de **checksum**).
5. Activer des composants à la demande (`if`) : HPA, Ingress ; tester la release (`helm test`).
6. Reprendre l'application existante sous la gestion de Helm, empaqueter le chart et le publier
   dans un registre **Harbor**.

**Documentation utile :** [Guide des templates](https://helm.sh/docs/chart_template_guide/getting_started/) · [Fonctions de template](https://helm.sh/docs/chart_template_guide/function_list/) · [Bonnes pratiques des charts](https://helm.sh/docs/chart_best_practices/) · [Astuces (checksum)](https://helm.sh/docs/howto/charts_tips_and_tricks/) · [Registres OCI](https://helm.sh/docs/topics/registries/) · [Harbor](https://goharbor.io/docs/)

---

## Partie 1 : L'échafaudage

**1.1 - Générer le chart**

Dans `~/labs/helm`, générez un chart nommé `countvisit` avec `helm create`. Affichez son
arborescence (`find countvisit -type f | sort`). À quoi sert chaque fichier ?

<details><summary>Correction</summary>

```bash
mkdir -p ~/labs/helm && cd ~/labs/helm
helm create countvisit
find countvisit -type f | sort
```

```text
countvisit/
├── .helmignore                   fichiers exclus de l'archive (comme un .gitignore)
├── Chart.yaml                    carte d'identité : nom, version du chart, version de l'application
├── values.yaml                   les paramètres par défaut, et leur documentation
├── charts/                       les sous-charts (dépendances), vide ici
└── templates/
    ├── _helpers.tpl              des fonctions réutilisables (noms, labels) : ne produit aucun objet
    ├── deployment.yaml
    ├── service.yaml
    ├── serviceaccount.yaml
    ├── ingress.yaml              désactivé par défaut
    ├── httproute.yaml            désactivé par défaut (versions récentes de Helm)
    ├── hpa.yaml                  désactivé par défaut
    ├── NOTES.txt                 le message affiché après install / upgrade
    └── tests/
        └── test-connection.yaml  un Pod de test, lancé par `helm test`
```

*(Selon la version de Helm, l'échafaudage peut légèrement différer.)*

L'échafaudage déploie un **nginx** d'exemple. Il est volontairement complet : on ne part pas d'une
page blanche, on retire et on adapte.

</details>

**1.2 - Le rendu de l'échafaudage**

Sans rien modifier, faites le rendu du chart avec un nom de release `demo`
(`helm template demo ./countvisit`). Quels objets sont produits ? Comment s'appelle le Deployment ?
Refaites le rendu avec le nom de release `countvisit` : comment s'appelle-t-il maintenant ?
Retrouvez dans `_helpers.tpl` la fonction responsable.

<details><summary>Correction</summary>

```bash
helm template demo ./countvisit | grep -E "^kind:|^  name:"
# kind: ServiceAccount   name: demo-countvisit
# kind: Service          name: demo-countvisit
# kind: Deployment       name: demo-countvisit
# kind: Pod              name: "demo-countvisit-test-connection"

helm template countvisit ./countvisit | grep -E "^kind:|^  name:"
# ... name: countvisit

grep -A12 'define "countvisit.fullname"' countvisit/templates/_helpers.tpl
```

La fonction `countvisit.fullname` construit le nom `<release>-<chart>`, **sauf** si le nom de la
release contient déjà celui du chart. Tous les templates l'appellent avec
`{{ include "countvisit.fullname" . }}` : on peut installer le chart plusieurs fois dans le même
namespace sans collision de noms.

`_helpers.tpl` définit aussi les labels standards (`countvisit.labels`, `countvisit.selectorLabels`) :
`app.kubernetes.io/name`, `app.kubernetes.io/instance`, `helm.sh/chart`...

</details>

**1.3 - Les objets du langage de template**

Ouvrez `templates/service.yaml` et `templates/deployment.yaml`. Identifiez d'où viennent les valeurs
de `{{ .Values... }}`, `{{ .Release.Name }}`, `{{ .Chart.AppVersion }}`. Que font `toYaml` et
`nindent` dans les blocs `{{- with .Values.resources }}` ?

<details><summary>Correction</summary>

| Objet | Contenu | Exemple |
| --- | --- | --- |
| `.Values` | `values.yaml`, fusionné avec `-f` et `--set` | `.Values.replicaCount` |
| `.Release` | la release en cours | `.Release.Name`, `.Release.Namespace`, `.Release.Revision` |
| `.Chart` | le contenu de `Chart.yaml` | `.Chart.Name`, `.Chart.AppVersion` |
| `.Template` | le template en cours | `.Template.BasePath` |

```yaml
          {{- with .Values.resources }}       # si resources n'est pas vide, "." devient resources
          resources:
            {{- toYaml . | nindent 12 }}       # convertit l'objet en YAML, indenté de 12 espaces
          {{- end }}
```

* `{{-` et `-}}` suppriment les espaces et retours à la ligne avant / après la balise ;
* `toYaml` transforme une structure de `values.yaml` en YAML ; `nindent 12` ajoute un retour à la
  ligne puis indente chaque ligne de 12 espaces. **L'indentation est la première source d'erreur**
  dans un template.

</details>

---

## Partie 2 : De nginx à Countvisit

**2.1 - La carte d'identité**

Modifiez `Chart.yaml` : description `Countvisit - application web de comptage de visites`,
version du chart `0.1.0`, version de l'application `v1` (le tag de l'image actuelle). Supprimez le
template `httproute.yaml` s'il existe : l'application est exposée par un Ingress (Lab 08).

<details><summary>Correction</summary>

```yaml
# Chart.yaml
apiVersion: v2
name: countvisit
description: Countvisit - application web de comptage de visites
type: application
version: 0.1.0          # version du CHART : à incrémenter à chaque modification du chart
appVersion: "v1"        # version de l'APPLICATION : sert de tag d'image par défaut
```

```bash
rm -f countvisit/templates/httproute.yaml
```

Ces deux versions sont celles des colonnes `CHART VERSION` et `APP VERSION` de `helm search` (Lab 09).

</details>

**2.2 - Les valeurs de Countvisit**

Modifiez `values.yaml` pour décrire Countvisit (laissez en place les clés que vous ne modifiez pas) :

* 3 réplicas ;
* image `public.ecr.aws/wizetraining/webapp-count-secure`, **tag vide** (`""`) ;
* une nouvelle clé `containerPort: 5000` (le port d'écoute de l'application) ;
* le ServiceAccount ne doit pas monter de token d'API (Lab 12) ;
* les sondes : liveness sur `/healthz`, readiness sur `/readyz`, port nommé `http` (Lab 07) ;
* requests `50m` / `64Mi`, limits `250m` / `256Mi` (Lab 11).

Pourquoi laisser le tag vide ?

<details><summary>Correction</summary>

```yaml
# values.yaml (clés modifiées ou ajoutées)
replicaCount: 3

image:
  repository: public.ecr.aws/wizetraining/webapp-count-secure
  pullPolicy: IfNotPresent
  tag: ""                 # vide : le template utilise .Chart.AppVersion (v1)

containerPort: 5000       # nouvelle clé

serviceAccount:
  create: true
  automount: false        # Countvisit n'appelle pas l'API Kubernetes
  annotations: {}
  name: ""

service:
  type: ClusterIP
  port: 80

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 250m
    memory: 256Mi

livenessProbe:
  httpGet:
    path: /healthz
    port: http
readinessProbe:
  httpGet:
    path: /readyz
    port: http
```

Le template du Deployment contient `image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"`.
Avec un tag vide, c'est `appVersion` qui s'applique : la version de l'application est déclarée **une
seule fois**, dans `Chart.yaml`, et on la surcharge ponctuellement avec `--set image.tag=...`.

</details>

**2.3 - 🩺 Le port du conteneur**

Faites le rendu du chart et regardez la section `ports` du conteneur. Sur quel port le conteneur
est-il déclaré ? Pourquoi les sondes échoueraient-elles ? Corrigez le template `deployment.yaml` pour
utiliser la nouvelle clé `containerPort`, puis vérifiez le rendu et la syntaxe (`helm lint`).

<details><summary>Correction</summary>

```bash
helm template countvisit ./countvisit | grep -B2 -A3 "containerPort"
#   - name: http
#     containerPort: 80          ← l'échafaudage réutilise service.port
```

Countvisit écoute sur **5000**. Le port nommé `http` désignerait 80 : les sondes `port: http`
interrogeraient un port fermé, et les Pods redémarreraient en boucle.

```yaml
# templates/deployment.yaml
          ports:
            - name: http
              containerPort: {{ .Values.containerPort }}
              protocol: TCP
```

```bash
helm template countvisit ./countvisit | grep -A2 "name: http"      # containerPort: 5000
helm lint ./countvisit                                             # 0 chart(s) failed
```

Le Service (`templates/service.yaml`) n'a pas à changer : il publie le port 80 et vise
`targetPort: http`, le **nom** du port du conteneur. C'est l'intérêt des ports nommés.

</details>

**2.4 - Première installation**

Installez le chart dans le namespace `countvisit`, avec le nom de release `countvisit`. Les objets
créés entrent-ils en conflit avec ceux que vous avez créés à la main (`webapp`, `webapp-config`) ?
Suivez le démarrage des Pods, puis ouvrez l'application avec un `port-forward` vers le Service
`countvisit` (port local 8080 ; dans Cloud Shell, bouton **Aperçu sur le Web** → port 8080). Que
s'affiche-t-il ? Lisez les logs d'un Pod.

<details><summary>Correction</summary>

```bash
cd ~/labs/helm
helm install countvisit ./countvisit -n countvisit
helm list -n countvisit
kubectl get deploy,svc,sa -n countvisit -l app.kubernetes.io/instance=countvisit
kubectl get pods -n countvisit -l app.kubernetes.io/instance=countvisit -w

kubectl port-forward -n countvisit svc/countvisit 8080:80
# navigateur : http://localhost:8080 (ou Aperçu sur le Web)
kubectl logs -n countvisit deploy/countvisit          # après avoir chargé la page
# ... Configuration PostgreSQL: postgres:5432 (User: postgres, DB: postgres)
# ... WARNING - Secret introuvable dans /etc/app/secrets/pg_password. Tentative de connexion sans mot de passe.
```

Pas de conflit : les objets du chart s'appellent `countvisit` et portent les labels
`app.kubernetes.io/*`, alors que les vôtres s'appellent `webapp` et portent `app: webapp`. Les deux
versions de l'application cohabitent dans le namespace.

Les Pods sont `Running` et `Ready` (la readiness de Countvisit ne dépend pas de la base), mais la
page indique un échec de connexion : l'application utilise ses valeurs **par défaut** (`PG_HOST`
= `postgres`, un Service qui n'existe plus depuis le Lab 13) et n'a pas de mot de passe. Il manque la
**configuration** : c'est l'objet de la partie suivante.

</details>

---

## Partie 3 : La configuration

**3.1 - Un ConfigMap généré par le chart**

Ajoutez à `values.yaml` une section `config` avec `pgHost: pg-database-rw`, `pgPort: 5432` et
`pgDb: app_db`. Écrivez `templates/configmap.yaml` : un ConfigMap nommé `<fullname>-config`, portant
les labels standards du chart, avec les clés `PG_HOST`, `PG_PORT` et `PG_DB`. Ajoutez au conteneur du
Deployment un `envFrom` vers ce ConfigMap. Vérifiez le rendu.

<details><summary>Correction</summary>

```yaml
# values.yaml (ajout)
config:
  pgHost: pg-database-rw     # le Service du primaire CloudNativePG (Lab 13)
  pgPort: 5432
  pgDb: app_db
```

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "countvisit.fullname" . }}-config
  labels:
    {{- include "countvisit.labels" . | nindent 4 }}
data:
  PG_HOST: {{ .Values.config.pgHost | quote }}
  PG_PORT: {{ .Values.config.pgPort | quote }}
  PG_DB: {{ .Values.config.pgDb | quote }}
```

```yaml
# templates/deployment.yaml (dans le conteneur, après imagePullPolicy)
          envFrom:
            - configMapRef:
                name: {{ include "countvisit.fullname" . }}-config
```

```bash
helm template countvisit ./countvisit -s templates/configmap.yaml
```

🔎 **Attention** - sans `quote`, `PG_PORT: 5432` serait un **nombre**, et l'API refuserait le
ConfigMap : les valeurs d'un ConfigMap sont toujours des chaînes (Lab 05). `quote` est le réflexe
pour toute valeur insérée dans un champ texte.

</details>

**3.2 - Référencer un Secret existant**

Le mot de passe ne doit **jamais** apparaître dans `values.yaml` : ce fichier finira dans Git (Lab 15).
Le chart va donc **référencer** un Secret créé hors du chart.

1. Ajoutez à `values.yaml` une clé `existingSecret: pg-secret`.
2. Dans le Deployment : la variable `PG_USER` vient de la clé `username` de ce Secret ; la clé
   `password` est montée comme fichier `/etc/app/secrets/pg_password` (comme au Lab 05).
3. Le nom du Secret est obligatoire : utilisez la fonction `required` pour que le rendu échoue avec
   un message clair s'il est vide. Testez avec `--set existingSecret=`.

<details><summary>Correction</summary>

```yaml
# values.yaml (ajout)
# Secret de type kubernetes.io/basic-auth (clés username et password), créé HORS du chart
existingSecret: pg-secret
```

```yaml
# templates/deployment.yaml (dans le conteneur, après envFrom)
          env:
            - name: PG_USER
              valueFrom:
                secretKeyRef:
                  name: {{ required "existingSecret est obligatoire (Secret basic-auth de PostgreSQL)" .Values.existingSecret }}
                  key: username
```

Remplacez ensuite les blocs génériques `{{- with .Values.volumeMounts }}` (dans le conteneur) et
`{{- with .Values.volumes }}` (dans le Pod) par :

```yaml
          volumeMounts:
            - name: pg-password
              mountPath: /etc/app/secrets
              readOnly: true
```

```yaml
      volumes:
        - name: pg-password
          secret:
            secretName: {{ .Values.existingSecret }}
            items:
              - key: password
                path: pg_password
```

```bash
helm template countvisit ./countvisit -s templates/deployment.yaml | less
helm template countvisit ./countvisit --set existingSecret= > /dev/null
# Error: execution error at (countvisit/templates/deployment.yaml:...): existingSecret est obligatoire ...
```

Les clés `volumes` et `volumeMounts` de `values.yaml` ne servent plus : supprimez-les pour ne pas
induire l'utilisateur du chart en erreur.

💡 **Bon réflexe** - le motif `existingSecret` est celui de la plupart des charts publics (Bitnami,
Elastic...) : le chart **consomme** un Secret, il ne le **contient** pas. En production, le Secret est
créé par un outil dédié : External Secrets Operator alimenté par **Secret Manager** sur GKE,
Sealed Secrets, SOPS...

</details>

**3.3 - Mettre à jour la release**

Appliquez vos modifications à la release (`helm upgrade`). Vérifiez les variables d'environnement d'un
Pod, le fichier de mot de passe, et la page de l'application (relancez le `port-forward`). Quelle
valeur affiche le compteur ? Pourquoi ?

<details><summary>Correction</summary>

```bash
helm upgrade countvisit ./countvisit -n countvisit
helm history countvisit -n countvisit             # révision 2
kubectl rollout status -n countvisit deploy/countvisit

kubectl exec -n countvisit deploy/countvisit -- env | grep ^PG_
# PG_HOST=pg-database-rw  PG_PORT=5432  PG_DB=app_db  PG_USER=app_user
kubectl exec -n countvisit deploy/countvisit -- ls /etc/app/secrets     # pg_password

kubectl port-forward -n countvisit svc/countvisit 8080:80
```

« Connecté à PostgreSQL (Sécurisé) », et le compteur **continue** celui de l'application déployée à
la main : les deux versions utilisent la **même base** (`pg-database`). C'est la preuve que la
version Helm est prête à remplacer l'ancienne.

</details>

**3.4 - 🩺 La configuration change, les Pods ne suivent pas**

Mettez à jour la release en surchargeant `config.pgDb` avec la valeur `base_inexistante`. La release
passe en révision 3 : vérifiez `PG_DB` dans le ConfigMap, puis **dans un Pod**. Que constatez-vous ?

<details><summary>Correction</summary>

```bash
helm upgrade countvisit ./countvisit -n countvisit --set config.pgDb=base_inexistante
kubectl get cm countvisit-config -n countvisit -o jsonpath='{.data.PG_DB}'; echo     # base_inexistante
kubectl exec -n countvisit deploy/countvisit -- printenv PG_DB                         # app_db
kubectl get pods -n countvisit -l app.kubernetes.io/instance=countvisit             # aucun Pod recréé
```

Le ConfigMap a changé, mais le **template de Pod** du Deployment est identique : Kubernetes n'a
aucune raison de lancer un rolling update, et une variable d'environnement n'est lue qu'au démarrage
(Lab 05). La release dit `deployed`, mais l'application tourne avec l'**ancienne** configuration :
une dérive silencieuse, qui éclatera au prochain redémarrage d'un Pod.

</details>

**3.5 - L'annotation de checksum**

Ajoutez au **template de Pod** du Deployment (`spec.template.metadata.annotations`) une annotation
`checksum/config` qui contient l'empreinte SHA-256 du rendu de `configmap.yaml`, en conservant la
possibilité d'ajouter des `podAnnotations`. Relancez le même `upgrade`. Que se passe-t-il cette fois ?
Réparez ensuite la production avec `helm rollback`, vers la révision 2.

<details><summary>Correction</summary>

```yaml
# templates/deployment.yaml
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
```

```bash
helm upgrade countvisit ./countvisit -n countvisit --set config.pgDb=base_inexistante
kubectl get pods -n countvisit -l app.kubernetes.io/instance=countvisit -w     # rolling update
kubectl exec -n countvisit deploy/countvisit -- printenv PG_DB                    # base_inexistante
# la page affiche un échec de connexion : la base n'existe pas

helm history countvisit -n countvisit
helm rollback countvisit 2 -n countvisit
kubectl exec -n countvisit deploy/countvisit -- printenv PG_DB                    # app_db
```

Le contenu du ConfigMap fait désormais partie du template de Pod (via son empreinte) : toute
modification de la configuration change le template, donc déclenche un **rolling update**. C'est
l'équivalent Helm du suffixe de hachage des générateurs de Kustomize.

🔎 **Attention** - la révision 2 a été rendue **sans** l'annotation : le rollback crée une
révision 5 sans checksum. Ce n'est pas grave ici, mais rappelez-vous qu'un rollback rejoue
**l'ancien template**, pas seulement les anciennes valeurs. Le prochain `helm upgrade` réintroduira
l'annotation.

</details>

---

## Partie 4 : Un chart modulable

**4.1 - L'autoscaling à la demande**

Lisez `templates/hpa.yaml` et la section `spec.replicas` de `templates/deployment.yaml`. Que se passe-t-il
si `autoscaling.enabled` vaut `true` ? Faites le rendu avec l'autoscaling activé entre 3 et 6
réplicas, à 50 % de CPU, et vérifiez la cohérence avec la règle vue au Lab 11.

<details><summary>Correction</summary>

```bash
helm template countvisit ./countvisit \
  --set autoscaling.enabled=true \
  --set autoscaling.minReplicas=3 --set autoscaling.maxReplicas=6 \
  --set autoscaling.targetCPUUtilizationPercentage=50 \
  | grep -E "^kind:|replicas:|Replicas:|averageUtilization"
# kind: HorizontalPodAutoscaler ... minReplicas: 3, maxReplicas: 6, averageUtilization: 50
# kind: Deployment             ... aucune ligne "replicas:"
```

```yaml
# templates/deployment.yaml (échafaudage)
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
```

Quand le HPA est activé, le chart **omet** `replicas` du Deployment : c'est exactement la règle du
Lab 11 (« avec un HPA, retirez `replicas` du manifeste »), appliquée automatiquement. Un `if` dans un
template permet de livrer un composant optionnel, désactivé par défaut.

Mettez à jour `values.yaml` pour documenter des valeurs par défaut sensées (`minReplicas: 3`,
`maxReplicas: 6`, `targetCPUUtilizationPercentage: 50`), en laissant `enabled: false`.

</details>

**4.2 - Le test de la release**

Ouvrez `templates/tests/test-connection.yaml`. Que fait ce Pod, et quand est-il lancé ? Adaptez-le :
image `public.ecr.aws/wizetraining/busybox:latest`, et requête sur le chemin `/readyz` du Service.
Mettez à jour la release, puis lancez `helm test`.

<details><summary>Correction</summary>

```yaml
# templates/tests/test-connection.yaml (extrait)
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: wget
      image: public.ecr.aws/wizetraining/busybox:latest
      command: ['wget']
      args: ['-qO-', '{{ include "countvisit.fullname" . }}:{{ .Values.service.port }}/readyz']
  restartPolicy: Never
```

```bash
helm upgrade countvisit ./countvisit -n countvisit
helm test countvisit -n countvisit --logs
# Phase: Succeeded  ... {"status":"ready","ui":"available"}
```

L'annotation `helm.sh/hook: test` fait de ce Pod un **hook** : il n'est pas créé par `install` ni par
`upgrade`, seulement par `helm test`. C'est un contrôle de bon fonctionnement après déploiement,
utile en CI/CD (là où un `deployed` ne garantit rien, Lab 09).

</details>

**4.3 - Le message d'après installation**

Remplacez le contenu de `templates/NOTES.txt` par un message qui affiche le nom de la release, la
version de l'application et, si l'Ingress est activé, l'URL de chaque hôte ; sinon, la commande de
`port-forward`. Vérifiez avec `helm get notes`.

<details><summary>Correction</summary>

```text
Countvisit {{ .Chart.AppVersion }} est déployée (release {{ .Release.Name }}, révision {{ .Release.Revision }}).
{{- if .Values.ingress.enabled }}
{{- range .Values.ingress.hosts }}
  URL : http{{ if $.Values.ingress.tls }}s{{ end }}://{{ .host }}/
{{- end }}
{{- else }}
  Accès local :
    kubectl port-forward -n {{ .Release.Namespace }} svc/{{ include "countvisit.fullname" . }} 8080:{{ .Values.service.port }}
{{- end }}

La base de données est attendue sur {{ .Values.config.pgHost }}:{{ .Values.config.pgPort }}
(Secret {{ .Values.existingSecret }}).
```

```bash
helm upgrade countvisit ./countvisit -n countvisit
helm get notes countvisit -n countvisit
```

`$` désigne la racine du contexte : à l'intérieur d'un `range`, `.` désigne l'élément courant, et
`$.Values` permet de revenir aux valeurs globales.

</details>

---

## Partie 5 : Reprendre la production

La version Helm fonctionne en parallèle de la version manuelle. Il reste à lui confier l'Ingress
public, puis à supprimer les objets créés à la main.

**5.1 - Un fichier de valeurs pour la production**

Créez `~/labs/helm/values-prod.yaml` qui active l'Ingress du chart avec la configuration du Lab 08 :

* la classe `nginx` ;
* l'hôte `$HOST_APP`, chemin `/`, type `Prefix` ;
* TLS avec le Secret `countvisit-tls` pour cet hôte.

Faites le rendu de l'Ingress avec ce fichier et comparez-le à `~/labs/countvisit/webapp-ingress.yaml`.

<details><summary>Correction</summary>

```bash
cd ~/labs/helm
cat > values-prod.yaml <<EOF
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: ${HOST_APP}
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: countvisit-tls
      hosts:
        - ${HOST_APP}
EOF

helm template countvisit ./countvisit -f values-prod.yaml -s templates/ingress.yaml
```

Le template `ingress.yaml` de l'échafaudage convient tel quel : tout ce qui varie d'un environnement
à l'autre (classe, annotations, nom d'hôte, certificat) passe par les **valeurs**. Le même chart,
avec un autre fichier de valeurs, s'installerait sur un autre cluster avec un autre contrôleur.

</details>

**5.2 - La bascule**

Ingress NGINX refuse que deux Ingress déclarent le **même hôte et le même chemin**. Supprimez
l'Ingress `webapp` créé à la main, puis mettez à jour la release avec `values-prod.yaml`, et testez
`https://$HOST_APP`. Quelles sont les deux options possibles pour **ajouter** un fichier de valeurs
sans perdre les surcharges précédentes ?

<details><summary>Correction</summary>

```bash
kubectl delete ingress webapp -n countvisit
helm upgrade countvisit ./countvisit -n countvisit -f values-prod.yaml
helm get notes countvisit -n countvisit             # l'URL s'affiche

kubectl get ingress -n countvisit                   # countvisit   nginx   countvisit.<IP>.nip.io   <INGRESS_IP>
curl -sk https://$HOST_APP | grep -c Bonjour
curl -sI http://$HOST_APP | head -1                 # 308 : redirection HTTPS
```

L'URL ne change pas : le point d'entrée est toujours le contrôleur NGINX, seule la règle a changé de
propriétaire.

À chaque `helm upgrade`, Helm repart des valeurs **du chart** plus celles que vous passez : une
surcharge oubliée est perdue (Lab 09). Pour ne rien perdre :

* repasser **tous** les fichiers à chaque fois (`-f values-prod.yaml -f ...`) : c'est la méthode
  recommandée, reproductible et versionnable ;
* ou `--reuse-values` / `--reset-then-reuse-values`, qui repartent des valeurs de la révision
  précédente : pratique, mais l'état réel n'est plus décrit par vos fichiers.

</details>

**5.3 - Supprimer l'ancienne version**

Supprimez le Deployment et le Service `webapp` créés à la main, ainsi que le site `static-site` du
Lab 08. Pourquoi faut-il **conserver** le ConfigMap `webapp-config` ? Vérifiez que l'application
fonctionne toujours, puis listez tout ce que la release gère.

<details><summary>Correction</summary>

```bash
kubectl delete deployment webapp static-site -n countvisit
kubectl delete service webapp static-site -n countvisit
kubectl get cronjob pg-backup -n countvisit -o yaml | grep -B3 webapp-config     # configMapKeyRef
curl -sk https://$HOST_APP | grep -c Bonjour

helm get manifest countvisit -n countvisit | grep -E "^kind:"
kubectl get all,cm,ingress,sa -n countvisit -l app.kubernetes.io/instance=countvisit
```

Le CronJob `pg-backup` (Lab 05) lit toujours sa configuration dans `webapp-config` : un objet qui
semble inutilisé ne l'est pas forcément. Vérifiez qui le référence avant de supprimer.

À ce stade, l'application web est entièrement décrite par le chart et deux fichiers de valeurs. Le
namespace contient encore des objets hors chart, chacun pour une bonne raison : `pg-secret`
(secret), `pg-database` (géré par l'Opérateur), `pg-backup` et `webapp-config` (sauvegarde).

</details>

**5.4 - Empaqueter**

Passez la version du chart à `0.2.0`, vérifiez-le avec `helm lint` (en lui passant aussi
`values-prod.yaml`), puis créez l'archive avec `helm package`. Que contient-elle ?

<details><summary>Correction</summary>

```bash
sed -i 's/^version: 0.1.0/version: 0.2.0/' countvisit/Chart.yaml     # macOS : sed -i ''
helm lint ./countvisit -f values-prod.yaml
helm package ./countvisit                    # countvisit-0.2.0.tgz
tar tzf countvisit-0.2.0.tgz
```

L'archive contient le chart, sans les fichiers listés dans `.helmignore`. Elle peut être publiée dans
un dépôt de charts ou dans un registre **OCI**, et installée par son nom et sa version, comme
`bitnami/nginx` au Lab 09.

</details>

**5.5 - Publier le chart dans Harbor**

Le formateur a mis à votre disposition un registre **Harbor** : `harbor.wizetraining.com`, avec un
**projet** et des identifiants qu'il vous communique. Connectez Helm au registre, poussez l'archive
dans le chemin `oci://harbor.wizetraining.com/<projet>/<prénom>` (pour ne pas écraser le chart d'un
autre participant), puis vérifiez la publication : métadonnées du chart, et rendu directement depuis
le registre.

<details><summary>Correction</summary>

```bash
helm registry login harbor.wizetraining.com -u <utilisateur>       # le mot de passe est demandé

helm push countvisit-0.2.0.tgz oci://harbor.wizetraining.com/<projet>/<prénom>
# Pushed: harbor.wizetraining.com/<projet>/<prénom>/countvisit:0.2.0

helm show chart oci://harbor.wizetraining.com/<projet>/<prénom>/countvisit --version 0.2.0
helm template test oci://harbor.wizetraining.com/<projet>/<prénom>/countvisit --version 0.2.0 | head
```

Le chart apparaît aussi dans l'interface web de Harbor (`https://harbor.wizetraining.com`), dans le
projet, comme un *artefact* de type chart Helm.

Un registre **OCI** stocke charts et images de conteneurs au même endroit, avec les mêmes droits et
les mêmes outils (analyse de vulnérabilités, signature...). Une installation se ferait directement
depuis le registre :
`helm install countvisit oci://harbor.wizetraining.com/<projet>/<prénom>/countvisit --version 0.2.0 -f values-prod.yaml`.

🔎 **Attention** - une version publiée ne se republie pas : pour toute modification du chart, on
incrémente `version` dans `Chart.yaml`. C'est ce qui rend une installation reproductible.

</details>

---

## ✅ État attendu en fin de lab

* `helm list -n countvisit` affiche la release `countvisit`, `deployed` ;
* `https://$HOST_APP` sert l'application via l'Ingress `countvisit` du chart ;
* plus de Deployment, Service ni Ingress `webapp`, plus de `static-site` ; `webapp-config` et
  `pg-backup` sont conservés ;
* `~/labs/helm/countvisit/` (chart en version `0.2.0`) et `~/labs/helm/values-prod.yaml` existent :
  ils seront poussés dans Git au Lab 15.

---

## Points clés

* `helm create` fournit un échafaudage complet : on **retire** et on **adapte** plutôt que d'écrire
  de zéro. `_helpers.tpl` centralise noms (`fullname`) et labels.
* Tout ce qui varie d'un environnement à l'autre va dans **`values.yaml`** ; un fichier de valeurs
  par environnement (`values-prod.yaml`) plutôt que des `--set`.
* `Chart.yaml` : `version` (le chart) ≠ `appVersion` (l'application, tag d'image par défaut).
* Fonctions essentielles : `include`, `quote`, `required`, `default`, `toYaml | nindent`, `with`,
  `if`, `range` (et `$` pour revenir à la racine).
* Un chart **référence** les Secrets (`existingSecret`), il ne les contient pas.
* L'annotation **`checksum/config`** déclenche un rolling update quand la configuration change.
* `helm template -s <fichier>` et `helm lint` pour mettre au point ; `helm test` pour vérifier une
  release ; `helm package` / `helm push` pour distribuer.
