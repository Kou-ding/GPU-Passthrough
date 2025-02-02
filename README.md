# GPU-Passthrough
A tutorial on how to achieve gpu passthrough in linux. This enables running windows applications in a virtual machine achieving near native performance.
\\
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
VFIO stands for Virtual Function Input/Output 

Find the pci id of both the audio device and the vga controller of the GPU you want to use inside the virtual machine:
```bash
lspci -nn
```
Example output:
```
01:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Tonga PRO [Radeon R9 285/380] [1002:6939] (rev f1)
01:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Tonga HDMI Audio [Radeon R9 285/380] [1002:aad8]
```
This means our ids are:
```
1002:6939,1002:aad8
```

### IOMMU
IOMMU stands for 
To enable iommu we have to modify grub:
```bash
sudo nano /etc/default/grub
```
Add to the GRUB_CMDLINE_LINUX_DEFAULT the parameters(after having removed all others):
```
intel_iommu=on iommu=pt vfio-pci.ids=1002:6939,1002:aad8
```
If you have and amd cpu replace intel_iommu with amd_iommu. pt means passthrough and the the pci ids are the ids from the GPU we collected earlier.
\\
Finally update grub to apply the boot changes:
```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```
Reboot.

### 
Create a vfio profile:
```bash
sudo touch /etc/modprobe.d/vfio.conf
sudo nano /etc/modprobe.d/vfio.conf
```
Inside the created file type:
```
options vfio-pci ids=1002:6939,1002:aad8
softdep nvidia pre: vfio-pci
```
The second option blocks the os from downloading the nvidia drivers on startup.
\\
Update initramfs 
```bash
sudo mkinticpio -p linux
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
# Optional: Enable it to start on boot
sudo systemctl enable virtqemud
```

Download the virtio windows drivers and import them to the virtual machine:

Add the both the audio device and gpu to the vm from inside the vm settings:
```
  Add Hardware
       |
       v
PCI Host Device
```
Start the vm and install the imported virtio drivers.
\\
Restart
\\
The last step is downloading the GPU's drivers from the NVIDIA website. 

Source
-------
https://www.youtube.com/watch?v=g--fe8_kEcw