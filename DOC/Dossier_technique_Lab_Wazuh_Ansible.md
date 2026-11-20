# Dossier technique --- Conception et réalisation du laboratoire Wazuh / Ansible

> **Public visé :** administrateurs systèmes et réseaux, ingénieurs
> DevOps, analystes SOC, étudiants en cybersécurité et lecteurs
> souhaitant comprendre les choix techniques du projet.\
> **Projet :** `lab-cyber-wazuh`\
> **Périmètre :** virtualisation locale, réseau, provisionnement,
> automatisation de configuration et déploiement Wazuh.

------------------------------------------------------------------------

## 1. Résumé technique

Le projet met en œuvre un environnement de supervision de sécurité
composé de deux machines virtuelles Ubuntu Server 22.04. La première
héberge une installation Wazuh All-in-One. La seconde exécute un agent
Linux supervisé par le Manager.

Le poste physique joue le rôle de nœud de contrôle : il héberge Vagrant,
VirtualBox et Ansible. Vagrant gère le cycle de vie des machines
virtuelles, tandis qu'Ansible configure les systèmes et déploie les
composants logiciels.

La séparation des responsabilités est volontaire :

-   **VirtualBox** fournit l'hyperviseur de type 2.
-   **Vagrant** décrit et pilote les machines virtuelles.
-   **Ansible** applique la configuration à distance via SSH.
-   **Wazuh** collecte, analyse, indexe et présente les événements de
    sécurité.

Cette architecture permet de reconstruire le laboratoire sans installer
manuellement chaque composant dans chaque VM.

------------------------------------------------------------------------

## 2. Architecture générale

### 2.1 Vue physique et logique

``` text
+--------------------------------------------------------------+
|                       Poste hôte Linux                       |
|                                                              |
|  +----------+       +----------+       +------------------+  |
|  | Vagrant  | ----> |VirtualBox|       | Ansible          |  |
|  | Lifecycle|       | Hypervisor|      | Control node     |  |
|  +----------+       +-----+----+       +--------+---------+  |
|                            |                     |            |
+----------------------------|---------------------|------------+
                             |                     |
                    Host-only network              | SSH
                     192.168.56.0/24                |
                             |                     |
              +--------------+---------------------+----------+
              |                                                 |
       +------+-------+                                  +------+------+
       | wazuh-01     |                                  | client-01   |
       | .10          |                                  | .11         |
       +--------------+                                  +-------------+
       | Manager      |<----------- TCP 1514 -------------| Agent       |
       | Indexer      |<----------- TCP 1515 -------------|             |
       | Dashboard    |                                  |             |
       +--------------+                                  +-------------+
```

### 2.2 Plan d'adressage

Le réseau Host-only permet au poste hôte et aux VM de communiquer via un
réseau privé local.

  Élément        Adresse
  -------------- -------------------
  Réseau         `192.168.56.0/24`
  Wazuh Server   `192.168.56.10`
  Agent          `192.168.56.11`

Le réseau NAT de Vagrant peut rester présent pour fournir l'accès
Internet aux VM. Le réseau Host-only sert aux échanges de laboratoire et
à l'accès depuis l'hôte.

> La présence exacte des interfaces dépend du Vagrantfile et du
> fournisseur VirtualBox. Vérifier les interfaces et routes réellement
> créées avec `ip -br address` et `ip route`.

------------------------------------------------------------------------

## 3. Choix de virtualisation

### 3.1 Pourquoi VirtualBox ?

VirtualBox est un hyperviseur hébergé qui permet d'exécuter des systèmes
invités sur le poste de travail. Il fournit les ressources CPU, mémoire,
stockage et réseau nécessaires aux VM.

Avantages pour ce laboratoire : - environnement local ; - coût d'entrée
faible ; - snapshots possibles selon le mode de stockage ; - intégration
Vagrant ; - réseaux NAT et Host-only ; - destruction et reconstruction
rapides des VM.

