# LFS - ft_linux project of school42

0. Check that all system requirements are fulfilled with the [script](./0/version-check.sh)
1. Will need a new empty partition of 30 Gb. I used an iSCSI lun from storage system and performed the following steps on it:
* create a partition table (gpt). A partition table is located at the start of a hard drive and it stores data about the size and location of each partition.
```
sudo parted /dev/sda
> mklabel gpt
```
* create partition, which start will be at 1 MB and end at 25 GB.
```
mkpart primary ext4 1MB 25GB
```
* create swap partition, which start will be at 25 GB and end at 100%.
```
mkpart swap linux-swap 25GB 100%
```
* format the primary partitions
```
sudo mkfs.ext4 /dev/sda1
```
* initialize swap partition
```
mkswap /dev/sda2
```

2. Mounting
* edit .bashrc file of current user and root user adding the following line
```
export LFS=/mnt/lfs
```
* create the mount point and mount the LFS file system by running:
```
mkdir -pv $LFS
mount -v -t ext4 /dev/sda1 $LFS
swapon -s # check that the swap partition is recognized
```

