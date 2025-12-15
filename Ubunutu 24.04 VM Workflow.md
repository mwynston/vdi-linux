Ubuntu 24.04 Autoinstall VM Workflow (libvirt + cloud-init)

1. Prepare Storage Volume

# Create qcow2 disk in vdi-pool
sudo virsh vol-create-as vdi-pool dev-vdi-01.qcow2 100G --format qcow2

# Verify volume exists
sudo virsh vol-list vdi-pool

# Get full path (optional)
sudo virsh vol-path dev-vdi-01.qcow2 --pool vdi-pool

2. Directory Layout

All files live in /var/lib/libvirt/dev-vdi-01/:

ubuntu-24.04.1-live-server-amd64.iso → Ubuntu install ISO

user-data → cloud-init config (autoinstall, SSH key, packages, firewall, etc.)

meta-data → VM metadata (hostname, instance ID)

seed.iso → generated from user-data + meta-data

3. Build Seed ISO

Install the tool:

sudo apt update
sudo apt install cloud-image-utils -y

Generate seed ISO:

sudo cloud-localds /var/lib/libvirt/dev-vdi-01/seed.iso \
    /var/lib/libvirt/dev-vdi-01/user-data \
    /var/lib/libvirt/dev-vdi-01/meta-data

4. virt-install Command

sudo virt-install \
  --name dev-vdi-01 \
  --memory 16384 \
  --vcpus 4 \
  --disk vol=vdi-pool/dev-vdi-01.qcow2 \
  --cdrom /var/lib/libvirt/dev-vdi-01/ubuntu-24.04.1-live-server-amd64.iso \
  --disk path=/var/lib/libvirt/dev-vdi-01/seed.iso,device=cdrom \
  --os-variant ubuntu24.04 \
  --network bridge=br0 \
  --graphics none \
  --console pty,target_type=serial

5. Console Access

Attach to VM console:

sudo virsh console dev-vdi-01

Detach with Ctrl + ].

6. Monitoring Install

# Check VM status
sudo virsh list --all

# Watch disk activity
sudo virsh domblkstat dev-vdi-01

# Watch CPU usage
sudo virsh dominfo dev-vdi-01

# Check qcow2 file growth
ls -lh /var/lib/libvirt/images/dev-vdi-01.qcow2

7. Post-Install Verification

Inside VM:

# Confirm static IP
ip addr show ens3

# Confirm firewall
sudo ufw status

# Confirm xrdp service
systemctl status xrdp

From host/iPad:

# SSH
ssh -i ~/.ssh/id_ed25519 mwynston@192.168.1.7

# RDP (Microsoft Remote Desktop app on iPad)
IP: 192.168.1.7
User: mwynston
Password: (set via cloud-init or manually with `sudo passwd mwynston`)

8. Password Setup for RDP

Generate hash:

openssl passwd -6

Update user-data:

users:
  - name: mwynston
    groups: sudo
    shell: /bin/bash
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    ssh-authorized-keys:
      - ssh-ed25519 AAAAC3Nza... dev-vdi-01
    passwd: "$6$rounds=4096$abcdefghijklmnop$Q9X...restOfHash..."
    lock_passwd: false

Rebuild seed ISO after editing:

sudo cloud-localds /var/lib/libvirt/dev-vdi-01/seed.iso \
    /var/lib/libvirt/dev-vdi-01/user-data \
    /var/lib/libvirt/dev-vdi-01/meta-data

9. Lifecycle Commands

# Start VM
sudo virsh start dev-vdi-01

# Shutdown VM
sudo virsh shutdown dev-vdi-01

# Destroy (force stop)
sudo virsh destroy dev-vdi-01

# Delete volume (cleanup)
sudo virsh vol-delete dev-vdi-01.qcow2 --pool vdi-pool

✅ Summary

This workflow gives you a repeatable autoinstall pattern:

Clean directory per VM

qcow2 disk in storage pool

ISO + seed ISO

Single virt-install command

SSH key + password for RDP baked in