Limites : - les performances dépendent du poste hôte ; - la mémoire de
l'hôte est partagée avec les VM ; - le laboratoire n'offre pas la
résilience d'une infrastructure multi-nœuds ; - les réseaux virtuels ne
reproduisent pas tous les comportements d'un réseau d'entreprise.

### 3.2 Pourquoi Vagrant ?

Vagrant formalise la définition des VM dans un fichier versionnable, le
`Vagrantfile`.

Il permet de déclarer : - l'image système de base ; - le nom d'hôte ; -
l'adresse IP privée ; - le nombre de processeurs ; - la mémoire ; - le
disque additionnel ou redimensionné selon le fournisseur.

Exemple de structure utilisée dans le projet :

``` ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
  config.vm.box_check_update = false

  machines = [
    {
      name: "wazuh",
      hostname: "wazuh-01",
      ip: "192.168.56.10",
      cpus: 4,
      memory: 8192
    },
    {
      name: "client",
      hostname: "client-01",
      ip: "192.168.56.11",
      cpus: 2,
      memory: 2048
    }
  ]

  machines.each do |machine|
    config.vm.define machine[:name] do |node|
      node.vm.hostname = machine[:hostname]
      node.vm.network "private_network", ip: machine[:ip]

      node.vm.provider "virtualbox" do |vb|
        vb.name = "wazuh-lab-#{machine[:name]}"
        vb.cpus = machine[:cpus]
        vb.memory = machine[:memory]
        vb.gui = false
      end
    end
  end
end
```

Le disque de la VM serveur a été prévu à 60 Go dans la configuration de
ce laboratoire. La syntaxe de gestion du disque dépend de la version de
Vagrant et du plugin/provider utilisé ; vérifier le Vagrantfile
réellement exécuté et `lsblk` dans la VM.

### 3.3 Pourquoi ne pas installer Wazuh directement sur l'hôte ?

L'isolation apporte plusieurs avantages : - les dépendances Wazuh
restent séparées du poste de travail ; - une VM peut être reconstruite
sans affecter l'hôte ; - les échanges réseau entre Manager et Agent sont
observables ; - les tests d'administration peuvent être réalisés sur une
machine dédiée ; - les ressources et services sont clairement délimités.

------------------------------------------------------------------------

## 4. Ansible : architecture de contrôle

### 4.1 Modèle agentless

Ansible fonctionne ici depuis le poste hôte et se connecte aux VM via
SSH. Il n'est pas nécessaire d'installer un agent Ansible permanent sur
les VM.

Le flux de gestion est :

1.  Ansible lit son inventaire.
2.  Il détermine les hôtes cibles et leurs variables.
3.  Il établit une connexion SSH.
4.  Il exécute les modules sur la machine distante.
5.  Il utilise `become` pour obtenir les privilèges nécessaires.
6.  Il retourne le résultat de chaque tâche.

### 4.2 Inventaire YAML

L'inventaire sépare les machines selon leur rôle :

``` yaml
---
all:
  children:
    wazuh_server:
      hosts:
        wazuh-01:
          ansible_host: 192.168.56.10
          ansible_user: vagrant
          ansible_ssh_private_key_file: ../.vagrant/machines/wazuh/virtualbox/private_key

    wazuh_agents:
      hosts:
        client-01:
          ansible_host: 192.168.56.11
          ansible_user: vagrant
          ansible_ssh_private_key_file: ../.vagrant/machines/client/virtualbox/private_key
```

Les chemins ci-dessus sont relatifs au répertoire de travail d'Ansible
dans l'organisation du projet décrite. Ils doivent être confirmés avec
`vagrant ssh-config` et l'existence réelle des fichiers de clé.

### 4.3 Configuration Ansible

Exemple de configuration :

``` ini
[defaults]
inventory = ./inventory/hosts.yml
roles_path = ./roles
host_key_checking = False
retry_files_enabled = False
interpreter_python = auto_silent

[privilege_escalation]
become = True
become_method = sudo
become_ask_pass = False

[ssh_connection]
pipelining = True
```

