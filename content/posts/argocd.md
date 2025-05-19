---
date: '2025-05-17T20:00:00+01:00'
draft: false
title: 'Argo CD : Repenser le déploiement d'application et la gestion d'infrastructure avec le GitOps'
categories: ['kubernetes', 'devops', 'gitops', 'cloud', 'argocd']
---

**Argo CD** est un outil de déploiement continu spécifiquement conçu pour les environnements **Kubernetes**. Fonctionnant selon le principe du **GitOps**, il permet de simplifier et d'automatiser le déploiement d'application et la gestion d'infrastructures conteneurisées en s'appuyant sur des dépôts **Git** comme source de vérité.

## Le GitOps

C'est une méthodologie liée au déploiement d'applications et à la gestion automatisée d'infrastructure. Elle consiste à gérer l'infrastructure en tant que code, sous forme de fichiers (souvent au format `YAML`) stockés dans des dépôts git utilisés comme **source de vérité**. L'intérêt est de garantir que l'état de l'infrastructure et des applications est toujours en phase avec la configuration définie dans le dépôt.

Il existe plusieurs "modes" de GitOps : **push** ou **pull** :

- **Mode push** : Cas où c'est la modification du référentiel qui provoque la livraison d'une nouvelle configuration afin de placer l'infrastructure dans l'état souhaité. Par exemple, une pipeline (Jenkins, GitLab CI…) déclenche l'application de la configuration après une modification dans Git.

- **Mode pull** : un agent (ex : Argo CD, Flux) scrute le dépôt Git en continu et applique les changements dès qu'ils sont détectés. Les modifications sont apportées automatiquement par l'outil sans autre intervention humaine que la modification du référentiel.

**Remarques** :

- Le framework ne remplace pas l'usage d'une **CI** classique pour les étapes de test ou de build d'images.

- L'approche **pull** est plus robuste, mieux adaptée à Kubernetes, et permet une synchronisation continue sans intervention manuelle. Dès lors, l'utilisation des fonctionnalités de **Merge Request** et de **Pull Request** afin d'effectuer une relecture et une validation des modifications est recommandée.

## Argo CD : principes et fonctionnalités

**Argo CD** est un outil de la catégorie **pull**. Il permet de gérer le déploiement d'applications et la configuration de clusters **Kubernetes** en se basant sur diverses sources : **Git**, **Helm** ou encore **Kustomize**.

**Son principe de fonctionnement est très simple** : il scrute en permanence les dépôts **Git** enregistrés en tant que sources afin de détecter des modifications et appliquer automatiquement les changements correspondant à la configuration du cluster **Kubernetes** et des applications qu'il héberge.

### Fonctionnalités

- **Déploiement & synchronisation** : Argo déploie les ressources sur le cluster **Kubernetes** en se basant sur la configuration définie dans le dépôt **Git**. Il s'assure que l'état du cluster correspond à l'état souhaité défini dans le dépôt en intéragissant avec l'API **Kubernetes**.
- **Custom Resource Definitions (CRD)** : Argo CD intégère ses propres ressources personnalisées (CRD) pour gérer l'ensemble des configurations de l'outil en tant que code, voire en modèle de **GitOps**. Quelques exemples de CRD :
  - `Application` : représente une application déployée sur le cluster.
  - `AppProject` : permet de définir des projets d'application, facilitant la gestion des autorisations et des ressources.
  - `ApplicationSet` : permet de gérer plusieurs applications en même temps, facilitant la gestion de déploiements complexes.
