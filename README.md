# macOS on Linux with QEMU/KVM

This guide provides instructions for installing **macOS in a virtual machine on Linux** using **QEMU/KVM** and the **macOS-Simple-KVM** project.

> **⚠️ Important Disclaimer**
> 
> Apple's End User License Agreement (EULA) for macOS restricts its use to Apple-branded hardware. This guide is provided **for educational, testing, and development purposes only**. Always ensure compliance with applicable laws and software licenses.

## 1. Requirements

### Hardware Requirements
- **CPU**: 64-bit Intel (VT-x) or AMD (SVM) processor with virtualization support
- **RAM**: Minimum 8 GB (16 GB recommended for smoother performance)
- **Storage**: SSD or NVMe drive with at least 20 GB free space (64+ GB recommended for the VM disk)
- **Motherboard**: Virtualization enabled in BIOS/UEFI settings

### Software Requirements
- **Linux Distribution**: Ubuntu 20.04+, Debian 11+, Fedora 36+, Arch Linux, or derivatives
- **QEMU**: Version 4.2 or newer
- **libvirt** and **virt-manager** for management
- **OVMF**: UEFI firmware for virtual machines
- **Git** for cloning the repository

### Verify Virtualization Support
Check if your CPU supports hardware virtualization:
```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
```
A result greater than `0` confirms support.

Enable virtualization in your BIOS/UEFI if not already enabled.

---

## 2. Install Dependencies

### Debian / Ubuntu / Pop!_OS
```bash
sudo apt update
sudo apt install -y \
  qemu-system \
  qemu-kvm \
  libvirt-daemon-system \
  libvirt-clients \
  virt-manager \
  bridge-utils \
  ovmf \
  git \
  wget
```

### Fedora / RHEL / CentOS Stream
```bash
sudo dnf install -y \
  qemu-kvm \
  libvirt \
  virt-manager \
  virt-install \
  libvirt-devel \
  virt-top \
  libguestfs-tools \
  guestfs-tools \
  bridge-utils \
  edk2-ovmf \
  git \
  wget
```

### Arch Linux / Manjaro
```bash
sudo pacman -S --needed \
  qemu-full \
  libvirt \
  virt-manager \
  dnsmasq \
  dmidecode \
  ebtables \
  edk2-ovmf \
  git \
  wget
```

### Enable and Start Services
```bash
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt $(whoami)
sudo usermod -aG kvm $(whoami)
```

**Important**: Log out and log back in (or restart your system) for group membership changes to take effect.

---

## 3. Clone the macOS-Simple-KVM Repository
```bash
cd ~
git clone --depth 1 --recursive https://github.com/foxlet/macOS-Simple-KVM
cd macOS-Simple-KVM
```

---

## 4. Download macOS Installer

The project supports several macOS versions. For best compatibility on both Intel and AMD systems, **macOS Catalina (10.15)** is recommended.

### Available Options:
```bash
# macOS Catalina (Recommended for compatibility)
./jumpstart.sh --catalina

# macOS Big Sur (11.x)
./jumpstart.sh --big-sur

# macOS Monterey (12.x)
./jumpstart.sh --monterey

# macOS Ventura (13.x)
./jumpstart.sh --ventura

# macOS Sonoma (14.x) - Experimental
./jumpstart.sh --sonoma
```

This script downloads the macOS installer directly from Apple's servers. No product key is required.

**Note**: Newer macOS versions may have additional requirements or compatibility issues. Catalina is the most tested and stable option for this setup.

---

## 5. Create the Virtual Machine

```bash
# Create basic VM configuration
sudo ./make.sh --add

# Alternative: Create VM with specific RAM/CPU settings
# sudo ./make.sh --add --ram 4096 --cpu 4
```

This will:
- Generate a basic VM configuration file
- Set up OpenCore as the bootloader
- Register the VM in **virt-manager** for easy management

The default configuration uses:
- 2 GB RAM (recommend increasing to 4-8 GB)
- 2 CPU cores
- QXL graphics adapter

---

## 6. Storage Configuration

The `make.sh` script creates a small boot disk. You need to add a primary storage disk for macOS itself.

