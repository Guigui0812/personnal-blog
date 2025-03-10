---
date: '2024-11-16T00:00:00+01:00'
draft: false
title: 'Créer un dépôt Helm sur Google Cloud Storage'
categories: ['kubernetes', 'cloud', 'helm', 'gcp']
---

Il est possible d'utiliser **Google Cloud Storage** en tant que dépôt pour les **charts Helm**.

L'intérêt est de rapidement rendre disponible des **charts** pour des applications ou des services depuis un **bucket** accessible en ligne, sans une configuration complexe.

## Créer un bucket sur GCP

Le **bucket GCS** peut être créé via la **Console GCP** ou via **Terraform** :

```ruby
provider "google" {
  project = var.project_id
  region  = var.region
}

resource "google_storage_bucket" "helm_repo" {
  name          = var.bucket_name
  location      = var.region
  storage_class = "STANDARD"

  versioning {
    enabled = true
  }

  uniform_bucket_level_access = true

  lifecycle_rule {
    action {
      type = "Delete"
    }
    condition {
      age = 365
    }
  }
}

resource "google_storage_bucket_iam_member" "helm_repo_admin" {
  bucket = google_storage_bucket.helm_repo.name
  role   = "roles/storage.objectAdmin"
  member = "serviceAccount:${var.service_account_email}"
}

output "bucket_url" {
  value = "https://storage.googleapis.com/${google_storage_bucket.helm_repo.name}"
}
```

Une fois le **bucket** créé, il est temps de créer un **chart**.

## Création d'un chart Helm

La création d'un **chart Helm** passe par la commande `helm create` :

```bash
helm create [chart_name]
```

Une fois créé, on va modifier le fichier `Chart.yaml` pour ajouter des informations sur le **chart** :

```yaml
apiVersion: v2
name: nginx-sample
description: A nginx sample chart for Kubernetes
version: 1.0.0
appVersion: 1.0.0
```

Il faut ensuite ajouter différentes ressources dans le répertoire `templates`. Par exemple un fichier `deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-sample
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-sample
  template:
    metadata:
      labels:
        app: nginx-sample
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

Lorsque les différents **templates** sont ajoutés, il est possible de **packager** le **chart**.

Cette étape passe par l'usage de la commande `helm package` :

```bash
helm package /path/to/chart/directory
```

On obtient alors un fichier `.tgz` qui peut être ajouté au **bucket**.

## Configurer le fichier `index.yaml` du dépôt

Avant d'ajouter le fichier `.tgz` au **bucket**, il faut générer un fichier `index.yaml` qui va référencer les différents **charts** disponibles. C'est cette étape qui permet de transformer un simple **bucket** en dépôt **Helm**.

La génération du fichier `index.yaml` passe par la **cli Helm** en utilisant la commande `helm repo index` depuis le répertoire contenant les **charts** (parent du répertoire du **chart ngnix-sample**).

```bash
helm repo index .
```

Le résultat obtenu est un fichier `index.yaml` référançant le **chart** `nginx-sample` :

```yaml
apiVersion: v1
entries:
  nginx-sample:
  - apiVersion: v1
    created: "2025-01-06T22:23:38.311373009+01:00"
    description: Hello World API
    digest: 506b474ee123e445d644fc3e1a5eb331d8629e59db1b1bcba8e3419dde684b22
    name: nginx-sample
    urls:
    - nginx-sample-1.0.0.tgz
    version: 1.0.0
generated: "2025-01-06T22:23:38.311185591+01:00"
```

Avant d'ajouter le fichier `index.yaml` et le **chart** au **bucket**, il faut modifier le fichier `index.yaml` pour ajouter l'URL du **bucket**.

Elle doit être rajoutée dans la section `urls` :

```yaml
...
    urls:
    - https://storage.googleapis.com/guillaume-helm-repo-3/nginx-sample-1.0.0.tgz
```

Dès lors, il faut importer le fichier `index.yaml` et le **chart** dans le **bucket**.

Pour cela, il est recommandé d'utiliser l'utilitaire `gsutil` qui permet de manipuler les **buckets** GCS depuis la ligne de commande :

```bash
gsutil cp index.yaml gs://[bucket-name]
gsutil cp nginx-sample-1.0.0.tgz gs://[bucket-name]
```

Le **bucket** est ainsi transformé en dépôt **Helm** et contient le **chart** `m

## Utiliser le dépôt Helm

Afin d'utiliser le **bucket** comme dépôt **Helm**, il faut d'abord l'ajouter aux dépôts du **cluster** Kubernetes :

```bash
 helm repo add [repo-name] https://storage.googleapis.com/[bucket-name]
 helm repo update
```

L'étape suivante consiste à installer le **chart** depuis le dépôt :

```bash
helm install nginx-sample [repo-name]/nginx-sample
```

Et voilà, en quelques étapes, il est possible de créer un dépôt **Helm** sur **Google Cloud Storage** afin de distribuer des **charts** pour **Kubernetes**.

## Références

- [Helm | The Chart Repository Guide](https://helm.sh/docs/topics/chart_repository/)
- [Creating a Helm Chart Repository on Google Cloud Storage | jimangel.io](https://www.jimangel.io/posts/create-helm-chart-repo-gcs/)