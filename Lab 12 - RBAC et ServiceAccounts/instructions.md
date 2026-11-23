# LAB 12 - Utilisateurs, RBAC et ServiceAccounts

> **Cluster : GKE** (contexte `cluster-lab-dkdc`) | **Durée : ~45 min** (+ 10 min de bonus)
> **Prérequis :** ce lab est indépendant du fil rouge. Travaillez depuis Cloud Shell (`openssl` et
> `base64` y sont installés).

Toute requête adressée à l'API Server franchit deux contrôles successifs :

```text
Requête → [1] AUTHENTIFICATION     → [2] AUTORISATION             → exécution
              « qui êtes-vous ? »       « avez-vous le droit ? »
              certificat, token, OIDC   RBAC
```

Kubernetes n'a **pas d'objet « utilisateur »** : il fait confiance à une identité prouvée
(certificat client, token, OIDC...). Avec un certificat client, le **CN** devient le nom
d'utilisateur et chaque **O** un groupe.

Avant de commencer, notez votre contexte d'administration et créez le namespace du lab :

```bash
kubectl config current-context        # notez-le : vous y reviendrez à la fin (cluster-lab-dkdc)
kubectl create namespace rbac-lab
```

## Objectifs

1. Créer un utilisateur à partir d'un **certificat client** signé par le cluster.
2. Donner des droits précis avec **Role / RoleBinding**, réutiliser les **ClusterRoles** prédéfinis.
3. Vérifier des droits avec `kubectl auth can-i` et l'**impersonation**.
4. Donner une identité à une application avec un **ServiceAccount**.
5. Distinguer un échec d'**authentification** d'un échec d'**autorisation**.
6. Bonus : comprendre comment GKE combine les identités Google (IAM) et RBAC.

