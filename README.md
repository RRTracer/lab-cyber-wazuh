# Lab Cyber Wazuh & Caldera --- Infrastructure as Code, supervision et émulation

![Platform](https://img.shields.io/badge/platform-Linux-informational)
![Virtualization](https://img.shields.io/badge/virtualization-VirtualBox-blue)
![Provisioning](https://img.shields.io/badge/provisioning-Vagrant-orange)
![Automation](https://img.shields.io/badge/automation-Ansible-red)
![SIEM](https://img.shields.io/badge/SIEM-Wazuh-4.x-purple)
![Adversary Emulation](https://img.shields.io/badge/Adversary%20Emulation-Caldera-blueviolet)
![Status](https://img.shields.io/badge/status-lab%20en%20development-yellow)

Laboratoire local de cybersécurité construit autour de **Wazuh**, **MITRE Caldera**,
**Ansible**, **Vagrant** et **VirtualBox**.\
Le projet déploie une plateforme centralisée de supervision et de détection
(Wazuh), un client Linux supervisé et un serveur Caldera dédié aux
expérimentations contrôlées de Purple Team, dans un environnement
virtualisé reproductible.

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
-   [Accès aux interfaces web](#-accès-aux-interfaces-web)
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
-   Installer et enrôler un agent Wazuh Linux.
-   Déployer automatiquement **MITRE Caldera** sur une VM dédiée avec un rôle Ansible.
-   Centraliser les événements de sécurité.
-   Comprendre les flux réseau entre agent et serveur.
-   Utiliser Ansible Vault pour protéger les variables sensibles.
-   Construire des rôles Ansible réutilisables et idempotents.
-   Disposer d'un environnement reproductible pour tester les
    fonctionnalités de supervision, de détection et les scénarios
    pédagogiques contrôlés de Purple Team.

------------------------------------------------------------------------

## 🏗️ Architecture

Le laboratoire repose sur trois machines virtuelles Ubuntu Server 22.04 LTS,
connectées au réseau privé Host-only de VirtualBox.

```text
                         POSTE HÔTE LINUX
                +--------------------------------+
                | Vagrant / VirtualBox / Ansible |
                +---------------+----------------+
                                |
                         Réseau Host-only
                          192.168.56.0/24
                                |
          +---------------------+----------------------+
          |                     |                      |
   +------+-------+      +------+-------+       +------+---------+
   |   wazuh-01   |      |  client-01   |       | caldera-server |
   | 192.168.56.10|      |192.168.56.11|       | 192.168.56.12 |
   +--------------+      +--------------+       +----------------+
   | Wazuh Manager|<---->| Wazuh Agent  |       | MITRE Caldera  |
   | Indexer      |      | Client Linux |       | Server / UI    |
   | Dashboard    |      +--------------+       +----------------+
   +--------------+                                      |
          |                                      TCP 8888 / HTTP
     HTTPS / TCP 443                                       |
          +---------------- Navigateur hôte ---------------+
```

### Machines virtuelles

| Machine | Adresse IP | Fonction | Ressources prévues |
|---|---|---|---|
| `wazuh-01` | `192.168.56.10` | Wazuh Manager, Indexer et Dashboard | 4 vCPU / 8 Go RAM / 60 Go disque |
| `client-01` | `192.168.56.11` | Client Linux avec agent Wazuh | 2 vCPU / 2 Go RAM |
| `caldera-01` (Vagrant : `caldera-server`) | `192.168.56.12` | Serveur MITRE Caldera et interface web | 2 vCPU / 4 Go RAM |

Les ressources sont les valeurs configurées pour le laboratoire. Elles
peuvent être adaptées aux capacités du poste hôte.

### Flux réseau principaux

| Source | Destination | Port | Fonction |
|---|---|---:|---|
| Hôte Linux | `wazuh-01` | TCP 22 | Administration SSH / Ansible |
| Hôte Linux | `wazuh-01` | TCP 443 | Wazuh Dashboard HTTPS |
| Hôte Linux | `client-01` | TCP 22 | Administration SSH / Ansible |
| Hôte Linux | `caldera-server` | TCP 22 | Administration SSH / Ansible |
| Hôte Linux | `caldera-server` | TCP 8888 | Interface web Caldera (HTTP dans la configuration actuelle) |
| `client-01` | `wazuh-01` | TCP 1514 | Communication Agent / Manager |
| `client-01` | `wazuh-01` | TCP 1515 | Enrôlement de l'agent Wazuh |

Le réseau Host-only permet aux VM de communiquer avec le poste hôte sans
exposer directement le laboratoire au réseau physique. Le réseau NAT de
Vagrant peut également être utilisé pour l'accès Internet des VM.

------------------------------------------------------------------------

## 🧰 Technologies

  ------------------------------------------------------------------------------------------------------------
  Technologie                                                              Utilisation
  ------------------------------------------------------------------------ -----------------------------------
  [Wazuh](https://wazuh.com/)                                              SIEM / XDR open source, collecte et
                                                                           analyse des événements

  [MITRE Caldera](https://caldera.mitre.org/)                               Plateforme d'émulation d'adversaire et
                                                                           orchestration de laboratoire contrôlé

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
│       │   ├── caldera-server.yml
│       │   └── site.yml
│       │
│       └── roles/
│           ├── common/
│           ├── wazuh-server/
│           ├── wazuh-agent/
│           └── caldera-server/
│
├── .gitignore
├── LICENSE
└── README.md
```

L'arborescence ci-dessus présente les composants principaux du dépôt.
Les fichiers internes de chaque rôle sont organisés selon les
conventions Ansible (`defaults/`, `tasks/`, `handlers/`, `templates/`,
`files/`).

### Organisation Ansible

-   `inventory/` : déclaration des machines et variables.
-   `group_vars/` : variables communes aux groupes.
-   `playbooks/` : orchestration des déploiements.
-   `roles/common/` : préparation des systèmes.
-   `roles/wazuh-server/` : installation du serveur Wazuh.
-   `roles/wazuh-agent/` : installation et configuration de l'agent Wazuh.
-   `roles/caldera-server/` : installation et configuration du serveur MITRE Caldera, de son environnement applicatif et de son service systemd.

------------------------------------------------------------------------

## 📋 Prérequis

Sur le poste hôte Linux :

-   VirtualBox installé.
-   Vagrant installé.
-   Ansible installé.
-   Connexion Internet pour télécharger les images et paquets.
-   Virtualisation matérielle activée dans le BIOS/UEFI.
-   Ressources suffisantes pour exécuter les trois VM simultanément (prévoir environ 14 Go de RAM selon la configuration retenue).

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
ping -c 4 192.168.56.12
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

Les trois machines doivent répondre avec `pong`.

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

### 10. Déployer le serveur MITRE Caldera

Le serveur Caldera est déployé sur la VM `caldera-server` (`192.168.56.12`)
par le rôle Ansible `caldera-server`. Le playbook dédié est
`playbooks/caldera-server.yml`.

Vérifier la connexion SSH et l'inventaire :

```bash
ansible caldera_server -m ping
```

Vérifier la syntaxe du playbook :

```bash
ansible-playbook playbooks/caldera-server.yml --syntax-check
```

Lancer le déploiement :

```bash
ansible-playbook playbooks/caldera-server.yml
```

Si l'inventaire charge des variables chiffrées avec Ansible Vault, ajouter
`--ask-vault-pass` aux commandes.

Le rôle installe Caldera dans `/opt/caldera` et configure le service
systemd `caldera`. L'interface web est accessible depuis le poste hôte à
`http://192.168.56.12:8888`.

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
vagrant ssh caldera-server
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
ansible-playbook playbooks/caldera-server.yml --syntax-check
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

## 🌐 Accès aux interfaces web

### Wazuh Dashboard

Une fois le serveur déployé et les services actifs, ouvrir :

**https://192.168.56.10**

Le certificat TLS du laboratoire peut être auto-signé. Les identifiants
initiaux sont générés lors de l'installation et doivent être conservés
de manière sécurisée. Dans le Dashboard, vérifier que `client-01`
apparaît avec le statut **Active**.

### MITRE Caldera

Ouvrir l'interface depuis le poste hôte :

**http://192.168.56.12:8888**

Dans la configuration actuelle, le service est lancé avec l'option
`--insecure`. Les journaux indiquent que Caldera utilise alors les
comptes définis dans `conf/default.yml`. Consulte ce fichier directement
sur le serveur pour retrouver les identifiants configurés.

> **Sécurité :** `--insecure` utilise les comptes par défaut. Ce mode
> est réservé au laboratoire isolé et ne doit pas être utilisé sur une
> instance exposée à Internet ou à un réseau non fiable. Ne publie jamais
> les identifiants dans le dépôt. Prévois une authentification adaptée
> avant toute exposition hors du réseau privé.

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

### Le service Caldera ne démarre pas ou l'interface est inaccessible

Sur `caldera-server` :

```bash
sudo systemctl status caldera
sudo journalctl -u caldera -n 100 --no-pager
sudo ss -lntp | grep 8888
```

Suivre les journaux en direct :

```bash
sudo journalctl -u caldera -f
```

#### Erreur npm `EACCES` sur `/home/caldera/.npm`

Si la compilation de l'interface Vue.js échoue avec une erreur de
permissions, corriger le propriétaire du cache npm :

```bash
sudo chown -R caldera:caldera /home/caldera/.npm
sudo systemctl restart caldera
```

Le rôle Ansible `caldera-server` doit également garantir que le cache npm
appartient à l'utilisateur de service `caldera`, afin que la correction
soit persistante après un nouveau déploiement.

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
-   [MITRE Caldera](https://caldera.mitre.org/)
-   [MITRE Caldera GitHub](https://github.com/mitre/caldera)
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
-   [ ] Documenter les scénarios pédagogiques Purple Team et leur détection dans Wazuh.
-   [ ] Renforcer la configuration de sécurité de Caldera avant toute utilisation sur un réseau autre que le Host-only.
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

**Projet personnel --- Lab Cyber Wazuh & Caldera**\
*Infrastructure as Code · Automatisation · Supervision · Détection*
