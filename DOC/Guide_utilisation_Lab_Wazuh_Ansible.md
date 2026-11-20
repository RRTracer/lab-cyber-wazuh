# Guide d'utilisation --- Laboratoire Wazuh + Ansible

> **Projet :** `lab-cyber-wazuh`\
> **Environnement :** Linux hôte, Vagrant, VirtualBox, Ubuntu Server
> 22.04, Ansible, Wazuh 4.x\
> **Objectif :** disposer d'un laboratoire local permettant de déployer
> un serveur Wazuh et un agent Linux, puis d'expérimenter la supervision
> et la détection.

------------------------------------------------------------------------

## 1. Présentation du laboratoire

Ce laboratoire met en place une architecture centralisée de supervision
de sécurité. Une machine héberge la plateforme Wazuh et une seconde
machine joue le rôle de poste supervisé.

### 1.1 Architecture logique

``` text
                 Poste physique Linux
              Ansible + Vagrant + VirtualBox
                           |
                   Réseau Host-only
                    192.168.56.0/24
                           |
             +-------------+-------------+
             |                           |
      wazuh-01                      client-01
    192.168.56.10                 192.168.56.11
             |                           |
      Wazuh Manager                 Wazuh Agent
      Wazuh Indexer                       |
      Wazuh Dashboard                    |
             |                           |
             +<------ TCP 1514 ----------+
             +<------ TCP 1515 ----------+
```

-   **Poste hôte :** exécute Ansible et pilote le cycle de vie des VM.
-   **`wazuh-01` :** centralise les événements, les analyse, les indexe
    et expose le tableau de bord.
-   **`client-01` :** exécute l'agent qui collecte les événements et
    communique avec le Manager.

### 1.2 Inventaire des machines

  -------------------------------------------------------------------------
  Machine                      Adresse Rôle              Ressources prévues
  --------------- -------------------- --------------- --------------------
  `wazuh-01`           `192.168.56.10` Manager,           4 vCPU, 8 Go RAM,
                                       Indexer,                disque 60 Go
                                       Dashboard       

  `client-01`          `192.168.56.11` Agent Linux         2 vCPU, 2 Go RAM
  -------------------------------------------------------------------------

Les ressources sont les valeurs prévues dans le Vagrantfile du
laboratoire. Elles peuvent être ajustées selon les capacités de l'hôte.

### 1.3 Flux réseau essentiels

  Source        Destination   Port/protocole   Usage
  ------------- ------------- ---------------- ------------------------------
  Poste hôte    `wazuh-01`    TCP 443          Interface web HTTPS
  `client-01`   `wazuh-01`    TCP 1514         Communication agent/manager
  `client-01`   `wazuh-01`    TCP 1515         Enrôlement de l'agent
  Poste hôte    VM            TCP 22           Administration SSH / Ansible

Le port 1515 est généralement utilisé pour l'enrôlement automatique. Il
n'est pas nécessaire de l'exposer à des réseaux non fiables. Dans ce
laboratoire, les VM communiquent sur le réseau privé VirtualBox.

------------------------------------------------------------------------

## 2. Prérequis

Sur le poste hôte :

-   Linux avec une session utilisateur disposant des droits nécessaires
    à VirtualBox.
-   VirtualBox installé et fonctionnel.
-   Vagrant installé.
-   Ansible installé sur le poste hôte.
-   Accès Internet depuis les VM pour récupérer les paquets et
    l'assistant Wazuh.
-   Ressources suffisantes pour exécuter les deux VM simultanément.

Vérifier les versions :

``` bash
vagrant --version
VBoxManage --version
ansible --version
```

Vérifier que VirtualBox voit ses interfaces réseau :

``` bash
VBoxManage list hostonlyifs
```

> La version exacte de Wazuh et les versions de paquets doivent être
> contrôlées dans les fichiers du projet et dans la documentation
> officielle avant une nouvelle installation. Ne pas supposer qu'une
> version mentionnée dans une ancienne note est encore la dernière
> version disponible.

------------------------------------------------------------------------

## 3. Arborescence du projet

L'organisation actuellement retenue sépare l'infrastructure Vagrant de
l'automatisation Ansible :

