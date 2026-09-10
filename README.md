[README (1).md](https://github.com/user-attachments/files/32047633/README.1.md)

# Holodeck — Machines virtuelles web pour les ingénieurs de Starfleet

Projet école (La Plateforme) : mise en place d'une infrastructure web
complète sur 2 VM Debian, dans le respect des contraintes du sujet
"Holodeck".

## Sommaire

- [Architecture générale](#architecture-générale)
- [VM Serveur — détail](#vm-serveur--détail)
- [VM Cliente — détail](#vm-cliente--détail)
- [Services installés](#services-installés)
- [Pare-feu](#pare-feu)
- [Annuaire LDAP](#annuaire-ldap)
- [Certificat SSL](#certificat-ssl)
- [Contrainte "pas de compte sudo"](#contrainte-pas-de-compte-sudo)
- [⚠️ Sécurité — fichiers à ne jamais publier](#️-sécurité--fichiers-à-ne-jamais-publier)
- [Documentation complémentaire](#documentation-complémentaire)
- [Compétences visées](#compétences-visées)

---

## Architecture générale

```
                    Internet / réseau hôte
                            │
                            │  (WAN, DHCP)
                    ┌───────┴────────┐
                    │   VM Serveur   │
                    │  enp1s0 (WAN)  │
                    │  enp2s0 (LAN)  │──── 192.168.10.1/24
                    └───────┬────────┘
                            │
                     réseau interne isolé
                     starfleet-lan (192.168.10.0/24)
                            │
                    ┌───────┴────────┐
                    │   VM Cliente   │──── 192.168.10.x (DHCP)
                    └────────────────┘
```

Domaine interne : **starfleet.lan**

| Sous-domaine | Rôle |
|---|---|
| `www7.starfleet.lan` | Site web en PHP 7.4 |
| `www8.starfleet.lan` | Site web en PHP 8.3 |
| `php.starfleet.lan` | phpMyAdmin (administration MariaDB) |
| `admin.starfleet.lan` | Cockpit (administration système de la VM) |

---

## VM Serveur — détail

- **OS** : Debian (installation minimale, sans interface graphique)
- **Ressources** : 2 Go RAM, 2 vCPU, disque 32 Go
- **Réseau** : 2 cartes réseau
  - `enp1s0` (WAN) — DHCP, accès Internet pour les mises à jour
  - `enp2s0` (LAN) — IP statique `192.168.10.1/24`, réseau interne
    `starfleet.lan`

## VM Cliente — détail

- **OS** : Debian avec interface graphique
- **Ressources** : 2 Go RAM, 2 vCPU, disque 16 Go
- **Réseau** : 1 carte réseau, connectée uniquement au LAN interne,
  adresse obtenue par DHCP (ex. `192.168.10.100`)
- Navigateur web installé pour accéder aux services

---

## Services installés

| Service | Logiciel | Source du paquet | Détail |
|---|---|---|---|
| DHCP | isc-dhcp-server | dépôt Debian | Écoute sur `enp2s0` uniquement |
| DNS | BIND9 | dépôt Debian | Zone `starfleet.lan` (directe + inverse) |
| Serveur web | Nginx | dépôt officiel nginx.org | 4 vhosts en HTTPS |
| PHP | PHP-FPM 7.4.33 + 8.3.33 | dépôt sury.org | Deux pools FPM séparés, cohabitation des deux versions |
| Base de données | MariaDB 11.4.13 | dépôt officiel MariaDB | Sécurisée via `mysql_secure_installation` |
| Administration BDD | phpMyAdmin | téléchargement officiel | Sur `php.starfleet.lan` |
| Administration système | Cockpit | dépôt Debian | Sur `admin.starfleet.lan`, via reverse proxy Nginx |
| Annuaire | OpenLDAP (slapd) | dépôt Debian | Authentification du site web |
| FTP | vsftpd | dépôt Debian | SSL/TLS obligatoire, chrooté |
| Pare-feu | nftables | dépôt Debian | Politique par défaut : tout bloquer |

### Authentification web via LDAP

Le site `www8.starfleet.lan` intègre un formulaire de connexion qui
authentifie les utilisateurs directement contre l'annuaire LDAP (via
`ldap_bind`), démontrant que l'authentification LDAP fonctionne
réellement depuis une application web, et pas seulement en ligne de
commande.

---

## Pare-feu

Le pare-feu nftables applique une politique **par défaut restrictive**
(`policy drop`) : rien n'est autorisé sauf ce qui est explicitement listé
ci-dessous.

| Port | Protocole | Interface | Service |
|---|---|---|---|
| 22 | TCP | WAN (`enp1s0`) | SSH (administration) |
| 53 | TCP + UDP | LAN (`enp2s0`) | DNS |
| 67 | UDP | LAN (`enp2s0`) | DHCP |
| 80, 443 | TCP | LAN (`enp2s0`) + WAN (`enp1s0`) | HTTP / HTTPS |
| 21 | TCP | LAN (`enp2s0`) | FTP (contrôle) |
| 40000–40100 | TCP | LAN (`enp2s0`) | FTP (données, mode passif) |
| 389 | TCP | LAN (`enp2s0`) | LDAP |

Toutes les connexions déjà établies sont acceptées (`ct state
established,related`), le loopback est toujours autorisé, et tout le
trafic non explicitement autorisé est journalisé puis rejeté.

---

## Annuaire LDAP

- **Base DN** : `dc=starfleet,dc=lan`
- **Unités organisationnelles** :
  - `ou=people,dc=starfleet,dc=lan` — comptes utilisateurs
  - `ou=groups,dc=starfleet,dc=lan` — groupes

Exemple d'utilisateur créé pour les tests :
`uid=picard,ou=people,dc=starfleet,dc=lan`

---

## Certificat SSL

Une autorité de certification (CA) interne au projet a été créée avec
OpenSSL, afin de fournir **un seul certificat** couvrant tous les
sous-domaines (via l'extension Subject Alternative Name), utilisé à la
fois par :

- Nginx (HTTPS sur les 4 vhosts)
- vsftpd (FTP en SSL/TLS)

Sous-domaines couverts par le certificat : `starfleet.lan`,
`www7.starfleet.lan`, `www8.starfleet.lan`, `php.starfleet.lan`,
`admin.starfleet.lan`, `vscore.starfleet.lan`.

Le certificat de la CA (`ca.crt`) doit être importé dans le magasin de
confiance du navigateur de la VM Cliente pour naviguer sans avertissement
de sécurité (voir la notice d'installation et d'utilisation).

---

## Contrainte "pas de compte sudo"

Conformément aux directives du sujet, aucun compte utilisateur avec accès
`sudo` n'a été configuré sur la VM Serveur. Toute l'administration système
(installation des paquets, configuration des services, gestion des
fichiers système) est effectuée directement en tant que **root**, y
compris pour l'accès à Cockpit (`admin.starfleet.lan`), où la connexion
root a été explicitement autorisée à cet effet.

---

## ⚠️ Sécurité — fichiers à ne jamais publier

Ce dépôt est public. Les fichiers suivants sont des **clés privées** et ne
doivent **jamais** être versionnés ni publiés :

- `ca.key` (clé privée de l'autorité de certification)
- `starfleet.key` (clé privée du certificat serveur)
- Tout fichier contenant des mots de passe en clair (LDAP, MariaDB,
  comptes système)

Un fichier `.gitignore` doit exclure ces éléments avant tout `git push`.
Les identifiants nécessaires à l'évaluation sont transmis séparément à
l'équipe pédagogique, hors de ce dépôt.

---

## Documentation complémentaire

- [`01_procedure_export_vm.md`](./01_procedure_export_vm.md) — comment
  exporter les VM
- [`02_notice_installation_utilisation.md`](./02_notice_installation_utilisation.md)
  — installation et utilisation pour l'utilisateur final

---

## Compétences visées

- Administrer et sécuriser les infrastructures systèmes
- Concevoir une solution technique répondant à des besoins d'évolution de
  l'infrastructure
- Participer à l'élaboration et à la mise en œuvre de la politique de
  sécurité