**Attention sécurité :** `host_key_checking = False` est une
simplification tolérable dans un environnement de test jetable, mais
elle désactive la vérification de l'identité SSH des hôtes. Dans un
environnement réel, conserver le contrôle des clés d'hôte et gérer
`known_hosts` proprement.

### 4.4 Rôles et idempotence

Un rôle Ansible regroupe les tâches liées à une fonction. Les rôles du
projet sont : - `common` : préparation commune ; - `wazuh-server` :
installation et démarrage des composants serveur ; - `wazuh-agent` :
installation et configuration de l'agent.

L'idempotence signifie qu'une tâche décrit un état cible plutôt qu'une
suite aveugle de commandes. Si l'état est déjà conforme, une nouvelle
exécution ne doit pas provoquer de changement inutile.

Modules déclaratifs à privilégier : - `ansible.builtin.apt` pour les
paquets ; - `ansible.builtin.copy` pour les fichiers ; -
`ansible.builtin.template` pour les fichiers paramétrés ; -
`ansible.builtin.file` pour les permissions et répertoires ; -
`ansible.builtin.systemd` pour les services ; -
`ansible.builtin.get_url` pour les téléchargements.

Le module `command` est nécessaire pour appeler un assistant
d'installation qui ne dispose pas d'un module Ansible dédié. Il doit
être protégé par une condition d'état afin de ne pas relancer
l'installation à chaque exécution.

------------------------------------------------------------------------

## 5. Gestion des secrets avec Ansible Vault

Le projet prévoit un fichier `inventory/group_vars/all/vault.yml`
chiffré avec Ansible Vault.

Exemple de variable (structure indicative) :

``` yaml
---
vault_wazuh_dashboard_password: "VALEUR_SECRETE"
```

Le contenu réel doit être chiffré, et non stocké en clair.

Commandes :

``` bash
ansible-vault create inventory/group_vars/all/vault.yml
ansible-vault edit inventory/group_vars/all/vault.yml
ansible-vault view inventory/group_vars/all/vault.yml
```

Pour lancer un playbook :

``` bash
ansible-playbook playbooks/site.yml --ask-vault-pass
```

Le mot de passe Vault est une clé de déchiffrement. Le fichier chiffré
peut être versionné si la politique du projet l'autorise, mais le mot de
passe lui-même ne doit jamais être commité.

Un fichier local de mot de passe peut être utilisé pour les opérations
répétitives, à condition d'être protégé par des permissions restrictives
et exclu de Git.

------------------------------------------------------------------------

## 6. Architecture Wazuh

### 6.1 Les composants

#### Wazuh Manager

Le Manager reçoit les données des agents, décode et analyse les
événements, applique les règles et produit des alertes.

#### Wazuh Indexer

L'Indexer stocke et indexe les données afin de permettre leur recherche
et leur exploitation par l'interface.

#### Wazuh Dashboard

Le Dashboard fournit l'interface web d'administration et d'analyse.

#### Wazuh Agent

L'agent est installé sur les systèmes supervisés. Il collecte les
données configurées localement et les transmet au Manager.

### 6.2 Pourquoi une installation All-in-One ?

Pour ce laboratoire, les trois composants serveur sont installés sur
`wazuh-01`.

Avantages : - simplicité d'installation ; - peu de VM nécessaires ; -
consommation de ressources plus faible qu'une architecture distribuée
; - adaptée à l'apprentissage et aux tests.

Limites : - point unique de défaillance ; - ressources CPU, RAM et
disque partagées ; - pas de haute disponibilité ; - séparation des rôles
moins proche d'une grande plateforme SOC.

Une architecture de production peut séparer Manager, Indexer et
Dashboard, voire utiliser plusieurs nœuds et des mécanismes de haute
disponibilité.

------------------------------------------------------------------------

## 7. Déploiement du serveur

### 7.1 Principe

Le rôle `wazuh-server` prépare le système, récupère l'assistant officiel
et l'exécute en mode All-in-One.

