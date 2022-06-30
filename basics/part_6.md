# Basics Part 6: Automating Linux deployment

In the previous chapter we set up a VHD that, when booted again will set itself up correctly as if it's a new machine.  
This will reduce the time it takes to deploy.  

When first starting with this series, I was playing around with the idea of using a netboot system
and a Debian answer file.  
This, while clean, was a slow process.  
Deploying a new VM would take several minutes, but with our premade VHD, it takes seconds.  

## A small hitch (several really)

This way of deploying has it's own issues though, and my solution might come off as a bit... hacky.  
So if you have a weak stomach, better hold on to your lunch.  

That said, let's get to it.  
First we'll set up a new VM with the module we created and remove it's VHD:  

```Powershell
New-AutoDeployVM -Name Test -VMHost Base-1 -SwitchName "Admin LAN"
Remove-VMHardDiskDrive -VMName Test -ControllerType SCSI -ControllerNumber 0 -ControllerLocation 0
Copy-Item -Path "F:\Images\Debian\DebianBase.vhdx" -Destination "PathToVM\Test.vhdx"
Add-VMHardDiskDrive -VMName Test -Path "PathToVM\Test.vhdx"
```

This will set up our VM with a copy of our DebianBase VHD.  
If you want to save on diskspace, you can use a differencing disk instead of a copy, but that's not
something I recommend.  

Next, we'd like to boot our system, but this is where our first problem arises.  
If we check our previous linux machines, you'll see that they have a boot entry called `File`.  
This is a `UEFI` boot system using it's `NVRAM`.  
This was set up during the installation.  
Something that did not happen on our fresh VM.  
So we're going to have to go in and do this manualy.  

What we're going to do is create a small `VHDx` that has a `EFI bootable shell` on it.  
With this shell we'll simply create the required bootpoint in the `NVRAM`.  

to get started, download the required program from [here](https://github.com/kazaamjt/Scripts/blob/master/bin/bootx64.efi).  
Next, create a vhdx, initialise and format it:

```Powershell

```
