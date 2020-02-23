## [OpenProject](https://www.openproject.org/) on a [RaspberryPi](https://www.amazon.com/Raspberry-Model-2019-Quad-Bluetooth/dp/B07TC2BK1X/ref=sxin_3_ac_d_pm?ac_md=3-0-VW5kZXIgJDc1-ac_d_pm&cv_ct_cx=raspberry+pi+4+4gb&keywords=raspberry+pi+4+4gb&pd_rd_i=B07TC2BK1X&pd_rd_r=e622118f-8bbf-43e5-ba7b-2da482551e9b&pd_rd_w=30pUf&pd_rd_wg=8t2FQ&pf_rd_p=0e223c60-bcf8-4663-98f3-da892fbd4372&pf_rd_r=8PG76Q9PBRF32SW9MH8G&psc=1&qid=1582422684)-based Server (with Arm)

### Background

[OpenProject](https://www.openproject.org/) is a fully features project management toolbox, 
open source and with a free community version, although some premium features are limited to the paid-for cloud & enterprise versions. 
If you want to run your own server, the project suggests a Linux-based server with 4 GB RAM, and supplies various .deb/.rpm packages, or docker images for easy installation.

The newer Raspberry Pi 4 boards are compact and power efficient, and with 4 cores & 4 GB of RAM at least on paper look capable of running OpenProject, making
for a power-efficient and compact solution to run a server for at least a small team. 

The downside? OpenProject does not support the ARM architecture, so any described methods based on .deb/.rpm or docker fails miserably. But if you found this page, your probably know all about that anyway. 

The good news? It's actually possible to get OpenProject working on a Raspberry Pi. I'd highly recommend a RPi4 with 4 GB RAM. It might be
possible to get by with the 2 GB version and sufficient swap space, but that's something I haven't tested yet. 

As of now, my OpenProject Raspberry Pi server has an uptime of about 2 weeks, without any obvious problems, good responsibility and no issues. 

The downside? Consider the installation protocol an 'alpha' version. I seem to invariably run into a problem where (independently of which versions I start out with) 
OpenProject doesn't compile correctly, and after changing npm/ruby versions a couple of times, it suddenly works. Which makes this obviously a pretty bad instruction set.

The quickest bypass: send me a message, and I can supply a prepared system image with Raspian/OpenProject preinstalled for now, or help me find the remaining kinks in the instruction set.

