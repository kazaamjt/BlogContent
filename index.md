# Between Cloud and Cables

## What this is all about

This is a blog. Of sorts.  

In this blog I aim to document how I set up my private cloud at home,
both for my own sake, so I can look up what the hell I was thinking at the time,
and maybe someone else could find something useful in here.  

## The Goals

The primary goal really is just for me (and my roommates) to muck about of course.  
I have a lot of hardware lying around catching dust.  
That being said, I do have certain projects I'd like to have running on this, and services it should provide.  
I'll also use this system as a testbed for any software I'm working on myself.  

Some of the things I'm gonna try to get up and going:

- A searchable library of all my PDFs. I have a LOT of ebooks on various technical subjects.  
- VMs that can host GPU-bound tasks. For example, my significant other does a lot of wacky stuff with AI's
but having her laptop train models overnight is not ideal.  
- CI/CD. I could really use some self hosted CI/CD for several of my projects...  
- A GIT server?  
- VPN services, so I can access the cluster from anywhere.  

## The Plan

The plan is simple, create a frankenstein cluster of machines using any hardware I can get my hands on.  
Step one is providing space for all these different machine.  
Luckily my roommate had a server-rack in a garage somewhere. (Who fucking doesn't?)  

Now all of the hardware I have, are old desktops.  
So at some point, the plan is to mount the hardware from those machines on panels,
that nicely slide in to the rack.  
Worries for later.  

For now we set up the server rack in our living room:

![Nice rack](/images/server_rack.jpg "After about 2 hours of trying, we managed to get this rack in to our 2nd floor apartment.")

The bottom machine is going the main one for a while.  
It's a decent machine, [but it was not without its share of problems.](/misc/I219-V.md)  
The 2 machines lying on their side are going to be dismantled at some point.  

At any rate, I've been going on about the hardware too much already...  
The the bottom machine is going to run Hyper-V server, for the sole reason that I like Hyper-V.  

This machine has 2 onboard ethernet ports, which allows me to host a VM with a firewall to section off my closet from the rest of the house.  
Here's a rough idea of what the network will look like:

![Home network plan](/images/home_net.png "If it's not clear, the physical machine does not have direct access to the home network.")

## The execution

The actual work will be covered in a multi part blog.  
As a quick note, I'd like to add that there will be a bunch of passwords that need to be set and saved somewhere.  
Instead of using the same password over and over, I recommend using [Keepass](https://keepass.info/)
to store the passwords of local admins and credentials of service accounts.  

[Part 1: The first three >](/base/part_1.md)
