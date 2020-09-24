# Part 1: the first three

So, on the bottom server, I'll be creating 2 virtual switches, conencted to the 2 ethernet ports.  
The first virtual switch, will be connected to the home network, and I will specifically disable access to to for the physical machine hosting it.  
(Something Hyper-V virtual switches allow for.)  

Installing pfSense is quite simple, but there is another snag.  
Hyper-V server does not come with a GUI, and thus is ill suited to setting up VMs on directly.  
Instead I'm going to create a pfSense VM on my own PC and, using Hyper-V export, move it's intended host.  