L'assistant officiel prend en charge une partie importante des
opérations : - ajout des dépôts nécessaires ; - installation des paquets
Wazuh ; - génération et mise en place de certificats ; - configuration
des composants ; - génération des informations d'accès initiales.

### 7.2 Contrôle de l'état avant installation

Le rôle utilise `package_facts` pour identifier les composants déjà
installés. Il distingue : - aucun composant installé : installation
initiale possible ; - tous les composants installés : ne pas relancer
l'assistant ; - installation partielle : arrêter et demander une
intervention plutôt que de tenter une réparation implicite.

Cette stratégie est prudente, mais ne remplace pas une procédure
complète de reprise après incident.

### 7.3 Téléchargement et exécution

L'assistant est récupéré avec `get_url`, puis appelé avec `command`. La
version de la branche de téléchargement doit être fixée et vérifiée. Il
faut aussi contrôler la somme de contrôle ou la signature du script
selon les mécanismes publiés par l'éditeur, plutôt que de faire
confiance aveuglément à un script téléchargé.

### 7.4 Récupération des secrets

L'installation génère une archive contenant notamment les mots de passe
et certificats. Le rôle peut la récupérer avec `fetch`.

Cet artefact est hautement sensible : - droits d'accès restreints ; -
exclusion de Git ; - stockage chiffré ou coffre-fort si conservation
nécessaire ; - aucune inclusion dans les logs ou les rapports publics.

------------------------------------------------------------------------

## 8. Déploiement de l'agent

### 8.1 Objectif

Le rôle `wazuh-agent` configure `client-01` pour communiquer avec le
Manager.

Il doit : 1. vérifier le système d'exploitation ; 2. installer les
dépendances ; 3. installer la clé du dépôt Wazuh ; 4. configurer le
dépôt APT ; 5. rafraîchir le cache APT ; 6. vérifier la disponibilité du
Manager ; 7. installer la version d'agent compatible ; 8. configurer
l'adresse et le nom de l'agent ; 9. activer et démarrer le service.

### 8.2 Cohérence de version

Le playbook récupère la version exacte du paquet `wazuh-manager` avec
`dpkg-query`, puis la transmet au rôle agent.

Cette approche évite de coder deux versions indépendantes. Elle suppose
néanmoins que la version exacte du paquet Manager est disponible dans le
dépôt configuré sur le client.

Le paquet peut être maintenu en état `hold` afin d'éviter une mise à
niveau accidentelle non coordonnée. Cette politique doit être réévaluée
lors des mises à jour planifiées.

### 8.3 Enrôlement

L'installation et l'enrôlement sont deux étapes distinctes sur le plan
fonctionnel. L'agent doit connaître le Manager et disposer d'une
identité reconnue par celui-ci.

Le port TCP 1515 est utilisé par le service d'enrôlement lorsque cette
méthode est activée. Le port TCP 1514 sert à la communication normale
agent/manager.

Ne pas supposer qu'un simple `systemctl is-active wazuh-agent` prouve
que l'agent est correctement enrôlé. Il faut vérifier son état côté
Manager et les journaux des deux côtés.

------------------------------------------------------------------------

## 9. Flux réseau et sécurité

### 9.1 Ports principaux

    Port Protocole   Flux               Fonction
  ------ ----------- ------------------ -----------------------------
      22 TCP         Hôte → VM          SSH / Ansible
     443 TCP         Hôte → Dashboard   Interface HTTPS
    1514 TCP         Agent → Manager    Événements et communication
    1515 TCP         Agent → Manager    Enrôlement

La configuration réelle peut inclure d'autres ports internes entre
Dashboard et Indexer, ou selon les fonctionnalités Wazuh activées. Ne
pas ouvrir ces ports au-delà de ce qui est nécessaire.

### 9.2 Segmentation

Le réseau Host-only isole les échanges du laboratoire du réseau physique
ordinaire. Il ne remplace pas un pare-feu et ne constitue pas une
frontière de sécurité absolue.

