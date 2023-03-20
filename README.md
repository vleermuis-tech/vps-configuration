# Smol-Metal

Notes for configuring Debian Bookworm nodes for use as VPS hosts.
The steps below setup the system to be further controlled by ansible. Eventually most of this will move into a cloid-init or pre-seed files.

## As Sudo:

1. Fix apt sources (Debian only)

```bash
cat << EOF > /etc/apt/sources.list
deb http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware

deb http://deb.debian.org/debian-security/ bookworm-security main contrib non-free
deb-src http://deb.debian.org/debian-security/ bookworm-security main contrib non-free

deb http://deb.debian.org/debian bookworm-updates main contrib non-free
deb-src http://deb.debian.org/debian bookworm-updates main contrib non-free
EOF
```

2. install deps

```bash
# Package Choice Justifications
# sudo - a user with passwordless sudo for automations
# ssh-import-id - ssh key imports
# wireguard - setup vpn (get keys from bitwarden)
# curl - for onboardme
# nvidia-driver firmware-misc-nonfree linux-headers-amd64 are for GPU

apt-get update && apt-get install -y wireguard \
  ssh-import-id \
  sudo \
  curl \
  netplan.io \
  docker.io \
  netplan.io \
  git-extras \
  rsyslog \
  gpg \
  neovim
  

# Debain
apt-get install -y nvidia-driver \
  firmware-misc-nonfree \
  linux-headers-amd64 \
  linux-headers-`uname -r`

# Ubuntu:
apt-get install -y ubuntu-drivers-common linux-headers-generic
ubuntu-drivers install nvidia:525
```

3. setup user

```bash
useradd -s /bin/bash -d /home/friend/ -m -G sudo friend
echo "friend ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
sudo -u friend ssh-import-id-gh cloudymax
passwd friend
```

Add user to docker group
```bash
usermod -a -G docker friend
```

4. bridge the network adapter (optional)

https://wiki.debian.org/NetworkInterfaceNames

/etc/udev/rules.d/70-persistent-net.rules
```bash

```

```bash
# /etc/netplan/99-bridge.yaml
network:
  bridges:
    br0:
      dhcp4: no
      dhcp6: no
      interfaces: [enp4s0]
      addresses: [192.168.50.101/24]
      routes:
        - to: default
          via: 192.168.50.1
      mtu: 1500
      nameservers:
        addresses: [192.168.50.50]
      parameters:
        stp: true
        forward-delay: 4
  ethernets:
    enp4s0:
      dhcp4: no
      dhcp6: no
  renderer: networkd
  version: 2

sudo netplan --debug generate
sudo netplan --debug apply
```

5. Set grub to enable iommu (GPU pass-through only)

```bash
# /etc/default/grub
GRUB_DEFAULT=0
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT="quiet preempt=voluntary iommu=pt amd_iommu=on intel_iommu=on"
GRUB_CMDLINE_LINUX=""

sudo update-grub
sudo reboot now
```

6. Enable GPU passthrough (Optional) 

See: https://github.com/small-hack/smol-gpu-passthrough

```bash
wget https://raw.githubusercontent.com/small-hack/smol-gpu-passthrough/main/setup.sh

bash setup.sh full_run NVIDIA
sudo reboot now
```

## As User:

```bash
sudo nano /etc/wireguard/wg0.conf

sudo systemctl enable wg-quick@wg0

sudo systemctl restart wg-quick@wg0
```

```bash
sudo wget -O /etc/ssh/sshd_config https://raw.githubusercontent.com/cloudymax/linux_notes/main/sshd_config

sudo systemctl reload sshd
```

# For GPUs

Install nvidia-container-tooklit: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html

you may have to manually set distribution=debian11 or debian10 to get this to work on
debian bookworm/sid systems.

```bash
distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
      && curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
      && curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
            sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
            sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

For cuda driveers go here: https://developer.nvidia.com/cuda-downloads

Test with:

```bash
sudo docker run --rm --runtime=nvidia --gpus all nvidia/cuda:11.6.2-base-ubuntu20.04 nvidia-smi
```

## How to run the ansible playbooks

Start the api server:

```bash
# Create a directory for a volume to store settings and a sqlite database
mkdir -p ~/.ara/server

# Start an API server with docker from the image on DockerHub:
docker run --name api-server --detach --tty \
  --volume ~/.ara/server:/opt/ara -p 8000:8000 \
  -e ARA_ALLOWED_HOSTS="['*']" \
  docker.io/recordsansible/ara-api:latest
```

build the ansible runner container

```bash
docker build -t ansible-runner .
```

Run the main playbook (insert your own user and password values)

```bash
docker run --platform linux/amd64 -it \
  -v $(pwd)/ansible:/ansible \
  -e ARA_API_SERVER="http://192.168.50.100:8000" \
  -e ARA_API_CLIENT=http \
  ansible-runner ansible-playbook playbooks/main-playbook.yaml \
  -i sample-inventory.yaml \
  --extra-vars "admin_password=ChangeMe!" \
  --extra-vars "admin_user=ChangeMe"
```
