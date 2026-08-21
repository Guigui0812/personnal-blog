---
date: '2026-08-20T00:00:00+01:00'
draft: false
title: "SELinux en production : l'apprivoiser pour ne plus le désactiver"
categories: ['linux', 'security', 'selinux', 'sysadmin']
cover:
  image: '/images/selinux-cover-purple-style.png'
  alt: 'SELinux'
  caption: 'SELinux en production'
  focalPoint: 'center center'
---

Mon poste d'ingénieur système m'a rapidement amené à travailler sur des environnements critiques où la sécurité est un enjeu important. Par défaut, **SELinux** était donc activé sur les serveurs, et j'ai très vite été confronté à des blocages de services que je ne comprenais pas. En effet, la documentation était parfois incomplète et n'indiquait pas la présence de **SELinux**. J'avais donc dû le découvrir par moi-même, et apprendre à l'apprivoiser sans le désactiver puisqu'il était un composant essentiel du **hardening** des serveurs.

Ce court article est donc un retour d'expérience sur la manière dont j'ai appris à travailler avec **SELinux** afin de ne jamais avoir recours à la commande `setenforce 0`.

---

## Pourquoi SELinux ?

**SELinux** (Security-Enhanced Linux) est un module de sécurité du noyau Linux qui implémente un contrôle d'accès obligatoire (MAC). Il permet de définir des politiques de sécurité pour les processus et les fichiers, limitant ainsi les actions qu'un utilisateur ou un service peut effectuer sur le système. 

**Quelques exemples d'actions que SELinux peut restreindre :**

- Empêcher un serveur d'écouter sur un port non autorisé
- Interdire à un service compromis d'accéder à des fichiers ou des répertoires
- Interdire à un processus de s'exécuter avec des privilèges élevés
- Interdire à un processus d'écrire sur le systeme de fichiers ou d'accéder à des ressources sensibles

Bien que contraignant pour les administrateurs système, **SELinux** permet de renforcer fortement la sécurité des systèmes puisqu'il bloque les actions non légitimes par défaut. Il faut donc autoriser explicitement les actions légitimes pour que les services fonctionnent correctement. C'est donc un composant qui permet de respecter la règle du moindre privilège et d'éviter la compromission d'un système d'exploitation.

Désactiver cet outil avec la commande `setenforce 0` revient alors à supprimer cette protection et à rendre le système vulnérable aux attaques. Sur des serveurs de production, il s'agit d'une faute grave qui peut avoir des conséquences désastreuses, surtout si **SELinux** est mentionné dans les exigences de sécurité de l'entreprise ou dans les audits de sécurité.

---

## Identifier les blocages liés à SELinux

Le problème des blocages liées à **SELinux**, c'est qu'ils ne sont pas toujours évidents à identifier. En effet, les messages d'erreur des services ne mentionnent pas explicitement avoir été bloqué par ce composant. On va plutôt avoir des erreurs de permissions, de connexion réseau ou de fichiers inaccessibles. La première étape est donc de réussir à identifier **SELinux** en tant que cause potentielle du problème.

Avant de changer quoi que ce soit, il faut vérifier le statut de **SELinux** sur le serveur :

```bash
sestatus
getenforce
```

Si un service ne fonctionne pas correctement sans raison apparente (permissions qui semblent correctes mais accès refusé), la première chose à consulter est le journal d'audit :

```bash
grep [service] /var/log/audit/audit.log
ausearch -m avc
```

On peut aussi utiliser la commande `sealert` qui va plus loin en analysant les logs et en proposant directement une explication et une piste de correction :

```bash
sealert -a /var/log/audit/audit.log
```

Ces étapes permettent d'identifier rapidement si **SELinux** est la cause du problème et de comprendre pourquoi le service est bloqué.

---

## Résoudre les blocages liés à SELinux

### Cas concret : rsyslog qui refuse d'écrire dans un répertoire monté

