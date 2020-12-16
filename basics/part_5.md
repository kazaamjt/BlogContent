# Basics Part 5: Dipping our toes in Linux

In the previous part, we ended by creating 3 virtual machines.  
Each one will be running Debian.  

Let's run through a basic installation and configuration of a Debian machine.  
This will be the same for all 3.  

## Installation of Debian

When you start the VMs you'll be greeted by an option menu.  
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

## Active Directory and Linux

Before we can set up `SSO`, or `Single Sign On`, we'll have to set an account used to add the linux machines to the domain and a couple of groups to denote certain rights.  
so, for the user account, under service accounts `Orginizational Unit`:  

```Powershell
$Password = Read-Host -AsSecureString

New-ADUser -AccountPassword $Password -DisplayName "Linux Domain Join" -Enabled $True -Name "LinuxJoin" -PasswordNeverExpires $True -SamAccountName "LinuxJoin" -CannotChangePassword $True -Description "Used to join Linux machines to the domain"

cd ..
cd .\OU=Groups
cd .\OU=DomainLocal

New-ADGroup "DL.Allow.Linux.Login" -GroupCategory Security -GroupScope DomainLocal
New-ADGroup "DL.Allow.Linux.Sudo" -GroupCategory Security -GroupScope DomainLocal

cd .\OU=Global
New-ADGroup "Linux Admins" -GroupCategory Security -GroupScope Global
Add-ADPrincipalGroupMembership -Identity "Linux Admins" -MemberOf DL.Allow.Linux.Login
Add-ADPrincipalGroupMembership -Identity "Linux Admins" -MemberOf DL.Allow.Linux.Sudo
Add-ADPrincipalGroupMembership -Identity "Root Admins" -MemberOf "Linux Admins"
```

Next, let's create an `Orginizational Unit` in which we'll put the accoutns created when joining the domain.  
You can really choose whatever path you like, but I'm going with a new `OU` under the domain root in Active Directory:  

```Powershell
New-ADOrganizationalUnit -Name "Linux"
```

At this point, I do hope you have access to a GUI, because using powershell to give the account the right to domain join on a specific `OU` is an absolute PAIN.  
So I'll be showing it with a GUI, because it's easier to follow and less error-prone.  

![Delegate 1](/images/part_5/delegate_1.png)
![Delegate 2](/images/part_5/delegate_2.png)
![Delegate 3](/images/part_5/delegate_3.png)
![Delegate 4](/images/part_5/delegate_4.png)
![Delegate 5](/images/part_5/delegate_5.png)

Having done this will allow us to use this account to domain join our Linux machines in the next part.  

## Debian first time setup

Now, while writing this, I noticed that upon having installed the OS, the dhcp client-id changed.  
During the install it uses the `MAC Address` as it's id, but once it boots into the installed version, it used a randomly generated `DUID-LLT`.

We can change this behavior by logging on to the machine and doing the following:  

```Bash
su -
echo "send dhcp-client-identifier = hardware;" >> /etc/dhcp/dhclient.conf
rm /var/lib/dhcp/*
reboot
```

Now the machine should be back to using it's `MAC Address` as its dhcp client-id and will have the correctly assigned IP.  
Next, let's set up some basics on our machines.  
As an aside, I'm not doing this on the DebianBase machine, we'll come back to that machine later.  

So, let's install some stuff and then join the machine to our `AD Domain`:  

```Powershell
apt-get update
apt-get dist-upgrade --assume-yes

apt-get install --assume-yes hyperv-daemons
apt-get install --assume-yes packagekit policykit-1 realmd ntp adcli sssd samba-common-bin sssd-tools sudo
```

You'll get a prompt for the last `apt-get` command about `WINS`. You can safely answer no.  

Now, let's add this machine to the domain, set the login and sudo groups and finaly reboot for good measure:  

```bash
systemctl enable sssd

realm join yourdomain.local --user=LinuxJoin --computer-ou='OU=Linux,DC=yourdomain,DC=local'

realm permit -g DL.Allow.Linux.Login@yourdomain.local

systemctl start sssd
echo "session required pam_mkhomedir.so skel=/etc/skel/ umask=0022" | tee -a /etc/pam.d/common-session
echo "%DL.Allow.Linux.Sudo@ServerCademy.local ALL=(ALL) ALL" | tee -a /etc/sudoers.d/ad_admins

update-initramfs -u

systemctl daemon-reload
reboot
```

When all this is said and done, you should be able to log in to both Debian machines as following:

```bash
ssh you@yourdomain@AutomationStation.yourdomain.local
sudo apt-get update
```

In my case, it was a great success!  
