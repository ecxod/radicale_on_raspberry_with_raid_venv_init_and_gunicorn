# Radicale on Raspberry with init.d with gunicorn

I will try to show the best way to install a Radicale server on a Raspberry on a Devuan without systemd.

We know that the "hard disk" of a Raspberry is a small USB sdcard, which makes the Raspberry a dangerous thing to store anything important on. That's why the Raspberry cries out for a USB raid system with many thousands of gigabytes or even terabytes.
Which raid system is another matter, but we don't want to make any such recommendations here.

## The raid is mounted via fstab

### cat /etc/fstab
```sh
proc                                            /proc   proc    defaults              0    0
PARTUUID=2184f6ac-01                            /boot   vfat    defaults              0    2
PARTUUID=2184f6ac-02                            /       ext4    defaults,acl,noatime  0    1
PARTUUID=6c4bc44f-0629-4441-aca3-6e275d622b7e   /raid   ext4    defaults,acl          0    2

# a swapfile is not a swap partition, no line here
# use  dphys-swapfile swap[on|off]  for that
```

#### So I have a user `radicale` in a venv whose home directory is on the raid in `/raid/home/radicale`

```sh
black:~# adduser --system --disabled-password --shell=/bin/bash --home /raid/home/radicale --group radicale
Lege Systembenutzer »radicale« (UID 114) an ...
Lege neue Gruppe »radicale« (GID 120) an ...
Lege neuen Benutzer »radicale« (UID 114) mit Gruppe »radicale« an ...
Erstelle Home-Verzeichnis »/raid/home/radicale« ...
```
#### next we create the venv

```sh
black:~# su - radicale
radicale@black:~$
radicale@black:~$ python3 -m venv virtual
radicale@black:~$ source virtual/bin/activate
(virtual) radicale@black:~$ cd virtual/
(virtual) radicale@black:~/virtual$
(virtual) radicale@black:~/virtual$ which python
/raid/home/radicale/virtual/bin/python
(virtual) radicale@black:~/virtual$
```

#### Installing PIP into venv

```sh
(virtual) radicale@black:~/virtual$ python3 -m pip install --upgrade pip
Looking in indexes: https://pypi.org/simple, https://www.piwheels.org/simple
Requirement already satisfied: pip in ./lib/python3.9/site-packages (20.3.4)
Collecting pip
  Downloading https://www.piwheels.org/simple/pip/pip-23.3.2-py3-none-any.whl (2.1 MB)
     |████████████████████████████████| 2.1 MB 652 kB/s
Installing collected packages: pip
  Attempting uninstall: pip
    Found existing installation: pip 20.3.4
    Uninstalling pip-20.3.4:
      Successfully uninstalled pip-20.3.4
Successfully installed pip-23.3.2
(virtual) radicale@black:~/virtual$ python3 -m pip --version
pip 23.3.2 from /raid/home/radicale/virtual/lib/python3.9/site-packages/pip (python 3.9)
(virtual) radicale@black:~/virtual$

# DEBUGGING: 
/raid/home/radicale/virtual/bin/python3: No module named pip
(virtual) radicale@black:~/virtual$python -m ensurepip --default-pip

```
#### Installing radicale in the venv

```sh
(virtual) radicale@black:~/virtual$ python3 -m pip install --upgrade radicale
[...]
Installing collected packages: passlib, six, defusedxml, python-dateutil, vobject, radicale
Successfully installed defusedxml-0.7.1 passlib-1.7.4 python-dateutil-2.8.2 radicale-3.1.8 six-1.16.0 vobject-0.9.6.1
(virtual) radicale@black:~/virtual$
```
#### create the storage

```sh
(virtual) radicale@black:~/virtual$ python -m radicale --storage-filesystem-folder=~/virtual/radicale/collections
```
#### create (local)conifig for Radicale

