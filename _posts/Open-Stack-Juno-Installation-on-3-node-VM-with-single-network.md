# Open Stack Installation on a 3 node VM with single network

## Introduction
One of the most common problem faced by user while setting up a 3 node open stack installtion is unavailability of 3 different types of network as listed in the [open stack documentation](http://docs.openstack.org/juno/install-guide/install/apt/content/index.html). The document suggests to have 3 networks namely management network, instance tunnel network and external network. While setting up open stack on your laptop or desktop it is not possible to have 3 NIC cards which could provide 3 types of network as listed. In the following sections I will provide some tips to achieve the same while running in on your laptop or desktop.

## Initial Setup
I have used Ubuntu 14.04 with 8 GB RAM for that. My physical router distributes IP with the following CIDR 10.0.0.0/24. Install Virtual Box and fork out 3 VM's with Bridge Network and promiscious mode set as "Allow All". Here is how the VM configuration would look like

**VM1**
* HostName - openstackcontroller
* RAM - 1 GB
* HDD - 8 GB
* CPU - 1 
* Network Adapter - 1 nos

**VM2**
* HostName - openstacknetwork
* RAM - 512 MB
* HDD - 8 GB
* CPU - 1 
* Network Adapter - 3 nos

**VM3**
* HostName - openstackcompute
* RAM - 4 GB
* HDD - 8 GB
* CPU - 1 
* Network Adapter - 2 nos

Once your VM is up and running setup static IP addresses for all the adapter in each of the VM except eth2 in openstacknetwork host. Moreover whereever referred eth0 is out management network, eth1 is internal tunnel network and eth2 is external network. Here is how you can go about it making the changes to your network on each of the nodes

**Openstackcontroller node**

Modify /etc/network/interfaces as follows:  
auto lo  
iface lo inet loopback  
  
auto eth0  
iface eth0 inet static  
address 10.0.0.100  
network 10.0.0.0  
netmask 255.255.255.0  
broadcast 10.0.0.255  
gateway 10.0.0.1  
  
**Openstacknetwork node**  
auto lo  
iface lo inet loopback  
  
auto eth0  
iface eth0 inet static  
address 10.0.0.103  
network 10.0.0.0  
netmask 255.255.255.0  
broadcast 10.0.0.255  
gateway 10.0.0.1  
  
auto eth1  
iface eth1 inet static  
address 10.0.0.104  
netmask 255.255.255.0  
  
auto eth2  
iface eth2 inet manual  
up ip link set dev $IFACE up  
down ip link set dev $IFACE down  

**Openstackcompute node**  
  
auto lo  
iface lo inet loopback  
auto eth0  
iface eth0 inet static  
address 10.0.0.101  
network 10.0.0.0  
broadcast 10.0.0.255  
netmask 255.255.255.0  
gateway 10.0.0.1  
  
auto eth1  
iface eth1 inet static  
address 10.0.0.102  
netmask 255.255.255.0  

Once these changes are done reboot all the nodes and check if you are able to ping each of the IP addresses as defined in the configuration.

**Few Other Configuration**
* Setup  NTP on all the nodes
* Setup UTC on all the nodes
* Setup /etc/hosts on all the nodes
* Install the Ubuntu Cloud archive keyring and repository on all the nodes:
  * apt-get install ubuntu-cloud-keyring
  * echo "deb http://ubuntu-cloud.archive.canonical.com/ubuntu"
  "trusty-updates/juno main" > /etc/apt/sources.list.d/cloudarchive-juno.list
* apt-get update && apt-get dist-upgrade on all the nodes. This will update the packages to latest in your system. If the upgrade process upgrades kernel than reboot your system
* Setup MySQL Database on controller node
  * 