Mon premier contact avec **SELinux** s'est produit lors du déploiement d'un nouveau concentrateur de logs sur un projet. Malgré des permissions correctes, le service `rsyslog` n'était pas en capacité d'écrire dans le répertoire `/LOGS` monté sur le serveur. Au fur et à mesure de mes recherches, j'ai découvert que le problème venait de **SELinux**, qui empêchait `rsyslog` d'accéder et d'écrire dans ce répertoire.

Les droits Unix sur le dossier étaient corrects. Cependant, le contexte de sécurité du répertoire monté ne correspondait pas à celui attendu pour des logs qui devait être `var_log_t`.

Vérification du contexte de sécurité du répertoire :

```bash
ls -Z /LOGS
```

### Fix temporaire vs fix permanent

Corriger le contexte à la volée pour tester la solution avec la commande suivante :

```bash
sudo chcon -R -t var_log_t /LOGS
```

> `chcon` modifie le contexte d'un fichier ponctuellement.

Cette commande fonctionne instantanément, mais le changement **ne survit pas à un relabel**.

La vraie solution pérenne passe par la commande `semanage` :

```bash
sudo semanage fcontext -a -t var_log_t '/LOGS(/.*)?'
sudo restorecon -R /LOGS
```

> `semanage fcontext` modifie la règle qui définit quel contexte *devrait* avoir ce chemin. Dans l'exemple, le contexte `var_log_t` est appliqué à `/LOGS` et à tous ses sous-répertoires. Ensuite, `restorecon` applique le contexte défini par la règle **SELinux** sur le répertoire.

---

## Générer une policy proprement avec `audit2allow`

Le défaut de la méthode précédente, c'est qu'elle ne fonctionne que pour des cas simples où le blocage se résume à un contexte de fichier. Quand le blocage est plus complexe, il faut générer une policy **SELinux** pour autoriser l'action bloquée. Dans le cas de `rsyslog` précédemment évoqué, le service devait avoir le droit d'écouter sur un port non standard pour pouvoir recevoir les logs d'autres serveurs. Il fallait donc générer une policy **SELinux** *custom* pour autoriser cette action.

Pour cela, il faut utiliser la commande `audit2allow` qui génère une policy à partir des violations enregistrées dans les logs d'audit :

```bash
grep syslogd /var/log/audit/audit.log | audit2allow -M rsyslog
```

**Explications de la commande :**

- `grep syslogd /var/log/audit/audit.log` : on filtre les logs pour ne garder que ceux concernant le service `syslogd`.
- `audit2allow -M rsyslog` : on génère un module **SELinux** nommé `rsyslog` à partir des logs filtrés.

Ça produit un fichier `.te` (type enforcement) qu'il faut **relire avant d'appliquer**. En effet, `audit2allow` a tendance à englober toutes les violations récentes dans les logs, y compris celles qui n'ont rien à voir avec le problème. Appliquer une policy trop large annule une partie du bénéfice de **SELinux**.

**Exemple de fichier `.te` généré par `audit2allow` :**

```
module rsyslog 1.0;

require {
    type syslogd_t;
    type auditd_log_t;
    type unreserved_port_t;
    class dir { getattr open read search };
    class file { getattr open read ioctl };
    class tcp_socket name_connect;
}

allow syslogd_t auditd_log_t:dir open;
allow syslogd_t auditd_log_t:dir { getattr read search };
allow syslogd_t auditd_log_t:file getattr;
allow syslogd_t auditd_log_t:file open;
allow syslogd_t auditd_log_t:file read;
allow syslogd_t auditd_log_t:file ioctl;
allow syslogd_t unreserved_port_t:tcp_socket name_connect;
```

Une fois le contenu vérifié et validé, il suffit de l'appliquer avec la commande suivante :

```bash
semodule -i rsyslog.pp
```

Reste alors à redémarrer le service pour que la policy soit prise en compte et vérifier que le problème est résolu.