### Option A: Using virt-manager (Recommended for Beginners)
1. Open **virt-manager**: `sudo virt-manager`
2. Double-click your macOS VM
3. Click the **"Add Hardware"** button
4. Select **"Storage"**
5. Choose:
   - **Device type**: Disk device
   - **Select or create custom storage**: Create new disk
   - **Format**: qcow2 (recommended for better space management)
   - **Size**: Minimum 64 GB (128 GB recommended for comfort)

### Option B: Command Line
```bash
cd ~/macOS-Simple-KVM
qemu-img create -f qcow2 macOS.qcow2 64G

# Edit the basic.sh or launch.sh to include the disk
# Add: -drive id=SystemDisk,if=none,file=macOS.qcow2 \
#      -device ide-hd,bus=sata.4,drive=SystemDisk
```

### Disk Location Tips
- Place the disk on your **fastest storage** (SSD/NVMe)
- qcow2 format allows for **thin provisioning** (disk grows as needed)
- For better performance, consider using **raw format** if you have ample disk space

---

## 7. Install macOS

### Step 1: Start the Installation
1. In **virt-manager**, select your macOS VM and click **"Begin Installation"**
2. The VM will boot to the **OpenCore boot menu**
3. Select **"macOS Installer"** and press Enter

### Step 2: Disk Utility
1. When the macOS Utilities screen appears, select **"Disk Utility"**
2. Select your virtual disk (should be listed as "ATA" or "VirtIO" disk)
3. Click **"Erase"**
4. Configure:
   - **Name**: "Macintosh HD" (or your preferred name)
   - **Format**: **APFS** (default and recommended)
   - **Scheme**: GUID Partition Map
5. Click **"Erase"**, then close Disk Utility

### Step 3: Install macOS
1. Back in the Utilities menu, select **"Install macOS"**
2. Follow the on-screen instructions
3. When prompted to select a disk, choose the one you just formatted
4. The installation will begin

### Installation Notes:
- **First boot is slow**: The initial installation may take 30-60 minutes
- **Multiple reboots**: The system will reboot several times during installation
- **OpenCore selection**: After each reboot, you may need to select the appropriate boot option in OpenCore
- **Post-install setup**: After the final reboot, complete the macOS first-time setup (Apple ID, preferences, etc.)

---

## 8. Post-Installation Setup

### Install VirtIO Drivers (Recommended)
For better performance with virtualized hardware:

1. **Shut down** the macOS VM completely
2. In **virt-manager**, select the VM and click **"Open"**
3. Click **"Add Hardware"** → **"Storage"**
4. Configure:
   - **Device type**: CDROM device
   - **Select or create custom storage**: Browse to `~/macOS-Simple-KVM/`
   - Select **`virtio-*.iso`** (e.g., `virtio-0.1.229.iso`)
5. Start the VM
6. In macOS, the VirtIO driver disk will appear on the desktop
7. Install the drivers (especially important for network and storage)

### OpenCore Configuration
- By default, the VM boots to the **OpenCore boot menu**
- To automatically boot macOS, you can modify the OpenCore configuration:
  ```bash
  # Mount and edit the OpenCore image (advanced users)
  sudo modprobe nbd max_part=8
  sudo qemu-nbd -c /dev/nbd0 OpenCore-Catalina/opencore.qcow2
  sudo mount /dev/nbd0p1 /mnt
  # Edit /mnt/EFI/OC/config.plist
  sudo umount /mnt
  sudo qemu-nbd -d /dev/nbd0
  ```

### Performance Optimization
In **virt-manager**, under VM hardware settings:

1. **CPU**:
   - Set topology: 2 Sockets, 2 Cores (for 4 vCPUs)
   - Copy host CPU configuration
   - Add CPU features: `+vmx, +hypervisor, +invtsc`

2. **Memory**:
   - Increase to at least 4 GB (8 GB recommended)
   - Enable "Enable shared memory"

3. **Video**:
   - Change from QXL to **virtio** (requires VirtIO drivers)
   - Allocate more video memory (e.g., 128 MB)

4. **Sound**:
   - Change from AC97 to **ich9** for better compatibility

---

## 9. Networking

### Default Configuration
- **NAT networking** works immediately after installation
- Internet access is available within the VM
- The VM gets an IP in a private range (e.g., 192.168.122.x)

### Bridged Networking (Advanced)
For giving the VM its own IP on your local network:

1. Create a network bridge on your host:
   ```bash
   # Ubuntu/Debian - Edit /etc/netplan/01-netcfg.yaml
   # Add bridge configuration
   ```

