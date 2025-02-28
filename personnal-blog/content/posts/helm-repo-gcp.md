---
date: '2024-11-16T00:00:00+01:00'
draft: false
title: 'Dépôt Helm avec Google Cloud Storage'
tags: ['kubernetes', 'cloud', 'helm', 'gcp']
---

Il est possible d'utiliser `Cloud Storage` en tant que dépôt pour les `charts` Helm.

## Créer un bucket sur GCP

Le code `terraform` peut être le suivant : 

```HCL
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

Importez vos fichiers 

## Créer un `chart`

Créer un `chart` :

```bash
helm create [chart_name]
```

Packager le `chart` :

```bash
helm package /path/to/chart/directory
```

Ajouter ce chart dans le **bucket**.

## Configurer le fichier `index.yaml` du dépôt

Générer un fichier `index.yaml` depuis un répertoire :

```bash
helm repo index .
```

Résultat :

```bash
apiVersion: v1
entries:
  myapp:
  - apiVersion: v1
    created: "2025-01-06T22:23:38.311373009+01:00"
    description: Hello World API
    digest: 506b474ee123e445d644fc3e1a5eb331d8629e59db1b1bcba8e3419dde684b22
    name: myapp
    urls:
    - myapp-1.0.0.tgz
    version: 1.0.0
generated: "2025-01-06T22:23:38.311185591+01:00"
```

Ajouter l'URL du **bucket** avant le nom du fichier sous `urls` : 

```bash
urls:
- https://storage.googleapis.com/guillaume-helm-repo-3/myapp-1.0.0.tgz
```

Une fois complété, il faut ajouter le fichier dans le **bucket** :

```bash
gsutil cp /chemin/vers/le/fichier.ext gs://[bucket-name]/
```

## Ajouter le bucket en tant que dépôt

Ajouter le **bucket** aux dépôts :

```bash
 helm repo add [repo-name] https://storage.googleapis.com/[bucket-name]
 helm repo update
```

Ca fonctionne :

```bash
helm install myapp guillaume-repo-2/myapp --version 1.0.0
```

## Références

- [Helm | The Chart Repository Guide](https://helm.sh/docs/topics/chart_repository/)
- [Creating a Helm Chart Repository on Google Cloud Storage | jimangel.io](https://www.jimangel.io/posts/create-helm-chart-repo-gcs/)