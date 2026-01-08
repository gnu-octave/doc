# Setup guide for the Digital Ocean server

- <https://www.cloudbooklet.com/how-to-install-lamp-apache-mysql-php-on-debian-11/>

## System software

```
apt install -y bundler curl emacs git htop mercurial screen vim zip
```

Install Docker https://docs.docker.com/engine/install/debian/#install-using-the-repository.

## Harddisk setup

```
# lsblk 
NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda       8:0    0    50G  0 disk /var/www/nightly.octave.org
vda     254:0    0    80G  0 disk 
|-vda1  254:1    0  79.9G  0 part /
|-vda14 254:14   0     3M  0 part 
`-vda15 254:15   0   124M  0 part /boot/efi
vdb     254:16   0   474K  1 disk
```

## Fail2Ban - Firewall

```
apt install -y fail2ban
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```
Make one change in `/etc/fail2ban/jail.local`:
```diff
[apache-badbots]

# Ban hosts which agent identifies spammer robots crawling the web
# for email addresses. The mail outputs are buffered.
+enabled  = true
```
Modify `diff /etc/fail2ban/filter.d/apache-badbots.conf /etc/fail2ban/filter.d/apache-badbots.conf.orig`:
```diff
+badbotscustom = EmailCollector|WebEMailExtrac|TrackBack/1\.02|sogou music spider|(?:Mozilla/\d+\.\d+ )?Jorgee|Bytespider|Amazonbot|DotBot|PetalBot|bingbot
-badbotscustom = EmailCollector|WebEMailExtrac|TrackBack/1\.02|sogou music spider|(?:Mozilla/\d+\.\d+ )?Jorgee
+failregex = ^hg.octave.org:[0-9]* <HOST> -.*"(GET|POST|HEAD).*HTTP.*".*(?:%(badbots)s|%(badbotscustom)s).*"$
-failregex = ^<HOST> -.*"(GET|POST|HEAD).*HTTP.*"(?:%(badbots)s|%(badbotscustom)s)"$
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
sudo ln -s /snap/bin/certbot /usr/bin/certbot/emi
```

## `/var/www` setup

In general permissions are `chown -R root:www-data *` with 755 (directory) 644 (file) permissions.

```
bugs.octave.org
buildbot.octave.org
docs.octave.org
ftp.octave.org
hg.octave.org
html
  not used
mxe-pkg-src.octave.org
packages.octave.org
  `chmod g+w .` to enable writing of ´cache.txt´-file by php
wiki.octave.org
www.octave.org
```

Create `/var/www/robots.txt` with the following content: 
```
User-agent: Amazonbot
Disallow: /hg.octave.org/

User-agent: *
Disallow: /hg.octave.org/
```

## `/etc/apache/sites-available` setup

- Copy Apache config files from this repo `2022-10-29-digital-ocean-apache2-configs` to `/etc/apache/sites-available` (e.g. SFTP program).
- Enable all sites `a2ensite /etc/apache/sites-available/0*.conf` (symlinks to `/etc/apache/sites-enabled`).
- `systemctl restart apache2`
- Check configuration `apache2ctl -S`.
- Get Let's Encrypt certificates `/snap/bin/certbot --apache`

## Monit

Used for monitoring Apache2 and restart in out-of-memory DDoS attacks.
```
apt install monit
cp /etc/monit/conf-available/apache2 /etc/monit/conf-enabled/
cp /etc/monit/conf-available/mysql   /etc/monit/conf-enabled/mariadb
```
Change `/etc/monit/conf-enabled/mariadb`: replace 3x `/etc/init.d/mysql` with `etc/init.d/mariadb`,
see <https://wpbeaches.com/set-monit-to-monitor-mariadb-on-a-cloudpanel-instance-on-ubuntu-22-04/#mariadb>.

Add to `/etc/monit/monitrc`:
```
set httpd port 2812 and
    use address localhost
    allow localhost
