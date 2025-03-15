---
date: '2024-10-02T00:00:00+01:00'
draft: true
title: 'Pi-hole : bloquer les publicités avec Docker'
categories: ['docker', 'dns', 'pihole', 'networking']
---

**Pi-hole** est un service permettant de bloquer les publicités et de protéger la vie privée en bloquant les domaines de suivi. Il est possible de le déployer avec Docker et d'activer l'IPv6 pour bloquer les publicités sur ce protocole.

## Installation

Pour installer **Pi-hole**avec **Docker**, il suffit de créer un fichier `docker-compose.yml` avec le contenu suivant :

```yaml
version: '3'

services:
	pihole:
		container_name: pihole
		image: pihole/pihole:latest
		ports:
			- "53:53/tcp"
			- "53:53/udp"
			- "67:67/udp"
			- "80:80/tcp"
			- "443:443/tcp"
		environment:
			TZ: 'Europe/Paris'
			ServerIP: '192.168.1.2'
			IPv6: 'true'
			ServerIPv6: <server_ipv6>
		volumes:
			- './etc-pihole/:/etc/pihole/'
			- './etc-dnsmasq.d/:/etc/dnsmasq.d/'
		dns:
			- 8.8.8.8
			- 1.1.1.1
		cap_add:
		- NET_ADMIN
		restart: always
```

**Quelques explications :**

- L'option `IPv6` est importante si votre réseau local utilise ce protocole afin que **PiHole** soit en mesure de bloquer ce type de trafic.
- Les `volumes` permettent d'assurer la pérennité de la configuration même en cas de redémarrage ou de suppression du conteneur.

Une fois le fichier créé, il suffit de démarrer le conteneur pour démarrer le service `pihole` :

```bash
docker-compose up -d
```

## Configuration

Une fois le service démarré, il est possible de se rendre sur l'interface web de `pihole` à l'adresse `http://<ip_du_serveur>/admin`. 

Le `mot de passe` par défaut est généré aléatoirement et est affiché dans les logs du conteneur. 

Pour accéder à ces **logs** :

```bash
docker logs pihole | grep 'password'
```

## Sources et références

- [Pi-hole](https://pi-hole.net/)
- [Complete Guide to Setting up Pi-hole on a Raspberry Pi with IPv6 Support on Docker - Daniel Rampelt](https://danielrampelt.com/blog/install-pihole-raspberry-pi-docker-ipv6/)
- [The Big Blocklist Collection - Firebog](https://firebog.net/)
- [docker-pihole-openvpn-raspberry-ipv6 - GitHub](https://github.com/diogomartino/docker-pihole-openvpn-raspberry-ipv6/tree/master)
- [Pi-Hole : un bloqueur de pubs pour tout votre réseau - it-connect](https://www.it-connect.fr/pi-hole-un-bloqueur-de-pubs-pour-tout-votre-reseau/)
- [USING PI-HOLE LOCAL DNS AND WHY IT’S IMPORTANT IN YOUR HOMELAB - techaddressed](https://www.techaddressed.com/tutorials/using-pi-hole-local-dns/)