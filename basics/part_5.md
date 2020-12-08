# Basics Part 5: Dipping our toes in Linux

In the previous part, we ended by creating 3 virtual machines.  
Each one will be running Debian.  

Let's run through a basic installation and configuration of a Debian machine.  
This will be the same for all 3.  

## Installation of Debian

When you start teh VMs you'll be greeted by an option menu.  
Wether you choose graphical install or the normal one makes no difference to the actual procedure.  

After choosing a language and keyboard-layout the Debian installer will try to automatically detect a network.  
If you set up the DHCP server correctly and the deploy script, the hostname as well as the domain name will be correctly filled in.  

Next is choosing a password for the root user.  
I choose something easy, as it'll be temporary.  
Once I can SSH in to the machine, I'm changing it to a default password I keep in my Keepass database.  
The same goes for our first user's password, ofcourse.  

For our first user, I choose to stick with `debian`.  
Naming the default user after the OS is a common practise in the cloud world.  

Partition, I tend to just stick to using the entire disk, with no `LVM` and all files in one partition.  
For configuring `apt` I just stick to the defaults.  

I choose to participate in the `package usage survey`.  
Next is the selection of software to install.  
None of them need a desktop or print server.  
I only select `SSH server` and the `standard system utilities`.  

That's it for the installation.  
On to the configuration.  

## Debian first time setup

You should be able to connect to your virtual machines using their DNS name now.  
If you can't connect, try using nslookup while connected to your VPN.  
If the correct server is nto repsonding, at this to your config:  

```txt
setenv opt block-outside-dns
```
