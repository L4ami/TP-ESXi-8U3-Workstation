---
title: Installation ESXi 8.0 U3 sous VMware Workstation
tags: ESXi, VMware, Virtualisation, TP
lang: fr
---

# Installation de VMware ESXi 8.0 Update 3 sous VMware Workstation Pro

:::info
- **Travaux pratiques :** Virtualisation
- **Auteur :** Lami · **Date :** 26 et 27 octobre 2026
- **Objectif :** installer un hyperviseur de type 1 (ESXi 8.0 U3) dans une machine virtuelle VMware Workstation, puis l'administrer de deux façons : depuis l'interface web ESXi Host Client et en ligne de commande via SSH.
:::

---

## Sommaire

[TOC]

---

## 1. Contexte et notions clés

**VMware ESXi** est un **hyperviseur de type 1** (dit *bare-metal*) : il s'installe directement sur le matériel d'un serveur, sans système d'exploitation intermédiaire. Il sert à héberger et faire tourner de nombreuses machines virtuelles en entreprise.

**VMware Workstation** est un **hyperviseur de type 2** : c'est un logiciel qui tourne *par-dessus* Windows.

| | Type 1 (ESXi) | Type 2 (Workstation) |
|---|---|---|
| S'installe sur | le matériel directement | un système d'exploitation (Windows, Linux) |
| Usage | serveurs de production, datacenters | poste de travail, tests, formation |
| Performances | maximales | réduites par l'OS hôte |

Dans ce TP, faute de serveur physique dédié, ESXi est installé **dans une VM Workstation**. On parle de **virtualisation imbriquée** (*nested virtualization*) : un hyperviseur qui tourne à l'intérieur d'un autre hyperviseur. C'est une méthode courante pour se former.

---

## 2. Environnement de travail

| Élément | Valeur |
|---|---|
| Machine hôte | PC portable Lenovo Legion Pro 5 |
| Processeur | Intel Core i9-14900HX (24 cœurs, 32 threads) |
| Mémoire vive | 32 Go |
| Système hôte | Windows 11 Pro 64 bits |
| Hyperviseur de type 2 | VMware Workstation Pro 25H2u1 (25.0.1) |
| Image d'installation | `VMware-VMvisor-Installer-8.0U3-24022510.x86_64.iso` (fournie par le formateur) |

---

## 3. Vérification des prérequis

ESXi 8 exige au minimum **2 cœurs CPU** et **8 Go de RAM**. La VM empruntant ses ressources au PC hôte, on vérifie d'abord que celui-ci peut suivre, et que la **virtualisation matérielle** (Intel VT-x) est activée dans le BIOS.

Gestionnaire des tâches (`Ctrl + Maj + Échap`) → onglet **Performance** → **Processeur** :

![Vérification du processeur, de la RAM et de la virtualisation](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/01-verif-pc-virtualisation.png)

*Capture 01 : 32 Go de RAM, i9-14900HX, **Virtualisation : Activé**.*

Version de Workstation (**Aide → À propos de VMware Workstation**) :

![Version de VMware Workstation](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/02-version-workstation.png)

*Capture 02 : VMware Workstation Pro 25H2u1, compatible avec ESXi 8.*

---

## 4. Création de la machine virtuelle

**Fichier → Nouvelle machine virtuelle → Personnalisé (avancé)**.

### 4.1 Image ISO

L'assistant lit l'ISO et reconnaît automatiquement le système invité, ce qui lui permet de proposer des réglages adaptés.

![ISO ESXi détectée](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/03-iso-esxi-detectee.png)

*Capture 03 : « VMware ESXi 8 détecté ».*

- **Compatibilité matérielle :** `Workstation 25H2 or later` (génération de matériel virtuel la plus récente).
- **Nom :** `ESXi 8` · **Emplacement :** `D:\VM\ESXi 8` (disque secondaire, pour ne pas encombrer le disque système).

### 4.2 Processeurs

![Configuration des processeurs](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/04-vm-processeurs.png)

*Capture 04 : 1 processeur × 4 cœurs = 4 cœurs.*

- ESXi refuse de s'installer avec moins de 2 cœurs. 4 cœurs lui laissent de la marge.
- Un seul *socket* reproduit un serveur d'entrée de gamme réaliste.

### 4.3 Mémoire

![Configuration de la mémoire](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/05-vm-memoire.png)

*Capture 05 : 8192 Mo (8 Go).*

:::warning
Workstation « recommande » 4 Go, mais **ESXi 8 exige 8 Go minimum**. Avec 4 Go, l'installeur s'arrête.
:::

### 4.4 Réseau

![Type de réseau NAT](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/06-vm-reseau-nat.png)

*Capture 06 : mode NAT.*

