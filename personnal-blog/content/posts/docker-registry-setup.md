---
date: '2024-10-15T00:00:00+01:00'
draft: false
title: 'Mise en place et utilisation de Docker Registry'
tags: ['docker']
---

`Docker Registry` est un service permettant de stocker et de distribuer des images Docker, facilitant leur partage et leur gestion dans des environnements de développement et de production.

## Mise en place

Une méthode simple consiste à installer la `Docker Registry` en tant que conteneur via `docker-compose`. 

`Prérequis` : avant d'installer le service, il faut définir l'endroit où la `registry` enregistrera les données. Dans mon cas, je choisis d'utiliser un point de montage `data` sur mon `nas` afin d'assurer la persistance des éléments. 

Fichier `docker-compose.yaml` :

```yaml
version: '3'

services:
  registry:
    image: registry:2
    ports:
    - "5000:5000"
    environment:
      REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /Data
    volumes:
      - /Data:/Data
```

Une fois défini, il faut exécuter la commande suivante : 

```bash
docker compose up -f docker-compose.yaml -d
```

## Mettre en place l'authentification

**Prérequis :** configuration de reverse proxy comme `nginx`.

**Exemple** :

```nginx
  server {
    listen 80;

    server_name docker-registry.home www.docker-registry.home;

    location / {
      proxy_pass http://docker-registry:5000;
      proxy_set_header  Host              $http_host;   # required for docker client's sake
      proxy_set_header  X-Real-IP         $remote_addr; # pass on real client's IP
      proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
      proxy_set_header  X-Forwarded-Proto $scheme;
    }
  }
```

Les `headers` sont nécessaires pour l'authentification auprès de la `registry`.

Utiliser `htpasswd` du paquet `apache2-utils` pour créer un fichier d'authentification : 

```bash
sudo apt install apache2-utils -y
```

Une fois installé, on peut créer un utilisateur dans un répertoire dédié, ici on va créer un répertoire `auth` qui sera monté comme volume sur le conteneur :

```bash
mkdir ~/docker-registry/auth # Créer le répertoire qui contient les richiers d'auth
cd ~/docker-registry/auth
sudo htpasswd -B registry.password [username]
```

Il faut aussi modifier le `docker-compose.yaml` : 

```yaml
version: '3'

services:
  registry:
    image: registry:2
    restart: always
    ports:
    - "5000:5000"
    environment:
	  REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: Registry
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/registry.password
      REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /Data
    volumes:
	  - ./auth:/auth
      - /Data:/Data
```

### Tester une registry via `http` 

`docker login` utilise par défaut `https`, donc si `nginx` ne sert que du `http` il faut créer une exception.

Pour cela, il faut ajouter la `regitry` au fait fichier `/etc/docker/daemon.json`:

```json
{
   "insecure-registries": ["<registry>:<port>"]
}
```

**Remarque** : sous **Windows** (pour **docker desktop**) le fichier se trouve sous `%userprofile%\.docker\daemon.json`.

Via `docker desktop` :

![[docker-desktop-daemong-config.png]]

Ensuite,; il faut adapter la commande `docker-login` pour utiliser le port `80` : 

```bash
docker login docker-registry.home:80
```

## `Insecure registry` sur Kubernetes

Ajouter dans `/etc/default/docker.json` :

```bash
{
    "insecure-registries": ["<regsitry>:<port>"]
}
```

Ensuite, il faut créer un `secret` qui contiendra les éléments pour s'authentifier à la registry : 

```bash
kubectl create secret docker-registry docker-cred \
    --docker-server=https://index.docker.io/v1/ \
    --docker-username=your-username \
    --docker-password=your-password
```

## Utiliser la registry

### Se connecter à Docker Registry

```bash
docker login [registry_url]
```

Les informations seront stockées dans `~/.docker/config.json`.

### Pousser une image sur la registry

Avant de pousser une image vers un registre, il faut la tagger avec l'URL du registre.

La structure de la commande sera la suivante :

```bash
docker tag [nom_image_locale] [url_du_registry]/[nom_utilisateur]/[nom_image]:[tag]
```

**Exemple :**

```bash
docker tag my-app myregistry.example.com/myuser/my-app:latest
```

Enfin, on peut pousser l'image avec la commande `push` :

```bash
docker push <url_du_registry>/<nom_utilisateur>/<nom_image>:<tag>
```

**Remarque** : si l'authentification est nécessaire, il faut ajouter le mot de passe via la variable `auth` dans `~/.docker/config.json` :

```bash
{
	"auth": "mdp"
}
```

**Erreur liée** : `push access denied, repository does not exist or may require authorization: authorization failed: no basic auth credentials`

### Récupérer une image depuis la registry

```bash
docker pull <url_du_registry>/<nom_utilisateur>/<nom_image>:<tag>
docker pull docker-registry.home:80/guillaume/hello-world-api:v1
```

### Lister les images (repositories) disponibles sur la registry

```bash
curl -X GET https://myregistry:5000/v2/_catalog
```

Exemple si authentification nécessaire :

```bash
curl -X GET -u guillaume:<password> http://docker-registry.home:80/v2/_cata
log
```

Pour lister les tags d'une image : 

```bash
curl -X GET https://myregistry:5000/v2/ubuntu/tags/list
```

## Références

- [Image Officielle de Docker Registry](https://hub.docker.com/_/registry)
- [How To Set Up a Private Docker Registry on Ubuntu 20.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-private-docker-registry-on-ubuntu-20-04)
- [UI for docker registry](https://github.com/Joxit/docker-registry-ui)
- [A Guide to Docker Private Registry](https://www.baeldung.com/ops/docker-private-registry)