``` text
lab-cyber-wazuh/
├── DOC/
│   └── Doc.md
├── Infra/
│   └── ansible/
│       ├── ansible.cfg
│       ├── inventory/
│       │   ├── hosts.yml
│       │   └── group_vars/
│       │       └── all/
│       │           ├── vars.yml
│       │           └── vault.yml
│       ├── playbooks/
│       │   ├── common.yml
│       │   ├── wazuh-server.yml
│       │   ├── wazuh-agent.yml
│       │   └── site.yml
│       ├── roles/
│       │   ├── common/
│       │   ├── wazuh-server/
│       │   └── wazuh-agent/
│       └── artifacts/
├── Vagrantfile
├── LICENSE
└── README.md
```

Certains fichiers ou dossiers peuvent ne pas encore exister si les
étapes correspondantes n'ont pas été réalisées. Adapter cette
arborescence à l'état réel du dépôt.

### 3.1 Rôle de chaque répertoire

  --------------------------------------------------------------------------
  Chemin                                 Responsabilité
  -------------------------------------- -----------------------------------
  `Vagrantfile`                    Définition des VM, ressources et
                                         réseaux

  `Infra/ansible/ansible.cfg`            Configuration générale d'Ansible

  `inventory/hosts.yml`                  Déclaration des hôtes et groupes

  `inventory/group_vars/all/vars.yml`    Variables non sensibles communes

  `inventory/group_vars/all/vault.yml`   Variables chiffrées avec Ansible
                                         Vault

  `playbooks/`                           Orchestration des opérations

  `roles/common/`                        Préparation commune des systèmes

  `roles/wazuh-server/`                  Déploiement du serveur Wazuh

  `roles/wazuh-agent/`                   Déploiement de l'agent Wazuh

  `artifacts/`                           Artefacts d'installation et
                                         archives sensibles, à exclure de
                                         Git
  --------------------------------------------------------------------------

------------------------------------------------------------------------

## 4. Démarrer et arrêter le laboratoire

Toutes les commandes Vagrant doivent être lancées depuis le répertoire
contenant le `Vagrantfile`, donc ici `Infra/`.

``` bash
cd ~/Documents/lab-cyber-wazuh/Infra
```

### 4.1 État des machines

``` bash
vagrant status
```

### 4.2 Démarrer les deux VM

``` bash
vagrant up
```

Démarrer une seule VM :

``` bash
vagrant up wazuh
vagrant up client
```

### 4.3 Arrêter proprement

``` bash
vagrant halt
```

Ou une seule machine :

``` bash
vagrant halt client
```

### 4.4 Redémarrer

``` bash
vagrant reload
```

### 4.5 Suspendre / reprendre

``` bash
vagrant suspend
vagrant resume
```

### 4.6 Détruire les VM

``` bash
vagrant destroy
```

Cette commande supprime les VM et leurs disques gérés par Vagrant. Les
données stockées uniquement dans les disques virtuels seront perdues.
Les fichiers du dépôt et les sauvegardes conservées hors des VM ne sont
pas supprimés.

> Ne pas utiliser `vagrant destroy -f` par réflexe. Vérifier d'abord que
> les données utiles ont été sauvegardées.

------------------------------------------------------------------------

## 5. Accéder aux machines

### 5.1 Connexion avec Vagrant

Depuis `Infra/` :

``` bash
vagrant ssh wazuh
vagrant ssh client
```

Exécuter une commande sans ouvrir de session :

``` bash
vagrant ssh wazuh -c "hostnamectl"
vagrant ssh client -c "ip -br address"
```

### 5.2 Connexion SSH directe

Afficher la configuration SSH générée par Vagrant :

``` bash
vagrant ssh-config wazuh
vagrant ssh-config client
```

Cette sortie donne notamment le nom d'hôte, le port SSH, l'utilisateur
et le chemin de la clé privée.

### 5.3 Vérifier les adresses IP

Sur chaque VM :

``` bash
ip -br address
ip route
```

Depuis le poste hôte :

``` bash
ping -c 4 192.168.56.10
ping -c 4 192.168.56.11
```

Si le ping échoue, vérifier l'état des VM, l'adresse privée, l'interface
host-only VirtualBox et les règles de pare-feu.

------------------------------------------------------------------------

## 6. Utiliser Ansible

Les commandes Ansible sont exécutées depuis le répertoire
`Infra/ansible` :

``` bash
cd ~/Documents/lab-cyber-wazuh/Infra/ansible
```

### 6.1 Vérifier la configuration