```sh
(virtual) radicale@black:~/virtual$ mkdir -p ~/.config/radicale && touch ~/.config/radicale/config

(virtual) radicale@black:~/virtual$ cat ~/.config/radicale/config

[server]
# Bind all addresses
#hosts = 0.0.0.0:5232, [::]:5232
# only ipv4
hosts = 192.168.178.6:5232
# max_connections = 8
# max_content_length = 100000000
# timeout = 30
# ssl = false
# certificate = /etc/ssl/radicale.cert.pem
# key = /etc/ssl/radicale.key.pem
# certificate_authority =
# request = utf-8
# stock = utf-8

[auth]
# type
# none : Just allows all usernames and passwords.
# htpasswd : Use an Apache htpasswd file to store usernames and passwords.
# remote_user : Takes the username from the REMOTE_USER environment variable and disables HTTP authentication. This can be used to provide the username from a WSGI server.
# http_x_remote_user : Takes the username from the X-Remote-User HTTP header and disables HTTP authentication. This can be used to provide the username from a reverse proxy.
type = htpasswd

# htpasswd_filename
# Path to the htpasswd file.
htpasswd_filename = ~/.config/radicale/users

# htpasswd_encryption
# The encryption method that is used in the htpasswd file. Use the htpasswd or similar to generate this files.
# Available methods:
# plain : Passwords are stored in plaintext. This is obviously not secure! The htpasswd file for this can be created by hand and looks like:
#  user1:password1
#  user2:password2
# bcrypt : This uses a modified version of the Blowfish stream cipher. It's very secure. The installation of radicale[bcrypt] is required for this.
# md5 : This uses an iterated md5 digest of the password with a salt.
htpasswd_encryption = md5

# delay = 1
# realm = Radicale - Password Required

[rights]
# type
# The backend that is used to check the access rights of collections.
# The recommended backend is owner_only. If access to calendars and address books outside the home directory of users (that's /USERNAME/) is granted,
# clients won't detect these collections and will not show them to the user. Choosing any other method is only useful if you access calendars and address books directly via URL.
# Available backends:
# authenticated : Authenticated users can read and write everything.
# owner_only : Authenticated users can read and write their own collections under the path /USERNAME/.
# owner_write : Authenticated users can read everything and write their own collections under the path /USERNAME/.
# from_file : Load the rules from a file.
# Default: owner_only

# file
# File for the rights backend "from_file". See the Rights section.

[storage]
# type
# The backend that is used to store data.
# Available backends:
# multifilesystem : Stores the data in the filesystem.
# multifilesystem_nolock : The multifilesystem backend without file-based locking. Must only be used with a single process.
# Default: multifilesystem

# filesystem_folder
# Folder for storing local collections, created if not present.
filesystem_folder = ~/virtual/radicale/collections

# max_sync_token_age
# Delete sync-token that are older than the specified time. (seconds)
# Default: 2592000

# hook
# Command that is run after changes to storage. Take a look at the Versioning with Git tutorial for an example.
# Default:

[web]
# type
# The backend that provides the web interface of Radicale.
# Available backends:
# none : Just shows the message "Radicale works!".
# internal : Allows creation and management of address books and calendars.
# Default: internal

[logging]
# level
# Set the logging level.
# Available levels: debug, info, warning, error, critical
# Default: warning

# mask_passwords
# Don't include passwords in logs.
# Default: True

[headers]
# In this section additional HTTP headers that are sent to clients can be specified.
# An example to relax the same-origin policy:
# Access-Control-Allow-Origin = *

```

#### create radicale users

```sh
# Create a new htpasswd file with the user "user1"
(virtual) radicale@black:~/virtual$ htpasswd -c ~/.config/radicale/users user1
New password:
Re-type new password:
```
#### Start radicale manually (not recommended)

```sh
(virtual) radicale@black:~/virtual$ python3 -m radicale &
[1] 15505
```
#### start radicale with gunicorn

