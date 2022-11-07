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

## I just want to give this a go quickly - is there a ready-made system image?

I prepared a Raspian lite-based image in Feb 2021, which should get anyone up and running in half an hour, but has some security issues, unless passwords are changes, etc.

The image file is [here](https://drive.google.com/file/d/1qBzWME8BCVja0HickLo_SOcvsUL5sUEC/view?usp=sharing), around 12gb and should fit any 16gb+ memory card. It does require a RPi4, with ideally 2 or 4gb memory. SSH is enabled (pi//raspberry), OpenProject is still on the default login (admin//admin), and you will have to enable wifi (/etc/wpa-supplicant/wpa-supplicant) or ethernet to connect to the system. Then set up email notificatiosns in OpenProject (see below). 

I'm curious to hear about any issues or success stories!

## Status (Feb 2021)

I original wrote the instructions in February 2020, for OpenProject 10 - and while they (mostly) worked, they were still somewhat buggy in an unpredictable way. I have received a surprising amount of emails asking for the system image, or various fixes, and finally revisited the protocol in 2021. OpenProject recently released version 11 - and somewhat surprisingly, in 2021 the process is a lot smoother. Starting with a Raspian Lite image on a  RPi 4, the whole process was done in a few hours, with minimal issues. The protocol below is updated to reflect the required changes. 

There is an ongoing discussion in the OpenProject forum regarding tweaks and fixes: https://community.openproject.org/topics/6873
## Key issues

* to be populated as discovered

### How to install Openproject on Raspian

This protocol is based on the outdated manual installation protocol that can be found [here](https://docs.openproject.org/installation-and-operations/installation/manual/). By now it has several issues, but it does spend more time explaining the various steps. 

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
sudo apt-get install -y zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libgdbm-dev libncurses5-dev automake libtool bison libffi-dev git curl poppler-utils unrtf tesseract-ocr catdoc libxml2 libxml2-dev libxslt1-dev memcached postgresql postgresql-contrib libpq-dev libsass1 libsass-dev sassc npm nodejs
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
createuser -drsW openproject
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
MAKE_OPTS="-j 4" rbenv install 2.7.4
rbenv rehash
rbenv global 2.7.4

```
This will take 15 minutes. 

Earlier version of the protocol required an older (2.6) version, now 2.7.4 compiles just fine. 

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
git clone https://github.com/OiNutter/node-build.git ~/.nodenv/plugins/node-build
MAKE_OPTS="-j 4" nodenv install 13.7.0
nodenv rehash
nodenv global 13.7.0
```

This will take 1 minute.  

## Compile and install OpenProject

Careful - the manual installation I linked to above still uses stable/9, the current release is stable/11 (as of Feb 2021). So, using release stable/11 here. Earlier issues with bcrypt and other gems appear to have resolved themselves. 

```
cd ~
git clone https://github.com/opf/openproject.git --branch stable/11 --depth 1
cd openproject
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


  
## Setup OpenProject
Set secret key and store in environmental variable SECRET_KEY_BASE. 

```
su -- openproject -login
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
 
Edit the /etc/apache2/apache2.conf file:

```
<Directory /home/openproject/>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
        Allow from all
</Directory>
```

This might create some security risks, but is required as long as the openproject folder lives in /home/openproject instead of the /var/www/ folders. 


Reboot or restart the apache service again: 

```
sudo systemctl restart apache2
```


And now: OpenProject should be accessible on **raspberry_pi-IP**:80. The standard login is admin // admin. If this does not work, see **Troubleshooting** below. 

## Email notifications

The template configuration above works, if an 'app specific password' is created in the corresponding google account, and the POP/IMAP option is activated for the same account. See details here: https://support.google.com/accounts/answer/185833?hl=en

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
2020/02/26
2020/05/05

### I made it through the process, the openproject login page shows up, but I cannot use the admin // admin login. 

You might have missed an error that triggered during the **RAILS_ENV="production" ./bin/rake db:seed** step. When the WorkPackages section is executed, a **getaddrinfo** error is triggered, and **rake aborted** is displayed. 
This can be easily fixed by executing the following commands as a superuser (e.g. from the pi account), to make sure that /etc/hosts and /etc/resolv.conf can be read by all users. This doesn't seem to reliably fix the problem - removed from the above instructions. 2020/02/26 

```
sudo chmod o+r /etc/resolv.conf
sudo chmod o+r /etc/hosts
```
### I still cannot login with admin // admin

You can manually change the password for the admin account by opening an IRB terminal. This is generally useful if a user forgot their password, and outlined originally [here](https://community.openproject.com/topics/6584).

As the openproject user, open an IRB shell:

```
cd ~/openproject
RAILS_ENV=production bundle exec rails c
```
Once the console is open, type out the following:
```
admin = User.find_by(login: 'admin')
admin.password = 'RaspberryPi' # Must confirm to the password rules you defined!
admin.password_confirmation = 'RaspberryPi'

admin.save! # Watch the output for errors
```
The final command should return 'true'. Use the new password to login.

### Error page when accessing the login page & error message the error log:
"Could not spawn process for application /home/openproject/openproject: A timeout occurred while starting a preloader process."

Discovered, solved & reported by Matteo Nespoli (thanks!): Add "PassengerStartTimeout 200" line to passenger.conf, as outlined here: https://community.bitnami.com/t/openproject-8-3-2-0-stack-not-working-after-fresh-install/67444/12

### Test emails work, but notifications don't arrive

Background jobs are probably not activated, follow these steps: 

https://docs.openproject.org/installation-and-operations/installation/manual/

### More questions?

Email or ask here: https://community.openproject.com/topics/6873