```
Finally:
```
$ systemctl enable monit
$ systemctl start  monit
$ monit summary
Monit 5.27.2 uptime: 16m
┌─────────────────────────────────┬────────────────────────────┬───────────────┐
│ Service Name                    │ Status                     │ Type          │
├─────────────────────────────────┼────────────────────────────┼───────────────┤
│ digital-ocean-1                 │ OK                         │ System        │
├─────────────────────────────────┼────────────────────────────┼───────────────┤
│ mysqld                          │ OK                         │ Process       │
├─────────────────────────────────┼────────────────────────────┼───────────────┤
│ apache                          │ OK                         │ Process       │
├─────────────────────────────────┼────────────────────────────┼───────────────┤
│ mysql_bin                       │ OK                         │ File          │
├─────────────────────────────────┼────────────────────────────┼───────────────┤
│ mysql_rc                        │ OK                         │ File          │
├─────────────────────────────────┼────────────────────────────┼───────────────┤
│ apache_bin                      │ OK                         │ File          │
├─────────────────────────────────┼────────────────────────────┼───────────────┤
│ apache_rc                       │ OK                         │ File          │
└─────────────────────────────────┴────────────────────────────┴───────────────┘
```


## MediaWiki

<https://www.mediawiki.org/wiki/Manual:Running_MediaWiki_on_Debian_or_Ubuntu>

```
apt install -y php-apcu php-intl imagemagick inkscape php-gd php-cli php-curl php-bcmath

mysql_secure_installation  # Set root password, follow the steps
mysql -u root              # Setup like described in the installation manual.
                           # Check LocalSettings.php for data.

cd /var/www/wiki.octave.org
wget https://releases.wikimedia.org/mediawiki/1.35/mediawiki-1.35.6.tar.gz
```
Install following skins and plugins:
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
Create `/var/www/hg.octave.org/hgweb.config` with the following content:
```
[paths]
/ = /var/www/hg.octave.org/repos/*

[web]
encoding = utf-8
#allow_archive = zip, gz, bz2

[extensions]
highlight =
```
Create `/var/www/hg.octave.org/robots.txt` with the following content:
```
User-agent: Amazonbot
Disallow: /

User-agent: *
Disallow: /
```
Further in each repository, e.g. `/var/www/hg.octave.org/repos/web-octave/.hg/hgrc` one can enter information like:
```
[web]
description = Repository moved to https://github.com/gnu-octave/gnu-octave.github.io
```

Install [Pygments](https://pypi.org/project/Pygments/) for source code display:
```
pip install pygments
```

## Cronjobs

```
###################################################################
#minute (0-59),                                                   #
#|    hour (0-23),                                                #
#|    |        day of the month (1-31),                           #
#|    |        |       month of the year (1-12),                  #
#|    |        |       |       day of the week (0-6 with 0=Sunday)#
#|    |        |       |       |       commands                   #
###################################################################
*/5   *        *       *       *        /root/bin/sync-repos.sh
 5    0        *       *       *        /root/bin/backup.sh
```

The script `/root/bin/sync-repos.sh`:
```
#!/bin/bash

OCTAVE_REPO_DIR=/var/www/hg.octave.org/repos/octave/
MXE_REPO_DIR=/var/www/hg.octave.org/repos/mxe-octave/

cd ${OCTAVE_REPO_DIR}
hg pull
chown -R root:www-data ${OCTAVE_REPO_DIR}
chown -R mxe:www-data  ${MXE_REPO_DIR}
```

## Backup

The backup of the Digital Ocean server is sent daily to a private server of jwe.

### Digital Ocean side

Run the script `/root/bin/backup.sh` in a daily cronjob:

```bash
#!/bin/bash

WIKI_DB_BACKUP_FILE=/var/www/wiki.octave.org/backup/octave-wiki.sql.gz

rm -Rf ${WIKI_DB_BACKUP_FILE} 
mysqldump -h hostname -u userid --password=password dbname \
    | gzip > ${WIKI_DB_BACKUP_FILE}

rsync --archive --no-owner --no-group --compress --delete \
    -e "ssh -p PORT -l USER" /var/www/ jwe-server.org:backup/path
