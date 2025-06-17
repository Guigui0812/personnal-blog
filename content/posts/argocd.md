---
date: '2025-05-26T20:00:00+01:00'
draft: false
title: 'Argo CD : guide complet GitOps pour Kubernetes (en français)'
categories: ['kubernetes', 'devops', 'gitops', 'cloud', 'argocd', 'cicd']
description: 'Guide complet sur Argo CD pour déployer vos applications Kubernetes avec GitOps. Installation, configuration, RBAC, OIDC, monitoring et meilleures pratiques.'
cover:
  image: '/images/argocd.png'
  alt: 'Argo CD'
  caption: 'Argo CD'
  focalPoint: 'center center'
---

> 🧠 Dans ce guide **en français**, découvrez comment utiliser **Argo CD** pour automatiser vos déploiements Kubernetes selon la méthode **GitOps**. Vous apprendrez à l'installer, le configurer, gérer les accès, et suivre vos applications avec des outils comme Prometheus.


**Argo CD** est un outil de déploiement continu spécifiquement conçu pour les environnements **Kubernetes**. Fonctionnant selon le principe du **GitOps**, il permet de simplifier et d'automatiser le déploiement d'applications et la gestion d'infrastructures conteneurisées en s'appuyant sur des dépôts **Git** comme source de vérité.

## Le GitOps

C'est une méthodologie liée au déploiement d'applications et à la gestion automatisée d'infrastructure. Elle consiste à gérer l'infrastructure en tant que code, sous forme de fichiers (souvent au format `YAML`) stockés dans des dépôts git utilisés comme **source de vérité**. L'intérêt est de garantir que l'état de l'infrastructure et des applications est toujours en phase avec la configuration définie dans le dépôt.

Il existe plusieurs "modes" de GitOps nommés **push** et **pull** :

- **Mode push** : Cas où c'est la modification du référentiel qui provoque la livraison d'une nouvelle configuration afin de placer l'infrastructure dans l'état souhaité. Par exemple, une pipeline (Jenkins, GitLab CI…) déclenche l'application de la configuration après une modification dans Git.
- **Mode pull** : un agent (ex : Argo CD, Flux) scrute le dépôt Git en continu et applique les changements dès qu'ils sont détectés. Les modifications sont apportées automatiquement par l'outil sans autre intervention humaine que la modification du référentiel.

**Remarques** :

- Le**GitOps** ne remplace pas l'usage d'une **CI** classique pour les étapes de test ou de build d'images, il s'agit plutôt d'une approche de **Déploiement Continu** (CD).
- L'approche **pull** est plus robuste, mieux adaptée à Kubernetes, et permet une synchronisation continue sans intervention manuelle.

**Important** : Dans une approche **pull** l'utilisation des fonctionnalités de **Merge Request** afin d'effectuer une relecture et une validation des modifications est recommandée.

## Argo CD : principes et fonctionnalités

**Argo CD** est un outil de la catégorie **pull**. Il permet de gérer le déploiement d'applications et la configuration de clusters **Kubernetes** en se basant sur diverses sources : **Git**, **Helm** ou encore **Kustomize**.

**Son principe de fonctionnement est le suivant** : il scrute en permanence les dépôts **Git** enregistrés en tant que sources afin de détecter des modifications et appliquer automatiquement les changements correspondant à la configuration du cluster **Kubernetes** et des applications qu'il héberge.

### Fonctionnalités

- **Déploiement & synchronisation** : Argo déploie les ressources sur le cluster **Kubernetes** en se basant sur la configuration définie dans le dépôt **Git**. Il s'assure que l'état du cluster correspond à l'état souhaité défini dans le dépôt en intéragissant avec l'API **Kubernetes**.
- **Custom Resource Definitions (CRD)** : Argo CD intègre ses propres ressources personnalisées (CRD) pour gérer l'ensemble des configurations de l'outil en tant que code. Quelques exemples de CRD :
  - `Application` : représente une application déployée sur le cluster.
  - `AppProject` : permet de définir des projets d'application, facilitant la gestion des autorisations et des ressources.
  - `ApplicationSet` : permet de déployer plusieurs applications à partir d'un modèle ou d'une configuration commune.
