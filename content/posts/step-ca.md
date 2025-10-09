---
date: '2025-03-21T18:00:00+01:00'
draft: false
title: 'Step CA : devenir sa propre autorité de certification TLS'
categories: ['docker', 'https', 'sécurité']
---

Dans un environnement fermé comme un **homelab**, il est commun d'avoir recours à des **certificats autosignés**, ce qui n'est jamais idéal. **Step CA** permet de devenir sa propre autorité de certification, de manière simple, sécurisée et automatisable.

## Mise en place de la CA

### Installation

Installer la **CLI** selon l'architecture, par exemple **armv7** :

```bash
wget https://dl.smallstep.com/gh-release/cli/gh-release-header/v0.28.6/step_linux_0.28.6_armv7.tar.gz

tar -xf step_linux_0.28.6_armv7.tar.gz
sudo mv step*/step /usr/local/bin/
```

*Voir les différentes versions disponibles* : [GitHub - smallstep/cli](https://github.com/smallstep/cli)

Télécharger et exécuter l'image :

```bash
mkdir /opt/step-ca # OPT car contient des applications tierces non gérées par le système de paquets
cd /opt/step-ca
mkdir data
docker pull smallstep/step-ca
docker run -it -u <step_ca_user_id> -v ./data:/home/step smallstep/step-ca step ca init
```

**Remarque** : il est important de paramétrer les droits des fichiers dans le volume `data` pour l'utilisateur `step-ca` avec le bon **UID**.

Remplir les différents champs proposés :

```yaml
✔ What would you like to name your new PKI? (e.g. Smallstep): Smallstep
✔ What DNS names or IP addresses would you like to add to your new CA? (e.g. ca.smallstep.com[,1.1.1.1,etc.]): localhost
✔ What address will your new CA listen at? (e.g. :443): :9000
✔ What would you like to name the first provisioner for your new CA? (e.g. you@smallstep.com): admin@smallstep.com
✔ What do you want your password to be? [leave empty and we'll generate one]:
```

Explications :

- Nom de la CA
- Nom DNS pris en compte
- Port d'écoute de la CA (9000 par défaut)
- Nom de l'admin de la CA
- Mot de passe

Dans le volume `<path_to_step_ca_vol>/data` monté dans le conteneur `step-ca`, créez le fichier `<path_to_step_ca_vol>/data/secrets/password` et ajoutez le mot de passe de la **CA**. Cette étape est obligatoire pour le démarrage du conteneur.

Ensuite, il suffira d'exécuter le conteneur de la **CA** :

```bash
docker run -d -u <step-ca_user_id> -p 9000:9000 -v ./data:/home/step smallstep/step-ca
```

**Cas particulier sur Raspberry pi où une erreur s'affiche** :

```bash
badger 2025/03/19 18:53:50 INFO: All 0 tables opened in 0s
Error opening database of Type badgerv2: error opening Badger database: During db.vlog.open: Error while creating log file in valueLog.open: Mmap value log file. Path=/home/step/db/000000.vlog. Error=cannot allocate memory
```

La **Solution** est d'éditer le fichier `config/ca.json` dans le volume avec `vim` et de modifier les lignes qui concernent la **base de données** :

```json
{
...
    "db": {
          "type": "badgerv2",
          "dataSource": "/home/step/db",
          "badgerFileLoadingMode": "FileIO"
    },
...
}
```

Cela est également possible via `docker compose` :

```yaml
services:
  step-ca:
    user: "<step-ca_user_id>
    image: smallstep/step-ca
    ports:
      - 9000:9000
    restart: always
    volumes:
      - ./data:/home/step
```

### Récupérer la CA

Récupérez la **CA** sur l'hôte **docker** (présente aussi sous `$step-ca-path/data/certs/`) :

```bash
CA_FINGERPRINT=$(docker run -u <step-ca_user_id> -v ./data:/home/step smallstep/step-ca step certificate fingerprint /home/step/certs/root_ca.crt)

step ca bootstrap --ca-url https://localhost:9000 --fingerprint $CA_FINGERPRINT
```

### Ajouter le certificat **Root CA** au système

**Sous Linux (Debian, Ubuntu) :**

```bash
sudo cp ~/step-ca-root.crt /usr/local/share/ca-certificates/step-ca.crt
sudo update-ca-certificates
```

**Sous RHEL, Centos ou Fedora :**

```bash
sudo cp ~/step-ca-root.crt /etc/pki/ca-trust/source/anchors/step-ca.crt
sudo update-ca-trust
```

**Sous macOS :**

```bash
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain ~/step-ca-root.crt
```

**Sous Windows :**

- Ouvrir `certlm.msc` (via `Win + R`)
- Se rendre dans **"Autorités de certification racines de confiance"** et **"Certificats"**.
- Via clic droit → **"Toutes les tâches"** puis **"Importer"**.
- Sélectionner le fichier de la **Root CA** puis valider.

### Vérification

Après l'ajout du certificat, il reste à tester le bon fonctionnement de la **CA** :

```bash
curl -v https://localhost:9000
```

Réponse attendue :

```bash
{"status":"ok"}
```

## Obtenir un certificat depuis la CA

Afin d'obtenir un **certificat** depuis la **CA** on utilise :

```bash
step ca certificate svc.example.com svc.crt svc.key
```

Cette commande génère une clé privée (`.key`) et un certificat (`.crt`) signé par la CA.

Par défaut les certificats ne sont valides que **24 heures**. Pour augmenter la durée de validité, il faut ajouter le paramètre suivant dans `config/ca.json` :

```json
 "authority": {
	"provisioners": [
		...        
	],
	"claims": {
		"minTLSCertDuration": "5m",
		"maxTLSCertDuration": "730h",
		"defaultTLSCertDuration": "730h"
	}
}
```

La validité doit être définie en **heures** : `2190h` équivaut à **3 mois**.

## Renouveler un certificat

### Renouvellement manuel

Lorsqu'il est expiré, on peut renouveler un certificat avec la commande suivante :

```bash
step ca renew svc.crt svc.key
```

Il est possible de renouvellement un certificat en forçant la réécriture des fichiers `.crt` et `.key` :

```bash
step ca renew --force svc.crt svc.key
```

### Renouvellement automatique

#### Renouvellement via un **daemon**

Utilisation de la commande suivante :

```bash
step ca renew --daemon --renew-period 24h svc.crt svc.key
```

D'autres techniques seront abordées prochainement pour automatiser le renouvellement des certificats :

- Renouvellement via un **cron**
- **ACME** (Automated Certificate Management Environment)

#### Renouvellement via **ACME**

**Step CA** intègre un serveur **ACME**. Il est possible d'utiliser des clients comme **Certbot** ou **acme.sh** pour obtenir et renouveler des certificats.

Pour activer le serveur **ACME**, il faut exécuter la commande suivante dans le conteneur **step-ca** :

```bash
step ca provisioner add acme --type ACME
```

Une fois le conteneur redémarré, le serveur **ACME** sera disponible selon le pattern suivant : `http(s)://<ca-domain>:<ca-port>/acme/acme/directory`.


##### Exemple **proxmox**

- Ajouter la **Root CA** dans **Proxmox** en suivant la même méthode que pour **Debian/Ubuntu**.
- Se rendre dans **pve → Certificates** puis **Add → ACME Account**. Configurer le compte avec l'URL du serveur **ACME** et le compte **Step CA**.

##### Exemple avec la CLI **step ca**

Il est possible d'utiliser la CLI `step ca` pour obtenir un certificat via **ACME**. Le prérequis est d'avoir un compte **ACME** dans la **CA** et d'avoir exécuté la commande `step ca bootstrap` avec la **fingerprint** de la **CA**.

```bash
step ca certificate <cert_domain> <domain_cert_file> <domain_private_key> --provisioner acme
```

Une fois le certificat obtenu, il est possible de le renouveler avec la commande :

```bash
step ca renew <domain_cert_file> <domain_private_key> --provisioner acme
```

On peut également automatiser le renouvellement avec l'option `--daemon` comme vu précédemment.

L'idée est alors de créer un service `systemd` qui exécute la commande de renouvellement en tâche de fond :

```ini
[Unit]
Description=Step CA Renew Service for domain.crt
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=1
User=root
ExecStart=/usr/bin/step ca renew --daemon --exec "systemctl reload nginx" /path/to/domain.crt /path/to/domain.key --provisioner acme

[Install]
WantedBy=multi-user.target
```

## Références

- [Run a private online TLS certificate authority in a Docker container](https://smallstep.com/docs/tutorials/docker-tls-certificate-authority/#troubleshooting)
- [Install step](https://smallstep.com/docs/step-cli/installation/#debian-ubuntu)
- [hub.docker.com/r/smallstep/step-ca](https://hub.docker.com/r/smallstep/step-ca)
- [GitHub - dogukancagatay/step-ca-example: step-ca ACME server example with Traefik](https://github.com/dogukancagatay/step-ca-example)
- [Automated Local TLS Certificates With Step-CA | Tech Tutorials](https://www.techtutorials.tv/sections/it-security/automated-tls-certificates-step-ca/)