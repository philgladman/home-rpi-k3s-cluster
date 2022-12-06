# home-rpi-k3s-cluster
How to configure and create a K3s Kubernetes Cluster on raspberry pi(s), with MetalLB and Nginx Ingress installed.

## Step 1.) - Raspberry Pi Setup
- Use Raspberry Pi Imager (https://www.raspberrypi.com/software/) to write Ubuntu Server 20.04 LTS (64-BIT) to sd card
- After image has been succesfully written, Eject SD Card from computer, and then reinsert card into computer.
- click on SD Card, and open up the newly created `boot` or `system-boot` folder
- Open up the `network-config.txt` file in a txt editor, create this file if it does not exisit.
- Add the contents below to the end of the file, this will configure your static ip. Replace the ip address with the ip address you want your rpi to have. Also replace the gateway ip, and dns ip with your home networks values.

```bash
version: 2
ethernets:
  eth0:
    addresses:
      - 192.168.1.x/24
    gateway4: 192.168.1.x
    nameservers:
      addresses: [192.168.1.x]
    optional: true
```

- In the `boot`  or `system-boot` folder, open up the `cmdline.txt` file in a txt editor
- Add the following `cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory` to the end of the top line, this will enable cgroups
- add a blank file named `ssh` into the `boot` or `system-boot` folder by running the following command. `touch ssh` to create the blank `ssh` file. Find newly created `ssh` file and move it into the sd cards `boot` or `system-boot` folder.
- Eject SD card and plug into rpi.
- Turn rpi on and let rpi boot up.
- ssh into the pi with `ssh ubuntu@<rpi-ip-address>`. The password will be `ubuntu`
- change password

## Step 2.) - Deploy and configure k3s single node cluster
- create k3s cluster without install teaefik (we will use nginx ingress instead later) `curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --disable traefik" sh`
- copy newly created kubeconfig to home dir `mkdir -p ~/.kube && sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config && sudo chown ubuntu:ubuntu ~/.kube/config`
- export kubeconfig `echo "export KUBECONFIG=~/.kube/config" >> ~/.bashrc && source ~/.bashrc`
- install helm `curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash`

## Step 3.) - Install MetalLB on K3s Cluster
- edit `addresses:` field in the file `kustomize/metallb/metallb-ip-pool.yaml` to have the avaliable ip address on your home network for MetalLB to use. These IP Addresses need to be reserved, so they will not be given out via DHCP.
- FYI - if you want to get the latest metallb deplopyment, run the following command`MetalLB_RTAG=$(curl -s https://api.github.com/repos/metallb/metallb/releases/latest|grep tag_name|cut -d '"' -f 4|sed 's/v//') && wget https://raw.githubusercontent.com/metallb/metallb/v$MetalLB_RTAG/config/manifests/metallb-native.yaml -O kustomize/metallb/metallb-deployment.yaml` This will override the existing `ustomize/metallb/metallb-deployment.yaml` file.

## Step 4.) - Install Nginx Ingress on K3s Cluster
- If your K3s cluster already has Nginx Ingress installed, move onto step [## Step 5.) - Turn off systemd-resolver (Ubuntu)]
