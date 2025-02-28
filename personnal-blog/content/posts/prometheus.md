---
date: '2024-12-28T00:00:00+01:00'
draft: false
title: 'Le monitoring avec Prometheus et Grafana'
categories: ['monitoring', 'prometheus', 'grafana']
---

**Prometheus** est un outil open-source de surveillance et de gestion des métriques conçu pour collecter, stocker et analyser des données en temps réel. Il est particulièrement populaire dans les environnements cloud et DevOps grâce à son intégration facile avec des systèmes modernes comme Kubernetes.

Afin de visualiser les **métriques** collectées par `prometheus`, on utilise souvent **Grafana** qui permet de créer des tableaux de bord personnalisés.

## Caractéristiques et fonctionnalités

### Collecte de métriques

`Prometheus` fonctionne en mode `pull`. Il extrait des métriques en interrogeant des points d'exposition (`endpoints HTTP`) appelés `exporters`. Ces derniers peuvent exposer des métriques d'applications, systèmes, conteneurs, etc.

Les données sont recueillies dans un format de type **clé-valeur** :

```prometheus
cpu_usage{host="server1"} 0.45
```

Selon sa configuration, `prometheus` va collecter les métriques exposées à interval régulier.

### Stockage

Les métriques sont stockées dans une **tsdb** (*Time series database*) qui permet de stocker temporelles au format **clé / valeur**.

`Prometheus` embarque sa propre base de données `TSDB`, mais il est possible d'utiliser d'autres bases de données comme `InfluxDB` ou `OpenTSDB`.

### Interroger les données

Pour cela on utilise le langage de requête de `prometheus` qui est `PromQL`. Il permet de créer des requêtes pour analyser les données collectées.

C'est grâc à ce langage qu'il est possible créer des graphiques, des alertes et des tableaux de bord dans `Grafana`.

### Alerting

`Prometheus` permet de définir des alertes basées sur les données collectées, grâce à son composant `Alertmanager`. Ces alertes peuvent être envoyées par mail par exemple.

## Installation et configuration

Il est possible de mettre en place `prometheus` et `grafana` via leurs images `docker`.

### Création des conteneurs

`docker compose` est le choix privilégié ici en raison de sa facilité d'utilisation et de sa lisibilité :

```yaml

```

### Configuration de Prometheus

La configuration de `prometheus` se fait habituellement via le fichier `prometheus.yml`.

```yaml
global:
  scrape_interval: 15s  # Fréquence de collecte des métriques

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node_exporter' # On peut ajouter des exporters pour scrapper + de métriques
    static_configs:
      - targets: ['[target_ip]:9100']  
```

**Quelques explications** :

- `global` : contient les paramètres globaux de `prometheus`.
- `scrape_interval` : fréquence de collecte des métriques.	
- `scrape_configs` : contient les configurations des `jobs` qui sont les cibles de `prometheus`.
- `job_name` : nom du job de collecte.
- `static_configs` : contient les cibles à scrapper.

Les différents éléments sont évidemment visibles sur l'interface graphique mise à disposition par `prometheus`.

## Configurer des exporters

Comme indiqué précédemment, `prometheus` scrape les `targets` définies dans le fichier de configuration. Ces `targets` sont souvent des `exporters` qui exposent des métriques : `node_exporter`, `docker_exporter`, etc.

### Le node exporter

Cet `exporter` expose une grande variété de métriques du système : `cpu`, `ram`...

Afin de le configurer, il faut le télécharger depuis le  [dépôt git](https://github.com/prometheus/node_exporter) en fonction la **version** et de l'architecture **CPU** sur laquelle on souhaite l'exécuter (armv7, amd64, etc.).

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
tar xvfz node_exporter-1.8.2.linux-amd64.tar.gz
cd node_exporter-1.8.2.linux-amd64
./node_exporter
```

**Remarque** : Automatiser l'installation des **exporters** est possible via Ansible ou tout autre outil de **gestion de configurations**.

Il faut ensuite configurer cet `exporter` en tant que service sur la machine.

Ainsi, sous `/etc/systemd/system/` on va créer un service `node_exporter.service` :

```bash
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/etc/prometheus-exporters/node_exporter-1.8.2.linux-armv7/node_exporter --web.listen-address=:9100

[Install]
WantedBy=multi-user.target
```

Il faut évidemment avoir créée un `user` dédié au service. Il est important de **ne pas exécuter les exporters en tant que root**.

## Références

- [Documentation](https://prometheus.io/docs/guides/node-exporter/)
- [Dépôt git](https://github.com/prometheus/node_exporter)
- [How To Install Prometheus on Ubuntu 16.04 - Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-install-prometheus-on-ubuntu-16-04)
- [Prometheus node exporter on Raspberry Pi – How to install](https://linuxhit.com/prometheus-node-exporter-on-raspberry-pi-how-to-install/)
- - [Prometheus - Stephane Robert](https://blog.stephane-robert.info/docs/observer/metriques/prometheus/)