- **UI Web** : interface web intuitive pour visualiser l'état des applications, gérer les déploiements, effectuer des opérations de synchronisation et même consulter les logs des pods.
- **Authentification et SSO** : Argo CD prend en charge plusieurs méthodes d'authentification, y compris l'intégration avec des systèmes d'authentification tiers (ex : OAuth2, LDAP).
- **Gestion des accès** : Argo CD embarque un système de gestion de permissions basé sur les rôles (RBAC) permettant de contrôler l'accès aux ressources et aux fonctionnalités de l'outil. Il est possiblede lier ces rôles aux utilisateurs et groupes d'un fournisseur d'identité (ex : LDAP, SSO).
- **Métriques et alertes** : Argo CD expose des métriques **Prometheus** pour surveiller l'état des applications et des clusters. Il est possible de configurer des alertes basées sur ces métriques.
- **Gestion multi-cluster** : possibilité de gérer plusieurs clusters **Kubernetes** à partir d'une seule instance, facilitant ainsi la gestion d'environnements complexes.
- Intégration avec des **Webhooks** : Argo CD peut être configuré pour recevoir des notifications de changements dans les dépôts **Git** via des **webhooks**, permettant ainsi une synchronisation instantanée des modifications.
- **Gestion des secrets**

### Fonctionnement et architecture

**Argo CD** est un ensemble de composants qui sont déployés sur le cluster **Kubernetes** sous forme de microservices. Voici les principaux composants et leur fonction :

- **L’API Server** :
  - Expose l'API utilisée par l'interface web et la CLI d'ArgoCD
  - Gestion des applications et la récupération de leur statut
  - Exécution des opérations telles que la synchronisation, le retour en arrière et les actions définies par l'utilisateur
  - Gestion des identifiants des dépôts et des clusters, qui sont stockés en tant que `Secrets` **Kubernetes**
  
- **Le Repository Server** :
  - Stocke les informations sur les dépôts git: URL, version, chemin vers la définition de l'application
  - Mise en cache des manifestes d'application pour une récupération rapide

- **L’Application Controller** :
  - Surveille l'état des applications déployées sur le cluster
  - Compare l'état actuel des applications à l'état cible défini dans le dépôt
  - Prend des mesures correctives en cas de détection d'états OutOfSync ou de changement dans la source (via le controller manager de K8s)
  - Gère les événements définis par l'utilisateur relatifs au cycle de vie de l’application (PreSync, Sync et PostSync)

D'autres composants sont présents comme **l'application set controller** pour la gestion des ressources du même nom, ou **dex** pour la gestion des utilisateurs et de l'authentification.

### Avantages

- Centralisation des configurations dans un référentiel unique (git)
- Versionninf des configurations d’infrastructure et possibilité de revenir à une version antérieure en cas de problème
- Garantit la cohérence entre l'état réel et l'état souhaité de l'infrastructure
- Éviter les modifications manuelles non tracées
- Fiabilisation des déploiements et limitation des erreurs humaines
- Aperçu de l'ensemble des ressources déployées dans le cluster via l'interface web

## Installation

### Méthode de base

La première manière d'insaller **Argo CD** est de le déployer en suivant la documentation officielle.

Pour cela, il suffit d'appliquer le fichier **manifest** fournit par la documentation dans un `namespace` dédié :

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Une fois installé, il suffit de faire un `port-forward` sur le service `argocd-server` pour accéder à l'interface web :

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 --address 0.0.0.0
```

Une fois connecté sur l'interface web, il est possible de se connecter avec le nom d'utilisateur `admin` et le mot de passe par défaut qu'il est possible d'obtenir via la commande suivante :

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### Installation via Helm

Dans le cas d'un environnement **Kubernetes** en production, il est souvent plus propre de déployer les ressources via un **chart Helm**. Il existe un grand nombre de charts pour tout type de services, et **Argo CD** n'échappe pas à la règle.

L'installation via **Helm** s'effectue avec les commandes suivantes :

```bash
helm repo add argo https://argoproj.github.io/argo-helm # Ajout du dépôt Helm d'Argo
helm install my-argo-cd argo/argo-cd -n argocd --create-namespace # Installation d'Argo dans le namespace
```

**Astuce** : si le dépôt **Helm** n'est pas déte, utiliser `helm repo update`.

Dans le cas d'une installation par **Helm**, il est possible de personnaliser la configuration de l'outil dès l'installation via des **values**.

Par exemple, il est possible de désactiver **tls** sur le serveur **argocd-server** :

```yaml
configs:
  params:
    server.insecure: "true"
