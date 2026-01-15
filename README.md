# Installing macOS on Linux using QEMU/KVM

This document describes how to install **macOS in a virtual machine on Linux** using **QEMU/KVM** and the **macOS-Simple-KVM** project.

> **Disclaimer**
> macOS is licensed by Apple for use on Apple hardware only.
> This guide is provided for **educational and testing purposes**.

---

## 1. Requirements

### Hardware

* 64-bit CPU with virtualization support

  * Intel: VT-x
  * AMD: SVM
* Minimum 8 GB RAM (16 GB recommended)
* SSD or NVMe storage

### Software

* Linux distribution (Ubuntu, Pop!_OS, Debian, Arch, Fedora)
* QEMU ≥ 4.2
* libvirt
* virt-manager
* OVMF (UEFI firmware)

### Verify virtualization support

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
```

A result greater than 0 confirms support.

---

## 2. Install Dependencies (Debian / Ubuntu / Pop!_OS)

```bash
sudo apt update
sudo apt install -y \
  qemu-kvm \
  libvirt-daemon-system \
  libvirt-clients \
  virt-manager \
  ovmf \
  git
```

Enable and start libvirt:

```bash
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt $(whoami)
```

Log out and log back in to apply group changes.

---

## 3. Clone the macOS-Simple-KVM Repository

```bash
cd ~
git clone --depth 1 --recursive https://github.com/foxlet/macOS-Simple-KVM
cd macOS-Simple-KVM
```

---

## 4. Download macOS Installer

For best compatibility on both Intel and AMD systems, **macOS Catalina** is recommended.

```bash
./jumpstart.sh --catalina
```

This script downloads the installer directly from Apple servers.
No product key is required.

---

## 5. Create the Virtual Machine

```bash
sudo ./make.sh --add
```

This will:

* Create a macOS virtual machine
* Configure OpenCore
* Register the VM in **virt-manager**

---

## 6. Storage Configuration

### Option A: Virtual Disk (Recommended)

1. Open **virt-manager**
2. Select the macOS VM → *Add Hardware*
3. Choose **Storage**
4. Create a new QCOW2 disk on SSD/NVMe

---

## 7. Install macOS

1. Start the virtual machine
2. Select the macOS installer
3. Open **Disk Utility**
4. Format the virtual disk as **APFS**
5. Proceed with the installation

The system may reboot multiple times during installation.

---

## 8. Networking

* Default NAT networking works without additional configuration
* Bridged networking is optional and may require manual setup
* Wi-Fi bridge support varies by distribution and driver

---

## 9. GPU Considerations

* Modern NVIDIA GPUs (RTX series) are **not supported** by macOS
* The VM will use:

  * QEMU emulated graphics, or
  * Integrated GPU passthrough (advanced setup)

This setup is intended for development, testing, and general macOS usage, not for GPU-intensive workloads.

---

## 10. Known Limitations

* No hardware acceleration for modern NVIDIA GPUs
* Lower graphical performance compared to native macOS
* Not suitable as a full replacement for Apple hardware

---

## 11. References

* macOS-Simple-KVM
  [https://github.com/foxlet/macOS-Simple-KVM](https://github.com/foxlet/macOS-Simple-KVM)
* Chris Titus Tech
  [https://christitus.com](https://christitus.com)

---