| Mode | Fonctionnement |
|---|---|
| Pont (*bridged*) | la VM est vue comme une machine à part entière sur le réseau physique |
| **NAT** (retenu) | la VM sort sur Internet via l'adresse IP du PC hôte, et le PC peut la joindre. Simple et fiable, y compris en Wi-Fi |
| Hôte uniquement | la VM ne communique qu'avec le PC hôte |

### 4.5 Contrôleur et disque

![Contrôleur SCSI paravirtualisé](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/07-vm-controleur-pvscsi.png)

*Capture 07 : contrôleur **SCSI paravirtualisé** (PVSCSI), conçu pour les VM et plus performant qu'une carte émulée.*

![Type de disque SCSI](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/08-vm-type-disque-scsi.png)

*Capture 08 : disque de type **SCSI**.*

![Capacité du disque](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/09-vm-disque-142go-thin.png)

*Capture 09 : 142 Go (taille recommandée pour ESXi 8), allocation dynamique.*

:::success
- **Provisionnement dynamique (*thin*)** : le fichier `.vmdk` commence petit et grossit selon les besoins.
- **Provisionnement statique (*thick*)** : tout l'espace est réservé dès la création.

On retrouve ces deux notions dans ESXi lors de la création de VM.
:::

### 4.6 Récapitulatif et moteur de virtualisation

![Récapitulatif de la VM](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/10-vm-recapitulatif.png)

*Capture 10 : récapitulatif avant création.*

Dans **Personnaliser le matériel → Processeurs**, l'option **Virtualiser Intel VT-x/EPT ou AMD-V/RVI** est laissée **décochée** (voir section 9).

![VT-x/EPT décoché](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/11-vm-vtx-decoche.png)

*Capture 11 : moteur de virtualisation, option VT-x/EPT décochée.*

### Synthèse de la VM

| Paramètre | Valeur |
|---|---|
| Système invité | VMware ESXi 8 |
| Processeurs | 1 socket × 4 cœurs |
| Mémoire | 8 Go |
| Disque | 142 Go, SCSI, PVSCSI, provisionnement dynamique |
| Réseau | NAT |
| VT-x/EPT virtualisé | désactivé |

---

## 5. Installation d'ESXi

Mise sous tension de la VM : l'installeur démarre depuis l'ISO.

