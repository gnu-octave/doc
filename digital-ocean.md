# Setup guide for the Digital Ocean server

- <https://www.cloudbooklet.com/how-to-install-lamp-apache-mysql-php-on-debian-11/>

## System software

```
apt install -y bundler curl emacs git htop mercurial screen vim zip
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
a2enmod proxy_http
a2enmod proxy_wstunnel
```

## CertBot

```
apt install -y snapd python3-augeas
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

- <https://www.mercurial-scm.org/wiki/PublishingRepositories#Configuring_Apache>
- <http://510x.se/notes/posts/Install_Mercurial_and_hgweb_on_a_Debian-based_system/>

```
mkdir -p /var/www/hg.octave.org/repos
cp /usr/share/doc/mercurial-common/examples/hgweb.cgi /var/www/hg.octave.org/
cd /var/www/hg.octave.org/
```
Create `hgweb.config` with the following content:
```
[paths]
/ = /var/www/hg.octave.org/repos/*

[web]
encoding = utf-8
allow_archive = zip, gz, bz2

[extensions]
highlight =
```
Further in each repository, e.g. `/var/www/hg.octave.org/repos/web-octave/.hg/hgrc` one can enter information like:
```
[web]
description = Repository moved to https://github.com/gnu-octave/gnu-octave.github.io
```

## Backup

The backup of the Fosshost server is sent daily to the Dreamhost server from jwe.

### Fosshost side

Run the script `/root/bin/backup-to-dreamhost.sh` in a daily cronjob:

```
# Code to backup the wiki

rsync -az --delete /root/backup/octave-wiki.*.gz \
      gnuoctave@backup.octave.org:/home/gnuoctave/backup/fosshost/wiki
rsync -az --delete /var/www/ \
      gnuoctave@backup.octave.org:/home/gnuoctave/backup/fosshost/web
```

### Dreamhost side

Just sending the latest Fosshost server state to Dreamhost is not failsafe enough.
If some bad deletion happens on Fosshost,
after 24h this mistake is synchonized with Dreamhost as well.
Therefore we seek to have a little history of backups with the help of **rsnapshot**.

Compile and install in userspace `./configure --prefix=$HOME`
[rsnapshot](https://github.com/rsnapshot/rsnapshot/blob/master/INSTALL.md).

Create a file `/home/gnuoctave/etc/rsnapshot/fosshost.conf`
and replace **EVERY** space with a tab:
```
config_version 1.2
snapshot_root /home/gnuoctave/backup/fosshost_snapshots/
cmd_cp /bin/cp
cmd_rm /bin/rm
cmd_rsync /usr/bin/rsync
cmd_ssh /usr/bin/ssh
cmd_logger /usr/bin/logger
cmd_du  /usr/bin/du
interval daily 7
interval weekly 4
interval monthly 3
verbose 2
loglevel 4
logfile /home/gnuoctave/backup/logs/rsnapshot.log
rsync_long_args --delete --numeric-ids --delete-excluded
lockfile /home/gnuoctave/backup/rsnapshot.pid
backup /home/gnuoctave/backup/fosshost/ fosshost/
```
Finally test the configuration file:
```
/home/gnuoctave/bin/rsnapshot -c /home/gnuoctave/etc/rsnapshot/fosshost.conf configtest
```
Establish a cronjob:
```
###################################################################
#minute (0-59),                                                   #
#|    hour (0-23),                                                #
#|    |        day of the month (1-31),                           #
#|    |        |       month of the year (1-12),                  #
#|    |        |       |       day of the week (0-6 with 0=Sunday)#
#|    |        |       |       |       commands                   #
###################################################################
10    01       *       *       *        /home/gnuoctave/bin/rsnapshot -c /home/gnuoctave/etc/rsnapshot/fosshost.conf daily
10    02       *       *       0        /home/gnuoctave/bin/rsnapshot -c /home/gnuoctave/etc/rsnapshot/fosshost.conf weekly
10    03       1       *       *        /home/gnuoctave/bin/rsnapshot -c /home/gnuoctave/etc/rsnapshot/fosshost.conf monthly
```

## buildbot.octave.org

```
apt install --no-install-recommends buildbot
pip install buildbot_www buildbot_waterfall_view
```

Edited the `/etc/default/buildbot` file and install the following files in the /var/lib/buildbot/masters/octave directory, all owned by the buildbot user and group:
- `buildbot.tac` - managed at https://hg.octave.org/octave-buildbot
- `master.cfg` - managed at https://hg.octave.org/octave-buildbot
- `octave_buildbot_config.py` - has some password info for each worker so it is not in the mercurial archive with the other config files
- `state.sqlite` (copied from the old digital ocean buildbot server)

Unmask the systemd buildbot service and now starting the buildbot server with
```
/etc/init.d/buildbot restart
```
