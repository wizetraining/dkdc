# Parcours de labs Kubernetes sur GKE

Ce dossier contient les travaux pratiques de la formation Kubernetes destinée aux équipes d'Elastic
qui **utilisent** Kubernetes au quotidien, sans préparer de certification.

La formation porte sur **Kubernetes**, pas sur Google Cloud : **Google Kubernetes Engine (GKE)** sert
uniquement à fournir à chaque participant un cluster complet en quelques minutes. Le cluster se crée
depuis la console web, on s'y connecte depuis **Cloud Shell** (terminal et éditeur intégrés au
navigateur), et tout le reste se fait avec `kubectl` et des fichiers YAML. Les spécificités de GKE
n'apparaissent que là où elles changent le comportement de Kubernetes (repères ☁️), toujours
accompagnées de ce que serait la réponse sur un cluster installé soi-même (repères 🏠).

Le parcours suit la vie d'une application, **Countvisit** (une application web Python et sa base
PostgreSQL) : du premier Pod à un déploiement GitOps, en passant par la configuration, le stockage,
l'exposition HTTPS, l'autoscaling et l'empaquetage Helm.

---

## 1. Le plan

| Lab | Titre | Durée | Fil rouge Countvisit |
| --- | --- | --- | --- |
| 00 | [Créer son cluster Kubernetes sur GKE](./Lab%2000%20-%20Deployer%20un%20cluster%20GKE/instructions.md) | 40 min | |
| 01 | [kubectl, contextes et API Kubernetes](./Lab%2001%20-%20kubectl%20et%20API%20Kubernetes/instructions.md) | 50 min | |
| 02 | [Pods, namespaces et labels](./Lab%2002%20-%20Pods,%20namespaces%20et%20labels/instructions.md) | 40 min | |
| 03 | [Workloads : Deployments et DaemonSet](./Lab%2003%20-%20Workloads%20-%20Deployments%20et%20DaemonSet/instructions.md) | 55 min | ✅ déploiement de l'application web |
| 04 | [Services et découverte de services](./Lab%2004%20-%20Services%20et%20decouverte%20de%20services/instructions.md) | 55 min | ✅ PostgreSQL, Services |
| 05 | [ConfigMaps, Secrets, Job et CronJob](./Lab%2005%20-%20ConfigMaps,%20Secrets,%20Job%20et%20CronJob/instructions.md) | 55 min | ✅ version « Entreprise », sauvegardes |
| 06 | [Stockage persistant](./Lab%2006%20-%20Stockage%20persistant%20sur%20GKE/instructions.md) | 55 min | ✅ volumes persistants, StatefulSet |
| 07 | [Sondes de santé](./Lab%2007%20-%20Sondes%20de%20sante/instructions.md) | 40 min | ✅ readiness et liveness |
| 08 | [Ingress et TLS](./Lab%2008%20-%20Ingress%20et%20TLS/instructions.md) | 50 min | ✅ HTTPS via Ingress NGINX |
| 09 | [Helm 1 : installer et exploiter un chart](./Lab%2009%20-%20Helm%201%20-%20installer%20et%20exploiter%20un%20chart/instructions.md) | 35 min | |
| 10 | [Ordonnancement et ressources](./Lab%2010%20-%20Ordonnancement%20et%20ressources/instructions.md) | 50 min | |
| 11 | [Autoscaling](./Lab%2011%20-%20Autoscaling/instructions.md) | 50 min | ✅ HPA, VPA |
| 12 | [Utilisateurs, RBAC et ServiceAccounts](./Lab%2012%20-%20RBAC%20et%20ServiceAccounts/instructions.md) | 45 min | |
| 13 | [CRD et Opérateurs](./Lab%2013%20-%20CRD%20et%20Operateurs/instructions.md) | 55 min | ✅ PostgreSQL haute disponibilité (CloudNativePG) |
| 14 | [Helm 2 : créer la chart de Countvisit](./Lab%2014%20-%20Helm%202%20-%20creer%20la%20chart%20Countvisit/instructions.md) | 75 min | ✅ l'application devient un chart, publié dans Harbor |
| 15 | [GitOps avec Argo CD *(optionnel)*](./Lab%2015%20-%20GitOps%20avec%20Argo%20CD%20(optionnel)/instructions.md) | 60 min | ✅ déploiement depuis Git |

Soit environ **13 h 30** de pratique, Lab 15 compris.

Les Labs 09, 10 et 12 sont **indépendants** du fil rouge (namespaces dédiés) : ils peuvent être
déplacés ou raccourcis. Les labs marqués ✅ s'enchaînent : chacun part de l'état laissé par le
précédent (section **✅ État attendu en fin de lab**).

