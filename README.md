# macOS on Linux with QEMU/KVM

This guide provides instructions for installing **macOS in a virtual machine on Linux** using **QEMU/KVM** and the **macOS-Simple-KVM** project.

> **⚠️ Important Disclaimer**
>
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
./jumpstart.sh --catalina
./jumpstart.sh --big-sur
./jumpstart.sh --monterey
./jumpstart.sh --ventura
./jumpstart.sh --sonoma
```

This script downloads the macOS installer directly from Apple's servers. No product key is required.

**Note**: Newer macOS versions may have additional requirements or compatibility issues. Catalina is the most tested option for this setup.

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

* 2 GB RAM
* 2 CPU cores
* QXL graphics adapter

![Virtual Machine Creation](images/Screenshot_2026-01-15_21-07-58.png)

---

## 6. Storage Configuration

The `make.sh` script creates a small boot disk. You need to add a primary storage disk for macOS itself.

### Option A: Using virt-manager

1. Open **virt-manager**
2. Double-click your macOS VM
3. Click **Add Hardware**
4. Select **Storage**
5. Create a new disk:

   * **Format**: qcow2
   * **Size**: Minimum 64 GB
   * **Bus**: SATA (AHCI)

![Storage Configuration](images/Screenshot_2026-01-15_21-08-39.png)

### Option B: Command Line

```bash
qemu-img create -f qcow2 macOS.qcow2 64G
```

macOS disks must use **SATA (AHCI)**.

---

## 7. Install macOS

1. Start the VM
2. OpenCore boot menu → **macOS Installer**
3. Open **Disk Utility**
4. Erase disk:

   * **Format**: APFS
   * **Scheme**: GUID Partition Map
5. Install macOS

---

## 8. Post-Installation Setup

### Correct Device Models for macOS

macOS **does not support VirtIO storage or network drivers**.

Use the following device models:

* **Storage**: SATA (AHCI)
* **Network**: `vmxnet3`
* **Graphics**: `virtio-gpu`

Do **not** install VirtIO driver ISO files inside macOS.

---

### CPU Configuration

In **virt-manager**:

* Use **Copy host CPU configuration**
* Do not manually add CPU flags unless required

Manual flags like `+vmx` are Intel-only and may break AMD systems.

---

### Memory

* Increase to at least 4 GB
* 8 GB recommended
* Disable memory ballooning

---

### Video

* Change from QXL to **virtio**
* Allocate 128 MB video memory

---

### Sound

* Change from AC97 to **ich9**

---

## 9. Networking

### Default Configuration

* NAT networking works immediately
* Use **vmxnet3** as the network device model

### Bridged Networking

* Requires a host bridge
* Wi-Fi bridging may require additional setup

---

## 10. GPU Considerations

### Emulated Graphics

* virtio-gpu
* Software rendering
* Suitable for development and testing

### GPU Passthrough

* Requires secondary GPU or iGPU
* IOMMU enabled
* Compatible AMD GPUs only

---

## 11. Troubleshooting

### VM Won't Start

```bash
groups | grep -E '(libvirt|kvm)'
```

Log out and back in if groups are missing.

### Slow Installation

* Be patient
* Use SSD storage
* Increase RAM

---

## 12. References

* [https://github.com/foxlet/macOS-Simple-KVM](https://github.com/foxlet/macOS-Simple-KVM)
* [https://dortania.github.io/OpenCore-Install-Guide](https://dortania.github.io/OpenCore-Install-Guide)
* [https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF)

---