Bonnes pratiques : - ne pas exposer 1514 et 1515 à Internet ; - limiter
l'accès au Dashboard ; - protéger les clés SSH et les mots de passe ; -
conserver les mises à jour du système ; - utiliser des certificats TLS
de confiance dans un environnement partagé ; - limiter les privilèges
des comptes et des services.

------------------------------------------------------------------------

## 10. Observabilité et validation

### 10.1 Validation système

Sur chaque VM :

``` bash
hostnamectl
ip -br address
free -h
df -h
```

### 10.2 Validation services serveur

``` bash
systemctl is-active wazuh-manager
systemctl is-active wazuh-indexer
systemctl is-active wazuh-dashboard
```

### 10.3 Validation agent

``` bash
systemctl is-active wazuh-agent
```

### 10.4 Validation réseau

Depuis le client :

``` bash
nc -vz 192.168.56.10 1514
nc -vz 192.168.56.10 1515
```

### 10.5 Validation applicative

-   consulter les journaux Manager ;
-   consulter les journaux Agent ;
-   vérifier que l'agent est `Active` dans le Dashboard ;
-   vérifier que des événements sont indexés ;
-   confirmer que les événements sont attribués au bon agent.

Une validation complète doit croiser l'état du service, la connectivité
réseau, l'enrôlement et la réception effective des événements.

------------------------------------------------------------------------

## 11. Limites de l'implémentation actuelle et améliorations

Cette section est importante : une automatisation qui installe un
logiciel n'est pas automatiquement une automatisation de production.

### 11.1 Rôle serveur

Améliorations recommandées : - vérifier la version exacte de l'assistant
et son intégrité ; - gérer explicitement les mises à niveau ; - gérer
une installation partielle avec une procédure de reprise documentée ; -
vérifier les états de santé applicatifs, pas seulement les services
systemd ; - gérer les identifiants avec Vault ou un gestionnaire de
secrets ; - vérifier la configuration TLS ; - ajouter des tests de
convergence après déploiement.

### 11.2 Rôle agent

Améliorations recommandées : - gérer la configuration avec un template
`ossec.conf` ; - séparer l'installation du dépôt et la configuration du
service ; - gérer les changements de configuration avec un handler de
redémarrage ; - définir précisément le cycle d'enrôlement et la rotation
des clés ; - gérer les agents déjà enrôlés ; - gérer les mises à niveau
de manière coordonnée ; - vérifier l'état réel de l'agent côté Manager.

### 11.3 Idempotence plus fine

Le rôle agent présenté dans les premières étapes utilise les variables
`WAZUH_MANAGER`, `WAZUH_AGENT_NAME` et `WAZUH_AGENT_GROUP` à
l'installation du paquet. Il faut vérifier leur comportement exact pour
la version ciblée et ne pas supposer qu'elles réécrivent la
configuration lors d'une installation déjà présente.

Pour une configuration plus robuste, gérer explicitement
`/var/ossec/etc/ossec.conf` avec un template Ansible, puis déclencher un
handler `systemd` lorsque le fichier change.

### 11.4 Dépôt et clé APT

Le rôle doit utiliser une clé de dépôt vérifiée, un keyring dédié et une
définition APT conforme aux recommandations actuelles de Wazuh. Une clé
déjà présente ne doit pas empêcher sa mise à jour si l'éditeur la
renouvelle.

### 11.5 Tests automatisés

Ajouter : - `ansible-playbook --syntax-check` ; - `ansible-lint` ; -
tests de variables et d'inventaire ; - tests de connectivité ; -
contrôles de services ; - contrôle de l'enrôlement ; - validation d'une
seconde exécution sans changement inattendu.

------------------------------------------------------------------------

## 12. Pistes d'évolution du laboratoire

### 12.1 Supervision de l'intégrité (FIM)

Configurer File Integrity Monitoring pour suivre des répertoires de test
et détecter des changements contrôlés.

### 12.2 Security Configuration Assessment (SCA)

Activer et étudier les contrôles de configuration de sécurité sur
Ubuntu. Comparer les résultats à un référentiel et documenter les
écarts.