2. In **virt-manager**:
   - Go to VM details → **"NIC"**
   - Change "Device model" to **"virtio"** (after installing VirtIO drivers)
   - Change "Network source" to **"Bridge device"**
   - Select your bridge (e.g., `br0`)

**Note**: Bridging with Wi-Fi interfaces is complex and may require additional setup.

---

## 10. GPU Considerations

### Current Limitations
- **Modern NVIDIA GPUs** (RTX series) are **not supported** by macOS without proper drivers
- **AMD GPUs** have better support but still require specific models
- **Intel integrated graphics** (UHD 630, Iris Plus) work best with passthrough

### Graphics Options

#### Option A: Emulated Graphics (Default)
- Uses QEMU's **QXL** or **VirtIO-GPU**
- Software rendering only
- Suitable for basic usage, development, and testing
- Lower performance for graphical applications

#### Option B: GPU Passthrough (Advanced)
Requires:
- Secondary GPU or integrated graphics
- IOMMU support enabled in BIOS
- Proper driver configuration on host

```bash
# Check IOMMU groups
sudo dmesg | grep -i iommu
sudo find /sys/kernel/iommu_groups/ -type l
```

#### Option C: Looking Glass (Recommended for Single GPU)
- Low-latency streaming from passed-through GPU to host
- Requires secondary GPU or integrated graphics for host
- [Looking Glass Project](https://looking-glass.io)

---

## 11. Troubleshooting

### Common Issues and Solutions

#### 1. Installation Stuck or Very Slow
- **Solution**: Be patient - first boot can take 30+ minutes
- Ensure you're using an SSD for the VM disk
- Increase RAM to at least 4 GB
- Try macOS Catalina instead of newer versions

#### 2. "This copy of macOS Installer is damaged"
- **Solution**: The downloaded image might be corrupted
- Delete the downloaded BaseSystem files and run `jumpstart.sh` again
- Check your internet connection during download

#### 3. No Boot Device Found
- **Solution**: Ensure you added a storage disk in Section 6
- Check that the disk is properly formatted as APFS in Disk Utility
- Verify the disk is connected to a SATA or VirtIO controller

#### 4. Poor Graphics Performance
- **Solution**: Install VirtIO drivers and switch to virtio-gpu
- Allocate more video memory (128-256 MB)
- Consider GPU passthrough if you have compatible hardware

#### 5. No Network Connection
- **Solution**: Install VirtIO network drivers
- Check that the network adapter is set to "virtio" in VM settings
- Verify host networking is functioning

#### 6. Audio Not Working
- **Solution**: Change sound card from AC97 to **ich9**
- Install any available audio drivers from the VirtIO disk

#### 7. VM Won't Start (Permission Errors)
- **Solution**: Ensure you're in the correct groups
  ```bash
  groups | grep -E '(libvirt|kvm)'
  # If not showing, log out and log back in
  ```

### Logs and Debugging
```bash
# Check QEMU logs
sudo journalctl -xe | grep qemu

# Check libvirt logs
sudo journalctl -u libvirtd

# Run in debug mode (from macOS-Simple-KVM directory)
./basic.sh -d
```

---

## 12. References

### Primary Resources
- **macOS-Simple-KVM GitHub**: [https://github.com/foxlet/macOS-Simple-KVM](https://github.com/foxlet/macOS-Simple-KVM)
- **OpenCore Install Guide**: [https://dortania.github.io/OpenCore-Install-Guide](https://dortania.github.io/OpenCore-Install-Guide)

### Supplementary Guides
- **Chris Titus Tech**: [https://christitus.com/macos-virtual-machine](https://christitus.com/macos-virtual-machine)
- **KVM GPU Passthrough**: [https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF)

### Community Support
- **r/VFIO on Reddit**: GPU passthrough community
- **r/hackintosh**: macOS on non-Apple hardware discussions
- **OSX-KVM GitHub Discussions**: Problem-solving and tips

### Legal Note
This guide is for **educational purposes only**. Always respect software licenses and use only legally obtained software. Consider supporting Apple by purchasing their hardware if you find macOS valuable for your work.

---

---

**Good luck with your macOS virtual machine!** Remember that virtualization is about flexibility and learning, not about circumventing legitimate licensing requirements. Use this knowledge responsibly and consider supporting developers and companies whose software you find valuable.

*Last updated: November 2023*