**Remarque** : il est possible que le service soit bloqué par plusieurs violations **SELinux**. Dans ce cas, il faudra répéter la procédure jusqu'à ce que le service fonctionne correctement. Il convient alors de regrouper toutes les violations dans un seul module **SELinux** pour éviter d'avoir à appliquer plusieurs modules.

---

## Les booléens : le raccourci pour les cas courants

Pour des besoins fréquents, **SELinux** propose des booléens qui permettent d'activer ou de désactiver certaines règles de sécurité. Par exemple, si un service a besoin d'accéder à des fichiers dans le répertoire personnel d'un utilisateur, il est possible d'activer le booléen `httpd_enable_homedirs` pour autoriser cette action.

Exemple :

```bash
sudo semanage boolean --list | head
sudo semanage boolean --modify --on httpd_enable_homedirs
```

Ces cas fréquents sont identifiés et suggérés par `audit2allow` ou `sealert`. Il est donc recommandé de vérifier si un booléen existe avant de générer une policy custom.

---

## Passer à l'échelle : automatiser plutôt que corriger machine par machine

Dans un contexte de production, il est fréquent d'avoir plusieurs serveurs qui doivent appliquer la même policy **SELinux**. Afin de documenter et de reproduire les changements de manière fiable, il est nécessaire d'automatiser l'application des policies **SELinux**. Cela permet de s'assurer que tous les serveurs sont configurés de manière cohérente et de réduire le risque d'erreurs humaines.

### Ansible

Avec **Ansible**, on peut copier le fichier `.te` généré par `audit2allow` sur le serveur et l'appliquer avec les commandes `checkmodule`, `semodule_package` et `semodule`. Voici un exemple de playbook Ansible pour appliquer une policy **SELinux** :

```yaml
- name: Apply SELinux policy
  hosts: selinux
  become: yes
  tasks:
    - name: Copy SELinux type enforcement file
      copy:
        src: myapp.te
        dest: /tmp/

    - name: Compile SELinux module file
      command: checkmodule -M -m -o /tmp/myapp.mod /tmp/myapp.te

    - name: Build SELinux policy package
      command: semodule_package -o /tmp/myapp.pp -m /tmp/myapp.mod

    - name: Load SELinux policy package
      command: semodule -i /tmp/myapp.pp
```

### Puppet

Avec Puppet, le [module SELinux officiel](https://forge.puppet.com/modules/puppet/selinux/dependencies) permet de déclarer directement les modules et booléens.

Exemple :

```puppet
selinux::module { 'selinux_module':
  ensure    => 'present',
  source_te => "puppet:///module.te",
  builder   => 'simple';
}
```

---

## Conclusion

**SELinux**, bien qu'intimidant au premier abord, propose pourtant l'ensemble des outils nécessaires afin d'identifier et résoudre les blocages liés à la sécurité. Il est donc possible de travailler avec sans jamais avoir à le désactiver, et ainsi bénéficier d'une protection supplémentaire pour les services critiques.

Cette anecdote permet d'illustrer l'importance des compétences de résolution de problème pour identifier la source d'un blocage et de la nécessité de comprendre les mécanismes de sécurité pour les contourner correctement. La désactivation de composant de sécurité comme **SELinux** est une solution de facilité qui peut avoir des conséquences graves pour la sécurité d'un système, surtout sur des serveurs de production. Comprendre, documenter et automatiser les changements de configuration est donc une étape cruciale pour garantir la sécurité d'un système d'information.

## Pour aller plus loin

- [What is SELinux? — Red Hat](https://www.redhat.com/en/topics/linux/what-is-selinux)
- [SELinux cheatsheet — WhiteWinterWolf](https://www.whitewinterwolf.com/posts/2017/09/08/selinux-cheatsheet/)
- [How to modify SELinux settings with booleans — Red Hat](https://www.redhat.com/en/blog/change-selinux-settings-boolean)
- [SELinux expliqué aux administrateurs frileux](https://blog.microlinux.fr/selinux/)