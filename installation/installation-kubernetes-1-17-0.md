## Kubernetes HA Installation Manual
​
##### Component
​
- Kubernetes 1.17.0
- kubeadm 1.17.0
- kubelet 1.17.0
- kubernetes-cni 0.7.5
- Docker CE 19.03.8
- RHEL 7.8
​
​
​
#### VMs
​
- 3 VMs for Kubernetes Masters (Keepalived Installed in Master 1)
- VMs for Kubernetes Workers
​
​
​
#### Prerequisite Package
​
- rpm-packages.tar.gz (assume the package is under path /Source)
​
​
​
​
​
### 1.) Install Docker and Kubernetes for all workers and masters
​
- Run command as root user.
​
- Apply these steps to all master and worker nodes
​
 ​
##### 1.1) Base Prerequisite
​
Add kubernetes user (* execute on master nodes only)
​
```shell
groupadd k8sadm -g 5100
useradd k8sadm -u 5101 -s /bin/bash -d /home/k8sadm -g k8sadm
echo k8sadm | passwd --stdin k8sadm
echo "k8sadm        ALL=(ALL)       ALL" >> /etc/sudoers
```
​
​
​
Untar the package
​
```shell
cd /Source
tar -xzvf rpm-packages.tar.gz
```
​
​
​
##### 1.2) Install docker
​
Install yum utilities
​
```shell
yum install -y yum-utils
```
​
​
​
Install Docker file drivers
​
```shell
yum install -y --cacheonly --disablerepo=* /Source/rpm-packages/docker/dm/*.rpm
yum install -y --cacheonly --disablerepo=* /Source/rpm-packages/docker/lvm2/*.rpm
```
​
​
​
Install container-selinux
​
```shell
yum install -y policycoreutils-python
yum install -y --cacheonly --disablerepo=* /Source/rpm-packages/docker/se/*.rpm
```
​
​
​
Install Docker
​
```shell
yum install -y libseccomp
yum install -y --cacheonly --disablerepo=* /Source/rpm-packages/docker/docker-ce/*.rpm
```
​
​
​
Enable and start docker service
​
```shell
systemctl enable docker
systemctl start docker
systemctl status docker
docker version
```
​
​
​
Add username to the docker group to avoid sudo  (* execute on master nodes only)
​
```shell
su - k8sadm
sudo usermod -aG docker $USER
newgrp docker
exit
```
​
​
​
##### 1.3) Install Kubernetes
​
Disable Firewall
​
```shell
yum install -y --cacheonly --disablerepo=* /Source/rpm-packages/firewalld/*.rpm
yes Y | systemctl disable firewalld
yes Y | systemctl stop firewalld
systemctl status firewalld
```
​
​
​
Install Kubernetes
​
```shell
yum install -y --cacheonly --disablerepo=* /Source/rpm-packages/kubernetes/*.rpm
```
​
​
​
Enable Kubelet service
​
```shell
systemctl enable kubelet
```
​
​
​
Turn-off swap & Permanently disable swap
​
```shell
swapoff -a
sed -e '/swap/ s/^#*/#/' -i /etc/fstab
```
​
​
​
### 2.) Install Keepalived
​
- Run command as root user.
- Apply these steps to all master nodes
​
​
​
##### 2.1) Install Prerequisite Package
​
```shell
yum install -y libmpc
yum install -y cpp
yum install -y glibc-devel
yum install -y --cacheonly --disablerepo=* /Source/rpm-packages/keepalived/*.rpm
```
​
​
​
##### 2.2) Configurations
​
```shell
mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
```
​
```shell
vi /etc/keepalived/keepalived.conf
```
​
Copy these lines to the script and change the interface, hostname according to VM, virtual IP **
​
For Master1 node:
​
- ```shell
 vrrp_instance VI_01 {
         state MASTER
         interface <interface name(ex. eth0)>
         virtual_router_id 51
         priority 103
         advert_int 1
         unicast_peer {
                 <Master2 hostname>
                 <Master3 hostname>
         }
         mcast_src_ip <Master1 hostname>
         authentication {
           auth_type PASS
           auth_pass 12345678
         }
          virtual_ipaddress {
                 <virtual ip>/<netmask> label <interface name(ex. eth0)>:1
         }
  }
 ```
​
For Master2 node:
​
- ```shell
 vrrp_instance VI_01 {
         state MASTER
         interface <interface name(ex. eth0)>
         virtual_router_id 51
         priority 102
         advert_int 1
         unicast_peer {
                 <Master1 hostname>
                 <Master3 hostname>
         }
         mcast_src_ip <Master2 hostname>
         authentication {
           auth_type PASS
           auth_pass 12345678
         }
          virtual_ipaddress {
                 <virtual ip>/<netmask> label <interface name>(ex. eth0):1
         }
  }
 ```