```
Lookup the values for `mysqldump` in wiki's `LocalSettings.php` file
and setup public key authentication with JWEs home server.

### JWE home server side

Just sending the latest Digital Ocean server state to the backup server is not failsafe enough.
If some bad deletion happens on Digital Ocean,
after 24h this mistake is synchonized with the backup server as well.
Therefore we seek to have a little history of backups with the help of **rsnapshot**.

Compile and install in userspace `./configure --prefix=$HOME`
[rsnapshot](https://github.com/rsnapshot/rsnapshot/blob/master/INSTALL.md).

Create a file `/home/gnuoctave/etc/rsnapshot/digital-ocean.conf`
and replace **EVERY** space with a tab:
```
config_version 1.2
snapshot_root /home/gnuoctave/backup/digital-ocean-snapshots/
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
logfile /home/gnuoctave/backup/logs/rsnapshot/digital-ocean.log
rsync_long_args --delete --numeric-ids --delete-excluded
lockfile /home/gnuoctave/backup/run/rsnapshot/digital-ocean.pid
backup /home/gnuoctave/backup/digital-ocean/ digital-ocean/
```
Finally test the configuration file:
```
/home/gnuoctave/bin/rsnapshot -c /home/gnuoctave/etc/rsnapshot/digital-ocean.conf configtest
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
10    01       *       *       *        /home/gnuoctave/bin/rsnapshot -c /home/gnuoctave/etc/rsnapshot/digital-ocean.conf daily
10    02       *       *       0        /home/gnuoctave/bin/rsnapshot -c /home/gnuoctave/etc/rsnapshot/digital-ocean.conf weekly
10    03       1       *       *        /home/gnuoctave/bin/rsnapshot -c /home/gnuoctave/etc/rsnapshot/digital-ocean.conf monthly
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


## GitHub synchronization

- The work happens in a folder `/opt/github_repo_sync`.
- Add the public key `cat /root/.ssh/id_rsa.pub` to your GitHub account as "authentication key".
  Currently, @siko1056 has this key assigned to his account.
  If there is a problem: regenerate the key and assign the new key to another repository.

### Octave Packages

Sync https://github.com/gnu-octave/packages to https://github.com/gnu-octave/packages-sandbox.

Clone the original repository and set the sandbox as upstream:
```
cd /opt/github_repo_sync
git clone https://github.com/gnu-octave/packages.git
cd /opt/github_repo_sync/packages
git remote add sandbox git@github.com:gnu-octave/packages-sandbox.git
```

Create a file `/opt/github_repo_sync/package_sandbox_do_update.sh`:
```
#!/bin/bash

cd /opt/github_repo_sync/packages
git pull origin
git push --force sandbox
```

Setup a CronJob `crontab -e`:
```
 0    0        *       *       *        /opt/github_repo_sync/package_sandbox_do_update.sh
```

### Octave Git mirrow

> **WARNING affecting downstream users (forks, etc.)**
> 
> Optimally, the folder `/opt/github_repo_sync/octave` with the hg to git converted Octave repository
> **should NEVER get lost**.
> The conversion tool cannot reproduce the same git commit IDs, thus loosing the original folder
> means a restart of the Octave GitHub mirrow project.

Sync and convert https://www.octave.org/hg/octave to https://github.com/gnu-octave/octave.

Install:
```
pip install git-remote-hg
```

Create a file `/opt/github_repo_sync/octave_do_update.sh`:
```
#!/bin/bash

export LOCKDIR=/tmp/lock-github-sync

if ! mkdir "$LOCKDIR"; then
  echo >&2 "Failed to create lock directory '$LOCKDIR'"
  exit 1
fi

# Remove the lock directory on exit or error
cleanup() {
    if rmdir -- "$LOCKDIR"; then
        echo "Finished"
    else
        echo >&2 "Failed to remove lock directory '$LOCKDIR'"
        exit 1
    fi
}

trap "cleanup" EXIT

echo "Acquired lock, running"

export PATH=$PATH:/usr/local/bin

cd /opt/github_repo_sync/octave
git fetch origin  # https://www.octave.org/hg/octave"
unset git_branches
for hg_branch in $(git branch --remote --list origin/branches/*)
do
    hg_branch=${hg_branch#*/}
    git_branch=${hg_branch#*/}
    git_branches+=( ${git_branch} )
    if ! $(git show-ref --quiet refs/heads/${git_branch})
    then
        git checkout ${hg_branch}
        git branch --move ${git_branch}
    fi
    git checkout ${git_branch}
    git pull --ff-only
done
git push --tags github ${git_branches[@]}  # https://github.com/gnu-octave/octave
```

Setup a CronJob `crontab -e`:
```
*/10  *        *       *       *        /opt/github_repo_sync/octave_do_update.sh
```

Test the CronJob (clean environment):
```
env - /bin/bash /opt/github_repo_sync/octave_do_update.sh
```

Restarting the project (read warning above):
```
cd /opt/github_repo_sync
git clone "hg::https://www.octave.org/hg/octave"

# Use compression to shrink 2 GB to 200 MB.
git gc --aggressive

git remote add github git@github.com:gnu-octave/octave.git

# Default "master" branch is immediately available.
# Create local "stable" and "default" branch.
git checkout branches/stable
git branch -m stable
git checkout branches/default
git branch -m default

# Push with release tags.
git push --tags github default stable
```

