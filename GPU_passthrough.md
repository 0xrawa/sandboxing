# Nvidia GPU passthrough to QEMU VM

1. Enable IOMMU

Open `/etc/default/grub` and edit the following line:
- for Intel: `GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_iommu=on"`
- for AMD: `GRUB_CMDLINE_LINUX_DEFAULT="quiet splash amd_iommu=on"`

2. Find the IOMMU Group and PCI address of the address

Create a file iommu.sh and put the following code in it:
```bash
#!/bin/bash
for d in /sys/kernel/iommu_groups/*/devices/*; do
  n=${d#*/iommu_groups/*}; n=${n%%/*}
  printf 'IOMMU Group %s ' "$n"
  lspci -nns "${d##*/}"
done
```