---

## 2. Le cluster de chaque participant

Chaque participant crée **son propre cluster** au Lab 00, depuis la console web :

| Paramètre | Valeur |
| --- | --- |
| Mode | GKE **Standard** |
| Emplacement | **régional** (3 zones), région indiquée par le formateur |
| Nœuds | **1 par zone**, soit 3 nœuds **e2-medium** (2 vCPU partagés, 4 Go) |
| Option | *Autoscaling vertical des pods* activé (Lab 11) |
| Contexte `kubectl` | **`cluster-lab-dkdc`** |

Les nœuds sont volontairement petits : les labs ont été dimensionnés en conséquence (requests et
quotas du Lab 10). Le caractère régional du cluster sert de support pédagogique : répartition des
Pods par zone (Labs 10 et 13), disques persistants liés à une zone (Lab 06).

---

## 3. Prérequis

* Un **projet Google Cloud** de formation, avec, pour chaque participant, le droit de créer un
  cluster GKE (rôle *Kubernetes Engine Admin*).
* Des **quotas** suffisants dans la région : 6 vCPU par participant, et 2 à 3 adresses IP externes
  (Services `LoadBalancer` des Labs 04 et 08).
* Un navigateur : **Cloud Shell** fournit `kubectl`, `helm`, `git`, `openssl` et un éditeur de code.
* Pour le Lab 14 : un compte sur le registre **Harbor** `harbor.wizetraining.com` (un projet et des
  identifiants par groupe, fournis par le formateur).
* Pour le Lab 15 : un compte **GitHub**.

---

## 4. Conventions des énoncés

Chaque question donne un énoncé suffisant pour être traitée **sans** ouvrir la correction, placée
dans un bloc repliable **Correction**.

| Repère | Signification |
| --- | --- |
| `> **Cluster : GKE** ...` | en-tête : contexte `kubectl`, durée, prérequis |
| 🔎 **Attention** | un piège fréquent |
| 💡 **Astuce** / **Bon réflexe** | une méthode de travail à retenir |
| ☁️ **Sur GKE** | ce que GKE fait (ou gère) pour vous |
| 🏠 **En local** | ce que serait la réponse sur un cluster installé soi-même (kubeadm, VM, bare-metal) |
| 🩺 | une question de diagnostic : quelque chose ne marche pas, trouvez pourquoi |
| 🧹 | du ménage à faire avant de continuer |

La commande `gcloud` n'est indispensable qu'une fois (récupérer les identifiants du cluster, Lab 00).
Ailleurs, elle n'apparaît que pour **consulter** des ressources, en alternative à la console web.

Les images de conteneurs de l'application et des exercices viennent du registre public **Amazon ECR**
`public.ecr.aws/wizetraining/`. Quelques labs utilisent des images tierces (Ingress NGINX au Lab 08,
Bitnami au Lab 09, CloudNativePG au Lab 13, Argo CD au Lab 15), signalées dans le lab.

---

## 5. Ce que ce parcours ne couvre pas

Volontairement, il n'y a ni installation manuelle de cluster, ni mise à jour, ni sauvegarde d'etcd,
ni dépannage des nœuds : sur GKE, Google les prend en charge. Les NetworkPolicies ne sont pas
traitées non plus, et la Gateway API n'est que présentée (Lab 08). L'autoscaling des nœuds et les
node pools sont expliqués sans manipulation (Labs 10 et 11), pour ne pas consommer de ressources
supplémentaires.

---

## 6. Coûts et nettoyage

Ce qui coûte, en plus des 3 nœuds :

* chaque Service `LoadBalancer` (un équilibreur Google Cloud et une IP publique chacun) : celui du
  Lab 04, puis celui d'Ingress NGINX à partir du Lab 08 ;
* les disques persistants, qui peuvent **survivre** à la suppression du cluster.

Chaque lab se termine par son ménage (🧹). En fin de formation, la **Partie 6 du Lab 00** explique
comment supprimer le cluster sans rien laisser derrière soi.

> **Note pour le formateur :** les labs n'ont pas été rejoués de bout en bout sur GKE au moment de
> leur rédaction. Faites un passage complet avant la première session, en particulier : les libellés
> du formulaire de création de la console (Lab 00), la signature des certificats clients par l'API
> `CertificateSigningRequest` (Lab 12), l'échafaudage de `helm create` selon la version de Helm et la
> publication dans Harbor (Lab 14), l'accès à Argo CD via Ingress NGINX (Lab 15), et la tenue en
> ressources des nœuds e2-medium.
