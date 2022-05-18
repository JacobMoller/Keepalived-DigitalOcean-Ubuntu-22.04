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

```
./configure
make
sudo make install
```
The deamon is now installed
## Create a Keepalived Upstart Script
```
sudo nano /etc/init/keepalived.conf
```

```
description "load-balancing and high-availability service"

start on runlevel [2345]
stop on runlevel [!2345]

respawn

exec /usr/local/sbin/keepalived --dont-fork
```
## Create the Keepalived Configuration File
```
sudo mkdir -p /etc/keepalived
```
### Creating the Primary Server’s Configuration
TODO: Here we use pgrep
```
vrrp_script chk_nginx {
    script "pgrep nginx"
    interval 2
}

vrrp_instance VI_1 {
    interface eth1
    state MASTER
    priority 200

    virtual_router_id 33
    unicast_src_ip [INSERT primary_private_IP HERE]
    unicast_peer {
        [INSERT secondary_private_IP HERE]
    }

    authentication {
        auth_type PASS
        auth_pass [INSERT password HERE]
    }

    track_script {
        chk_nginx
    }

    notify_master /etc/keepalived/master.sh
}
```

### Creating the Secondary Server’s Configuration
```
vrrp_script chk_nginx {
    script "pidof nginx"
    interval 2
}

vrrp_instance VI_1 {
    interface eth1
    state BACKUP
    priority 100

    virtual_router_id 33
    unicast_src_ip [INSERT secondary_private_IP HERE]
    unicast_peer {
        [INSERT primary_private_IP HERE]
    }

    authentication {
        auth_type PASS
        auth_pass [INSERT password HERE]
    }

    track_script {
        chk_nginx
    }

    notify_master /etc/keepalived/master.sh
}
```

## Create the Floating IP Transition Scripts
```
cd /usr/local/bin
sudo curl -LO http://do.co/assign-ip
```

### Create a DigitalOcean API Token
TODO: DESCRIBE WHY WE HAVE TO MAKE DO_TOKEN
```
export DO_TOKEN=[INSERT api_token HERE]
```


```
python3 /usr/local/bin/assign-ip [floating_ip] [droplet_ID]
```

### Create the Wrapper Script
```console
sudo nano /etc/keepalived/master.sh
```

```
#!/bin/sh
export DO_TOKEN='digitalocean_api_token'
IP='floating_ip_addr'
ID=$(curl -s http://169.254.169.254/metadata/v1/id)
HAS_FLOATING_IP=$(curl -s http://169.254.169.254/metadata/v1/floating_ip/ipv4/active)

if [ $HAS_FLOATING_IP = "false" ]; then
    n=0
    while [ $n -lt 10 ]
    do
        python /usr/local/bin/assign-ip $IP $ID && break
        n=$((n+1))
        sleep 3
    done
fi
```

```
sudo chmod +x /etc/keepalived/master.sh
```

## Start Up the Keepalived Service and Test Failover
```

```
