# Journal technique complet — Projet Holodeck

> Ce document retrace **toutes les étapes réellement suivies** pour
> construire l'infrastructure Holodeck, avec les commandes exactes
> utilisées, le raisonnement derrière chaque choix, et les problèmes
> rencontrés en cours de route (avec leur diagnostic et leur résolution).
> Objectif : garder une trace exploitable dans le temps (mémoire, révision,
> reproduction du projet).

## Sommaire

1. [Contexte et objectif du projet](#1-contexte-et-objectif-du-projet)
2. [Architecture cible](#2-architecture-cible)
3. [Préparation des VM](#3-préparation-des-vm)
4. [DHCP et DNS](#4-dhcp-et-dns)
5. [PHP 7.4 et 8.3 en cohabitation](#5-php-74-et-83-en-cohabitation)
6. [Nginx](#6-nginx)
7. [Certificat SSL — autorité de certification maison](#7-certificat-ssl--autorité-de-certification-maison)
8. [Passage de tous les vhosts en HTTPS](#8-passage-de-tous-les-vhosts-en-https)
9. [MariaDB](#9-mariadb)
10. [phpMyAdmin](#10-phpmyadmin)
11. [Cockpit et admin.starfleet.lan](#11-cockpit-et-adminstarfleetlan)
12. [LDAP](#12-ldap)
13. [FTP en SSL/TLS chrooté](#13-ftp-en-ssltls-chrooté)
14. [Pare-feu nftables](#14-pare-feu-nftables)
15. [Configuration de la VM Cliente](#15-configuration-de-la-vm-cliente)
16. [Annexe — pièges rencontrés et leçons apprises](#16-annexe--pièges-rencontrés-et-leçons-apprises)

---

## 1. Contexte et objectif du projet

Projet réalisé dans le cadre de la formation Bac+3 Cybersécurité à La
Plateforme (Marseille). Le sujet, présenté sous la forme d'un scénario
Star Trek, demande de construire pour la Fédération des Planètes Unies
une infrastructure web complète sur 2 VM Debian :

- une VM **Serveur** hébergeant DHCP, DNS, un serveur web (Nginx, PHP,
  MariaDB), un annuaire LDAP, un serveur FTP, et une interface
  d'administration
- une VM **Cliente** pour tester l'accès aux services depuis un
  navigateur

Contraintes imposées par le sujet :
- Aucun compte `sudo` (administration en root direct)
- Pare-feu n'autorisant que les ports strictement nécessaires
- Nginx en HTTPS
- PHP, MariaDB et Nginx en dernière version (pas celle des dépôts Debian
  de base)
- PHP 7.x et 8.x doivent cohabiter sur le même serveur
- FTP en SSL/TLS, chrooté sur le dossier web
- Authentification web via LDAP
- Un seul certificat SSL commun au serveur web et au serveur FTP

## 2. Architecture cible

```
VM Serveur (Debian, sans GUI, 2Go RAM, 2vCPU, 32Go disque)
├── enp1s0 (WAN, DHCP)         → accès Internet (mises à jour)
└── enp2s0 (LAN, 192.168.10.1/24) → réseau interne starfleet.lan
        │
        └── VM Cliente (Debian, GUI, 2Go RAM, 2vCPU, 16Go disque)
            └── IP obtenue par DHCP sur le LAN (ex. 192.168.10.100)
```

Domaine interne : `starfleet.lan`, avec les enregistrements suivants :

| Nom | Rôle |
|---|---|
| www7.starfleet.lan | Site PHP 7.4 |
| www8.starfleet.lan | Site PHP 8.3 |
| php.starfleet.lan | phpMyAdmin |
| admin.starfleet.lan | Cockpit (administration système) |

## 3. Préparation des VM

Hyperviseur utilisé : **QEMU/KVM** avec **virt-manager**.

### VM Serveur

- Installation Debian minimale (pas d'environnement de bureau)
- 2 cartes réseau ajoutées dans les paramètres de la VM :
  - une carte reliée au réseau NAT par défaut de l'hyperviseur (deviendra
    `enp1s0`, le WAN)
  - une carte reliée à un réseau virtuel interne dédié (deviendra
    `enp2s0`, le LAN)

### VM Cliente

- Installation Debian avec environnement de bureau
- 1 seule carte réseau, reliée **au même réseau interne** que la carte
  LAN de la VM Serveur

> **Piège rencontré** : la VM Cliente avait initialement sa carte réseau
> reliée au réseau NAT de l'hyperviseur au lieu du réseau interne LAN.
> Résultat : elle recevait une IP en `192.168.122.x` (plage du NAT) au
> lieu de `192.168.10.x`, et ne pouvait donc pas atteindre le DHCP/DNS du
> serveur. Correction : changer la source réseau de la carte dans les
> paramètres de la VM au niveau de l'hyperviseur (pas dans l'OS).

## 4. DHCP et DNS

### Installation

```bash
apt install -y isc-dhcp-server bind9
```

### Configuration DHCP (isc-dhcp-server)

Fichier `/etc/default/isc-dhcp-server` : préciser l'interface d'écoute
(`INTERFACESv4="enp2s0"`), pour que le service n'écoute que sur le LAN et
jamais sur le WAN.

Fichier `/etc/dhcp/dhcpd.conf` : définir la plage d'adresses distribuées
sur le réseau `192.168.10.0/24`, avec le serveur DNS et le nom de domaine
`starfleet.lan`.

### Configuration DNS (BIND9)

Création d'une zone directe et d'une zone inverse pour `starfleet.lan`,
avec les enregistrements A pour chaque sous-domaine (www7, www8, php,
admin, et plus tard vscore).

### Vérification

```bash
systemctl status isc-dhcp-server
systemctl status bind9
dig @127.0.0.1 www7.starfleet.lan
```

La commande `dig` doit renvoyer une réponse dans la section `ANSWER`
pointant vers l'IP du serveur (`192.168.10.1`).

> **Piège rencontré et résolu plus tard (voir section 16)** : le fichier
> `/etc/resolv.conf` de la VM Serveur se faisait régulièrement écraser
> par le client DHCP sur l'interface WAN, qui imposait le DNS du réseau
> NAT de l'hyperviseur (`192.168.122.1`) au lieu du DNS local
> (`127.0.0.1` / `192.168.10.1`). Ce problème a nécessité plusieurs
> tentatives de correction avant d'être résolu définitivement — voir
> l'annexe pour le détail complet.

## 5. PHP 7.4 et 8.3 en cohabitation

### Pourquoi ne pas compiler PHP depuis les sources

Une première tentative a consisté à télécharger les archives sources de
PHP (`.tar.gz`) via `wget` et à les extraire avec `tar -xzf`, dans l'idée
de les compiler manuellement. Cette approche a été abandonnée avant
d'aller plus loin, pour plusieurs raisons :

- nécessite d'installer manuellement toutes les bibliothèques de
  compilation (libxml2-dev, libssl-dev, etc.)
- ne s'intègre pas nativement comme service systemd
- très difficile à documenter de façon reproductible
- le sujet demande juste "pas la version des dépôts Debian de base", ce
  qu'un dépôt tiers reconnu satisfait sans compilation

### Solution retenue : dépôt tiers sury.org

Ce dépôt (maintenu par Ondřej Surý, un mainteneur officiel des paquets
PHP Debian) fournit des paquets PHP à jour, avec plusieurs versions
installables en parallèle.

```bash
apt install -y apt-transport-https lsb-release ca-certificates curl gnupg2
curl -sSL https://packages.sury.org/php/apt.gpg -o /usr/share/keyrings/deb.sury.org-php.gpg
echo "deb [signed-by=/usr/share/keyrings/deb.sury.org-php.gpg] https://packages.sury.org/php/ $(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list
apt update
```

### Installation de PHP 7.4

```bash
apt install -y php7.4 php7.4-fpm php7.4-cli php7.4-mysql php7.4-curl php7.4-xml php7.4-mbstring
systemctl enable php7.4-fpm
systemctl start php7.4-fpm
```

> Le paquet n'active pas le service FPM par défaut
> (`NOTICE: Not enabling PHP 7.4 FPM by default`) — il faut l'activer
> manuellement.

### Installation de PHP 8.3

```bash
apt install -y php8.3 php8.3-fpm php8.3-cli php8.3-mysql php8.3-curl php8.3-xml php8.3-mbstring
systemctl enable php8.3-fpm
systemctl start php8.3-fpm
```

### Vérification

```bash
php7.4 -v
php8.3 -v
systemctl status php7.4-fpm
systemctl status php8.3-fpm
ls -l /run/php/
```

Les deux sockets `php7.4-fpm.sock` et `php8.3-fpm.sock` doivent
apparaître dans `/run/php/`.

## 6. Nginx

### Pourquoi le dépôt officiel plutôt que Debian

Même logique que pour PHP : le sujet impose une version récente, pas
celle des dépôts Debian.

```bash
apt install -y curl gnupg2 ca-certificates lsb-release debian-archive-keyring
curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor | tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/debian $(lsb_release -cs) nginx" | tee /etc/apt/sources.list.d/nginx.list
printf "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" | tee /etc/apt/preferences.d/99nginx
apt update
apt install -y nginx
systemctl enable nginx
systemctl start nginx
```

### Vérification de la version installée

```bash
nginx -v
apt-cache policy nginx
```

Vérifier que la ligne "Installed" pointe bien vers `nginx.org`, pas
`deb.debian.org`.

### Création des vhosts

Un fichier de configuration par sous-domaine dans
`/etc/nginx/conf.d/<nom>.conf`, chacun avec :
- son `server_name`
- son `root` (dossier `/var/www/<nom>/public`)
- un bloc `location ~ \.php$` pointant vers le bon socket PHP-FPM (7.4
  pour www7, 8.3 pour www8)

### Piège n°1 — ordre de chargement des vhosts

Le paquet nginx.org installe un fichier `default.conf` avec
`server_name localhost;`, qui capte tout le trafic non reconnu à cause de
l'ordre alphabétique de chargement des fichiers dans `conf.d/`. Un des
vhosts (www7) n'apparaissait même pas dans la liste des vhosts chargés —
le fichier n'avait en réalité jamais été créé correctement lors d'une
tentative précédente. Solution :

```bash
mv /etc/nginx/conf.d/default.conf /etc/nginx/conf.d/default.conf.disabled
nginx -t
systemctl reload nginx
nginx -T | grep "server_name"
```

### Piège n°2 — permissions du socket PHP-FPM

Une fois tous les vhosts bien chargés, l'accès à www7/www8 renvoyait une
erreur **502 Bad Gateway**. Le log Nginx a révélé la cause exacte :

```
connect() to unix:/run/php/php7.4-fpm.sock failed (13: Permission denied)
```

Nginx (installé depuis nginx.org) tourne sous l'utilisateur `nginx`,
tandis que PHP-FPM (paquet Debian/sury.org) crée son socket avec les
permissions par défaut pour l'utilisateur `www-data`. Correction dans les
pools PHP-FPM :

```bash
sed -i 's/^listen.owner = .*/listen.owner = nginx/' /etc/php/7.4/fpm/pool.d/www.conf
sed -i 's/^listen.group = .*/listen.group = nginx/' /etc/php/7.4/fpm/pool.d/www.conf
sed -i 's/^;\?listen.mode = .*/listen.mode = 0660/' /etc/php/7.4/fpm/pool.d/www.conf
# même chose pour /etc/php/8.3/fpm/pool.d/www.conf
systemctl restart php7.4-fpm
systemctl restart php8.3-fpm
```

### Vérification finale

```bash
curl -s -H "Host: www7.starfleet.lan" http://localhost | grep -i "PHP Version"
curl -s -H "Host: www8.starfleet.lan" http://localhost | grep -i "PHP Version"
```

Doit afficher respectivement `7.4.33` et `8.3.33`.

## 7. Certificat SSL — autorité de certification maison

Le sujet impose **un seul certificat SSL** utilisé à la fois par Nginx et
par le serveur FTP. Choix retenu : créer une petite autorité de
certification (CA) interne au projet, plutôt qu'un simple certificat
auto-signé — cette approche est plus proche du fonctionnement réel d'une
CA publique (type Let's Encrypt), et plus valorisante à présenter à
l'oral.

```bash
mkdir -p /etc/ssl/starfleet-ca
cd /etc/ssl/starfleet-ca

# 1. Clé privée de la CA
openssl genrsa -out ca.key 4096

# 2. Certificat racine auto-signé de la CA (valable 10 ans)
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt \
  -subj "/C=FR/ST=Starfleet/L=Enterprise/O=United Federation of Planets/OU=Starfleet Command/CN=Starfleet Root CA"

# 3. Clé privée du certificat serveur
openssl genrsa -out starfleet.key 2048

# 4. Demande de signature (CSR)
openssl req -new -key starfleet.key -out starfleet.csr \
  -subj "/C=FR/ST=Starfleet/L=Enterprise/O=United Federation of Planets/OU=Web Server/CN=starfleet.lan"

# 5. Fichier d'extensions listant tous les sous-domaines (SAN)
cat > starfleet.ext << 'EOF'
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = starfleet.lan
DNS.2 = www7.starfleet.lan
DNS.3 = www8.starfleet.lan
DNS.4 = php.starfleet.lan
DNS.5 = admin.starfleet.lan
DNS.6 = vscore.starfleet.lan
EOF

# 6. Signature du certificat serveur par la CA
openssl x509 -req -in starfleet.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out starfleet.crt -days 825 -sha256 -extfile starfleet.ext
```

### Vérification

```bash
openssl x509 -in starfleet.crt -text -noout | grep -A 8 "Subject Alternative Name"
```

Doit afficher tous les sous-domaines listés dans le fichier d'extensions.

> **Pourquoi un SAN et pas juste un CN** : les navigateurs modernes
> refusent un certificat qui ne couvre son nom que via le Common Name
> (CN) — il faut impérativement lister chaque nom de domaine dans le
> Subject Alternative Name pour que le certificat soit accepté sans
> avertissement sur chaque sous-domaine.

### Fichiers sensibles

`ca.key` et `starfleet.key` sont les clés privées : elles ne doivent
**jamais** être publiées (voir le README pour l'avertissement complet).

## 8. Passage de tous les vhosts en HTTPS

Chaque vhost a été réécrit avec deux blocs `server` :
- un premier bloc en `listen 80;` qui redirige tout vers HTTPS
  (`return 301 https://$host$request_uri;`)
- un second bloc en `listen 443 ssl;` qui référence le certificat commun :

```nginx
ssl_certificate /etc/ssl/starfleet-ca/starfleet.crt;
ssl_certificate_key /etc/ssl/starfleet-ca/starfleet.key;
```

### Piège — HTTPS non détecté par PHP

phpMyAdmin affichait un avertissement "mismatch between HTTPS indicated
on the server and client", car PHP-FPM ne reçoit pas nativement
l'information "cette requête est en HTTPS" (c'est Nginx qui gère le
chiffrement, pas PHP). Correction : ajouter dans chaque bloc
`location ~ \.php$` du vhost HTTPS :

```nginx
fastcgi_param HTTPS on;
```

### Test en HTTPS

```bash
curl -sk -H "Host: www7.starfleet.lan" https://localhost | grep -i "PHP Version"
```

Le `-k` ignore la vérification de confiance (normal tant que la CA n'est
pas encore reconnue par l'outil de test).

### Importer la CA dans le navigateur de la VM Cliente

1. Copier temporairement `ca.crt` dans un dossier web accessible (ex.
   `/var/www/www7.starfleet.lan/public/ca.crt`)
2. Télécharger le fichier depuis le navigateur de la VM Cliente via
   `https://www7.starfleet.lan/ca.crt`
3. Retirer le fichier du dossier web une fois téléchargé (ne pas laisser
   traîner la CA sur un site accessible)
4. Dans Firefox : Paramètres → Vie privée et sécurité → Certificats →
   Afficher les certificats → onglet Autorités → Importer → cocher
   "Faire confiance à cette autorité de certification pour identifier des
   sites web"

## 9. MariaDB

### Installation via le dépôt officiel

```bash
curl -LsS -O https://downloads.mariadb.com/MariaDB/mariadb_repo_setup
bash mariadb_repo_setup --mariadb-server-version=mariadb-11.4
apt update
apt install -y mariadb-server mariadb-client
systemctl enable mariadb
systemctl start mariadb
```

### Sécurisation

```bash
mysql_secure_installation
```

Répondre "oui" à toutes les questions de sécurisation (mot de passe root,
suppression des comptes anonymes, désactivation de la connexion root à
distance, suppression de la base de test).

### Vérification

```bash
mysql --version
mysql -u root -p -e "SELECT user, plugin FROM mysql.user WHERE user='root';"
```

Le plugin d'authentification de root peut être `unix_socket` (auth par
utilisateur système, pas de mot de passe classique) ou
`mysql_native_password` (mot de passe classique). Dans ce projet, c'est
`mysql_native_password` — la connexion à phpMyAdmin se fait donc
directement avec `root` et le mot de passe défini lors de la
sécurisation.

## 10. phpMyAdmin

```bash
apt install -y php8.3-mbstring php8.3-zip php8.3-gd unzip wget
cd /tmp
wget https://www.phpmyadmin.net/downloads/phpMyAdmin-latest-all-languages.zip
unzip phpMyAdmin-latest-all-languages.zip
mkdir -p /var/www/php.starfleet.lan/public
mv phpMyAdmin-*-all-languages/* /var/www/php.starfleet.lan/public/
cp /var/www/php.starfleet.lan/public/config.sample.inc.php /var/www/php.starfleet.lan/public/config.inc.php
```

### Clé de session obligatoire

```bash
openssl rand -base64 32
```

Insérer la clé générée dans `config.inc.php` :

```bash
sed -i "s/\$cfg\['blowfish_secret'\] = '';/\$cfg['blowfish_secret'] = 'VALEUR_GENEREE';/" /var/www/php.starfleet.lan/public/config.inc.php
```

> Attention si la clé générée contient des caractères `/` : ils doivent
> être échappés en `\/` dans la commande `sed`, car `/` est le séparateur
> utilisé par cette commande.

### Vhost dédié

Vhost `php.starfleet.lan` créé sur le même modèle que www7/www8, pointant
vers le pool PHP 8.3.

### Vérification

```bash
curl -s -H "Host: php.starfleet.lan" http://localhost | grep -i "phpMyAdmin"
```

## 11. Cockpit et admin.starfleet.lan

Cockpit est l'outil retenu pour l'administration système
(`admin.starfleet.lan ⇒ administration de la VM`), parmi les deux options
proposées par le sujet (Webmin ou Cockpit).

### Installation

```bash
apt install -y cockpit
systemctl enable --now cockpit.socket
```

> Cockpit fonctionne par activation à la demande (socket activation) :
> `cockpit.socket` est actif en permanence, mais `cockpit.service`
> n'apparaît "actif" qu'au moment d'une connexion réelle. Voir
> `cockpit.service` en `inactive (dead)` alors que le socket est
> `active (listening)` est donc normal et ne signale aucun problème.

### Vhost Nginx en reverse proxy

Cockpit écoute nativement sur `127.0.0.1:9090`. Nginx sert de proxy pour
exposer ce service sous `admin.starfleet.lan` avec le certificat commun.

### Piège n°1 — root bloqué par défaut

Cockpit refuse par défaut les connexions avec le compte `root`, listé
dans `/etc/cockpit/disallowed-users`. Comme le sujet n'impose pas de
compte dédié pour cette interface, root a été explicitement autorisé :

```bash
sed -i '/^root$/d' /etc/cockpit/disallowed-users
systemctl restart cockpit
```

### Piège n°2 — mauvais protocole vers le backend

Le vhost pointait initialement vers `proxy_pass https://127.0.0.1:9090;`.
Or Cockpit sert en réalité du **HTTP en clair** sur ce port (vérifié via
`curl -v http://127.0.0.1:9090`, qui renvoie un `200 OK` normal). Ce
mauvais protocole empêchait le WebSocket de Cockpit de s'établir
correctement. Correction :

```nginx
proxy_pass http://127.0.0.1:9090;   # et non https://
```

### Piège n°3 — en-têtes WebSocket manquants

Cockpit utilise des WebSockets pour son shell interactif. Sans les
en-têtes suivants, la connexion reste bloquée en chargement infini après
le login :

```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
proxy_read_timeout 900;
proxy_buffering off;
```

### Piège n°4 — vérification d'Origin de Cockpit (403 sur le WebSocket)

Même avec les en-têtes WebSocket corrects, la console du navigateur (F12)
affichait :

```
Firefox ne peut établir de connexion avec le serveur à l'adresse
wss://admin.starfleet.lan/cockpit/socket.
```

Et l'onglet Réseau montrait un code **403 Forbidden** sur cette requête.
Cause : Cockpit vérifie l'en-tête `Origin` de la requête WebSocket pour
se protéger des attaques CSRF, et ne reconnaissait pas
`admin.starfleet.lan` comme origine de confiance (il ne "sait" pas
nativement qu'il est exposé derrière un reverse proxy sous ce nom).
Correction :

```bash
cat > /etc/cockpit/cockpit.conf << 'EOF'
[WebService]
Origins = https://admin.starfleet.lan
ProtocolHeader = X-Forwarded-Proto
EOF
systemctl restart cockpit
```

### Vhost final complet

```nginx
server {
    listen 80;
    server_name admin.starfleet.lan;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name admin.starfleet.lan;

    ssl_certificate /etc/ssl/starfleet-ca/starfleet.crt;
    ssl_certificate_key /etc/ssl/starfleet-ca/starfleet.key;

    location / {
        proxy_pass http://127.0.0.1:9090;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 900;
        proxy_buffering off;
    }
}
```

## 12. LDAP

### Installation

```bash
apt install -y slapd ldap-utils
dpkg-reconfigure slapd
```

Réponses données lors de la reconfiguration :
- Omit OpenLDAP server configuration ? → **No**
- DNS domain name → `starfleet.lan`
- Organization name → `Starfleet`
- Administrator password → mot de passe défini (non documenté ici pour
  des raisons de sécurité)
- Database backend → **MDB**
- Remove database when slapd is purged ? → **No**
- Move old database ? → **Yes**

Base DN résultante : `dc=starfleet,dc=lan`

### Vérification de base

```bash
systemctl status slapd
ldapsearch -x -D "cn=admin,dc=starfleet,dc=lan" -W -b "dc=starfleet,dc=lan"
```

### Arborescence

```bash
cat > /root/base_structure.ldif << 'EOF'
dn: ou=people,dc=starfleet,dc=lan
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=starfleet,dc=lan
objectClass: organizationalUnit
ou: groups
EOF
ldapadd -x -D "cn=admin,dc=starfleet,dc=lan" -W -f /root/base_structure.ldif
```

### Utilisateur de test

```bash
slappasswd
# copier le hash {SSHA}... généré

cat > /root/user_test.ldif << 'EOF'
dn: uid=picard,ou=people,dc=starfleet,dc=lan
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: picard
cn: Jean-Luc Picard
sn: Picard
givenName: Jean-Luc
mail: picard@starfleet.lan
userPassword: <hash SSHA généré par slappasswd>
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/picard
loginShell: /bin/bash
EOF
ldapadd -x -D "cn=admin,dc=starfleet,dc=lan" -W -f /root/user_test.ldif
```

### Authentification web réelle

Pour prouver que l'authentification LDAP fonctionne réellement depuis une
application web (et pas seulement en ligne de commande), un petit
formulaire PHP a été créé sur `www8.starfleet.lan`, qui effectue un
`ldap_bind()` avec les identifiants saisis :

```bash
apt install -y php7.4-ldap php8.3-ldap
systemctl restart php7.4-fpm
systemctl restart php8.3-fpm
```

Le formulaire construit le DN `uid=<saisi>,ou=people,dc=starfleet,dc=lan`
et tente un `ldap_bind()` avec le mot de passe saisi — succès confirmé
avec l'utilisateur `picard`.

## 13. FTP en SSL/TLS chrooté

### Installation

```bash
apt install -y vsftpd
cp /etc/vsftpd.conf /etc/vsftpd.conf.bak
```

### Configuration

```bash
cat > /etc/vsftpd.conf << 'EOF'
listen=YES
listen_ipv6=NO

anonymous_enable=NO
local_enable=YES
write_enable=YES

chroot_local_user=YES
allow_writeable_chroot=YES

local_root=/var/www/www7.starfleet.lan/public

pasv_enable=YES
pasv_min_port=40000
pasv_max_port=40100

ssl_enable=YES
rsa_cert_file=/etc/ssl/starfleet-ca/starfleet.crt
rsa_private_key_file=/etc/ssl/starfleet-ca/starfleet.key
force_local_data_ssl=YES
force_local_logins_ssl=YES
ssl_tlsv1=YES
ssl_sslv2=NO
ssl_sslv3=NO
require_ssl_reuse=NO
ssl_ciphers=HIGH

xferlog_enable=YES
EOF
```

Le même certificat que Nginx (`starfleet.crt` / `starfleet.key`) est
réutilisé ici, conformément à l'exigence du sujet d'un certificat commun.

### Utilisateur de test

```bash
adduser ftpuser
chown ftpuser:ftpuser /var/www/www7.starfleet.lan/public -R
systemctl enable vsftpd
systemctl restart vsftpd
```

### Test de connexion

```bash
lftp ftp://ftpuser@127.0.0.1 -e "set ftp:ssl-force true; set ssl:verify-certificate no; ls; bye"
```

> **Faux problème rencontré** : la syntaxe `lftp -u ftpuser, ftp://...`
> (avec une virgule suivie de rien) envoie un mot de passe **vide**,
> provoquant systématiquement un "530 Login incorrect" qui n'a rien à
> voir avec la configuration de vsftpd, PAM ou `/etc/ftpusers`. Ces
> pistes ont été explorées inutilement avant de comprendre que c'était un
> problème de syntaxe de la commande de test elle-même. La syntaxe
> correcte pour un test interactif (avec prompt de mot de passe) est
> `lftp ftp://ftpuser@127.0.0.1 -e "..."`.

### Vérification de l'étanchéité du chroot

```bash
lftp ftp://ftpuser@127.0.0.1 -e "set ftp:ssl-force true; set ssl:verify-certificate no; cd /; pwd; ls; bye"
```

`pwd` doit afficher `/` (racine du chroot) et `ls` ne doit montrer que le
contenu du dossier web (pas `/etc`, `/root`, `/home`...), confirmant que
l'utilisateur ne peut pas sortir de son chroot.

## 14. Pare-feu nftables

### Installation

```bash
apt install -y nftables
cp /etc/nftables.conf /etc/nftables.conf.bak
```

### Configuration — politique par défaut restrictive

```bash
cat > /etc/nftables.conf << 'EOF'
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;

        ct state established,related accept
        iif "lo" accept

        ip protocol icmp accept
        ip6 nexthdr icmpv6 accept

        iifname "enp1s0" tcp dport 22 accept

        iifname "enp2s0" tcp dport 53 accept
        iifname "enp2s0" udp dport 53 accept
        iifname "enp2s0" udp dport 67 accept

        iifname "enp2s0" tcp dport { 80, 443 } accept
        iifname "enp1s0" tcp dport { 80, 443 } accept

        iifname "enp2s0" tcp dport 21 accept
        iifname "enp2s0" tcp dport 40000-40100 accept

        iifname "enp2s0" tcp dport 389 accept

        log prefix "nftables-dropped: " counter drop
    }

    chain forward {
        type filter hook forward priority 0; policy drop;
    }

    chain output {
        type filter hook output priority 0; policy accept;
    }
}
EOF
```

### Vérification avant application

```bash
nft -c -f /etc/nftables.conf
```

> **Point de vigilance critique** : appliquer un pare-feu par une
> connexion SSH distante comporte un risque réel de se couper soi-même
> l'accès si la règle SSH est mal écrite. Il est fortement recommandé de
> garder ouverte la console locale de l'hyperviseur (pas seulement une
> session SSH) avant d'appliquer un nouveau ruleset — ce principe s'est
> vérifié plus tôt dans le projet avec un incident similaire lors du
> renouvellement d'un bail DHCP (voir annexe).

### Application

```bash
systemctl enable nftables
systemctl restart nftables
nft list ruleset
```

### Vérification post-application

Retester tous les services (web, FTP, DNS) depuis la VM Cliente pour
confirmer que le pare-feu n'a rien cassé.

## 15. Configuration de la VM Cliente

- Carte réseau reliée au réseau interne LAN (pas au NAT de l'hyperviseur)
- IP obtenue par DHCP (ex. `192.168.10.100`)
- Résolution DNS validée avec `ping www7.starfleet.lan`
- Certificat de la CA importé dans Firefox (voir section 8)
- Navigateur utilisé pour valider chaque service (www7, www8, php,
  admin) et le client FTP FileZilla pour valider le FTP en SSL/TLS

## 16. Annexe — pièges rencontrés et leçons apprises

### 16.1 — resolv.conf écrasé en boucle par dhclient (le piège le plus long à résoudre)

**Symptôme** : `curl http://localhost` échouait avec une erreur de
résolution DNS sur la VM Serveur elle-même, alors que BIND9 fonctionnait
parfaitement en interrogation directe (`dig @127.0.0.1` répondait
correctement).

**Diagnostic** : `/etc/resolv.conf` contenait `nameserver 192.168.122.1`
(le DNS du réseau NAT de l'hyperviseur) au lieu de pointer vers le DNS
local. Le client DHCP sur l'interface WAN (`enp1s0`) réécrivait ce
fichier à chaque renouvellement de bail, en y imposant le DNS reçu du
NAT.

**Tentatives successives** :
1. Remettre manuellement le bon contenu dans `/etc/resolv.conf` — tenait
   jusqu'au prochain renouvellement de bail, puis était à nouveau écrasé
2. Désactiver NetworkManager (qui était actif en parallèle du système
   `/etc/network/interfaces`, source de confusion) — nécessaire mais pas
   suffisant, le vrai coupable était dhclient lui-même, pas
   NetworkManager
3. Ajouter une directive `supersede domain-name-servers 127.0.0.1,
   192.168.10.1;` dans `/etc/dhcp/dhclient.conf` — la première tentative
   d'ajout via `nano` n'a pas été correctement enregistrée (la ligne
   n'apparaissait pas au `grep` suivant) ; un second ajout via `echo ...
   >> /etc/dhcp/dhclient.conf` a bien fonctionné, mais même correctement
   écrite, cette directive ne s'est pas toujours révélée fiable au
   renouvellement du bail

**Solution finale retenue** : rendre le fichier immuable au niveau du
système de fichiers, ce qui empêche **toute** réécriture, y compris par
dhclient exécuté en root :

```bash
cat > /etc/resolv.conf << 'EOF'
nameserver 127.0.0.1
nameserver 192.168.10.1
EOF
chattr +i /etc/resolv.conf
```

Vérification que la protection fonctionne :

```bash
lsattr /etc/resolv.conf   # doit afficher l'attribut i
dhclient -r enp1s0
dhclient enp1s0
cat /etc/resolv.conf      # doit être resté inchangé
```

Le journal confirme la protection en action :
```
/sbin/dhclient-script: 88: cannot create /etc/resolv.conf: Operation not permitted
```

**Conséquence pratique à retenir** : pour modifier `/etc/resolv.conf` à
l'avenir sur cette VM, il faut d'abord lever la protection
(`chattr -i /etc/resolv.conf`), modifier, puis la remettre
(`chattr +i /etc/resolv.conf`).

**Incident lié** : une tentative de renouvellement de bail
(`dhclient -r enp1s0`) via une session **SSH** a provoqué la coupure
immédiate de la connexion (la carte WAN, support de la session SSH, a été
temporairement désactivée par la commande). Reprise de la main uniquement
possible via la **console locale de l'hyperviseur** (fenêtre d'affichage
direct de la VM, pas une connexion réseau). Leçon retenue : toute
manipulation réseau sensible (renouvellement de bail, application d'un
pare-feu) doit être testée avec la console locale ouverte en parallèle,
jamais uniquement via une connexion qui dépend elle-même du réseau qu'on
modifie.

### 16.2 — Ordre de chargement des vhosts Nginx

Voir section 6, piège n°1.

### 16.3 — Permissions de socket PHP-FPM entre nginx et www-data

Voir section 6, piège n°2.

### 16.4 — Faux problème d'authentification FTP (syntaxe lftp)

Voir section 13.

### 16.5 — Cockpit et le reverse proxy (protocole, WebSocket, Origin)

Voir section 11 — le débogage le plus long du projet après le problème
DNS, avec 4 causes différentes identifiées et corrigées successivement
avant d'obtenir un fonctionnement complet.

---

## Conclusion

Ce projet a permis de mettre en pratique, sur une infrastructure
complète et fonctionnelle, l'ensemble des compétences visées par le
sujet : administration système sans privilèges sudo, sécurisation d'une
infrastructure (pare-feu, chiffrement TLS de bout en bout avec une CA
maison, chroot FTP), et mise en œuvre d'un annuaire LDAP avec
authentification applicative réelle. La majorité des difficultés
rencontrées n'étaient pas des erreurs de conception, mais des
interactions subtiles entre composants (permissions inter-services,
comportement du client DHCP, vérifications de sécurité internes à
Cockpit) — le genre de problèmes typiques d'une administration système
réelle, où le diagnostic méthodique (logs, tests isolés composant par
composant) s'est avéré plus efficace que les corrections à l'aveugle.
