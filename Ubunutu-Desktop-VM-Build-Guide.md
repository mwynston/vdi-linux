# Ubuntu Desktop VM Build Guide

This guide documents the full step-by-step process for building an Ubuntu 24.04.3 Desktop VM with libvirt/KVM, using a cloud-init YAML file. Each command includes an explanation of why it is used, so you can understand and repeat the process confidently.

---

## Table of Contents
- [Prerequisites](#prerequisites)
- [Step 1: Create the VM Disk in vdi-pool](#step-1-create-the-vm-disk-in-vdi-pool)
- [Step 2: Launch the VM with virt-install](#step-2-launch-the-vm-with-virt-install)
- [Step 3: First Boot & Verification](#step-3-first-boot--verification)
- [Step 4: Networking Setup](#step-4-networking-setup)
- [Step 5: Connect to the VM](#step-5-connect-to-the-vm)
- [Step 6: Post-Install Hardening](#step-6-post-install-hardening)
- [Optional: Static IP Configuration](#optional-static-ip-configuration)
- [Summary](#summary)

---

## Prerequisites
- Host server with libvirt/KVM installed and running.
- ISO image: `/var/lib/libvirt/images/ubuntu-24.04.3-desktop-amd64.iso`
- Cloud-init YAML file: `/var/lib/libvirt/images/vdi-01-cloudinit.yaml`
- Active storage pool: `vdi-pool`

---

## Step 1: Create the VM Disk in vdi-pool
We need a virtual disk for the VM. Using `virsh vol-create-as` ensures the disk is tracked by libvirt inside the `vdi-pool`.

```bash
sudo virsh vol-create-as vdi-pool dev-vdi-01.qcow2 100G --format qcow2
```
**Why**: Creates a 100 GB qcow2 disk image managed by libvirt in the `vdi-pool`.

Verify the volume exists:
```bash
sudo virsh vol-list vdi-pool
```
**Why**: Lists all volumes in the pool to confirm creation.

Get the full path:
```bash
sudo virsh vol-path dev-vdi-01.qcow2 --pool vdi-pool
```
**Why**: Shows the actual filesystem path of the disk image.

---

## Step 2: Launch the VM with virt-install
We use `virt-install` to define and start the VM with the desired specs.

```bash
sudo virt-install \
  --name dev-vdi-01 \
  --memory 16384 \
  --vcpus 4 \
  --disk vol=vdi-pool/dev-vdi-01.qcow2 \
  --cdrom /var/lib/libvirt/images/ubuntu-24.04.3-desktop-amd64.iso \
  --os-variant ubuntu24.04 \
  --network bridge=br0 \
  --graphics spice \
  --console pty,target_type=serial \
  --cloud-init user-data=/var/lib/libvirt/images/vdi-01-cloudinit.yaml
```

**Why each flag**:
- `--name`: Names the VM `dev-vdi-01`.
- `--memory`: Allocates 16 GB RAM.
- `--vcpus`: Assigns 4 virtual CPUs.
- `--disk`: Attaches the 100 GB qcow2 disk from `vdi-pool`.
- `--cdrom`: Boots from the Ubuntu Desktop ISO.
- `--os-variant`: Optimizes settings for Ubuntu 24.04.
- `--network bridge=br0`: Connects VM to LAN via bridge `br0`.
- `--graphics spice`: Enables graphical console access.
- `--console pty,target_type=serial`: Allows `virsh console` access.
- `--cloud-init`: Injects your YAML file to configure packages, user, and RDP setup.

---

## Step 3: First Boot & Verification
After installation, check VM status:
```bash
sudo virsh list --all
```
**Why**: Confirms the VM is running.

Access console:
```bash
sudo virsh console dev-vdi-01
```
**Why**: Provides serial console access for troubleshooting.

---

## Step 4: Networking Setup
Since the VM uses `br0`, it should get an IP from your LAN DHCP.

Find the VM’s MAC address:
```bash
sudo virsh domiflist dev-vdi-01
```
**Why**: Shows the VM’s network interfaces and MAC.

Check IP inside the VM:
```bash
ip addr show
```
**Why**: Displays assigned IP addresses.

---

## Step 5: Connect to the VM
- **RDP from iPad**: Use Microsoft Remote Desktop app → connect to `<VM-IP>:3389`, login as `mwynston`.
- **SSH**: Connect with your certificate/private key:
```bash
ssh mwynston@<VM-IP>
```
**Why**: Provides secure shell access for development and recovery.

---

## Step 6: Post-Install Hardening
Secure the VM after initial setup.

Allow SSH and RDP through firewall:
```bash
ufw allow 22
ufw allow 3389
ufw enable
```
**Why**: Opens required ports and enables firewall.

Verify xrdp service:
```bash
systemctl status xrdp
```
**Why**: Confirms RDP service is running.

---

## Optional: Static IP Configuration
To avoid relying on DHCP, configure Netplan inside the VM:

```yaml
network:
  version: 2
  ethernets:
    ens3:
      dhcp4: no
      addresses:
        - 192.168.1.50/24
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

Apply changes:
```bash
sudo netplan apply
```
**Why**: Ensures the VM always has a predictable IP address.

---

## Summary
This process builds a developer-ready Ubuntu Desktop VM with:
- RDP access from iPad Pro
- SSH access with certificate authentication
- Developer tools (VS Code, Python, Docker, etc.)
- Secure configuration with firewall and optional MFA

Future enhancements can include Google Authenticator MFA and static IP setup for predictable connectivity.
