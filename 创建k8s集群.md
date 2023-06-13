# <center>Ubuntu创建K8S集群步骤</center>

# 一、机器的准备

## 1、所有机器的设置

### （1）设置主机名

```shell
#master节点的名字
hostnamectl set-hostname k8s-master
#worker节点名字设置
hostnamectl set-hostname k8s-node1
hostnamectl set-hostname k8s-node2
...
```

### （2）同步时间

```shell
#安装工具
apt-get install ntpdate -y
#同步时间
ntpdate time.windows.com
```

### （3）防火墙设置

```shell
#查看防火墙状态
ufw status
#如果防护墙未关闭，则需要关闭
ufw disable
```

### （4）SElinux设置

Ubuntu默认是没有安装SELinux的，如果你安装了SELinux，如果安装了则需要修改配置文件，

```shell
vi /etc/selinux/config

SELINUX=disabled
```

### （5）虚拟交换分区设置

```shell
#永久关闭swap
sed -ri 's/.*swap.*/#&/' /etc/fstab
#查看
cat /etc/fstab
```

### （6）将桥接的 IPv4 流量传递到 iptables 的链

一般新建的机器上，没有设置，则可以添加以下设置

```shell
#修改/etc/sysctl.conf文件
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.conf
echo "net.ipv6.conf.all.disable_ipv6 = 1" >> /etc/sysctl.conf
echo "net.ipv6.conf.default.disable_ipv6 = 1" >> /etc/sysctl.conf
echo "net.ipv6.conf.lo.disable_ipv6 = 1" >> /etc/sysctl.conf
echo "net.ipv6.conf.all.forwarding = 1"  >> /etc/sysctl.conf

#加载 br_netfilter 模块
modprobe br_netfilter

#永久生效
sysctl -p
```

### （7）安装ipvs

```shell
#安装工具，后续会有相应的配置
apt-get install ipset ipvsadm -y
```

### （8）集群节点机器的优化

```shell
#编辑/etc/sysctl.conf
vim /etc/sysctl.conf
#添加下面两行配置
kernel.pid_max = 4194304
fs.inotify.max_user_watches=1048576

#编辑/etc/security/limits.conf
vi /etc/security/limits.conf
#添加下面四行配置
root soft nofile 102400
root hard nofile 102400
root soft nproc 102400
root hard nproc 102400
```

### （9）重启

之前有的设置需要重启后才能生效

```shell
reboot
```

### （10）安装docker并配置

docker 的版本需要和k8s的组件版本匹配

```shell
#查看版本
apt-cache madison docker.io
 docker.io | 20.10.21-0ubuntu1~20.04.2 | http://cn.archive.ubuntu.com/ubuntu focal-updates/universe amd64 Packages
 docker.io | 20.10.21-0ubuntu1~20.04.2 | http://cn.archive.ubuntu.com/ubuntu focal-security/universe amd64 Packages
 docker.io | 19.03.8-0ubuntu1 | http://cn.archive.ubuntu.com/ubuntu focal/universe amd64 Packages

#因为目前最新的版本是20.10的版本，所以可以直接下载，否则需要加版本号
apt-get install docker.io

#安装完成后，修改配置
{
    "data-root": "/mnt/data/docker",
    "bip":"172.18.0.1/16",
    "insecure-registries":["http://harbor.geovisearth.com:8080"],
    "exec-opts": ["native.cgroupdriver=systemd"],
    "default-address-pools":
        [
            {
                "base": "172.18.0.0/16",
                "size": 24
            }
        ]
}

#重启docker
systemctl restart docker
```

### （11）添加k8s镜像源

```shell
# 安装基础环境
apt-get install -y ca-certificates curl software-properties-common apt-transport-https curl
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
# 执行配置k8s阿里云源  
vim /etc/apt/sources.list.d/kubernetes.list
#加入以下内容
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
# 执行更新
apt-get update -y
```

### （12）安装kubeadm、kubectl、kubelet

需要指定相应的版本