Add crontab entry:
```
# crontab -l
###################################################################
#minute (0-59),                                                   #
#|    hour (0-23),                                                #
#|    |        day of the month (1-31),                           #
#|    |        |       month of the year (1-12),                  #
#|    |        |       |       day of the week (0-6 with 0=Sunday)#
#|    |        |       |       |        commands                  #
###################################################################
*/5   *        *       *       *        curl https://savannah.octave.org/api.php?Action=update
```

# savannah.octave.org

Setup the project https://github.com/gnu-octave/SavannahAPI via Docker-compose.
```
cd /opt
git clone https://github.com/gnu-octave/SavannahAPI
cd /var/www/savannah.octave.org/SavannahAPI
docker compose up --detach server-init
docker compose up --detach server
```
The data is stored in a Docker volume.
Find volume path via:
```
docker volume ls
docker volume inspect savannahapi_savannah_data
```
Ensure that the folder and SQLite database `savannah.cache.sqlite`
is owned and writable by user `www-data`:
```
$ ls -al /var/lib/docker/volumes/savannahapi_savannah_data/_data
total 93520
drwxr-xr-x 2 www-data www-data     4096 Apr  9 14:50 .
drwx-----x 3 root     root         4096 Apr  7 15:22 ..
-rw-r--r-- 1 root     root            0 Apr  7 15:21 .gitkeep
-rw-r--r-- 1 www-data www-data 95752192 Apr  9 14:50 savannah.cache.sqlite
```

# nightly.octave.org

This project is basically a single Docker container launched and updated
by a script `update-buildbot-nighty.sh` which mounts two secret files:
1. `authorized_keys`: the public key of the Buildbot worker (currently JWE)
2. `master.cfg`: An adapted version for <https://nightly.octave.org> of
   <https://github.com/gnu-octave/octave-buildbot/blob/68eb6d369fec80e0ed17bc6ca5a186e0ce424d7a/master/defaults/master.cfg>
```
$ ls -al /opt/Buildbot
total 36
drwxr-xr-x 2 root root  4096 Apr  9 15:20 .
drwxr-xr-x 7 root root  4096 Apr  9 14:56 ..
-rw-r--r-- 1 root root   574 Apr  9 15:20 authorized_keys
-rw-r--r-- 1 root root 17188 Apr  9 15:12 master.cfg
-rwxr-xr-x 1 root root   579 Apr  9 15:13 update-buildbot-nighty.sh
```

The script `update-buildbot-nighty.sh` is as follows and is run like a normal BASH script
to launch or update the Buildbot master:
```
#!/bin/bash

docker pull docker.io/gnuoctave/buildbot:latest-master

docker stop octave-buildbot-master
docker rm   octave-buildbot-master

docker create \
  --volume octave-buildbot-master:/buildbot/master \
  --volume $(pwd)/master.cfg:/buildbot/master/master.cfg \
  --volume $(pwd)/authorized_keys:/root/.ssh/authorized_keys \
  --volume /var/www/nightly.octave.org/data:/buildbot/data \
  --publish 8070:8010 \
  --publish 9989:9989 \
  --publish 9988:22 \
  --restart always \
  --name   octave-buildbot-master \
  gnuoctave/buildbot:latest-master

docker start octave-buildbot-master
docker ps -a
```

The Buildbot master state is stored in a Docker volume.
Find volume path via:
```
docker volume ls
docker volume inspect octave-buildbot-master
```
For example:
```
$ ls -al /var/lib/docker/volumes/octave-buildbot-master/_data
total 484844
drwxr-xr-x 3 root root      4096 Apr  9 15:31 .
drwx-----x 3 root root      4096 Apr  7 17:01 ..
drwxr-xr-x 3 root root      4096 Nov  4  2021 app
-rw-r--r-- 1 root root      1024 Apr  1  2021 buildbot.tac
-rw-r--r-- 1 root root   9207831 Apr  9 15:22 http.log
-rw-r--r-- 1 root root     17179 Apr  7 18:04 master.cfg
-rw-r--r-- 1 root root      4051 Oct 14  2022 master.cfg.sample
-rw-r--r-- 1 root root 280834048 Apr  9 15:31 state.sqlite
-rwxr-xr-x 1 root root       642 Jun  7  2021 tidy_up.sh
-rw-r--r-- 1 root root   6329096 Apr  9 15:24 twistd.log
-rw-r--r-- 1 root root         1 Apr  9 15:13 twistd.pid
```
