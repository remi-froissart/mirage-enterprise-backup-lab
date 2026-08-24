# 05 - Veeam Backup et restauration

## Objectif

Cette partie du lab consiste à mettre en œuvre la sauvegarde des machines virtuelles hébergées sur ESXI01 avec **Veeam Backup & Replication**.

Après avoir préparé le Linux Hardened Repository sur NAS01, l'objectif est maintenant de valider toute la chaîne de sauvegarde :

- connecter Veeam à l'hyperviseur ESXi ;
- détecter les machines virtuelles ;
- utiliser le repository `NAS01-Hardened` ;
- créer un job de sauvegarde pour FS01 ;
- réaliser une sauvegarde complète ;
- vérifier le fonctionnement du Changed Block Tracking ;
- supprimer volontairement un fichier métier ;
- restaurer ce fichier avec File Level Restore ;
- vérifier son retour sur FS01 ;
- tester concrètement l'immutabilité du backup.

---

## Serveur VEEAM01

Veeam Backup & Replication est installé sur un serveur Windows Server dédié.

| Paramètre | Valeur |
|---|---|
| Nom | VEEAM01 |
| Système | Windows Server 2022 |
| Adresse IP | 192.168.100.120 |
| Domaine | WORKGROUP |
| Rôle | Serveur de sauvegarde |
| Logiciel | Veeam Backup & Replication Community Edition |

VEEAM01 est exécuté directement dans VMware Workstation Pro et n'est pas hébergé dans ESXI01.

Il reste ainsi indépendant des machines virtuelles protégées hébergées sur l'hyperviseur ESXi.

---

## Chaîne de sauvegarde

Le fonctionnement général peut être représenté ainsi :

```text
FS01 sur ESXI01
       │
       ▼
Veeam Backup & Replication
       │
       │ Orchestration du job
       ▼
NAS01-Hardened
       │
       ▼
/data/veeam
       │
       ▼
XFS
       │
       ▼
Backup immutable
```

VEEAM01 assure donc l'orchestration des opérations de sauvegarde et de restauration tandis que les fichiers de sauvegarde sont stockés sur NAS01.

---

## Ajout d'ESXI01 dans Veeam

L'hyperviseur a été ajouté dans l'infrastructure Veeam avec son adresse de management :

```text
192.168.110.10
```

Après enregistrement, Veeam détecte les machines virtuelles présentes sur l'hyperviseur :

```text
ESXI01
│
├── FW01
├── DC01
├── FS01
└── CLT-W10-01
```

### Validation de l'inventaire ESXi

Après l'ajout de l'hyperviseur, les machines virtuelles hébergées sur ESXI01 sont visibles directement dans l'inventaire Veeam :

![Inventaire ESXi dans Veeam](../images/veeam-backup-restore/01-esxi-inventory.png)

Veeam identifie correctement `FW01`, `DC01`, `FS01` et `CLT-W10-01` ainsi que leurs systèmes invités.

L'hyperviseur est donc correctement intégré à l'infrastructure de sauvegarde.

Veeam peut ainsi sélectionner directement les machines virtuelles à protéger.

---

## Repository de destination

Le job utilise le Linux Hardened Repository préparé dans la partie précédente :

```text
NAS01-Hardened
└── /data/veeam
```

Il repose sur un volume XFS avec Fast Clone et une période d'immutabilité de 7 jours.

Les fichiers de sauvegarde sont ainsi stockés en dehors d'ESXI01.

---

## Création du job FS01

Un premier job a été créé afin de protéger le serveur de fichiers :

```text
FS01
```

FS01 est particulièrement intéressant pour ce test car il contient les données métier utilisées précédemment :

```text
D:\Partages
│
├── Direction
├── RH
├── Ventes
└── Informatique
```

Le job utilise comme destination :

```text
NAS01-Hardened
```

La chaîne logique est donc :

```text
FS01
   ↓
Veeam Backup & Replication
   ↓
NAS01-Hardened
   ↓
/data/veeam
```

---

## Première sauvegarde de FS01

Le job a été exécuté afin de créer une première sauvegarde complète de la machine virtuelle.

L'exécution s'est terminée avec le statut :

```text
Success
```

Les statistiques observées lors de cette sauvegarde étaient approximativement :