### 12.3 Collecte des journaux

Ajouter progressivement : - journaux système ; - authentification SSH
; - journaux APT ; - journaux de services ; - journaux applicatifs de
test.

### 12.4 Détection et règles

Étudier : - les événements bruts ; - les décodeurs ; - les règles ; -
les niveaux de sévérité ; - les faux positifs ; - la corrélation
d'événements.

Les tests doivent rester bénins et limités aux machines du laboratoire.

### 12.5 Infrastructure as Code plus complète

Évolutions possibles : - variables par environnement ; - inventaires
`dev`, `test` et `prod` ; - rôles réutilisables ; - Ansible Vault ou
coffre-fort externe ; - pipeline CI pour lint et syntax-check ; - tests
Molecule ; - sauvegarde de la configuration et des certificats ; -
procédure de restauration.

### 12.6 Architecture multi-nœuds

Pour se rapprocher d'un déploiement de taille supérieure : - séparer
Manager et Indexer ; - déployer plusieurs nœuds Indexer ; - isoler
Dashboard ; - définir des certificats et secrets par composant ; -
dimensionner selon le volume d'événements et la rétention.

------------------------------------------------------------------------

## 13. Glossaire

  -----------------------------------------------------------------------
  Terme                               Définition
  ----------------------------------- -----------------------------------
  Agent                               Composant installé sur un système
                                      supervisé

  Manager                             Composant central de réception et
                                      d'analyse

  Indexer                             Moteur de stockage et d'indexation

  Dashboard                           Interface web d'administration et
                                      d'analyse

  All-in-One                          Installation regroupant les
                                      composants serveur sur un hôte

  Provisionnement                     Création et préparation d'une
                                      infrastructure

  Configuration management            Application déclarative d'un état
                                      système

  Idempotence                         Propriété d'une opération qui peut
                                      être répétée sans effet
                                      supplémentaire indésirable

  Inventory                           Liste des hôtes gérés par Ansible

  Playbook                            Fichier YAML décrivant une
                                      orchestration Ansible

  Role                                Unité réutilisable de tâches,
                                      variables, fichiers et templates

  Handler                             Tâche déclenchée lorsqu'une tâche
                                      notifie un changement

  Vault                               Mécanisme Ansible de chiffrement
                                      des variables et fichiers

  Host-only                           Réseau privé entre l'hôte et les VM

  NAT                                 Mode réseau permettant à une VM de
                                      sortir via l'adresse de l'hôte
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 14. Conclusion

Ce laboratoire illustre une chaîne d'automatisation complète à petite
échelle :

1.  décrire l'infrastructure dans Vagrant ;
2.  créer les VM dans VirtualBox ;
3.  inventorier les hôtes dans Ansible ;
4.  préparer les systèmes avec un rôle commun ;
5.  installer la plateforme Wazuh ;
6.  déployer un agent ;
7.  valider la connectivité, l'enrôlement et les événements ;
8.  faire évoluer progressivement la supervision.

Le principal intérêt pédagogique n'est pas seulement d'obtenir un
Dashboard fonctionnel. Il est de comprendre la séparation entre
infrastructure, configuration, collecte de télémétrie, traitement des
événements et validation opérationnelle.

------------------------------------------------------------------------

## 15. Documentation de référence

-   Wazuh Documentation : https://documentation.wazuh.com/
-   Wazuh Installation Guide :
    https://documentation.wazuh.com/current/installation-guide/
-   Wazuh Agent deployment :
    https://documentation.wazuh.com/current/installation-guide/wazuh-agent/
-   Ansible Documentation : https://docs.ansible.com/
-   Ansible Best Practices :
    https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_best_practices.html
-   Ansible Vault :
    https://docs.ansible.com/ansible/latest/vault_guide/index.html
-   Vagrant Documentation : https://developer.hashicorp.com/vagrant/docs
-   VirtualBox Manual : https://www.virtualbox.org/manual/

Les documentations éditeurs évoluent. Vérifier les instructions de la
version réellement déployée avant toute modification de production.
