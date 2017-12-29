# PCIE-Passthrough_Arch-Linux
config files and simple setup guide for pcie passthrough on arch linux



dis
https://dominicm.com/gpu-passthrough-qemu-arch-linux/

first modified /etc/default/grub for iommu support, by addig intel_iommu=on
after this grub config was remade via 
# grub-mkconfig -o /boot/grub/grub.cfg

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
