# k8s 最新版本单节点极速部署
## 1. 系统设置
本次安装使用系统为 ubuntu22.04
### 1.1. 加载内核模块
```bash
# 执行以下命令将需要加载的内核模块写入 /etc/modules-load.d/k8s.conf 文件中
# 节点重启后会自动加载文件中的内核模块，安装过程中可通过命令手动加载不需要重启
cat << EOF > /etc/modules-load.d/k8s.conf
overlay
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
EOF

modprobe -a overlay br_netfilter ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh

# 查看内核模块是否加载成功
lsmod | grep -e ip_vs -e overlay -e br_netfilter -e nf_conntrack
```

### 1.2. 设置内核参数
```bash
# 执行以下命令将需要修改的内核参数写入 /etc/sysctl.d/99-kubernetes-cri.conf 文件中
cat << EOF > /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-ip6tables=1
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_reuse=1
net.ipv4.tcp_max_tw_buckets=10000
net.core.somaxconn=1024
user.max_user_namespaces=28633
vm.swappiness=0
vm.max_map_count=262144
fs.inotify.max_user_watches=10000000
fs.inotify.max_user_instances=81920
EOF

sysctl -p /etc/sysctl.d/99-kubernetes-cri.conf
```
### 1.3. 修改 ulimits
```bash
echo " * soft nofile 65535" > /etc/security/limits.conf
echo " * hard nofile 65535" >> /etc/security/limits.conf
echo " root soft nofile 65535" >> /etc/security/limits.conf
echo " root hard nofile 65535" >> /etc/security/limits.conf
```
### 1.4. 关闭交换分区
```bash
swapoff -a
sed -i '/\sswap\s/s/^/#/' /etc/fstab
```
### 1.5. 安装系统依赖
```bash
apt-get update
apt-get install ipvsadm ipset socat -y
```
## 2. 安装容器运行时
> 使用 nerdctl-full 包安装容器运行时，包含 containerd、runc、nerdctl 等，下载链接
> * arm64 版本：https://github.com/containerd/nerdctl/releases/download/v2.0.2/nerdctl-full-2.0.2-linux-arm64.tar.gz
> * amd64 版本：https://github.com/containerd/nerdctl/releases/download/v2.0.2/nerdctl-full-2.0.2-linux-amd64.tar.gz

安装步骤
```bash
wget https://github.com/containerd/nerdctl/releases/download/v2.0.2/nerdctl-full-2.0.2-linux-amd64.tar.gz

tar xf nerdctl-full-2.0.2-linux-amd64.tar.gz -C /usr/local/

# 配置 containerd
# 生成配置文件，并修改配置文件中的 `SystemdCgroup = false` 为 `SystemdCgroup = true`, containerd v2.0 及以上版本可以不做修改
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml

# 启动 containerd
systemctl daemon-reload
systemctl enable --now containerd

# 查看 containerd 运行状态
systemctl status containerd.service

# 验证是否安装成功
nerdctl version
nerdctl info
```

## 3. 安装 k8s
### 3.1. 安装 kubeadm kubelet kubectl 等
> 针对不同操作系统，安装方式略有差别，其他系统可参考官方文档 https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm

```bash
apt-get update
apt-get install -y apt-transport-https ca-certificates curl

mkdir -p /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list

chmod 644 /etc/apt/sources.list.d/kubernetes.list

apt-get update

# 查询可用版本
apt-cache madison kubeadm
apt-cache madison kubelet
apt-cache madison kubectl

# 安装 kubelet kubeadm kubectl
apt install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

# 启动 kubelet
systemctl enable kubelet --now
```
### 3.2. 使用 kubeadm 初始化集群
#### 3.2.1. 生成集群配置文件
```bash
kubeadm config print init-defaults --component-configs KubeProxyConfiguration,KubeletConfiguration > kubeadm.yaml
```
#### 3.2.2. 修改集群配置
修改刚才生成的 kubeadm.yaml 文件中的以下配置项，其中的 `192.168.220.101` 是当前主机的 ip 地址，根据自身情况进行修改
```bash
localAPIEndpoint:
  advertiseAddress: 192.168.220.101
nodeRegistration:
  # name: node

kubernetesVersion: 1.30.5
controlPlaneEndpoint: 192.168.220.101:6443
networking:
  podSubnet: 10.244.0.0/16

mode: "ipvs"
```
#### 3.2.3. 初始化集群
```bash
# 执行以下命令预拉取镜像，或导入提前准备好的镜像离线包
kubeadm config images pull --config /root/kubeadm.yaml

# 执行以下命令初始化 master1
kubeadm init --node-name master1 --config /root/kubeadm.yaml --upload-certs

# 使用默认配置初始化集群
# kubeadm init --pod-network-cidr="10.244.0.0/16" --upload-certs
```
### 3.2.4. 安装网络插件
安装前需要集群初始化成功，如果使用 helm 安装需要先安装 helm
> 安装 helm https://helm.sh/docs/intro/install/
```bash
# Needs manual creation of namespace to avoid helm error
kubectl create ns kube-flannel
kubectl label --overwrite ns kube-flannel pod-security.kubernetes.io/enforce=privileged

helm repo add flannel https://flannel-io.github.io/flannel/
helm install flannel --set podCidr="10.244.0.0/16" --namespace kube-flannel flannel/flannel
```
