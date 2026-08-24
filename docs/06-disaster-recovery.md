# 06 - Disaster Recovery

## Objectif

Cette dernière partie du lab consiste à valider un scénario de **Disaster Recovery** sur le serveur de fichiers FS01.

Les étapes précédentes ont permis de vérifier :

- la sauvegarde de FS01 avec Veeam ;
- le stockage des backups sur `NAS01-Hardened` ;
- la restauration granulaire d'un fichier ;
- l'immutabilité des sauvegardes.

L'objectif est maintenant de simuler un incident beaucoup plus important :

```text
Perte complète de FS01
```

puis de vérifier qu'il est possible de restaurer l'intégralité de la machine virtuelle à partir du backup.

Le scénario doit permettre de retrouver :

- la machine virtuelle FS01 ;
- son système Windows Server 2022 Core ;
- sa configuration réseau ;
- son appartenance au domaine ;
- son disque de données ;
- les partages SMB ;
- les permissions NTFS ;
- les données métier ;
- le fonctionnement des lecteurs réseau côté utilisateur.

---

# Scénario de sinistre

Le serveur concerné est :

| Paramètre | Valeur |
|---|---|
| Serveur | FS01 |
| Système | Windows Server 2022 Core |
| Adresse IP | 10.10.10.20 |
| Domaine | ad.mirage-lab.cloud |
| Rôle | Serveur de fichiers |
| Hyperviseur | ESXI01 |
| Repository de sauvegarde | NAS01-Hardened |

FS01 contient les ressources utilisées dans le reste du lab :

```text
D:\Partages
│
├── Direction
├── RH
├── Ventes
├── Informatique
└── Ressources
```

La perte de cette machine entraîne donc l'indisponibilité des différents partages métier.

---

# Situation avant le sinistre

Avant de simuler la perte de FS01, une sauvegarde valide de la machine est disponible dans Veeam.

La chaîne de protection est :

```text
FS01
   │
   │ Backup
   ▼
ESXI01
   │
   ▼
VEEAM01
   │
   ▼
NAS01-Hardened
   │
   ▼
/data/veeam
   │
   ▼
Restore point FS01
```

Le repository étant hébergé en dehors d'ESXI01, la sauvegarde reste disponible même si la machine virtuelle de production disparaît.

---

# Simulation de la perte de FS01

La machine virtuelle FS01 a été volontairement supprimée de l'environnement ESXi afin de simuler une perte complète du serveur.

La situation devient :

```text
ESXI01
│
├── FW01
├── DC01
├── CLT-W10-01
└── FS01
    ✗
```

Les autres composants de l'infrastructure restent disponibles :

```text
DC01
✓

VEEAM01
✓

NAS01-Hardened
✓
```

La sauvegarde de FS01 n'étant pas stockée sur ESXI01, elle reste exploitable depuis Veeam.

---

# Impact de la perte du serveur

Sans FS01, les ressources suivantes deviennent indisponibles :

```text
\\FS01\Direction
\\FS01\RH
\\FS01\Ventes
\\FS01\Informatique
```

Les lecteurs réseau utilisateurs qui pointent vers ces partages ne peuvent donc plus accéder aux données.

La logique du sinistre est :

```text
Utilisateur
    │
    ▼
Lecteur S:
    │
    ▼
\\FS01\Service
    │
    ▼
FS01 indisponible
    │
    ▼
Accès aux données impossible
```

L'objectif du Disaster Recovery est de rétablir cette chaîne.

---

# Choix du mode de restauration

Dans Veeam Backup & Replication, le backup de FS01 reste disponible malgré l'absence de la machine virtuelle originale.

Pour ce scénario, le mode utilisé est :

```text
Entire VM Restore
```

Contrairement au File Level Restore utilisé précédemment, cette opération ne restaure pas un fichier isolé.

Elle permet de restaurer la **machine virtuelle complète** à partir du restore point.

La logique devient :

```text
Restore point FS01
        │
        ▼
Entire VM Restore
        │
        ▼
ESXI01
        │
        ▼
FS01 restauré
```

