# Mirage Enterprise Backup Lab

Infrastructure de lab d’entreprise : ESXi nested, OPNsense, Active Directory, serveur de fichiers, GPO/AGDLP, Veeam Backup & Replication et NAS Debian/XFS avec **Linux Hardened Repository**, immutabilité et tests de restauration.

## Objectif du projet

Construire une petite infrastructure d’entreprise réaliste afin de mettre en pratique :

- la virtualisation avec VMware ESXi ;
- la segmentation réseau avec OPNsense ;
- Active Directory, DNS et les GPO ;
- le modèle AGDLP et les permissions SMB/NTFS ;
- un serveur de fichiers Windows Server Core ;
- une stratégie de sauvegarde séparée de l’infrastructure de production ;
- Veeam Backup & Replication ;
- un Linux Hardened Repository sous Debian/XFS ;
- l’immutabilité des sauvegardes ;
- la restauration granulaire et la restauration complète après sinistre.

## Architecture

![Architecture du lab](images/architecture/mirage-enterprise-backup-lab.png)

## Composants principaux

| Composant | Rôle | Adresse |
|---|---|---|
| ESXI01 | Hyperviseur VMware ESXi 8 | 192.168.110.10 |
| FW01 | Firewall / routage OPNsense | LAN 10.10.10.1 |
| DC01 | Active Directory / DNS | 10.10.10.10 |
| FS01 | Serveur de fichiers Windows Server 2022 Core | 10.10.10.20 |
| CLT-W10-01 | Poste client Windows 10 | 10.10.10.50 |
| VEEAM01 | Veeam Backup & Replication | 192.168.100.120 |
| NAS01 | Debian 13 / XFS Hardened Repository | 192.168.100.130 |

## Fonctions mises en œuvre

- Active Directory avec OU structurées
- Modèle AGDLP
- ACL NTFS et permissions SMB
- Lecteurs réseau mappés par GPO
- Fonds d'écran par service via GPO
- Repository Veeam séparé de l’hyperviseur
- XFS + Fast Clone
- Sauvegardes immuables pendant 7 jours
- Single-use credentials
- SSH désactivé après déploiement
- File Level Restore
- Restauration complète de FS01 après suppression de la VM

## Résultat final

L'infrastructure a été validée jusqu'au scénario de reprise après sinistre : après suppression volontaire de FS01, la machine virtuelle a été restaurée intégralement depuis le Linux Hardened Repository `NAS01-Hardened`.

Les partages SMB, les ACL, les données et les lecteurs réseau utilisateurs étaient de nouveau opérationnels après restauration.

## Scénarios validés

Le lab ne se limite pas au déploiement de l'infrastructure. Plusieurs scénarios ont été testés de bout en bout :

- contrôle des accès aux partages SMB selon le modèle AGDLP ;
- attribution automatique des lecteurs réseau par GPO ;
- sauvegarde complète d'une machine virtuelle avec Veeam ;
- restauration granulaire d'un fichier supprimé ;
- tentative de suppression d'un backup immutable ;
- suppression complète du serveur de fichiers FS01 ;
- restauration complète de la VM avec `Entire VM Restore` ;
- validation des partages et des données depuis une session utilisateur après restauration.

## Limites du lab

Cette infrastructure est un environnement de laboratoire exécuté sur un unique hôte physique.

La stratégie de sauvegarde ne respecte donc pas intégralement la règle **3-2-1**, qui recommande :

- 3 copies des données au total : les données de production et 2 sauvegardes ;
- 2 supports de stockage distincts ;
- 1 copie conservée hors site.

Dans ce lab, VEEAM01 et NAS01 sont volontairement séparés de l'hyperviseur ESXI01, ce qui permet de conserver l'infrastructure de sauvegarde disponible lors de la perte d'une VM hébergée sur ESXi.

Cependant, ESXI01, VEEAM01 et NAS01 restent hébergés sur le même ordinateur physique via VMware Workstation Pro. Une défaillance complète de cet hôte pourrait donc affecter simultanément la production et les sauvegardes.

Dans un environnement de production, cette architecture serait complétée par une copie supplémentaire sur un stockage physiquement distinct et/ou hors site, par exemple via un second repository, un site distant ou du stockage objet compatible avec l'immutabilité.

## Documentation détaillée

- [01 - Architecture réseau](docs/01-architecture-reseau.md)
- [02 - Active Directory et AGDLP](docs/02-active-directory-agdlp.md)
- [03 - Serveur de fichiers et GPO](docs/03-serveur-fichiers-gpo.md)
- [04 - NAS XFS et Hardened Repository](docs/04-nas-xfs-hardened-repository.md)
- [05 - Veeam Backup et restauration](docs/05-veeam-backup-restore.md)
- [06 - Disaster Recovery](docs/06-disaster-recovery.md)