``` bash
ansible --version
ansible-config dump --only-changed
```

### 6.2 Afficher l'inventaire

``` bash
ansible-inventory --graph
ansible-inventory --list
```

L'inventaire YAML doit déclarer les groupes `wazuh_server` et
`wazuh_agents`, avec les adresses IP, l'utilisateur `vagrant` et les
clés SSH correspondantes.

### 6.3 Tester la connectivité

``` bash
ansible all -m ping --ask-vault-pass
```

Résultat attendu : `SUCCESS` et `pong` pour chaque hôte joignable.

Si Ansible ne doit pas charger de variables chiffrées pour un test
particulier, vérifier la structure des `group_vars`. Le message
`Attempting to decrypt but no vault secrets found` indique qu'un fichier
Vault est chargé sans que le mot de passe Vault soit fourni.

### 6.4 Vérifier les privilèges

``` bash
ansible all -b -m command -a "id" --ask-vault-pass
```

Le résultat doit montrer un contexte root lorsque `become` est
correctement configuré.

### 6.5 Lancer le socle commun

``` bash
ansible-playbook playbooks/common.yml --ask-vault-pass
```

Le rôle commun peut notamment mettre à jour le cache APT et installer
les utilitaires nécessaires. Examiner les changements avant de lancer
une mise à niveau de distribution sur une VM importante.

### 6.6 Déployer le serveur Wazuh

``` bash
ansible-playbook playbooks/wazuh-server.yml --syntax-check --ask-vault-pass
ansible-playbook playbooks/wazuh-server.yml --ask-vault-pass
```

Le rôle serveur prévu utilise l'assistant officiel Wazuh en mode
All-in-One pour installer Manager, Indexer et Dashboard sur `wazuh-01`.

### 6.7 Déployer l'agent

``` bash
ansible-playbook playbooks/wazuh-agent.yml --syntax-check --ask-vault-pass
ansible-playbook playbooks/wazuh-agent.yml --ask-vault-pass
```

Le playbook agent récupère d'abord la version installée du paquet
`wazuh-manager`, puis la transmet au rôle de l'agent afin de conserver
une version cohérente.

### 6.8 Déployer l'ensemble

Si `site.yml` importe bien les playbooks commun, serveur et agent dans
le bon ordre :

``` bash
ansible-playbook playbooks/site.yml --ask-vault-pass
```

Pour une première installation, il est préférable d'exécuter les
playbooks séparément et de valider chaque étape avant d'utiliser
l'orchestration globale.

------------------------------------------------------------------------

## 7. Ansible Vault et secrets

Ansible Vault chiffre les variables sensibles stockées dans des fichiers
YAML.

Créer ou modifier le fichier Vault :

``` bash
ansible-vault create inventory/group_vars/all/vault.yml
ansible-vault edit inventory/group_vars/all/vault.yml
```

Afficher le contenu en clair dans le terminal (à éviter dans un terminal
partagé ou enregistré) :

``` bash
ansible-vault view inventory/group_vars/all/vault.yml
```

Exécuter un playbook en demandant le mot de passe :

``` bash
ansible-playbook playbooks/site.yml --ask-vault-pass
```

### 7.1 Fichier de mot de passe local (facultatif)

Pour éviter de saisir le mot de passe à chaque exécution, il est
possible d'utiliser un fichier local protégé :

``` bash
printf '%s\n' 'MOT_DE_PASSE_VAULT' > .vault_pass
chmod 600 .vault_pass
```

Puis configurer `ansible.cfg` :

``` ini
[defaults]
vault_password_file = ./.vault_pass
```

Le fichier `.vault_pass` doit être ignoré par Git. Ne jamais l'ajouter
au dépôt, à un ticket ou à une capture d'écran.

### 7.2 Générer un secret

Exemple de génération locale :

``` bash
openssl rand -base64 32
```

Ne pas coller les secrets dans les journaux de commandes, les messages,
les commits ou les sorties de debug.

------------------------------------------------------------------------

## 8. Accéder au Wazuh Dashboard

Depuis le poste hôte, ouvrir :

``` text
https://192.168.56.10
```

Le certificat généré par une installation de laboratoire peut être
auto-signé. Le navigateur peut donc afficher un avertissement. Cela ne
doit pas être reproduit tel quel en production : utiliser une chaîne de
confiance adaptée et protéger les accès.