---

# Restauration de la machine virtuelle

Le restore point de FS01 a été sélectionné dans Veeam.

L'objectif est de replacer la machine virtuelle dans l'environnement ESXi afin de retrouver le serveur tel qu'il était au moment de la sauvegarde.

La source de restauration est :

```text
NAS01-Hardened
```

avec les fichiers de sauvegarde stockés sous :

```text
/data/veeam
```

La destination est l'hyperviseur :

```text
ESXI01
```

Veeam utilise les données présentes dans le repository afin de reconstruire la machine virtuelle.

---

# Chaîne de restauration

Le flux peut être représenté ainsi :

```text
NAS01-Hardened
192.168.100.130
        │
        │ Backup FS01
        ▼
VEEAM01
192.168.100.120
        │
        │ Orchestration de la restauration
        ▼
ESXI01
192.168.110.10
        │
        ▼
FS01
10.10.10.20
```

Cette architecture permet de restaurer une machine hébergée sur ESXI01 à partir d'une infrastructure de sauvegarde indépendante.

---

# Retour de FS01 dans ESXi

Une fois la restauration terminée, FS01 est de nouveau présent dans l'inventaire de l'hyperviseur.

La machine restaurée retrouve notamment :

```text
FS01
├── Windows Server 2022 Core
├── configuration système
├── configuration réseau
└── disque de données
```

Le serveur peut alors être redémarré et les services vérifiés.

---

# Vérification des ressources SMB

La première validation consiste à vérifier que les partages réseau sont de nouveau accessibles.

Depuis un système disposant d'un accès au domaine, les chemins suivants ont été testés :

```powershell
Test-Path \\FS01\Direction
Test-Path \\FS01\RH
Test-Path \\FS01\Ventes
Test-Path \\FS01\Informatique
```

Les quatre commandes retournent :

```text
True
```

Les partages principaux du serveur sont donc de nouveau disponibles.

La vérification donne :

| Partage | Résultat |
|---|---|
| `\\FS01\Direction` | True |
| `\\FS01\RH` | True |
| `\\FS01\Ventes` | True |
| `\\FS01\Informatique` | True |

---

# Vérification des données

La restauration de la machine virtuelle doit également permettre de retrouver le contenu du disque de données.

Par exemple, le dossier RH contient de nouveau :

```text
RH
├── employes.xlsx
└── Procédures.txt
```

Les autres espaces métier sont également disponibles :

```text
Direction
RH
Ventes
Informatique
```

La restauration ne permet donc pas seulement de récupérer le système d'exploitation : elle restitue également le disque contenant les données du serveur de fichiers.

---

# Validation depuis une session utilisateur

Une validation supplémentaire a été réalisée depuis une session utilisateur du domaine appartenant au service RH.

Le lecteur réseau :

```text
S:
```

est de nouveau disponible.

Pour cette session, il pointe vers :

```text
\\FS01\RH
```

L'utilisateur retrouve notamment :

```text
employes.xlsx
Procédures.txt
```

Cette vérification est importante car elle valide toute la chaîne et pas uniquement le démarrage de la machine virtuelle.

---

# Chaîne fonctionnelle après restauration

Après le Disaster Recovery, la chaîne complète est de nouveau opérationnelle :

```text
Utilisateur RH
      │
      ▼
Active Directory
      │
      ▼
GG_RH
      │
      ▼
DL_RH_Modification
      │
      ▼
GPO lecteur réseau
      │
      ▼
S:
      │
      ▼
\\FS01\RH
      │
      ▼
D:\Partages\RH
      │
      ▼
employes.xlsx
Procédures.txt
```

Les mécanismes configurés dans les étapes précédentes continuent donc de fonctionner après la restauration complète de FS01.

---

# Du File Level Restore au Disaster Recovery

Deux niveaux de restauration ont maintenant été testés dans le lab.

| Scénario | Solution utilisée |
|---|---|
| Suppression d'un fichier | File Level Restore |
| Perte complète de FS01 | Entire VM Restore |

