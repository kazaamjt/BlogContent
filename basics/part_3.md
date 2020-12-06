# Basics Part 3: Putting SSO to the test

Let's immediatly get down to business.  
First, we'll set up an account for our pfSense firewall, in active directory.  
This account will be used by pfsense to bind to Active Directory:  

```Powershell
$Password = Read-Host -AsSecureString

New-ADUser -AccountPassword $Password -CannotChangePassword $true -Name "pfsense" -Enabled $true
```

I did this under the `ServiceAccounts` organizational unit.  
Then I also created a `pfSense administrator` group, under `groups > global`:  

```Powershell
New-ADGroup "pfSense Administrators" -GroupCategory Security -GroupScope Global
Add-ADPrincipalGroupMembership -Identity "you" -MemberOf "pfSense Administrators"
```

## Optional: Setting up AD CS and LDAPS

This step is optional, but I do recommend it.  
LDAP by default is not encrypted, meaning that our firewall will send clear text passwords to the Domain Controller when authenticating.  
Using LDAPS, our LDAP traffic will be SSL/TLS encrypted, and thus unreadable to third parties.  

Install and cofnigure the `Active Directory Certificate Services`.  
This is used to sign the LDAPS Certificate.  

```Powershell
Install-WindowsFeature ADCS-Cert-Authority -IncludeManagementTools
Install-AdcsCertificationAuthority -CAType EnterpriseRootCA -CryptoProviderName "RSA#Microsoft Software Key Storage Provider" -KeyLength 2048 -HashAlgorithmName SHA1 -Force
```

After this is done, restart the machine.  
Once it comes back up, it'll have LDAPS cofnigured.  

Use `ldp` on a windows machine to check if LDAPS is available on port 636.  
You can also check using the shell pfsense on pfsense:

```sh
openssl s_client -showcerts -connect 172.16.1.1:636
```

You should see the LDAPS server certificate
I did not have to a firewall rule, but if you are having connect you might have to:

```Powershell
New-NetFirewallRule -DisplayName "LDAPS" -Direction Inbound -LocalPort 636 -Protocol TCP -Profile Domain -Action Allow -Enabled True
```

## Configuring the pfsense Authentication Backend

Next we'll set up authentication on pfSense.  
In the pfSense Web Interface, go to `System` > `User Manager` > `Authentication Servers` > `Add`.  
Then fill in the following (I left some things out because their defaults are fine):

```txt
Descriptive name:           Active Directory (LDAP)
Type:                       LDAP

Hostname or IP address      WinServer1.yourdomain.local
Port value                  636
Transport                   SSL - Encrypted

Search scope Level          Entire Subtree
Search Base DN              DC=yourdomain,DC=local

Authentication containers   OU=yourdomain,DC=yourdomain,DC=local

Extended query              [x] Enable extended query
Query                       memberOf=CN=pfSense administrators,OU=Global,OU=Groups,OU=yourdomain,DC=yourdomain,DC=local

Bind credentials            pfsense@servercademy.local    *****************
Initial Template            Microsoft AD
RFC 2307 Groups             [ ]  LDAP Server uses RFC 2307 style group membership
```

Save this, then check if it works, by going to `System` > `User Manager` > `Settings`,
set the `Authentication Server` to `Active Directory (LDAPS)`.  
Then, press `Save&Test`.  
If everything is correct, you should get 3 OK's and a list of OU's.  

Now go to `System` > `User Manager` > `groups` and create a new group:  

```txt
Group name                  pfSense Administrators
Scope                       Remote
```

Keep in mind that the group name is case sensitive.  

Add them to members of the admin group.  
Save and then open the newly created group, and under assigned privileges, press add.  
Then add all privileges, EXCEPT for `User - Config: Deny Config Write`.  
If everything is set up correctly, you should be able to log in to the firewall with your own account now.  

With that working, we can now set up OpenVPN using Active Directory as a backend as well.  

## Setting up OpenVPN

To start, I created a global group named `OpenVPN Users` in active Directory.  
Membership of this group will allow our users to connect to our systems using the vpn we're going to set up.  
But your requirements might be different.  
Keep in mind that the memberOf query I provide below does not query recursively, so nested groups will not work,
you'll need a more complex LDAP query to get that done.  

Next I repeat the steps above to add another authentication server in pfSense, only changing its name to `OpenVPN Users`
and changing the extended query parameter to `memberOf=CN=OpenVPN Users,OU=Global,OU=Groups,OU=yourdomain,DC=yourdomain,DC=local`.  
The reason we do this, is so that we can seperate OpenVPN users from our administrators.  

The last thing we're going to do, is set up OpenVPN so we can reach our system from the outside.  
Got to `VPN` > `OpenVPN` > `Wizard`, and follow the wizard.  
It's that easy.  
For the `Tunnel Network`, I used `172.16.10.0/28`.  
And for the `Local Network`, I used in `172.16.0.0/16`.  
As for the authentication, I set it to our newly created `OpenVPN Users`.  

Then, under `System` > `Package Manager` > `Available packages`, I installed the `openvpn-client-export` package.  
This adds a gui that allows for easy OpenVPN config exporting.  
I then exported a Windows 10 installer (With the option to block outside dns set), disconnected my desktop from the backbone network and put it back in the home network and removed the Base-1 server from the backbone network.  
Using my own user, I was able to connect to the VPN and ssh in to both machines, as well as opening the pfSense GUI. (Be careful with deploying changes to the firewall remotely)  

Furthermore, I forwarded all traffic from my Public IP to this pfSense box, making my system internet capable.  
Changing the remote address in the OpenVPN config from `192.168.0.100` to my public IP and then using my phone, allowed me to verify that this was also up and running.  

That was it for part 3.  
Part 3 was short but quite heavy and, if you're anything like me, you'll spend a bit of time debugging getting the authentication working properly.  

In part 4 we'll set up several extra Windows Server Roles that will comei n handy for automation, and we'll set up 2 linux VMs.  

[< Basics Part 2: Setting up Centralized authentication](/basics/part_2.md) | [Basics Part 4: A peek at Linux >](/basics/part_4.md)
