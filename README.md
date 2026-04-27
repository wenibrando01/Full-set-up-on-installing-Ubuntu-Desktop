# Full-set-up-on-installing-Ubuntu-Desktop

Part 2: Install KVM on Ubuntu Desktop (via Terminal)
1. Open Terminal and update:
bashsudo apt update && sudo apt upgrade -y
2. Check virtualization support:
bashgrep -Ec '(vmx|svm)' /proc/cpuinfo
3. Install KVM and all tools:
bashsudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients \
bridge-utils virt-manager virtinst cpu-checker
4. Verify KVM is ready:
bashkvm-ok
Expected output: INFO: /dev/kvm exists and KVM acceleration can be used
5. Enable libvirt service:
bashsudo systemctl enable --now libvirtd
sudo systemctl status libvirtd
6. Add user to groups:
bashsudo usermod -aG libvirt $USER
sudo usermod -aG kvm $USER
newgrp libvirt

Part 3: Create a Virtual Machine via Terminal (virt-install)
1. Download Ubuntu Server ISO for the VM:
bashwget https://releases.ubuntu.com/24.04/ubuntu-24.04-live-server-amd64.iso \
-P ~/iso/
2. Create a disk image for the VM:
bashsudo qemu-img create -f qcow2 /var/lib/libvirt/images/myvm.qcow2 20G
3. Install the VM using virt-install:
bashsudo virt-install \
  --name myvm \
  --ram 2048 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/myvm.qcow2,format=qcow2 \
  --cdrom ~/iso/ubuntu-24.04-live-server-amd64.iso \
  --os-variant ubuntu22.04 \
  --network network=default \
  --graphics vnc,listen=0.0.0.0 \
  --noautoconsole
4. Check VM is running:
bashvirsh list --all
5. Connect to VM console:
bashvirsh console myvm

Press Ctrl+] to exit the console anytime.


Part 4: Set Network to Bridge Adapter
1. Install bridge utilities:
bashsudo apt install -y bridge-utils
2. Find your network interface:
baship link show
# Note your main interface e.g. enp3s0 or eth0
3. Create a bridge configuration using Netplan:
bashsudo nano /etc/netplan/01-netcfg.yaml
Paste this (replace enp3s0 with your interface):
yamlnetwork:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0:
      dhcp4: no

  bridges:
    br0:
      interfaces: [enp3s0]
      dhcp4: yes
      parameters:
        stp: false
        forward-delay: 0
4. Apply the network config:
bashsudo netplan apply
5. Verify bridge is active:
baship addr show br0
brctl show
6. Create a bridge network XML for libvirt:
bashnano ~/bridge.xml
Paste:
xml<network>
  <name>br0</name>
  <forward mode="bridge"/>
  <bridge name="br0"/>
</network>
7. Define and activate the bridge network:
bashsudo virsh net-define ~/bridge.xml
sudo virsh net-start br0
sudo virsh net-autostart br0
8. Attach bridge to your VM:
bashsudo virsh attach-interface --domain myvm \
  --type bridge \
  --source br0 \
  --model virtio \
  --config --live
9. Verify inside VM:
bashvirsh console myvm
ip addr   # Should show an IP from your LAN

Part 5: Deploy a Simple Website Inside the VM
1. Connect to your VM:
bashvirsh console myvm
# OR SSH into the VM's IP
ssh username@<vm-ip>
2. Get the VM's IP address:
bashvirsh domifaddr myvm
3. Inside the VM — install Nginx:
bashsudo apt update
sudo apt install -y nginx
sudo systemctl enable --now nginx
4. Create your simple website:
bashsudo nano /var/www/html/index.html
Paste this HTML:
html<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>My KVM Website</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100vh;
      background: #1e1e2e;
      color: #cdd6f4;
    }
    .card {
      text-align: center;
      padding: 40px;
      background: #313244;
      border-radius: 16px;
      box-shadow: 0 8px 32px rgba(0,0,0,0.4);
    }
    h1 { color: #89b4fa; }
    p  { color: #a6e3a1; }
  </style>
</head>
<body>
  <div class="card">
    <h1>🚀 Hello from KVM!</h1>
    <p>This website is running inside a KVM virtual machine.</p>
    <p>Served by <strong>Nginx</strong> on Ubuntu Server.</p>
  </div>
</body>
</html>
5. Allow firewall access:
bashsudo ufw allow 'Nginx Full'
sudo ufw enable
6. Test the website:
From your host machine, open a browser and go to:
http://<vm-ip-address>
Or test via terminal:
bashcurl http://<vm-ip-address>
