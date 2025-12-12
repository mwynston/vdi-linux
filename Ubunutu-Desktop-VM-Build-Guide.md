# Ubuntu Server VM Build Guide (with GUI + RDP)

This guide documents the full step-by-step process for building an Ubuntu 24.04.3 Server VM with libvirt/KVM, using cloud-init/autoinstall to run unattended. It installs a lightweight desktop (XFCE) and RDP support so you can connect from your iPad or other clients.

---

## Table of Contents
- [Prerequisites](#prerequisites)
- [Step 1: Prepare Cloud-Init Files](#step-1-prepare-cloud-init-files)
- [Step 2: Create the VM Disk in vdi-pool](#step-2-create-the-vm-disk-in-vdi-pool)
- [Step 3: Launch the VM with virt-install](#step-3-launch-the-vm-with-virt-install)
- [Step 4: First Boot & Verification](#step-4-first-boot--verification)
- [Step 5: Networking Setup](#step-5-networking-setup)
- [Step 6: Connect to the VM](#step-6-connect-to-the-vm)
- [Step 7: Post-Install Hardening](#step-7-post-install-hardening)
- [Summary](#summary)

---

## Prerequisites
- Host server with libvirt/KVM installed and running.
- ISO image: `/var/lib/libvirt/images/ubuntu-24.04.3-live-server-amd64.iso`
- Cloud-init files in `/var/lib/libvirt/images/`:
  - `user-data`
  - `meta-data`
- Active storage pool: `vdi-pool`

---

## Step 1: Prepare Cloud-Init Files

### `user-data`
This file defines your user, packages, and configuration:

```yaml
#cloud-config
users:
  - name: mwynston
    groups: sudo
    shell: /bin/bash
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    ssh-authorized-keys:
      - <your-public-ssh-key>

packages:
  - xfce4
  - xfce4-goodies
  - xrdp
  - snapd
  - code
  - python3
  - python3-pip
  - docker.io
  - git
  - build-essential

runcmd:
  - systemctl enable xrdp
  - ufw allow 22
  - ufw allow 3389
  - ufw enable
```

### `meta-data`
This file is minimal:

```yaml
instance-id: dev-vdi-01
local-hostname: dev-vdi-01
```

---

## Step 2: Create the VM Disk in vdi-pool
Allocate a 100 GB qcow2 disk volume:

```bash
sudo virsh vol-create-as vdi-pool dev-vdi-01.qcow2 100G --format qcow2
```

Verify the volume exists:
```bash
sudo virsh vol-list vdi-pool
sudo virsh vol-path dev-vdi-01.qcow2 --pool vdi-pool
```

---

## Step 3: Launch the VM with virt-install
Use `--location` and autoinstall arguments instead of `--cdrom`:

```bash
sudo virt-install \
  --name dev-vdi-01 \
  --memory 16384 \
  --vcpus 4 \
  --disk vol=vdi-pool/dev-vdi-01.qcow2 \
  --location /var/lib/libvirt/images/ubuntu-24.04.3-live-server-amd64.iso \
  --os-variant ubuntu24.04 \
  --network bridge=br0 \
  --graphics none \
  --console pty,target_type=serial \
  --extra-args 'autoinstall ds=nocloud-net;s=/var/lib/libvirt/images/'
```

**Why these changes**:
- `--location` triggers the server autoinstall workflow.
- `--extra-args` points to your cloud-init files (`user-data` and `meta-data`).
- `--graphics none` keeps it headless; install runs unattended.

---

## Step 4: First Boot & Verification
After installation, check VM status:
```bash
sudo virsh list --all
```

Access console:
```bash
sudo virsh console dev-vdi-01
```

You should see cloud-init logs as it installs packages and configures the system.

---

## Step 5: Networking Setup
Since the VM uses `br0`, it should get an IP from your LAN DHCP.

Find the VM’s MAC address:
```bash
sudo virsh domiflist dev-vdi-01
```

Check IP inside the VM:
```bash
ip addr show
```

---

## Step 6: Connect to the VM
- **RDP from iPad**: Use Microsoft Remote Desktop app → connect to `<VM-IP>:3389`, login as `mwynston`.
- **SSH**: Connect with your certificate/private key:
```bash
ssh mwynston@<VM-IP>
```

---

## Step 7: Post-Install Hardening
Secure the VM after initial setup.

Allow SSH and RDP through firewall (already in cloud-init, but verify):
```bash
ufw allow 22
ufw allow 3389
ufw enable
```

Verify xrdp service:
```bash
systemctl status xrdp
```

Optional: add MFA with Google Authenticator PAM module once SSH works.

---

## Summary
This process builds a developer-ready Ubuntu Server VM with:
- Automated unattended install via autoinstall + cloud-init
- XFCE desktop and xrdp for RDP access
- SSH access with certificate authentication
- Developer tools (VS Code, Python, Docker, etc.)
- Secure configuration with firewall and optional MFA

Future enhancements can include static IP setup in Netplan for predictable connectivity.
