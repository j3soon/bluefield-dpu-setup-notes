# BlueField DPU Setup Notes

Unofficial notes for setting up and configuring NVIDIA BlueField DPUs on custom server systems (non-DGX platforms).

## Terminology

Linux Systems (Host/DPU/BMC):

- `Host`: The system running on the server.
  - In the optional Proxmox VE section, it will be further divided into `PVE` and `VM`.
  - In the remaining sections, the host refers to the server's operating system, regardless of whether it's running directly on hardware or within the VM of Proxmox VE.
- `DPU`: The system running on the DPU.
- `BMC`: The system on the board management controller of the DPU. This is an independent system that provides out-of-band management capabilities, separate from the DPU's main operating system.

## Hardware

- Server: G493-ZB3-AAP1-rev-1x [[ref](https://www.gigabyte.com/us/Enterprise/GPU-Server/G493-ZB3-AAP1-rev-1x)]
- BlueField-3 DPU: B3210E 900-9D3B6-00SC-EA0 [[ref](https://docs.nvidia.com/networking/display/bf3dpu)]
  - BMC Management Interface (RJ45): Ethernet Cable [[ref](https://docs.nvidia.com/networking/display/bluefieldbmcv2401/connecting+to+bmc+interfaces)]
  - DPU Eth/IB Port 0 (QSFP112): InfiniBand Cable [[ref](https://docs.nvidia.com/doca/archive/2-6-0/dpu+functional+diagram/index.html)]
  - DPU Eth/IB Port 1 (QSFP112): Empty
- V100 GPU: Tesla V100-PCIE-16GB

## Hardware Setup

> Require a supplementary 8-pin ATX power supply connectivity available through the external power supply connector .
>
> Do not link the CPU power cable to the BlueField-3 DPU PCIe ATX power connector, as their pin configurations differ. **Using the CPU power cable in this manner is strictly prohibited and can potentially damage the BlueField-3 DPU**. Please refer to [External PCIe Power Supply Connector Pins](https://docs.nvidia.com/networking/display/bf3dpu) for the external PCIe power supply pins.
>
> -- [Hardware Installation and PCIe Bifurcation](https://docs.nvidia.com/networking/display/bf3dpu/hardware+installation+and+pcie+bifurcation)

> - DPU BMC 1GbE interface connected to the management network via ToR
> - Remote Management Controller (RMC) connected to DPU BMC 1GbE via ToR
>    > **Info** \
>    > RMC is the platform for data center infrastructure managers to manage DPUs.**
> - DHCP server existing in the management network
> - An NVQual certified server
>
> -- [Prerequisites for Initial BlueField-3 Deployment](https://docs.nvidia.com/networking/display/bf3dpucontroller/bluefield-3+administrator+quick+start+guide#src-2729714701_BlueField3AdministratorQuickStartGuide-PrerequisitesforInitialBlueField-3Deployment)

References:

- [NVIDIA BlueField-3 Networking Platform User Guide](https://docs.nvidia.com/networking/display/bf3dpu)
- [BlueField-3 Administrator Quick Start Guide](https://docs.nvidia.com/networking/display/bf3dpu/bluefield-3+administrator+quick+start+guide)
- [Hardware Installation and PCIe Bifurcation](https://docs.nvidia.com/networking/display/bf3dpu/hardware+installation+and+pcie+bifurcation)

## Software

- (Optional) Proxmox VE 8.2.2
- Host OS
    - Operating System: Ubuntu 24.04.2 LTS
    - Kernel: Linux 6.8.0-54-generic
    - Architecture: x86-64
    - DOCA-Host: 2.9.2 LTS [[ref](https://developer.nvidia.com/doca-archive)]
- BMC
    - Operating System: NVIDIA Moonraker/RoyB BMC (OpenBMC     Project Reference Distro) BF-24.01-5
    - Kernel: Linux 5.15.50-e62bf17
    - Architecture: arm
- DPU
    - Operating System: Ubuntu 22.04.5 LTS
    - Kernel: Linux 5.15.0-1035-bluefield
    - Architecture: arm64
    - DOCA-BlueField: 2.9.2 [[ref](https://developer.nvidia.com/doca-archive)]
    - Mode: DPU Mode [[ref](https://docs.nvidia.com/doca/sdk/bluefield+modes+of+operation/index.html)]
    - DPU image and firmware: [[ref](https://docs.nvidia.com/networking/display/bluefield2dpuenug/bluefield+dpu+administrator+quick+start+guide)]
      ```
      $ sudo bfvcheck
      Beginning version check...

      -RECOMMENDED VERSIONS-
      ATF: v2.2(release):4.9.2-14-geeb9a6f94
      UEFI: 4.9.2-25-ge0f86cebd6
      FW: 32.43.2566

      -INSTALLED VERSIONS-
      ATF: v2.2(release):4.9.2-14-geeb9a6f94
      UEFI: 4.9.2-25-ge0f86cebd6
      FW: 32.43.2566

      Version check complete.
      No issues found.
      ```
    - BlueField OS image version: [[ref](https://docs.nvidia.com/networking/display/bluefielddpubspv420/deploying+bluefield+software+using+bfb+from+host#src-80580925_DeployingBlueFieldSoftwareUsingBFBfromHost-VerifyBFBisInstalled)]
      ```
      $ cat /etc/mlnx-release
      bf-bundle-2.9.2-31_25.02_ubuntu-22.04_prod
      ```

## (Optional) Proxmox VE Passthrough

> Please skip to [the next section](#system-testing) if Proxmox VE is not used.

- IOMMU Setup

    - Ensure that IOMMU (VT-d or AMD-Vi) is enabled in the BIOS/UEFI.

        ```bash
        lscpu | grep Virtualization
        ```

    - AMD enables it by default, check it using the following command:

        ```bash
        for d in /sys/kernel/iommu_groups/*/devices/*; do n=${d#*/iommu_groups/*}; n=${n%%/*}; printf 'IOMMU group %s ' "$n"; lspci -nns "${d##*/}"; done
        ```

        - It works when you see multiple groups and you can check which devices are properly isolated (no other devices in the same group except for PCI bridges) for PCI passthrough.

    - If it cannot be enabled, modify the GRUB configuration. Locate `GRUB_CMDLINE_LINUX_DEFAULT`, and for AMD, set it to:

        ```bash
        bash GRUB_CMDLINE_LINUX_DEFAULT="quiet iommu=pt"

        # for Intel
        GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
        ```

        - Verify whether IOMMU is enabled (though it's uncertain if this method works) by using:

        ```bash
        dmesg | grep -e DMAR -e IOMMU

        # works
        DMAR: IOMMU enabled
        ```

    - check NIC info

        ```bash
        bash lspci -nn | grep -i bluefield
        bash lspci -nn | grep -i nvidia
        ```

- Proxmox VE Setup

    - Find PCI ID

        ```bash
        lspci -nn | grep -i mellanox
        lspci -nn | grep -i nvidia

        # Take note of the device's PCI address (e.g., 0000:03:00.0) and Vendor:Device ID (e.g., 1b4b:xxxx).
        ```

    - Check vfio module

        ```bash
        lsmod | grep vfio
        ```

        - enable if there is no vfio module:

        ```bash
        echo "vfio" >> /etc/modules
        echo "vfio_iommu_type1" >> /etc/modules
        echo "vfio_pci" >> /etc/modules
        update-initramfs -u
        reboot
        dmesg | grep -i vfio
        ```

    - Configure VFIO: The BlueField card must be managed by `vfio-pci` to prevent the default driver from automatically loading.

        ```bash
        nano /etc/modprobe.d/vfio.conf

        # vendor device
        options vfio-pci ids=15b3:a2d2
        # update
        update-initramfs -u

        # Not sure if this is needed
        softdep mlx5_core pre: vfio-pci
        # Blacklist default driver (edit `/etc/modprobe.d/pve-blacklist.conf`)
        `blacklist mlx5_core`
        ```

    - Reboot the system and verify that the PCI device is bound to `vfio-pci`:

        ```bash
        lspci -nnk -d 1b4b:xxxx
        ```

- VM Setup

    - Create or stop the target VM, add the following line in Proxmox Web UI or directly edit the VM configuration file (e.g. `/etc/pve/qemu-server/<VMID>.conf`), replace `0000:03:00.0` with the PCI address of your BlueField card.

        ```bash
        hostpci0: 0000:03:00.0,pcie=1
        ```

        - If the card has multiple functions (multi-function device), you can add `hostpci1`, `hostpci2`, etc. or add `multifunction=on` (adjust as needed).

        - Check the VM

            - `lspci -nn | grep -i nvidia`

    - Appendix
        - V100 Passthrough in Proxmox VE GUI: `Datacenter > Resource Mappings > Add`
        - DPU Passthrough in Proxmox VE GUI: `VM > Hardware > Add > PCI Device`

References:

- [PCI(e) Passthrough](https://pve.proxmox.com/wiki/PCI(e)_Passthrough)

## Software Setup

### Host

Execute the following commands on the host.

Check PCI devices:

```bash
lspci -nn | grep -i mellanox
lspci -nn | grep -i nvidia
```

Install common packages:

```bash
sudo apt-get update
# Install pv for viewing progress of the commands below
sudo apt-get install -y pv
```

(Optional) Uninstall old DOCA-Host: [[ref](https://docs.nvidia.com/doca/sdk/doca-host+installation+and+upgrade/index.html)]

```bash
for f in $( dpkg --list | grep -E 'doca|flexio|dpa-gdbserver|dpa-stats|dpaeumgmt' | awk '{print $2}' ); do echo $f ; sudo apt remove --purge $f -y ; done
sudo /usr/sbin/ofed_uninstall.sh --force
sudo apt-get autoremove
```

Install DOCA-Host (DPU Driver) 2.9.2 LTS [[download](https://developer.nvidia.com/doca-2-9-2-download-archive?deployment_platform=Host-Server)]:

```bash
# DPU Driver (DOCA-Host)
wget https://www.mellanox.com/downloads/DOCA/DOCA_v2.9.2/host/doca-host_2.9.2-012000-24.10-ubuntu2404_amd64.deb
sudo dpkg -i doca-host_2.9.2-012000-24.10-ubuntu2404_amd64.deb
sudo apt-get update
sudo apt-get -y install doca-all
# Check DOCA-Host
dpkg -l | grep doca

# GPU Driver & CUDA
wget https://developer.download.nvidia.com/compute/cuda/12.8.0/local_installers/cuda_12.8.0_570.86.10_linux.run
sudo sh cuda_12.8.0_570.86.10_linux.run
# Check Driver
nvidia-smi
```

> **Note**: Build `macsec` driver issue
>
> Strangely, Ubuntu 24.04's kernel binary package doesn't seem to include the `macsec` driver, causing `mlx5_ib` not being able to load
>
> To mitigate the issue, we build the `macsec` driver ourselves:
>
> ```bash
> # Download macsec from kernel source
> wget https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/plain/drivers/net/macsec.c?h=v6.8 -O macsec.c
> 
> # Create Makefile
> cat << EOF > Makefile
> obj-m += macsec.o
> all:
> 	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
> EOF
> 
> make
> 
> sudo cp macsec.ko /lib/modules/$(uname -r)/kernel/drivers/net
> sudo depmod -a
> 
> # macsec module should be available
> modinfo macsec
> sudo modprobe macsec
> ```

Connect to the DPU via RShim: [[ref](https://docs.nvidia.com/networking/display/bluefielddpuosv460/deploying+bluefield+software+using+bfb+from+host#src-2571331391_DeployingBlueFieldSoftwareUsingBFBfromHost-FirmwareUpgrade)]

```bash
sudo systemctl enable --now rshim
sudo ip addr add 192.168.100.1/24 dev tmfifo_net0
ping 192.168.100.2
# connect to the DPU
ssh ubuntu@192.168.100.2
```

Change DPU to IB mode: [[ref](https://docs.nvidia.com/networking/display/mftv422/using+mlxconfig#src-99400487_safe-id-VXNpbmdtbHhjb25maWctVXNpbmdtbHhjb25maWd0b1NldElCL0VUSFBhcmFtZXRlcnM)]

```bash
# Note that this can also be done on DPU
sudo mlxconfig -d /dev/mst/mt41692_pciconf0 set LINK_TYPE_P1=1
sudo mlxconfig -d /dev/mst/mt41692_pciconf0 set LINK_TYPE_P2=1
# Cold reboot the machine
```

Deploying DPU OS Using BFB from Host: [[download](https://developer.nvidia.com/doca-2-9-2-download-archive?deployment_platform=BlueField&deployment_package=BF-Bundle&Distribution=Ubuntu&version=22.04&installer_type=BFB)] [[ref](https://docs.nvidia.com/networking/display/bluefielddpuosv470/deploying+bluefield+software+using+bfb+from+host#src-2821766645_DeployingBlueFieldSoftwareUsingBFBfromHost-InstallingUbuntuonBlueField)]

```sh
# update DOCA-BlueField to 2.9.2
wget https://content.mellanox.com/BlueField/BFBs/Ubuntu22.04/bf-bundle-2.9.2-31_25.02_ubuntu-22.04_prod.bfb
sudo bfb-install --bfb bf-bundle-2.9.2-31_25.02_ubuntu-22.04_prod.bfb --rshim /dev/rshim0
```

> (Optional, Unconfirmed) Update DPU Firmware: [[download](https://developer.nvidia.com/doca-2-9-2-download-archive?deployment_platform=BlueField&deployment_package=BF-FW-Bundle&installer_type=BFB)]
> 
> ```sh
> # update firmware
> wget https://content.mellanox.com/BlueField/FW-Bundle/bf-fwbundle-2.9.2-31_25.02-prod.bfb
> sudo bfb-install --bfb bf-fwbundle-2.9.2-31_25.02-prod.bfb --rshim rshim0
> ```

### DPU

Execute the following commands on the DPU.

```bash
# Check BlueField OS image version
cat /etc/mlnx-release
# Check DOCA-BlueField
dpkg -l | grep doca
```

Update DPU firmware: [[ref](https://docs.nvidia.com/networking/display/mlnxenv24010331/changes+and+new+features#src-2591728722_ChangesandNewFeatures-CustomerAffectingChanges)] [[firmware-tools](https://network.nvidia.com/products/adapter-software/firmware-tools/)] [[flint](https://docs.nvidia.com/networking/display/mftv4310/flint+%E2%80%93+firmware+burning+tool)] [[mlxfwmanager](https://docs.nvidia.com/networking/display/mftv4310/mlxfwmanager+%E2%80%93+firmware+update+and+query+tool)]

```bash
# check firmware
sudo mlxfwmanager --query
sudo flint -d /dev/mst/mt41692_pciconf0 q

# update firmware
sudo /opt/mellanox/mlnx-fw-updater/mlnx_fw_updater.pl

# force update
# sudo /opt/mellanox/mlnx-fw-updater/mlnx_fw_updater.pl --force-fw-update

# Need to cold reboot the machine
```

Resetting DPU: [[ref](https://docs.nvidia.com/networking/display/mftv4261lts/mstfwreset+%E2%80%93+loading+firmware+on+5th+generation+devices+tool)]

```bash
# Query for reset level required to load new firmware
sudo mlxfwreset -d /dev/mst/mt*pciconf0 q
```

> Output of the query command:
>
> ```
> Reset-levels:
> 0: Driver, PCI link, network link will remain up ("live-Patch")  -Supported     (default)
> 1: Only ARM side will not remain up ("Immediate reset").         -Not Supported
> 3: Driver restart and PCI reset                                  -Supported
> 4: Warm Reboot                                                   -Supported
> 
> Reset-types (relevant only for reset-levels 1,3,4):
> 0: Full chip reset                                               -Supported     (default)
> 1: Phy-less reset (keep network port active during reset)        -Not Supported
> 2: NIC only reset (for SoC devices)                              -Not Supported
> 3: ARM only reset                                                -Not Supported
> 4: ARM OS shut down                                              -Not Supported
> 
> Reset-sync (relevant only for reset-level 3):
> 0: Tool is the owner                                             -Not supported
> 1: Driver is the owner                                           -Supported     (default)
> ```

Debugging:

```bash
# collect all debug message in host
sudo /usr/sbin/sysinfo-snapshot.py
```

References:

- [NVIDIA Firmware Tools (MFT)](https://network.nvidia.com/products/adapter-software/firmware-tools/)
- [NVIDIA BlueField-3 DPU Firmware Download Center](https://network.nvidia.com/support/firmware/bluefield3/)

### BMC

IPMI:

```bash
# Check sensors
ipmitool sdr

# Power control
ipmitool chassis power
# chassis power Commands: status, on, off, cycle, reset, diag, soft

# Check power status
ipmitool chassis status

# Control the BMC itself
ipmitool mc
```

Redfish:

```bash
# Check BMC version
curl -k -u 'root:<password>' -H 'Content-Type: application/json' -X GET https://<bmc_ip>/redfish/v1/UpdateService/FirmwareInventory/BMC_Firmware
```

References:

- [Connecting to BMC Interfaces](https://docs.nvidia.com/networking/display/bluefieldbmcv2501/connecting+to+bmc+interfaces)
- [Reset Control](https://docs.nvidia.com/networking/display/bluefieldbmcv2410ltsu1/reset+control)
- [Table of Common Redfish Commands](https://docs.nvidia.com/networking/display/bluefieldbmcv2410ltsu1/table+of+common+redfish+commands)
- [NVIDIA BlueField Reset and Reboot Procedures](https://docs.nvidia.com/doca/sdk/nvidia+bluefield+reset+and+reboot+procedures/index.html)

## Contributors & Acknowledgements

Contributors: [@tsw303005](https://github.com/tsw303005), [Aiden128](https://github.com/Aiden128), [@YiPrograms](https://github.com/YiPrograms), and [@j3soon](https://github.com/j3soon).

This note has been made possible through the support of [LSA Lab](https://github.com/NTHU-LSALAB), and [NVIDIA AI Technology Center (NVAITC)](https://github.com/NVAITC).