- **UI Web** : interface web intuitive pour visualiser l'état des applications, gérer les déploiements, effectuer des opérations de synchronisation et même consulter les logs des pods.
- **Authentification et SSO** : Argo CD prend en charge plusieurs méthodes d'authentification, y compris l'intégration avec des systèmes d'authentification tiers via **OpenID Connect (OIDC)** ou **LDAP**.
- **Gestion des accès** : Argo CD embarque un système de gestion de permissions basé sur les rôles (RBAC) permettant de contrôler l'accès aux ressources et aux fonctionnalités de l'outil. 
- **Métriques et alertes** : Argo CD expose des métriques **Prometheus** pour surveiller l'état des applications et des clusters. Il est possible de configurer des alertes basées sur ces métriques.
- **Gestion multi-cluster** : possibilité de gérer plusieurs clusters **Kubernetes** à partir d'une seule instance, facilitant ainsi la gestion d'environnements complexes.
- Intégration avec des **Webhooks** : Argo CD peut être configuré pour recevoir des notifications de changements dans les dépôts **Git** via des **webhooks**, permettant ainsi une synchronisation instantanée des modifications.
- **Gestion des secrets** : gestion des secrets Kubernetes

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
- Versionning des configurations d’infrastructure et possibilité de revenir à une version antérieure en cas de problème
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

**Remarque** : il peut être nécessaire de fixer la version du chart à installer en ajoutant `--version <version>` à la commande `helm install` dans le cas d'environnements de production, afin d'éviter les changements de comportement suite à une mise à jour du chart.

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

**Remarque** : pour avoir un déploiement totalement automatisé et reproductible, il est possible d'utiliser le  [provider Helm](https://registry.terraform.io/providers/hashicorp/helm/latest/docs) pour **Terraform**.

## Configuration

**Argo CD** est un outil complet et riche en fonctionnalités : authentification, gestion des accès, gestion d'utilisateurs, gestion des applications, monitoring, etc.

### Authentification

