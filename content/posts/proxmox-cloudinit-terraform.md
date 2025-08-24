---
date: '2025-08-23T20:00:00+01:00'
draft: false
title: 'Proxmox : Automatiser le déploiement de VM avec Cloud-Init et Terraform'
categories: ['proxmox', 'terraform', 'cloud-init', 'virtualisation', 'infrastructure as code']
cover:
  image: '/images/proxmox-terraform-cloudinit.png'
  alt: 'Schéma expliquant l’automatisation du déploiement de VM sur Proxmox avec Terraform et Cloud-Init'
  caption: 'Proxmox avec Terraform & Cloud Init'
  focalPoint: 'center center'
---

Déployer des machines virtuelles sur un hyperviseur tel que `Proxmox` est une tâche fréquente pour tout administrateur système. Que ce soit dans un contexte de production ou dans un homelab, il s'agit d'une tâche récurrente. L'automatisation étant désormais au coeur des métiers de l'infrastructure, c'est exactement ce type de tâche qu'il convient de repenser sous le prisme de **l'infrastructure as code** via des outils modernes tels que `Terraform` et `Cloud-Init`.

## Contexte

Dans mon **homelab personnel**, je suis très souvent amené à mettre en place de nouvelles machines virtuelles sur mon hyperviseur Proxmox lorsque je souhaite faire des tests ou héberger un nouveau service. Cela peut vite devenir fastidieux puisque sans automatisation, je dois créer manuellement une nouvelle machine à chaque fois. 

Dès lors, la solution est simple : **automatiser la mise en place de nouvelles machines virtuelles sur l'hyperviseur** pour pouvoir se concentrer sur les tests et le déploiement des nouveaux services, sans passer par l'étape supplémentaire de configuration d'un hôte supplémentaire.

## Préparer Proxmox pour l'automatisation

Avant d'aller plus avant, il faut préparer l’environnement Proxmox. 

### Activation des snippets

Dans Proxmox, les **snippets sont des fichiers de configuration** (YAML, scripts) stockés dans un datastore. Ils permettent d’automatiser des actions au moment du provisionnement. C'est donc dans cet emplacement que seront stockés les fichiers `Cloud-Init`. Ces derniers **contiendront la configuration des machines virtuelles déployées** afin de disposer de serveurs "clés en main", prêts à être utilisés.

L'activation est à effectuer en passant par les menus `Datacenter`, `Storage` puis `local` afin de finalement cocher le paramètre `Snippets`.

### Télécharger une image Cloud-Init

Les distributions majeures maintiennent des images `Cloud-init` prêtes à l’emploi, compatibles avec l'outil du même nom.

Quelques exemples par distributions : 