```shell
# 安装kubeadm、kubectl、kubelet  
apt install  kubelet=1.21.10-00 kubeadm=1.21.10-00 kubectl=1.21.10-00
```

### （13）kubelet配置

实现Docker 使用的 cgroup drvier 和 kubelet 使用的 cgroup driver 一致

```shell
#编辑配置文件/etc/systemd/system/kubelet.service.d/10-kubeadm.conf 
vi /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
#加上--cgroup-driver=systemd
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --cgroup-driver=systemd"

#添加ipvs配置
vi /etc/default/kubelet
KUBE_PROXY_MODE="ipvs"

#重新加载配置
systemctl daemon-reload
systemctl restart kubelet
systemctl enable kubelet
```

### （14）安装K8S所需镜像

```shell
#查看所需的镜像
kubeadm config images list

#根据所需镜像的版本拉取相应的镜像版本
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.21.14
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.21.14
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.21.14
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.21.14
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.4.1
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:v1.8.0
```



# 二、集群的初始化

## 1、Master的初始化

### （1）查看master节点的ip

```shell
ip a
```

### （2）初始化master节点

```shell
#由于默认拉取镜像地址k8s.gcr.io国内无法访问，这里需要指定阿里云镜像仓库地址
#1.1.1.1是master节点的ip apiserver-advertise-address 一定要是主机的IP地址
#apiserver-advertise-address 、service-cidr 和 pod-network-cidr 不能在同一个网络范围内
#不要使用 172.17.0.1/16 网段范围，因为这是 Docker 默认使用的。
kubeadm init \
  --apiserver-advertise-address=1.1.1.1 \
  --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers \
  --kubernetes-version=v1.21.10 \
  --service-cidr=10.96.0.0/16 \
  --pod-network-cidr=10.244.0.0/16
```

### （3）按照初始化命令结果的提示设置

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

#如果是 root 用户，还可以执行如下命令
export KUBECONFIG=/etc/kubernetes/admin.conf
```

### （4）安装nfs-server

**注：需要安装kubesphere时需要安装**

```shell
#安装
apt install -y nfs-kernel-server

#修改配置文件/etc/exports
vim /etc/exports
#添加 /mnt/data/k8snfs为设置的nfs共享的路径
/mnt/data/k8snfs  *(rw,sync,no_root_squash,no_subtree_check)

#重启nfs-server
systemctl restart nfs-kernel-server
```



## 2、worker节点的初始化

每个节点上都需要执行

### （1）加入集群

```shell
#根据master节点初始化的结果提示输入命令，在每个节点上执行
kubeadm join 192.168.65.100:6443 --token tluojk.1n43p0wemwehcmmh \
	--discovery-token-ca-cert-hash sha256:c50b25a5e00e1a06cef46fa5d885265598b51303f1154f4b582e0df21abfa7cb
```

### （2）安装nfs-common

```shell
apt install -y nfs-common
```



## 3、查看节点

```shell
#所有节点都初始化之后，可以查看下节点
kubectl get node
```

此时，会出现节点的状态为notready的状态，就需要部署网络插件了

## 4、部署网络插件

可在master或者连接集群的机器上部署

```shell
#下载部署文件
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

#根据文件部署
kubectl apply -f kube-flannel.yml

#此时再查看node，会发现当前所有的node状态为ready
kuebctl get node
```

## 5、设置 kube-proxy 的 ipvs 模式

### （1）编辑kube-proxy的configmap

```shell
kubectl edit cm kube-proxy -n kube-system

