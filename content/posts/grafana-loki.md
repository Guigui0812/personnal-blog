---
date: '2025-03-11T00:00:00+01:00'
draft: false
title: 'Grafana Loki : une stack complète pour la gestion des logs'
categories: ['monitoring', 'grafana', 'logs', 'loki', 'promtail', 'docker']
---

**Grafana Loki** est un système de gestion de logs, qui permet de stocker et de rechercher des logs de manière efficace. Il est conçu pour être utilisé avec **Promtail** et **Grafana** pour une solution de monitoring complète.

**Promtail** est un agent qui collecte les logs et les envoie à **Loki**.

**Grafana** est un outil de visualisation de données initialement connu pour ses tableaux de bord de métriques, mais qui peut également être utilisé pour afficher des logs.

L'ensemble des configurations détaillées ont été automatisées via des rôles **Ansible** stockés dans ces dépôts :

- [GitHub - Guigui0812/ansible-promtail](https://github.com/Guigui0812/ansible-promtail)
- [GitHub - Guigui0812/ansible-loki](https://github.com/Guigui0812/ansible-loki)
- [GitHub - Guigui0812/grafana-ansible](https://github.com/Guigui0812/grafana-ansible)

## Loki - Installation et configuration

On peut installer **Loki** de différentes manières, mais la plus simple est d'utiliser **docker-compose** :

```yaml
services:
  loki:
    user: <loki_user>:<loki_user>
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    volumes:
      - ./loki-local-config.yaml:/etc/loki/local-config.yaml
      - /var/loki:/data
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - loki_network
```

**Explications** :

- `user` : permet de spécifier un utilisateur pour le conteneur, pour des raisons de sécurité afin de ne exécuter le conteneur en tant que **root**, ce qui constitue une faille de sécurité.
- `image` : utilisant l'image de la dernière version de **Loki**.
- `ports` : par défaut, **Loki** écoute sur le port **3100**.
- `volumes` : monte le fichier de configuration local de **Loki** et le répertoire de stockage des données (pour la persistance en cas de redémarrage ou de suppression du conteneur).
- `command` : exécute **Loki** avec le fichier de configuration local.
- `networks` : permet de spécifier un réseau pour le conteneur (optionnel si on veut utiliser le nom du conteneur pour la communication entre les conteneurs du même hôte).

**Remarque** : l'utilisateur `loki` dédié doit avoir les permissions nécessaires pour accéder aux fichiers et répertoires montés.

Le fichier `loki-local-config.yaml` va contenir la configuration de **Loki**. Dans le cadre de mon **homelab**, j'utilise la configuration suivante :

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory
  replication_factor: 1
  path_prefix: /tmp/loki

schema_config:
  configs:
  - from: 2025-02-26
    store: tsdb
    object_store: filesystem
    schema: v13
    index:
      prefix: index_
      period: 24h

storage_config:
  filesystem:
    directory: /tmp/loki/index

limits_config:
  retention_period: 672h
  max_query_lookback: 672h

compactor:
  working_directory: /tmp/loki/retention
  retention_enabled: true
  retention_delete_delay: 2h
  delete_request_store: filesystem
```

**Explications** :

- `auth_enabled` : permet d'activer l'authentification pour **Loki**.
- `server` : permet de spécifier le port d'écoute pour **Loki**.
- `common` (configuration commune) :
  - `ring` (anneau de stockage) :
    - `instance_addr` : permet de spécifier l'adresse de l'instance de **Loki**.
    - `kvstore` : permet de spécifier le type de stockage des clés-valeurs (inmemory pour un stockage en mémoire). **inmemory** est pratiques sur de petits déploiements, mais pour des déploiements plus importants, il est recommandé d'utiliser un stockage externe.
  - `replication_factor` : permet de spécifier le facteur de réplication des données.
  - `path_prefix` : permet de spécifier le préfixe des chemins pour **Loki**. Il doit être le même pour tous les chemins de **Loki** définis dans le fichier de configuration (`/tmp/loki`, `/tmp/loki/index`, `/tmp/loki/retention`). Dans le cas contraire, cela peut entraîner des erreurs lors du démarrage de **Loki**.
- `schema_config` (configuration du schéma) :
  - `configs` : permet de spécifier la configuration du schéma.
    - `from` : permet de spécifier la date à partir de laquelle le schéma est utilisé.
    - `store` : permet de spécifier le type de stockage (tsdb pour un stockage en base de données de séries temporelles).
    - `object_store` : permet de spécifier le type de stockage des objets (filesystem pour un stockage sur le système de fichiers). Il est possible d'utiliser d'autres types de stockage comme **GCS** ou **S3**.
    - `schema` : permet de spécifier la version du schéma.
    - `index` : permet de spécifier la configuration de l'index.
      - `prefix` : permet de spécifier le préfixe de l'index.
      - `period` : permet de spécifier la période de l'index.
- `storage_config` (configuration du stockage) :
  - `filesystem` : permet de spécifier le répertoire de stockage des données.
- `limits_config` (configuration des limites) :
  - `retention_period` : permet de spécifier la période de rétention des données. Après cette période, les données sont supprimées. Cela permet de gérer l'espace disque utilisé par **Loki** et de ne pas conserver indéfiniment les données.
  - `max_query_lookback` : permet de spécifier la période maximale de recherche. Cela permet de limiter la recherche à une période donnée afin de ne pas surcharger le système.
- `compactor` (compacteur qui supprime les données après que la période de rétention soit dépassée) :
  - `working_directory` : permet de spécifier le répertoire de travail du compacteur.
  - `retention_enabled` : permet d'activer la rétention des données.
  - `retention_delete_delay` : permet de spécifier le délai de suppression des données après la rétention.
  - `delete_request_store` : permet de spécifier le type de stockage des demandes de suppression.

Une fois **Loki** configuré, on peut le démarrer avec **docker-compose** :

```bash
docker-compose up -d
```

**Loki** est ainsi prêt à recevoir des logs en provenance de **promtail**.

La documentation de **Loki** détaille les différentes configurations possibles : [Grafana Loki configuration parameters | Grafana Loki documentation](https://grafana.com/docs/loki/latest/configure/)
### Configuration avancée

#### Ajout du plugin **docker** à **Loki**

Pour que **Loki** soit en capacité de récupérer les logs des conteneurs **docker**, il est nécessaire d'ajouter le plugin **docker** à **Loki**.

L'ajout se fait via la commande suivante :

```bash
docker plugin install grafana/loki-docker-driver:3.3.2-arm64 --alias loki --grant-all-permissions
```

Plus d'informations dans la documentation de **Loki** : [Docker Driver | Grafana Loki documentation](https://grafana.com/docs/loki/latest/send-data/docker-driver/)

### Bonnes pratiques

La documentation de **Loki** liste quelques bonnes pratiques : [Configuration best practices | Grafana Loki documentation](https://grafana.com/docs/loki/latest/configure/bp-configure/)

#### Authentification

*A venir*

#### Stockage et rétention des logs

- Afin d'éviter une saturation de l'espace disque, il est important de configurer la rétention des logs via `limits_config` dans le fichier de configuration de **Loki**.
- Suppression des logs après la période de rétention configurée dans `limits_config` via le `compactor`. Possibilité de configurer un délai de suppression après la période de rétention pour éviter une suppression prématurée des logs.
- Dans le cas de la production, il est recommandé d'utiliser un stockage externe pour les logs pour des questions de redondance et de performance : **GCS**, **S3**, **Minio**.
- Le **ring** de **Loki** permet de stocker les logs en mémoire. Il est recommandé d'utiliser un stockage externe pour les données persistantes : **consul**, **etcd**, etc.

## Promtail

### Installation

On peut installer **promtail** de différentes manières :

- **Docker** : en utilisant l'image officielle de **promtail**.
- **Binaire** : en téléchargeant le binaire de **promtail** et en l'exécutant directement.
- **Source** : en compilant les sources de **promtail** et en l'exécutant directement.

La méthode la plus propre est d'installer **promtail** en tant qu'agent sur la machine à surveiller via le gestionnaire de paquets de la distribution. Cela permet d'enlever une couche de complexité, notamment concernant les permissions d'accès aux logs ou des configurations supplémentaires pour la récupération des logs de conteneurs.

Installation de **promtail** sur **Debian** :

```bash
sudo apt install promtail
```

Installation avec `docker compose` :

```yaml
services:
  promtail:
    user: <promtail_user>:<promtail_user>
    group_add : 
      - <log_reader_group>
    image: grafana/promtail:latest
    volumes:
      - /var/log:/var/log
      - ./config.yaml:/etc/promtail/config.yaml
    command: -config.file=/etc/promtail/config.yaml
```

**Explications** :

- `user` : permet de spécifier un utilisateur pour le conteneur, pour des raisons de sécurité afin de ne exécuter le conteneur en tant que **root**, ce qui constitue une faille de sécurité.
- `group_add` : permet d'ajouter le groupe de lecture des logs dans le conteneur pour que **promtail** puisse lire les logs (montés dans le conteneur via le volume `/var/log`).
- `volume` : monte les logs de la machine hôte dans le conteneur pour que **promtail** puisse les lire.
  - `/var/log` : monte les logs de la machine hôte dans le conteneur.
  - `./config.yaml` : monte le fichier de configuration local de **promtail**.

**Remarque** : les `id` de l'utilisateur et du groupe doivent correspondre à ceux de l'utilisateur et du groupe sur la machine hôte pour que **promtail** puisse lire les logs.

### Configuration

La configuration de **promtail** se fait via un fichier de configuration `config.yaml`. Ce fichier contient la configuration de **promtail** pour la récupération des logs et l'envoi à **Loki**.

Dans le cadre de mon **homelab**, j'utilise la configuration suivante :

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: <loki_url>

scrape_configs:
  - job_name: syslog
    static_configs:
    - targets:
        - "localhost"
      labels:
        job: syslog
        host: <hostname>
        __path__: /var/log/syslog
  - job_name: system
    static_configs:
    - targets:
        - "localhost"
      labels:
        job: system
        host: <hostname>
        __path__: /var/log/auth.log
  - job_name: lastlog
    static_configs:
    - targets:
        - "localhost"
      labels:
        job: lastlog
        host: <hostname>
        __path__: /var/log/lastlog
  - job_name: containers
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: 'container'
```

**Explications** :

- `server` : permet de spécifier le port d'écoute pour **promtail** (par défaut, **promtail** écoute sur le port **9080**).
- `positions` : permet de spécifier le fichier de position pour **promtail**. Ce fichier contient les positions des logs lus par **promtail** pour ne pas les relire à chaque redémarrage.
- `clients` : permet de spécifier les clients de **promtail** (dans ce cas, **Loki**).
  - `url` : permet de spécifier l'URL de **Loki**.
- `scrape_configs` : permet de spécifier les configurations de récupération des logs.

Dans cet exemple, on récupère les logs **syslog**, **auth.log** et **lastlog** de la machine locale, ainsi que les logs des conteneurs **docker**.

#### Récupération des logs système

Dans `config.yaml` on va pouvoir spécifier différents jobs `scrape_configs` pour récupérer les logs de différents services ou fichiers de logs.

Dans le cas de fichiers de logs classiques, on utilise `static_configs` pour spécifier les cibles à surveiller :

```yaml
...
scrape_configs:
  - job_name: syslog
    static_configs:
    - targets:
        - "localhost"
      labels:
        job: syslog
        host: <hostname>
        __path__: /var/log/syslog
```

**Explications** :

- `job_name` : permet de spécifier le nom du job.
- `static_configs` (type de job) :
  - `targets` : permet de spécifier les cibles à surveiller (dans ce cas, `localhost`).
  - `labels` :
    - `job` (obligatoire): permet de spécifier le job pour les logs.
    - `host` (label personnalisé) : permet de spécifier le nom de l'hôte pour les logs.
    - `__path__` (obligatoire) : permet de spécifier le chemin du fichier de logs à surveiller.

Plus d'informations sur **static_configs** : [Configure Promtail | Grafana Loki documentation](https://grafana.com/docs/loki/latest/send-data/promtail/configuration/#scrape_configs)

#### Récupération des logs des conteneurs

Afin de récupérer les logs des conteneurs **docker**, on utilise `docker_sd_configs`.

Cette partie possède cependant des **prérequis** : 

- **docker** doit être configuré pour utiliser le **driver** de logs **json-file** ou **journald**.
- Le **plugin** **docker** doit être ajouté à **Loki**.

Cette configuration se fait dans le fichier `/etc/docker/daemon.json` :

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

Plus d'informations sur la configuration des drivers de logs **docker** : [JSON File logging driver | Docker Docs](https://docs.docker.com/engine/logging/drivers/json-file/)

Exemple de configuration pour récupérer les logs des conteneurs **docker** :

```yaml
...
scrape_configs:
  - job_name: containers
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: 'container'
```

**Explications** :

- `job_name` : permet de spécifier le nom du job.
- `docker_sd_configs` (type de job) :
  - `host` : permet de spécifier l'URL du socket **docker** afin que **promtail** puisse récupérer les logs des conteneurs.
  - `refresh_interval` : permet de spécifier l'intervalle de rafraîchissement pour la récupération des logs.
- `relabel_configs` : permet de spécifier les configurations de relabeling des logs.
  - `source_labels` : permet de spécifier les labels sources pour le relabeling.
  - `regex` : permet de spécifier une expression régulière pour extraire le nom du conteneur.
  - `target_label` : permet de spécifier le label cible pour le relabeling.

**Important** : l'utilisateur **promtail** doit être membre du groupe **docker** afin d'accéder à la socket **docker**.

Plus d'informations sur **docker_sd_configs** : [Configure Promtail | Grafana Loki documentation](https://grafana.com/docs/loki/latest/send-data/promtail/configuration/#docker_sd_configs)

**Remarque** : il est possible de récupérer les logs via **static_configs** en récupérant les logs des conteneurs **docker** se trouvant dans `/var/lib/docker/containers`. 

Exemple avec **static_configs** :

```yaml
...
scrape_configs:
  - job_name: containers
    static_configs:
    - targets:
        - "localhost"
      labels:
        job: containers
        host: <hostname>
        __path__: /var/lib/docker/containers/*/*.log
```

## Visualisation des logs avec Grafana

Une fois **Loki** et **promtail** configurés, on peut visualiser les logs avec **Grafana**. Pour cela on doit ajouter **Loki** comme source de données dans **Grafana** via les **datasources**. Ensuite, il va être possible de créer un **dashboard** pour visualiser les logs. 

A l'instar de **Prometheus**, **Loki** utilise un langage de requête **LogQL** pour interroger les logs. **LogQL** est similaire à **PromQL** mais adapté pour les logs.

### Syntaxe de LogQL

**LogQL** permet de rechercher des logs en utilisant des expressions de requête. Voici quelques exemples de requêtes **LogQL** :

- Rechercher des logs contenant un texte spécifique :

```logql
{job="syslog"} |= "texte"
```

- Rechercher des logs contenant plusieurs textes :

```logql
{job="syslog"} |= "texte1" | "texte2"
```

- Filtrer sur des labels :

```logql
{job="syslog"} # Un seul label
{job="syslog", host="hostname"} # Plusieurs labels
```

- Compter le nombre de logs :

```logql
count_over_time({job="syslog"} |="texte" [5m])
```

## A venir

- Plus d'exemples de requêtes **LogQL**.
- Sécurisation de **Loki** et **Promtail** via l'authentification.
- Plus de détails sur la configuration du stockage et de la rétention des logs.

## Références

- [Grafana Loki | Grafana Loki documentation](https://grafana.com/docs/loki/latest/)
- [Grafana Loki - Stéphane Robert](https://blog.stephane-robert.info/docs/observer/logs/loki/)
- [Implementing the logging stack using Promtail, Loki, and Grafana using Docker-Compose. | by NetOpsChic | Medium](https://medium.com/@netopschic/implementing-the-log-monitoring-stack-using-promtail-loki-and-grafana-using-docker-compose-bcb07d1a51aa)
- [Monitoring Docker Containers with Grafana, Loki, and Promtail | by Abhiraj Thakur | Medium](https://abhiraj2001.medium.com/monitoring-docker-containers-with-grafana-loki-and-promtail-4302a9417c0d)
- [Enabling Log Retention in Loki with Compactor | by Aditya Hilman | Medium](https://medium.com/@aditya.hilman_10961/enabling-log-retention-in-loki-with-compactor-fe3de2002b47)