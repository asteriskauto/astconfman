# Asterisk ConfBridge Manager

**MAINTAINER IS WANTED** If you would like to maintain this repo pls create a new ticket.

This is a WEB based interface for managing Asterisk ConfBridge application.

**Built on Asterisk ConfBridge, Flask, SSE, React.js**

You can request a [new feature](https://github.com/litnimax/astconfman/issues/new) or see current requests and bugs [here](https://github.com/litnimax/astconfman/issues).

### How it works
Flask is used as a WEB server. By default it uses SQLite3 database for storage but other datasources are also supported (see config.py). 

Conference participants are invited using Asterisk call out files. To track participant dial status local channel is used. No AMI/AGI/ARI is used. Everything is built around ```asterisk -rx 'confbridge <...>'``` CLI commands. Asterisk and Flask are supposed to be run on the same server but it's possible to implement remote asterisk command execution via SSH. The software is distributed as as on BSD license. Asterisk resellers can easily implement their own logo and footer and freely redistribute it to own customers (see BRAND_ options in config.py).

### Features

* Private (only for configured participants) and public (guests can join) conferences.
* Muted participant can indicate unmute request. 
* Contact management (addressbook) with import contacts feature.
* Conference recording (always / ondemand, web access to recordings).
* Support for dynamic ConfBridge [profiles](https://wiki.asterisk.org/wiki/display/AST/ConfBridge#ConfBridge-BridgeProfileConfigurationOptions) (any profile option can be set).
* Invite participants from WEB or phone (on press DTMF digit).
* Invite guests on demand by phone number.
* Conference management:
 * Lock / unlock conference;
 * Kick one / all;
 * Mute / unmute one / all 
* Realtime conference events log (enter, leave, kicked, mute / unmute, dial status, etc)
* Asterisk intergrators re-branding ready (change logo, banner, footer)

### Demo

Here is the demo with the folling scenatio:
* Import contacts.
* Add contacts to participants.
* Invite all participants into conference.
* Enter conference from phone.
* Unmute request from phone.
* Invite customer by his PSTN number.
* Enter non-public conference.

[![Demo](http://img.youtube.com/vi/R1EV4D8cFj8/0.jpg)](https://youtu.be/R1EV4D8cFj8 "Demo")

### Installation
#### Requirements

* Asterisk 11, 12 or 13. Only Asterisk 12/13 have confbridge list flags (muted, admin, marked) so Asterisk 11 is supported partially. 
* Python 2.7

On Ubuntu:
```
sudo apt-get install python-pip python-virtualenv python-dev
```

Download the latest version:
```
wget https://github.com/litnimax/astconfman/archive/master.zip
unzip master.zip
mv astconfman-master astconfman
```
Or you can clone the repository with:
```
git clone https://github.com/litnimax/astconfman.git
```
Next steps:
```
cd astconfman
virtualenv env
source env/bin/activate
pip install -r requirements.txt
```
The above will download and install all runtime requirements.

Now you should init database and run the server:
```
cd astconfman
./manage.py init
./run.py
```
Now visit http://localhost:5000/ in your browser.

**Default user/password is admin/admin**. Don't forget to override it.

### Configuration
#### WEB server configuration
Go to *instance* folder and create there config.py file with your local settings. See [config.py](https://github.com/litnimax/astconfman/blob/master/astconfman/config.py) for possible options to override.
Options in config.py file are self-descriptive. 

#### Asterisk configuration
Asterisk must have CURL function compiled and loaded. Check it with
```
*CLI> core show  function CURL
```
You must include files in astconfman/asterisk_etc folder from your Asterisk installation.

Put 
```
#include /path/to/astconfman/asterisk_etc/extensions.conf
```
to your /etc/asterisk/extensions.conf
and
```
#include /path/to/astconfman/asterisk_etc/confbridge.conf
```
to your /etc/asterisk/confbridge.conf.

Open extensions.conf with your text editor and set your settings in *globals* section.

Open /etc/asterisk/asterisk.conf and be sure that 
```
live_dangerously = no
```

### Participant menu
While in the conference participants can use the following DTMF options:

* 1 - Toggle mute / unmute myself.
* 2 - Unmute request.
* 3 - Toggle mute all participants (admin profile only).
* 4 - Decrease listening volume.
* 5 - Reset listening volume.
* 6 - Increase listening volume.
* 7 - Decrease talking volume.
* 8 - Reset talking volume.
* 9 - Increase talking volume.
* 0 - Invite all / not yet connected participants (admin profile only).

### Dialplan for calling external users
```
[confman-dialout]
include => localph
include => extph

[localph]
exten => _XXX,1,Dial(SIP/${EXTEN},60)
exten => _ХXX,2,Set(ret=${CURL(${CONFMAN_HOST}/asterisk/dial_status/${conf_number}/${participant_number}/${DIALSTATUS})})
[extph]
exten => _XXXX.,1,Dial(${DIALOUT_TRUNK1}/${EXTEN},60)
exten => _XXXX.,2,Set(ret=${CURL(${CONFMAN_HOST}/asterisk/dial_status/${conf_number}/${participant_number}/${DIALSTATUS})})
```

### Frequent errors
#### Asterisk monitor path not accessible
```
(env)max@linux:~/astconfman/astconfman$ ./manage.py init
Traceback (most recent call last):
  File "./manage.py", line 7, in <module>
    from app import app, db, migrate
  File "/home/max/astconfman/astconfman/app.py", line 60, in <module>
    from views import asterisk
  File "/home/max/astconfman/astconfman/views.py", line 608, in <module>
    menu_icon_value='glyphicon-hdd'
  File "/home/max/astconfman/env/local/lib/python2.7/site-packages/flask_admin/contrib/fileadmin.py", line 193, in __init__
    raise IOError('FileAdmin path "%s" does not exist or is not accessible' % base_path)
IOError: FileAdmin path "/var/spool/asterisk/monitor/" does not exist or is not accessible

(env)max@linux:~/astconfman/astconfman$ ls -l /var/spool/
итого 16
drwxr-x--- 9 asterisk asterisk 4096 сент.  1 22:20 asterisk
drwxr-xr-x 5 root     root     4096 сент.  1 22:09 cron
lrwxrwxrwx 1 root     root        7 сент.  1 22:05 mail -> ../mail
drwxr-xr-x 2 root     root     4096 апр.  11  2014 plymouth
drwx------ 2 syslog   adm      4096 дек.   4  2013 rsyslog
(env)max@linux:~/astconfman/astconfman$
```
To fix it add user running astwebconf to asterisk group.


### add to cron: */2 * * * * /usr/bin/sqlite3 /etc/asterisk/astconfman/astconfman.db 'select name, phone from contact' > /etc/asterisk/astconfman_contacts
