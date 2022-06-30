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
The same goes for our first user's password, of course.  

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

Note how I used the `su -` command.  
This will change our current user to the `root` user.  
All the commands in this chapter will be executed as `root`.  
Keep this in mind if something doesn't work.  

Now the machine should be back to using it's `MAC Address` as its dhcp client-id and will have the correctly assigned IP.  
Next, let's set up some basics on our machines.  
As an aside, I'm not doing this on the DebianBase machine, we'll come back to that machine later.  

So, let's install some stuff and then join the machine to our `AD Domain`:  

```bash
apt-get update
apt-get dist-upgrade --assume-yes

apt-get install --assume-yes hyperv-daemons
apt-get install --assume-yes packagekit policykit-1 realmd ntp adcli sssd samba-common-bin sssd-tools sudo
```

You might get a prompt for the last `apt-get` command about `WINS`. You can safely answer no.  

Now, let's add this machine to the domain, set the login and sudo groups and finaly reboot for good measure:  

```bash
systemctl enable sssd

realm join yourdomain.local --user=LinuxJoin --computer-ou='OU=Linux,DC=yourdomain,DC=local'

realm permit -g DL.Allow.Linux.Login@yourdomain.local

systemctl start sssd
echo "session required pam_mkhomedir.so skel=/etc/skel/ umask=0022" | tee -a /etc/pam.d/common-session
echo "debian ALL=(ALL) NOPASSWD: ALL" | tee -a /etc/sudoers.d/ad_admins
echo "%DL.Allow.Linux.Sudo@yourdomain.local ALL=(ALL) ALL" | tee -a /etc/sudoers.d/ad_admins

systemctl daemon-reload
reboot
```

When all this is said and done, you should be able to log in to both Debian machines as following:

```bash
ssh you@yourdomain@AutomationStation.yourdomain.local
sudo apt-get update
```

In my case, it was great success!  
Finaly, we'll configure the DebianBase VM and prep it for use.  

## A Debian Base System VHD

So what we're going to do, is set up a base VHD, so when we create new VMs we just copy the VHD and the system is easily and quickly installed.  
First, let's do our installations like last time:  

```bash
apt-get update
apt-get upgrade --assume-yes
apt-get dist-upgrade --assume-yes

apt-get install --assume-yes hyperv-daemons
apt-get install --assume-yes packagekit policykit-1 realmd ntp adcli sssd samba-common-bin sssd-tools sudo
```

Next, let's configure some small things:

```bash
echo "session required pam_mkhomedir.so skel=/etc/skel/ umask=0022" | tee -a /etc/pam.d/common-session
echo "debian ALL=(ALL) NOPASSWD: ALL" | tee -a /etc/sudoers.d/ad_admins
echo "%DL.Allow.Linux.Sudo@yourdomain.local ALL=(ALL) ALL" | tee -a /etc/sudoers.d/ad_admins
mkdir /opt/boot
```

After this the machine has the needed basic configuration to be cloned.  
But of course we want to also handle the SSO.  
Due to each machine having to register seperately, we'll handle this on our first boot.  

I'll be putting this script under `/opt/boot/first_boot.sh`.  
So let's have at it:  

```bash
#!/usr/bin/env bash
sleep 10
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

apt-get update
apt-get upgrade --assume-yes
apt-get dist-upgrade --assume-yes

hostnamectl set-hostname $(host $(ip route get 8.8.8.8 | awk -F"src " 'NR==1{split($2,a," ");print a[1]}') | awk '{print substr($5, 1, length($5)-1)}')
echo -e "YourSuperSecretPassword" | realm join --verbose yourdomain.local --user=LinuxJoin --computer-ou='OU=Linux,DC=yourdomain,DC=local'
realm permit --verbose -g DL.Allow.Linux.Login@yourdomain.local

systemctl start sssd
systemctl daemon-reload

crontab -r
```

You can create script using an editor like `nano` or `vim`.  

Be sure not to skip those env variables as they are important.  
`crontab` does not pass them by fefault, it passes a limited set instead.  
If they are not in your script, the `realm` command will not work.  

The `hostnamectl set-hostname` command sets the hostname.  
In thise case I'm using IP route to get our own IP, then use the IP in reverse dns lookup to get the name of this machine.  

We use echo to pass the `install` users it's password to register the machine.  
Then we add the linux login group to our login system.  
Finaly, we remove the `crontab`.  
If you want, you can add `rm -f /opt/boot/first_boot.sh` at the end to make sure the script get deleted.  

Speaking of `crontab`, `crontab` will make sure our script runs at startup.  
Invoke `crontab -e` and add `@reboot /opt/boot/first_boot.sh > /opt/boot/first_boot.log 2>&1` at the end.  
The `> /opt/boot/first_boot.log 2>&1` part of the command will redirect any output to the file specified.  
This way we can try to diagnose if any problems occur.  

Finaly, we'll change the permissions on the script and delete our dhcp info before we shut this machine down:  

```bash
chmod 755 /opt/boot/first_boot.sh
echo "send dhcp-client-identifier = hardware;" >> /etc/dhcp/dhclient.conf
rm /var/lib/dhcp/*
shutdown -h now
```

Before moving on, copy the VHD.  
I stored it under `F:\images\debian\DebianBase.vhdx`.  
You can start the machine again after copying the VHD to make sure the script executed correctly.  

[< Basics Part 4: A peek at Linux](/basics/part_4.md)[Basics Part 6: Automating Linux deployment >](/basics/part_6.md)
