# LAB 07 - Sondes de santé (Probes)

> **Cluster : GKE** (contexte `cluster-lab-dkdc`) | **Durée : ~40 min**
> **Prérequis :** Lab 06 terminé (namespace `countvisit`, Deployment `webapp` avec l'image
> `webapp-count-secure:v1`, StatefulSet `postgres`).

Sans sonde, Kubernetes n'a qu'un critère de santé : « le processus principal tourne-t-il encore ? ».
Une application bloquée, ou qui démarre sans être prête, passe inaperçue. Les **sondes** permettent
à l'application de dire elle-même si elle va bien.

L'image `webapp-count-secure:v1` expose deux routes prévues pour cela (voir la fiche du Lab 05) :

* `GET /healthz` → `200` si le processus est vivant (liveness) ;
* `GET /readyz` → `200` si l'application peut recevoir du trafic (readiness).

| Sonde | Question posée | Si elle échoue |
| --- | --- | --- |
| **Startup** | A-t-elle fini de démarrer ? | redémarrage ; les deux autres sondes sont **désactivées** tant qu'elle n'a pas réussi |
| **Readiness** | Peut-elle servir du trafic ? | le Pod reste en vie mais est **retiré des endpoints du Service** |
| **Liveness** | Est-elle encore vivante ? | le conteneur est **redémarré** |

Types de vérification : `httpGet`, `tcpSocket`, `exec`, `grpc`.

## Objectifs

1. Configurer une **readiness probe** et constater qu'elle protège les mises à jour et le trafic.
2. Configurer une **liveness probe** pour l'auto-réparation, et en mesurer les risques.
3. Utiliser une **startup probe** pour les démarrages lents.
4. Diagnostiquer un Pod qui redémarre en boucle ou qui n'est jamais `Ready`.
5. Inspecter un conteneur sans outils avec `kubectl debug` et un **conteneur éphémère**.

**Documentation utile :** [Configurer les sondes liveness, readiness et startup](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) · [Cycle de vie d'un Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) · [Débugger un Pod en cours d'exécution](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)

---

## Partie 1 : Readiness - ne pas envoyer de trafic à un Pod qui n'est pas prêt

**1.1 - Ajouter la sonde**

Ajoutez au conteneur `webapp` de `webapp-deployment.yaml` une **readinessProbe** HTTP sur `/readyz`,
port 5000, délai initial 3 s, période 5 s. Appliquez et vérifiez que les Pods passent `1/1` Ready.

<details><summary>Correction</summary>

```yaml
# webapp-deployment.yaml (conteneur webapp)
        readinessProbe:
          httpGet:
            path: /readyz
            port: 5000
          initialDelaySeconds: 3
          periodSeconds: 5
```

```bash
kubectl apply -f webapp-deployment.yaml
kubectl rollout status deployment/webapp
kubectl get pods -l app=webapp          # READY 1/1
```

💡 **Astuce** - ne mémorisez pas la syntaxe : `kubectl explain pod.spec.containers.readinessProbe`
ou la page *Configure Liveness, Readiness and Startup Probes* de la documentation.

</details>

**1.2 - 🩺 Casser volontairement la readiness**

Changez le chemin de la sonde en `/inexistant` et ré-appliquez. Observez avec `kubectl get pods`,
`kubectl rollout status deployment/webapp --timeout=45s` et `kubectl describe` sur un **nouveau** Pod :

* dans quel état sont les nouveaux Pods ? Et les anciens ?
* l'application reste-t-elle accessible dans le navigateur ?
* quel message d'erreur trouvez-vous dans les événements ?
* quels Pods figurent dans les endpoints du Service `webapp` ?

<details><summary>Correction</summary>

```bash
kubectl apply -f webapp-deployment.yaml
kubectl get pods -l app=webapp
```

```text
webapp-6d8f...-a1b2c   1/1   Running   ← anciens Pods : toujours en service
webapp-6d8f...-d3e4f   1/1   Running
webapp-6d8f...-g5h6i   1/1   Running
webapp-7c9e...-j7k8l   0/1   Running   ← nouveau Pod : vivant mais jamais Ready
```

