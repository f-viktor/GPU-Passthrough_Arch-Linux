# GPU-Passthrough on Arch Linux
Config files and down-to-earth setup guide for pcie passthrough on arch linux  

# Setup
Date: 2017  
Kernel: 4.14.idontremember  
CPU: i5-4570 (host gpu)  
GPU: GTX1060 (guest gpu)  
RAM: 8GB (dont listen to the haters who say you need at least 12)  
Guest: Windwos 8.1 (7 does not support UEFI boot which is bad for some reason so you'll have an easier time doing it with 8 and 10)  

a guide out of many, which is base this on. Not completely up to date, will throw issues  
https://dominicm.com/gpu-passthrough-qemu-arch-linux/

pretty good video, if you're the visual type:  
https://www.youtube.com/watch?v=6FI31QDtyy4  
will not work without fixing error 43, see below

pretty good guide  
https://bufferoverflow.io/gpu-passthrough/

Obligatory Arch wiki entry:  
https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF

# Checking software requirements
Just check if your kernel supports vfio-pci via
```
modinfo vfio-pci
```
if this does not throw "wtf is vfio-pci?" then you are golden

**You can skip this if you didnt get an error**  
You need some kernel after 4.1 I belive, to support vfio-pci. You can check your kernel version via  
```
uname -r
```
or  
```
uname -a
```

if you, for some reason do not have an up-to-date kernel.... why? Also there is a way to do this without vfio-pci, with pci-stub that should work on older kernels, but I have no idea how.  
just write  
```
# pacman -Syyu
```
to update your Arch and thank me later ;)

# Checking hardware requirements
Bottom line, it's easier to just try and check. You can enable IOMMU and check whether it worked.

**Blabbering about marketing buzzwords**  
So there is a bunch of links and things on the arch wiki page for what CPU/motherboard supports this,
however none of those links are complete and very clear to understand. I read somewhere that your GPU should have an UEFI rom. I'll be the first to say I have no idea what that means, and how to check this. if you have a gpu that is worth passing through it probably supports it

Do note that if you have two of the same card(e.g.: an msi GTX1060 and another msi GTX1060) and wanna pass only one, you're gonna have a hard time. Dunno if this applies to cards made by different manufacturers, run lspci -nn and check if they have different hardware IDs (gonna be in the format of [1337:ree1] )
there is some script on the arch wiki for this but I have no idea how that works and I've never tried it.

**Blabbering about marketing buzzwords 2: electric blabberoo**  
*"A rose by any other name is just as botnet"*  
IOMMU==Vti-d==Vti-x=="Virtualization technology"==(whatever amd calls this) - its all the same s#!t mane.  
Technically this is incorrect, as some of these are different technologies but if you bought a CPU and motherboard in this century, you should be OK. I'm gonna call this feature IOMMU throught this guide, and I'll also call UEFI BIOS, because I'm oldschool like that, and I dont like to learn new words. I may also write vfio as vifo because I misread it the first time I saw it and it just burned into my subconcious.


# Enabling IOMMU
**Step 1 - enabling it in BIOS**  
What you wanna do is boot into your bios settings (aka mash F12 or Del or TAB while your PC boots) ,and look for a setting called any of the following:  
 - IOMMU  
 - vti-d  
 - vti-x  
 - "Virtualization technology"  

and enable it. 

ThinkPads currently come with a bios that mark this as vti-d and vti-x and MSI motherboards come with a bios that call this "Virtualization technology" as long as you found anything relevant to this, enable it. Look in Advanced settings, cpu settings, OC settings it may be well hidden. If it aint there, you may be out of luck. If you enabled it, boot into Arch.

