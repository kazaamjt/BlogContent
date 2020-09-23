# What is this?

This is a blog. Of sorts.  

In this blog I aim to document how I set up my private cloud at home,
both for my own sake, so I can look up what the hell I was thinking at the time,
and maybe someone else could find something usefull in here.  

## The Goals

The primary goal really is just for me (and my roommates) to muck about ofcourse.  
I have a lot of hardware lying around catching dust.  
That being said, I do have certain projects I'd like to have running on this, and services it should provide.  
I'll also use this system as a testbed for any software I'm working on myself.  

Some of the things I'm gonna try to get up and going:

- A searchable library of all my PDF's. I have a LOT of technical books in PDF form.  
- VMs that can host GPU-bound tasks. For example, my significant other does a lot of wacky stuff with AI's
but having her laptop train models overnight is not ideal.  
- CI/CD. I could really use some self hosted CI/CD for several of my projects.  
- A GIT server?  
- Accesible from anywhere by VPN.  

## The Plan

The plan is simple, create a frankenstein cluster of machines using any hardware I could get my hands on.
Step one is providing space for all these different machine.  
Luckily my roommate had a server-rack in a garage somewhere. (Who doesn't?)  

Now all of the hardware I have, are old desktops.
So at some point, the plan is to mount the hardware from those machines on panels,
that nicely slide in to the rack.  
Worries for later.  

For now we set up the server rack in our living room:

<img src="/static/images/server_rack.jpg" alt="Nice rack">

The bottom machine is going the main one for a while.  
The 2 machines lying on their side are going to be dismantled at some point.  

At any rate, I've been going on aboout the hardware too much already...  
So the the bottom machine is going to run Hyper-V server, the sole reason for this is simple:  
I like Hyper-V.  

Now this machine has 2 onboard ethernet ports, which allows me to host a VM with a firewall to section off my closet from the rest of the house.  
Here's a rough idea of what the network will look like:

<img src="/static/images/home_net.jpg" alt="Home network">

So, on the bottom server, I'll be creating 2 virtual switches, conencted to the 2 ethernet ports.  
The first virtual switch, will be connected to the home network, and I will specifically disable access to to for the physical machine hosting it.
(Something Hyper-V virtual switches allow for)  