- [Ubuntu](https://cloud-images.ubuntu.com/)
- [Debian](https://cloud.debian.org/images/cloud/) (choisir une image `generic`)
- [Fedora](https://fedoraproject.org/fr/cloud/download)

Choisissez une distribution, téléchargez l'image de votre choix et importez-la dans le répertoire `ISO images` de votre hyperviseur `Proxmox`.

## Initier le projet Terraform

**Prérequis** : si `Terraform` n'est pas installé sur votre machine, installez le en suivant la [documentation](https://developer.hashicorp.com/terraform/install).

Afin d'intéragir avec l'API de `Proxmox` via `Terraform`, il faut utiliser un `provider`. Il s'agit du plugin à l'outil d'intéragir avec des services ou plateformes cloud (`Proxmox`, `AWS`, `GCP`,...) en traduisant les fichiers de configuration `HCL` (*Hashicorp Configuration Language*) en appels API à destination de ces plateformes.

Comme un grand nombre de services modernes, `Proxmox` expose une `API` permettant d'effectuer la plupart des actions qui peuvent être déclenchées via l'interface graphique. Dès lors, différents `provider` ont été développés par la communauté pour interagir avec cette plateforme. 

Parmi ceux-ci, j'ai choisi [bpg/proxmox](https://registry.terraform.io/providers/bpg/proxmox/latest/docs) qui se dégage clairement du lot de par ses fonctionnalités, sa documentation et la fréquence de ses mises à jour. C'est donc celui-ci que j'ai décidé d'utiliser dans le cadre de l'automatisation de mon **Homelab**.


Ajouter le `provider` à votre fichier `provider.tf` :

```hcl
terraform {
  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = "<version>"
    }
  }
}
```

Ajouter le ensuite le bloc de configuration de `bpg/proxmox` à `provider.tf` :

```hcl
provider "proxmox" {
    endpoint = "https://10.0.0.2:8006/"

    username = "root@pam"
    password = "<proxmox_password>"

    insecure = true # if self-signed TLS certificate is in use

    ssh {
        agent = true
    }
}
```

Enfin, il ne reste plus qu'à initier votre projet avec `terraform init`, ce qui permettra d'initialiser le projet `Terraform` : création du fichier `tfstate`, téléchargement des providers, etc.

Pour éviter de stocker des secrets tels que le mot de passage `Proxmox` dans le code `Terraform`, il est possible de passer par une **variable sensible** :

```hcl
variable "proxmox_password" {
    description = "The password to connect to the Proxmox API"
    type        = string
    sensitive   = true
}
```

Il faut ensuite utiliser cette variable comme valeur du paramètre `password` dans le `provider` :

```hcl
provider "proxmox" {
    endpoint = "https://10.0.0.2:8006/"

    username = "root@pam"
    password = var.proxmox_password

    insecure = true # if self-signed TLS certificate is in use

    ssh {
        agent = true
    }
}
```

Avant d'effectuer les étapes `plan` et `apply`, il faudra créer la variable d'environnement `TF_VAR_proxmox_password` :

```bash
export TF_VAR_proxmox_password=<password>
echo $TF_VAR_proxmox_password # vérification
```

`Terraform` récupérant l'ensemble des variables d'environnement préfixées par `TF_VAR`, la valeur du paramètre sera celle définie dans la variable.

Le lien entre l'outil `IaC` et l'hyperviseur étant maintenant opérationnel, il est temps de passer à l'étape de création d'une machine virtuelle.

## Création d'une machines virtuelle proxmox avec Terraform

Le déploiement d'un serveur simple, sans configuration personnalisée, s'avère rapide avec `Terraform` et `bpg/proxmox`.

Dans un premier temps, il suffit de créer un fichier `main.tf` (ou autre nom de votre choix) et d'y ajouter le code permettant de créer une machinne virtuelle : 

```hcl
data "local_file" "ssh_public_key" {
  filename = "./id_rsa.pub"
}

resource "proxmox_virtual_environment_vm" "ubuntu_vm" {
  name      = "example-ubuntu"
  node_name = "pve" # Doit correspondre au nom de votre hyperviseur

  initialization {
    ip_config {
      ipv4 {
        address = "<ip>/<cidr>"
        gateway = "<ip>"
      }
    }

    user_account {
      username = "<username>"
      keys     = [trimspace(data.local_file.ssh_public_key.content)]
    }
  }

  disk {
    datastore_id = "local-lvm"
    file_id      = proxmox_virtual_environment_download_file.ubuntu_cloud_image.id
    interface    = "virtio0"
    iothread     = true
    discard      = "on"
    size         = 30
  }

  network_device {
    bridge = "vmbr0"
  }
}

# Téléchargement automatisé de l'ISO, sans action manuelle
resource "proxmox_virtual_environment_download_file" "ubuntu_cloud_image" {
  content_type = "iso"
  datastore_id = "local"
  node_name    = "pve" # Doit correspondre au nom de votre hyperviseur
  url          = "<url_image_iso>"
}
```

**Remarque** : le fichier `./id_rsa.pub` (clé publique) doit être présent dans votre projet `Terraform`.

Pour déployer le nouveau serveur, il suffit de suivre les étapes classiques d'un déploiement `Terraform`:

- Effectuer le **plan** afin de vérifier les changements apportés à l'infrastructure : `terraform plan -out <fichier>.plan`
- Appliquer les changements : `terraform apply <fichier>.plan`

Après quelques instants, la nouvelle machine virtuelle devrait être déployée et en fonctionnement sur l'hyperviseur.

**Remarque** : Si vous souhaitez détruire le serveur, évitez les actions manuelles car cela pourrait créer une inconsistance dans votre fichier `tfstate`. Effectuez plutôt la destruction via `terraform destroy` une fois le code de votre VM commenté ou en utilisant l'option `-target`.

## Créer une VM avec configuration personnalisée

### Exemple de déploiement avec Terraform et Cloud Init

Bien qu'automatiser le déploiement d'une VM sur `Proxmox` permette un léger gain de temps, ce n'est pas suffisant pour parler d'automatisation. Les étapes postérieures au déploiement comme la configuration d'utilisateurs ou l'installation de paquets restent à être effectuées. 

Heureusement, `bpg/proxmox` permet d'exploiter la capacité de `Proxmox` d'utiliser des templates de configuration `Cloud Init` pour appliquer des configurations personnalisées aux VMs utilisant des **images cloud** (impossible avec des images standards).

`Cloud Init` est un **outil permettant de définir la configuration de machines virtuelles lors de leur premier démarrage**. Via un fichier de configuration au format `YAML`, l'administrateur va pouvoir créer des utilisateurs, installer des paquets, personnaliser des éléments du serveur et même exécuter des scripts. C'est donc un outil très **utile pour déployer des serveurs prêts à l'emploi de manière standardisée**. 

La création d'une VM utilisant `Cloud Init` avec `Terraform` peut être effectuée à l'aide de l'exemple suivant :

```hcl
resource "proxmox_virtual_environment_file" "example-ubuntu-cloud-config" {
    content_type = "snippets"
    datastore_id = "local"
    node_name    = "pve"

    source_raw {
    data = <<-EOF
    #cloud-config
    hostname: example-ubuntu
    timezone: Europe/Paris
    users:
        - default
        - name: <username>
          sudo: ALL=(ALL) NOPASSWD:ALL
          shell: /bin/bash
          groups:
            - sudo

    expire: False
    ssh_pwauth: true
    runcmd:
        - echo "MOTD_CONTENT" > /etc/motd
    package_update: true
    packages:
      - qemu-guest-agent
      - net-tools
      - curl
    runcmd:
      - systemctl enable qemu-guest-agent
      - systemctl start qemu-guest-agent
    EOF

    file_name = "example-ubuntu-cloud-config.yaml"
  }
}

resource "proxmox_virtual_environment_vm" "example-ubuntu" {
  name      = "example-ubuntu" # Doit être le même à celui défini dans la configuration cloud-init
  node_name = "pve"

  cpu {
    cores = "2"
  }

  memory {
    dedicated = "2048"
  }

  initialization {
    user_account {
      username = "<username>"
      password = var.proxmox_password
    }

    ip_config {
      ipv4 {
        address = "<ip>/<cidr>"
        gateway = "<ip>"
      }
    }

    user_data_file_id = proxmox_virtual_environment_file.example-ubuntu-cloud-config.id
  }

  disk {
    datastore_id = "local-lvm"
    file_id      = "local:iso/<iso_image_name>.img" # Convertir l'image au format img si nécessaire
    interface    = "virtio0"
    iothread     = "true"
    discard      = "on"
    size         = "30"
  }

  network_device {
    bridge = "vmbr0"
  }
}
```

Le déploiement de cette VM s'effectue de la même manière que précédemment en suivant les étapes de **plan** et **apply**. Après quelques instants, une nouvelle machine virtuelle sera déployée avec l'ensemble des configurations correspondant à celles définies dans le bloc dédié à `Cloud Init`. De nombreuses autres directives sont d'ailleurs disponibles pour pousser la personnalisation.

### Zoom sur Cloud Init

`Cloud Init` propose un grand nombre de directives pour personnaliser différents aspects des déploiements.

#### Paquets

Parfait 👍 Tu peux faire une section **“Zoom sur Cloud-Init”** bien découpée par thématique. Voici une version prête à intégrer dans ton article, avec des exemples réalistes et commentés :

---

### Zoom sur Cloud-Init

`Cloud-Init` propose un grand nombre de directives pour personnaliser différents aspects des déploiements. L'ensemble des directive est détaillée dans la [documentation de Cloud Init](https://cloudinit.readthedocs.io/en/latest/reference/index.html).

#### Paquets

Installer automatiquement des paquets au premier boot :

```yaml
#cloud-config
packages:
  - vim
  - htop
  - curl
  - git
```

L'installation des `guest-agents` est d'ailleurs fortement recommandée sur vos VMs :

```yaml
#cloud-config
packages:
   - qemu-guest-agent
runcmd:
   - systemctl enable qemu-guest-agent
   - systemctl start qemu-guest-agent
```

#### Réseau

Configurer une interface réseau avec une IP statique :

```yaml
#cloud-config
network:
  version: 2
  ethernets:
    ens18:
      dhcp4: no
      addresses:
        - 192.168.1.50/24
      gateway4: 192.168.1.1
      nameservers:
        addresses: [192.168.1.1, 8.8.8.8]
```

Configurer les `dns`via `resolv.conf` :

```yaml
#cloud-config
manage_resolv_conf: true
resolv_conf:
  domain: example.com
  nameservers: [8.8.8.8, 8.8.4.4]
  options: {rotate: true, timeout: 1}
  searchdomains: [foo.example.com, bar.example.com]
  sortlist: [10.0.0.1/255, 10.0.0.2]
```

#### Utilisateurs

Définir des utilisateurs supplémentaires avec sudo et clés SSH :

```yaml
#cloud-config
users:
  - name: username
    groups: sudo
    shell: /bin/bash
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh_authorized_keys:
      - <ssh_pub_key>
    plain_text_passwd: <password>
```

#### Commandes à l’initialisation

Lancer des commandes après le boot :

```yaml
#cloud-config
runcmd:
  - echo "Server Initialised" > init.log
```

---

#### Gestion des mots de passe

Définir un mot de passe pour un utilisateur et activer l’authentification SSH par mot de passe (optionnel) :

```yaml
#cloud-config
chpasswd:
  list: |
    username:motdepasse
  expire: False
ssh_pwauth: true
```

#### Mise à jour et upgrade système

Forcer la mise à jour du système lors du premier boot :

```yaml
#cloud-config
package_update: true
package_upgrade: true
```

## Standardisation des déploiement via un module dédié

L'utilisation de `Terraform` pour déployer des machines virtuelles dans `Proxmox` m'a apporté fluidité et facilité d'utilisation lors de la mise en place de nouveaux services dans mon Homelab. Cependant, si dans un premier temps j'effectuais une copie de mes configurations pour créer un nouveau serveur, j'ai rapidement préféré créer un module `Terraform` afin de standardiser mes déploiements et de disposer d'un bloc de code réutilisable afin d'éviter ces étapes fastidieuses. 

Ce module est d'ailleurs disponible publiquement sur [github](https://github.com/Guigui0812/Terraform-Proxmox-Module/tree/main) afin d'aider des personnes qui souhaiteraient déployer facilement des VMs dans leur mon homelab personnel.

## Conclusion

Cette approche permet de déployer rapidement des VMs prêtes à l’emploi. L’étape suivante, dans mon cas, a été d’intégrer Ansible pour configurer automatiquement les services au-dessus des VMs. Cela permet de pousser l’automatisation encore plus loin, dans une logique complète d’Infrastructure as Code.

## Références

- [Documentation de bpg/proxmox](https://registry.terraform.io/providers/bpg/proxmox/latest/docs)
- [Proxmox : déployer une machine virtuelle KVM avec cloud-init – memo-linux.com](https://memo-linux.com/proxmox-deployer-une-machine-virtuelle-kvm-avec-cloud-init/)
- [Créer des templates de machines virtuelles avec Cloud-Init](https://www.bejean.eu/2023/03/24/creer-des-templates-de-vm-avec-cloud-init)
- [Provisionner des VM avec Terraform - Stéphane Roberts](https://blog.stephane-robert.info/docs/virtualiser/type1/proxmox/terraform/)
- [Terraform Apply - When to Run & Quick Usage Examples](https://spacelift.io/blog/terraform-apply)
- [Instancier une machine virtuelle Debian dans Proxmox avec OpenTofu et Cloud-Init](https://xieme-art.org/post/instancier-une-machine-virtuelle-debian-dans-proxmox-avec-opentofu-et-cloud-init/)
- [cloud-init 25.1 documentation](https://cloudinit.readthedocs.io/en/latest/index.html)

