# Intel Hardware Virtualization & SR-IOV Configuration Reference

Single Root I/O Virtualization (SR-IOV) is a specification that allows a single PCIe physical device (such as an Intel NIC) to appear as multiple separate virtual PCIe devices (Virtual Functions).

## 1. BIOS/UEFI Prerequisites
To utilize SR-IOV, virtualisation features must be enabled in the platform BIOS/UEFI:
- **Intel VT-d** (Virtualization Technology for Directed I/O) must be set to `Enabled`.
- **SR-IOV Global Support** (or similar PCIe virtualization settings) must be set to `Enabled`.

## 2. Host OS Kernel Configuration (Linux)
You must enable IOMMU in the host Linux kernel to support safe PCI device passthrough:
1. Edit the GRUB configuration file (typically `/etc/default/grub`).
2. Add `intel_iommu=on` and `iommu=pt` to the `GRUB_CMDLINE_LINUX_DEFAULT` parameter.
   - *Example:* `GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"`
3. Update GRUB configuration:
   - For Debian/Ubuntu: `sudo update-grub`
   - For RHEL/CentOS: `sudo grub2-mkconfig -o /boot/grub2/grub.cfg` (or matching path)
4. Add the required virtual function IO (VFIO) modules to the boot load list (e.g., `/etc/modules` or `/etc/modules-load.d/`):
   - `vfio`
   - `vfio_iommu_type1`
   - `vfio_pci`
   - `vfio_virqfd`
5. Reboot the host.

## 3. Creating Virtual Functions (VFs)
After booting the host with IOMMU enabled:
1. Locate your Intel NIC device using `lspci` or `ip link`.
2. Enable Virtual Functions using the sysfs interface. Write the number of desired virtual functions (e.g., `4`) to `sriov_numvfs` of your physical network interface:
   ```bash
   echo 4 > /sys/class/net/<interface_name>/device/sriov_numvfs
   ```
3. Verify that the Virtual Functions have been initialized:
   ```bash
   lspci | grep -i ether
   ```
   You should see the virtual functions listed under the physical controller.

## 4. Persistence
Writing directly to `/sys/class/...` is non-persistent and will reset upon reboot. To make SR-IOV virtual functions persistent:
- Use startup services or systemd-udev rules to set the number of virtual functions.
- Or configure your network management framework (e.g., Netplan config with `virtual-function-count` or interface config scripts in `/etc/sysconfig/network-scripts/`).