Les identifiants initiaux sont générés par l'assistant d'installation.
L'archive de déploiement peut contenir des mots de passe et des
certificats.

Si l'archive a été récupérée localement, vérifier ses permissions :

``` bash
ls -l artifacts/
chmod 600 artifacts/wazuh-install-files.tar
```

Afficher le fichier de mots de passe uniquement dans un environnement
local protégé :

``` bash
tar -O -xf artifacts/wazuh-install-files.tar \
  wazuh-install-files/wazuh-passwords.txt
```

Ne pas publier cette archive. La placer dans `.gitignore` et prévoir une
sauvegarde sécurisée si les identifiants doivent être conservés.

------------------------------------------------------------------------

## 9. Vérifier le serveur Wazuh

Se connecter à `wazuh-01` :

``` bash
vagrant ssh wazuh
```

### 9.1 État des services

``` bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```

Résumé rapide :

``` bash
sudo systemctl is-active wazuh-manager wazuh-indexer wazuh-dashboard
```

### 9.2 Ports en écoute

``` bash
sudo ss -lntup
```

Filtrer les ports principaux :

``` bash
sudo ss -lntup | grep -E ':(443|1514|1515)\b'
```

### 9.3 Journaux

``` bash
sudo journalctl -u wazuh-manager -n 100 --no-pager
sudo journalctl -u wazuh-indexer -n 100 --no-pager
sudo journalctl -u wazuh-dashboard -n 100 --no-pager
```

Journaux applicatifs :

``` bash
sudo tail -n 100 /var/ossec/logs/ossec.log
```

### 9.4 Vérifier le Dashboard depuis l'hôte

``` bash
curl -k -I https://192.168.56.10
```

`-k` désactive la validation du certificat TLS pour ce test seulement.
Ne pas l'utiliser comme solution de sécurité permanente dans une
application.

------------------------------------------------------------------------

## 10. Vérifier l'agent

Se connecter à `client-01` :

``` bash
vagrant ssh client
```

### 10.1 Service

``` bash
sudo systemctl status wazuh-agent
sudo systemctl is-active wazuh-agent
```

### 10.2 Configuration

Le fichier principal est généralement :

``` text
/var/ossec/etc/ossec.conf
```

Inspecter la configuration Manager sans modifier le fichier à l'aveugle
:

``` bash
sudo grep -A8 -B2 '<client>' /var/ossec/etc/ossec.conf
```

Le Manager configuré doit être `192.168.56.10`.

### 10.3 Journaux de l'agent

``` bash
sudo tail -n 100 /var/ossec/logs/ossec.log
sudo journalctl -u wazuh-agent -n 100 --no-pager
```

### 10.4 Vérifier le réseau vers le Manager

``` bash
nc -vz 192.168.56.10 1514
nc -vz 192.168.56.10 1515
```

Si `nc` n'est pas installé :

``` bash
sudo apt update
sudo apt install netcat-openbsd
```

La connectivité TCP ne prouve pas à elle seule que l'enrôlement est
réussi : confirmer également le statut de l'agent dans le Dashboard et
les journaux côté Manager.

------------------------------------------------------------------------

## 11. Cycle de vie d'un agent

Un agent doit être installé, configuré, enrôlé et connecté au Manager.

### 11.1 Enrôlement

Le processus habituel comprend : 1. l'agent connaît l'adresse du Manager
; 2. l'agent contacte le service d'enrôlement si l'enrôlement
automatique est configuré ; 3. le Manager valide l'identité de l'agent
et lui attribue une clé ; 4. l'agent utilise cette identité pour
communiquer ; 5. le Manager affiche l'agent comme actif lorsqu'il
communique correctement.

La configuration exacte dépend de la méthode d'enrôlement et de la
politique de sécurité du laboratoire.

### 11.2 Vérification dans le Dashboard

Dans l'interface, consulter la section de gestion des agents. Vérifier
: - que `client-01` est présent ; - que son adresse IP correspond à
`192.168.56.11` ; - que son statut est `Active` ; - que les événements
remontent avec des horodatages cohérents.

Un agent absent ou inactif ne doit pas être considéré comme supervisé.

------------------------------------------------------------------------

## 12. Dépannage

### 12.1 `REMOTE HOST IDENTIFICATION HAS CHANGED`

