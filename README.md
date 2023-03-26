# Smol-Metal

Notes for configuring Debian Bookworm nodes for use as VPS hosts.
The steps below setup the system to be further controlled by ansible. Eventually most of this will move into a cloid-init or pre-seed files.

## Host Setup:

1. Fix apt sources (Debian only)

    <details>
      <summary>Click to expand</summary>
  
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
  
    </details>


2. install basic dependancies

    ```bash
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
    ```
    
3. Setup the user

    ```bash
    useradd -s /bin/bash -d /home/friend/ -m -G sudo friend
    echo "friend ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
    sudo -u friend ssh-import-id-gh cloudymax
    passwd friend
    ```

4. bridge the network adapter (Optional)

    <details>
      <summary>Click to expand</summary>
  
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

    </details>
    
5. Setup Wireguard (Optional)

    <details>
      <summary>Click to expand</summary>

    ```bash
    sudo nano /etc/wireguard/wg0.conf

    sudo systemctl enable wg-quick@wg0
    sudo systemctl restart wg-quick@wg0
    ```
    </details>

6. Disable insecure ssh login options

    ```bash
    sudo wget -O /etc/ssh/sshd_config https://raw.githubusercontent.com/cloudymax/linux_notes/main/sshd_config

    sudo systemctl reload sshd
    ```

7. Setup PCI/IOMMU Passthrough (Optional)

    <details>
      <summary>Enable iommu via grub</summary>
  
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
  
    </details>


    <details>
      <summary>Setup GPU-Passthrough</summary>
  
    ```bash
    # See: https://github.com/small-hack/smol-gpu-passthrough

    wget https://raw.githubusercontent.com/small-hack/smol-gpu-passthrough/main/setup.sh

    bash setup.sh full_run NVIDIA
    sudo reboot now
    ```
  
    </details>


## Guests

1. Install GPU Drivers (Skip if kuberntes node)

    <details>
      <summary>Debain Drivers</summary>
  
      ```bash
      apt-get install -y nvidia-driver \
      firmware-misc-nonfree \
      linux-headers-amd64 \
      linux-headers-`uname -r`
      ```
  
    </details>


    <details>
      <summary>Ubuntu Drivers</summary>
  
      ```bash
      apt-get install -y ubuntu-drivers-common linux-headers-generic
      ubuntu-drivers install nvidia:525
      ```
  
    </details>
    
      <details>
      <summary>Nvidia Container Runtime</summary>
  
      ```bash
      apt-get install -y ubuntu-drivers-common linux-headers-generic
      ubuntu-drivers install nvidia:525
      ```
    
    </details>  
    
2. Install Container Toolkit

  - nvidia-container-tooklit: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html
  - For Cuda drivers go here: https://developer.nvidia.com/cuda-downloads

    </details>
    
      <details>
      <summary>Ubuntu 22.04</summary>
  
      ```bash
      distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
  
      curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
      && curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
            sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
            sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
 
      sudo apt-get update
      sudo apt-get install -y nvidia-container-toolkit
      sudo nvidia-ctk runtime configure --runtime=docker
      sudo systemctl restart docker
      ```
    
    </details> 
    
    </details>
    
      <details>
      <summary>Debian 12</summary>
  
      ```bash
      distribution=debian11
  
      curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
      && curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
            sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
            sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
 
      sudo apt-get update
      sudo apt-get install -y nvidia-container-toolkit
      sudo nvidia-ctk runtime configure --runtime=docker
      sudo systemctl restart docker
      ```
    
    </details> 
    
3. Test its workign with:

    ```bash
    sudo docker run --rm --runtime=nvidia --gpus all nvidia/cuda:11.6.2-base-ubuntu20.04 nvidia-smi
    ```

## setup smol-k8s-lab

1. install Python3.11 and brew

    ```bash
    wget -O setup.sh https://raw.githubusercontent.com/jessebot/onboardme/main/setup.sh
    . ./setup.sh 
    ```

2. install smol-k8s-lab
```bash
pip3.11 install smol-k8s-lab
```

3. write setup config to `~/.config/smol-k8s-lab/config.yaml`

```bash
mkdir -p ~/.config/smol-k8s-lab
nvim ~/.config/smol-k8s-lab/config.yaml
```

```yaml
domain:
  base: "cloudydev.net"
  argo_cd: "argocd"
  minio: "minio"
  minio_console: "console.minio"
metallb_address_pool:
   - 10.0.2.16/32
   - 10.0.2.17/32
   - 10.0.2.18/32
email: "admin@cloudydev.net"
external_secrets:
  enabled: false
log:
  level: "info"
```

```bash
export KUBECONFIG="~/.config/kube/config"
```

4. Install HAproxy (if usiing SLIRP VM)

5. Nvidia gpu operator

```bash

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