```

Ces éléments sont à ajouter dans un fichier `values.yaml` et à passer lors de l'installation :

```bash
helm install my-argo-cd argo/argo-cd -n argocd --create-namespace -f values.yaml
```

L'ensemble des valeurs configurables sont détaillées dans le dépôt git du projet [argo-helm](https://github.com/argoproj/argo-helm/blob/main/charts/argo-cd/values.yaml).

**Remarque** : pour avoir un déploiement totalement automatisé et reproductible, il est possible d'utiliser le **provider Helm** pour **Terraform** afin de déployer **Argo CD** sur le cluster **Kubernetes**.

## Configuration

### Authentification

### Gestion des accès et permissions

### Monitoring

## Déploiement d'applications

### Déployer une application simple

**Argo CD** étant plutôt simple à utiliser, il est possible de déployer directement une application via l'interface web.

On retrouve deux méthodes de définition d'application :

- Via un formulaire web
- Via un éditeur de texte permettant de définir l'application en **YAML**

Ces éléments permettent de créer la ressource `Application` qui contient toutes les informations nécessaires au déploiement de l'application, et à son suivi :

- **Source** : dépôt **Git**, **Helm** ou **Kustomize**.
- **Destination** : cluster **Kubernetes** et namespace cible
- **Sync Policy** et **Sync Options** : stratégie de synchronisation (automatique ou manuelle) et options de synchronisation (ex : prune, self heal)
- **Project** : projet auquel l'application appartient afin de gérer les accès et les ressources

Concernant l'utilisation de l'interface web, les champs étant assez explicites, il est possible de créer une application en quelques clics. 

Pour la création d'une application via un fichier **YAML**, voici un exemple de définition d'application :

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: some-app # Nom de l'application
  namespace: argocd # Namespace où la ressource application sera créée
spec:
  project: default # Projet par défaut
  source:
    repoURL: https://github.com/example-org/gitops-demo # URL du dépôt
    targetRevision: main # Branche
    path: apps/ # Chemin vers les fichiers de configuration
  destination:
    server: https://kubernetes.default.svc # URL de l'API du cluster (ici le cluster par défaut)
    namespace: app # Namespace cible
  syncPolicy:
    automated: # Synchronisation automatique
      prune: true # Prune les ressources non définies dans le dépôt (supprime les ressources non gérées par Argo CD)
      selfHeal: true # Auto-réparation des ressources
    syncOptions:
    - CreateNamespace=true
    - PruneLast=true # Prune les ressources après la synchronisation
```

Il est possible d'utiliser des **Charts Helm** comme source d'application. Dans ce cas, il faut ajouter la section `helm` dans la définition de l'application :

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kubernetes-dashboard
  namespace: argocd
spec:
  project: default
  source:
    chart: kubernetes-dashboard
    repoURL: https://kubernetes.github.io/dashboard/
    targetRevision: 7.10.0
    helm:
      releaseName: kubernetes-dashboard
      values: |
        kong.admin.tls.enabled: false
  destination:
    server: "https://kubernetes.default.svc"
    namespace: monitoring
  syncPolicy:
    syncOptions: 
    - CreateNamespace=true
```

La personnalisation des **values** du **chart Helm** peut se faire de plusieurs manières :

- En utilisant la section `values` dans la définition de l'application (comme ci-dessus)
- En utilisant un fichier **YAML** externe (ex : `values.yaml`) en l'ajoutant dans la section `valuesFiles`.

### Déployer plusieurs applications

#### App of Apps

Bien qu'il soit possible d'utiliser l'interface web pour créer chaque application une à une, il est préférable de le faire via des fichiers **YAML** stockés dans un dépôt **Git**. Cela permet de versionner la configuration des applications et de les gérer comme n'importe quelle autre ressource **Kubernetes**.

Ce type d'implémentation est appelé **App of Apps**, c'est à dire qu'à partir d'une application Argo CD, une multitudes d'autres applications seront déployées car définies dans des fichiers **YAML** stockés dans un dépôt **Git** scruté par l'application racine.

#### ApplicationSet



### Les projets dans Argo CD



### Gestion des secrets