Cette alerte signifie que la clé d'hôte présentée par la VM diffère de
celle enregistrée sur le poste hôte. Cela arrive souvent après la
destruction et la recréation d'une VM.

Après avoir vérifié que l'adresse correspond bien à ta VM de laboratoire
:

``` bash
ssh-keygen -f ~/.ssh/known_hosts -R 192.168.56.10
ssh-keygen -f ~/.ssh/known_hosts -R 192.168.56.11
```

Reconnecte-toi ensuite et vérifie la nouvelle empreinte. Ne supprime pas
les clés d'hôtes sans vérifier l'identité de la machine.

### 12.2 `no such identity: ... private_key`

Ansible ne trouve pas la clé privée indiquée dans l'inventaire.

Depuis `Infra/`, Vagrant conserve normalement son état local dans
`.vagrant/`. Dans ce projet, ce répertoire est attendu à côté du
`Vagrantfile`, sous `Infra/.vagrant/`.

Depuis `Infra/ansible`, les chemins relatifs correspondants sont donc
généralement :

``` yaml
ansible_ssh_private_key_file: ../.vagrant/machines/wazuh/virtualbox/private_key
```

et :

``` yaml
ansible_ssh_private_key_file: ../.vagrant/machines/client/virtualbox/private_key
```

Vérifier les chemins réels :

``` bash
ls -l ../.vagrant/machines/wazuh/virtualbox/private_key
ls -l ../.vagrant/machines/client/virtualbox/private_key
```

Comparer avec la sortie de :

``` bash
cd ..
vagrant ssh-config wazuh
vagrant ssh-config client
```

### 12.3 `Permission denied (publickey)`

Vérifier : - le bon utilisateur SSH (`vagrant`) ; - le chemin de la clé
privée ; - les permissions de la clé ; - l'état de la VM ; - la
configuration fournie par `vagrant ssh-config`.

Ne pas remplacer une clé privée au hasard et ne pas partager son
contenu.

### 12.4 `Attempting to decrypt but no vault secrets found`

Un fichier chiffré est chargé, mais Ansible ne dispose pas du mot de
passe Vault.

Solution ponctuelle :

``` bash
ansible all -m ping --ask-vault-pass
```

Ou configurer un fichier de mot de passe local, protégé et ignoré par
Git.

### 12.5 L'agent ne devient pas `Active`

Vérifier dans cet ordre :

1.  `wazuh-manager` est actif sur le serveur.
2.  Le port TCP 1514 est joignable depuis l'agent.
3.  Le port TCP 1515 est joignable si l'enrôlement automatique
    l'utilise.
4.  L'adresse du Manager dans `ossec.conf` est correcte.
5.  L'enrôlement est configuré et l'identité de l'agent est valide.
6.  Les horloges et les journaux ne révèlent pas d'erreur de connexion
    ou de certificat.

Commandes utiles :

``` bash
# Sur le serveur
sudo systemctl status wazuh-manager
sudo tail -n 100 /var/ossec/logs/ossec.log

# Sur le client
sudo systemctl status wazuh-agent
sudo tail -n 100 /var/ossec/logs/ossec.log
```

### 12.6 Le Dashboard ne répond pas

Vérifier le service :

``` bash
sudo systemctl status wazuh-dashboard
sudo journalctl -u wazuh-dashboard -n 100 --no-pager
sudo ss -lntp | grep ':443'
```

Vérifier également l'état de l'Indexer et du Manager. Le Dashboard
dépend de l'Indexer pour afficher les données.

### 12.7 Disque ou mémoire insuffisants

Sur les VM :

``` bash
free -h
df -h
lsblk
```

Sur l'hôte, vérifier la mémoire disponible et les ressources allouées à
VirtualBox. Indexer peut être particulièrement consommateur de mémoire
et d'espace disque.

------------------------------------------------------------------------

## 13. Bonnes pratiques d'exploitation du lab

-   Démarrer les VM avant d'exécuter Ansible.
-   Exécuter les playbooks par étape lors d'une première installation.
-   Vérifier la syntaxe des playbooks avant leur exécution.
-   Ne jamais stocker les secrets en clair dans Git.
-   Exclure `.vagrant/`, les clés privées, les archives de mots de passe
    et les fichiers de mot de passe Vault.
-   Conserver les versions des composants cohérentes.
-   Consulter les journaux avant de relancer une installation qui a
    échoué.
