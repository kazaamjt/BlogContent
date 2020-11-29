# Basics Part 1: The first one

The first machine, or Base-1 as I've named it, will be an important machine in our cluster.  
I'll be running Windows Server 2019 on it.  
So far this is the only piece of paid software I'm currently running.  
Other machines will simply run Hyper-V Server 2019, which is 100% free.  
The virtual machines I'll be running will mostly run Debian. Also free.  

Installing Windows Server is easy. I'm not going to go voer it in detail.  
However, if you install Windows Server, you have a choice to make: to install the GUI, or not.  

In my case I will be installing the GUI, or `desktop-experience` as the installer refers to it.  
The reason for this is, that I will be using it as a workstation of sorts as well.  

As an alternative you can forgo the GUI and instead install Windows 10 in a virtual machine on this machine.  
It can then serve the same role of workstation and this scenario is technically more secure.  
This also means you will not be able to easily interface with the Virtual Machines at first,
and so I recommend using a Windows 10 pro host to create the first VM's until your Windows 10 host has been domain joined.  

## Configuring Windows Server

Once the Windows Server is installed, we can start configuring it.  
For this we can use `sconfig`.  

- Change the Name
- Opt in/out of Windows Telemetry
- Configure the networking
- Update the server

When these are done, let's enable SSH.  
To install SSH, you'll need to have update `KB4480116` installed, otherwise the following procedure will not work.  
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

I also installed the following software:

- [Firefox (Developer Edition)](https://www.mozilla.org/en-US/firefox/developer/)
- [Git](https://git-scm.com/downloads)
- [vscode](https://code.visualstudio.com/)
- [Keepass](https://keepass.info/)

Also download a [pfsense](https://www.pfsense.org/download/) iso, it'll come in handy later.  

Finaly, we'll use Powershell to install and configure the Hyper-V role:  

```Powershell
Install-WindowsFeature -Name Hyper-V -IncludeManagementTools -Restart
```

After the restart we'll be able to start using Hyper-V.  
The first goal will be setting up the firewall.  
We'll use Hyper-V's virtual switch capabilities to put the Firewall between our base machine and the internet.  

We'll first create 2 `external` switches, one facing the internet and one facing the rest of our cloud system.  
We'll also create an `Internal` switch.  
Well put important servers that hold management functions in this network, se we can section them off and control incomming and outgoing data.  
List all your network adapters using powershell, like so:

```Powershell
PS C:\Users\Administrator> Get-NetAdapter

Name                      InterfaceDescription                    ifIndex Status       MacAddress             LinkSpeed
----                      --------------------                    ------- ------       ----------             ---------
Ethernet                  Killer E2200 Gigabit Ethernet Contro...      15 Up           11-22-AA-BB-CC-01         1 Gbps
Ethernet 2                Intel(R) Ethernet Connection (2) I219-V       5 Up           11-22-AA-BB-CC-02         1 Gbps
```

Now create the virtual switches:

```Powershell
New-VMSwitch -NetAdapterName "Ethernet" -Name "Internet Facing" -AllowManagementOS $false -EnableIov $true
New-VMSwitch -NetAdapterName "Ethernet 2" -Name "Backbone" -AllowManagementOS $false -EnableIov $true | Set-VMSwitch -Notes "Subnet 172.16.0.0/24"
New-VMSwitch -Name "Admin LAN" -SwitchType Internal | Set-VMSwitch -Notes "Subnet 172.16.1.0/24"
```

Don't just copy my commands!  
For one, your network adapters might be reversed, or have different names.  
Second, I cannot garantue that `SR-Iov` is available for both cards on your system.  
Be sure to check using `(Get-VMHost).IovSupportReasons`.  
Lastly, we blocked the management OS from using the adapters, so that we are sure this machine can only be reached through our firewall.  

Now give the server an IP on it's adapter connected to the `Admin Lan` switch.  
I went with `172.16.1.1`.  

Next, we'll configure the default storage locations for Virtual Machines and Virtual Hard Disks.  

```Powershell
New-Item -Path "D:\Hyper-V" -ItemType Directory
New-Item -Path "D:\Hyper-V\Virtual Machines" -ItemType Directory
New-Item -Path "D:\Hyper-V\Virtual Hard Disks" -ItemType Directory
Set-VMHost -VirtualMachinePath "D:\Hyper-V\Virtual Machines" -VirtualHardDiskPath "D:\Hyper-V\Virtual Hard Disks"
```

This will mostly be of importance when create virtual machines using the GUI, but it's always good to configure the defaults.  

Next we'll set up a new virtual machine, that will act as our firewall:

```Powershell
$Name = "firewall-1"
$Path = "D:\Hyper-V\Virtual Machines\$Name"

New-Item -Path $Path -ItemType Directory
New-Item -Path "$Path\VM" -ItemType Directory

New-VM -Name $Name -NoVHD -Path "$Path\VM" -MemoryStartupBytes 1024MB -Generation 2 -SwitchName "Internet Facing" -BootDevice "CD"
Set-VMFirmware -VMName $Name -EnableSecureBoot Off
Add-VMNetworkAdapter -VMName $Name -SwitchName "Admin Lan"

$VHDPath = "$Path\$Name.vhdx"
New-VHD -Path $VHDPath -Dynamic -SizeBytes 10GB
Add-VMHardDiskDrive -VMName $Name -Path $VHDPath
```

This'll set us up with a VM that has 2 network adapters and will start from a mounted iso.  
All we have to do it actually mount that iso and start the vm:

```powershell
Set-VMDvdDrive -VMName $Name -Path path/to/your/pfsense-iso
Start-VM -Name $Name
```

With the VM created, we can install and configure pfsense

## Installing and configuring the pfSense box

The installation of pfSense is pretty straight-forward these days, so nothing much to note here.  
Some Operating systems are secure boot capable on Hyper-V, but pfSense is currently not one of those.  
After the installation is done, be sure to remove the virtual cd-drive and change the boot order to boot from disk first.  

Now, the first thing I'm going to do is change the IP addresses on both network adapters.  
For now I'm not using any IPv6, but I might come back to change that later.  

Next I go through the inital set up of pdsense.  
I set the ethernet adapter connected to `Admin LAN` to `172.16.1.245`.  
After you did this, you can set this the default gateway for the server.  
Check to make sure the server could reach the Internet.  

Once this is done, you can sue firefox on the server to surf to `https://172.16.1.254`.  
The default logins is admin/pfsense.  
Log in and change this, walk through the first time setup.  

![pfSense setup](/images/pfSense-first-time.png "The pfSense first-time setup.")