```bash
kubectl rollout status deployment/webapp --timeout=45s       # bloqué, puis timeout
kubectl describe pod <nouveau-pod> | grep -i readiness
# Readiness probe failed: HTTP probe failed with statuscode: 404
kubectl get endpointslices -l kubernetes.io/service-name=webapp -o wide   # seulement les anciens Pods
```

L'application **reste accessible** : un Pod qui n'est jamais `Ready` ne reçoit pas de trafic, et le
rolling update ne retire pas les anciens Pods tant que les nouveaux ne sont pas prêts.

Sans readiness probe, Kubernetes aurait considéré les nouveaux Pods comme prêts dès le démarrage du
processus, et basculé le trafic vers une version cassée.

🔎 **Attention** - un Pod `Running` mais `0/1` n'est **pas** en panne au sens de Kubernetes : il est
vivant, simplement exclu du Service. C'est aussi l'explication classique d'un Service dont les
endpoints sont vides alors que tous ses Pods sont `Running` (Lab 04).

</details>

**1.3 - Réparer**

Revenez à la configuration saine.

<details><summary>Correction</summary>

```bash
kubectl rollout undo deployment/webapp
# puis remettre path: /readyz dans le fichier, pour qu'il reste la source de vérité
kubectl apply -f webapp-deployment.yaml
```

</details>

**1.4 - Bonus : readiness sur PostgreSQL**

Ajoutez au StatefulSet `postgres` une readiness probe de type `exec` qui lance
`pg_isready -U app_user -d app_db`, période 5 s.

<details><summary>Correction</summary>

```yaml
# postgres-statefulset.yaml (conteneur postgres)
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "app_user", "-d", "app_db"]
          periodSeconds: 5
```

```bash
kubectl apply -f postgres-statefulset.yaml
kubectl rollout status statefulset/postgres
```

Un `tcpSocket` sur le port 5432 vérifierait seulement que le port est ouvert ; `pg_isready` vérifie
que PostgreSQL accepte réellement des connexions.

</details>

---

## Partie 2 : Liveness - redémarrer un conteneur bloqué

**2.1 - Ajouter la sonde**

Ajoutez au conteneur `webapp` une **livenessProbe** HTTP sur `/healthz`, port 5000, délai initial
10 s, période 10 s. Appliquez.

<details><summary>Correction</summary>

```yaml
        livenessProbe:
          httpGet:
            path: /healthz
            port: 5000
          initialDelaySeconds: 10
          periodSeconds: 10
```

```bash
kubectl apply -f webapp-deployment.yaml
kubectl rollout status deployment/webapp
```

</details>

**2.2 - 🩺 Casser volontairement la liveness**

Mettez le chemin `/inexistant`, `periodSeconds: 5` et `failureThreshold: 2`, puis appliquez.
Observez la colonne `RESTARTS` (`kubectl get pods -w`) et les événements :

* au bout de combien de temps survient le premier redémarrage ?
* quelle différence de comportement avec la readiness en échec de la Partie 1 ?
* que montre `kubectl logs --previous` sur un Pod redémarré ?

<details><summary>Correction</summary>

```bash
kubectl apply -f webapp-deployment.yaml
kubectl get pods -l app=webapp -w            # RESTARTS augmente, puis CrashLoopBackOff
kubectl describe pod <pod> | grep -iE "liveness|Killing"
# Liveness probe failed: HTTP probe failed with statuscode: 404
# Container webapp failed liveness probe, will be restarted
kubectl logs <pod> --previous | tail -3      # les dernières lignes avant le redémarrage
```

Premier redémarrage après environ `initialDelaySeconds + failureThreshold × periodSeconds`,
soit ~20 s. La readiness en échec **retire du trafic** sans toucher au conteneur ; la liveness en
échec **tue et redémarre** le conteneur.

⚠️ Une liveness trop agressive est **pire que pas de liveness** : sous forte charge, une
application lente échoue à la sonde, est redémarrée, reprend du retard au démarrage, échoue de
nouveau. Ne faites **jamais** dépendre une liveness d'un service externe (une base de données
indisponible ferait redémarrer toute l'application en cascade).

</details>

**2.3 - Réparer**

Remettez `/healthz`, `periodSeconds: 10` et supprimez `failureThreshold`. Gardez cette configuration
saine pour la suite.

<details><summary>Correction</summary>