**Step 2 - enabling it in the bootloader**  
For the purposes of this demonstration (and also because I've never used anything else), I'll describe how to do it if you use grub as a bootloader. If you use "Systemd-boot" whatever that is, a link https://dominicm.com/gpu-passthrough-qemu-arch-linux/ that details that.

Please bear in mind, that your config file will be different to mine, however nothing else needs to be changed for this to work. 

What you wanna do is
```
# vim /etc/default/grub
```
and add `intel_iommu=on` in the place I added it in the uploaded config file. (for amd cpus this would be `amd_iommu=on`)
then remake the grub config via:  
```
# grub-mkconfig -o /boot/grub/grub.cfg
```
(this is the arch equivalent of `update-grub` from debian if you read one of those pesky debian guides)

cool, now reboot.

**Step 3 - did it work?**  
open a terminal and enter
```
dmesg | grep -e DMAR -e IOMMU
```
around the second line you should have something like `Intel-IOMMU: enabled`
you should also probably have like 10-20 other lines.
if your output doesn't look something like this
```
dmesg | grep -e DMAR -e IOMMU

[    0.000000] ACPI: DMAR 0x00000000BDCB1CB0 0000B8 (v01 INTEL  BDW      00000001 INTL 00000001)
[    0.000000] Intel-IOMMU: enabled
[    0.028879] dmar: IOMMU 0: reg_base_addr fed90000 ver 1:0 cap c0000020660462 ecap f0101a
[    0.028883] dmar: IOMMU 1: reg_base_addr fed91000 ver 1:0 cap d2008c20660462 ecap f010da
[    0.028950] IOAPIC id 8 under DRHD base  0xfed91000 IOMMU 1
[    0.536212] DMAR: No ATSR found
[    0.536229] IOMMU 0 0xfed90000: using Queued invalidation
[    0.536230] IOMMU 1 0xfed91000: using Queued invalidation
[    0.536231] IOMMU: Setting RMRR:
[    0.536241] IOMMU: Setting identity map for device 0000:00:02.0 [0xbf000000 - 0xcf1fffff]
[    0.537490] IOMMU: Setting identity map for device 0000:00:14.0 [0xbdea8000 - 0xbdeb6fff]
[    0.537512] IOMMU: Setting identity map for device 0000:00:1a.0 [0xbdea8000 - 0xbdeb6fff]
[    0.537530] IOMMU: Setting identity map for device 0000:00:1d.0 [0xbdea8000 - 0xbdeb6fff]
[    0.537543] IOMMU: Prepare 0-16MiB unity mapping for LPC
[    0.537549] IOMMU: Setting identity map for device 0000:00:1f.0 [0x0 - 0xffffff]
[    2.182790] [drm] DMAR active, disabling use of stolen memory

```
I can't help you :(

# Checking the IOMMU groups
IOMMU groups devices based on some magic I'm not smart enough to tell you about. But I dont really need to understand it to take advantage of it right?
so type:
```
lspci -nn
```
and
```
find /sys/kernel/iommu_groups/ -type l
```

In the output of `lspci -nn`, you'll have a number of pci devices with their  respective IDs like this:

find the line describing the gpu you wanna passthrough, its gonna be something like "VGA" or 3D controller and then the name of your GPU e.g.: NVIDIA Corporation GTX1060 or something like this:
```
01:00.0 3D controller [0302]: NVIDIA Corporation GF117M [GeForce 610M/710M/810M/820M / GT 620M/625M/630M/720M] [10de:1140] (rev a1)
```
note the number at the beginning of the line **01:00.0**

Note: this is not the ID of a GTX1060, this is another laptops dedicated GPU that I'm writing this from.


Now look at the output of `find /sys/kernel/iommu_groups/ -type l`. Its gonna have lines like this:
```
/sys/kernel/iommu_groups/1/devices/0000:00:02.0
```
at the end of these entries you also have a number in the format **00:02.0**

so every line in `find /sys/kernel/iommu_groups/ -type l` contains a group ID  
/sys/kernel/iommu_groups/**1**/devices/0000:00:02.0  
and a device ID  
/sys/kernel/iommu_groups/1/devices/0000:**00:02.0**  


what you wanna do is look for the line that has your GPU's device ID (identified from `lspci -nn`) and check the group ID of that line. There lie now 4 possibilities in front of us:  

**Possibility 1: there is no line containing the hardware ID of your GPU in the output of find /sys/kernel/iommu_groups/ -type l**  
in this case, I think you might have messed something up, I have no idea how you did this.

**Possibility 2: the hardware ID of your GPU is the only one in its group, there are no other lines with that group ID**  
You're golden.

**Possibility 3: the GPU built in HDMI audio device is also in that group and/or the PCI bridge is in that group**  
This should be easy to spot, the HDMI audio device is going to have the same NVIDIA tag in its name, etc, the PCI bridge is gonna be called PCI bridge. 

This is not a problem, you do not have to do anything in this case. However be sure, that when you bind the devices, and pass the devices and do anything always bind the GPU+ the HDMI audio device together, but **not** the PCI bridge. e.g. later on we'll have a step where we add device IDs to a config file. Only add the GPU and the audio device, do **not** add the PCI bridge. We'll also have a step where we add pci devices in qemu, do the same, do **not** add the pci bridge.

**Possibility 4: the GPU is grouped with all kinds of wierd things, like an audio card or some network controller or another GPU you wanna pass**  
This ain't good sonny boy, but do not worry.  
I never had to do this, I believe this is kind of a rare issue, but here is how you fix it:  

**Fix#1 - Welcome to the real world**  
Turn off your PC, rip out your GPU, put it in another PCIe slot if you have one, check again.  

**Fix#2 - Apply the ACS patch.**  
The only way I know of how to do this involves installing the patched kernel from AUR and enabling it in the bootloader, here's how you do that:  
https://dominicm.com/install-vfio-kernel-arch-linux/  
do not use packer, its a mess, just   
```
git clone https://aur.archlinux.org/linux-vfio.git   
```
go into the directory
```
cd ./linux-vfio/
```
and  
```
makepkg --skippgpcheck
```
this is gonna take like 4 hours, I'm not even joking.  
it may also throw things in the beginning like "command not found fakeroot" or something else, whatever it throws just install it with:
```
sudo pacman -S fakeroot
```
and restart it  
after its done do 
```
sudo pacman -U linux-vfio-4.9.8-1-x86_64.pkg.tar.xz
sudo pacman -U linux-vfio-headers-4.9.8-1-x86_64.pkg.tar.xz
```

now remake grub
```
# grub-mkconfig -o /boot/grub/grub.cfg
```
should automagically find the new kernels that are named something like vmlinuz-linux-vifo  
find the relevan kernel entry in `/boot/grub/grub.conf` (or wherever is your default boot directory) and add to the options after `intel_iommu=on` the following separated by a space:
```
pcie_acs_override=downstream
```
(you can also just add it in `/etc/default/grub` like we did for ˙`intel_iommu`, but that will add it for all your kernels, and it may or may not cause an issue when running other kernels. if you do this, rerun `grub-mkconfig`)
save it and reboot, then choose this kernel in the boot menu (it may be under advanced options in the grub menu)  
check if it is the vfio kernel via 
```
uname -a
```
should have vfio written in it somewhere, that means you did it.  
This should allow you to pass any device regardless of IOMMU groups, however you still wanna pass the gpu hdmi audio if you have one, as some cards just do not work without it for reasons beyond me.

# Adding vfio kernelmodules
you gotta add the vfio kernelmodules or something so the vfio drivers can hold your gpu for you when the VM is not running. You gotta:
```
# vim /etc/mkinitcpio.conf
```
There is going to be a line like `MODULES()` or `MODULES=""`  (depending on wether your kernel is in python2 or python3 eh? :D)  
regardless add the following between the "" or ():  
```
vfio vfio_iommu_type1 vfio_pci vfio_virqfd
```
if there was anything already there like i915 or nouveau, be sure to write this **before** whatever was there, e.g.:
```
MODULES(vfio vfio_iommu_type1 vfio_pci vfio_virqfd i915 nouveau)
```
save the file, then run:
```
sudo mkinitcpio -p linux
```

# Defining devices to be bound by vfio drivers
Okay cool, so now vfio drivers will be the first to choose devices
you gotta tell them what devices to choose tho.
edit
```
# vim /etc/modprobe.d/vfio.conf
```
(dont worry if the file does not exist, for me the entire directory was missing)

and write the line
```
options vfio-pci ids=10de:1c03,10de:10f1
```
where the ids= are the hardware ID of the GPU and HDMI audio device from the output of `lspci -nn`

rerun
```
sudo mkinitcpio -p linux
```
for good measure
then reboot

afte rebooting, write
```
lspci -nnk
```
and check if the devices were bound by the vfio-pci driver
it should look something like this 
```
    01:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Ellesmere [Polaris10] [1002:67df] (rev c7)
    	Subsystem: Advanced Micro Devices, Inc. [AMD/ATI] Device [1002:0b37]
    	Kernel driver in use: vfio-pci
    01:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Device [1002:aaf0]
    	Subsystem: Advanced Micro Devices, Inc. [AMD/ATI] Device [1002:aaf0]
    	Kernel driver in use: vfio-pci
    	Kernel modules: snd_hda_intel
```
note the `Kernel driver in use:` parameter. If it doesnt look like that, you messed something up, recheck your conf files, re run 
```
sudo mkinitcpio -p linux
```
re reboot

you can also check
```
sudo dmesg | grep -i vfio
```
the device ids should be in there somewhere if it's working right

# Installing necessary software for emulation
```
sudo pacman -S qemu libvirt ovmf virt-manager
```
boy, that was an easy step, really refreshing isn't it?

# Getting the latest ovmf
Now its true that we installed ovmf in the last step, but the repo wersion includes firmware files that just did not work for me. When I tried to boot the windows installer it just blanked, and threw me in the UEFI shell. Sidenote: if that happens to you, try typing exit and opening the image from the boot manager, or typing `fs0:` and manually browsing to the `/boot/efi` folder in the windows disk and typing the .efi file name, see if that works. (side sidenote, it probably wont)

So there is a great guy who does this:  
https://www.kraxel.org/repos/jenkins/edk2/  
you'll need the one that looks like: `edk2.git.ovmf-x64-<somedate>.<jibberish>.<jibberish>.noarch.rpm`  
download that one, and since it is in .rpm, install rpmextract:  
```
sudo pacman -S rpmextract
```
now do this:
```
rpmextract.sh edk2.git-ovmf-x64-0-20150916.b1214.g2f667c5.noarch.rpm 
ls edk2.git-ovmf-x64-0-20150916.b1214.g2f667c5.noarch.rpm usr 
sudo cp -R usr/share/* /usr/share/
```
Your files should be in 
```
/usr/share/edk2.git/ovmf-x64/
```
after this.

# Apllying the latest ovmf
now we gotta tell qemu to actually use what you downloaded
for this you'll need to edit this file:
```
# vim /etc/libvirt/qemu.conf
```
the whole thing should be commented out
there is going to be commented lines that look something like this(check for format clues), but feel free to just add this to the end of the file:
```
nvram = [
"/usr/share/edk2.git/ovmf-x64/OVMF-pure-efi.fd:/usr/share/edk2.git/ovmf-x64/OVMF_VARS-pure-efi.fd"
]
```
check if those two files actually exist, I may have mistyped something. If you do not even get a bootloader on your vm, you messed this step up.

# Enable libvirtd
just open a console and type
```
systemctl enable --now libvirtd
systemctl enable virtlogd.socket
```
do a reboot just for good measure, even though its probably unnecessary

# Creating the VM
kay so lets do this, LEEEEROOOOY JENKIIIHNS  
the video I mentioned earlier makes this way easier but misses some things:  
https://www.youtube.com/watch?v=6FI31QDtyy4  

**Step 1: starting virt-manager**
before starting virt-manager, you probably want to start the default network so it does not complain about it  
just do   
```
# virsh net-start default
```
this will throw an error this time, because the network does not yet exist, but you'll have to do this every time after creating your first vm if you put it in the default network, otherwise it will not start. You can also just pass the host network device to the VM, but this will not allow you to communicate from your host to the guest through the network, as they would have to use the same NIC.
and to start virt-manager
```
# virt-manager
```
you wanna run this as root. especially if you get some sort of login error from virtmanager otherwise

**Step 2: create new vm**
basic stuff, select windows iso, set resources, check customize options.

on the configure window set the following (you gotta check appy  on every screen)  
CPU:  
  copy host CPU configuration - check  
IDE Disk 1:  
  change IDE to VirtIO  
IDE CDrom:  
  change IDE to SATA and browse windows .iso  
Add Hardware -> Storage-> Cd drive,  
  change IDE to SATA and browse viirtio driver (https://fedoraproject.org/wiki/Windows_Virtio_Drivers)  
Boot Options  
  Enable boot menu - check  
  set boot order as: #1 Windows CD #2 virto driver #3 hdd  
Add Hardware-PCIe something  
  select the GPU and HDMI audio devices in PCIE  

At this point you should be able install windows in a window  
you may have to load the virtio driver when selecting the harddrive its not clear, i just installed on an IDE drive accidentally, I'll redo this.

**Step 3: fixing CPU performance**
If your CPU multicore performance benchmarks horribly with this setup, it may be because QEMU by default virtualizes cpu cores as separate sockets. The issue is that Windows can handle up to 2 separate sockets in a system, and if you pass more than that, it'll just ignore those cores/sockets. A telltale sign of this is when you press Win+Pause|Breake, you'll see it saying 2 cpus, but in device managers it will show 4 cpu devices.  To fix tihs, simpy go to the CPU menu in virt-manager and choose  
CPU:  
Manually set CPU topology  
you should see something like     
Sockets: 4  
Cores: 1   
Threads: 1  

Change this to:  
Sockets: 1  
Cores: 4  
Threads: 1  

Now Windows should see the CPU as 1 cpu with 4 cores.

# After installation
Remove the devices QXL and spice, and change the windows should now come through your GPU that you passed through.
mouse and keyboard will no longer work however,
you can plug in extra mice and keyboard, or just give it your own,
Add Hardware-> USB something
add your keyboard first, as you'll need your mouse to add your mouse. (you'll get them back when you shut down the VM)

Install the latest drivers, nvidia drivers will install but will refuse to work.
In device manager the device will be disabled due to error: 43 and your resolution winn be bound to 800x600
to fix this you'll have to find the .xml file of your VM, 
```
$ locate <nameofyourvm>
```
if you dont have locate, you can use find. It should be in `/etc/libvirt/qemu/<nameofyourvm>.xml` or something like this

you'll have to add the lines


```
<domain>
    ...
        <features>
            ...
            <kvm>
                <hidden state='on'/>
            </kvm>
            ...
            <hyperv>
                ...
                <vendor_id state='on' value='whatever'/>
            </hyperv>
            ...
        </features>
    ...
</domain>
```
(domain, features and hyperv will probably already be there, but kvm may not be there, just ensure these all exist in this structure in your xml)  
save the file, reboot a few times, etc  
the driver should now work.  
however it may not for some people  
there is a workaround by patching the driver itself on the guest windows machine, it is kind of experimental:  
https://github.com/sk1080/nvidia-kvm-patcher  

# Creating sound
You can either buy a PCIe/USB soundboard and pass it through, or you can be cheapskate like me

**Setting up audio with PulseAudio**
You can pass back the audio output of the VM to PulseAudio as a normal application.
Please note that this does not allow audio into the VM (it may be possible to do but idk how)
Just follow the arch wiki guide, it is simple and detailed:
https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Passing_VM_audio_to_host_via_PulseAudio

# Plus tips

**Config doesn't change in virt-manager**
If you changed some config in a vm xml or just some anytihng you'll have to restart a good few services before virt-manager realises this, your best bet is to just reboot.

**Moving the image**  
lets say you created the image in the wrong place or wanna lend it to your friend,  
real simple, you just gotta change the disk line in `/etc/qemu/wmaneme.xml`  
and move the file that it was pointing at  