#将mode设置为ipvs
apiVersion: v1
data:
  config.conf: |-
    apiVersion: kubeproxy.config.k8s.io/v1alpha1
    bindAddress: 0.0.0.0
    bindAddressHardFail: false
    clientConnection:
      acceptContentTypes: ""
      burst: 0
      contentType: ""
      kubeconfig: /var/lib/kube-proxy/kubeconfig.conf
      qps: 0
    clusterCIDR: 10.244.0.0/16
    configSyncPeriod: 0s
    conntrack:
      maxPerCore: null
      min: null
      tcpCloseWaitTimeout: null
      tcpEstablishedTimeout: null
    detectLocalMode: ""
    enableProfiling: false
    healthzBindAddress: ""
    hostnameOverride: ""
    iptables:
      masqueradeAll: false
      masqueradeBit: null
      minSyncPeriod: 0s
      syncPeriod: 0s
    ipvs:
      excludeCIDRs: null
      minSyncPeriod: 0s
      scheduler: ""
      strictARP: false
      syncPeriod: 0s
      tcpFinTimeout: 0s
      tcpTimeout: 0s
      udpTimeout: 0s
    kind: KubeProxyConfiguration
    metricsBindAddress: ""
    mode: ""
    nodePortAddresses: null
      minSyncPeriod: 0s
      syncPeriod: 0s
    ipvs:
      excludeCIDRs: null
      minSyncPeriod: 0s
      scheduler: ""
      strictARP: false
      syncPeriod: 0s
      tcpFinTimeout: 0s
      tcpTimeout: 0s
      udpTimeout: 0s
    kind: KubeProxyConfiguration
    metricsBindAddress: ""
    mode: "ipvs" # 修改此处
```

### （2）更新kube-proxy pod

```shell
#删除 kube-proxy ，让 Kubernetes 集群自动创建新的 kube-proxy 
kubectl delete pod -l k8s-app=kube-proxy -n kube-system
```



# 三、设置集群外节点连接

将master节点的~路径想的.kube 文件夹拷贝到想要连接集群的~文件夹下即可

```shell
#在master上执行
cd ~
scp -r .kube root@1.1.1.1:~
```



# 四、安装kubesphere容器管理平台

可以在连接集群的集群外节点上部署

## 1、部署默认的存储类SC

### （1）使用helm部署SC

```shell
#使用helm安装（需先安装helm）
#nfs.server是前面设置的nfs-server的ip
#nfs.path是在nfs-server上设置的nfs共享的路径
helm install nfs-client-provisioner stable/nfs-client-provisioner --set nfs.server=192.168.1.154 --set nfs.path=/mnt/data/k8snfs --set storageClass.defaultClass=true --set storageClass.reclaimPolicy=Retain
```

### （2）查看SC

此时，会多一个默认的SC

```shell
root@k8s-client:~# kubectl get sc -A
NAME                   PROVISIONER                            RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-client (default)   cluster.local/nfs-client-provisioner   Retain          Immediate           true                   21h
```

## 2、安装kubesphere

```shell
#通过yaml的形式安装
kubectl apply -f https://github.com/kubesphere/ks-installer/releases/download/v3.3.2/kubesphere-installer.yaml
kubectl apply -f https://github.com/kubesphere/ks-installer/releases/download/v3.3.2/cluster-configuration.yaml
```

## 3、检查安装日志

```shell
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l 'app in (ks-install, ks-installer)' -o jsonpath='{.items[0].metadata.name}') -f
```

log的最后会给出kubesphere的登录方式

## 4、问题处理

### （1）prometheus容器报错

```shell
#查看pod,可能会出现prometheus-k8s-0和prometheus-k8s-0的容器报错
kubecttl get pod -A

#需要在master节点上修改kube-apiserver
vi /etc/kubernetes/manifests/kube-apiserver.yaml

#在 - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key下面添加如下：
- --feature-gates=RemoveSelfLink=false

#然后更新
kubectl apply -f /etc/kubernetes/manifests/kube-apiserver.yaml
```

### （2）nodeexporter容器报错

如果在创建集群之前就安装了node_exporter，那么kubesphere的的node_exporter容器会因端口占用而无法创建，需要作以下修改

```shell
#修改这两个地方,将9100端口换成19100即可
kubectl edit svc node-exporter -n kubesphere-monitoring-system
kubectl edit ds node-exporter -n kubesphere-monitoring-system
```

