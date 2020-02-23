## [OpenProject](https://www.openproject.org/) on a [RaspberryPi](https://www.amazon.com/Raspberry-Model-2019-Quad-Bluetooth/dp/B07TC2BK1X/ref=sxin_3_ac_d_pm?ac_md=3-0-VW5kZXIgJDc1-ac_d_pm&cv_ct_cx=raspberry+pi+4+4gb&keywords=raspberry+pi+4+4gb&pd_rd_i=B07TC2BK1X&pd_rd_r=e622118f-8bbf-43e5-ba7b-2da482551e9b&pd_rd_w=30pUf&pd_rd_wg=8t2FQ&pf_rd_p=0e223c60-bcf8-4663-98f3-da892fbd4372&pf_rd_r=8PG76Q9PBRF32SW9MH8G&psc=1&qid=1582422684)-based server 


### Background


### Introduction

[OpenProject](https://www.openproject.org/) is a fully features project management toolbox, 
open source and with a free community version, although some premium features are limited to the paid-for cloud & enterprise versions. 
If you want to run your own server, the project suggests a Linux-based server with 4 GB RAM, and supplies various .deb/.rpm packages, or docker images for easy installation.

The newer Raspberry Pi 4 boards are compact and power efficient, and with 4 cores & 4 GB of RAM at least on paper look capable of running OpenProject. This would make for a very power-efficient and compact solution for (at least) small teams or local test installations.  

The downside? OpenProject officially does not support the ARM architecture, so any described installation methods based on .deb/.rpm packages or docker fails miserably. The support forum has a few threads on this dating back several years, but little to no help on making this happen. But if you found this page, your probably know all about that already. 

## Good news: OpenProject runs on ARM (unofficially, at least)!

It's possible to get OpenProject working on a Raspberry Pi. I highly recommend a RPi4 with 4 GB RAM (or a similar board). It might be possible to get by with the 2 GB version and sufficient swap space, but that's something I haven't tested yet, and seems ill advised. During the installation and compilation process, memory usage maxes out at about 3.1 Gb. 

## Status (Feb 2020)

As of now, my OpenProject Raspberry Pi server has an uptime of about 2 weeks, good responsiveness and no issues worth talking about. 

The downside? Consider this installation protocol an 'alpha' version. It takes 4 to 6 hours to go through, and I seem to invariably run into a problem where (independently of which ruby/npm/gem versions I start out with) OpenProject does initially not compile, usually due to a SASS-related problem. After changing npm/ruby versions a couple of times it suddenly works. I am fully aware that this 'bug' makes this a rather rough guide. 

Github doesn't allow me to upload a prepared system image, but if you send me a message I can give a link to a prepared system image with Raspian/OpenProject preinstalled. Just copy that onto a SD card, change the passwords and you are good to go. 

Or help me fix the last few kinks in the protocol. Any hints & suggestions are appreciated. 

## Key issues

* bcrypt 3.1.13 does not build on the Pi, 3.1.12 is fine. See [here](https://github.com/codahale/bcrypt-ruby/issues/201).
* apache configuration
* PostgreSQL user privileges are not set correctly when following the 'official' protocol.

### How to install Openproject on Raspian

This protocol is based on the somewhat outdated manual installation protocol that can be found [here](https://docs.openproject.org/installation-and-operations/installation/manual/).

## Setting up the Raspberry and Raspian Buster

Download a [Raspian Buster Lite](https://www.raspberrypi.org/downloads/raspbian/) system image and [flash](https://www.balena.io/etcher/) it on a microSD card (I'm using a fast 32 GB one, smaller is fine too).

Assuming that access via SSH & the local wifi network is required:

* Create an empty SSH file to enable [shell access](https://www.raspberrypi.org/documentation/remote-access/ssh/) (or hook up the Pi to a keyboard, mouse & display directly). 
* Create an [wpa_supplicant.conf](https://www.raspberrypi.org/documentation/configuration/wireless/headless.md) file with the correct credentials for the local wifi (or hook the Pi up to a LA).
* Check your router for the IP address of the Pi, and use this to log in using your favourite SSH client (Putty, etc)

Raspian standard username // password is: pi // raspberry

Update the system and install necessary system packages, PostgreSQL and the optional memcached package):

```
sudo apt-get update -y
sudo apt-get full-upgrade -y
sudo apt-get install -y zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libgdbm-dev libncurses5-dev automake libtool bison libffi-dev git curl poppler-utils unrtf tesseract-ocr catdoc libxml2 libxml2-dev libxslt1-dev memcached postgresql postgresql-contrib libpq-dev
```

## Expand filesystem
Expand the filesystem to take full advantage of the size of SD card chosen;
Start **raspi-config** and use the first option (**expand filesystem**). 
Quit.


## Increase swap space to 4 GB
Not actually necessary, but possibly useful for the 2 GB board version (not recommended). Make sure to reverse the setting to avoid burning out the memory card by excessive read/write cycles.

```
sudo dphys-swapfile swapoff.
sudo nano /etc/dphys-swapfile 
***find the CONF_SWAPSIZE=100 line and change to 4096 or similar***
Start the swap. sudo dphys-swapfile swapon.

sudo dphys-swapfile swapon.

```

## Set up user accounts

Create the openproject group/user. For the standard installation I set 'openproject' as the password.

```
sudo groupadd openproject
sudo useradd --create-home --gid openproject openproject
sudo passwd openproject #(***pick a password***)
```

Switch to the PostgreSQL system user and create the database user. The official documentation creates a user without privileges, the -sd flags will create a superuser with CREATEDB privileges.

```
sudo su - postgres
createuser -sd openproject
```
Check PostgreSQL users and their privileges. If the CREATE DB privilege is missing, the installation will fail at a later point.
```
psql
\du
```
The output should look like this:

```
postgres=# \du
                                    List of roles
  Role name  |                         Attributes                         | Member of
-------------+------------------------------------------------------------+-----------
 openproject | Superuser, Create role, Create DB                          | {}
 postgres    | Superuser, Create role, Creaboglte DB, Replication, Bypass RLS | {}
```

 
 Create the database and revert to the standard 'pi' user account:
 
 ```
 createdb -O openproject openproject
 exit
 ```

## Preparing software packages

Following the manual installation suggestions, rbenv is used to install Ruby. Whenever possible, I use all four cores to speed up compiling (watch out for the **-j 4** flags). These tasks have to be done as the openproject user, hence the **su openproject** command.  

```
sudo su openproject --login
git clone https://github.com/sstephenson/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.profile
echo 'eval "$(rbenv init -)"' >> ~/.profile
source ~/.profile
git clone https://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build
MAKE_OPTS="-j 4" rbenv install 2.6.5
rbenv rehash
rbenv global 2.6.5

```
Will take 15 minutes. 

rbenv 2.6.5 and 2.7.0 might work. 2.7.0 is too new for OpenProject, unless this version is edited into the OpenProject gem file. Possible, but 2.6.5 should be simpler. 

Check version with
```
ruby --version
```

Following the manual installation suggestions, nodenv is used to install Ruby. Whenever possible, I use all four cores to speed up compiling (watch out for the **-j 4** flags).

```
git clone https://github.com/OiNutter/nodenv.git ~/.nodenv
echo 'export PATH="$HOME/.nodenv/bin:$PATH"' >> ~/.profile
echo 'eval "$(nodenv init -)"' >> ~/.profile
source ~/.profile
git clone git://github.com/OiNutter/node-build.git ~/.nodenv/plugins/node-build
MAKE_OPTS="-j 4" nodenv install 10.15.2
nodenv rehash
nodenv global 10.15.2
```

Will take 1 minute.  

## Compile and install OpenProject

Careful - the manual installation I linked to above still uses stable/9, the current release is stable/10 (as of Feb 2020). 

```
cd ~
git clone https://github.com/opf/openproject.git --branch stable/10 --depth 1
cd openproject
gem update --system 
gem install bundler
bundle install --deployment --without mysql2 sqlite development test therubyracer docker
npm install
```

**gem update --system** will take about 5 minutes.


## Access OpenProject