| Indicateur | Valeur |
|---|---:|
| Données traitées | 14,7 Go |
| Données lues | 11 Go |
| Données transférées | 6,7 Go |
| Ratio | 1,6x |
| Débit de traitement | 153 MB/s |
| Durée | environ 2 min 39 s |
| Résultat | Success |

La quantité de données transférée vers le repository est inférieure au volume traité grâce aux mécanismes de réduction de données utilisés par Veeam pendant la sauvegarde.

### Validation du job de sauvegarde

L'exécution du job `Backup FS01` s'est terminée avec succès :

![Sauvegarde réussie de FS01](../images/veeam-backup-restore/02-fs01-backup-success.png)

La capture confirme également l'activation du **Changed Block Tracking** et le traitement réussi de la VM FS01.

Le repository utilisé est `NAS01-Hardened`.

---

## Changed Block Tracking

Veeam utilise le **Changed Block Tracking**, ou CBT, proposé par VMware.

Le principe est de permettre à l'hyperviseur d'indiquer quels blocs d'une machine virtuelle ont changé depuis la sauvegarde précédente.

Sans CBT :

```text
Disque virtuel
      ↓
Analyse d'une grande partie du disque
      ↓
Recherche des données modifiées
```

Avec CBT :

```text
Disque virtuel
      ↓
Liste des blocs modifiés
      ↓
Lecture ciblée
      ↓
Backup incrémental
```

Cela permet de réduire les lectures inutiles lors des sauvegardes suivantes.

Le job de sauvegarde de FS01 utilise donc le Changed Block Tracking fourni par VMware.

---

## Validation du backup

Après exécution du job, FS01 apparaît dans les sauvegardes disponibles.

Le restore point permet alors différentes opérations :

```text
Backup FS01
│
├── Entire VM Restore
├── Virtual Disk Restore
├── File Level Restore
└── autres modes de restauration
```

Pour cette première validation, le choix a été fait de tester une **restauration granulaire d'un fichier**.

---

## Scénario de File Level Restore

Le fichier choisi pour le test se trouve sur le partage RH :

```text
\\FS01\RH\employes.xlsx
```

Sur FS01, son emplacement réel est :

```text
D:\Partages\RH\employes.xlsx
```

L'objectif est de simuler une suppression accidentelle réalisée par un utilisateur.

---

## Suppression volontaire du fichier

Avant la suppression, le partage RH contient notamment :

```text
RH
├── employes.xlsx
└── Procédures.txt
```

Le fichier :

```text
employes.xlsx
```

a ensuite été volontairement supprimé.

Le partage ne contient alors plus que :

```text
RH
└── Procédures.txt
```

Cette situation représente un cas classique de restauration :

```text
Utilisateur
    ↓
Suppression accidentelle
    ↓
Fichier absent de FS01
    ↓
Besoin de restauration
```

---

## File Level Restore

Depuis Veeam Backup & Replication, le restore point de FS01 a été ouvert avec le mode de restauration de fichiers Windows.

Veeam monte temporairement le contenu de la sauvegarde et ouvre le **Veeam Backup Browser**.

L'arborescence sauvegardée peut alors être parcourue jusqu'à :

```text
D:
└── Partages
    └── RH
        ├── employes.xlsx
        └── Procédures.txt
```

Le fichier supprimé du serveur de production est donc toujours présent dans le restore point.

---

## Restauration vers l'emplacement d'origine

Le fichier :

```text
employes.xlsx
```

a été sélectionné dans Veeam Backup Browser.

La restauration a été effectuée vers son emplacement d'origine :

```text
D:\Partages\RH\employes.xlsx
```

Veeam a utilisé les mécanismes d'interaction avec le système invité pour replacer le fichier sur FS01.

Lors de cette opération, des identifiants disposant des droits nécessaires sur FS01 ont été fournis à Veeam.

---

## Résultat du File Level Restore

La restauration s'est terminée avec succès.

![Restauration granulaire du fichier employes.xlsx](../images/veeam-backup-restore/03-file-level-restore.png)

Veeam confirme :

| Paramètre | Valeur |
|---|---|
| Fichiers restaurés | 1 |
| Taille | environ 8,1 KB |
| Durée | environ 16 secondes |
| Erreurs | 0 |
| Destination | `D:\Partages\RH\employes.xlsx` |

La restauration a été réalisée via la **vSphere Guest Interaction API**.

Le fichier `employes.xlsx` a donc été replacé à son emplacement d'origine sur FS01.

---

## Validation côté utilisateur

