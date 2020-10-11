# Part 2: Setting up Windows Active Directory

At this point, I've been using quite a bit of Windows stuff and no Linux in sight.  
I promise, we'll get to cool Linux stuff soon!  
But for now, the dredge work needs to be done.  
And because I'm very familiar with Windows AD and Hyper-V that's what I'm sticking to for now.  

Anyway, I degress, setting up `Active Directory` will allow for centralized authentication and authorization.  
The goal is for people to be able to log in to *most-if-not-all* Services using a single account.  

## Installing and configuring Active Directory

So as I mentioned, we'll be using `Active Directory` for authentication and authorization.  
I'll be refering to `Active Directory` mostly as `AD` from here on out.  
Since our server is "headless", we'll be using Powershell to do so.  
As a headsup, I'll mostly be using PowerShell throughout this blog, when configuring Windows Stuff.  

First, let's install `AD-DS - Active Directory Domain Services`:  

```PowerShell
Install-WindowsFeature AD-Domain-Services
```

This will install the `AD domain Services` and the PowerShell administration tools so we can configure AD using Powershell.  
You can use `Get-WindowsFeature AD*,RSAT-AD*` to double-check that everything is installed.  

Since our `Domain Controller` or `DC` for short, will also function as our DNS server,
I'm setting it's address to `127.0.0.1`. This is considered good practice.  

Next, we'll set up a Forest and a Domain and isntall the `DNS Services`:  

```Powershell
Install-ADDSForest -DomainName "yourdomain.local" -InstallDns
```

You can find more information on this command [here](https://docs.microsoft.com/en-us/powershell/module/addsdeployment/install-addsforest?view=win10-ps).  
I recommend reading the Powershell docs for commands, they are well structured and pretty clear, in my humble opinion.  

This will throw a warning or 2 about not allowing weaker cryptography algorithms and about not being able to create a delegation for the DNS Server.  
You can safely ignore these warnings.  
Other warnings might come up and I do recommend reading those if they are not related to the aforementioned warnings.  
When it's done, it might also kick you from your ssh session, this, again, is normal.  
Give it a couple of moments to reconfigure itself before trying to ssh again.  

Now that we have a domain, let's add a DNS forwarder.  
I'm going to forward DNS requests to `8.8.8.8`, but you're free to forward to your prefered public DNS.  

```Powershell
Add-DnsServerForwarder -IPAddress 8.8.8.8
```

That's it for our base configuration.  
Next we'll set up some groups and users.  

## Setting up groups and users

Again we're going to be setting up our stuff using PowerShell.  
Using PowerShell we can explore AD as if it's a file structure:  

```Powershell
Import-Module -Name ActiveDirectory
cd AD:\DC=yourdomain,DC=local
ls
```

This will change your current directory to the `AD:` drive and cd in to the `DC=yourdomain,DC=local` directory.
Finally, ls will show you the objects in the current directory.  
You can navigate the `AD:` drive this way, much like you would the `C:` or `D:` drives using the objects distinguished names.  
For example we can look at the pre-made groups and users in the `Users` directory by simply typing `ls .\CN=Users`.  

NOTE: PowerShell autocomplete for paths works in this drive, but it will append a trailing `\` which you're going to have to remove.  

Now that we got this little bit of PowerShell navigation out of the way, let's start setting up our directories.  
Active directory uses *directory-like* objects called Organizational Units, or OU's for short to organize various things.  
So we're going to use these OU's to structure our AD Objects in similar way to how we might use directories to organize files.  
I'm going to use the `AGDLP` philosophy as the basis for structuring my Active Directory. ([More on AGDLP here](https://en.wikipedia.org/wiki/AGDLP))  

This will give us a directory structure, that looks something like this:  

OU=yourdomain,DC=yourdomain,DC=local  
  ├─OU=Users  
  ├─OU=Groups  
  │  ├─OU=Global  
  │  └─OU=DomainLocal  
  └─OU=ServiceAccounts  

Using PowerShell we can create OU's using the `New-ADOrganizationalUnit` cmdlet, much like we would use mkdir:  

```Powershell
New-ADOrganizationalUnit -Name "yourdomain"

cd .\OU=yourdomain
New-ADOrganizationalUnit -Name "Users"
New-ADOrganizationalUnit -Name "Groups"
New-ADOrganizationalUnit -Name "ServiceAccounts"

cd .\OU=Groups
New-ADOrganizationalUnit -Name "Global"
New-ADOrganizationalUnit -Name "DomainLocal"
```

NOTE: Make sure you perform these under the `OU=yourdomain,DC=yourdomain,DC=local` path.  

Next we'll set up a user account for ourselves and an `Root Admins` role/group.  
The `Root Admins` group will basically have root access to everything.  
As we go along we'll be setting up roles and permissions as they come up.  

First, we'll read a password from stdin and store it as a `SecureString`.  
Be sure to execute this line on it's own, to prevent pasting the rest of the command in to the password variable!  
Then we'll cd in to the newly create Users OU and create our user.  
Next we'll cd in to `Groups/Global` and create the `Root Admins` Group.  
Finally, we'll add our new user to the `Root Admins` group and make `Root Admins` a member of multiple Active Directory Admin groups.  

```Powershell
$Password = Read-Host -AsSecureString

cd .\OU=Global
New-ADGroup -Name "Root Admins" -SamAccountName RootAdmins -GroupCategory Security -GroupScope Global

cd ..\..\OU=Users
New-ADUser -AccountPassword $Password -DisplayName "you" -Enabled $True -Name "you" -PasswordNeverExpires $True

Add-ADPrincipalGroupMembership -Identity you -MemberOf RootAdmins
Add-ADPrincipalGroupMembership `
    -Identity RootAdmins `
    -MemberOf Administrators, "Schema Admins", "Enterprise Admins", "Domain Admins", "Group Policy Creator Owners"
```

If you did everything correctly, we can now use our new user to ssh to the Windows server.  

## Domain joining a windows machine

As a final step for part 2, we're going to domain join our Windows 10 machine
and then log in to it with our new user and configure it to our liking.  
First, change the DNS server to 192.168.1.1, then change the domain to `yourdomain.local`.  
To change the domain go to `Control Panel > System and Security > System` and then select `Change settings > Change... > Domain`

This will prompt you for credentials.  
Use your new credentials, `DomainName\UserName`.  
Then, once you close those dialogs, you will be prompted to restart the machine.  
Go ahead and do so.  

The final part is setting up the machine to your preference and doing some cleanup.  
For example I prefer using the full screen startup. (I know, I'm weird. One of the weirdos that actually liked Windows 8.1)  
Lastly I used `disk cleanup` to clean up a bunch of things.  
On my Win10 VM this freed ~24Gb of space... Yikes!  

And that's it for part 2!  
In the next part we'll be moving the VMs to their new home and expanding our centralized authentication/authorization.  

[< Part 1: The first three](/base/part_1.md) | [Part 3: The Great Migration >](/base/part_3.md)
