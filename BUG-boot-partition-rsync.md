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

## Analyse de la cause

La cause exacte n'a pas été confirmée. Hypothèses par ordre de probabilité :

### 1. Incompatibilité des flags rsync avec FAT32 (plus probable)

`rsync_options` inclut `-A` (ACLs) et `-X` (xattrs). FAT32 ne supporte ni
les ACLs POSIX ni les attributs étendus. Rsync pourrait échouer à lire
les attributs de la source FAT32 et abandonner silencieusement avec rc=0.

Le flag `-o` (preserve owner) est également problématique sur FAT32.

### 2. Problème de cache noyau après mkfs

Après `mkfs.vfat /dev/sda1` suivi d'un montage immédiat, le noyau pourrait
présenter une vue incohérente de la partition. Rsync comparerait alors des
métadonnées corrompues et conclurait qu'il n'y a rien à transférer.

### 3. Granularité des timestamps FAT32

FAT32 a une granularité de 2 secondes sur les timestamps. Combiné avec `-t`
(preserve times) et `-W` (whole-file), rsync pourrait considérer les fichiers
comme identiques si la comparaison de timestamps est faussée.

## Comment reproduire pour diagnostic

```bash
# Vérifier que /boot/firmware est montée
mount | grep boot

# Si montée, lancer le clone en debug
rpi-clone sda -f --debug

# Observer dans /var/log/rpi-clone-debug.log :
# - "p1: fs=fat32  mounted=/boot/firmware" → chemin rsync (bug potentiel)
# - "p1: fs=fat32  mounted=" → chemin dd (fiable)
```

Ajouter `--stats` au rsync en debug (déjà fait en v3.2.0) permet de voir si
rsync détecte les fichiers source :
```
Number of files: 72       ← rsync voit les fichiers
Number of regular files transferred: 0   ← mais ne transfère rien → bug confirmé
```

## Fix proposé (non encore implémenté)

Dans le mode INITIALIZE, toujours utiliser `dd` pour la partition boot (p1),
même si elle est montée, plutôt que mkfs + rsync. Modifier le bloc autour de
la ligne 1610 du script :

```bash
# Remplacer le chemin mkfs-seulement par dd direct :
if [ "${src_mounted_dir[p]}" == "$boot_mount" ] && ((p == 1))
then
    printf "  => dd if=${src_device[$p]} of=$dst_dev bs=1M ..."
    dd if=${src_device[$p]} of=$dst_dev bs=1M &>> $tmp_out
    # Pas besoin de mkfs ni de rsync ultérieur — src_sync_part[p]=0
    src_sync_part[p]=0
```

Ce changement uniformise le comportement : la partition boot est toujours
copiée par `dd` en mode INITIALIZE, indépendamment de son état de montage.
