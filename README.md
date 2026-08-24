# Mirage Enterprise Backup Lab

Infrastructure de lab d’entreprise : ESXi nested, OPNsense, Active Directory, serveur de fichiers, GPO/AGDLP, Veeam Backup & Replication et NAS Debian/XFS avec hardened repository, immutabilité et tests de restauration.

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
- Wallpapers par service via GPO
- Repository Veeam séparé de l’hyperviseur
- XFS + Fast Clone
- Sauvegardes immuables pendant 7 jours
- Single-use credentials
- SSH désactivé après déploiement
- File Level Restore
- Restauration complète de FS01 après suppression de la VM

## Résultat final

Le scénario de reprise après sinistre a été validé :

1. sauvegarde de FS01 vers le repository hardened ;
2. suppression volontaire de la VM FS01 ;
3. restauration complète depuis Veeam ;
4. restauration des partages SMB ;
5. conservation des ACL et des données ;
6. lecteurs réseau utilisateurs de nouveau opérationnels.

## Documentation détaillée

La documentation complète sera disponible dans le dossier [`docs/`](docs/).
