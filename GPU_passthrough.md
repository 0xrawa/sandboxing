# Nvidia GPU passthrough to QEMU VM

There are three scenarios you can find yourself in. See which one applies to you:
1. You have Laptop with a GPU next to the integrated graphics. [Go here](#laptop-gpu-passthrough)
2. You have a headless server that you access over a remote connection. [Go here](#server-gpu-passthrough)
3. You have a Desktop environment. [Go here](#desktop-gpu-passthrough)

First follow the two generic steps and then continue according to the scenario that applies to you.

## Generic Steps
### 1. Enable IOMMU

First we need to enable the [IOMMU](https://en.wikipedia.org/wiki/Input%E2%80%93output_memory_management_unit). 

Open `/etc/default/grub` and edit the following line:
- for Intel: `GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_iommu=on"`
- for AMD: `GRUB_CMDLINE_LINUX_DEFAULT="quiet splash amd_iommu=on"`

Reboot: `sudo reboot`.

### 2. IOMMU Group and PCI address of the GPU

Second we need to find the IOMMU Group and the PCI address of the GPU we want to passthrough. 

Create a file iommu.sh and put the following code in it:
```bash
#!/bin/bash
for d in /sys/kernel/iommu_groups/*/devices/*; do
  n=${d#*/iommu_groups/*}; n=${n%%/*}
  printf 'IOMMU Group %s ' "$n"
  lspci -nns "${d##*/}"
done
```

Run it with `sh iommu.sh | grep NVIDIA`. The script will give you all NVIDIA devices and their corresponding groups. Note the __IOMMU group__ and __PCI address__ of the VGA compatible controller and the associated Audio device. Both devices have to be passed through to the VM!

Now that these two generic steps are completed, choose which scenario applies to you and continue the instructions from there:
- [Laptop](#laptop-gpu-passthrough)
- [Server](#server-gpu-passthrough)
- [Desktop](#desktop-gpu-passthrough)

## Laptop GPU passthrough

This is the most difficult situation. But no worries, we will work it out together. The difficulty stems from the complication that in some cases the discrete GPU can not easily be released from the display driver. 

### 3. Switch to Integrated Graphics

First we need to make sure that the display is driven by the integrated graphics. We will use the installed nvidia prime profiles:
- for Intel: `sudo prime-select intel`
- for AMD: `sudo prime-select amd`

Reboot: `sudo reboot`.

### 4. Unbind the GPU

Now we can try to unbind the GPU. First run `nvidia-smi` to check that no processes are running anymore on the GPU. If that is the case we can release the NVIDIA drivers:

```bash
sudo modprobe -r nvidia_uvm
sudo modprobe -r nvidia_drm
sudo modprobe -r nvidia_modeset
sudo modprobe -r nvidia
```

Load the VFIO modules in the kernel:
```bash
sudo modprobe vfio_pci
sudo modprobe vfio_iommu_type1
sudo modprobe vfio
```

Now we unbind the GPU and register it as a VFIO device. Replace `<GPU_PCI_ADDRESS>` with the PCI address and `<GPU_ID_1>` `<GPU_ID_2>` with the device ID of your GPU, i.e `"0000:01:00.0"`, `"10de"` and `"22c4"`:

```bash
# Usually not necessary as `sudo modprobe -r nvidia_uvm` removes the driver from the GPU
echo -n <GPU_PCI_ADDRESS> | tee /sys/bus/pci/drivers/nvidia/unbind
```

```bash
# Creates the IOMMU group file
echo -n <GPU_ID_1> <GPU_ID_2> > /sys/bus/pci/drivers/vfio-pci/new_id
```

Next we do the same with the audio device:.

```bash
echo -n <AUDIO_PCI_ADDRESS> > /sys/bus/pci/devices/<AUDIO_PCI_ADDRESS>/driver/unbind
```

```bash
echo -n <AUDIO_PCI_ADDRESS> | tee /sys/bus/pci/drivers/vfio-pci/bind
```

To verify that everything worked out correctly, you can run `lspci -nnv -s <GPU_PCI_ADDRESS>`. The line should have changed from `Kernel driver in use: nvidia` to `Kernel driver in use: vfio-pci`.


Change the ownership of the created IOMMU group:
`sudo chown <user>:<group> /dev/vfio/<IOMMU_GROUP>`

### 5. Start the VM

```bash
qemu-system-x86_64 \
-enable-kvm \
-daemonize \
-hda gpu.qcow2 \
-smp 4 \
-m 8164 \
-net user,hostfwd=tcp::10022-:22 \
-net nic \
-display none \
-cpu host,-svm \
-device vfio-pci,host=01:00.0,multifunction=on \
-device vfio-pci,host=01:00.1
```

## Server GPU Passthrough

TODO

## Desktop GPU Passthrough

### 3. Unbind the GPU

If you are running in a Desktop environment, you have to stop the display manager otherwise the GPU will not be freed up. __Attention__: if you stop the display manager, the display will go blank. Make sure you are sshed into the system:
`sudo systemctl stop display-manager.service`

Unload the NVIDIA drivers:
```bash
sudo modprobe -r nvidia_uvm
sudo modprobe -r nvidia_drm
sudo modprobe -r nvidia_modeset
sudo modprobe -r nvidia
```

Detatch the GPU:
`sudo virsh nodedev-detach <gpu_pci_address>`
`sudo virsh nodedev-detach <audio_pci_address>`

Load VFIO:
```bash
sudo modprobe vfio_pci
sudo modprobe vfio_iommu_type1
sudo modprobe vfio
```

### 4. Start the VM

```bash
qemu-system-x86_64 \
-enable-kvm \
-daemonize \
-hda gpu.qcow2 \
-smp 4 \
-m 8164 \
-net user,hostfwd=tcp::10022-:22 \
-net nic \
-display none \
-cpu host,-svm \
-device vfio-pci,host=01:00.0,multifunction=on \
-device vfio-pci,host=01:00.1
```


## Errors

### Cannot allocate memory

```bash
qemu-system-x86_64: -device vfio-pci,host=01:00.0,multifunction=on: VFIO_MAP_DMA: -12
qemu-system-x86_64: -device vfio-pci,host=01:00.0,multifunction=on: VFIO_MAP_DMA: -12
qemu-system-x86_64: -device vfio-pci,host=01:00.0,multifunction=on: vfio 0000:01:00.0: failed to setup container for group 14: memory listener initialization failed: Region pc.ram: vfio_dma_map(0x55f3ffb9a3d0, 0x100000, 0xbff00000, 0x7f4aa1b00000) = -12 (Cannot allocate memory)
```
The error message vfio_dma_map() = -12 (Cannot allocate memory) means that the system failed to allocate enough contiguous memory to map the region for the device. This is typically due to memory fragmentation within the Linux kernel.

One of the most common solutions to this issue is to increase the amount of hugepages on your system, which reserves large blocks of contiguous memory that can be used for these device mappings. 

Increase the size of the pages: `echo 4096 | sudo tee /proc/sys/vm/nr_hugepages`

Add the following flag when starting `qemu-system`: `-mem-path /dev/hugepages`