**Argo CD** propose une grande variété de méthodes d'authentifications, allant de l'authentification basique (nom d'utilisateur et mot de passe) à l'intégration de **OpenID Connect** (OIDC) pour une authentification unique (SSO) avec des fournisseurs d'identité tels que **Google**, **GitHub**, **Keycloak**, **LDAP**, etc.

La plupart de ces fonctionnalités sont offertes par [dex](https://dexidp.io/) qui est un composant open source permettant de gérer l'authentification des utilisateurs.

#### LDAP

Il est possible de configurer **Argo CD** pour utiliser un annuaire **LDAP** afin d'y authentifier les utilisateurs et de gérer les groupes d'utilisateurs.

**Exemple de configuration pour le connecteur LDAP** :

```yaml
apiVersion: v1
data:
  dex.config: |
    connectors:
    - type: ldap
      name: LDAP
      id: ldap
      config:
        # Ldap server address
        host: "<LDAP_URL>"
        insecureNoSSL: true
        insecureSkipVerify: true
        # Variable name stores ldap bindDN in argocd-secret
        bindDN: "$dex.ldap.bindDN"
        # Variable name stores ldap bind password in argocd-secret
        bindPW: "$dex.ldap.bindPW"
        usernamePrompt: Username
        # Ldap user search attributes
        userSearch:
          baseDN: "<LDAP_USER_BASE_DN>"
          filter: ""
          username: sAMAccountName
          idAttr: sAMAccountName
          emailAttr: mail
          nameAttr: givenName
        # Ldap group search attributes
        groupSearch:
          baseDN: "<LDAP_GROUP_BASE_DN>"
          filter: "(objectClass=group)"
          userAttr: DN
          groupAttr: member
          nameAttr: cn
  url: https://<YOUR_ARGOCD_URL>
```

L'utilisation du connecteur **LDAP** de dex ([Documentation dex](https://dexidp.io/docs/connectors/ldap/)) permet de configurer les paramètres de connexion à l'annuaire **LDAP**, les attributs des utilisateurs et des groupes, ainsi que les filtres de recherche (pour par exemple autoriser uniquement certains groupes d'utilisateurs à se connecter à **Argo CD**).

#### OpenID Connect (OIDC)

##### Google

Grâce à **dex**, il est possible de configurer **Argo CD** pour utiliser **Google** comme fournisseur d'identité via **OpenID Connect (OIDC)**.

**Exemple de configuration** :

```yaml
apiVersion: v1
data:
  admin.enabled: "true"
  application.instanceLabelKey: argocd.argoproj.io/instance
  dex.config: |
    connectors:
    - config:
        issuer: https://accounts.google.com
        clientID: <google-client-id>
        clientSecret: <google-client-secret>
        redirectURI: https://<YOUR_ARGOCD_URL>/api/dex/callback
      type: oidc
      id: google
      name: Google
  exec.enabled: "false"
  oidc.tls.insecure.skip.verify: "true"
  server.rbac.log.enforce.enable: "false"
  statusbadge.enabled: "false"
  timeout.hard.reconciliation: 0s
  timeout.reconciliation: 180s
  url: https://<YOUR_ARGOCD_URL>
kind: ConfigMap
```

**Quelques points à noter** :

- Le `clientID` et le `clientSecret` sont à récupérer dans la console de gestion des APIs de **Google Cloud**.
- Le `redirectURI` doit correspondre à l'URL de votre instance **Argo CD** suivi de `/api/dex/callback` (représente le point de terminaison de redirection pour l'authentification OIDC).
- Plusieurs fournisseurs d'identité peuvent être configurés dans le même fichier de configuration `dex.config`.
- La récupération des groupes d'utilisateurs peut être configurée mais nécessite un compte de service avec une **délégation au niveau de l'organisation** pour accéder aux groupes d'utilisateurs dans **Google Workspace**.

**Ressource** :

- [Connecteur OIDC - Documentation dex](https://dexidp.io/docs/connectors/oidc/)
- [Authentification avec Google - Argo CD](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/google/#openid-connect-plus-google-groups-using-dex)

### Gestion des accès et permissions

Les fournisseurs d'identité (ex : **LDAP**, **OIDC**) permettant la récupération des utilisateurs et des groupes, **Argo CD** permet à ses administrateurs de définir des rôles et des permissions pour contrôler l'accès aux ressources et aux fonctionnalités de l'outil.

Plusieurs fonctionnalités vont permettre de gérer les politiques **RBAC** (Role-Based Access Control) :

- **Projects** : permettent de regrouper les applications et de définir des politiques globales pour toutes les applications d'un projet.
- **Roles** : définissent les permissions accordées aux utilisateurs ou groupes d'utilisateurs.
- **Role Bindings** : lient les rôles aux utilisateurs ou groupes d'utilisateurs, leur accordant ainsi les permissions définies dans le rôle.
- **Policies** : définissent les règles d'accès aux ressources et aux fonctionnalités d'Argo CD, en fonction des rôles et des utilisateurs.

### Projets

Un projet dans **Argo CD** est une ressource qui se déclare d'une manière similaire à une application. La différence réside évidemment dans le contenu de la ressource qui permet de :

- Définir les `namespaces` sources et destination autorisés pour les applications du projet
- Définir les dépôts **Git** autorisés pour les applications du projet
- Définir des rôles et permissions spécifiques au projet
- Définir des **fenêtres de maintenance** pour les applications du projet :
  - Permet de limiter les opérations de synchronisation pendant une période donnée (ex : pendant les heures de travail)
  - Utile pour éviter les modifications pendant des périodes critiques (ex : maintenance, déploiement de nouvelles versions)
- Autoriser ou restreindre l'usage de certaines ressources Kubernetes (ex : `ConfigMap`, `Secret`, `ServiceAccount`, etc.) pour les applications du projet

**Aperçu du contenu d'un projet :**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: example-app-project
  namespace: argocd
spec:
    description: ArgoCD project for an example application
    sourceRepos:
    - '*'
    destinations:
    - namespace: <namespace_1>
      server: https://kubernetes.default.svc
    - namespace: <namespace_2>
      server: https://kubernetes.default.svc
    - namespace: "!kube-system" # Exclut le namespace kube-system
      server: "*"
    clusterResourceWhitelist: []
    clusterResourceBlacklist: []
    namespaceResourceBlacklist:
    - group: ''
      kind: ResourceQuota
    - group: ''
      kind: LimitRange
    - group: ''
      kind: NetworkPolicy
      orphanedResources: {}
      roles: []
    namespaceResourceWhitelist:
    - group: 'apps'
      kind: Deployment
    - group: 'apps'
      kind: StatefulSet
    orphanedResources: {}
    roles: []
    syncWindows: 
    - kind: allow
      schedule: '10 1 * * *'
      duration: 1h
      applications:
      - '*-prod'
      manualSync: true
    signatureKeys:
    - keyID: ABCDEF1234567890
    sourceNamespaces:
    - ${argocd_namespace}
    - ${argocd_namespace}
```

**Quelques explications** :

- **orphanedResources** : permet de définir les ressources orphelines (ressources qui ne sont plus gérées par Argo CD) à prendre en compte ou à ignorer lors de la synchronisation.
- **signatureKeys** : permet de définir les clés de signature utilisées pour vérifier l'intégrité des applications déployées.

Plus d'informations dans la [documentation de Argo CD](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/).

#### Les politiques de sécurité

Les politiques de sécurité dans **Argo CD** permettent de définir des règles d'accès aux ressources et aux fonctionnalités de l'outil. Elles sont définies dans le fichier de configuration `argocd-rbac-cm` dans le `namespace` **argocd**.

On va pouvoir attribuer des rôles par défaut tels que `admin` et `readonly`, mais aussi définir des rôles personnalisés.

Chaque **ressource** (ex : `Application`, `Project`, `ApplicationSet`, etc.) possède des permissions spécifiques qui peuvent être accordées ou restreintes aux utilisateurs et groupes d'utilisateurs :

| Ressource       | get | create | update | delete | sync | action | override | invoke |
|-----------------|:---:|:------:|:------:|:------:|:----:|:------:|:--------:|:------:|
| applications    | ✅  | ✅     | ✅     | ✅     | ✅   | ✅     | ✅       | ❌     |
| applicationsets | ✅  | ✅     | ✅     | ✅     | ❌   | ❌     | ❌       | ❌     |
| clusters        | ✅  | ✅     | ✅     | ✅     | ❌   | ❌     | ❌       | ❌     |
| projects        | ✅  | ✅     | ✅     | ✅     | ❌   | ❌     | ❌       | ❌     |
| repositories    | ✅  | ✅     | ✅     | ✅     | ❌   | ❌     | ❌       | ❌     |
| accounts        | ✅  | ❌     | ✅     | ❌     | ❌   | ❌     | ❌       | ❌     |
| certificates    | ✅  | ✅     | ❌     | ✅     | ❌   | ❌     | ❌       | ❌     |
| gpgkeys         | ✅  | ✅     | ❌     | ✅     | ❌   | ❌     | ❌       | ❌     |
| logs            | ✅  | ❌     | ❌     | ❌     | ❌   | ❌     | ❌       | ❌     |
| exec            | ❌  | ✅     | ❌     | ❌     | ❌   | ❌     | ❌       | ❌     |
| extensions      | ❌  | ❌     | ❌     | ❌     | ❌   | ❌     | ❌       | ✅     |

*Tableau tiré de la [documentation officielle](https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/)*

Dès lors, il est possible de définir des rôles et des permissions pour les utilisateurs et groupes en se basant sur ces différents types de droits pour chaque ressource.

**Exemple de définition de rôle** :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-rbac-cm
    app.kubernetes.io/part-of: argocd
data:
  policy.csv: |

    p, role:developer, applications, *, my-project/*, allow # Permet aux développeurs de gérer les applications de leur projet
    p, role:developer, applications, get,*, allow # Permet aux développeurs de récupérer les applications de tous les projets
    p, role:developer, applicationsets, get,*, allow # Permet aux développeurs de récupérer les ApplicationSets de tous les projets
    p, role:developer, repositories, get,*, allow # Permet aux développeurs de récupérer les dépôts de tous les projets
    p, role:developer, projects, get,*, allow # Permet aux développeurs de récupérer les projets de tous les projets
    p, role:developer, clusters, get,*, allow # Permet aux développeurs de récupérer les clusters de tous les projets

    g, developer@example, role:developer # Associe le groupe developer@example à ce rôle
    g, user@example.org, role:admin
  policy.default: role:readonly
  scopes: '[groups, email]'
```

**Remarque** : Un rôle par défaut peut être définir avec `policy.default` et sera appliqué à tous les utilisateurs qui ne sont pas explicitement associés à un rôle.

### Monitoring

*A venir*

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

Il est possible d'utiliser des **charts Helm** comme source d'application. Dans ce cas, il faut ajouter la section `helm` dans la définition de l'application :

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

- En utilisant la section `values` ou `valuesObject` dans la définition de l'application (comme ci-dessus)
- En utilisant un fichier **YAML** externe (ex : `values.yaml`) en l'ajoutant dans la section `valuesFiles`.

Plus d'informations sont disponibles dans la section [Helm](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/) de la documentation.

### Déployer plusieurs applications

#### App of Apps

Bien qu'il soit possible d'utiliser l'interface web pour créer chaque application une à une, il est préférable de le faire via des fichiers **YAML** stockés dans un dépôt **Git**. Cela permet de versionner la configuration des applications et de les gérer comme n'importe quelle autre ressource **Kubernetes**.

Ce type d'implémentation est appelé **App of Apps**, c'est à dire qu'à partir d'une application Argo CD, une multitudes d'autres applications seront déployées car définies dans des fichiers **YAML** stockés dans un dépôt **Git** scruté par l'application racine.

#### ApplicationSet

Une autre solution pour déployer plusieurs applications est d'utiliser un **ApplicationSet**. Il s'agit d'une ressource permettant de définir des **templates** d'applications et de générer plusieurs applications à partir de ceux-ci.

On va pouvoir les utiiliser dans plusieurs cas :

- Déployer plusieurs instances d'une même application avec des configurations différentes (ex : différents environnements)
- Déployer des applications à partir de modèles définis dans un dépôt **Git**
- Déployer des applications dans plusieurs clusters 

**Exemple d'ApplicationSet basé sur un dépôt Git** :

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: apps-by-dir
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/mon-org/mon-repo.git
        revision: main
        directories:
          - path: apps/*
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/mon-org/mon-repo.git
        targetRevision: main
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

De nombreux générateurs sont disponibles pour l'`ApplicationSet` et sont détaillés dans la [documentation officielle](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/).

### Gestion des secrets

*A venir*

## Références

- [Documentation officielle d'Argo CD](https://argo-cd.readthedocs.io/en/stable/)
- [ArgoCD de A à Y - Une tasse de café](https://une-tasse-de.cafe/blog/argocd/)
- [Flux et ArgoCD, deux visions du GitOps sur Kubernetes - WeScale](https://blog.wescale.fr/flux-et-argocd-deux-visions-du-gitops-sur-kubernetes)
- [Argo CD, qu'est-ce que c'est ? - Red Hat](https://www.redhat.com/fr/topics/devops/what-is-argocd)
- [ArgoCD Tutorial for Beginners | GitOps CD for Kubernetes - TechWorld with Nana](https://www.youtube.com/watch?v=MeU5_k9ssrs)
