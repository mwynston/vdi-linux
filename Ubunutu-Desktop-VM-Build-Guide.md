# Ubuntu Server VM Build Guide (with GUI + RDP, Static IP)

This guide documents the full step-by-step process for building an Ubuntu 24.04.3 Server VM with libvirt/KVM, using cloud-init/autoinstall to run unattended. It installs a lightweight desktop (XFCE) and RDP support so you can connect from your iPad or other clients. The VM is configured with a static IP: **192.168.1.7/24** and gateway **192.168.1.254**.

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
- SSH keypair generated (`id_ed25519` private key on your client, `id_ed25519.pub` public key injected into cloud-init).

---

## Step 1: Prepare Cloud-Init Files

### `user-data`
This file defines your user, packages, firewall, and static IP:

```yaml
#cloud-config
users:
  - name: mwynston
    groups: sudo
    shell: /bin/bash
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    ssh-authorized-keys:
      - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEXAMPLEKEYSTRINGHERE... dev-vdi-01

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

network:
  version: 2
  ethernets:
    ens3:
      dhcp4: no
      addresses:
        - 192.168.1.7/24
      gateway4: 192.168.1.254
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

### `meta-data`
Minimal metadata file:

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
Use `--location` and autoinstall arguments:

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

You should see autoinstall and cloud-init logs as packages are installed.

---

## Step 5: Networking Setup
The VM will come up with:
- IP: `192.168.1.7`
- Mask: `/24`
- Gateway: `192.168.1.254`
- DNS: `8.8.8.8`, `1.1.1.1`

Verify inside VM:
```bash
ip addr show ens3
```

---

## Step 6: Connect to the VM
- **SSH**:
  ```bash
  ssh -i ~/.ssh/id_ed25519 mwynston@192.168.1.7
  ```
- **RDP from iPad**:
  Use Microsoft Remote Desktop app → connect to `192.168.1.7:3389`, login as `mwynston`.

---

## Step 7: Post-Install Hardening
Verify firewall:
```bash
sudo ufw status
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
- Static IP: `192.168.1.7/24` with gateway `192.168.1.254`
- Developer tools (VS Code, Python, Docker, etc.)
- Secure configuration with firewall and optional MFA
