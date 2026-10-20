# Absicherung-und-berwachung-eines-Debian-Servers
# Absicherung und Überwachung eines Debian-Servers

**Auteur :** Maxime Corthésy  
**Module :** Cyber Security — Projektarbeit  
**État du document :** infrastructure et état initial documentés ; durcissement non commencé  
**Date de l'état initial :** 20 octobre 2026

## 1. Présentation du projet

### 1.1 Contexte

Ce projet consiste à sécuriser et à surveiller un serveur Debian dans un laboratoire virtualisé. Il applique les notions de sécurité abordées dans les supports du module Cyber Security, notamment le hardening, la protection des accès SSH, la journalisation et le contrôle de l'intégrité des fichiers.

Le travail comprend une implémentation fonctionnelle, une documentation et une présentation. Le laboratoire permet de réaliser des tests sur des machines dédiées et de comparer l'état initial du serveur avec son état après sécurisation.

Le projet porte sur la configuration et l'évaluation des mesures de sécurité. L'automatisation, le développement de scripts et la création d'une application ne constituent pas des objectifs. Vagrant et VirtualBox servent uniquement à mettre à disposition les machines virtuelles.

### 1.2 Problématique

Comment renforcer la sécurité d'un serveur Debian, protéger son accès SSH et détecter les modifications de fichiers sensibles, tout en conservant un accès d'administration fonctionnel ?

### 1.3 Objectif général

Mettre en place une solution de sécurité associant le durcissement SSH, la protection contre les échecs d'authentification répétés, le contrôle d'intégrité et la traçabilité des modifications. L'efficacité et les limites de chaque mesure seront vérifiées par des tests documentés.

## 2. Objectifs spécifiques et critères de validation

Les objectifs suivants décrivent le résultat attendu du projet. À ce stade, seuls l'infrastructure et l'état initial ont été vérifiés.

| ID | Objectif | Critère de validation | État |
|---|---|---|---|
| O1 | Disposer d'un laboratoire de deux VM Debian | Les deux VM démarrent et la machine de test atteint le serveur sur le réseau interne | Validé |
| O2 | Établir l'état initial du serveur | Interfaces, ports TCP accessibles et principaux paramètres SSH relevés | Validé |
| O3 | Disposer d'un point de restauration avant modification | Snapshot de l'état initial créé ; sa restauration n'a pas encore été testée | Snapshot créé |
| O4 | Renforcer la configuration SSH | Connexion directe de root interdite, X11 désactivé, limite de tentatives réduite et accès d'administration par clé vérifié | À réaliser |
| O5 | Protéger SSH avec Fail2ban | Des échecs d'authentification reconnus provoquent un bannissement temporaire ; le bannissement et sa levée sont vérifiés | À réaliser |
| O6 | Contrôler l'intégrité avec AIDE | Une modification contrôlée d'un fichier surveillé est signalée lors d'une vérification | À réaliser |
| O7 | Tracer des modifications avec auditd | Une modification sur un fichier surveillé produit un événement permettant d'identifier l'action et le contexte utilisateur | À réaliser |
| O8 | Vérifier le fonctionnement des protections | Résultats attendus et observés comparés, avec preuves et limites | À réaliser |
| O9 | Documenter et présenter la solution | Architecture, choix, configurations, tests et bilan expliqués | En cours |

## 3. Périmètre

### 3.1 Éléments inclus

- Un serveur Debian à sécuriser.
- Une machine Debian dédiée aux tests.
- L'administration distante par SSH.
- Le durcissement des paramètres SSH retenus pour le laboratoire.
- La protection des accès SSH avec Fail2ban.
- La surveillance d'un ensemble limité de fichiers avec AIDE et auditd.
- L'examen des événements de sécurité dans les journaux disponibles.
- La comparaison avant/après, la documentation et une démonstration.

### 3.2 Limites du travail

Le laboratoire ne comprend ni infrastructure de production, ni IDS réseau, ni application métier. Les tests ciblent exclusivement les VM du projet. La conformité complète à un benchmark CIS n'est pas revendiquée : chaque mesure retenue sera expliquée et testée individuellement.

Les protections ne permettront pas de garantir l'absence de toute vulnérabilité. Le projet devra préciser ce que chaque outil détecte ou bloque et ce qui reste hors de son champ d'action.

## 4. Outils et lien avec les cours