Dans le premier scénario :

```text
FS01 fonctionne
      │
      └── employes.xlsx supprimé
                │
                ▼
        File Level Restore
```

Dans le second :

```text
FS01 perdu
     │
     ▼
Entire VM Restore
     │
     ▼
FS01 complet restauré
```

Cela permet de choisir une méthode de restauration proportionnée à l'incident rencontré.

---

# Importance de la séparation des sauvegardes

Ce test valide également le choix architectural réalisé au début du projet.

VEEAM01 et NAS01 sont hébergés en dehors d'ESXI01 :

```text
VMware Workstation Pro
│
├── ESXI01
│   ├── FW01
│   ├── DC01
│   ├── FS01
│   └── CLT-W10-01
│
├── VEEAM01
│
└── NAS01
```

Si les sauvegardes avaient été stockées uniquement dans l'environnement ESXi protégé, une perte de celui-ci aurait pu rendre simultanément indisponibles :

```text
Production
+
Sauvegardes
```

Avec l'architecture retenue :

```text
Production ESXi
      ✗

Infrastructure Veeam
      ✓

Hardened Repository
      ✓
```

La restauration reste possible.

---

# Du backup au PRA

Le projet ne valide donc pas uniquement la création d'un fichier de sauvegarde.

La chaîne testée est :

```text
Infrastructure de production
          │
          ▼
      Backup FS01
          │
          ▼
Linux Hardened Repository
          │
          ▼
     Immutabilité
          │
          ▼
Simulation d'un sinistre
          │
          ▼
Entire VM Restore
          │
          ▼
Retour de FS01
          │
          ▼
Validation des partages
          │
          ▼
Validation utilisateur
```

Le backup devient réellement utile lorsqu'il est possible de démontrer qu'il permet de remettre un service en fonctionnement après un incident.

---

# Validation finale

À l'issue du scénario de Disaster Recovery :

- une sauvegarde complète de FS01 était disponible sur `NAS01-Hardened` ;
- FS01 a été volontairement supprimé afin de simuler une perte complète ;
- VEEAM01 est resté disponible ;
- NAS01 et les backups sont restés disponibles ;
- le restore point de FS01 a pu être utilisé ;
- une restauration complète de la VM a été effectuée avec `Entire VM Restore` ;
- FS01 est revenu dans l'environnement ESXi ;
- le serveur a retrouvé son disque de données ;
- `\\FS01\Direction` est accessible ;
- `\\FS01\RH` est accessible ;
- `\\FS01\Ventes` est accessible ;
- `\\FS01\Informatique` est accessible ;
- les données métier sont présentes ;
- le lecteur réseau `S:` fonctionne de nouveau côté utilisateur ;
- le contenu du partage RH est de nouveau accessible.

Le scénario démontre donc qu'une perte complète du serveur de fichiers peut être suivie d'une restauration depuis l'infrastructure Veeam séparée de l'hyperviseur protégé.

---

# Conclusion du lab

L'ensemble du projet permet de valider une chaîne complète d'administration et de protection d'une petite infrastructure d'entreprise :

```text
Virtualisation
     │
     ▼
Segmentation réseau
     │
     ▼
Active Directory
     │
     ▼
AGDLP
     │
     ▼
Serveur de fichiers
     │
     ▼
Permissions NTFS / SMB
     │
     ▼
GPO
     │
     ▼
Veeam Backup
     │
     ▼
Linux Hardened Repository
     │
     ▼
Immutabilité
     │
     ▼
File Level Restore
     │
     ▼
Disaster Recovery
```

Le lab ne se limite donc pas au déploiement de services : les mécanismes mis en place ont été **testés jusqu'à la restauration réelle des données et d'une machine virtuelle complète**.

Cela permet de couvrir à la fois l'administration systèmes, l'infrastructure réseau, Active Directory, la gestion des permissions, la sauvegarde, le durcissement et la restauration après sinistre.

---

## Retour

⬅️ [05 - Veeam Backup et restauration](05-veeam-backup-restore.md)
