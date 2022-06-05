# Setup guide for the FossHost server

- <https://www.cloudbooklet.com/how-to-install-lamp-apache-mysql-php-on-debian-11/>

## System software

```
apt install -y curl git emacs mercurial screen vim zip
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

## Apache LAMP stack

```
apt install -y apache2 mariadb-server php php-mysql libapache2-mod-php php-xml php-mbstring
a2enmod rewrite
a2enmod ssl
a2enmod cgi
```

## CertBot

```
apt install -y snapd
snap install core
snap refresh core
snap install --classic certbot
```

## MediaWiki

<https://www.mediawiki.org/wiki/Manual:Running_MediaWiki_on_Debian_or_Ubuntu>

```
apt install -y php-apcu php-intl imagemagick inkscape php-gd php-cli php-curl php-bcmath

mysql_secure_installation  # Set root password, follow the steps
mysql -u root -p           # Setup like described in the installation manual.
                           # Check LocalSettings.php for data.

cd /var/www/wiki.octave.org
wget https://releases.wikimedia.org/mediawiki/1.35/mediawiki-1.35.6.tar.gz
```
Install following skins and plugins:
- https://www.mediawiki.org/wiki/Skin:Minerva_Neue
- https://www.mediawiki.org/wiki/Extension:FontAwesome
- https://www.mediawiki.org/wiki/Extension:Math
- https://www.mediawiki.org/wiki/Extension:MobileFrontend
- https://www.mediawiki.org/wiki/Extension:PageNotice

Import `mysqldump`-backup back to database and run `maintenence/update.php`.

## hg.octave.org - Mercurial repositories

```
cp /usr/share/doc/mercurial-common/examples/hgweb.wsgi /var/www/hg.octave.org/
```
