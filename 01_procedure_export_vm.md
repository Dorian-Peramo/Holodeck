# Procédure d'exportation des VM — Projet Holodeck

Ce document décrit comment exporter les deux machines virtuelles du projet
(VM Serveur et VM Cliente) créées sous **QEMU/KVM** avec **virt-manager**,
afin de pouvoir les transmettre ou les réinstaller sur une autre machine.

## 1. Pré-requis

- Accès à l'hôte où tournent les VM (avec les droits nécessaires sur
  `libvirt`)
- Les deux VM doivent être **éteintes** avant l'export, pour éviter toute
  corruption des disques

Éteindre proprement une VM depuis virt-manager : sélectionner la VM →
bouton "Arrêter" (ou `shutdown -h now` dans la VM elle-même).

## 2. Identifier les fichiers de chaque VM

Chaque VM QEMU/KVM est composée de deux éléments à exporter :

1. **La définition XML** de la VM (configuration : RAM, CPU, cartes
   réseau, disques attachés…)
2. **Le(s) disque(s) virtuel(s)** au format `.qcow2`

### Lister les VM existantes

```bash
virsh list --all
```

### Récupérer le nom exact de chaque VM

Notez les noms affichés (ex. `holodeck-serveur`, `holodeck-client`).

## 3. Exporter la définition XML de chaque VM

```bash
virsh dumpxml holodeck-serveur > holodeck-serveur.xml
virsh dumpxml holodeck-client > holodeck-client.xml
```

## 4. Localiser les fichiers disque (.qcow2)

```bash
virsh domblklist holodeck-serveur
virsh domblklist holodeck-client
```

Cette commande affiche le chemin complet de chaque disque virtuel
(généralement dans `/var/lib/libvirt/images/`).

## 5. Copier les disques vers le dossier d'export

```bash
mkdir -p ~/export-holodeck
cp /var/lib/libvirt/images/holodeck-serveur.qcow2 ~/export-holodeck/
cp /var/lib/libvirt/images/holodeck-client.qcow2 ~/export-holodeck/
cp holodeck-serveur.xml ~/export-holodeck/
cp holodeck-client.xml ~/export-holodeck/
```

## 6. (Optionnel) Compresser le disque pour réduire sa taille

Les fichiers `.qcow2` contiennent souvent de l'espace non utilisé. Pour
réduire leur taille avant transfert :

```bash
qemu-img convert -O qcow2 -c holodeck-serveur.qcow2 holodeck-serveur-compresse.qcow2
qemu-img convert -O qcow2 -c holodeck-client.qcow2 holodeck-client-compresse.qcow2
```

## 7. Archiver le tout

```bash
cd ~/export-holodeck
tar -czvf holodeck-export.tar.gz *.xml *.qcow2
```

L'archive `holodeck-export.tar.gz` contient tout le nécessaire pour
réimporter les deux VM sur une autre machine (voir la notice
d'installation et d'utilisation pour la procédure d'import).

## 8. Vérification

Avant de transmettre l'archive, vérifiez son intégrité :

```bash
tar -tzvf holodeck-export.tar.gz
```

La commande doit lister les 4 fichiers (2 `.xml` + 2 `.qcow2`) sans
erreur.
