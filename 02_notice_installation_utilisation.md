# Notice d'installation et d'utilisation — Projet Holodeck

Cette notice s'adresse à l'utilisateur final qui souhaite installer et
utiliser l'environnement Holodeck (VM Serveur + VM Cliente) fourni avec ce
projet.

---

## 1. Installation des outils de virtualisation

L'environnement a été développé et testé avec **QEMU/KVM** et
**virt-manager** sous Linux (Debian/Ubuntu).

### Installation sur Debian/Ubuntu

```bash
apt update
apt install -y qemu-kvm libvirt-daemon-system libvirt-clients virt-manager
```

### Vérifier que le service libvirt tourne

```bash
systemctl enable --now libvirtd
systemctl status libvirtd
```

### Ajouter son utilisateur au groupe libvirt (pour éviter d'utiliser root)

```bash
usermod -aG libvirt,kvm $USER
```

Déconnectez-vous puis reconnectez-vous pour que ce changement prenne
effet.

---

## 2. Création des réseaux virtuels

Le projet nécessite **deux réseaux virtuels distincts** :

- Un réseau **WAN** (accès Internet, peut être le réseau NAT par défaut de
  libvirt, généralement nommé `default`)
- Un réseau **LAN interne** isolé, pour le réseau `starfleet.lan`
  (192.168.10.0/24)

### Créer le réseau LAN interne via virt-manager

1. Ouvrir **virt-manager**
2. Menu **Édition → Détails de connexion**
3. Onglet **Réseaux virtuels**
4. Cliquer sur **+** pour ajouter un réseau
5. Nommer le réseau (ex. `starfleet-lan`)
6. Choisir le mode **Isolé** (pas de forwarding vers l'hôte/Internet)
7. Définir la plage IPv4 sur `192.168.10.0/24` (le DHCP de ce réseau
   virtuel doit être **désactivé**, car c'est la VM Serveur qui fait
   office de serveur DHCP)
8. Valider

---

## 3. Import des VM

Si vous partez de l'archive d'export (`holodeck-export.tar.gz`) :

### Extraire l'archive

```bash
mkdir -p ~/holodeck
tar -xzvf holodeck-export.tar.gz -C ~/holodeck
cd ~/holodeck
```

### Copier les disques au bon emplacement

```bash
cp *.qcow2 /var/lib/libvirt/images/
```

### Définir les VM à partir des fichiers XML

```bash
virsh define holodeck-serveur.xml
virsh define holodeck-client.xml
```

### Vérifier que les VM apparaissent

```bash
virsh list --all
```

---

## 4. Configuration réseau des VM

Avant de démarrer les VM, vérifiez dans virt-manager (Détails de la VM →
Matériel → carte réseau) que :

- La **VM Serveur** a bien **deux cartes réseau** :
  - Une carte reliée au réseau **WAN** (`default` ou équivalent)
  - Une carte reliée au réseau **LAN** (`starfleet-lan`)
- La **VM Cliente** a bien **une carte réseau** reliée uniquement au
  réseau **LAN** (`starfleet-lan`)

---

## 5. Démarrage des VM

Démarrer d'abord la **VM Serveur**, puis la **VM Cliente** :

```bash
virsh start holodeck-serveur
virsh start holodeck-client
```

Ou depuis virt-manager : sélectionner la VM → bouton "Lancer" (▶).

---

## 6. Première connexion

### VM Serveur

Connexion en `root` (voir identifiants transmis séparément par
l'administrateur du projet).

Vérifier que le réseau est bien opérationnel :

```bash
ip a
cat /etc/resolv.conf
```

> **Remarque** : le fichier `/etc/resolv.conf` est volontairement rendu
> immuable (`chattr +i`) pour éviter qu'il soit écrasé par le client DHCP
> sur la carte WAN. Pour le modifier, utiliser d'abord
> `chattr -i /etc/resolv.conf`, puis remettre `chattr +i` après
> modification.

### VM Cliente

Se connecter avec la session graphique fournie. Vérifier la connectivité :

```bash
ip a
ping -c 2 www7.starfleet.lan
```

---

## 7. Faire confiance au certificat SSL du domaine

Le domaine `starfleet.lan` utilise un certificat signé par une autorité
de certification (CA) interne au projet, et non par une CA publique
reconnue. Pour naviguer sans avertissement de sécurité, il faut importer
le certificat de cette CA dans le navigateur de la VM Cliente :

1. Dans Firefox, aller dans **Paramètres → Vie privée et sécurité →
   Certificats → Afficher les certificats**
2. Onglet **Autorités → Importer**
3. Sélectionner le fichier `ca.crt` (fourni par l'administrateur du
   projet, ou récupérable temporairement depuis
   `https://www7.starfleet.lan/ca.crt` si publié par le serveur)
4. Cocher **"Faire confiance à cette autorité de certification pour
   identifier des sites web"**
5. Valider

---

## 8. Accès aux services

Une fois les deux VM démarrées et le réseau opérationnel, les services
suivants sont accessibles depuis le navigateur de la VM Cliente :

| Service | URL | Description |
|---|---|---|
| Site web PHP 8 | `https://www8.starfleet.lan` | Site de test en PHP 8.3 |
| Site web PHP 7 | `https://www7.starfleet.lan` | Site de test en PHP 7.4 |
| phpMyAdmin | `https://php.starfleet.lan` | Administration de la base MariaDB |
| Administration système | `https://admin.starfleet.lan` | Interface Cockpit |

### Identifiants

Les identifiants d'accès (comptes système, MariaDB, LDAP) sont transmis
séparément par l'administrateur du projet et ne figurent pas dans ce
document pour des raisons de sécurité.

### Accès FTP

Le serveur FTP est accessible en **FTP explicite avec TLS** :

- **Hôte** : `www7.starfleet.lan` (ou l'IP `192.168.10.1`)
- **Port** : 21
- **Chiffrement** : FTP explicite via TLS obligatoire
- **Répertoire exposé** : chrooté sur le dossier web du site
  (l'utilisateur ne peut pas sortir de ce dossier)

---

## 9. Arrêt propre des VM

```bash
virsh shutdown holodeck-client
virsh shutdown holodeck-serveur
```

Attendre la fin de l'extinction avant de fermer virt-manager ou d'éteindre
l'hôte.

---

## 10. Dépannage rapide

| Problème | Piste de vérification |
|---|---|
| La VM Cliente ne reçoit pas d'IP | Vérifier qu'elle est bien reliée au réseau `starfleet-lan` (et non au réseau `default`/NAT) dans les paramètres réseau de la VM |
| Le navigateur affiche une alerte de sécurité | Importer le certificat `ca.crt` dans le magasin de confiance du navigateur (voir section 7) |
| Un site ne répond pas | Vérifier que la VM Serveur est bien démarrée et que les services concernés tournent (`systemctl status nginx`, `systemctl status mariadb`, etc.) |
| `resolv.conf` pointe vers le mauvais DNS | Voir la remarque de la section 6 concernant l'attribut immuable du fichier |
