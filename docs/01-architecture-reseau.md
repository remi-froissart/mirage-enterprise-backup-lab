# 01 - Architecture réseau et virtualisation

## Objectif

L'objectif de cette première partie est de construire une infrastructure virtualisée permettant de séparer les différents rôles du lab et de reproduire une architecture proche d'un environnement d'entreprise.

Le lab repose sur **VMware Workstation Pro** installé sur le poste physique.

Une VM **VMware ESXi 8** est utilisée en nested virtualization afin d'héberger l'infrastructure principale, tandis que les composants de sauvegarde sont volontairement placés en dehors de l'hyperviseur ESXi.

---

## Architecture générale

Le lab est organisé autour de trois éléments principaux :

- **ESXI01** : hyperviseur hébergeant les VM de production ;
- **VEEAM01** : serveur Veeam Backup & Replication ;
- **NAS01** : serveur Debian utilisé comme Hardened Repository.

VEEAM01 et NAS01 sont exécutés directement dans VMware Workstation et ne sont donc pas hébergés dans ESXI01.

Cette séparation permet de conserver l'infrastructure de sauvegarde disponible en cas de perte ou de compromission de l'hyperviseur ESXi.

![Architecture générale du lab](../images/architecture/mirage-enterprise-backup-lab.png)

---

## ESXi

L'hyperviseur utilisé est :

| Paramètre | Valeur |
|---|---|
| Nom | ESXI01 |
| Hyperviseur | VMware ESXi 8 |
| Type | Nested virtualization |
| Adresse de management | 192.168.110.10/24 |
| RAM allouée | 16 Go |
| vCPU | 6 |

ESXI01 héberge les machines virtuelles suivantes :

| VM | Rôle | Adresse IP |
|---|---|---|
| FW01 | Firewall OPNsense | Plusieurs interfaces |
| DC01 | Active Directory / DNS | 10.10.10.10 |
| FS01 | Serveur de fichiers | 10.10.10.20 |
| CLT-W10-01 | Poste client Windows 10 | 10.10.10.50 |

---

## Segmentation réseau

Trois réseaux sont utilisés dans le lab.

### LAN

```text
10.10.10.0/24
