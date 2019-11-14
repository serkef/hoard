# Burn and start with new hard drive


## Basic
* Switch to root
```bash
sudo su -
```
* Identify the disk
```bash
# Use following to identify disk's UUSER or ID
fdisk -l
lshw -class disk -short
ll /dev/disk/by-id/
```

```bash
export DISK_ID=...  # FILL the unique disk ID
export DISK_LABEL=...  # SIZE-MANUFACTURE-N
export USER=...  # Used to give ownership
export GROUP=...  # Used to give ownership

export DISK=/dev/disk/by-id/${DISK_ID}  # Entire disk goes here - not partition
export DISK_BADBLOCKS=/root/${DISK_ID}.badblocks
```

## Burn and check
from [here](https://github.com/trapexit/backup-and-recovery-howtos/blob/master/docs/setup_(storage_device).md) and here [here](https://github.com/Spearfoot/disk-burnin-and-testing/blob/master/disk-burnin.sh)

* 1st round of check
```bash
smartctl -t short -C ${DISK}
smartctl -t conveyance -C ${DISK}
smartctl -t long -C ${DISK}
```

* Burn disk **(ERASES ALL DATA! CAREFUL)**. Writes found bad blocks to root/${DISK_ID}
```bash
badblocks -b 4096 -wsv -o ${DISK_BADBLOCKS} ${DISK}
```

* 2nd round of check
```bash
smartctl -t long -C ${DISK}
```

* See results
```bash
smartctl -A ${DISK}
```

## Format and encrypt
* Format disk to ext4 (use badblocks from above) `-T largefile4` only for TV & movies
```bash
mkfs.ext4 -m 0 -T largefile4 -L ${DISK_LABEL} -l /root/${DISK_ID}.badblocks ${DISK}
```

* Setup LUKS
```bash
cryptsetup luksFormat ${DISK}  # Password will be asked
cryptsetup open ${DISK} ${DISK_LABEL}
```

* Format encrypted partition
```bash
mkfs.ext4 /dev/mapper/${DISK_LABEL}
```

* Add new key to encrypted volume file 
```bash
dd if=/dev/urandom of=/home/${USER}/credentials/${DISK_ID} bs=1024 count=4
cryptsetup luksAddKey ${DISK} /home/${USER}/credentials/${DISK_ID}  # Password confirmation will be asked
```

* Add disk to crypttab and fstab to auto-mount
```bash
echo "${DISK_LABEL} ${DISK}       /home/${USER}/credentials/${DISK_ID}        luks" | tee -a /etc/crypttab
echo "/dev/mapper/${DISK_LABEL}     /mnt/${DISK_LABEL}    ext4    defaults        0       2" | tee -a /etc/fstab
mkdir /mnt/${DISK_LABEL}
chown -R ${USER}:${GROUP} /mnt/${DISK_LABEL}
```
