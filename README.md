# Megatec based UPS on Home Assistant

The following tutorial depicts my adventure in making a [Megatec protocol](https://linux.die.net/man/8/megatec) based UPS working on Home Assistant either by compiling [NUT](https://github.com/networkupstools/nut) from source or using this forked docker image.

![UPS Home Assistant](https://i.imgur.com/KgP4tDF.png=250x250)

I bought a cheap UPS in Portugal from a lesser known brand called "Eurotech", it came with some installation cd with a program called UPSmart which only works when it is plugged to a computer running windows:

<p float="left">
  <img src="https://i.imgur.com/1ApNwFm.jpg" width=45% height=45%>
  <img src="https://i.imgur.com/GasrMR5.png" width=42% height=42%>
</p>

I was very happy when i found out about the [Network UPS Tools](https://github.com/networkupstools/nut) and i could finally connect my UPS to my Raspberry Pi running Home Assistant. But things were not so simple.

## The problem

After connecting the USB cable, running a simple ``` lsusb ``` should have given me the information that i needed about the Product Id and Vendor Id:

![lsusb](https://i.imgur.com/QPQLsQN.png)

but the manufacturer didn't bothered to write it's real IDs and we have a bogus "Fry's Electronics MEC0003". After searching some [open issues](https://github.com/networkupstools/nut/issues?q=is%3Aissue+is%3Aopen+fry%27s+electronics) regarding the topic, i noticed that i needed to grab a newer version of NUT (version 2.8) which was not available on package managers. In this readme i provide instructions in how to build NUT from source or use this forked docker image for your purposes. This fork allows to set extra fields like ``` subdriver ```, ``` protocol ```, ``` product ```, ``` langid_fix ``` and ``` norating ```.
 
## Build NUT from source

I used a pristine version of the Raspberry Pi OS (Bullseye) to build NUT. Other systems may vary.

### Installing the requirements

```
sudo apt-get update -y
sudo apt-get install -y \
    ccache time git \
    git python2.7 perl curl \
    make autoconf automake libltdl-dev libtool-bin libtool \
    valgrind \
    cppcheck \
    pkg-config \
    gcc g++ clang \
    libcppunit-dev \
    libssl-dev libnss3-dev \
    augeas-tools libaugeas-dev augeas-lenses \
    libusb-dev libusb-1.0-0-dev \
    libi2c-dev \
    libmodbus-dev \
    libsnmp-dev \
    libpowerman0-dev \
    libfreeipmi-dev libipmimonitoring-dev \
    libavahi-common-dev libavahi-core-dev libavahi-client-dev \
    libneon27-gnutls-dev \
    libgd-dev \
    asciidoc source-highlight python3-pygments dblatex
```

### Compiling NUT and installing

I downloaded the most recent version of nut when this tutorial was written (2.8.0):
```
wget https://github.com/networkupstools/nut/releases/download/v2.8.0/nut-2.8.0.tar.gz
tar xzvf nut-2.8.0.tar.gz
cd nut-2.8.0
```

And used the following command to configure what i wanted:

```
./configure --with-all --with-user=pi --with-group=pi --sysconfdir=/etc/nut --bindir=/usr/bin --sbindir=/usr/sbin
```
> You can find more information about the selected options [here](https://networkupstools.org/historic/v2.8.0/docs/user-manual.pdf). The command above installs all drivers and files (which may not be needed), sets the user and group (you should use yours if it is not pi) and sets the system directories for the configurations and binaries

Compile and install:
```
make
sudo make install
```

### Getting the correct configuration

Edit the necessary configuration files present on the sysconfdir
```
sudo nano /etc/nut/ups.conf
```
The content of the file will be something like this:
```
maxretry=3
pollinterval=1

[eurotech]
    driver = "nutdrv_qx"
    subdriver = "hunnox"
    protocol = q1
    port = "auto"
    vendorid = "0001"
    productid = "0000"
    product = "MEC0003"
    norating
    novendor
    langid_fix = "0x0409"
    override.battery.packs = 1
```
Your main challenge will be finding out which combination of configurations will work for you. Mine was a result of checking the [open issues](https://github.com/networkupstools/nut/issues?q=is%3Aissue+is%3Aopen+fry%27s+electronics) and the [driver documentation](https://networkupstools.org/docs/man/) for drivers like [blazer_ser](https://networkupstools.org/docs/man/blazer_ser.html), [blazer_usb](https://networkupstools.org/docs/man/blazer_usb.html), [nut_atcl_usb](https://networkupstools.org/docs/man/nutdrv_atcl_usb.html) or [nutdrv_qx](https://networkupstools.org/docs/man/nutdrv_qx.html) and checking their subdrivers, protocols and langid_fixes. **Don't ask me directly what will work in your UPS, it will be your own adventure.**

You can debug your configuration and check if you can read the results of your UPS simply by running:
```
/usr/bin/nutdrv_qx -a eurotech -DDDDDD
```
> The command needs to call the driver that you are using with the name of the UPS that you configured on ``` /etc/nut/ups.conf ```

You can also try to autoscan to help you fill this file and try your luck with the command
```
nut-scanner -U
```
but the configurations that i got didn't work. When you find the right configuration you are ready to set your NUT server.

### Setting up the server
Edit the file:
```
sudo nano /etc/nut/upsmon.conf
```
and add:
```
RUN_AS_USER root
MONITOR eurotech@localhost 1 admin secret master
```
``` eurotech ``` should be the name of your configured UPS, ``` admin ``` and ``` secret ``` should be your username and password

Edit the file:
```
sudo nano /etc/nut/upsd.conf
```
and add:
```
LISTEN 0.0.0.0 3493
```
Edit the file:
```
sudo nano /etc/nut/nut.conf
```
and add:
```
MODE=netserver
```
Edit the file:
```
sudo nano /etc/nut/upsd.users
```
and fill with your username and password:
```
[monuser]
	password = secret
	admin master
```

Create some missing needed folders:
```
sudo mkdir -p /var/state/ups
sudo chmod 0770 /var/state/ups
sudo chown root:pi /var/state/ups
```

And enable some services:
```
sudo systemctl enable nut-server.service
sudo systemctl enable nut-monitor.service
sudo systemctl enable nut-driver-enumerator.service
sudo systemctl enable nut.target
```

Reboot at the end and test it by running
```
upsc eurotech@localhost
```

## Installing using docker
Simply run the command with your desired options:
```
docker run -d \
-p 3493:3493 \
-e DESCRIPTION='eurotech' \
-e DRIVER='nutdrv_qx' \
-e SUBDRIVER='hunnox' \
-e PROTOCOL='q1' \
-e PRODUCT='MEC0003' \
-e LANGID_FIX='0x0409' \
-e NORATING='true' \
-e NOVENDOR='true' \
-e OVERRIDE_BATTERY_PACKS='1' \
-e VENDORID='0001' \
-e PRODUCTID='0000' \
-e POLLINTERVAL='1' \
-e API_USER='admin' \
-e API_PASSWORD='secret' \
--restart always \
--name nut-upsd \
 --privileged \
brunoalves/nut-upsd:latest
```

## Adding to Home Assistant

Add a new integration and select NUT, add the ip of the server where you installed it, the port and the desired username and password. All the information will be available as entities.

<p float="left">
  <img src="https://i.imgur.com/T4O3V5t.png" width=45% height=45%>
  <img src="https://i.imgur.com/Zzff8GV.png" width=42% height=42%>
</p>


## Practical Docker Tools

[![](https://gitlab.com/instantlinux/docker-tools/badges/master/pipeline.svg)](https://gitlab.com/instantlinux/docker-tools/pipelines "pipelines")

Kubernetes is hard--or is it? This repo is a collection of
multi-platform images and container resource definitions for managing
a software-dev organization using Kubernetes. These tools make it
easy. Contents:

| Directory | Description |
| --------- | ----------- |
| ansible | build your own cluster (Kubernetes or Swarm) |
| images | images which are published to Docker Hub |
| k8s | container resources in kubernetes yaml format |
| lib/build | build makefile and tools |
| services | non-clustered docker-compose services |
| ssl | PKI certificate tools (deprecated by k8s) |
| stacks | container resources in docker-compose format |

Find images at [docker hub/instantlinux](https://hub.docker.com/r/instantlinux/). Each image is scanned by [trivy](https://trivy.dev/) to ensure they contain no known CVE vulnerabilities before promotion to Docker Hub.

Find a lot more details about the Kubernetes bare-metal installer in [k8s/README](k8s/README.md).

### Kubernetes capabilities

The cluster-deployment tools here include helm charts and ansible playbooks to spin up bare-metal or VM master/worker nodes, and a Makefile to add several additional features.

* Direct-attached SSD local storage pools
* Dashboard
* Non-default namespace with its own service account (full permissions
  within namespace, limited read-only in kube-system namespaces)
* Keycloak for OpenID / OAuth2 user authentication / authorization
* Helm3
* Mozilla [sops](https://github.com/mozilla/sops/blob/master/README.rst) with encryption (to keep credentials in local git repo)
* Encryption for internal etcd
* MFA using [Authelia](https://github.com/clems4ever/authelia) and Google Authenticator
* Calico or flannel networking
* ingress-nginx
* Local-volume sync
* Pod security policies
* Automatic certificate issuing/renewal with Letsencrypt
* PostgreSQL-operator from CrunchyData

### Resource definitions

**Developer infrastructure**

| Service | Version | Notes |
| --- | --- | --- |
| artifactory | ** | binary repo |
| gitlab | ** | CI server and git repo |
| admin-git | [![](https://img.shields.io/docker/v/instantlinux/git-pull?sort=date)](https://hub.docker.com/r/instantlinux/git-pull "Version badge") | sync git repo across swarm |
| jira | ** | ticket tracking |
| mariadb-galera | [![](https://img.shields.io/docker/v/instantlinux/mariadb-galera?sort=date)](https://hub.docker.com/r/instantlinux/mariadb-galera "Version badge") | automatic cluster setup|
| nexus | ** | binary repo with docker registry |
| python-builder | [![](https://img.shields.io/docker/v/instantlinux/python-builder?sort=date)](https://hub.docker.com/r/instantlinux/python-builder "Version badge") | CI testing for python|
| python-wsgi | [![](https://img.shields.io/docker/v/instantlinux/python-wsgi?sort=date)](https://hub.docker.com/r/instantlinux/python-wsgi "Version badge") | WSGI runtime for python flask apps|
| wordpress | ** | |

**Networking and support**

| Service | Version | Notes |
| --- | --- | --- |
| authelia | ** | single-signon multi-factor auth |
| cloud | ** | nextcloud, private sync like Apple iCloud |
| data-sync | [![](https://img.shields.io/docker/v/instantlinux/data-sync?sort=date)](https://hub.docker.com/r/instantlinux/data-sync "Version badge") | poor-man's SAN for persistent storage |
| duplicati | [![](https://img.shields.io/docker/v/instantlinux/duplicati?sort=date)](https://hub.docker.com/r/instantlinux/duplicati "Version badge") | backups |
| ez-ipupdate | [![](https://img.shields.io/docker/v/instantlinux/ez-ipupdate?sort=date)](https://hub.docker.com/r/instantlinux/ez-ipupdate "Version badge") | Dynamic DNS client |
| haproxy-keepalived | [![](https://img.shields.io/docker/v/instantlinux/haproxy-keepalived?sort=date)](https://hub.docker.com/r/instantlinux/haproxy-keepalived "Version badge") | load balancer |
| guacamole | ** | authenticated remote-desktop server |
| logspout | ** | central logging for Docker |
| mysqldump | [![](https://img.shields.io/docker/v/instantlinux/mysqldump?sort=date)](https://hub.docker.com/r/instantlinux/mysqldump "Version badge") | per-database alternative to xtrabackup |
| nagios | [![](https://img.shields.io/docker/v/instantlinux/nagios?sort=date)](https://hub.docker.com/r/instantlinux/nagios "Version badge") | Nagios Core v4 for monitoring |
| nagiosql | [![](https://img.shields.io/docker/v/instantlinux/nagiosql?sort=date)](https://hub.docker.com/r/instantlinux/nagiosql "Version badge") | NagiosQL for configuring Nagios Core v4 |
| nut-upsd | [![](https://img.shields.io/docker/v/instantlinux/nut-upsd?sort=date)](https://hub.docker.com/r/instantlinux/nut-upsd "Version badge") | Network UPS Tools |
| openldap | [![](https://img.shields.io/docker/v/instantlinux/openldap?sort=date)](https://hub.docker.com/r/instantlinux/openldap "Version badge") | OpenLDAP authentication server |
| restic | ** | backups |
| rsyslogd | [![](https://img.shields.io/docker/v/instantlinux/rsyslogd?sort=date)](https://hub.docker.com/r/instantlinux/rsyslogd "Version badge") | logger in a 13MB image |
| samba | [![](https://img.shields.io/docker/v/instantlinux/samba?sort=date)](https://hub.docker.com/r/instantlinux/samba "Version badge") | file server |
| samba-dc | [![](https://img.shields.io/docker/v/instantlinux/samba-dc?sort=date)](https://hub.docker.com/r/instantlinux/samba-dc "Version badge") | Active-Directory compatible domain controller |
| [secondshot](https://github.com/instantlinux/secondshot) | [![](https://img.shields.io/docker/v/instantlinux/secondshot?sort=date)](https://hub.docker.com/r/instantlinux/secondshot "Version badge") | rsnapshot-based backups |
| splunk | ** | the free version |

**Email**

| Service | Version | Notes |
| --- | --- | --- |
| blacklist | [![](https://img.shields.io/docker/v/instantlinux/blacklist?sort=date)](https://hub.docker.com/r/instantlinux/blacklist "Version badge") | a local rbldnsd for spam control |
| dovecot | [![](https://img.shields.io/docker/v/instantlinux/dovecot?sort=date)](https://hub.docker.com/r/instantlinux/dovecot "Version badge") | imapd server |
| postfix | [![](https://img.shields.io/docker/v/instantlinux/postfix?sort=date)](https://hub.docker.com/r/instantlinux/postfix "Version badge") | compact general-purpose image in 11MB |
| postfix-python | [![](https://img.shields.io/docker/v/instantlinux/postfix-python?sort=date)](https://hub.docker.com/r/instantlinux/postfix-python "Version badge") | postfix with spam-control scripts |
| rainloop | ** | webmail imapd-client server |
| spamassassin | [![](https://img.shields.io/docker/v/instantlinux/spamassassin?sort=date)](https://hub.docker.com/r/instantlinux/spamassassin "Version badge") | spam control daemon |

**Entertainment**

| Service | Version | Notes |
| --- | --- | --- |
| davite | [![](https://img.shields.io/docker/v/instantlinux/davite?sort=date)](https://hub.docker.com/r/instantlinux/davite "Version badge") | party-invites manager like eVite |
| mt-daapd | [![](https://img.shields.io/docker/v/instantlinux/mt-daapd?sort=date)](https://hub.docker.com/r/instantlinux/mt-daapd "Version badge") | iTunes server |
| mythtv-backend | [![](https://img.shields.io/docker/v/instantlinux/mythtv-backend?sort=date)](https://hub.docker.com/r/instantlinux/mythtv-backend "Version badge") | MythTV backend |
| weewx | [![](https://img.shields.io/docker/v/instantlinux/weewx?sort=date)](https://hub.docker.com/r/instantlinux/weewx "Version badge") | Weather station software (Davis VantagePro2 etc.) |
| wxcam-upload | [![](https://img.shields.io/docker/v/instantlinux/wxcam-upload?sort=date)](https://hub.docker.com/r/instantlinux/wxcam-upload "Version badge") | Upload webcam images to Weather Underground |

### Credits

Thank you to the following contributors!

* [Mike Neir](https://github.com/d0ct0rvenkman)
* [Chad Hedstrom](https://github.com/Hadlock) - [personal site](http://nearlydeaf.com/)
* [Sean Mollet](https://github.com/SeanMollet)
* [Juan Manuel Carrillo Moreno](https://github.com/inetshell) - [personal site](https://wiki.inetshell.mx/)
* [nicxvan]( https://github.com/nicxvan)
* [Frank Riley](https://github.com/fhriley)
* [Devin Bayer](https://github.com/akvadrako)
* [Daniel Muller](https://github.com/DanielMuller)
* [Brian Hechinger](https://github.com/bhechinger)

Contents created 2017-23 under [Apache 2.0 License](https://www.apache.org/licenses/LICENSE-2.0) by Rich Braun.
