# Basics Part 3: The Great Migration

In part 3 of this series, we're going to be migrating our Virtual Machines from our own desktop,
to the target host.  
Then we're going to add the host to the domain.  

While working on part 3 I ran in to the following problem:  
Some ISP's will use it as the default home network subnet as well, so that would potentially cause issues.  
Others will use `192.168.0.0/24`.  
Also, Some network devices like to use `192.168.1.0/24` as their default network and will set up a dhcp server in case of some routers or firewalls.  

To alleviate these issues I redrew my network diagram a little bit.  
Instead of using `192.168.0.0/16` subnets, I switched to using `172.16.0.0/16`.  
While I was at it I decided to add a bit of network segmentation.  

So, here's the new layout:  
![Version 2](/images/net_v2.png "New net layout.")

I also added a bunch of firewall rules to traffic coming from the `backbone-network`:

![New firewall Rules](/images/Firewall_rules_backbone.png "Added firewall rules.")

Or for the lazy; allow all traffic from `backbone-network` to `172.16.1.1`.  
These will allow me, or someone else, to attach their laptop to the network and try to fix things, in case the VPN breaks and allow me to add the Hyper-V server to the domain.  

Anyway, back to the task at hand; moving the virtual machines.  
To do this, I have temporarily attached my own computer to the backbone network.  

## Moving boulders

First, let's start exporting the machines:  

```Powershell
get-VM EdgeFirewall,WinServer1,RelayClient | Export-VM -Path D:\Exports
```

While this is going on, we'll go prepare the host machine.  
First, we'll set up the virtual switches.  
If we give them the same name as the virtual switches on our desktop, they'll automatically be connected.  

Setting up the switches will look something like this:  

```Powershell
New-VMSwitch -NetAdapterName "Ethernet" -Name "External" -AllowManagementOS $false -EnableIov $true
New-VMSwitch -Name "Admin LAN" -SwitchType Private | Set-VMSwitch -Notes "Subnet 172.16.1.0/24"
```

I specifically only allow the management OS access on the backbone virtual switch.  
For communication with the outside, I want it's traffic to go over the firewall.  
Now, for the heavy lifting, let's use scp to copy over the VMs:

```Powershell
scp -r D:\Exports administrator@172.16.0.1:D:\
```

Go make some coffee, this might take a while.  
Once that is done, we'll import the VMs.  
This'll look something like this:  

```Powershell
import-vm -Path 'D:\Exports\the-virtual-machine-in-question\Virtual Machines\some-uuid.vmcx' -Copy
```

Do this for all 3 VMs.  
Once this is done, we can start up the machines:

```Powershell
Get-VM | Start-VM
```

Now that the machines have been migrated, we can add the Hyper-V server.  
This is not strictly required, but can be usefull.  
You can do this using `sconfig`.  
Before restarting, set the start and stop actions of the firewall to `save` and `startifrunning`.  
pfSense can take a while to start, so instead of rebooting, we'll have Hyper-V save its state instead.  

```Powershell
Set-VM EdgeFirewall -AutomaticStopAction Save -AutomaticStartAction StartIfRunning
```

This will start up all the machines on the given machine.  
Lastly we'll set up a VPN using active directory as authentication.  

## Setting up AD authentication on pfSense

First, we'll set up an account for our pfsense firewall, in active directory:  

```Powershell
$Password = Read-Host -AsSecureString

New-ADUser -AccountPassword $Password -CannotChangePassword $true -Name "pfsense" -Enabled $true
```

I did this under the `ServiceAccounts` organizational unit.  
Then I also created a `pfSense adminsitrator` group, under groups > global:  

```Powershell
New-ADGroup "pfSense administrators" -GroupCategory Security -GroupScope Global
Add-ADPrincipalGroupMembership -Identity you -MemberOf "pfsense administrators"
```

Next we'll set up authentication on pfSense.  
In the pfsense Web Interface, go to `System` > `User Manager` > `Authentication Servers` > `Add`.  
Then fill in the following:

```txt
Descriptive name:           Active Directory (LDAP)
Type:                       LDAP

Hostname or IP address      WinServer1.yourdomain.local
Search scope Level          Entire Subtree
Search Base DN              DC=yourdomain,DC=local

Authentication containers   OU=yourdomain,DC=yourdomain,DC=local

Bind credentials            pfsense@servercademy.local    *****************
Initial Template            Microsoft AD
RFC 2307 Groups             [ ]  LDAP Server uses RFC 2307 style group membership
```

Save this, then check if it works, by going to `System` > `User Manager` > `Settings`,
set the `Authentication Server` to `Active Directory (LDAP)`.  
Then, press `Save&Test`.  
If everything is correct, you should get 3 OK's and a list of OU's.  

Now go to `System` > `User Manager` > `groups` and create a new group:  

```txt
Group name                  pfSense Administrators
Scope                       Remote
```

Add them to members of the admin group.  
Save and then open the newly created group, and under assigned priviliges, press add.  
Then add all privileges, EXCEPT for `User - Config: Deny Config Write`.  
If everything is set up correctly, you should be able to log in to the firewall with your own account now.  

## OpenVPN

The last thing we're going to do, is set up OpenVPN so we can reach our system from the outside.  
Got to `VPN` > `OpenVPN` > `Wizard`, and follow the wizard.  
It's that easy.  
For the connection subnet, I used `172.16.10.0/28`.  
And for the remote subnet, I used in `172.16.0.0/16`.  

Then, under `System` > `Package Manager` > `Available packages`, I installed the `openvpn-client-export` package.  
This adds a gui that allows for easy OpenVPN config exporting.  
I then exported a Windows 10 installer, disconnected my desktop from the backbone network and put it back in the home network.  
Using my own user, I was able to connect to the VPN and ssh in to several machines, as well as opening the pfSense GUI. (Be carefull with deploying changes to the firwall remotely)  

Furthermore, I forwarded all traffic from my Public IP to this pfSense box, making my system internet capable.  
Changing the remote address in the OpenVPN config from `192.168.0.100` to my public IP and then using my phone, allowed me to verify that this was also up and running.  

## Enter Linux

Finishing up the basics, we'll set up a linux machine.  
This machine will form the basis of our automation and orchestration system.  

Personally I will be using Debian, but you can use your prefered flavor of Linux.  
There are many good options out there.  
Keep in mind that commands and configurations might differ from OS to OS, so your milage may vary.  

I'll be using my vpn and the Windows 10 client I set up to create the virtual machine and install Debian on it.  
The installation is pretty straightforward, but you have to look out for a couple of things:  
In the VM's security settings, set the boot template to `Microsoft UEFI Certificate Authority`, otherwise the iso will not be able to boot.  
We currently don't have DHCP set up, so you'll have to manually set the IP address.  
Also, I only installed the `Standard System Utilities` and the `SSH Server`. Everything else, we don't need.  

## What's next

Now that we're done with the basic setup, it's time to move on to the real fun stuff.  
In the next chapter we'll move on to some automation.  

[< Basics Part 2: Setting up Windows Active Directory](/basics/part_2.md) | [Automation Part 1: Setting up >](/orchestration/part_1.md)