| Outil | Rôle | État dans le projet |
|---|---|---|
| VirtualBox | Hébergement des VM | Utilisé |
| Vagrant | Définition, démarrage et accès aux VM | Utilisé |
| Debian 12, box `debian/bookworm64` | Système des deux machines | Utilisé |
| OpenSSH | Administration distante du serveur | Actif et vérifié |
| Nmap | Vérification des ports TCP accessibles | Utilisé sur `tester` |
| Fail2ban | Détection d'échecs d'authentification et bannissement temporaire | À configurer |
| AIDE | Comparaison des fichiers avec une référence d'intégrité | À configurer |
| auditd | Audit des actions sur les fichiers retenus | À configurer |
| journald / rsyslog | Consultation et, selon la configuration retenue, conservation des événements | À examiner lors de la surveillance |



## 5. Architecture du laboratoire

### 5.1 Machines virtuelles

| Identifiant Vagrant | Nom système | Adresse du réseau interne | Ressources définies | Fonction |
|---|---|---|---|---|
| `server` | `security-server` | `192.168.50.10/24` | 1 vCPU, 1 024 Mo RAM | Serveur à sécuriser |
| `tester` | `security-tester` | `192.168.50.20/24` | 1 vCPU, 1 024 Mo RAM | Machine de test |



### 5.2 Réseaux et accès

Le réseau interne VirtualBox porte le nom `debian-security-lab` et utilise le sous-réseau `192.168.50.0/24`. Il permet aux deux VM de communiquer pour les tests.

Chaque VM dispose également de l'interface NAT habituelle de Vagrant/VirtualBox. Celle-ci permet notamment les connexions sortantes nécessaires à l'installation des paquets. 

Le réseau de test est interne à VirtualBox. Cela ne signifie pas que les VM sont totalement déconnectées : l'interface NAT reste présente. Les dossiers partagés Vagrant sont désactivés dans le Vagrantfile.

## 6. Mise en place de l'environnement



### 6.1 Préparation logicielle des VM

Les commandes suivantes ont été fournies pour préparer le laboratoire manuellement.

Sur `server` :

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
```

Sur `tester` :

```bash
sudo apt update
sudo apt install -y nmap
```

Les captures confirment qu'OpenSSH est à l'écoute sur le serveur et que Nmap fonctionne sur la machine de test. Elles ne constituent pas un journal d'exécution de chacune des commandes d'installation : celles-ci sont conservées ici comme procédure de préparation.

## 7. État initial — Ist-Zustand

### 7.1 Interfaces du serveur

Commande exécutée sur `server` :

```bash
ip -br address
```

| Interface | Adresse IPv4 observée | Fonction |
|---|---|---|
| `lo` | `127.0.0.1/8` | Boucle locale |
| `eth0` | `10.0.2.15/24` | Interface NAT VirtualBox |
| `eth1` | `192.168.50.10/24` | Réseau interne du laboratoire |

Des adresses IPv6 apparaissent également dans le relevé. Le scan Nmap présenté ci-dessous porte uniquement sur l'adresse IPv4 `192.168.50.10`.

### 7.2 Services en écoute sur le serveur

Commande exécutée :

```bash
sudo ss -lntup
```

| Protocole | Adresse et port observés | Processus | Interprétation |
|---|---|---|---|
| TCP | `0.0.0.0:22` et `[::]:22` | `sshd` | SSH écoute sur les adresses IPv4 et IPv6 du serveur |
| UDP | `127.0.0.1:323` et `[::1]:323` | `chronyd` | Port de commande de chrony limité à la boucle locale |
| UDP | `0.0.0.0:68` | `dhclient` | Client DHCP |

Un service en écoute localement n'est pas nécessairement accessible à distance. La commande `ss` décrit l'état local ; le scan depuis `tester` complète ce relevé par une observation réseau.

### 7.3 Vérification du service SSH depuis tester

Commande exécutée :

```bash
nmap -sT -sV -p 22 192.168.50.10
```

Le scan a identifié le port `22/tcp` ouvert et le service suivant :

```text
OpenSSH 9.2p1 Debian 2+deb12u10 (protocol 2.0)
```

Il s'agit de l'identification rapportée par Nmap à cette date, et non d'une validation de l'absence de vulnérabilités.

### 7.4 Scan de tous les ports TCP

Commande exécutée depuis `tester` :

```bash
nmap -sT -p- --open 192.168.50.10
```

Résultat fourni :

```text
Nmap scan report for 192.168.50.10
Host is up (0.0053s latency).
Not shown: 65534 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh

Nmap done: 1 IP address (1 host up) scanned in 46.38 seconds
```

Interprétation :

- Le serveur est accessible depuis la machine de test.
- Le scan a examiné les ports TCP de 1 à 65 535.
- Seul le port TCP 22 apparaît ouvert depuis cette machine et sur cette adresse.
- Les 65 534 autres ports TCP sont signalés comme fermés, avec refus de connexion.
- Le scan ne couvre ni les ports UDP ni l'accessibilité IPv6.

Cette observation fournit une référence pour les comparaisons ultérieures. Elle ne prouve ni la présence d'un pare-feu restrictif, ni la sécurité de la configuration SSH.

### 7.5 Configuration SSH effective

Commande exécutée sur le serveur :

```bash
sudo /usr/sbin/sshd -T | grep -E '^(permitrootlogin|passwordauthentication|pubkeyauthentication|kbdinteractiveauthentication|maxauthtries|x11forwarding|allowusers) '
```

Résultat observé :

```text
maxauthtries 6
permitrootlogin without-password
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication no
x11forwarding yes
```

| Paramètre | Valeur initiale | Analyse |
|---|---|---|
| `PermitRootLogin` | `without-password` | La connexion directe de root n'est pas totalement interdite ; une authentification par clé reste possible si une clé autorisée est configurée |
| `PubkeyAuthentication` | `yes` | L'authentification par clé publique est activée |
| `PasswordAuthentication` | `no` | L'authentification SSH par mot de passe est désactivée |
| `KbdInteractiveAuthentication` | `no` | L'authentification interactive clavier est désactivée |
| `MaxAuthTries` | `6` | Six tentatives d'authentification au maximum sont autorisées par connexion |
| `X11Forwarding` | `yes` | Le transfert graphique X11 est autorisé |
| `AllowUsers` | Aucune ligne retournée | Aucune liste restrictive d'utilisateurs n'apparaît dans cette sortie |

La configuration initiale désactive déjà les mots de passe. Le projet devra distinguer ces protections préexistantes des mesures réellement ajoutées.

La commande `sshd -T` donne la configuration effective générale. Des blocs `Match` éventuels peuvent produire des différences selon l'utilisateur ou l'origine de la connexion ; leur présence devra être examinée si nécessaire lors du durcissement.

## 8. Point de restauration

Un snapshot de l'état actuel a été créé avant le durcissement, conformément à la confirmation de l'auteur du projet.

Il permettra de revenir à la situation initiale en cas de problème ou de reproduire certains tests. Le nom exact du snapshot et les VM couvertes devront être reportés depuis Vagrant ou VirtualBox dans la version finale du document.

La restauration du snapshot n'a pas encore été testée. Aucun durcissement SSH n'est considéré comme réalisé à ce stade.

## 9. Bilan des vérifications initiales

| ID | Vérification | Résultat observé | Conclusion |
|---|---|---|---|
| T01 | État Vagrant | `server` et `tester` sont `running` | Les deux VM fonctionnent |
| T02 | Connexion au serveur depuis tester | Nmap signale le serveur accessible | La communication sur le réseau de test fonctionne |
| T03 | Port SSH | `22/tcp open` | Le service SSH est accessible |
| T04 | Scan TCP complet | Un port ouvert et 65 534 ports fermés | SSH est le seul service TCP ouvert observé sur l'adresse testée |
| T05 | Interfaces serveur | NAT et réseau interne présents | L'adresse du serveur correspond au plan réseau |
| T06 | Configuration SSH | Paramètres relevés avec `sshd -T` | Une référence avant durcissement est disponible |
| T07 | Snapshot | Création confirmée par l'auteur | Point de restauration annoncé ; restauration non testée |

### Preuves disponibles

- `image(20261020-115327).png` : état Vagrant des deux VM.
- `image(20261020-115257).png` : identification du service SSH depuis tester.
- `Status_Tester_Ini.png` : scan TCP complet du serveur.
- `Status_Server_Initial.png` : interfaces, services en écoute et paramètres SSH effectifs.

Ces captures constituent les preuves de l'état initial. Elles pourront être insérées comme figures dans la mise en page finale.



## 10 Durcissement SSH