​
For Master3 node:
​
- ```shell
 vrrp_instance VI_01 {
         state MASTER
         interface <interface name(ex. eth0)>
         virtual_router_id 51
         priority 101
         advert_int 1
         unicast_peer {
                 <Master1 hostname>
                 <Master2 hostname>
         }
         mcast_src_ip <Master3 hostname>
         authentication {
           auth_type PASS
           auth_pass 12345678
         }
         virtual_ipaddress {
                 <virtual ip>/<netmask> label <interface name(ex. eth0)>:1
         }
  }
 ```
​
​
​
##### 2.3) Restart the service
​
```shell
systemctl restart keepalived
```
​
​
​
### 3.) Kubernetes Images and Network installation
​
- Run command as k8sadm user.
- Apply these steps to all master and worker nodes
​
​
​
##### 3.1) Install kubernetes Images
​
```shell
cd /Source/rpm-packages/kube-images
​
docker load -i coredns:1.6.5.tar
docker load -i etcd:3.4.3-0.tar
docker load -i kube-apiserver:v1.17.0.tar
docker load -i kube-controller-manager:v1.17.0.tar
docker load -i kube-proxy:v1.17.0.tar
docker load -i kube-scheduler:v1.17.0.tar
docker load -i pause:3.1.tar
```
​
​
​
##### 3.2) Install kubernetes Network
​
```shell
cd /Source/rpm-packages/kube-network
​
docker load -i cni:v3.8.8-1.tar
docker load -i kube-controllers:v3.8.8.tar
docker load -i node:v3.8.8-1.tar
docker load -i pod2daemon-flexvol:v3.8.8.tar
```
​
​
​
### 4.) Initialize the Master Node
​
- Run command as root user
- Apply these steps only on Master1 **
​
​
​
##### 4.1) Initialize Master1
​
Edit the yaml file for initializing Master1
​
```shell
cd /tmp
vi config
```
​
Copy these lines to the script and change the IP address according to virtual IP of keepalived that have been setup.
​
- ```shell
 apiVersion: kubeadm.k8s.io/v1beta2
 kind: ClusterConfiguration
 kubernetesVersion: stable
 apiServerCertSANs:
 - <virtual ip>
 controlPlaneEndpoint: "<virtual ip>:6443"
 networking:
   podSubnet: 10.201.0.0/16
 apiServerExtraArgs:
   apiserver-count: "3"
 ```
