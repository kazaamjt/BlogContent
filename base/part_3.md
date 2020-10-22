# Part 3: The Great Migration

In part 3 of this series, we're going to be migrating our Virtual Machines from our own desktop,
to the target host.  
Then we're going to add the host to the domain.  

While working on part 3 I ran in to the following problem:  
Some ISP's will use it as the default home network subnet as well, so that would potentially cause issues.  
Others will use `192.168.0.0/24`.  
Also, Some network devices like to use `192.168.1.0/24` as their default network and will set up a dhcp server in case of some routers or firewalls.  

To alleviate these issues I redrew my network diagram a little bit.  
Instead of using `192.168.0.0/16` subnets, I switched to using `172.16.0.0/16`
While I was at it I decided to add a bit of network segmentation.  

So, here's the new layout:  
![Version 2](/images/net_v2.png "New net layout.")

I also added a bunch of firewall rules to traffic coming from the `backbone-network`:

![New firewall Rules](/images/Firewall_rules_backbone.png "Added firewall rules.")

These will allow me, or someone else, to attach their laptop to the network and try to fix things, in case the VPN breaks.  

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

This will start up all the machines on the given machine.  
Lastly we'll set up a VPN using active directory as authentication.  

## Setting up OpenVPN
