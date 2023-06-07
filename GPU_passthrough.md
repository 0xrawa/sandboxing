# Nvidia GPU passthrough to QEMU VM

There are three scenarios you can find yourself in. See which one applies to you:
1. You have Laptop with a GPU next to the integrated graphics. 
2. You have a headless server that you access over a remote connection.
3. You have a Desktop environment.

## 1. Enable IOMMU

Open `/etc/default/grub` and edit the following line:
- for Intel: `GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_iommu=on"`
- for AMD: `GRUB_CMDLINE_LINUX_DEFAULT="quiet splash amd_iommu=on"`

Reboot: `sudo reboot`

## 2. Find the IOMMU Group and PCI address of the address

Create a file iommu.sh and put the following code in it:
```bash
#!/bin/bash
for d in /sys/kernel/iommu_groups/*/devices/*; do
  n=${d#*/iommu_groups/*}; n=${n%%/*}
  printf 'IOMMU Group %s ' "$n"
  lspci -nns "${d##*/}"
done
```

Run it with `sh iommu.sh | grep NVIDIA`. The script will give you all NVIDIA devices and their corresponding groups. Note the __IOMMU group__ and __PCI address__ of the VGA compatible controller and the associated Audio device. Both devices have to be passed through!

## 3. Unbind the GPU

If you are running in a Desktop environment, you have to stop the display manager otherwise the GPU will not be freed up. __Attention__: if you stop the display manager, the display will go blank. Make sure you are sshed into the system:
`sudo systemctl stop display-manager.service`

Unload the NVIDIA drivers:
`sudo modprobe -r nvidia_uvm`
`sudo modprobe -r nvidia_drm`
`sudo modprobe -r nvidia_modeset`
`sudo modprobe -r nvidia`

Detatch the GPU:
`sudo virsh nodedev-detach <gpu_pci_address>`
`sudo virsh nodedev-detach <audio_pci_address>`

Load VFIO:
`sudo modprobe vfio_pci`
`sudo modprobe vfio_iommu_type1`
`sudo modprobe vfio`

## 4. Start the VM

```bash
sudo qemu-system-x86_64 \
-enable-kvm \
-daemonize \
-hda auto_gpt.qcow2 \
-smp 4 \
-m 8164 \
-net user,hostfwd=tcp::10022-:22 \
-net nic \
-display none \
-cpu host,-svm \
-device vfio-pci,host=01:00.0,multifunction=on \
-device vfio-pci,host=01:00.1
```
