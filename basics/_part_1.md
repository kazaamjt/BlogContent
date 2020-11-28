# Basics Part 1: The first one

So, on the bottom server, I'll be creating 2 virtual switches, connected to the 2 ethernet ports.  
The first virtual switch, will be connected to the home network, and I will specifically disable access to to for the physical machine hosting it.  
(Something Hyper-V virtual switches allows for.)  

We're going to set up 3 VMs in total as a base for our private cloud.  
The first one will be running Windows Server 2019, for centralized authentication and authorization. And maybe more stuff later on.  
The second machine will be a PFSense box to do routing and NAT for our system, as well as VPN.  
Lastly, a Windows 10 machine. While this one isn't strictly required, it'll allow for easier management of the cluster as a whole.  

If you don't have legal keys, the evaluation version of both Windows Server 2019 and Windows 10 Enterprise are a valid alternatives.  
Do note that I did not find a simple *painless* way to convert Windows 10 EnterpriseEval to an activated Windows 10.  
Activating Windows Server 2019 was pretty easy on the other hand.  
For the Windows 10 machine, you will either an Enterprise, Pro, or Education version.  
Other version, such as Windows 10 Home will not suffice.  

Now, since Hyper-V Server 2019 does not come with a GUI, complicating the installation and configuration of the machines,  
I'll be building the machines on my own desktop instead and use Hyper-V export to move them to the server.  
Before created any machines, I created 2 virtual switches, 1 external (called external) and 1 internal (called backbone).  
These names will come in to play later.  

## Installing and configuring the Windows Server VM

I'm not going in to detail on how to create a VM or how to install Windows.  
These are trivial tasks outside of the scope of this blog.  

I will provide you with the specs of the machine:  

CPU: 1 core  
Memory: 1024 static  
HDD: 40GB  
Network: 1 connector, external  

This might seem like it's not a lot, but this should be plenty to get us going.  
While this machine is installing, I'll also create a pfSense VM.  
Again no detailed walk-through of the installation, but here are the specs:  

CPU: 1 core  
Memory: 1024 static  
HDD: 10GB  
Network: 2 connector, 1 external and 1 internal  

Once the Windows Server is installed, we can start configuring it.  
For this we can use `sconfig`.  

A quick checklist of things to do:

- Change the Name
- Opt in/out of Windows Telemetry
- Change the network settings
- Update the server

When these are done, let's enable SSH.  
Open Powershell and check what the server is called:

```Powershell
Get-WindowsCapability -Online | ? Name -like 'OpenSSH*'
```

For me this returned `OpenSSH.Server~~~~0.0.1.0`, so that's what I'll install:  

```Powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType 'Automatic'
```

There should be a firewall rule named "OpenSSH-Server-In-TCP", and it should be enabled  
If the firewall rule does not exist, create it:  

```Powershell
Get-NetFirewallRule -Name *ssh*
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server (sshd)' -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
```

Finally, I created a new Virtual Switch (type = Internal, name = BackBone).  
I switched the WinServer network adapter to this switch at this point.  

## Installing and configuring the pfSense box

The installation of pfSense is pretty straight-forward these days, so nothing much to note here.  
Some Operating systems can are secure boot capable, but pfSense is currently not one of those.  
After the installation is done, be sure to remove the virtual cd-drive and change the boot order to boot from disk first.  

Now, the first thing I'm going to do is change the IP addresses on both network adapters.  
For now I'm not using any IPv6, but I will come back to that later.  
This is going to be a modern cloud system damnit!  

After changing some IP information I checked to make sure my server could reach the Internet.  
Then I set a static IP on the virtual adapter (BackBone) on my host computer so I could reach the internal address of the pfSense box.  
(pfSense is internal 192.168.1.254 in my case)
I also double-checked if the ssh on my windows Server was working. It was, and all was right with the world.  

The default logins is admin/pfsense.  
Log in and change this, walk through the first time setup.  

![pfSense setup](/images/pfSense-first-time.png "The pfSense first-time setup.")

NOTES:

Be sure to set the primary DNS to your Windows Servers' address.  
If your pfSense, like mine, is behind another Router/NAT, be sure to turn off `Block RFC1918 Private Networks`.  

Finish up, by reloading and checking for updates.  
Then lock the console and, optionally, enable ssh access, because currently anyone with access to the VM Connect could reconfigure it.
(System > Advanced > Admin Access > Console Options)  
Optionally, change the pfSense theme to dark mode. (system > General Setup > webConfigurator)  

And that's it for now!  

## The Windows 10 VM

This is going to be the heaviest VM by far:  

CPU: 2 cores  
Memory: min 2048MB, max 4096MB, startup 2048MB  
HDD: 60GB  
Network: 1 connector, external  

I might beef up those specs too if need be.  
If you want the install to hurry up, you might want to beef it up.  
This install takes a while.  

After installing, I set the IP address, enabled remote desktop, changed the computers name, and started updating.  
Once you enable remote desktop, VM Connect should be able to use "enhanced session mode", which allows for copy pasting of files and text.  

Updating took another long while, so you might want to find something to do, to kill some time.  
Maybe set a cup of coffee. Or grow a beard. That sorta thing.  
After that I also installed the following:

- [Firefox (Developer Edition)](https://www.mozilla.org/en-US/firefox/developer/)
- [Windows RSAT Tools](https://docs.microsoft.com/en-us/windows-server/remote/remote-server-administration-tools)
- [vscode](https://code.visualstudio.com/)
- [Keepass](https://keepass.info/)

I'll go in to detail on which RSAT tools as they come up.  

As this part took the better part of an evening, the rest will be for part 2.  
In part 2 we'll be setting up a Windows Authentication domain, aka configuring `Active Directory`.  

[< Index: What this is all about](/index.md) | [Basics Part 2: Setting up Windows Active Directory >](/basics/part_2.md)
