In  this repo, I'm providing demo of complete installation of free5gc. Agenda for this demo is as below.

1) Readiness of ubuntu linux for free5gc, 
2) Installation of Container Runtime (Docker + Mirantis Container Runtime)
3) Baremetal installation of k8s cluster
4) Finally free5gc installation and basic ue browsing test


**Readiness of ubuntu linux for free5gc**

```
sudo su -
apt-get install -y linux-image-generic
apt install linux-headers-5.4.0-162-generic
vi /etc/default/grub
        GRUB_DEFAULT='Advanced options for Ubuntu>Ubuntu, with Linux 5.4.0-162-generic'
update-grub
reboot
apt install curl git make gcc -y
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

**Installation of Container Runtime**

```
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
swapoff -a
mount -a
free -h

cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

lsmod | grep br_netfilter
lsmod | grep overlay

** sysctl params required by setup, params persist across reboots**
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

 Apply sysctl params without reboot
```
sysctl --system
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

Install Docker Engine
```
apt-get update
apt-get install ca-certificates curl gnupg -y
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get update
apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
groupadd docker
usermod -aG docker $USER
newgrp docker
docker run hello-world
systemctl enable docker.service
systemctl enable containerd.service
```

Install Mirantis Container Runtime
```
VER=$(curl -s https://api.github.com/repos/Mirantis/cri-dockerd/releases/latest|grep tag_name | cut -d '"' -f 4|sed 's/v//g')
echo $VER
wget https://github.com/Mirantis/cri-dockerd/releases/download/v${VER}/cri-dockerd-${VER}.amd64.tgz
tar xvf cri-dockerd-${VER}.amd64.tgz
mv cri-dockerd/cri-dockerd /usr/local/bin/
cri-dockerd --version
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
mv cri-docker.socket cri-docker.service /etc/systemd/system/
sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
systemctl daemon-reload
systemctl enable cri-docker.service
systemctl enable --now cri-docker.socket
systemctl status cri-docker.socket
```

**Baremetal deployment of k8s cluster**

```
apt-get update
apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list

apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
kubeadm init --pod-network-cidr=172.24.0.0/16 --cri-socket unix:///run/cri-dockerd.sock

mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml
sed -ie 's/192.168.0.0/172.24.0.0/g' custom-resources.yaml
kubectl create -f custom-resources.yaml
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
kubectl taint nodes --all node-role.kubernetes.io/master-

curl -L https://github.com/projectcalico/calico/releases/latest/download/calicoctl-linux-amd64 -o calicoctl
mv calicoctl /usr/local/bin/
chmod +x /usr/local/bin/calicoctl

curl -L https://github.com/projectcalico/calico/releases/latest/download/calicoctl-linux-amd64 -o kubectl-calico
mv kubectl-calico /usr/local/bin/
chmod +x /usr/local/bin/kubectl-calico

git clone https://github.com/k8snetworkplumbingwg/multus-cni.git ; cd multus-cni
cat ./deployments/multus-daemonset-thick.yml | kubectl apply -f -

echo 'source <(kubectl completion bash)' >>~/.bashrc
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
source .bashrc
```

**Installing Free5GC**

Building gtp5g module
```
git clone https://github.com/free5gc/gtp5g.git
cd gtp5g
make
make install
```

```
cd /root/ ; mkdir kubedata
mkdir /root/5gc ; cd /root/5gc
helm repo add towards5gs 'https://raw.githubusercontent.com/Orange-OpenSource/towards5gs-helm/main/repo/'
helm repo update
helm search repo
helm pull towards5gs/free5gc; helm pull towards5gs/ueransim
cd /root/5gc/
vi pv.yaml
---------
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-local-pv9
  labels:
    project: free5gc
spec:
  capacity:
    storage: 8Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  local:
    path: /root/kubedata
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - <nodename>
```

Deploying Helm Charts
```
tar -zxvf ueransim-2.0.17.tgz
tar -zxvf free5gc-1.1.7.tgz
kubectl create ns up
kubectl create ns cp
kubectl create ns an

helm install userplane -n up \
--set global.n4network.masterIf=ens33 \
--set global.n3network.masterIf=ens33 \
--set global.n6network.masterIf=ens33 \
--set global.n6network.subnetIP="10.100.60.0" \
--set global.n6network.gatewayIP="10.100.60.2" \
--set upf.n6if.ipAddress="10.100.60.112" \
free5gc-upf

helm upgrade --install controlplane -n cp \
--set deployUPF=false \
--set global.n2network.masterIf=ens33 \
--set global.n3network.masterIf=ens33 \
--set global.n4network.masterIf=ens33 \
--set global.n6network.masterIf=ens33 \
--set global.n9network.masterIf=ens33 \
free5gc

** Login to webinterface and create UE**

helm upgrade --install an -n an \
--set global.n2network.masterIf=ens33 \
--set global.n3network.masterIf=ens33 \
ueransim
```

