# home-rpi-k3s-cluster
How to create a K3s Kubernetes Cluster on raspberry pi(s), with MetalLB and Nginx Ingress installed, utilizing Kustomization and Kustomized Helm Charts.

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
- repeat for each raspberry pi

## Single Node Cluster
- Move on to next step for Multi Node Clusters
- update the node `sudo apt update` and then install some extra modules with `sudo apt install linux-modules-extra-raspi`
- create k3s cluster without install teaefik (we will use nginx ingress instead later) `curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --disable traefik --disable servicelb" sh`
- copy newly created kubeconfig to home dir `mkdir -p ~/.kube && sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config && sudo chown ubuntu:ubuntu ~/.kube/config`
- export kubeconfig `echo "export KUBECONFIG=~/.kube/config" >> ~/.bashrc && source ~/.bashrc`
- install helm `curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash`

# Multi Node Cluster (4 Raspberry Pis)
- For multi node clusters, we will deploy k3s with ansible `brew install ansible`
- The current configuration is for a 4 pi cluster. IF you have more or less than 4 pis, do the steps in [here](README.md#edit-config-for-moreless-than-4-pis) first.
- First we will need to create a ssh key locally that we will then copy to each raspberry pi.
- run `ssh-keygen` on your computer, give it a path such as `/home/ubuntu/.ssh/ansible-key`, and hit enter for the passphrase inorder to create a key without a passphrase.
- copy new `ansible-key` to each pi with this command `ssh-copy-id -i /home/ubuntu/.ssh/ansible-key ubuntu@<rpi-ip-address>`. You will need to do this for each pi.
- clone this repo `git clone https://github.com/philgladman/home-rpi-k3s-cluster.git`
- cd into the repo `cd home-rpi-k3s-cluster`
- edit the `ansible/group_vars/all` file with the location of your new `ssh_private_key_file`, as well as your `local_home_dir`
- edit the `ansible/group_vars/all` file with the ipaddresses for each node (pi), the file path of the new ssh key you just created, and the home directory of your computer that the ssh key is on.
- If you have more or less than 4 raspberry pis, update the `ansible/group_vars/all` file to more or less `workerXX_ipaddress` variables. You will need to do the same for `ansible/inventory.txt`
- update the hostname of each node with this `ansible-playbook -i ansible/inventory.txt ansible/k3s/update-hostnames.yml`
- Install k3s with this `ansible-playbook -i ansible/inventory.txt ansible/k3s/install-k3s.yaml`. This ansible playbook will update all the nodes, install k3s, join the agent nodes to the cluster, and then copy the kubeconfig to your local machine.
- When the playbook finishes, run `kubectl get nodes` to see if your cluster is up and your nodes are ready.

## Edit Config for more/less than 4 pis
- If you have more or less than 4 raspberry pis, update the `ansible/group_vars/all` file to more or less `workerXX_ipaddress` variables. 
- You will need to do the same for `ansible/inventory.txt`
- Lastly, you will need to update the `ansible/k3s/update-hostnames.yaml` file to have the correct number of host.

## Install MetalLB on K3s 
- If you have not done so already, clone this repo `git clone https://github.com/philgladman/home-rpi-k3s-cluster.git` and then cd into the repo `cd home-rpi-k3s-cluster`
- edit `addresses:` field in the file `kustomize/metallb/metallb-ip-pool.yaml` to have the avaliable ip address on your home network for MetalLB to use. These IP Addresses need to be reserved, so they will not be given out via DHCP. `sed -i 's/192.168.1.x-192.168.1.x/<your-ip-range>/g' kustomize/metallb/metallb-ip-pool.yaml`
- Install MetalLB `kubectl apply -k kustomize/metallb/.`
- If you receive an error such as `ensure CRDs are installed first`, or a `Error from server...`, re run the kubectl apply command.

## Install Nginx Ingress on K3s Cluster
- Install Nginx Ingress `kubectl apply -k kustomize/nginx-ingress/.`

### Misc Info
- To get the latest metallb deployment, run the following command  before running kubectl apply `MetalLB_RTAG=$(curl -s https://api.github.com/repos/metallb/metallb/releases/latest|grep tag_name|cut -d '"' -f 4|sed 's/v//') && wget https://raw.githubusercontent.com/metallb/metallb/v$MetalLB_RTAG/config/manifests/metallb-native.yaml -O kustomize/metallb/metallb-deployment.yaml`
- FYI - `kustomize/nginx-ingress/release.yaml` was created with the following command `helm template nginx-ingress charts/nginx-ingress -f kustomize/nginx-ingress/values.yaml --include-crds --debug > kustomize/nginx-ingress/release.yaml`