```bash
kubectl apply -f webapp-deployment.yaml
kubectl rollout status deployment/webapp
kubectl get pods -l app=webapp               # 1/1, RESTARTS remis à 0 sur les nouveaux Pods
```

</details>

---

## Partie 3 : Startup - protéger un démarrage lent

Une application qui met 30 secondes à démarrer, et dont la liveness vérifie toutes les 5 secondes,
est tuée avant d'être prête, en boucle.

**3.1 - Le problème**

Créez un Pod `slow-app` (busybox ECR) qui exécute `sh -c "sleep 30; touch /tmp/started; sleep 3600"`,
avec une **livenessProbe** de type `exec` (`cat /tmp/started`), période 5 s, `failureThreshold: 1`.
Observez-le pendant une minute : que se passe-t-il ?

<details><summary>Correction</summary>

```yaml
# slow-app.yaml
apiVersion: v1
kind: Pod
metadata:
  name: slow-app
spec:
  containers:
  - name: slow-app
    image: public.ecr.aws/wizetraining/busybox:latest
    args: ["sh", "-c", "sleep 30; touch /tmp/started; sleep 3600"]
    livenessProbe:
      exec:
        command: ["cat", "/tmp/started"]
      periodSeconds: 5
      failureThreshold: 1
```

```bash
kubectl apply -f slow-app.yaml
kubectl get pod slow-app -w
```

Le conteneur est redémarré avant la fin de ses 30 s de démarrage, en boucle : `RESTARTS` augmente,
puis `CrashLoopBackOff`. L'application ne démarre **jamais**.

</details>

**3.2 - La solution**

Supprimez le Pod et recréez-le avec, en plus, une **startupProbe** utilisant la même commande,
`periodSeconds: 2`, `failureThreshold: 30`. Que se passe-t-il maintenant ?

<details><summary>Correction</summary>

```yaml
    startupProbe:
      exec:
        command: ["cat", "/tmp/started"]
      periodSeconds: 2
      failureThreshold: 30      # jusqu'à 30 × 2 = 60 s pour démarrer
```

```bash
kubectl delete pod slow-app
kubectl apply -f slow-app.yaml
kubectl get pod slow-app -w      # aucun redémarrage
```

Tant que la startup probe n'a pas réussi **une fois**, la liveness et la readiness sont désactivées.
On garde ainsi une liveness réactive en régime établi, tout en laissant à l'application le temps de
démarrer.

Gardez le fichier `slow-app.yaml` : il resservira en Partie 4.

🧹 `kubectl delete pod slow-app`

</details>

---

## Partie 4 : 🩺 Voir ce que voit la sonde - `kubectl debug`

Une sonde échoue et vous voulez vérifier **depuis l'intérieur du Pod** ce que reçoit le kubelet.
Problème : les images de production sont minimalistes, et volontairement dépourvues d'outils de
diagnostic (`curl`, `ps`, `netstat`...), pour réduire leur surface d'attaque.

**4.1 - Le constat**

Choisissez un Pod `webapp` et essayez d'y exécuter, avec `kubectl exec`, les commandes `curl`,
`ps` et `netstat`. Lesquelles sont disponibles ?

<details><summary>Correction</summary>

```bash
POD=$(kubectl get pod -l app=webapp -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- curl -s localhost:5000/healthz
kubectl exec $POD -- ps aux
kubectl exec $POD -- netstat -tln
# ... executable file not found in $PATH
```

Les outils absents varient selon l'image, mais le principe est général : on ne peut pas compter
sur ce qui se trouve dans le conteneur. Et il ne faut **pas** modifier l'image pour y ajouter des
outils de diagnostic.

</details>

**4.2 - Un conteneur éphémère**

Avec `kubectl debug`, attachez au Pod `webapp` un conteneur temporaire basé sur l'image busybox
ECR, ciblant le conteneur `webapp` (option `--target`). Depuis ce conteneur :

* interrogez `/healthz` et `/readyz` sur `localhost:5000`, comme le fait le kubelet ;
* listez les ports en écoute ;
* listez les processus. Voyez-vous celui de l'application ?

<details><summary>Correction</summary>

