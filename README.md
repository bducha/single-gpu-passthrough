# Single GPU passthrough

This is intended to be notes for me on how I achieved single GPU passthrough to a Windows 10 VM, but maybe you will find some useful tips if you want to to the same.


## Host configuration

Motherboard : Gigabyte B450m DS3H

CPU : Ryzen 3600

GPU : Nvidia GTX 1070

OS : Ubuntu 20.04 (5.4.0 kernel)

## Procedure

### First steps

First make sure that IOMMU and virtualization is enabled in the bios of the host.

Then in the `GRUB_CMD_LINE_LINUX_DEFAULT` variable in the `/etc/default/grub` file, add the following arguments :
```iommu=1 amd_iommu=on rd.driver.pre=vfio-pc```

Start virt-manager and create a new VM. Don't forget at the end to check the box for additional configuration before installing it, we need to modify some things.

#### Note : 
virt-manager creates the `default` pool when it is launched, even if it exists. If it exists, libvirt will crash with no way to restart the service so you need to reboot the computer.
A workaround is to launch `sudo virsh pool-destroy default && sudo pool-undefine default` before launching virt-manager so that the pool will be created by virt-manager, and there will be no problem.

When you're at the additional configuration step, in the overview tab, don't forget to set the firmware to `UEFI x86_64: /usr/share/OVMF/OVMF_CODE.fd`, otherwise the GPU won't work in the VM. Also keep the chipset to `Q35`.

If you already have a disk with a UEFI install of Windows, you can add it, and set the boot order accordingly :

```
<disk type="block" device="disk">
  <driver name="qemu" type="raw" cache="none" io="native"/>
  <source dev="/dev/sdb"/>
  <target dev="sdb" bus="sata"/>
  <boot order="1"/>
  <address type="drive" controller="0" bus="0" target="0" unit="1"/>
</disk>

```

For the CPU part, I just unchecked the "Copy host CPU configuration" box, we'll see later for optimization.

For the network, I used directly the host device, because the virtual network wasn't working in Windows for me.

![](https://i.imgur.com/0H24sJS.png)

It's working just fine for me, so for now I will not go further for the network

You can also passthrough mouse and keyboard if needed for the Windows install

We will not pass through the GPU for now, otherwise virt-manager will try to install the VM infinitely.

You can now begin the installation

### GPU rom dump and patch

Dumping the rom (and set the right pci slot) :

```
echo 1 > /sys/devices/pci0000:00/0000:00:07.0/rom
cat /sys/devices/pci0000:00/0000:00:07.0/rom > original.rom
echo 0 > /sys/devices/pci0000:00/0000:00:07.0/rom
```

Then patch it with https://github.com/Matoking/NVIDIA-vBIOS-VFIO-Patcher

and put it in `/usr/share/vgabios`

### Passing the GPU

You can now pass the GPU (and the devices that are in the same group), and add the following line with the patched rom to the GPU:

`<rom bar="on" file="/usr/share/vgabios/patched_rom.rom"/>`

Now before starting the VM, create the [kvm.conf](/kvm.conf) file and set the right values for the PCI addresses

Also add the `/etc/libvirt/hooks/qemu.d/win10/prepare/begin/start.sh` and the `/etc/libvirt/hooks/qemu.d/win10/release/end/revert.sh` files and modify them according to your needs.

And finally don't forget to remove the display and video devices from the VM.

Tada :tada: It works ! (Well it's supposed to)

## Warning

As I said, this is just notes that I wrote for me, and it might be incomplete, but I decided to share it because it might be useful to others. But I give 0 warranty that all I did will work for you (well it certainly won't work for you, but you can take just pieces of what I did). Just try and modify until it works for you, like I did :)

## Credit

Thanks to the following guides / posts for the help :

- https://gitlab.com/Karuri/vfio
- https://github.com/bryansteiner/gpu-passthrough-tutorial
- https://www.reddit.com/r/VFIO
