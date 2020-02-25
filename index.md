## [OpenProject](https://www.openproject.org/) on a [RaspberryPi](https://www.amazon.com/Raspberry-Model-2019-Quad-Bluetooth/dp/B07TC2BK1X/ref=sxin_3_ac_d_pm?ac_md=3-0-VW5kZXIgJDc1-ac_d_pm&cv_ct_cx=raspberry+pi+4+4gb&keywords=raspberry+pi+4+4gb&pd_rd_i=B07TC2BK1X&pd_rd_r=e622118f-8bbf-43e5-ba7b-2da482551e9b&pd_rd_w=30pUf&pd_rd_wg=8t2FQ&pf_rd_p=0e223c60-bcf8-4663-98f3-da892fbd4372&pf_rd_r=8PG76Q9PBRF32SW9MH8G&psc=1&qid=1582422684)-based server 


### Background


### Introduction

[OpenProject](https://www.openproject.org/) is a fully featured, open source project management toolbox.  
A free community version has been released, but some premium features are limited to the paid-for cloud & enterprise versions. 
If you want to run your own server, the project suggests a Linux-based system with 4 GB RAM, and supplies various .deb/.rpm packages, or docker images for easy installation. 

The newer Raspberry Pi 4 boards are compact and power efficient, and with 4 cores & 4 GB of RAM at least on paper look capable of running OpenProject. This would make for a very power-efficient and compact solution for (at least) small teams or local test installations.  

The problem? OpenProject officially does not support the ARM architecture, so any described installation methods based on .deb/.rpm packages or docker fail miserably. The support forum has a few threads on this dating back several years, but little to no help has been forthcoming to to make this happen. But if you found this page, your probably know all about that already. 

## Good news: OpenProject runs on ARM (unofficially, at least)!

It's possible to get OpenProject working on a Raspberry Pi. I highly recommend a RPi4 with 4 GB RAM (or a similar board). It might be possible to get by with the 2 GB version and sufficient swap space, but that would be untested as of now, and seems ill advised. During the installation and compilation process, memory usage maxes out at about 3.1 GB. 

## Status (Feb 2020)

As of now, my OpenProject Raspberry Pi server has an uptime of about 2 weeks, good responsiveness and no issues worth talking about. 

Caveats? Consider this installation protocol an 'alpha' version. It takes 4 to 6 hours to go through, and I seem to invariably run into a problem where (independently of which ruby/npm/gem versions I start out with) OpenProject does initially not compile, usually due to a SASS-related problem. After changing npm/ruby versions a couple of times it suddenly works. I am fully aware that this 'bug' makes this a rather rough guide. 

Github doesn't allow me to upload a prepared system image, but if you send me a message I can give a link to a prepared system image with Raspian/OpenProject preinstalled. Just copy that onto a SD card, change the passwords and you are good to go. 

Or help me fix the last few kinks in the protocol. Any hints & suggestions are appreciated. 

## Key issues

* bcrypt 3.1.13 does not build on the Pi, 3.1.12 is fine. See [here](https://github.com/codahale/bcrypt-ruby/issues/201).
* Apache configuration is [outdated](https://lonewolfonline.net/apache-error-invalid-command-expiresactive/).
* PostgreSQL user privileges are not set correctly when following the 'official' protocol.
* Ruby version 2.6.3 and 2.7.0 appear to work, but both require edits to the Gemfile or Gemfile.lock. 2.6.5 might be problematic. 

### How to install Openproject on Raspian

This protocol is based on the somewhat outdated manual installation protocol that can be found [here](https://docs.openproject.org/installation-and-operations/installation/manual/). By now it has a few issues, but it does spend more time explaining the various steps. 

## Setting up the Raspberry and Raspian Buster

Download a [Raspian Buster Lite](https://www.raspberrypi.org/downloads/raspbian/) system image and [flash](https://www.balena.io/etcher/) it on a microSD card (I'm using a fast 32 GB one, smaller is fine too).

Assuming that access via SSH & the local wifi network is required:

* Create an empty SSH file to enable [shell access](https://www.raspberrypi.org/documentation/remote-access/ssh/) (or hook up the Pi to a keyboard, mouse & display directly). 
* Create an [wpa_supplicant.conf](https://www.raspberrypi.org/documentation/configuration/wireless/headless.md) file with the correct credentials for the local wifi (or hook the Pi up to a LA).
* Check your router for the IP address of the Pi, and use this to log in using your favourite SSH client (Putty, etc)

Raspian standard username // password is: pi // raspberry

Update the system and install necessary system packages, PostgreSQL and the optional memcached package). Installing npm and nodejs here is probably redundant, but it looks like this works. Drop them at your own risk. 

```
sudo apt-get update -y
sudo apt-get full-upgrade -y
sudo apt-get install -y zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libgdbm-dev libncurses5-dev automake libtool bison libffi-dev git curl poppler-utils unrtf tesseract-ocr catdoc libxml2 libxml2-dev libxslt1-dev memcached postgresql postgresql-contrib libpq-dev libsass1 libsass-dev npm nodejs
```

## Expand filesystem
Expand the filesystem to take full advantage of the size of SD card chosen;
Start **raspi-config** and go to **advanced options**, **expand filesystem**.  
Quit and reboot.


## Increase swap space to 4 GB
Not actually necessary, but possibly useful for the 2 GB board version (not recommended). Make sure to reverse the setting after the installation to avoid burning out the memory card by excessive read/write cycles.

```
sudo dphys-swapfile swapoff
sudo nano /etc/dphys-swapfile 
***find the CONF_SWAPSIZE=100 line and change to 4096 or similar***
sudo dphys-swapfile swapon
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
createuser -dW openproject
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
 
 exit
```

 
 Create the database and revert to the standard 'pi' user account:
 
 ```
 createdb -O openproject openproject
 exit
 ```

## Preparing software packages

Following the manual installation suggestions, rbenv is used to install Ruby. Whenever possible, I use all four cores to speed up compiling (hence the **-j 4** flags). These tasks have to be done as the openproject user, hence the **su -- openproject** command.  

```
sudo su -- openproject -login
git clone https://github.com/sstephenson/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.profile
echo 'eval "$(rbenv init -)"' >> ~/.profile
source ~/.profile
git clone https://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build
MAKE_OPTS="-j 4" rbenv install 2.6.3
rbenv rehash
rbenv global 2.6.3

```
Will take 15 minutes. 

rbenv 2.6.5 and 2.7.0 might work as well. 2.7.0 is too new for OpenProject, unless the 2.7.0 version is forced by editing the OpenProject gem file. 2.6.3 or 2.6.5 are simpler to use, I would recommend 2.6.3 for now. 

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
MAKE_OPTS="-j 4" nodenv install 13.7.0
nodenv rehash
nodenv global 13.7.0
```

Will take 1 minute.  

## Compile and install OpenProject

Careful - the manual installation I linked to above still uses stable/9, the current release is stable/10 (as of Feb 2020). So, using release stable/10 here, and take note of the bcrypt version, 3.1.13 failes to build. Easiest fix is to edit Gemfile.lock and change the 'bcrypt' line to 3.1.12 (instead of 3.1.13 in my case). 
If using node 2.6.3, edit the Gemfile and change the ruby version from 2.6.5 to 2.6.3. 

```
cd ~
git clone https://github.com/opf/openproject.git --branch stable/10 --depth 1
cd openproject
nano Gemfile.lock
**edit the bcrypt line and change the gem version from (probably) 3.1.13 to 3.1.12**
nano Gemfile
**edit the ruby line and change the version from (probably) 2.6.5 to 2.6.3**
gem update --system 
gem install bundler
bundle install --deployment --without mysql2 sqlite development test therubyracer docker 
npm install
npm audit fix
```

**gem update --system** will take about 8 minutes.

**bundle install** will take about 5 minutes.

**npm install** will take  15 minutes.


## Prepare config files

### Database:
```
cp config/database.yml.example config/database.yml
nano config/database.yml
```
```
production:
  adapter: postgresql
  encoding: unicode
  database: openproject
  pool: 20
  username: openproject
  password: openproject
```
  
### Email & memcache:
 
Create an app password for gmail, and include it in the config file. Make sure to keep the .yml layout intact. 
 ```
   cp config/configuration.yml.example config/configuration.yml
 nano config/configuration.yml
 ```
 ```
production:                          
  smtp_address: smtp.gmail.com
  smtp_port: 587
  smtp_domain: smtp.gmail.com
  smtp_user_name: **@gmail.com**
  smtp_password: **enter gsuite app password here**
  smtp_enable_starttls_auto: true
  smtp_authentication: plain
  
** Add at the end of the file:**
 
 rails_cache_store: :memcache
 ```

## Ensure host lookup can be performed

As user pi (exit the openproject account):
```
sudo chmod o+r /etc/resolv.conf
sudo chmod o+r /etc/hosts
```

  
## Setup OpenProject
Set secret key and store in environmental variable SECRET_KEY_BASE. 

```
sudo su -- openproject -login
cd ~/openproject
echo "export SECRET_KEY_BASE=$(./bin/rake secret)" >> ~/.profile
source ~/.profile
RAILS_ENV="production" ./bin/rake db:create
RAILS_ENV="production" ./bin/rake db:migrate
RAILS_ENV="production" ./bin/rake db:seed
RAILS_ENV="production" ./bin/rake assets:precompile
exit
```
Last one is the slow one - expect to wait for 15 minutes. 


## Install Apache & Passenger

```
sudo apt-get install -y apache2 libcurl4-gnutls-dev apache2-dev libapr1-dev libaprutil1-dev
sudo chmod o+x "/home/openproject"
sudo su openproject --login
cd ~/openproject
gem install passenger
passenger-install-apache2-module
```
This will, again, take a while - expect 20 minutes. 

Once passenger is nearly done it lists the config information for Apache2. Open a second shell as root (or copy the lines into a text editor) and perform the edits as given. Use the information below as a guideline, the actual ones are likely to be different! Raspian uses a split Apache2 configuration, where information is spread out in several files (and not centralized in /etc/apache.conf)
Install passenger for 'ruby', when asked. 

In  /etc/apache2/mods-available/passenger.load:
```
LoadModule passenger_module /home/openproject/.rbenv/versions/2.6.3/lib/ruby/gems/2.6.0/gems/passenger-6.0.4/buildout/apache2/mod_passenger.so
```

In /etc/apache2/mods-available/passenger.conf:
```
<IfModule mod_passenger.c>
     PassengerRoot /home/openproject/.rbenv/versions/2.6.3/lib/ruby/gems/2.6.0/gems/passenger-6.0.4
     PassengerDefaultRuby /home/openproject/.rbenv/versions/2.6.3/bin/ruby
</IfModule>

```

Then, still as root:
```
sudo a2enmod passenger
sudo systemctl restart apache2
```

Create the /etc/apache2/sites-available/openproject.conf file:
```
SetEnv EXECJS_RUNTIME Disabled

<VirtualHost *:80>
   ServerName yourdomain.com
   # !!! Be sure to point DocumentRoot to 'public'!
   DocumentRoot /home/openproject/openproject/public
   <Directory /home/openproject/openproject/public>
      # This relaxes Apache security settings.
      AllowOverride all
      # MultiViews must be turned off.
      Options -MultiViews
      # Uncomment this if you're on Apache >= 2.4:
      Require all granted
   </Directory>

   # Request browser to cache assets
   <Location /assets/>
     ExpiresActive On ExpiresDefault "access plus 1 year"
   </Location>

</VirtualHost>
```

Then, still as root perform the fix as described [here](https://lonewolfonline.net/apache-error-invalid-command-expiresactive/)
```
ln -s /etc/apache2/mods-available/expires.load /etc/apache2/mods-enabled/
sudo a2dissite 000-default
sudo a2ensite openproject
sudo systemctl restart apache2
```

And now: OpenProject should be accessible on **raspberry_pi-IP**:80. The standard login is admin // admin. If this does not work, see **Troubleshooting** below. 

## Congratulations!


For email notification to work well, background jobs have to be enabled. This is untested. 

Switch back to the user openproject, and edit crontab:

```
sudo su --openproject -login
crontab -e **select the editor of choice if required**
```

Insert the following line at the end of the file, make sure to use the correct Ruby version:

```
*/1 * * * * cd /home/openproject/openproject; /home/openproject/.rvm/gems/ruby-2.6.3/wrappers/rake jobs:workoff
```

Save & reboot. Enjoy. 

## Final Thoughts

Given the limited experience of how well this works, and if it will survive future development and updates, think carefully about whether you want to use this installation for anything other than a hobby, home or testing environment. OpenProject might feel inspired to make ARM a supported architecture in the future (clearly, it shouldn't be too much of a problem!). 

And obviously, if you plan to make this installation available on any sort of network, change the various passwords and user accounts, and shore up security on Raspian!


## Troubleshooting

2020/02/25

### I made it through the process, the openproject login page shows up, but I cannot use the admin // admin login. 

You might have missed an error that triggered during the **RAILS_ENV="production" ./bin/rake db:seed** step. When the WorkPackages section is executed, a **getaddrinfo** error is triggered, and **rake aborted** is displayed. 
This can be easily fixed by executing the following commands as a superuser (e.g. from the pi account), to make sure that /etc/hosts and /etc/resolv.conf can be read by all users. This step is included in the instruction above as of 02/25/2020. 

```
sudo chmod o+r /etc/resolv.conf
sudo chmod o+r /etc/hosts
```





