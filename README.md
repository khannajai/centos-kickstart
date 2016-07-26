# Centos6.5 Kickstart
A guide on how to perform a minimal installation of CentOS using Kickstart configuration files on a server or VM

#How to configure and perform an automated CentOS 6.5 installation using Kickstart and HTTP Server


Kickstart configuration files are quite convenient because they save us many steps while performing a new installation. A sample Kickstart configuration file called anaconda-ks.cfg is in your /root folder and is built on the configurations chosen at the time of installation of your OS.

So how do you use a Kickstart configuration file for a new OS (on a VM or any other machine)? There are many ways to access a Kickstart configuration file at the time on installation. One of the ways is by hosting the configuration file and CentOS installation image on an HTTP server on the same network and accessing the url through the boot prompt. We will use that approach. 

I will show you how to set up your host HTTP server with all the files and configurations you need, after which I'll show you how to perform the installation on your target machine. 

In this entire example, I will use the 192.168.162.207 as the IP of the host machine and 192.168.194.2 as the IP of the target machine.


## Step 1: Configure Security settings

###Disable SELinux 
Edit the file ```/etc/selinux/config``` to set ```SELINUX=disabled```

###Turn off iptables
```
/etc/init.d/iptables save
/etc/init.d/iptables stop
chkconfig -level 35 iptables off
```
This ensures that the files in your HTTP server will be accessible from another machine.

## Step 2: Create your HTTP Server

###Install HTTPD
```
yum -y install httpd
```

###Configure HTTPD Service
```
service httpd start
service httpd status
chkconfig -level 35 httpd on
```
Your files for your HTTP server should be stored in /var/www/html/

## Step 3: Setup kickstart and installation files on HTTP server

###Download CentOS into Home folder
```
cd /home
wget http://192.168.0.127/software/OS/Centos/CentOS-6.5-x86_64-minimal.iso
```

###Create CentOS installation tree in HTTP server
```
mount /home/Centos-6.5-x86_64-minimal.iso /mnt
mkdir -p /var/www/html/centos
cp -pvR /mnt/* /var/www/html/centos
```

###Create Kickstart File
```
vi /var/www/html/rhel.cfg
```
 or, if you're using a VM
```
vi /var/www/html/rhelvm.cfg
```
I have already created a minimal rhel.cfg and rhelvm.cfg file, which you can see below. You can also download them from above on my GitHub repository.
You will have to edit the kickstart file to your specifications. 
```
###########################################################################################
#rhel.cfg:
###########################################################################################
#version=DEVEL
install
cdrom

#here put the installation folder which is on your http server
url --url=http://192.168.162.207/centos/
key --skip
lang en_US.UTF-8
keyboard us

#put your md5 hashed password below
#you can create it by giving this command in terminal: openssl passwd -1 "my password"
#and then pasting the hash here
rootpw --iscryped $1$Dz5af8NF$vbKa8ABf3TE48N/e65U7u/
firewall --disabled
authconfig --enableshadow --passalgo=md5
selinux --disabled
timezone --utc Etc/UTC
bootloader --location=mbr --driveorder=sda

#these two commands automatically clear all partitions and determine the partition size for you
clearpart --all
autopart

%packages --nobase
@core
%end
```

```
#############################################################################################
#rhelvm.cfg:
#############################################################################################
#version=DEVEL
install
cdrom

#here put the installation folder which is on your http server
url --url=http://192.168.162.207/centos/
key --skip
lang en_US.UTF-8
keyboard us

#put your md5 hashed password below
#you can create it by giving this command in terminal: openssl passwd -1 "my password"
#and then pasting the hash here
rootpw --iscrypted $1$Dz5af8NF$vbKa8ABf3TE48N/e65U7u/
firewall --disabled
authconfig --enableshadow --passalgo=md5
selinux --disabled
timezone --utc Etc/UTC
bootloader --location=mbr --driveorder=sda --append="crashkernel=auto console=ttyS0,115200"
clearpart --all
autopart

%packages --nobase
@core
%end
```

## Step 4: Install CentOS6.5 yay!

###On regular server/machine:
 1. Mount CentOS6.5 minimal ISO as virtual media on management console and reboot machine.
 2. Enter boot menu and select Virtual CDROM
 3. Press ESC and soon you will see a boot prompt that looks like
```
boot:
```
 4. Enter the following command in the boot prompt and press enter
```
linux ks=http://192.168.162.207/rhel.cfg append ip=192.168.194.2 netmask=255.255.252.0 gateway=192.168.192.1 bootproto=static 
```
You will have to enter the IP address, netmask and gateway of your target machine even if it is a fresh installation.
After that you will be prompted for which network interface to connect from, so keep all these details handy.
 5. If all goes well, you will see that your installation is automated.
 6. Reboot your system.


###On virtual machine:
 1. I'm assuming you have already set up network bridges and installed all packages required for virt-install.
   To install your VM, enter the required disk, ram, cpu core, graphics and network parameters according to your requirements and run the following command. 

```
virt-install --name=VM3 --disk path=/home/vm3/vm3.img,size=190 --vcpus=3 --ram=20000 --os-type=linux --network bridge=br0 --nographics --location=http://192.168.162.207/centos -x "ks=http://192.168.162.207/rhelvm.cfg append ip=192.168.194.2 netmask=255.255.252.0 gateway=192.168.192.1 bootproto=static console=ttyS0,115200"
```
 2. If all goes well, you will see that your installation is automated.
 3. Reboot your virtual machine.

##You now have CentOS6.5 installed on your system

##Tips for troubleshooting common issues:
Check that you're able to connect to your HTTP server from other machines.
Make sure you've specified the network details at the boot prompt or the installer won't be able to download the kickstarter file.
Make sure that the permissions on your kickstart file are set to 755. Also check the SELinux context of both the installation tree folder and kickstarter file.
