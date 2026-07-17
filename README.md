# Spin Up a Linux VM with KVM/QEMU/LIBVIRT

A lightweight, minimal guide to launching an Ubuntu VM using cloud images and `cloud-init` via CLI.

---

### 1. Install Required Packages
Install virtualization tools, QEMU/KVM components, and cloud-init utilities.
```bash
sudo apt update && sudo apt install -y \
    qemu-system-x86 \
    qemu-utils \
    libvirt-daemon-system \
    libvirt-clients \
    virtinst \
    cloud-image-utils \
    ovmf \
    dnsmasq-base \
    netcat-openbsd
```

### 2. Configure User Permissions
Grant your user permission to manage KVM/libvirt without `sudo`.
```bash
sudo usermod -aG libvirt,kvm $USER
```
> **Note:** Log out and log back in to apply group changes.

### 3. Setup Cloud-Init Configuration
Create a working directory and define initialization files for automatic OS configuration.
```bash
mkdir -p ~/vm && cd ~/vm
```

`user-data` (sets default user, password, and starts guest agent):
```yaml
#cloud-config
hostname: vm
users:
  - name: ghost
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    lock_passwd: false
chpasswd:
  list: |
    ghost:password
  expire: false
ssh_pwauth: true
packages:
  - qemu-guest-agent
runcmd:
  - systemctl enable --now qemu-guest-agent
```

`meta-data` (sets instance identity):
```yaml
instance-id: vm
local-hostname: vm
```

### 4. Configure Storage Pool *(Run Once)*
Initialize and start the default local directory pool for VM images.
```bash
sudo virsh pool-define-as default dir --target /var/lib/libvirt/images
sudo virsh pool-build default
sudo virsh pool-start default
sudo virsh pool-autostart default

# Verify pool status
virsh pool-list --all
```

### 5. Generate Cloud-Init Metadata ISO
Bake `user-data` and `meta-data` into a seed ISO that the VM will read on boot.
```bash
sudo cloud-localds /var/lib/libvirt/images/seed.iso user-data meta-data
```

### 6. Download & Prepare OS Image
Fetch the official Ubuntu cloud image, copy it to storage, and expand its capacity.
```bash
# Download cloud image
wget https://cloud-images.ubuntu.com/noble/20260307/noble-server-cloudimg-amd64.img

# Copy to storage pool
sudo cp noble-server-cloudimg-amd64.img /var/lib/libvirt/images/test.qcow2

# Resize disk (+20GB)
sudo qemu-img resize /var/lib/libvirt/images/test.qcow2 +20G
```

### 7. Provision and Launch VM
Deploy the VM in headless mode using the prepared disk and seed ISO.
```bash
sudo virt-install \
  --name vm \
  --virt-type kvm \
  --cpu host-model \
  --memory 8192 \
  --vcpus 4 \
  --import \
  --osinfo ubuntu24.04 \
  --disk /var/lib/libvirt/images/test.qcow2,format=qcow2,bus=virtio \
  --disk /var/lib/libvirt/images/seed.iso,device=cdrom,readonly=on \
  --network network=default,model=virtio \
  --graphics none \
  --noautoconsole
```

### 8. Fetch IP & Connect
Wait ~1 minute for cloud-init to finish provisioning, grab the IP, and SSH inside.
```bash
# Get VM IP address
virsh domifaddr vm

# SSH access (Credentials: ubuntu / ubuntu)
ssh -o PubkeyAuthentication=no ghost@192.168.122.25
```

### 9. Troubleshooting
Inspect cloud-init logs inside the VM to debug provisioning issues.
```bash
sudo tail -f /var/log/cloud-init-output.log
```

---

## Essential `virsh` Cheat Sheet

| Command | Action |
| :--- | :--- |
| `virsh list --all` | List all available VMs |
| `virsh start vm` | Start the VM |
| `virsh shutdown vm` | Gracefully shut down the VM |
| `virsh destroy vm` | Force stop the VM (Emergency power off) |
| `virsh reboot vm` | Reboot the VM |
| `virsh suspend vm` | Pause the VM |
| `virsh resume vm` | Resume the VM |
| `virsh dominfo vm` | Show VM details |
| `virsh net-start default` | Start the default virtual network |
| `virsh net-dhcp-leases default` | View active DHCP leases |

---

## Teardown
Remove the virtual machine, delete all associated storage, and stop the default virtual network.

```bash
virsh destroy vm 
virsh undefine vm --remove-all-storage
sudo virsh net-destroy default
```
