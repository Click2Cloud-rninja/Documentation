Setup Mizar with Kubernetes
==================

## Pre-requisites
### 1. Check Network Interface
`ip a`

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000 
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
inet 127.0.0.1/8 scope host lo
valid_lft forever preferred_lft forever
inet6 ::1/128 scope host
valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP group default qlen 1000
link/ether 00:50:56:86:f2:c1 brd ff:ff:ff:ff:ff:ff
inet 192.168.1.113/24 brd 192.168.1.255 scope global eth0
valid_lft forever preferred_lft forever
inet6 fe80::250:56ff:fe86:f2c1/64 scope link
valid_lft forever preferred_lft forever
```
Here network interface should be `eth0`, if it is `eth0` then skip following section and go to Step 2

If it is not `eth0` then to change interface to eth0 follow following steps
```bigquery
apt remove ifupdown
dmesg | grep -i eth #this will give information
```
Edit Grub file and change `GRUB_CMDLINE_LINUX` as shown below:

`sudo vi /etc/default/grub`

From `GRUB_CMDLINE_LINUX=""` change to `GRUB_CMDLINE_LINUX="net.ifnames=0 biosdevname=0"`
After changing configuration, run following command

`sudo grub-mkconfig -o /boot/grub/grub.cfg`

Edit network manager yaml 

```
vi /etc/netplan/01-network-manager-all.yaml 
#Please give proper file name present /etc/netplan location
```
It should have content like shown below
```bigquery
network:
  version: 2
  ethernets:
          eth0:
            addresses: [192.168.1.82/24]
            gateway4: 192.168.1.1
            nameservers:
              addresses: [8.8.8.8]
```
and then apply configuration before reboot your system

``
sudo netplan generate && netplan apply && sudo reboot
``


### 2. Update kernel
To check kernel, run following command

`uname -a`

If it `5.6.0-rc2` then you can skip following portion and continue with Step-3

To update Kernel download and run script for kernel update

```bigquery
wget https://raw.githubusercontent.com/CentaurusInfra/mizar/dev-next/kernelupdate.sh
chmod 777 kernelupdate.sh
./kernelupdate.sh
```
Please reboot your system after kernel update.

### 3. Install Kubernetes

1.Disable swap and enable IP forwarding on all nodes
```bigquery
swapoff -a
```
Update fstab file
```bigquery
sudo vi /etc/fstab
```

```bigquery
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/sda4 during installation
UUID=6f612675-026a-4547030ff8a7e /               ext4    errors=remount-ro 0       1
# swap was on /dev/sda6 during installation
#UUID=46ee415b-4afa-413c275e264 none    swap  sw    0     0 <------Comment this part
/dev/sda5 /Data               ext4   defaults  0 0
```

2. Install Kubectl
```bigquery
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

3. Letting iptables see bridged traffic
```bigquery
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```
4. Install runtime

```bigquery
sudo apt-get update
sudo apt-get -y install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

```bigquery
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

```bigquery
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
``` 

4. Install Kubeadm and Kubelet

```bigquery
sudo apt-get update
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

```bigquery
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

5. Run K8s cluster
```bigquery
kubeadm init --apiserver-advertise-address=<Host-IP> --pod-network-cidr=20.0.0.0/8
```


### 3. Deploy Mizar

```bigquery
kubectl apply -f https://raw.githubusercontent.com/CentaurusInfra/mizar/dev-next/etc/deploy/deploy.mizar.yaml
```

To check pods

```bigquery
kubectl get pods -A
```

To check status of node

```bigquery
kubectl get nodes
```