![Chargement de l'installeur](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/12-esxi-chargement.png)

*Capture 12 : ESXi 8.0.3 détecte le processeur i9-14900HX et 8 Gio de mémoire, puis décompresse ses modules en RAM.*

:::success
Dans Workstation : cliquer dans la VM pour lui donner le clavier, `Ctrl + Alt` pour récupérer la souris. L'installeur se pilote **uniquement au clavier**.
:::

![Écran d'accueil](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/13-esxi-welcome.png)

*Capture 13 : écran d'accueil → `Entrée`, puis licence (EULA) → `F11`.*

![Choix du disque](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/14-esxi-choix-disque.png)

*Capture 14 : sélection du disque virtuel de 142 Gio → `Entrée`.*

![Disposition du clavier](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/15-esxi-clavier.png)

*Capture 15 : disposition **Swiss French** (clavier QWERTZ du poste).*

:::danger
Choisir la bonne disposition de clavier **avant** le mot de passe. Sinon, le mot de passe enregistré ne correspond pas aux touches tapées.
:::

**Mot de passe root** : `root` est le compte administrateur d'ESXi. Le mot de passe doit faire au moins 8 caractères et combiner au moins 3 types (majuscules, minuscules, chiffres, symboles). *Il n'est volontairement pas indiqué dans ce document.*

![Avertissement virtualisation matérielle](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/16-esxi-warning-virtualisation.png)

*Capture 16 : avertissement `HARDWARE_VIRTUALIZATION`, attendu puisque VT-x n'est pas transmis à la VM. Ce n'est pas bloquant → `Entrée`, puis `F11` pour lancer l'installation.*

![Installation terminée](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/17-esxi-installation-terminee.png)

*Capture 17 : installation réussie. ESXi fonctionne en **mode évaluation pendant 60 jours**.*

Avant de redémarrer, on retire le support d'installation (**VM → Paramètres → CD/DVD**). Sinon, la VM pourrait relancer l'installeur.

![ISO déconnectée](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/18-vm-iso-deconnectee.png)

*Capture 18 : cases « Connecté » et « Se connecter lors de la mise sous tension » décochées.*

Puis `Entrée` → **Reboot**.

---

## 6. Premier démarrage et console DCUI

ESXi démarre désormais depuis son disque. On arrive sur la **DCUI** (*Direct Console User Interface*), la console locale du serveur.

![Console DCUI](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/19-esxi-dcui-ip.png)

*Capture 19 : ESXi a obtenu l'adresse **192.168.190.140** par DHCP, via le réseau NAT de Workstation.*

- `F2` : personnaliser le système (réseau, mot de passe, journaux).
- `F12` : arrêter ou redémarrer.

:::info
Après quelques minutes d'inactivité, la DCUI s'assombrit. C'est un simple économiseur d'écran, le serveur continue de tourner.
:::

---

## 7. Accès à l'interface web ESXi Host Client

Depuis le navigateur du PC hôte : `https://192.168.190.140`.

Le navigateur affiche un avertissement de sécurité : ESXi utilise un **certificat auto-signé** (il a signé lui-même son certificat, aucune autorité reconnue ne le garantit). C'est normal en laboratoire.

![Page de connexion Host Client](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/20-host-client-connexion.png)

*Capture 20 : connexion avec le compte `root`.*

![Tableau de bord ESXi](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/21-host-client-tableau-de-bord.png)

*Capture 21 : tableau de bord de l'hôte.*

Lecture du tableau de bord :

| Information | Valeur | Explication |
|---|---|---|
| Version | 8.0 Update 3 | build 24022510 |
| État | Normal (non connecté à vCenter Server) | **vCenter** gère plusieurs ESXi de façon centralisée. Ici, l'hôte est géré seul |
| CPU | 4 × Intel Core i9-14900HX | ressources données à la VM |
| Mémoire | 8 Go | |
| Stockage (datastore) | 13,75 Go | sur 142 Go : ESXi 8 réserve environ 128 Go à sa partition système **OSDATA**, le reste devient la banque de données |
| Licence | évaluation, 60 jours | |

---

## 8. Connexion en SSH

**SSH** (*Secure Shell*) permet d'administrer un serveur à distance **en ligne de commande**, à travers une connexion **chiffrée**. Là où le Host Client se pilote à la souris, SSH donne accès au shell d'ESXi et à ses outils (`esxcli`, `vim-cmd`…), indispensables pour l'automatisation et le dépannage.

:::danger
Sur ESXi, le service SSH est **désactivé par défaut** pour des raisons de sécurité. On l'active uniquement le temps de l'intervention.
:::

### 8.1 Activer le service SSH

Dans le Host Client : **Hôte → Gérer → onglet Services**, rechercher `ssh`, sélectionner **TSM-SSH** puis **Démarrer**.

![Service TSM-SSH démarré](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/22-esxi-service-ssh-demarre.png)

*Capture 22 : TSM-SSH « En cours d'exécution » (TSM = *Tech Support Mode*), tâche « Start Service » terminée.*

### 8.2 Se connecter depuis Windows

Windows 11 intègre un client SSH (OpenSSH). Dans une invite de commandes :

```
ssh root@192.168.190.140
```

`root@192.168.190.140` signifie : se connecter avec le compte **root** sur la machine **192.168.190.140**.

![Empreinte à la première connexion](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/23-ssh-empreinte-premiere-connexion.png)

*Capture 23 : à la première connexion, le client affiche l'**empreinte** (fingerprint) de la clé du serveur.*

:::success
L'empreinte est l'identité unique du serveur. En répondant `yes`, le PC l'enregistre dans son fichier `known_hosts`. Si un autre serveur tentait ensuite de se faire passer pour l'ESXi, l'empreinte différerait et SSH bloquerait la connexion (protection contre l'attaque de l'homme du milieu).
:::

:::info
Lors du premier essai, la question est restée plusieurs minutes sans réponse : ESXi a fermé la connexion (« Connection closed »), car il limite le temps accordé pour s'authentifier. Une nouvelle tentative a suffi, l'empreinte étant déjà enregistrée.
:::

Le mot de passe saisi ne s'affiche pas à l'écran (pas même d'astérisques), c'est normal.

![Connexion SSH réussie](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/24-ssh-connexion-reussie.png)

*Capture 24 : connexion établie. L'invite `[root@localhost:~]` indique que les commandes s'exécutent désormais sur l'ESXi. ESXi rappelle que toutes les commandes sont journalisées.*

### 8.3 Commandes de vérification

```
vmware -vl
esxcli network ip interface ipv4 get
```

![Commandes vmware et esxcli](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/25-ssh-commandes-esxcli.png)

*Capture 25 : résultat des commandes.*

| Commande | Résultat | Explication |
|---|---|---|
| `vmware -vl` | VMware ESXi 8.0 Update 3, build 24022510 | version exacte de l'hyperviseur |
| `esxcli network ip interface ipv4 get` | vmk0 · 192.168.190.140 · 255.255.255.0 · passerelle 192.168.190.2 · DHCP | **vmk0** est l'interface d'administration (VMkernel) d'ESXi. La passerelle 192.168.190.2 est le routeur NAT de Workstation |

On quitte la session avec la commande `exit`.

### 8.4 Désactiver SSH après utilisation

Retour dans **Hôte → Gérer → Services** : sélectionner **TSM-SSH** puis **Arrêter**.

![Service TSM-SSH arrêté](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/26-esxi-service-ssh-arrete.png)

*Capture 26 : TSM-SSH « Arrêté », tâche « Stop Service » terminée.*

:::warning
**Bonne pratique de sécurité :** ne laisser SSH actif que pendant l'intervention. Un service SSH ouvert en permanence est une porte d'entrée supplémentaire pour un attaquant.
:::

---

## 9. Problème rencontré et résolution

### Symptôme

Au premier démarrage, avec l'option **Virtualiser Intel VT-x/EPT** cochée, Workstation affiche :

![Erreur VT-x non pris en charge](https://raw.githubusercontent.com/L4ami/TP-ESXi-8U3-Workstation/main/captures/annexe-erreur-vtx-non-supporte.png)

> *« Intel VT-x/EPT virtualisé n'est pas pris en charge sur cette plate-forme. »*

Répondre « Non » empêche la VM de démarrer.

### Cause

Sur Windows 11, la **Sécurité basée sur la virtualisation (VBS)** et l'**Intégrité de la mémoire** sont actives par défaut. Elles reposent sur l'hyperviseur Hyper-V de Microsoft, qui se réserve les instructions VT-x. Workstation fonctionne alors au-dessus d'Hyper-V et ne peut plus transmettre VT-x à la VM.

### Solution retenue (consigne du formateur)

**Décocher** l'option *Virtualiser Intel VT-x/EPT* dans les paramètres processeur de la VM. La VM démarre, et l'installeur affiche simplement l'avertissement `HARDWARE_VIRTUALIZATION` (capture 16), qui n'est pas bloquant.

### Pistes écartées

Désactiver les protections de Windows (`bcdedit /set hypervisorlaunchtype off`, Intégrité de la mémoire, sécurité de connexion renforcée de Windows Hello) aurait permis de garder VT-x. Cette piste a été testée puis **annulée**, car elle réduit la sécurité du poste et n'est pas nécessaire pour ce TP. Toutes les protections ont été réactivées.

### Limite connue

Sans VT-x virtualisé, ESXi est installé et administrable, mais **il ne peut pas faire tourner de machines virtuelles 64 bits** à l'intérieur de lui-même. Pour aller plus loin (VM imbriquées), il faudrait un serveur physique ou un hôte sans VBS.

---

## 10. Bilan

- ✅ VM Workstation dimensionnée selon les exigences d'ESXi 8 (4 cœurs, 8 Go, 142 Go)
- ✅ ESXi 8.0 Update 3 installé et démarré
- ✅ Adresse IP obtenue par DHCP (192.168.190.140)
- ✅ Administration à distance via ESXi Host Client
- ✅ Connexion en ligne de commande via SSH, puis désactivation du service
- ✅ Problème de virtualisation imbriquée diagnostiqué et contourné

**Compétences mises en œuvre :** différence entre hyperviseurs de type 1 et 2, dimensionnement d'une VM, modes réseau de Workstation, provisionnement thin/thick, installation et premier accès à un hôte ESXi, administration en SSH et commandes `esxcli`, diagnostic d'un conflit entre Hyper-V et Workstation.

---

## 11. Glossaire

| Terme | Définition |
|---|---|
| **Hyperviseur** | logiciel qui crée et fait tourner des machines virtuelles |
| **ESXi** | hyperviseur de type 1 de VMware, installé directement sur un serveur |
| **VT-x / EPT** | instructions de virtualisation des processeurs Intel |
| **Virtualisation imbriquée** | faire tourner un hyperviseur dans une VM |
| **VBS** | *Virtualization-Based Security*, protections de Windows qui utilisent Hyper-V |
| **NAT** | *Network Address Translation*, partage de l'adresse IP de l'hôte avec la VM |
| **DHCP** | protocole qui attribue automatiquement une adresse IP |
| **PVSCSI** | contrôleur de disque paravirtualisé de VMware |
| **Thin / Thick** | provisionnement dynamique / statique d'un disque virtuel |
| **DCUI** | *Direct Console User Interface*, console locale d'ESXi |
| **Host Client** | interface web d'administration d'un hôte ESXi |
| **vCenter Server** | outil de gestion centralisée de plusieurs hôtes ESXi |
| **SSH** | *Secure Shell*, protocole d'administration à distance en ligne de commande, chiffré |
| **Empreinte (fingerprint)** | identifiant unique de la clé d'un serveur SSH, vérifié à chaque connexion |
| **esxcli** | outil en ligne de commande pour configurer et interroger ESXi |
| **vmk0** | première interface réseau VMkernel d'ESXi, utilisée pour l'administration |
| **Datastore** | espace de stockage où ESXi range les fichiers des VM |
| **OSDATA** | partition système d'ESXi 8 (journaux, outils, données internes) |
