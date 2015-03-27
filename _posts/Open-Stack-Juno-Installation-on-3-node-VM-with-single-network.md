---
layout: post
title: Open Stack Installation on a 3 node VM with single network!
---

# Open Stack Installation on a 3 node VM with single network

## Introduction
One of the most common problem faced by user while setting up a 3 node open stack installtion is unavailability of 3 different types of network as listed in the [open stack documentation](http://docs.openstack.org/juno/install-guide/install/apt/content/index.html). The document suggests to have 3 networks namely management network, instance tunnel network and external network. While setting up open stack on your laptop or desktop it is not possible to have 3 NIC cards which could provide 3 types of network as listed. In the following sections I will provide some tips to achieve the same while running in on your laptop or desktop which has got just one nic card

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
Modify /etc/network/interfaces as follows:  
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
  Modify /etc/network/interfaces as follows:  
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
  * apt-get install mariadb-server python-mysqldb
  * Choose a suitable password for the database root account
  * Edit the /etc/mysql/my.cnf file and complete the following actions:
    [mysqld]
     ...
     bind-address = 10.0.0.100
  * In the [mysqld] section, set the following keys to enable useful options and the UTF-8 character set:
    [mysqld]
     ...
     default-storage-engine = innodb
     innodb_file_per_table
     collation-server = utf8_general_ci
     init-connect = 'SET NAMES utf8'
     character-set-server = utf8
  * Restart the database service:
    service mysql restart
  * Secure the database service:
    mysql_secure_installation
* Setup Rabbitmq on controller node
  * apt-get install rabbitmq-server
  * rabbitmqctl change_password guest "specify your own password"
This completes the process of setting up yor base installation now we are already to start setting up various open stack services.

## Add the Identity service
Next step is to configure the keystone service which is the identity service for authentication & authorization. It also acts as a catalog for various services.

Follow the link [Set up Identity Service](http://docs.openstack.org/juno/install-guide/install/apt/content/ch_keystone.html) to set up openstack keystone.

## Add the Glance service
The OpenStack Image Service glance is the central repository for all the VM images. This also holds the metadata definitions for all the open stack services.

Follow the link [Set up Glance Service](http://docs.openstack.org/juno/install-guide/install/apt/content/ch_glance.html) to set up openstack glance.

## Add the Compute service
The OpenStack compute virtualizes the underneath Compute hardware(CPU and RAM) and provides those in the form of Virtual machines.

Follow the link [Set up Compute Service](http://docs.openstack.org/juno/install-guide/install/apt/content/ch_nova.html) to set up openstack compute.

## Add the Networking Service
In my opinion this is one of the most complicated part in complete open stack installation. This is probably the section which probably decides whether you will succeed in setting up open stack finally. we are going to use OpenStack Neutron for networking services here. 

Follow the link [Set up Networking Service] (http://docs.openstack.org/juno/install-guide/install/apt/content/section_neutron-networking.html) till [Install and configure compute node](http://docs.openstack.org/juno/install-guide/install/apt/content/neutron-compute-node.html). The last part Create Initial Networks will be covered in below section

**Create Initial Networks**  
There are two steps to this. One is creating external network and other is creating tenant network.  
**Creating External Network**  
Follow the steps as mentioned below
  * In controller node run the command "source admin-openrc.sh"
  * Run the command "neutron net-create ext-net --router:external True --provider:physical_network external --provider:network_type flat"
  * Creating the subnet. This is the most important part as we need to make sure we do not assign a IP range which        could overlap with any of the existing IP addresses assisgned any of the network connected devices. Run the           following command:
    neutron subnet-create ext-net --name ext-subnet --allocation-pool start=10.0.0.150,end=10.0.0.200 --disable-dhcp      --gateway 10.0.0.1 10.0.0.0/24  
**Creating Tenant Network**
  * In the controller node run the command "source demo-openrc.sh"
  * Create the tenant network using the command "neutron net-create demo-net"
  * Create a subnet for tenant network "neutron subnet-create demo-net --name demo-subnet --gateway 192.168.1.1           192.168.1.0/24. Don't worry this is a network internal to open stack and hence there is not problem if such a         gateway or network doesn't exist.
  * Create a router on the tenant network "neutron router-create demo-router"
  * Attach the router to the demo tenant subnet "neutron router-interface-add demo-router demo-subnet"
  * Attach the router to the external network by setting it as the gateway "neutron router-gateway-set demo-router        ext-net"
  * Once you are done with all this your router on the tenant network should get the lowes possible IP address which      is 10.0.0.150. You can cross check it with running the command "neutron router-list". Try pinging this router from     any of nodes. If you are unable to ping it then there is some problem with your setup

If all is well till here we are all set to launch our first VM.

##Lauch an Openstack VM Instance  
This is why all the above exercise was done. Follow the link [Launch an Openstack VM Instance ](http://docs.openstack.org/juno/install-guide/install/apt/content/launch-instance-neutron.html) to launch your first VM. Do not forget to SSH into it.

This is all I had hope this will be useful.
