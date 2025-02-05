# GPU-Passthrough
A tutorial on how to achieve gpu passthrough in linux. This enables running windows applications in a virtual machine achieving near native performance.

It should be noted that I am passing through an NVIDIA GPU to my vm while using an AMD GPU to render the host machine's graphics. The host OS is Arch linux and the vm OS is windows 10.

### Preparation
Ensure your cpu supports GPU passthrough.
```bash
lscpu | grep "Virtualization"
```
The output should either be:
```
VT-x
```
or
```
AMD-V
```
You should also have two GPUs one for the host and one for the virtual machine.

### VFIO
VFIO stands for Virtual Function Input/Output. It isolates our second GPU so that it can be used 
inside the virtual machine.

Find the pci id of both the audio device and the vga controller of the GPU you want to use inside the virtual machine:
```bash
lspci -nn
```
Example output:
```
06:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP107 [GeForce GTX 1050] [10de:1c81] (rev a1)
06:00.1 Audio device [0403]: NVIDIA Corporation GP107GL High Definition Audio Controller [10de:0fb9] (rev a1)
```
This means our ids are:
```
10de:1c81,10de:0fb9
```

### IOMMU
IOMMU stands for Input-output memory management unit and it maps device-visible virtual addresses
(also called device addresses or memory mapped I/O addresses in this context) to physical addresses.

To enable iommu we have to modify grub:
```bash
sudo nano /etc/default/grub
```
Add to the GRUB_CMDLINE_LINUX_DEFAULT the parameters(after having removed all others):
```
intel_iommu=on iommu=pt vfio-pci.ids=10de:1c81,10de:0fb9
```
If you have and amd cpu replace intel_iommu with amd_iommu. "pt" means passthrough and the pci ids are the ids from the GPU we collected earlier.

Finally update grub to apply the boot changes:
```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```
Reboot.

### Initramfs
Create a vfio profile:
```bash
sudo touch /etc/modprobe.d/vfio.conf
sudo nano /etc/modprobe.d/vfio.conf
```
Inside the created file type:
```
options vfio-pci ids=10de:1c81,10de:0fb9
softdep nvidia pre: vfio-pci
```
The second option blocks the os from downloading the nvidia drivers on startup.

Update initramfs 
```bash
sudo mkinitcpio -p linux
```
Reboot.

Verify changes:
```bash
lspci -k | grep -E "vfio-pci|NVIDIA"
```


### Application
We have reached to the point where we are ready to launch our vm. Make sure you have installed the necessary software:
```bash
sudo pacman -S qemu # vm
sudo pacman -S virt-manager # vm manager gui
```

Enable the libvirt deamon:
```bash
# Check the service's status
systemctl status virtqemud
# Start the deamon if it is not running
sudo systemctl start virtqemud
sudo systemctl start libvirtd
# Optional: Enable it to start on boot
sudo systemctl enable virtqemud
sudo systemctl enable libvirtd
```
Enable the virtual storage deamon:
```bash
sudo systemctl start virtstoraged
sudo systemctl enable virtstoraged
```
Enable the virtual network deamon:
```bash
sudo systemctl start virtnetworkd
sudo systemctl enable virtnetworkd
```
Have the network start at startup each time:
```bash
sudo virsh net-start default
sudo virsh net-autostart default
```
Additional steps that might be needed:
```bash
sudo modprobe vfio-pci
echo "0000:06:00.1" | sudo tee /sys/bus/pci/drivers/snd_hda_intel/unbind
echo "0000:06:00.1" | sudo tee /sys/bus/pci/drivers/vfio-pci/bind
```
Download the virtio windows drivers and import them to the virtual machine:

Stable virtio-win ISO [link](https://github.com/virtio-win/virtio-win-pkg-scripts/blob/master/README.md)

Add the downloaded ISO to the "SATA CDROM 1" section of the vm.

Add the both the audio device and gpu to the vm from inside the vm settings:
```
  Add Hardware
       |
       v
PCI Host Device
```
Start the vm and install the imported virtio drivers.

Restart

The last step is downloading the GPU's drivers from the NVIDIA website. 

Sources
-------
- https://www.youtube.com/watch?v=g--fe8_kEcw
- https://www.youtube.com/watch?v=4m6eHhPypWI