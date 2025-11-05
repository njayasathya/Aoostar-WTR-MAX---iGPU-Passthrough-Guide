# Aoostar WTR Max iGPU Passthrough - Complete Guide
## AMD R7 PRO 8845HS with Radeon 780M to TrueNAS VM on Proxmox 9.0.11

**Date Completed:** November 5, 2025  
**System:** Aoostar WTR Max  
**CPU:** AMD R7 PRO 8845HS  
**iGPU:** AMD Radeon 780M (Phoenix3) - Device ID: 1002:1900  
**Proxmox Version:** 9.0.11  
**TrueNAS VM ID:** 110

---

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Step-by-Step Configuration](#step-by-step-configuration)
3. [Verification Steps](#verification-steps)
4. [Expected Outputs](#expected-outputs)
5. [Troubleshooting](#troubleshooting)
6. [Using the GPU in TrueNAS](#using-the-gpu-in-truenas)

---

## Prerequisites

- Aoostar WTR Max with AMD R7 PRO 8845HS
- Proxmox VE 9.0.11 installed
- TrueNAS VM already created
- SSH access to Proxmox host
- **IMPORTANT:** This is the only GPU, so you'll lose console output after reboot

---

## Step-by-Step Configuration

### Step 1: Verify GPU Information

**Command:**
```bash
lspci -nn | grep -i vga
```

**Expected Output:**
```
01:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Phoenix3 [1002:1900] (rev d5)
```

**Check current driver:**
```bash
lspci -k -s 01:00.0
```

**Expected Output:**
```
01:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Phoenix3 (rev d5)
        Subsystem: Advanced Micro Devices, Inc. [AMD/ATI] Device 0124
        Kernel driver in use: amdgpu
        Kernel modules: amdgpu
```

---

### Step 2: Identify TrueNAS VM

**Command:**
```bash
qm list
```

**Expected Output:**
```
      VMID NAME                 STATUS     MEM(MB)    BOOTDISK(GB) PID       
       110 TrueNAS-Home         running    32768             32.00 172430    
       111 TrueNAS-Test         stopped    8192              32.00 0         
```

Note your TrueNAS VM ID (in this case: **110**)

---

### Step 3: Stop VM and Backup Configuration

**Commands:**
```bash
# Stop the VM
qm stop 110

# Backup VM configuration
cp /etc/pve/qemu-server/110.conf /etc/pve/qemu-server/110.conf.backup-$(date +%Y%m%d-%H%M%S)

# View current configuration
cat /etc/pve/qemu-server/110.conf
```

**Expected Output:**
```
agent: 1
balloon: 0
boot: order=scsi0;net0
cores: 12
cpu: host
machine: q35
memory: 32768
name: TrueNAS-Home
...
```

---

### Step 4: Dump VBIOS (CRITICAL - Do Before Making Changes!)

**Command:**
```bash
# Using debugfs method (works better on modern systems)
cp /sys/kernel/debug/dri/0/amdgpu_vbios /usr/share/kvm/vbios_1002_1900.bin
chmod 644 /usr/share/kvm/vbios_1002_1900.bin
ls -lh /usr/share/kvm/vbios_1002_1900.bin
```

**Expected Output:**
```
-rw-r--r-- 1 root root 17K Nov  5 13:11 /usr/share/kvm/vbios_1002_1900.bin
```

**Alternative method (if above doesn't work):**
```bash
cd /sys/bus/pci/devices/0000:01:00.0/
echo 1 > rom
cat rom > /usr/share/kvm/vbios_1002_1900.bin
echo 0 > rom
chmod 644 /usr/share/kvm/vbios_1002_1900.bin
```

---

### Step 5: Configure Module Blacklisting

**Commands:**
```bash
# Blacklist amdgpu driver
echo "blacklist amdgpu" > /etc/modprobe.d/amdgpu.conf
cat /etc/modprobe.d/amdgpu.conf
```

**Expected Output:**
```
blacklist amdgpu
```

**Configure VFIO-PCI:**
```bash
echo "options vfio-pci ids=1002:1900 disable_vga=1" > /etc/modprobe.d/vfio-vga.conf
cat /etc/modprobe.d/vfio-vga.conf
```

**Expected Output:**
```
options vfio-pci ids=1002:1900 disable_vga=1
```

---

### Step 6: Update GRUB Configuration

**Commands:**
```bash
# Backup GRUB config
cp /etc/default/grub /etc/default/grub.backup-$(date +%Y%m%d-%H%M%S)

# Update GRUB with kernel parameters
sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="quiet"/GRUB_CMDLINE_LINUX_DEFAULT="amd-pstate=passive quiet video=efifb:off initcall_blacklist=sysfb_init textonly iommu=pt pcie_acs_override=downstream,multifunction"/' /etc/default/grub

# Verify the change
grep GRUB_CMDLINE_LINUX_DEFAULT /etc/default/grub
```

**Expected Output:**
```
GRUB_CMDLINE_LINUX_DEFAULT="amd-pstate=passive quiet video=efifb:off initcall_blacklist=sysfb_init textonly iommu=pt pcie_acs_override=downstream,multifunction"
```

**Update GRUB:**
```bash
update-grub
```

**Expected Output:**
```
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-6.14.11-4-pve
Found initrd image: /boot/initrd.img-6.14.11-4-pve
...
done
```

**Kernel Parameters Explained:**
- `amd-pstate=passive` - AMD CPU power management
- `video=efifb:off` - Disables EFI framebuffer
- `initcall_blacklist=sysfb_init` - Prevents system framebuffer initialization
- `textonly` - Text-only console mode
- `iommu=pt` - IOMMU passthrough mode
- `pcie_acs_override=downstream,multifunction` - Overrides PCIe ACS for better IOMMU grouping

---

### Step 7: Update Initramfs and Boot Entries

**Commands:**
```bash
# Update initramfs
update-initramfs -u -k all

# Refresh Proxmox boot tool
proxmox-boot-tool refresh
```

**Expected Output:**
```
update-initramfs: Generating /boot/initrd.img-6.14.11-4-pve
Running hook script 'zz-proxmox-boot'..
...
```

---

### Step 8: Add GPU to VM Configuration

**Note:** Check if you have existing hostpci devices. If hostpci0-4 are already used (like for NVMe drives), use the next available number.

**Command:**
```bash
# Add GPU as hostpci5 (adjust number if needed)
echo "hostpci5: 0000:01:00.0,pcie=1,romfile=vbios_1002_1900.bin" >> /etc/pve/qemu-server/110.conf

# Verify it was added
tail -5 /etc/pve/qemu-server/110.conf
```

**Expected Output:**
```
scsihw: virtio-scsi-single
smbios1: uuid=f2499600-370f-400c-a9b3-3bf848b0b749
sockets: 1
vmgenid: fc6a75e8-882c-4843-b2d5-94c309c03df5
hostpci5: 0000:01:00.0,pcie=1,romfile=vbios_1002_1900.bin
```

---

### Step 9: Reboot Proxmox

⚠️ **WARNING:** You will lose console output after reboot. Ensure SSH access is working!

**Command:**
```bash
reboot
```

**What happens:**
- Connection will drop
- System will reboot (takes 1-2 minutes)
- Console output will be lost (expected)
- SSH will become available again after boot

**Wait 2-3 minutes, then reconnect via SSH**

---

## Verification Steps

### On Proxmox Host (After Reboot)

#### Verify GPU is Using vfio-pci Driver

**Command:**
```bash
lspci -k -s 01:00.0
```

**Expected Output:**
```
01:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Phoenix3 (rev d5)
        Subsystem: Advanced Micro Devices, Inc. [AMD/ATI] Device 0124
        Kernel driver in use: vfio-pci
        Kernel modules: amdgpu
```

✅ **Key Line:** `Kernel driver in use: vfio-pci`

---

#### Verify IOMMU is Enabled

**Command:**
```bash
dmesg | grep -i iommu | head -10
```

**Expected Output:**
```
[    0.000000] Command line: BOOT_IMAGE=/boot/vmlinuz-6.14.11-4-pve root=/dev/mapper/pve-root ro amd-pstate=passive quiet video=efifb:off initcall_blacklist=sysfb_init textonly iommu=pt pcie_acs_override=downstream,multifunction
[    0.000000] Warning: PCIe ACS overrides enabled; This may allow non-IOMMU protected peer-to-peer DMA
[    0.380884] iommu: Default domain type: Passthrough (set via kernel command line)
[    0.424257] pci 0000:00:00.2: AMD-Vi: IOMMU performance counters supported
```

✅ **Key Lines:** 
- IOMMU parameters in command line
- "IOMMU performance counters supported"

---

#### Verify VFIO Modules are Loaded

**Command:**
```bash
lsmod | grep vfio
```

**Expected Output:**
```
vfio_pci               16384  6
vfio_pci_core          86016  1 vfio_pci
vfio_iommu_type1       49152  1
vfio                   65536  23 vfio_pci_core,vfio_iommu_type1,vfio_pci
iommufd               110592  1 vfio
irqbypass              12288  56 vfio_pci_core,kvm
```

✅ **All VFIO modules loaded**

---

#### Check VM Status

**Command:**
```bash
qm status 110
```

**Expected Output:**
```
status: running
```

If stopped, start it:
```bash
qm start 110
```

---

### In TrueNAS VM

#### Check GPU is Visible

**Command:**
```bash
lspci | grep VGA
```

**Expected Output:**
```
00:01.0 VGA compatible controller: Device 1234:1111 (rev 02)
02:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Phoenix3 (rev d5)
```

✅ **Two VGA controllers:**
- `00:01.0` - Virtual display (bochs) for console
- `02:00.0` - Your AMD Radeon 780M GPU

---

#### Check DRI Devices

**Command:**
```bash
ls -l /dev/dri/
```

**Expected Output:**
```
total 0
drwxr-xr-x 2 root root        100 Nov  5 13:15 by-path
crw-rw---- 1 root video  226,   0 Nov  5 13:15 card0
crw-rw---- 1 root video  226,   1 Nov  5 13:15 card1
crw-rw---- 1 root render 226, 128 Nov  5 13:15 renderD128
```

✅ **Key Files:**
- `card0` - Virtual display
- `card1` - AMD GPU
- `renderD128` - Render device (used by apps for hardware acceleration)

---

#### Verify AMD Driver is Loaded

**Command:**
```bash
lspci -v | grep -A 10 VGA
```

**Expected Output:**
```
00:01.0 VGA compatible controller: Device 1234:1111 (rev 02) (prog-if 00 [VGA controller])
        Subsystem: Red Hat, Inc. Device 1100
        Flags: fast devsel
        Memory at fc000000 (32-bit, prefetchable) [size=16M]
        Memory at fea14000 (32-bit, non-prefetchable) [size=4K]
        Expansion ROM at 000c0000 [disabled] [size=128K]
        Kernel driver in use: bochs-drm
        Kernel modules: bochs

00:10.0 PCI bridge: Red Hat, Inc. QEMU PCIe Root port (prog-if 00 [Normal decode])
        Subsystem: Red Hat, Inc. QEMU PCIe Root port
--
02:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Phoenix3 (rev d5) (prog-if 00 [VGA controller])
        Subsystem: Advanced Micro Devices, Inc. [AMD/ATI] Phoenix3
        Physical Slot: 0-2
        Flags: bus master, fast devsel, latency 0, IRQ 20
        Memory at e1000000000 (64-bit, prefetchable) [size=256M]
        Memory at e1010000000 (64-bit, prefetchable) [size=2M]
        I/O ports at 5000 [size=256]
        Memory at fe600000 (32-bit, non-prefetchable) [size=512K]
        Expansion ROM at fe680000 [disabled] [size=32K]
        Capabilities: <access denied>
        Kernel driver in use: amdgpu
```

✅ **Key Line:** `Kernel driver in use: amdgpu` (for device 02:00.0)

---

#### Check AMD GPU Kernel Messages (Optional)

**Command:**
```bash
sudo dmesg | grep amdgpu
```

**Note:** Regular users will get "Operation not permitted" error. This is NORMAL and just a permission issue, not a GPU problem!

**To run with proper permissions:**
```bash
sudo dmesg | grep amdgpu | head -20
```

**Expected Output (sample):**
```

[   19.038626] [drm] amdgpu kernel modesetting enabled.
[   19.038837] amdgpu: Virtual CRAT table created for CPU
[   19.038855] amdgpu: Topology: Add CPU node
[   19.134812] amdgpu 0000:02:00.0: amdgpu: Fetched VBIOS from ROM BAR
[   19.134895] amdgpu: ATOM BIOS: 113-PHXGENERIC-001
[   19.145682] amdgpu 0000:02:00.0: amdgpu: Trusted Memory Zone (TMZ) feature enabled
[   19.145751] amdgpu 0000:02:00.0: amdgpu: VRAM: 2048M 0x0000008000000000 - 0x000000807FFFFFFF (2048M used)
[   19.145753] amdgpu 0000:02:00.0: amdgpu: GART: 512M 0x00007FFF00000000 - 0x00007FFF1FFFFFFF
[   19.145861] [drm] amdgpu: 2048M of VRAM memory ready
[   19.145863] [drm] amdgpu: 16050M of GTT memory ready.
[   19.173723] amdgpu 0000:02:00.0: amdgpu: reserve 0x4000000 from 0x8078000000 for PSP TMR
[   19.713527] amdgpu 0000:02:00.0: amdgpu: RAS: optional ras ta ucode is not available
[   19.719626] amdgpu 0000:02:00.0: amdgpu: RAP: optional rap ta ucode is not available
[   19.719628] amdgpu 0000:02:00.0: amdgpu: SECUREDISPLAY: securedisplay ta ucode is not available
[   19.753381] amdgpu 0000:02:00.0: amdgpu: SMU is initialized successfully!
[   19.768480] kfd kfd: amdgpu: Allocated 3969056 bytes on gart
[   19.768501] kfd kfd: amdgpu: Total number of KFD nodes to be created: 1
[   19.768632] amdgpu: Virtual CRAT table created for GPU
[   19.768824] amdgpu: Topology: Add dGPU node [0x1900:0x1002]
[   19.768827] kfd kfd: amdgpu: added device 1002:1900
```

---

#### Alternative GPU Checks (No sudo needed)

**Check if amdgpu module is loaded:**
```bash
lsmod | grep amdgpu
```

**Expected Output:**
```
amdgpu              14282752  0
amdxcp                 12288  1 amdgpu
drm_exec               12288  1 amdgpu
gpu_sched              65536  1 amdgpu
drm_buddy              24576  1 amdgpu
drm_suballoc_helper    12288  1 amdgpu
drm_display_helper    270336  1 amdgpu
i2c_algo_bit           12288  1 amdgpu
drm_ttm_helper         16384  4 bochs,drm_vram_helper,amdgpu
video                  81920  1 amdgpu
ttm                   106496  3 drm_vram_helper,amdgpu,drm_ttm_helper
crc16                  12288  1 amdgpu
drm_kms_helper        249856  5 bochs,drm_vram_helper,drm_display_helper,amdgpu,drm_ttm_helper
drm                   765952  13 gpu_sched,drm_kms_helper,drm_exec,bochs,drm_vram_helper,drm_suballoc_helper,drm_display_helper,drm_buddy,amdgpu,drm_ttm_helper,ttm,amdxcp
...
```

**Check by-path symlinks:**
```bash
ls -l /dev/dri/by-path/
```

**Expected Output:**
```
total 0
lrwxrwxrwx 1 root root 8 Nov  5 13:15 pci-0000:00:01.0-card -> ../card0
lrwxrwxrwx 1 root root 8 Nov  5 13:15 pci-0000:02:00.0-card -> ../card1
lrwxrwxrwx 1 root root 13 Nov  5 13:15 pci-0000:02:00.0-render -> ../renderD128
```

✅ **Key:** `pci-0000:02:00.0-render -> ../renderD128` shows AMD GPU render device

---

## Success Indicators

### ✅ Configuration is Successful When:

**On Proxmox Host:**
1. `lspci -k` shows GPU using `vfio-pci` driver
2. IOMMU is enabled in kernel messages
3. VFIO modules are loaded (`lsmod | grep vfio`)
4. VM starts without errors

**In TrueNAS VM:**
1. `lspci | grep VGA` shows AMD Phoenix3 GPU
2. `ls /dev/dri/` shows `card1` and `renderD128`
3. `lspci -v` shows `Kernel driver in use: amdgpu`
4. No error messages about GPU in dmesg (if you can check with sudo)

---

## Using the GPU in TrueNAS

### For Hardware Transcoding

The render device `/dev/dri/renderD128` is what applications use for hardware acceleration.

#### Plex Media Server

1. Install Plex in TrueNAS
2. When configuring the Plex app/container:
   - Enable GPU passthrough
   - Add device: `/dev/dri/renderD128`
   - Or add entire `/dev/dri` directory
3. In Plex Settings → Transcoder:
   - Enable "Use hardware acceleration when available"
   - Select appropriate codec support

#### Jellyfin

1. Install Jellyfin
2. Pass through `/dev/dri/renderD128` to container
3. In Jellyfin Dashboard → Playback:
   - Hardware acceleration: **Video Acceleration API (VAAPI)**
   - VA-API Device: `/dev/dri/renderD128`
   - Enable hardware decoding for desired codecs

#### Docker Containers

Add to your `docker-compose.yml`:
```yaml
services:
  your-app:
    devices:
      - /dev/dri/renderD128:/dev/dri/renderD128
    # Or pass entire DRI directory
    # - /dev/dri:/dev/dri
    group_add:
      - video  # Add container to video group
```

Or docker run command:
```bash
docker run --device=/dev/dri/renderD128:/dev/dri/renderD128 ...
```

#### TrueNAS SCALE Apps

1. When installing apps (Plex, Jellyfin, etc.):
2. Look for "GPU Allocation" or "Resources and Devices" section
3. Select the AMD GPU or specify `/dev/dri/renderD128`
4. Some apps auto-detect available GPUs

---

## Testing GPU Functionality

### Install vainfo (if available in TrueNAS)

```bash
sudo apt install vainfo  # For Debian/Ubuntu-based TrueNAS SCALE
# or
sudo pkg install libva-utils  # For FreeBSD-based TrueNAS CORE
```

**Test GPU capabilities:**
```bash
sudo vainfo --display drm --device /dev/dri/renderD128
```

**Expected Output:**
```
libva info: VA-API version 1.x.x
libva info: Trying to open /usr/lib/x86_64-linux-gnu/dri/radeonsi_drv_video.so
libva info: Found init function __vaDriverInit_1_x
libva info: va_openDriver() returns 0
vainfo: VA-API version: 1.x (libva x.x.x)
vainfo: Driver version: Mesa Gallium driver x.x.x
vainfo: Supported profile and entrypoints
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264Main               : VAEntrypointEncSlice
      VAProfileHEVCMain               : VAEntrypointVLD
      VAProfileHEVCMain               : VAEntrypointEncSlice
      ...
```

---

## Troubleshooting

### GPU Not Claimed by vfio-pci After Reboot

**Check what driver is using the GPU:**
```bash
lspci -k | grep -A 3 VGA
```

**Verify kernel command line was applied:**
```bash
cat /proc/cmdline
```

Should show: `iommu=pt pcie_acs_override=downstream,multifunction` etc.

**Check if modules are blacklisted:**
```bash
cat /etc/modprobe.d/amdgpu.conf
cat /etc/modprobe.d/vfio-vga.conf
```

**Rebuild initramfs if needed:**
```bash
update-initramfs -u -k all
proxmox-boot-tool refresh
reboot
```

---

### VM Won't Start

**Check VM logs:**
```bash
cat /var/log/qemu-server/110.log
```

**Common issues:**
- **Missing VBIOS file:** Check `/usr/share/kvm/vbios_1002_1900.bin` exists
- **Wrong PCI address:** Verify `01:00.0` is correct with `lspci`
- **Permissions:** Check VBIOS file permissions (`chmod 644`)
- **OVMF/UEFI:** Ensure VM is using UEFI if needed

**Verify VBIOS exists:**
```bash
ls -lh /usr/share/kvm/vbios_1002_1900.bin
```

---

### No GPU in TrueNAS VM

**Verify GPU is passed through:**
```bash
cat /etc/pve/qemu-server/110.conf | grep hostpci
```

**Check VM is running:**
```bash
qm status 110
```

**Try without VBIOS ROM:**
Edit `/etc/pve/qemu-server/110.conf` and change:
```
hostpci5: 0000:01:00.0,pcie=1,romfile=vbios_1002_1900.bin
```
To:
```
hostpci5: 0000:01:00.0,pcie=1
```

Then restart VM: `qm stop 110 && qm start 110`

---

### GPU Reset Issues

AMD GPUs can have reset issues when VM restarts. If problems occur:

**Install vendor-reset module:**
```bash
apt update
apt install pve-headers-$(uname -r) git dkms build-essential
git clone https://github.com/gnif/vendor-reset.git
cd vendor-reset
make
make install
echo "vendor-reset" >> /etc/modules
reboot
```

**Verify it's loaded:**
```bash
lsmod | grep vendor_reset
```

---

### Performance Issues

**Check IOMMU grouping:**
```bash
#!/bin/bash
for d in /sys/kernel/iommu_groups/*/devices/*; do
    n=${d#*/iommu_groups/*}; n=${n%%/*}
    printf 'IOMMU Group %s ' "$n"
    lspci -nns "${d##*/}"
done
```

**Ensure VM settings are optimal:**
- Machine type: `q35`
- CPU: `host`
- PCIe: `pcie=1` in hostpci line

---

## Important Files and Locations

### Proxmox Host

| File/Directory | Purpose |
|----------------|---------|
| `/etc/modprobe.d/amdgpu.conf` | Blacklists amdgpu driver |
| `/etc/modprobe.d/vfio-vga.conf` | VFIO-PCI configuration |
| `/etc/default/grub` | GRUB kernel parameters |
| `/usr/share/kvm/vbios_1002_1900.bin` | VBIOS ROM dump (17KB) |
| `/etc/pve/qemu-server/110.conf` | VM 110 configuration |
| `/etc/pve/qemu-server/110.conf.backup-*` | VM config backups |
| `/var/log/qemu-server/110.log` | VM 110 logs |

---

## Reverting Changes

If you need to undo the GPU passthrough:

### Remove GPU from VM

```bash
# Remove hostpci5 line from VM config
nano /etc/pve/qemu-server/110.conf
# Delete the line: hostpci5: 0000:01:00.0,pcie=1,romfile=vbios_1002_1900.bin

# Restart VM
qm stop 110
qm start 110
```

### Re-enable amdgpu on Proxmox Host

```bash
# Remove blacklist
rm /etc/modprobe.d/amdgpu.conf
rm /etc/modprobe.d/vfio-vga.conf

# Restore original GRUB config
cp /etc/default/grub.backup-* /etc/default/grub
update-grub

# Update initramfs
update-initramfs -u -k all
proxmox-boot-tool refresh

# Reboot
reboot
```

After reboot, Proxmox will use the GPU again for console output.

---

## Additional Notes

### Single GPU Considerations

- ⚠️ **Console Output Lost:** Since this is the only GPU, Proxmox console will be blank after configuration
- ✅ **SSH Always Works:** Always ensure SSH access is configured
- 💡 **Alternative:** Use a USB-to-HDMI adapter or remote management (IPMI) if available

### VBIOS ROM Importance

- Some GPUs require the VBIOS ROM file, others don't
- The Radeon 780M works reliably WITH the VBIOS
- Size should be ~17KB for this GPU
- Must be dumped BEFORE making configuration changes

### VM Configuration

- **Machine Type:** Use `q35` for better PCIe support
- **CPU Type:** Use `host` for best performance
- **BIOS:** OVMF (UEFI) recommended but not required
- **Memory:** Allocate enough RAM for GPU operations

### Performance Expectations

- Hardware transcoding in Plex/Jellyfin will be significantly faster
- GPU-accelerated encoding/decoding available
- Multiple simultaneous transcodes possible
- Power efficiency better than CPU-only transcoding

---

## Summary

This guide successfully configured GPU passthrough of the AMD Radeon 780M (Phoenix3) integrated GPU from an Aoostar WTR Max to a TrueNAS VM running on Proxmox VE 9.0.11.

**Key achievements:**
- ✅ GPU isolated with vfio-pci on Proxmox host
- ✅ IOMMU configured and functioning
- ✅ VBIOS successfully dumped and configured
- ✅ GPU visible and functional in TrueNAS VM
- ✅ Render device available for hardware acceleration
- ✅ Ready for use with Plex, Jellyfin, and other apps

**Configuration is permanent** and will persist across reboots. The GPU is now available for hardware transcoding and acceleration in TrueNAS applications.

---

## Quick Reference Commands

### Proxmox Host Status Check
```bash
# Check GPU driver
lspci -k -s 01:00.0

# Check VFIO loaded
lsmod | grep vfio

# Check VM status
qm status 110

# Start/stop VM
qm start 110
qm stop 110
```

### TrueNAS VM GPU Check
```bash
# Check GPU visible
lspci | grep VGA

# Check render devices
ls -l /dev/dri/

# Check AMD driver loaded
lspci -v -s 02:00.0 | grep driver

# Check with sudo
sudo dmesg | grep amdgpu
```

---

**Guide created:** November 5, 2025  
**Status:** ✅ Successfully tested and verified  
**Hardware:** Aoostar WTR Max, AMD R7 PRO 8845HS, Radeon 780M  
**Software:** Proxmox VE 9.0.11, TrueNAS VM 110