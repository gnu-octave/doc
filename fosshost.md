# Setup guide for the FossHost server

- <https://www.cloudbooklet.com/how-to-install-lamp-apache-mysql-php-on-debian-11/>

## System software

```
apt install -y emacs screen vim
```

## Fix Debian /sbin directory

Add to `/root/.bashrc`:
```
PATH=$PATH:/sbin:
```

## Harddisk setup

```
fdisk     /dev/sdb  # create partition table
mkfs.ext4 /dev/sdb1
vim /etc/fstab      # arrange mountpoint like below
```

```
# lsblk 
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   64G  0 disk 
├─sda1   8:1    0   63G  0 part /
├─sda2   8:2    0    1K  0 part 
└─sda5   8:5    0  975M  0 part [SWAP]
sdb      8:16   0  500G  0 disk 
└─sdb1   8:17   0  500G  0 part /var/www
sr0     11:0    1  378M  0 rom 
```

## Fail2Ban - Firewall

```
apt install -y fail2ban

```

## Apache

```
apt install -y apache2
a2enmod rewrite
a2enmod ssl
```

## CertBot

```
apt install -y snapd
snap install core
snap refresh core
snap install --classic certbot
```

## MediaWiki

```
```