-   Éviter les réinstallations répétées sur une VM partiellement
    configurée.
-   Sauvegarder les données et certificats utiles avant toute
    destruction de VM.
-   Ne pas exposer les ports d'administration ou d'enrôlement à un
    réseau public.

### 13.1 Exemple de `.gitignore`

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

------------------------------------------------------------------------

## 14. Routine de travail recommandée

### Démarrage quotidien

``` bash
cd ~/Documents/lab-cyber-wazuh/Infra
vagrant up
```

Puis :

``` bash
cd ansible
ansible all -m ping --ask-vault-pass
```

### Vérification rapide de la plateforme

``` bash
ansible wazuh_server -b -m command -a \
  "systemctl is-active wazuh-manager wazuh-indexer wazuh-dashboard" \
  --ask-vault-pass

ansible wazuh_agents -b -m command -a \
  "systemctl is-active wazuh-agent" \
  --ask-vault-pass
```

### Arrêt

``` bash
cd ..
vagrant halt
```

------------------------------------------------------------------------

## 15. Scénarios d'exercices

Ces exercices sont destinés à un environnement de laboratoire isolé et à
des machines que tu administres.

### Exercice 1 --- Inventaire et disponibilité

-   Démarrer les VM.
-   Afficher l'inventaire Ansible.
-   Tester la connectivité SSH.
-   Exécuter `ansible all -m ping`.
-   Identifier les groupes et les variables d'hôte.

**Validation :** les deux hôtes répondent `pong`.

### Exercice 2 --- Déploiement reproductible

-   Exécuter le rôle `common`.
-   Exécuter le rôle serveur.
-   Exécuter le rôle agent.
-   Relancer les playbooks.
-   Comparer les tâches `changed` et `ok`.

**Validation :** une seconde exécution ne relance pas l'assistant
d'installation et ne modifie pas inutilement la configuration.

### Exercice 3 --- Analyse des journaux

-   Générer des événements système bénins sur `client-01`.
-   Observer les journaux locaux de l'agent.
-   Retrouver les événements correspondants dans Wazuh.
-   Identifier l'horodatage, l'agent source et les champs utiles.

**Validation :** les événements attendus sont visibles et correctement
attribués à `client-01`.

### Exercice 4 --- Surveillance d'intégrité

-   Identifier un répertoire de test non sensible.
-   Configurer une surveillance FIM adaptée à ce répertoire.
-   Effectuer une modification bénigne d'un fichier de test.
-   Vérifier que l'événement d'intégrité remonte.

**Validation :** l'événement est visible dans le Dashboard et peut être
expliqué.

### Exercice 5 --- Résilience et diagnostic

-   Arrêter proprement l'agent.
-   Constater son changement d'état dans Wazuh.
-   Redémarrer l'agent.
-   Vérifier son retour à l'état actif.
-   Examiner les journaux pour comprendre la chronologie.

**Validation :** tu peux expliquer le cycle de perte et de reprise de
communication.

------------------------------------------------------------------------

## 16. Limites et statut du projet

Ce document décrit l'architecture et les commandes prévues à partir de
la configuration du laboratoire. Il ne constitue pas une preuve que
chaque étape a été exécutée avec succès.

Avant de considérer le laboratoire comme opérationnel, confirmer : - que
les deux VM démarrent ; - que les clés SSH sont valides ; - que les
playbooks passent la vérification syntaxique ; - que les trois services
serveur sont actifs ; - que le Dashboard est accessible ; - que
`client-01` apparaît `Active` ; - que des événements sont effectivement
reçus.

Une installation All-in-One est adaptée à un petit laboratoire, mais ne
représente pas une architecture de production hautement disponible ou
dimensionnée pour une grande volumétrie.

------------------------------------------------------------------------

## 17. Références officielles

-   Documentation Wazuh : https://documentation.wazuh.com/
-   Installation du serveur Wazuh :
    https://documentation.wazuh.com/current/installation-guide/
-   Installation des agents :
    https://documentation.wazuh.com/current/installation-guide/wazuh-agent/
-   Documentation Ansible : https://docs.ansible.com/
-   Documentation Vagrant : https://developer.hashicorp.com/vagrant/docs
-   Documentation VirtualBox : https://www.virtualbox.org/manual/

Toujours consulter la documentation correspondant à la version
réellement installée.
