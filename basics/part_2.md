# Basics Part 2: Setting up Centralized authentication

At this point, I've been using quite a bit of Windows stuff and no Linux in sight.  
I promise, we'll get to cool Linux stuff soon!  
But for now, the dredge work needs to be done.  
And because I'm very familiar with Windows AD and Hyper-V that's what I'm sticking to for now.  

Anyway, I digress, setting up `Active Directory` will allow for centralized authentication and authorization.  
The goal is for people to be able to log in to *most-if-not-all* Services using a single set of crendentials.  

## Installing and configuring Active Directory

So as I mentioned, we'll be using `Active Directory` for authentication and authorization.  
I'll be referring to `Active Directory` mostly as `AD` from here on out.  
I'll be using Powershell to do this.  
As a heads-up, I'll mostly be using PowerShell throughout this blog, when configuring Windows Stuff.  

First, let's install `AD-DS - Active Directory Domain Services`:  

```PowerShell
Install-WindowsFeature AD-Domain-Services
```

This will install the `AD domain Services` and the PowerShell administration tools so we can configure AD using Powershell.  
You can use `Get-WindowsFeature *AD*` to double-check that everything is installed.  

Next, we'll set up a Forest and a Domain and install the `DNS Services`:  

```Powershell
Install-ADDSForest -DomainName "yourdomain.local" -InstallDns
```

You can find more information on this command [here](https://docs.microsoft.com/en-us/powershell/module/addsdeployment/install-addsforest?view=win10-ps).  
I recommend reading the Powershell docs for commands, they are well structured and pretty clear, in my humble opinion.  

This will throw a warning or 2 about not allowing weaker cryptography algorithms and about not being able to create a delegation for the DNS Server.  
You can safely ignore these warnings.  
Other warnings might come up and I do recommend reading those if they are not related to the aforementioned warnings.  
If you are using ssh, when it's done, it might also kick you from your ssh session, this, again, is normal.  
Give it a couple of moments to reconfigure itself before trying to ssh again.  

upon checking DNS forwarders, `8.8.8.8` has been added for me.  
You're free to forward to your preferred public DNS.  
for example, adding 8.8.8.8 to the forwarders can eb done like this:

```Powershell
Add-DnsServerForwarder -IPAddress 8.8.8.8
```

I'll also set up reverse lookup zones.  

```Powershell
Add-DnsServerPrimaryZone -DynamicUpdate Secure -ReplicationScope Domain -NetworkId 172.16.0.0/16
```

Finaly, let's add a dns record for our firewalls management address:

```Powershell
Add-DnsServerResourceRecordA -Name firewall-1 -CreatePtr -AllowUpdateAny -IPv4Address 172.16.1.254 -ZoneName ServerCademy.local
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

```txt
OU=yourdomain,DC=yourdomain,DC=local
 ├── OU=Users
 ├── OU=Groups
 │    ├── OU=Global
 │    └── OU=DomainLocal
 └── OU=ServiceAccounts
```

Using PowerShell we can create OU's using the `New-ADOrganizationalUnit` cmdlet, much like we would use `mkdir`:  

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

Next we'll set up a user account for ourselves and a `Root Admins` role/group.  
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
New-ADGroup -Name "Root Admins" -GroupCategory Security -GroupScope Global

cd ..\..\OU=Users
New-ADUser -AccountPassword $Password -DisplayName "you" -Enabled $True -Name "you" -PasswordNeverExpires $True --SamAccountName "you"

Add-ADPrincipalGroupMembership -Identity you -MemberOf "Root Admins"
Add-ADPrincipalGroupMembership -Identity "Root Admins" `
    -MemberOf Administrators, "Schema Admins", "Enterprise Admins", "Domain Admins", "Group Policy Creator Owners"
```

If you did everything correctly, we can now use our new user to log in to the Windows server.  
With the basis of centralized authentication set up, let's immediatly put it to use.  

The next part we'll set up Active Directory as a pfsense authentication backend and set up OpenVPN, so we can remotly configure the cluster.  

[< Basics Part 1: The first machine](/basics/part_1.md) | [Basics Part 3: Putting SSO to the test >](/basics/part_3.md)
