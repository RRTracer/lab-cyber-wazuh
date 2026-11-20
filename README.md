# Lab Cyber Wazuh --- Infrastructure as Code & supervision de sécurité

![Platform](https://img.shields.io/badge/platform-Linux-informational)
![Virtualization](https://img.shields.io/badge/virtualization-VirtualBox-blue)
![Provisioning](https://img.shields.io/badge/provisioning-Vagrant-orange)
![Automation](https://img.shields.io/badge/automation-Ansible-red)
![SIEM](https://img.shields.io/badge/SIEM-Wazuh-4.x-purple)
![Status](https://img.shields.io/badge/status-lab%20en%20development-yellow)

Laboratoire local de cybersécurité construit autour de **Wazuh**,
**Ansible**, **Vagrant** et **VirtualBox**.\
Le projet a pour objectif de déployer automatiquement une plateforme
centralisée de supervision de sécurité ainsi qu'un agent Linux, dans un
environnement virtualisé reproductible.

> **Type de projet :** laboratoire personnel / apprentissage /
> démonstration technique.\
> **Attention :** ce dépôt décrit un environnement de test. Il ne
> constitue pas une architecture de production prête à l'emploi.

------------------------------------------------------------------------

## Sommaire

-   [Objectifs](#-objectifs)
-   [Architecture](#-architecture)
-   [Technologies](#-technologies)
-   [Arborescence](#-arborescence-du-dépôt)
-   [Prérequis](#-prérequis)
-   [Déploiement rapide](#-déploiement-rapide)
-   [Utilisation](#-utilisation)
-   [Ansible et automatisation](#-ansible-et-automatisation)
-   [Accès au Wazuh Dashboard](#-accès-au-wazuh-dashboard)
-   [Sécurité et secrets](#-sécurité-et-secrets)
-   [Dépannage](#-dépannage)
-   [Documentation](#-documentation)
-   [Pistes d'évolution](#-pistes-dévolution)
-   [Avertissement](#-avertissement)

------------------------------------------------------------------------

## 🎯 Objectifs

Ce laboratoire vise à mettre en pratique plusieurs compétences
d'administration système, réseau et cybersécurité :

-   Déployer une infrastructure virtualisée avec **Vagrant et
    VirtualBox**.
-   Automatiser la configuration des systèmes Linux avec **Ansible**.
-   Déployer une architecture **Wazuh All-in-One**.
-   Installer et enrôler un agent Linux.
-   Centraliser les événements de sécurité.
-   Comprendre les flux réseau entre agent et serveur.
-   Utiliser Ansible Vault pour protéger les variables sensibles.
-   Construire des rôles Ansible réutilisables et idempotents.
-   Disposer d'un environnement reproductible pour tester les
    fonctionnalités de supervision et de détection.

------------------------------------------------------------------------

## 🏗️ Architecture

Le laboratoire repose sur deux machines virtuelles Ubuntu Server 22.04
LTS.

``` text
                       POSTE HÔTE LINUX
              +--------------------------------+
              |                                |
              |  Vagrant                       |
              |  VirtualBox                    |
              |  Ansible                       |
              |                                |
              +---------------+----------------+
                              |
                       Réseau Host-only
                       192.168.56.0/24
                              |
              +---------------+----------------+
              |                                |
      +-------+--------+               +-------+--------+
      |    wazuh-01    |               |    client-01   |
      | 192.168.56.10  |               | 192.168.56.11  |
      +----------------+               +----------------+
      | Wazuh Manager  |<--- TCP 1514 -| Wazuh Agent    |
      | Wazuh Indexer  |<--- TCP 1515 -|                |
      | Wazuh Dashboard|               |                |
      +----------------+               +----------------+
              |
         HTTPS / TCP 443
              |
        Navigateur hôte
```

### Machines virtuelles

  -----------------------------------------------------------------------
  Machine           Adresse IP        Fonction          Ressources
                                                        prévues
  ----------------- ----------------- ----------------- -----------------
  `wazuh-01`        `192.168.56.10`   Manager, Indexer, 4 vCPU / 8 Go RAM
                                      Dashboard         / 60 Go disque

  `client-01`       `192.168.56.11`   Agent Wazuh       2 vCPU / 2 Go RAM
  -----------------------------------------------------------------------

Les ressources sont des valeurs de configuration prévues pour le
laboratoire. Elles peuvent être adaptées à la capacité du poste hôte.

### Flux réseau principaux

  Source        Destination         Port Fonction
  ------------- ------------- ---------- -------------------------------
  Hôte Linux    `wazuh-01`        TCP 22 Administration SSH / Ansible
  Hôte Linux    `wazuh-01`       TCP 443 Interface Dashboard HTTPS
  Hôte Linux    `client-01`       TCP 22 Administration SSH / Ansible
  `client-01`   `wazuh-01`      TCP 1514 Communication Agent / Manager
  `client-01`   `wazuh-01`      TCP 1515 Enrôlement de l'agent

Le réseau Host-only permet aux machines virtuelles de communiquer avec
le poste hôte sans exposer directement le laboratoire sur le réseau
local physique. Le réseau NAT de Vagrant peut également être utilisé
pour l'accès Internet des VM.

------------------------------------------------------------------------

## 🧰 Technologies

  ------------------------------------------------------------------------------------------------------------
  Technologie                                                              Utilisation
  ------------------------------------------------------------------------ -----------------------------------
  [Wazuh](https://wazuh.com/)                                              SIEM / XDR open source, collecte et
                                                                           analyse des événements

  [Ansible](https://www.ansible.com/)                                      Automatisation et gestion de
                                                                           configuration

  [Vagrant](https://developer.hashicorp.com/vagrant)                       Définition et cycle de vie des VM

  [VirtualBox](https://www.virtualbox.org/)                                Hyperviseur local

  [Ubuntu Server 22.04](https://ubuntu.com/server)                         Système d'exploitation des VM

  [Ansible                                                                 Chiffrement des variables sensibles
  Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html)   
  ------------------------------------------------------------------------------------------------------------

------------------------------------------------------------------------

## 📁 Arborescence du dépôt

``` text
lab-cyber-wazuh/
├── DOC/
│   ├── Guide-Utilisation.md
│   ├── Dossier-Technique.md
│   └── Doc.md
│
├── Infra/
│   ├── Vagrantfile
│   │
│   └── ansible/
│       ├── ansible.cfg
│       │
│       ├── inventory/
│       │   ├── hosts.yml
│       │   └── group_vars/
│       │       └── all/
│       │           ├── vars.yml
│       │           └── vault.yml
│       │
│       ├── playbooks/
│       │   ├── common.yml
│       │   ├── wazuh-server.yml
│       │   ├── wazuh-agent.yml
│       │   └── site.yml
│       │
│       └── roles/
│           ├── common/
│           ├── wazuh-server/
│           └── wazuh-agent/
│
├── .gitignore
├── LICENSE
└── README.md
```

> L'arborescence est la structure cible du projet. Certains fichiers ou
> rôles peuvent être encore en cours de développement selon l'avancement
> réel du laboratoire.

### Organisation Ansible

-   `inventory/` : déclaration des machines et variables.
-   `group_vars/` : variables communes aux groupes.
-   `playbooks/` : orchestration des déploiements.
-   `roles/common/` : préparation des systèmes.
-   `roles/wazuh-server/` : installation du serveur Wazuh.
-   `roles/wazuh-agent/` : installation et configuration de l'agent.

------------------------------------------------------------------------

## 📋 Prérequis

Sur le poste hôte Linux :

-   VirtualBox installé.
-   Vagrant installé.
-   Ansible installé.
-   Connexion Internet pour télécharger les images et paquets.
-   Virtualisation matérielle activée dans le BIOS/UEFI.
-   Ressources suffisantes pour exécuter les deux VM simultanément.

Vérifier les installations :

``` bash
vagrant --version
VBoxManage --version
ansible --version
```

Vérifier que VirtualBox expose son interface Host-only :

``` bash
VBoxManage list hostonlyifs
```

------------------------------------------------------------------------

## 🚀 Déploiement rapide

### 1. Cloner le dépôt

``` bash
git clone <URL_DU_DEPOT>
cd lab-cyber-wazuh
```

### 2. Démarrer les machines virtuelles

``` bash
cd Infra
vagrant up
```

Vagrant télécharge l'image Ubuntu si nécessaire, crée les VM et
configure leurs paramètres.

Vérifier leur état :

``` bash
vagrant status
```

### 3. Vérifier la connectivité réseau

Depuis le poste hôte :

``` bash
ping -c 4 192.168.56.10
ping -c 4 192.168.56.11
```

### 4. Accéder au répertoire Ansible

``` bash
cd ansible
```

### 5. Vérifier l'inventaire

``` bash
ansible-inventory --graph
```

### 6. Tester les connexions SSH

``` bash
ansible all -m ping --ask-vault-pass
```

Les deux machines doivent répondre avec `pong`.

### 7. Déployer le socle commun

``` bash
ansible-playbook playbooks/common.yml --ask-vault-pass
```

### 8. Déployer le serveur Wazuh

Vérifier la syntaxe :

``` bash
ansible-playbook playbooks/wazuh-server.yml --syntax-check --ask-vault-pass
```

Lancer le déploiement :

``` bash
ansible-playbook playbooks/wazuh-server.yml --ask-vault-pass
```

### 9. Déployer l'agent

Vérifier la syntaxe :

``` bash
ansible-playbook playbooks/wazuh-agent.yml --syntax-check --ask-vault-pass
```

Lancer le déploiement :

``` bash
ansible-playbook playbooks/wazuh-agent.yml --ask-vault-pass
```

> Les commandes supposent que les playbooks, les variables Vault et les
> rôles correspondants sont présents et configurés. Adapter les
> commandes si le projet évolue.

------------------------------------------------------------------------

## 🖥️ Utilisation

### Démarrer le laboratoire

Depuis `Infra/` :

``` bash
vagrant up
```

### Arrêter les machines

``` bash
vagrant halt
```

### Redémarrer les machines

``` bash
vagrant reload
```

### Se connecter aux VM

``` bash
vagrant ssh wazuh
vagrant ssh client
```

### Afficher l'état des machines

``` bash
vagrant status
```

### Détruire les machines

``` bash
vagrant destroy
```

⚠️ La destruction des VM supprime les données conservées uniquement sur
leurs disques virtuels. Sauvegarder les données utiles avant toute
opération destructive.

------------------------------------------------------------------------

## ⚙️ Ansible et automatisation

Les commandes Ansible sont à exécuter depuis `Infra/ansible/`.

### Afficher l'inventaire

``` bash
ansible-inventory --graph
ansible-inventory --list
```

### Tester les hôtes

``` bash
ansible all -m ping --ask-vault-pass
```

### Vérifier les privilèges

``` bash
ansible all -b -m command -a "id" --ask-vault-pass
```

### Vérifier la syntaxe d'un playbook

``` bash
ansible-playbook playbooks/site.yml --syntax-check --ask-vault-pass
```

### Lancer le déploiement global

``` bash
ansible-playbook playbooks/site.yml --ask-vault-pass
```

### Idempotence

Les rôles sont conçus pour décrire un état cible. Une deuxième exécution
ne devrait pas réinstaller inutilement les composants ni modifier des
fichiers inchangés.

L'idempotence doit être vérifiée en pratique en relançant les playbooks
et en examinant le nombre de tâches `changed`.

------------------------------------------------------------------------

## 🌐 Accès au Wazuh Dashboard

Une fois le serveur déployé et les services actifs, ouvrir :

**https://192.168.56.10**

Le certificat TLS généré pour un laboratoire peut être auto-signé. Un
avertissement du navigateur est donc possible.

Les identifiants initiaux sont générés lors de l'installation. Ils
doivent être récupérés dans l'artefact local prévu à cet effet et
conservés de manière sécurisée. Ne jamais les publier dans le dépôt Git.

Dans l'interface Wazuh, vérifier que `client-01` apparaît dans la
gestion des agents avec le statut **Active**.

------------------------------------------------------------------------

## 🔐 Sécurité et secrets

Le projet utilise Ansible Vault pour chiffrer les variables sensibles.

Créer ou modifier le fichier Vault :

``` bash
ansible-vault create inventory/group_vars/all/vault.yml
ansible-vault edit inventory/group_vars/all/vault.yml
```

Exécuter un playbook avec saisie du mot de passe Vault :

``` bash
ansible-playbook playbooks/site.yml --ask-vault-pass
```

### Fichiers à ne jamais publier

-   `.vagrant/`
-   clés privées SSH générées par Vagrant ;
-   fichiers de mot de passe Vault ;
-   mots de passe Wazuh ;
-   archives contenant les certificats ou secrets d'installation ;
-   fichiers d'environnement contenant des identifiants.

Exemple de `.gitignore` :

``` gitignore
# Vagrant
.vagrant/

# Ansible
*.retry
.vault_pass

# Secrets et artefacts Wazuh
artifacts/
wazuh-install-files.tar
wazuh-passwords.txt

# Fichiers locaux
.DS_Store
*.log
```

Le fichier `vault.yml` peut être versionné uniquement s'il est
effectivement chiffré avec Ansible Vault et si le mot de passe de
déchiffrement est conservé séparément.

------------------------------------------------------------------------

## 🧯 Dépannage

### Ansible ne trouve pas la clé SSH

Vérifier la configuration générée par Vagrant :

``` bash
cd Infra
vagrant ssh-config wazuh
vagrant ssh-config client
```

Vérifier que les fichiers de clé existent et que les chemins de
l'inventaire sont corrects.

### `REMOTE HOST IDENTIFICATION HAS CHANGED`

Après avoir confirmé que l'adresse IP correspond bien à une VM du
laboratoire, supprimer uniquement l'ancienne entrée concernée :

``` bash
ssh-keygen -f ~/.ssh/known_hosts -R 192.168.56.10
ssh-keygen -f ~/.ssh/known_hosts -R 192.168.56.11
```

Puis reconnecter la VM et vérifier sa nouvelle empreinte SSH.

### `Attempting to decrypt but no vault secrets found`

Fournir le mot de passe Vault :

``` bash
--ask-vault-pass
```

### L'agent n'apparaît pas comme actif

Vérifier sur `client-01` :

``` bash
sudo systemctl status wazuh-agent
sudo tail -n 100 /var/ossec/logs/ossec.log
```

Vérifier sur `wazuh-01` :

``` bash
sudo systemctl status wazuh-manager
sudo tail -n 100 /var/ossec/logs/ossec.log
```

Vérifier les ports depuis le client :

``` bash
nc -vz 192.168.56.10 1514
nc -vz 192.168.56.10 1515
```

La connectivité réseau seule ne garantit pas que l'enrôlement a réussi.
Confirmer le statut de l'agent dans le Dashboard.

------------------------------------------------------------------------

## 📚 Documentation

Deux documents complémentaires sont disponibles dans `DOC/` :

  ------------------------------------------------------------------------------
  Document                                   Contenu
  ------------------------------------------ -----------------------------------
  [Guide                                     Exploitation quotidienne,
  d'utilisation](DOC/Guide-Utilisation.md)   commandes, vérifications, dépannage
                                             et exercices

  [Dossier                                   Architecture, réseau, VirtualBox,
  technique](DOC/Dossier-Technique.md)       Vagrant, Ansible, Wazuh et choix
                                             techniques
  ------------------------------------------------------------------------------

Documentation officielle :

-   [Wazuh Documentation](https://documentation.wazuh.com/)
-   [Wazuh Installation
    Guide](https://documentation.wazuh.com/current/installation-guide/)
-   [Ansible Documentation](https://docs.ansible.com/)
-   [Ansible
    Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html)
-   [Vagrant
    Documentation](https://developer.hashicorp.com/vagrant/docs)
-   [VirtualBox Manual](https://www.virtualbox.org/manual/)

------------------------------------------------------------------------

## 🔭 Pistes d'évolution

Les évolutions envisagées pour enrichir le laboratoire :

-   [ ] Finaliser et valider l'idempotence de tous les rôles Ansible.
-   [ ] Ajouter `ansible-lint` et des contrôles de syntaxe.
-   [ ] Gérer la configuration de l'agent via des templates Ansible.
-   [ ] Ajouter la supervision File Integrity Monitoring (FIM).
-   [ ] Étudier Security Configuration Assessment (SCA).
-   [ ] Centraliser des journaux système et applicatifs supplémentaires.
-   [ ] Construire des règles de détection personnalisées.
-   [ ] Ajouter d'autres agents Linux.
-   [ ] Ajouter un poste Windows supervisé.
-   [ ] Mettre en place des sauvegardes et une procédure de
    restauration.
-   [ ] Étudier une architecture Wazuh distribuée.

------------------------------------------------------------------------

## ⚠️ Avertissement

Ce projet est destiné à l'apprentissage, aux expérimentations et à la
démonstration technique dans un environnement contrôlé.

Il ne doit pas être exposé directement sur Internet sans mesures de
sécurité adaptées. Les identifiants, certificats, clés privées et
archives de secrets ne doivent jamais être publiés.

Les adresses IP, versions, ressources et commandes présentées
correspondent à la configuration de laboratoire documentée et peuvent
nécessiter une adaptation selon l'environnement réel.

------------------------------------------------------------------------

## 📄 Licence

Consulter le fichier [`LICENSE`](LICENSE) pour connaître les conditions
de réutilisation du projet.

------------------------------------------------------------------------

**Projet personnel --- Lab Cyber Wazuh**\
*Infrastructure as Code · Automatisation · Supervision · Détection*
