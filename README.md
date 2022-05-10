# Keepalived-DigitalOcean-Ubuntu-22.04
Keepalived with DigitalOcean using Ubuntu 22.04. Based on ["How To Set Up Highly Available Web Servers with Keepalived and Floating IPs on Ubuntu 14.04"](https://www.digitalocean.com/community/tutorials/how-to-set-up-highly-available-web-servers-with-keepalived-and-floating-ips-on-ubuntu-14-04)
 By [Justin Ellingwood](https://www.digitalocean.com/community/users/jellingwood)
 
 
## Introduction
We will make a setup with two DigitalOcean Droplets and a Floating IP which enables one droplet to take over if the other one fails.

## Prerequisites

* Two droplets on DigitalOcean, located at the same datacenter. For this guide we used two droplets in Frankfurt with the following specifications:
  * $5/mo
  * $0.007/hour
  * 1GB/1 CPU
  * 25GB SSD Disk
  * 1000GB transfer
  * Ubuntu 22.04 (LTS) x64

* One Floating IP

## Install and Configure Nginx

Open the terminal for both the primary and secondary droplet. Update and install nginx.

```
sudo apt-get update
sudo apt-get install nginx
```

Now nginx should be installed, which generates a default webpage when accessing your droplet IP.
To clearly indicate if we use the primary or secondary droplet, edit the default page using
```
sudo nano /var/www/html/index.nginx-debian.html
```

On the primary server add something like `<h1>Primary</h1>`.

On the secondary server add something like `<h1>Secondary</h1>`.

## Build and Install Keepalived

```
sudo apt-get install build-essential libssl-dev
```

We install the newest keepalived version at the time of writing but you can find the latest version at [the Keepalived download page](https://www.keepalived.org/download.html#)
```
cd ~
wget http://www.keepalived.org/software/keepalived-2.2.7.tar.gz
```

```
tar xzvf keepalived*
cd keepalived-2.2.7
```
## Create a Keepalived Upstart Script

## Create the Keepalived Configuration File

## Create the Floating IP Transition Scripts

## Start Up the Keepalived Service and Test Failover
