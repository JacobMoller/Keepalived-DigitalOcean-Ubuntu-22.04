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

## Install Keepalived
We now need to install Keepalived, the service that enables fallover. Install Keepalived by entering the command in both terminals:
```
sudo apt-get install keepalived
```

## Create a Keepalived Upstart Script
On both servers we need to make a Keepalive upstart script. Open the Keepalived configuration script in both terminals:
```
sudo nano /etc/init/keepalived.conf
```
Now enter the following default configuration. This script automaticly restarts if it stops (as we need it at all times check if the primary server is live).
```
description "high-availability service"

start on runlevel [2345]
stop on runlevel [!2345]

respawn

exec /usr/local/sbin/keepalived --dont-fork
```
Save and close the file.
## Create the Keepalived Configuration File
For the individual servers we need to setup a configuration file that informs them on their own IP and the other server(s) IP(s). Start by making a directory for this on both servers:
```
sudo mkdir -p /etc/keepalived
```
### Creating the Primary Server’s Configuration
Use the following commang in the Primary servers console and note the Primary servers IP. Also, go to the Secondary server and note the Secondary servers IP:
```
curl http://169.254.169.254/metadata/v1/interfaces/private/0/ipv4/address && echo
```
Make a new file for configuration on the Primary server:
```
sudo nano /etc/keepalived/keepalived.conf
```
Enter the following code that checks if `nginx` is running every 2 seconds. This file also defines that the Primary server is `Master` and has a higher priority than the other server, `200`. Insert the Primary server IP, Secondary server IP and a password that needs to be the same on all servers: 
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
Save and close the file.
### Creating the Secondary Server’s Configuration
Make a new file for configuration on the Secondary server:
```
sudo nano /etc/keepalived/keepalived.conf
```
Enter the following script for the Secondary server. Note that the state is now `BACKUP` and that the priority of this server is lower. Like before, insert the Secondary server IP, Primary server IP and a password that needs to be the same on all servers:
```
vrrp_script chk_nginx {
    script "pgrep nginx"
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
Save and close the file.
## Create the Floating IP Transition Scripts
```
cd /usr/local/bin
sudo curl -LO http://do.co/assign-ip
```

### Create a DigitalOcean API Token
To enable the change of the Floating IP we need a DigitalOcean API Token. Create this on the DigitalOcean Website on the page "API".

Copy the API token and save it in an environment variable on both servers:
```
export DO_TOKEN=[INSERT api_token HERE]
```
### (Optional) Test if the Floating IP transition works
To test that the Floating IP can be moved enter the following command in the server that you want to assign. Note that the droplet_ID can be found on the DigitalOcean website by clicking on the server. It is now a part of the url (in the format https://cloud.digitalocean.com/droplets/{droplet_id}).
```
python3 /usr/local/bin/assign-ip [INSERT floating_ip HERE] [INSERT droplet_ID HERE]
```
The Floating IP should now be assigned to the droplet you choose.

### Create the Wrapper Script
We need a wrapper script that automatically checkes if the current server has the floating IP. On both servers make a new script:
```console
sudo nano /etc/keepalived/master.sh
```
Insert the following script, but replace the DO_TOKEN and IP with your API Token and Floating IP:
```
#!/bin/sh
export DO_TOKEN='[INSERT digitalocean_api_token HERE]'
IP='[INSERT floating_ip_addr HERE]'
ID=$(curl -s http://169.254.169.254/metadata/v1/id)
HAS_FLOATING_IP=$(curl -s http://169.254.169.254/metadata/v1/floating_ip/ipv4/active)

if [ $HAS_FLOATING_IP = "false" ]; then
    n=0
    while [ $n -lt 10 ]
    do
        python3 /usr/local/bin/assign-ip $IP $ID && break
        n=$((n+1))
        sleep 3
    done
fi
```
Save and close the file on both servers.

We need to make the script executable on both servers with the following command:
```
sudo chmod +x /etc/keepalived/master.sh
```

## Start Up the Keepalived Service and Test Failover
To start the service, enter the following command in both the Primary and Secondary console:
```
sudo service keepalived start
```
The primary server should now be displayed when accessing the Floating IP.

## Testing the functionality
To test that the functionality works, try to stop nginx on the primary server:
```
sudo service nginx stop
```
The Secondary server should now be displayed at the Floating IP after a few seconds.

To start the Primary server again run this command in the Primary console:
```
sudo service nginx start
```
To simulate a server failture you can make the Primary server restart using the following command in the Primary servers console:
```
sudo reboot
```
The Secondary server should now be displayed at the Floating IP after a few seconds. When the Primary server has rebotted it will take over and display at the Floating IP.

**Congratulations, you now have a high-availability setup using Keepalived!**

## Where to go from here
Keepalived offers other functionality such as load-balancing for your web-service. Read more about this at their [Documentation Page](https://keepalived.readthedocs.io/en/latest/index.html).