Le partage RH peut ensuite être contrôlé depuis un poste client :

```text
\\FS01\RH
```

ou par le lecteur réseau automatiquement mappé :

```text
S:
```

Le fichier :

```text
employes.xlsx
```

est de nouveau disponible.

La chaîne complète de restauration est donc validée :

```text
Suppression
    ↓
Fichier absent de FS01
    ↓
Sélection du restore point
    ↓
Veeam Backup Browser
    ↓
Restauration
    ↓
FS01
    ↓
\\FS01\RH\employes.xlsx
    ↓
Fichier de nouveau accessible
```

---

## Test de l'immutabilité

Après avoir validé le File Level Restore, l'immutabilité configurée sur `NAS01-Hardened` a également été testée.

Le repository utilise une période de :

```text
7 jours d'immutabilité
```

L'objectif est de vérifier qu'un backup encore protégé ne peut pas être supprimé prématurément.

Une suppression physique du backup a été demandée depuis Veeam Backup & Replication avec l'action :

```text
Delete from disk
```

![Test de l'immutabilité du backup FS01](../images/veeam-backup-restore/04-immutability-test.png)

Veeam refuse la suppression et retourne notamment :

```text
Unable to delete 1 immutable backup file
0 deleted
```

La session précise également la date à partir de laquelle le fichier pourra être supprimé.

Cette vérification confirme que l'immutabilité configurée sur `NAS01-Hardened` est réellement appliquée aux fichiers de sauvegarde.

---

## Intérêt de cette protection

Sans immutabilité :

```text
Compte Veeam compromis
        ↓
Suppression des backups
        ↓
Perte potentielle des possibilités de restauration
```

Avec le Hardened Repository :

```text
Compte Veeam compromis
        ↓
Tentative de suppression
        ↓
Backup immutable
        ↓
Suppression refusée
        ↓
Restore point conservé
```

L'immutabilité introduit donc une protection supplémentaire entre le serveur de sauvegarde et les fichiers stockés sur NAS01.

Elle ne remplace pas une stratégie de sauvegarde complète, mais réduit fortement le risque qu'une suppression logique depuis Veeam détruise immédiatement les restore points protégés.

---

## Validation de la chaîne complète

À ce stade, plusieurs mécanismes ont été testés concrètement.

```text
                  ESXI01
                     │
                     ▼
                    FS01
                     │
                     │ Backup
                     ▼
                   VEEAM01
                     │
                     ▼
               NAS01-Hardened
                     │
                     ▼
                  XFS /data
                     │
                     ▼
               Backup immutable
                     │
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
 File Level Restore      Suppression backup
          │                     │
          ▼                     ▼
 employes.xlsx          Refusée pendant
 restauré               l'immutabilité
```

La chaîne n'a donc pas uniquement été configurée : elle a été testée à la fois en **sauvegarde**, en **restauration** et en **protection contre la suppression**.

---

## Validation finale

À l'issue de cette partie :

- VEEAM01 fonctionne comme serveur de sauvegarde indépendant d'ESXI01 ;
- VEEAM01 communique avec l'interface de management d'ESXI01 ;
- les machines virtuelles de l'hyperviseur sont visibles dans Veeam ;
- `NAS01-Hardened` est utilisé comme destination ;
- FS01 dispose d'un job de sauvegarde dédié ;
- une sauvegarde complète de FS01 a été réalisée avec succès ;
- le Changed Block Tracking est actif ;
- les données sont stockées sur le repository XFS de NAS01 ;
- le fichier `employes.xlsx` a été volontairement supprimé ;
- le fichier était toujours disponible dans le restore point ;
- un File Level Restore a été effectué ;
- le fichier a été restauré à son emplacement d'origine ;
- la restauration s'est terminée sans erreur ;
- le fichier est redevenu accessible depuis le partage RH ;
- une tentative de suppression d'un backup immutable a été réalisée ;
- Veeam a refusé la suppression avant la fin de la période de protection.

La sauvegarde, la restauration granulaire et l'immutabilité du repository sont donc opérationnelles.

La prochaine étape consiste à aller plus loin en simulant non plus la perte d'un fichier, mais la **perte complète du serveur FS01**.

---

## Suite

La prochaine partie simule un scénario de sinistre avec suppression complète de FS01 puis restauration de l'intégralité de la machine virtuelle :

➡️ [06 - Disaster Recovery](06-disaster-recovery.md)