```bash
kubectl debug -it $POD --image=public.ecr.aws/wizetraining/busybox:latest --target=webapp -- sh

# Dans le conteneur éphémère :
wget -qO- localhost:5000/healthz      # {"status":"alive"}
wget -qO- localhost:5000/readyz       # {"status":"ready","ui":"available"}
netstat -tln                          # 0.0.0.0:5000 LISTEN
ps                                    # le processus python de l'application
exit
```

Le conteneur éphémère partage le **réseau** du Pod (même IP, même `localhost`) et, grâce à
`--target`, l'espace des **processus** du conteneur visé. Il ne redémarre pas le Pod et ne modifie
pas son `spec.containers`.

```bash
kubectl get pod $POD -o jsonpath='{.spec.ephemeralContainers[*].name}'; echo
kubectl describe pod $POD | grep -A3 "Ephemeral Containers"
```

Un conteneur éphémère ne peut **pas** être retiré : il disparaît avec le Pod. Sur un Deployment, il
suffit de supprimer le Pod pour revenir à l'état d'origine.

</details>

**4.3 - Bonus : déboguer un conteneur qui ne démarre pas**

Un conteneur qui sort immédiatement ne permet ni `exec` ni conteneur éphémère utile. Recréez le Pod
`slow-app` de la Partie 3 **sans** sa startup probe (il redémarre en boucle), puis créez-en une
**copie** dont la commande est remplacée par un shell, pour inspecter le système de fichiers.

<details><summary>Correction</summary>

```bash
kubectl debug slow-app -it --copy-to=slow-app-debug --container=slow-app -- sh
# Dans la copie : ls /tmp, env, etc.
exit
kubectl delete pod slow-app slow-app-debug
```

`--copy-to` crée un **nouveau** Pod, copie de l'original, avec la commande remplacée : on peut
explorer sans toucher au Pod d'origine. Par défaut, la copie est créée **sans** les sondes et sans
les labels (options `--keep-liveness`, `--keep-labels`... pour les conserver) : la liveness ne la
tue pas, et la copie ne reçoit pas le trafic des Services.

</details>

---

## Partie 5 : Synthèse

Pour chaque situation, quelle sonde est concernée ?

1. Un thread de l'application est en *deadlock* : le processus tourne toujours mais ne répond plus.
2. L'application charge un cache pendant 2 minutes au démarrage.
3. La base de données est momentanément injoignable : vous préférez ne plus envoyer de trafic
   plutôt que redémarrer l'application.
4. Vous déployez une nouvelle version qui ne répond pas correctement : vous voulez que l'ancienne
   continue à servir.

<details><summary>Correction</summary>

1. **Liveness** : seul un redémarrage débloque le processus.
2. **Startup** : protéger le démarrage sans affaiblir la liveness.
3. **Readiness** : retrait du Service, sans redémarrage.
4. **Readiness** : le rolling update s'arrête, l'ancienne version reste en place.

Paramètres communs à toutes les sondes :

| Champ | Défaut | Rôle |
| --- | --- | --- |
| `initialDelaySeconds` | 0 | attente avant la première vérification |
| `periodSeconds` | 10 | intervalle entre deux vérifications |
| `timeoutSeconds` | 1 | délai au-delà duquel une vérification échoue |
| `failureThreshold` | 3 | échecs consécutifs avant de déclarer l'échec |
| `successThreshold` | 1 | succès consécutifs pour repasser en succès |

</details>

---

## ✅ État attendu en fin de lab

`webapp-deployment.yaml` contient des sondes **readiness** et **liveness** saines ;
`kubectl get pods` affiche `1/1` partout, sans redémarrages récents.

---

## Points clés

* **Liveness** → redémarre. **Readiness** → retire du Service. **Startup** → protège le démarrage.
* Un Pod `Running 0/1` est vivant mais exclu du trafic : regardez sa readiness.
* Un `RESTARTS` qui grimpe avec `Liveness probe failed` dans les événements : sonde mal réglée ou
  application réellement bloquée. `kubectl logs --previous` montre ce qui s'est passé avant.
* La readiness rend le rolling update sûr : une version cassée ne reçoit jamais de trafic.
* Une liveness ne doit jamais dépendre d'un service externe.
* `kubectl debug -it <pod> --image=<outils> --target=<conteneur>` : un conteneur éphémère qui
  partage le réseau et les processus du Pod. `--copy-to` pour une copie modifiable.