```sh
(virtual) radicale@black:~/virtual$ gunicorn -D \
                                             --bind '192.168.178.6:5232' \
                                             --env='RADICALE_CONFIG=~/.config/radicale/config' \
                                             --workers 2 \
                                             radicale
```
#### start radicale with debugging (if the Gods are not your friend)

```sh
(virtual) radicale@black:~/virtual$ gunicorn -D \
                                             --bind '192.168.178.6:5232' \
                                             --env 'RADICALE_CONFIG=~/.config/radicale/config' \
                                             --log-level "debug" # Available debug, info, warning, error, critical
                                             --capture-output 
                                             --error-logfile ~/.log/gunicorn/error.log
                                             --workers 2 \
                                             radicale
```

#### kill radicale manually (not recommended)
```sh
(virtual) radicale@black:~/virtual$ pkill -f "python3 -m radicale"
```

#### kill radicals properly with gunicorn
```sh
(virtual) radicale@black:~/virtual$ pkill -P1 gunicorn
```

#### versioning radicale with git
```venv
(virtual) radicale@black:~/virtual$ cd ~/virtual/radicale/collections
(virtual) radicale@black:~/virtual/radicale/collections$ git init
(virtual) radicale@black:~/virtual/radicale/collections$ git config --global init.defaultBranch master
```

##### then either in `.gitignore` or in `~/virtual/radicale/collections/.git/info/exclude` enter the exceptions:
```git
.Radicale.cache
.Radicale.lock
.Radicale.tmp-*
```

##### then enter the git-hook in the config file in [storage]:
```sh
cat ~/.config/radicale/config

[storage]
hook = git add -A && (git diff --cached --quiet || git commit -m "Changes by "%(user)s)
```


#### create service with init.d

```sh
service radicale start
service radicale stop
service radicale status
```

cat /etc/init.d/radicale
```sh
#! /bin/sh

### BEGIN INIT INFO
# Provides:          radicale
# Required-Start:    $local_fs
# Required-Stop:
# Default-Start:     2 3 4 5
# Default-Stop:
# Short-Description: Provides webDav calendar
# Description: Provides radicale, a webDav server implementation
### END INIT INFO

N=/etc/init.d/radicale

set -e

case "$1" in
  start)
        # Prüfen, ob die Platte richtig eingehängt ist
        if mountpoint -q /raid; then
            # Radicale starten
            echo "started radicale"
            su - radicale -c "
                cd /raid/home/radicale &&
                source virtual/bin/activate &&
                gunicorn -D --bind '192.168.178.6:5232' \
                            --env='RADICALE_CONFIG=~/.config/radicale/config' \
                            --workers 2 radicale
            "
        else
            echo "Verzeichnis nicht eingehängt. Versuche, es neu einzuhängen..."
            # Check if the disk is mounted correctly this time, if not reboot
            mount -a
            echo "Mountpoit eingehongen, bitte erneut versuchen zu starten"
            sleep 2
            if mountpoint -q /raid; then
                # Radicale starten
                echo "starte radicale"
                su - radicale -c "
                    cd /raid/home/radicale &&
                    source virtual/bin/activate &&
                    gunicorn -D --bind '192.168.178.6:5232' \
                                --env='RADICALE_CONFIG=~/.config/radicale/config' \
                                --workers 2 radicale
                "
            else
                echo "Directory still not mounted. Now trying to restart Raspberry..."
                reboot -h now
            fi
        fi
        ;;
  stop)
        # Terminate any hanging processes
        pkill -P1 gunicorn
        sleep 2
        echo "radicale killed"
        ;;
  status)
        # Using netstat to find the PID of the process running on port 5232
        pid=$(netstat -tlp 2>/dev/null | awk '/:5232/ {split($NF, a, "/"); print a[1]}')
        if [ -n "$pid" ] && [ -e /proc/${pid} ]; then
            echo "Radicale is running with PID $pid"
        else
            echo "Radicale is not running"
        fi
        ;;
      *)
        echo "Usage: $N {start|stop|status}" >&2
        exit 1
        ;;
esac

exit 0
```