​
​
​
Change the yaml file name.
​
```shell
mv config config.yaml
```
​
​
​
Initialize Master1.
​
```shell
sudo kubeadm init --config=config.yaml --ignore-preflight-errors=all
```
​
​
​
-Copy the “kubeadm join” command line printed as the result of the previous command for other Masters and Workers to join the cluster (in step 5.1.3 and 6) as example below.
​
```shell
#for master2 and master3
kubeadm join 172.31.121.100:6443 --token y6n5eo.z2hmbiragnlvd2ds --discovery-token-ca-cert-hash sha256:1465a5e4a22fc111a3ddc629f80920de190b734c465c26bbcd174800fd9799bb --control-plane
​
#for the other workers
kubeadm join 172.31.121.100:6443 --token y6n5eo.z2hmbiragnlvd2ds --discovery-token-ca-cert-hash sha256:1465a5e4a22fc111a3ddc629f80920de190b734c465c26bbcd174800fd9799bb
```
​
​
​
Change the Kubernetes owner to k8sadm
​
```shell
su - k8sadm
source /home/k8sadm/.bashrc			
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
​
​
​
Apply CNI Network (user : k8sadm)
​
```shell
kubectl apply -f /Source/rpm-packages/kube-network/calico.yaml
```
​
​
​
Check kubernetes status (user : k8sadm)
​
```shell
kubectl get nodes
kubectl get pods --all-namespaces
```
​
​
​
### 5.) Join the Master node
​
​
​
##### 5.1) For the other Master
​
###### 5.1.1) Copy required files from master1 to master2 and master3
​
- Run command as root user.
- Apply these steps to master1 node.
​
​
​
Change file permissions
​
```shell
chmod 755 /etc/kubernetes/pki
chmod 755 /etc/kubernetes/pki/etcd
chmod 644 /etc/kubernetes/pki/ca.crt
chmod 644 /etc/kubernetes/pki/ca.key
chmod 644 /etc/kubernetes/pki/sa.key
chmod 644 /etc/kubernetes/pki/sa.pub
chmod 644 /etc/kubernetes/pki/front-proxy-ca.crt
chmod 644 /etc/kubernetes/pki/front-proxy-ca.key
chmod 644 /etc/kubernetes/pki/etcd/ca.crt
chmod 644 /etc/kubernetes/pki/etcd/ca.key
```
​
​
​
Copy the files to Master2 and Master3
​
```shell
scp /etc/kubernetes/pki/ca.crt <master2 ip>:/Source/
scp /etc/kubernetes/pki/ca.key <master2 ip>:/Source/
scp /etc/kubernetes/pki/sa.key <master2 ip>:/Source/
scp /etc/kubernetes/pki/sa.pub <master2 ip>:/Source/
scp /etc/kubernetes/pki/front-proxy-ca.crt <master2 ip>:/Source/
scp /etc/kubernetes/pki/front-proxy-ca.key <master2 ip>:/Source/
scp /etc/kubernetes/pki/etcd/ca.crt <master2 ip>:/Source/etcd-ca.crt
scp /etc/kubernetes/pki/etcd/ca.key <master2 ip>:/Source/etcd-ca.key
```
​
​
​
Copy the file to Master3
​
```shell
scp /etc/kubernetes/pki/ca.crt <master3 ip>:/Source/
scp /etc/kubernetes/pki/ca.key <master3 ip>:/Source/
scp /etc/kubernetes/pki/sa.key <master3 ip>:/Source/
scp /etc/kubernetes/pki/sa.pub <master3 ip>:/Source/
scp /etc/kubernetes/pki/front-proxy-ca.crt <master3 ip>:/Source/
scp /etc/kubernetes/pki/front-proxy-ca.key <master3 ip>:/Source/
scp /etc/kubernetes/pki/etcd/ca.crt <master3 ip>:/Source/etcd-ca.crt
scp /etc/kubernetes/pki/etcd/ca.key <master3 ip>:/Source/etcd-ca.key
```
​
​
​
###### 5.1.2) Move the key and cert file in Master2 and Master3
​
- Run command as root user.
- Apply these steps to master2 and master3 nodes.
​
```shell
sudo su
mkdir -p /etc/kubernetes/pki/etcd
mv /Source/ca.crt /etc/kubernetes/pki/
mv /Source/ca.key /etc/kubernetes/pki/
mv /Source/sa.pub /etc/kubernetes/pki/
mv /Source/sa.key /etc/kubernetes/pki/
mv /Source/front-proxy-ca.crt /etc/kubernetes/pki/
mv /Source/front-proxy-ca.key /etc/kubernetes/pki/
mv /Source/etcd-ca.crt /etc/kubernetes/pki/etcd/ca.crt
mv /Source/etcd-ca.key /etc/kubernetes/pki/etcd/ca.key
```
​
​
​
###### 5.1.3) Join the Cluster initiated by Master1 in Master2 and Master3
​
- Run command as root user.
- Apply these steps to master2 and master3 nodes.
​
### print kube join command on master node 1 and add tag --ignore-preflight-errors=all
kubeadm token create --print-join-command

Paste the “kubeadm join” command line printed as the result of the 4.) step and add tag --ignore-preflight-errors=all
​
```shell
kubeadm join 172.31.121.100:6443 --token y6n5eo.z2hmbiragnlvd2ds --discovery-token-ca-cert-hash sha256:1465a5e4a22fc111a3ddc629f80920de190b734c465c26bbcd174800fd9799bb --control-plane --ignore-preflight-errors=all
```
​
​
​
Change the Kubernetes owner to k8sadm
​
```shell
su - k8sadm
source /home/k8sadm/.bashrc			
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
​
​
​
Check kubernetes status (user : k8sadm)
​
```shell
kubectl get nodes
kubectl get pods --all-namespaces
```
​
​
​
#### 6.) Join the Cluster for Worker nodes.
​
- Run command as root user
- Apply this step to all Worker nodes.
​
Paste the “kubeadm join” command line printed as the result of the 4.) step and add tag --ignore-preflight-errors=all
​
```shell
kubeadm join 172.31.121.100:6443 --token y6n5eo.z2hmbiragnlvd2ds --discovery-token-ca-cert-hash sha256:1465a5e4a22fc111a3ddc629f80920de190b734c465c26bbcd174800fd9799bb --ignore-preflight-errors=all
```

# OPTIONAL:
=============UPDATE POD CIDR NETWORK ADDRESS=============

1. Create a new ippool for calico using this yaml:
```
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
name: new-pool
spec:
cidr: 10.201.0.0/16
ipipMode: Always
natOutgoing: true
```

2. Backup and disable current ippool which used 192.168.0.0/16
```
kubectl get ippool default-ipv4-ippool -o yaml > ippool-backup.yaml
kubectl edit ippool default-ipv4-ippool
>> disabled: true # add line
```

3. Change network CIDR on kubeadm configmap
```
kubectl -n kube-system edit cm kubeadm-config
```

4. Change network CIDR on controller manager manifest file on every master nodes:
file: /etc/kubernetes/manifests/kube-controller-manager.yaml

5. Restart pods that is still using 192.168.x.x. Special case for CoreDNS and calico-kube-controller pod, should add node affinity to the deployment, so it will only be deployed like where it was deployed before
