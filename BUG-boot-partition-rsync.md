# Bug : partition boot FAT32 vide après clone en mode INITIALIZE

## Symptôme

Après un `rpi-clone sda -f`, la partition boot de destination (sda1, FAT32)
est vide ou quasi-vide (4 Ko) alors que la partition root (sda2) est correctement
clonée.

Constaté sur : Raspberry Pi OS Trixie, source mmcblk0 (29 GB), destination sda (8 GB).

## Conditions de déclenchement

Le bug se produit **uniquement** quand la partition boot FAT32 est **montée activement**
sur le système en cours d'exécution, typiquement à `/boot/firmware` (Bookworm/Trixie).

Dans ce cas, `rpi-clone` prend le chemin :

1. INITIALIZE : `mkfs -t vfat -F 32 /dev/sda1` — reformate sda1 (vide)
2. SYNC : monte sda1 sur `/mnt/clone/boot/firmware`, puis :
   ```
   rsync --force -rltWDEHXAgoptx --delete /boot/firmware/ /mnt/clone/boot/firmware
   ```
   → retourne **rc=0 instantanément** sans avoir copié aucun fichier

Résultat : sda1 reste vide, le Pi cloné ne peut pas démarrer.

## Quand le bug n'apparaît PAS

Si la partition boot **n'est pas montée** au moment du clone, le script prend
la branche `dd` (ligne ~1653 du script) :

```bash
dd if=/dev/mmcblk0p1 of=/dev/sda1 bs=1M
```

Cette copie brute fonctionne parfaitement (vérifié : 67 Mo, 423 fichiers).

La détection de montage se voit dans les logs debug :
```
# Bug présent (partition montée) :
DEBUG:   p1: fs=fat32  mounted=/boot/firmware  sync=1  mount_flag=1

# Bug absent (partition non montée) :
DEBUG:   p1: fs=fat32  mounted=  sync=0  mount_flag=0
```

## Analyse de la cause (confirmée v3.3.0)

### Cause racine : conflit de PARTUUID provoqué par sfdisk

Séquence exacte du bug, confirmée par les logs systemd :

```
19:35:22 → systemd : Mounted boot-firmware.mount (/boot/firmware)
19:36:13 → systemd : Unmounted boot-firmware.mount  ← pendant rpi-clone !
19:36:21 → rpi-clone : mkfs p1 et p2 (trop tard)
19:36:30 → rpi-clone pré-sync : source=0 entrées → rsync copie rien
```

**Explication** : `sfdisk` copie la table de partition de mmcblk0 vers sda,
y compris le **Disk ID MBR** — ce qui donne à sda le même PARTUUID que mmcblk0.
Le fstab de ce Pi contient :
```
PARTUUID=92e253d6-01  /boot/firmware  vfat  defaults  0  2
```
Dès que sda1 apparaît avec le même PARTUUID=92e253d6-01, systemd détecte un
conflit et **démonte automatiquement /boot/firmware**. Le changement de Disk ID
(fdisk) arrivait après un `sleep 2`, trop tard.

Une fois /boot/firmware démonté, `ls /boot/firmware/` retourne 0 entrées
(le répertoire vide sur l'ext4 root, pas la FAT32). Rsync croit que la source
est vide et retourne rc=0 sans rien copier.

### Cause secondaire : rsync incompatible avec FAT32 monté

Même sans le démontage systemd, rsync avec les flags `-A` (ACLs) et `-X`
(xattrs) sur une source FAT32 retourne rc=0 sans rien copier. FAT32 ne
supporte pas ces attributs POSIX.

## Fix implémenté en v3.3.0

### 1. Changement de Disk ID immédiatement après sfdisk

Le Disk ID est maintenant changé **avant** `partprobe` et le `sleep 2`,
ce qui ferme la fenêtre de conflit PARTUUID :

```bash
sfdisk --force /dev/sda <<< "$sfd1"
# Immédiatement :
fdisk /dev/sda  # nouveau Disk ID aléatoire
sync
sleep 2
partprobe /dev/sda  # kernel voit le nouveau PARTUUID, pas de conflit
```

### 2. Toujours dd pour la partition boot en mode INITIALIZE

`dd if=/dev/mmcblk0p1 of=/dev/sda1 bs=1M` lit le device brut, indépendamment
de l'état de montage. Plus robuste que mkfs + rsync :
- Immune au démontage systemd
- Immune aux incompatibilités rsync/FAT32
- Copie exacte octet par octet, y compris UUID FAT et métadonnées

### 3. Vérification immédiate après dd

Après le dd, la partition est montée dans un répertoire tmp, les fichiers sont
comptés, et le clone est abandonné si la partition est vide — avant le long
rsync root (~10 min).

## Bug n°2 (introduit par le fix v3.4.0) : PARTUUID jamais corrigé dans cmdline.txt

### Symptôme

Après un clone "réussi" en v3.4.0 (boot copié, vérification OK : 50 fichiers),
le Pi cloné ne démarre pas. Au boot :

```
Gave up waiting for root file system device.
ALERT!  PARTUUID=92e253d6-02 does not exist.  Dropping to a shell!
```

### Cause

`92e253d6-02` est le PARTUUID **de la source** (mmcblk0p2), encore présent
dans `root=PARTUUID=...` de `cmdline.txt` après le clone. En fin de script,
rpi-clone corrige normalement cette référence (section "Fix PARTUUID or
device name references in cmdline.txt and fstab") en remplaçant
`$src_disk_ID` par `$dst_disk_ID` dans :

```bash
cmdline_txt=${clone}${boot_mount}/cmdline.txt   # ex: /mnt/clone/boot/firmware/cmdline.txt
```

Cette correction suppose que la partition boot de destination est **montée**
sous `${clone}${boot_mount}`. Or le fix v3.4.0 copie la partition boot avec
`dd` directement sur le device brut (`/dev/sda1`), sans jamais la monter à cet
endroit : les boucles de sync (pré-sync et post-sync) ignorent volontairement
cette partition puisque `src_sync_part[1]=0`. Résultat : `[ -f $cmdline_txt ]`
est faux, le bloc de correction est silencieusement sauté, et `cmdline.txt`
garde l'ancien PARTUUID de la source → le kernel cible cherche un PARTUUID
qui n'existe pas sur le disque cloné.

### Fix implémenté en v3.5.0

La partition boot copiée par `dd` est désormais explicitement remontée sur
`${clone}${boot_mount}` dans la boucle de pré-sync, **après** que la
partition root a été montée sur `$clone` (pour éviter qu'un montage
prématuré ne soit masqué par le montage ultérieur de root — problème
classique de mounts imbriqués sous Linux). Cela permet à la correction du
PARTUUID de `cmdline.txt` de s'exécuter normalement, comme pour n'importe
quelle partition synchronisée par rsync.

Le nombre de fichiers visibles après ce montage est vérifié (variable
`boot_dd_part`), avec abandon immédiat si la partition apparaît vide.