**Documentation utile :** [Autorisation RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) · [Certificate Signing Requests - utilisateur](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#normal-user) · [ServiceAccounts](https://kubernetes.io/docs/concepts/security/service-accounts/) · [Contrôle d'accès GKE](https://cloud.google.com/kubernetes-engine/docs/concepts/access-control)

---

## Partie 1 : Créer l'utilisateur `devuser`

Étapes : clé privée → demande de signature (CSR) → signature par l'autorité du cluster →
configuration de `kubectl`.

**1.1 - Clé et demande de signature**

Générez une clé privée RSA `devuser.key` (2048 bits), puis une CSR `devuser.csr` avec le sujet
`/CN=devuser/O=dev`. Que représentent `CN` et `O` pour Kubernetes ?

<details><summary>Correction</summary>

```bash
mkdir -p ~/labs/rbac && cd ~/labs/rbac
openssl genrsa -out devuser.key 2048
openssl req -new -key devuser.key -out devuser.csr -subj "/CN=devuser/O=dev"
openssl req -in devuser.csr -noout -subject
```

* `CN` (Common Name) = **nom d'utilisateur** ;
* chaque `O` (Organization) = un **groupe** d'appartenance. On peut en mettre plusieurs :
  `/CN=devuser/O=dev/O=qa`.

</details>

**1.2 - Soumettre la demande au cluster**

Soumettez la CSR au cluster avec un objet `CertificateSigningRequest` (`certificates.k8s.io/v1`) nommé
`devuser`, avec :

* `signerName: kubernetes.io/kube-apiserver-client` ;
* `usages: [client auth]` ;
* une durée de vie d'une heure (`expirationSeconds: 3600`) ;
* le champ `request` contenant la CSR encodée en base64 **sur une seule ligne**.

Dans quel état est la demande ?

<details><summary>Correction</summary>

```bash
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: devuser
spec:
  request: $(base64 < devuser.csr | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 3600
  usages:
  - client auth
EOF

kubectl get csr devuser            # CONDITION : Pending
```

🔎 **Attention** - trois détails font échouer cette étape : un base64 **sur plusieurs lignes**
(d'où `tr -d '\n'`), un `signerName` inexact, ou l'oubli de `client auth` dans `usages`.

</details>

**1.3 - Approuver et récupérer le certificat**

En tant qu'administrateur, approuvez la demande, puis extrayez le certificat signé dans `devuser.crt`.
Vérifiez son sujet et ses dates de validité.

<details><summary>Correction</summary>

```bash
kubectl certificate approve devuser
kubectl get csr devuser            # Approved,Issued

kubectl get csr devuser -o jsonpath='{.status.certificate}' | base64 -d > devuser.crt
openssl x509 -in devuser.crt -noout -subject -dates
```

</details>

**1.4 - Configurer kubectl**

Ajoutez à votre kubeconfig :

* un utilisateur `devuser` (certificat + clé, option `--embed-certs=true`) ;
* un contexte `devuser` qui réutilise **le même cluster** que votre contexte actuel (retrouvez son
  nom avec `kubectl config view --minify`).

<details><summary>Correction</summary>

```bash
CLUSTER=$(kubectl config view --minify -o jsonpath='{.clusters[0].name}')

kubectl config set-credentials devuser \
  --client-certificate=devuser.crt --client-key=devuser.key --embed-certs=true
kubectl config set-context devuser --cluster=$CLUSTER --user=devuser

kubectl config get-contexts
```

`--embed-certs=true` copie le certificat et la clé **dans** le kubeconfig : le fichier devient
autonome, et donc sensible. Il se protège comme un mot de passe.

</details>

**1.5 - 🩺 Authentifié, mais pas autorisé**

Basculez sur le contexte `devuser` et listez les Pods de `rbac-lab`. Quel est le résultat ? Que
prouve exactement ce message ? Revenez ensuite au contexte d'administration.

<details><summary>Correction</summary>

```bash
kubectl config use-context devuser
kubectl get pods -n rbac-lab
```

```text
Error from server (Forbidden): pods is forbidden: User "devuser" cannot list resource "pods"
in API group "" in the namespace "rbac-lab"
```

**C'est une bonne nouvelle.** Le message nomme `User "devuser"` : le cluster sait qui vous êtes,
l'**authentification a réussi**. C'est l'**autorisation** qui échoue : aucun droit n'a encore été
donné.

| Message | Étape en échec | Où chercher |
| --- | --- | --- |
| `Forbidden ... User "x" cannot ...` | **autorisation** | Roles, RoleBindings |
| `Unauthorized` / `You must be logged in to the server` | **authentification** | certificat expiré, mauvaise CA, mauvais utilisateur dans le kubeconfig |

```bash
kubectl config use-context cluster-lab-dkdc   # votre contexte d'administration
```

</details>

---

## Partie 2 : Donner des droits avec RBAC

* **Role** : un ensemble de permissions (verbes sur des ressources) dans **un namespace**.
  **ClusterRole** : la même chose, à l'échelle du cluster, ou pour des ressources non namespacées
  (nœuds, PV...).
* **RoleBinding** : accorde un Role **ou un ClusterRole** à des sujets (utilisateur, groupe,
  ServiceAccount), **dans un namespace**. **ClusterRoleBinding** : sur tout le cluster.

**2.1 - Un Role pour le groupe `dev`**

Depuis le contexte d'administration, créez dans `rbac-lab` un Role `pod-reader` autorisant `get`,
`list` et `watch` sur les ressources `pods` et `pods/log`. Liez-le au **groupe** `dev` par un
RoleBinding `dev-pod-reader`.

<details><summary>Correction</summary>

```bash
kubectl create role pod-reader -n rbac-lab --verb=get,list,watch --resource=pods,pods/log
kubectl create rolebinding dev-pod-reader -n rbac-lab --role=pod-reader --group=dev
```

Équivalent déclaratif :

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: rbac-lab
rules:
- apiGroups: [""]                    # "" = le groupe d'API core (pods, services, configmaps...)
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-pod-reader
  namespace: rbac-lab
subjects:
- kind: Group
  name: dev                          # le O= du certificat
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

🔎 **Attention** - `get` ne suffit pas pour `kubectl get pods` sans nom : lister exige **`list`**.
Et `kubectl logs` demande le droit sur la **sous-ressource** `pods/log`.

</details>

**2.2 - Vérifier par impersonation**

Avec `kubectl auth can-i` et l'option `--as devuser`, vérifiez si `devuser` peut :
lister les Pods dans `rbac-lab`, les supprimer, les lister dans `default`.
Le premier résultat est-il celui attendu ? Ajoutez `--as-group dev` : pourquoi est-ce nécessaire ?

<details><summary>Correction</summary>

```bash
kubectl auth can-i list pods -n rbac-lab --as devuser                    # no  ← surprise
kubectl auth can-i list pods -n rbac-lab --as devuser --as-group dev     # yes
kubectl auth can-i delete pods -n rbac-lab --as devuser --as-group dev   # no
kubectl auth can-i list pods -n default --as devuser --as-group dev      # no
```

Kubernetes **ne mémorise pas** les groupes d'un utilisateur : ils viennent du certificat présenté à
chaque requête (champs `O`). En impersonation, aucun certificat n'est présenté : il faut déclarer
les groupes avec `--as-group`.

💡 **Astuce** - pour vérifier des droits en une commande :
`kubectl auth can-i --list -n <ns> --as <user> --as-group <groupe>`

</details>

**2.3 - Vérifier en conditions réelles**

Passez au contexte `devuser` et contrôlez que le comportement est le même, **sans** impersonation.
Revenez ensuite au contexte d'administration.

<details><summary>Correction</summary>

```bash
kubectl config use-context devuser
kubectl get pods -n rbac-lab                 # autorisé (liste vide)
kubectl run test --image=public.ecr.aws/wizetraining/nginx:latest -n rbac-lab   # Forbidden
kubectl auth can-i --list -n rbac-lab
kubectl config use-context cluster-lab-dkdc
```

</details>

**2.4 - Défi : réutiliser un ClusterRole prédéfini**

Donnez à l'**utilisateur** `devuser` (sujet de type `User`, pas le groupe) le droit de créer et lire
des Deployments **uniquement dans `rbac-lab`**, en utilisant le ClusterRole prédéfini adapté
(indice : `kubectl get clusterrole | grep -E "^(view|edit|admin) "`). Vérifiez avec `can-i` dans
`rbac-lab` puis dans `default`.

<details><summary>Correction</summary>

```bash
kubectl describe clusterrole edit | head -20
kubectl create rolebinding devuser-edit -n rbac-lab --clusterrole=edit --user=devuser

kubectl auth can-i create deployments -n rbac-lab --as devuser    # yes
kubectl auth can-i create deployments -n default  --as devuser    # no
```

Un **RoleBinding** vers un **ClusterRole** réutilise des permissions standard en les **limitant au
namespace** du binding. Les rôles prédéfinis couvrent la plupart des besoins :

| ClusterRole | Droits |
| --- | --- |
| `view` | lecture seule sur la plupart des ressources, **sauf les Secrets** |
| `edit` | `view` + création et modification, sans toucher au RBAC |
| `admin` | `edit` + gestion des Roles et RoleBindings du namespace |
| `cluster-admin` | tout, partout |

🔎 **Attention** - un **ClusterRoleBinding** vers `edit` aurait donné ce droit dans **tous** les
namespaces. Le type de binding décide de la portée, pas le type de rôle.

</details>

---

## Partie 3 : ServiceAccount, l'identité d'une application

Un **ServiceAccount** est l'identité d'un Pod lorsqu'il s'adresse à l'API Kubernetes (opérateurs,
CI, outils de supervision). Son nom complet, côté RBAC, est
`system:serviceaccount:<namespace>:<nom>`.

**3.1 - Un ServiceAccount sans droits**

Dans `rbac-lab`, créez un ServiceAccount `monitoring`. A-t-il des droits par défaut ?
(indice : `kubectl auth can-i ... --as=system:serviceaccount:rbac-lab:monitoring`)

<details><summary>Correction</summary>

```bash
kubectl create serviceaccount monitoring -n rbac-lab
kubectl auth can-i list pods -n rbac-lab --as=system:serviceaccount:rbac-lab:monitoring    # no
```

Un ServiceAccount nouvellement créé n'a aucun droit sur les ressources : moindre privilège par défaut.

</details>

**3.2 - Lui donner le strict nécessaire**

Donnez-lui **uniquement** le droit de lister les Pods de `rbac-lab` (un Role `pod-lister` et un
RoleBinding). Vérifiez l'ensemble de ses droits avec `kubectl auth can-i --list`.

<details><summary>Correction</summary>

```bash
kubectl create role pod-lister -n rbac-lab --verb=list --resource=pods
kubectl create rolebinding monitoring-pod-lister -n rbac-lab \
  --role=pod-lister --serviceaccount=rbac-lab:monitoring

kubectl auth can-i list pods -n rbac-lab --as=system:serviceaccount:rbac-lab:monitoring    # yes
kubectl auth can-i --list -n rbac-lab --as=system:serviceaccount:rbac-lab:monitoring
```

🔎 **Attention** - l'option s'écrit `--serviceaccount=<namespace>:<nom>`. Oublier le namespace est
une erreur fréquente.

</details>

**3.3 - Le token d'un ServiceAccount**

Générez un token de 10 minutes pour ce ServiceAccount. Lancez un Pod `sa-test` (busybox ECR,
`sleep 3600`) qui utilise ce ServiceAccount, et retrouvez le token monté dans le conteneur. Pourquoi
est-il recommandé de désactiver ce montage automatique pour une application qui n'appelle jamais
l'API ?

<details><summary>Correction</summary>

```bash
kubectl create token monitoring -n rbac-lab --duration=10m

kubectl run sa-test -n rbac-lab --image=public.ecr.aws/wizetraining/busybox:latest \
  --overrides='{"spec":{"serviceAccountName":"monitoring"}}' -- sleep 3600
kubectl exec -n rbac-lab sa-test -- ls /var/run/secrets/kubernetes.io/serviceaccount/
# ca.crt  namespace  token
```

Le kubelet monte un token **à durée limitée**, renouvelé automatiquement et invalidé à la
suppression du Pod. L'application le présente à l'API Server pour s'authentifier.

Si l'application n'a pas besoin de l'API, mieux vaut ne pas lui donner ce token : une application
compromise ne pourrait pas s'en servir. Principe du **moindre privilège**.

```yaml
# au niveau du ServiceAccount, ou du Pod (le Pod est prioritaire)
automountServiceAccountToken: false
```

> *Registre privé :* les images du lab sont publiques sur ECR. Pour tirer une image d'un registre
> **privé**, on crée un Secret `docker-registry`, référencé par `imagePullSecrets` sur le Pod ou,
> mieux, sur le ServiceAccount : tous les Pods qui l'utilisent en héritent.

</details>

---

## Partie 4 : Bonus - comment GKE gère les identités

*(Lecture et commandes en lecture seule : aucune modification des droits Google Cloud.)*

Dans ce lab, `devuser` s'est authentifié avec un **certificat client** : c'est le mécanisme de base de
Kubernetes, utilisé sur les clusters installés avec kubeadm. Vous, en revanche, vous vous
authentifiez autrement depuis le Lab 00.

**4.1 - Qui êtes-vous ?**

Affichez votre propre identité (`kubectl auth whoami`), puis la section `users` de votre contexte
(`kubectl config view --minify`). Cherchez ensuite un `ClusterRoleBinding` qui mentionne votre
adresse e-mail. D'où viennent vos droits ?

<details><summary>Correction</summary>

```bash
kubectl auth whoami                       # Username : votre adresse e-mail Google
kubectl config view --minify -o jsonpath='{.users[0].user.exec.command}'; echo   # gke-gcloud-auth-plugin
kubectl get clusterrolebindings -o wide | grep -i "@"                            # rien à votre nom
```

Sur GKE, l'authentification passe par votre **compte Google** : le plugin `gke-gcloud-auth-plugin`
fournit à `kubectl` un jeton Google temporaire. Vos droits ne viennent pas de RBAC mais d'**IAM**,
le système de droits de Google Cloud : le rôle IAM qui vous a permis de créer le cluster vous donne
aussi tous les droits **dans** le cluster.

Sur GKE, une requête est autorisée si **IAM ou RBAC** l'autorise. Les deux s'additionnent :

| Rôle IAM (sur le projet) | Accès aux objets Kubernetes |
| --- | --- |
| *Kubernetes Engine Admin* | tout, dans tous les clusters du projet |
| *Kubernetes Engine Developer* | lecture et écriture sur les objets des clusters du projet |
| *Kubernetes Engine Viewer* | lecture seule sur les objets des clusters du projet |
| *Kubernetes Engine Cluster Viewer* | **aucun** : seulement le droit de se connecter (`get-credentials`) |

💡 **Bon réflexe** - IAM agit au niveau du **projet** : tous les clusters, tous les namespaces. Pour
un accès fin (une équipe, un namespace), la pratique recommandée est : *Cluster Viewer* en IAM pour
pouvoir se connecter, puis **RBAC**, exactement comme dans ce lab, pour les droits réels. Les sujets
des bindings sont alors des adresses e-mail (`--user=alice@example.com`) ou des **groupes Google**
(`--group=dev-team@example.com`, fonctionnalité *Google Groups for RBAC*).

</details>

**4.2 - Et les droits d'une application sur Google Cloud ?**

Un ServiceAccount donne à un Pod des droits **dans Kubernetes**. Comment donner à un Pod le droit de
lire un bucket Cloud Storage ou un secret de Secret Manager, sans stocker de clé dans un Secret ?

<details><summary>Correction</summary>

Avec **Workload Identity Federation for GKE** (option à activer sur le cluster) : un administrateur
Google Cloud accorde un rôle IAM **directement à un ServiceAccount Kubernetes**, identifié par son
namespace et son nom :

```text
principal://iam.googleapis.com/projects/<numéro>/locations/global/workloadIdentityPools/<projet>.svc.id.goog/subject/ns/<namespace>/sa/<serviceaccount>
```

Les Pods qui utilisent ce ServiceAccount obtiennent alors des jetons Google à courte durée de vie,
sans aucune clé à stocker ni à faire tourner. Le droit est attaché à **un** ServiceAccount d'**un**
namespace : c'est le prolongement naturel du moindre privilège vu en Partie 3.

🏠 **En local** - on stocke une clé de compte de service dans un Secret (à éviter), ou on configure
une fédération d'identité côté fournisseur cloud, qui fait confiance aux tokens de ServiceAccount
signés par le cluster.

</details>

---

## 🧹 Nettoyage

```bash
kubectl config use-context cluster-lab-dkdc
kubectl config delete-context devuser
kubectl config delete-user devuser
kubectl delete csr devuser
kubectl delete namespace rbac-lab
rm -f ~/labs/rbac/devuser.*
```

## ✅ État attendu en fin de lab

Vous êtes revenu sur votre contexte d'administration (`kubectl config current-context` affiche
`cluster-lab-dkdc`), et vous savez distinguer un `Forbidden` (autorisation) d'un `Unauthorized`
(authentification).

---

## Points clés

* Pas d'objet `User` : `CN` = nom, chaque `O` = un groupe. Un certificat ne se révoque pas : on
  retire ses bindings.
* Role / RoleBinding = un namespace. ClusterRole / ClusterRoleBinding = tout le cluster.
  **ClusterRole + RoleBinding** = droits standard limités à un namespace.
* `get` ≠ `list` ; `kubectl logs` passe par `pods/log`.
* `kubectl auth can-i <verbe> <ressource> -n <ns> --as <user> --as-group <groupe>` ; l'impersonation
  ne devine pas les groupes.
* ServiceAccount : `system:serviceaccount:<ns>:<nom>`, aucun droit par défaut.
  `--serviceaccount=<ns>:<nom>` dans les bindings.
* `Forbidden` = autorisation ; `Unauthorized` = authentification.
* Sur GKE, vous vous authentifiez avec votre compte Google, et une requête est autorisée si **IAM ou
  RBAC** l'autorise. Workload Identity donne à un ServiceAccount des droits Google Cloud sans clé.
