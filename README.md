# macOS on Linux with QEMU/KVM

This guide provides instructions for installing **macOS in a virtual machine on Linux** using **QEMU/KVM** and the **macOS-Simple-KVM** project.

> **⚠️ Important Disclaimer**
> Apple's End User License Agreement (EULA) for macOS restricts its use to Apple-branded hardware. This guide is provided **for educational, testing, and development purposes only**. Always ensure compliance with applicable laws and software licenses.

---

## 1. Requirements

### Hardware Requirements

* **CPU**: 64-bit Intel (VT-x) or AMD (SVM) processor with virtualization support
* **RAM**: Minimum 8 GB (16 GB recommended for smoother performance)
* **Storage**: SSD or NVMe drive with at least 20 GB free space (64+ GB recommended for the VM disk)
* **Motherboard**: Virtualization enabled in BIOS/UEFI settings

### Software Requirements

* **Linux Distribution**: Ubuntu 20.04+, Debian 11+, Fedora 36+, Arch Linux, or derivatives
* **QEMU**: Version 4.2 or newer
* **libvirt** and **virt-manager** for management
* **OVMF**: UEFI firmware for virtual machines
* **Git** for cloning the repository

### Verify Virtualization Support

Check if your CPU supports hardware virtualization:

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
```

A result greater than `0` confirms support.

Enable virtualization in your BIOS/UEFI if not already enabled.

![Virtualization Support](images/Screenshot_2026-01-15_21-06-13.png)

---

## 2. Install Dependencies

### Debian / Ubuntu / Pop!_OS

```bash
sudo apt update
sudo apt install -y \
  qemu-system-x86 \
  qemu-kvm \
  qemu-utils \
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
./jumpstart.sh --catalina
./jumpstart.sh --big-sur
./jumpstart.sh --monterey
./jumpstart.sh --ventura
./jumpstart.sh --sonoma
```

This script downloads the macOS installer directly from Apple's servers. No product key is required.

**Note**: Newer macOS versions (Ventura/Sonoma) may have additional requirements or compatibility issues. Catalina is the most tested option for this setup.

---

## 5. Create the Virtual Machine

```bash
sudo ./make.sh --add
```

This will:

* Generate a basic VM configuration file
* Set up OpenCore as the bootloader
* Register the VM in **virt-manager** for easy management

The default configuration uses:

* 2 GB RAM (Note: You should manually increase this to 4GB+ for a stable install)
* 2 CPU cores
* QXL graphics adapter

![Virtual Machine Creation](images/Screenshot_2026-01-15_21-07-58.png)

---

## 6. Storage Configuration

The `make.sh` script creates a small boot disk. You need to add a primary storage disk for macOS itself.

### Option A: Using virt-manager

1. Open **virt-manager**
2. Double-click your macOS VM
3. Click **Add Hardware** (the lightbulb icon)
4. Select **Storage**
5. Create a new disk:

* **Format**: qcow2
* **Size**: Minimum 64 GB
* **Bus**: SATA

![Storage Configuration](images/Screenshot_2026-01-15_21-08-39.png)

### Option B: Command Line

```bash
qemu-img create -f qcow2 macOS.qcow2 64G
```

macOS disks must use **SATA (AHCI)**. VirtIO storage is not natively supported during the installation.

---

## 7. Install macOS

1. Start the VM in virt-manager.
2. At the OpenCore boot menu → Select **macOS Base System**.
3. Once the utilities load, open **Disk Utility**.
4. **Crucial:** Click "View" -> "**Show All Devices**" in the top left.
5. Select your virtual disk (e.g., "QEMU HARDDISK") and click **Erase**:

* **Name**: Macintosh HD
* **Format**: APFS
* **Scheme**: GUID Partition Map

6. Close Disk Utility and select **Install macOS**.

---

## 8. Post-Installation Setup

### Correct Device Models for macOS

macOS **does not support VirtIO storage or network drivers natively**.

Use the following device models in your VM settings:

* **Storage**: SATA (AHCI)
* **Network**: `vmxnet3`
* **Graphics**: `virtio`
* **USB Controller**: USB 3.0 (xHCI)

---

### CPU Configuration

In **virt-manager**:

* Use **Copy host CPU configuration**.
* **Topology**: Recommended 1 Socket, 2 Cores, 2 Threads (4 vCPUs total).

---

### Memory

* Increase to at least **4 GB** (8 GB recommended).
* Disable memory ballooning as it can cause macOS kernels to panic.

---

### Video

* Change from QXL to **virtio**.
* Allocate 128 MB video memory.

---

### Sound

* Change from AC97 to **ich9**.

---

## 9. Networking

### Default Configuration

* NAT networking works immediately with the **vmxnet3** model.
* If internet doesn't work, ensure your host has the `dnsmasq` package installed.

---

## 10. GPU Considerations

### Emulated Graphics

* Without a dedicated GPU, macOS uses software rendering.
* This is suitable for development/testing but will have UI lag.

### GPU Passthrough

* Requires a secondary GPU.
* **AMD GPUs** (RX 480/580/5000/6000) have the best native support.
* Modern NVIDIA GPUs are not supported.

---

## 11. Troubleshooting

### "This copy of macOS Installer is damaged"

This is usually a date/time error. In the macOS terminal (inside the VM), run:

```bash
date 010101012024
```

(This sets the date to Jan 1, 2024).

### VM Won't Start (Permissions)

```bash
groups | grep -E '(libvirt|kvm)'
```

If your user isn't in these groups, run `sudo usermod -aG libvirt,kvm $(whoami)` and reboot.

---

## 12. References

* [https://github.com/foxlet/macOS-Simple-KVM](https://github.com/foxlet/macOS-Simple-KVM)
* [https://dortania.github.io/OpenCore-Install-Guide](https://dortania.github.io/OpenCore-Install-Guide)
* [https://github.com/kholia/OSX-KVM](https://github.com/kholia/OSX-KVM) (Alternative for newer macOS versions)


