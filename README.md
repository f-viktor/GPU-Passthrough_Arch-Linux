# PCIE-Passthrough_Arch-Linux
config files and down-to-earth setup guide for pcie passthrough on arch linux
This was done (2017) on standard ARCH kernel 4.14. using a GTX1060 and an i5-4570(the integrated gpu was used for the host linux)
the guest is a windows 8.1 (7 does not support UEFI which is bad for some reason so you'll have an easier time doing it with 8 and 10)

a guide out of many, which is base this on. Not completely up to date, will throw issues
https://dominicm.com/gpu-passthrough-qemu-arch-linux/

# Checking software requirements
easiest things first, you need some kernel after 4.1 I belive to support vfio-pci, you can check your kernel version via 
uname -r 
or
uname -a

you can also just check if your kernel supports vfio-pci via
modinfo vfio-pci
if this does not throw "i dont know wtf is vfio-pci" then you are golden

# checking hardware requirements
So there is a bunch of links and things on the arch wiki page for what CPU/motherboard supports this,
however none of those links are complete and very clear to understand.

Do note that if you have two of the same card(e.g.: an msi GTX1060 and another msi GTX1060) and wanna pass only one, you're gonna have a hard time. Dunno if this applies to cards made by different manufacturers, run lspci -nn and check if they have different hardware IDs (gonna be in the format of [1337:ree1] )
there is soem script on the arch wiki for this but I have no idea how that works and I've never tried it.

**What you need to know:**
*"A rose by any other name is just as botnet"*
IOMMU=Vti-d=Vti-x="Virtualization technology"=(whatever amd calls this) - its all the same s#!t mane.
Technically this is incorrect, as some of these are different technologies but if you bought a CPU and motherboard in this century, you should be OK. I'm gonna call this feature IOMMU throught this guide, and I'll also call UEFI BIOS, because I'm oldschool like that, and I dont like to learn new words.

It's easier to just try and check. You can enable IOMMU and check whether it worked.

# Enabling IOMMU
**Step 1 - enabling it in BIOS**
What you wanna do is boot into your bios settings(aka mash F12 or Del or TAB while your PC boots) ,and look for a setting called any of the aformentioned names for this (IOMMU vti-d vti-x "Virtualization technology" something like this) and enable it. ThinkPads currently come with a bios that mark this as vti-d and vti-x and MSI motherboards come with a bios that call this "Virtualization technology" as long as you found anything relevant to this, enable it. Look in Advanced settings, cpu settings, OC settings it may be well hidden. If it aint there, you may be out of luck. If you enabled it, boot into Arch.

**Step 2 - enabling it in the bootloader**
For the purposes of this demonstration (and also because I've never used anything else), I'll describe how to do it if you use grub as a bootloader. If you use "Systemd-boot" whatever that is, a link in at the start of this details that.

Please bear in mind, that your config file will be different to mine, however nothing else needs to be changed for this to work. You'll have  a number of entries in your bootloader (these represent the entries in the grub menu when you boot), you can add this to all of them, or just to the one you usually use. Add it to one and choose that one in the grub menu.

What you wanna do is open /etc/default/grub (this may be in a different place depending on your boot folder) and add **intel_iommu=on** in the place I added it in the uploaded config file. (for amd cpus this would be amd_iommu=on)
then remake the grub config via:
**#grub-mkconfig -o /boot/grub/grub.cfg**
(this is the arch equivalent of update-grub form debian if you read one of those pesky debian guides)

cool, now reboot and choose the entry in the grub menu that you have modified.

**Step 3 - did it work?**
open a terminal and enter
dmesg | grep -e DMAR -e IOMMU
around the second line you should have something like Intel-IOMMU: enabled
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
lspci -nn
and
find /sys/kernel/iommu_groups/ -type l

In the output of lspci -nn, you'll have a number of pci devices with their  respective IDs like this:

find the line describing the gpu you wanna passthrough, its gonna be something like "VGA" or 3D controller and then the name of your GPU e.g.: NVIDIA Corporation GTX1060 or something like this:
01:00.0 3D controller [0302]: NVIDIA Corporation GF117M [GeForce 610M/710M/810M/820M / GT 620M/625M/630M/720M] [10de:1140] (rev a1)
note the number at the beginning of the line **01:00.0**

Note: this is not the ID of a GTX1060, this is another laptops dedicated GPU that I'm writing this from.


Now look at the output of find /sys/kernel/iommu_groups/ -type l. Its gonna have lines like this:
/sys/kernel/iommu_groups/1/devices/0000:00:02.0
at the end of these entries you also have a number in the format **00:02.0**

so every line in find /sys/kernel/iommu_groups/ -type l contains a group ID
/sys/kernel/iommu_groups/**1**/devices/0000:00:02.0
and a device ID
/sys/kernel/iommu_groups/1/devices/0000:**00:02.0**

what you wanna do is look for the line that has your GPU's device ID (identified from lspci -nn) and check the group ID of that line. There lie now 4 possibilities in front of us:
**Possibility 1: there is no line containing the hardware ID of your GPU in the output of find /sys/kernel/iommu_groups/ -type l**
in this case, I think you might have messed something up, I have no idea how you did this.

**Possibility 2: the hardware ID of your GPU is the only one in its group, there are no other lines with that group ID**
You're golden.

**Possibility 3: the GPU built in HDMI audio device is also in that group and/or the PCI bridge is in that group**
This should be easy to spot, the HDMI audio device is going to have the same NVIDIA tag in its name, etc, the PCI bridge is gonnea bi called PCI bridge, its pretty straightforward.
This is not a problem, you do not have to do anything in this case. However be sure, that when you bind the devices, and pass the devices and do anything always bind the GPU+ the HDMI audio device together, but **not** the PCI bridge. e.g. later on we'll have a step where we add device IDs to a config file. Only add the GPU and the audio device, do **not** add the PCI bridge. We'll also have a step where we add pci devices in qemu, do the same, do **not** add the pci bridge.

**Possibility 4: the GPU is grouped with all kinds of wierd things, like an audio card or some network controller or another GPU you wanna pass**
This ain't good sonny boy, but do not worry. 
I never had to do this, I believe this is kind of a rare issue, but here is how you fix it:
Fix#1 - Turn off your PC, rip out your GPU, put it in another PCIe slot if you have one, check again.
Fix#2 - apply an ACS patch.
The only way I know of how to do this involves installing the patched kernel from AUR and enabling it in the bootloader, here's how you do that:
https://dominicm.com/install-vfio-kernel-arch-linux/
do not use packer, its a mess, just 
git clone https://aur.archlinux.org/linux-vfio.git 
go into the directory and
makepkg --skippgpcheck
this is gonna take like 4 hours, I'm not even joking
it may also throw things in the beginning like "command not found fakeroot" or something else, whatever it throws just install it with:
sudo pacman -S fakeroot
and restart
after its done do 
sudo pacman -U linux-vfio-4.9.8-1-x86_64.pkg.tar.xz
sudo pacman -U linux-vfio-headers-4.9.8-1-x86_64.pkg.tar.xz

now remake grub
#grub-mkconfig -o /boot/grub/grub.cfg
should automagically find the new kernels that are named something like vmlinuz-linux-vifo

find that line in grub.conf (same file we added intel_iommu='on') and add to that line separated by a space:
pcie_acs_override=downstream
save it and reboot, then choose this kernel in the boot menu (it may be under advanced options in the grub menu)

check if it is the vfio kernel via 
uname -a
should have vfio written in it somewhere, that means you did it.


changed kernelmodules in /etc/mkinitcpio.conf
added to the MODULES(  ide ) list separated by spaces then

created /etc/modprobe.d/vfio.conf file and entered device ids as such:
options vfio-pci ids=1002:67df,1002:aaf0

then remade kernel or soemthing by this:
# mkinitcpio -p linux


okay so needed latest and greatest ovmf for it to boot the installer, then write exit in uefi shell and open from boot options maybe
this is probably a way better guide:
https://bufferoverflow.io/gpu-passthrough/

probably shouldve done this instead of hand modifying /boot/grub/grub.cfg:
sudo grub-mkconfig -o /boot/grub/grub.cfg

totally didnt need acs

you have to change the xml file locate vmame
its in /etc/something/wmname.xml
and add lines
<kvm>
some shit i forgot
</kvm>

and something in the hypervisor tags also
