# Install K8S Cluster HA on RHEL 10.1

## K8S arch

1.Linux OS

RHEL 10.1

kernal: 6.12.0-124.56.1.el10_1.x86_64

CPU:4

Memory:8G 

Harddisk: 50G



2.Kubernetes version: v1.36.1

3.CRI: containerd-2.3.0-linux-amd64

4.cgroup drivers: systemd

5.CNI: Cilium 1.19.4 install by helm

6.Deploy method: kubeadm

7.HA cluster: Stacked etcd topology

8.HA LB: kube-vip

9.Ingress LB: MetalLB

10.Ingress Controller: ingress-nginx

## 0.Environmental preparation

1. install 3 VMs RHEL 10.1  by minimal 
2. config DNS

```
10.124.3.11    dlc-aci-k8s01.com  (control-plan 1)
10.124.3.12    dlc-aci-k8s02.com  (control-plan 2)
10.124.3.13    dlc-aci-k8s03.com  (control-plan 3)
10.124.3.14    dlc-aci-k8s04.com  (work 1)
10.124.3.15    dlc-aci-k8s05.com  (work 2)
10.124.3.16    dlc-aci-k8s06.com  (work 3)
10.124.3.17    dlc-aci-k8s07.com  (work 4)
10.124.3.18    dlc-aci-k8s08.com  (work 5)
```

3. disable Firewall on all Node. 

   ```bash
   sudo systemctl disable firewalld.service 
   sudo systemctl stop firewalld.service 
   sudo systemctl status firewalld.service 
   ```

4. disable swap on all Node.

   ```bash
   #
   sudo swapoff -a
   #
   sudo vim /etc/fstab
   #UUID=ce6c1494-cb10-4e84-bb2b-d289f7113ac9 none                    swap    defaults        0 0
   
   ```

5. disable SELINUX on all Node.

   ```bash
   sudo setenforce 0
   sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
   ```

6. config NTP on all Node.

7. upgrade to software 

   ```bash
   sudo dnf upgrade
   ```

   

   

## 1.install Container Runtimes(Containerd) on all Nodes.

### Install and configure prerequisites

###  Enable IPv4 packet forwarding

### (https://kubernetes.io/docs/setup/production-environment/container-runtimes/#prerequisite-ipv4-forwarding-optional)

Execute the below mentioned instructions:

```bash
# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

# Verify that net.ipv4.ip_forward is set to 1 with:
sudo sysctl net.ipv4.ip_forward

```



### Installing containerd

The official binary releases of containerd are available for the `amd64` (also known as `x86_64`) and `arm64` (also known as `aarch64`) architectures.

Typically, you will have to install [runc](https://github.com/opencontainers/runc/releases) and [CNI plugins](https://github.com/containernetworking/plugins/releases) from their official sites too.

#### Step 1: Installing containerd

Download the `containerd-<VERSION>-<OS>-<ARCH>.tar.gz` archive from https://github.com/containerd/containerd/releases , verify its sha256sum, and extract it under `/usr/local`:

```bash
wget https://github.com/containerd/containerd/releases/download/v2.3.0/containerd-2.3.0-linux-amd64.tar.gz

sudo tar Cxzvf /usr/local containerd-2.3.0-linux-amd64.tar.gz 
bin/
bin/containerd
bin/containerd-stress
bin/ctr
bin/containerd-shim-runc-v2
```



The `containerd` binary is built dynamically for glibc-based Linux distributions such as Ubuntu and Rocky Linux. This binary may not work on musl-based distributions such as Alpine Linux. Users of such distributions may have to install containerd from the source or a third party package.

> **FAQ**: For Kubernetes, do I need to download `cri-containerd-(cni-)<VERSION>-<OS-<ARCH>.tar.gz` too?
>
> **Answer**: No.
>
> As the Kubernetes CRI feature has been already included in `containerd-<VERSION>-<OS>-<ARCH>.tar.gz`, you do not need to download the `cri-containerd-....` archives to use CRI.
>
> The `cri-containerd-...` archives are [deprecated](https://github.com/containerd/containerd/blob/main/RELEASES.md#deprecated-features), do not work on old Linux distributions, and will be removed in containerd 2.0.

##### systemd

If you intend to start containerd via systemd, you should also download the `containerd.service` unit file from https://raw.githubusercontent.com/containerd/containerd/main/containerd.service into `/usr/local/lib/systemd/system/containerd.service`, and run the following commands:

```bash
wget  https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo mv containerd.service  /etc/systemd/system/
sudo chown root:root /etc/systemd/system/containerd.service 


sudo systemctl daemon-reload
sudo systemctl enable --now containerd

#if SELINUX is enable, the containerd services will be failed.
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
Failed to enable unit: Unit containerd.service does not exist
```



#### Step 2: Installing runc

Download the `runc.<ARCH>` binary from https://github.com/opencontainers/runc/releases , verify its sha256sum, and install it as `/usr/local/sbin/runc`.

```bash
wget https://github.com/opencontainers/runc/releases/download/v1.5.0-rc.2/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```



The binary is built statically and should work on any Linux distribution.

#### Step 3: Installing CNI plugins

Download the `cni-plugins-<OS>-<ARCH>-<VERSION>.tgz` archive from https://github.com/containernetworking/plugins/releases , verify its sha256sum, and extract it under `/opt/cni/bin`:

```bash
wget https://github.com/containernetworking/plugins/releases/download/v1.9.1/cni-plugins-linux-amd64-v1.9.1.tgz
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.9.1.tgz
./
./sbr
./tap
./dhcp
./dummy
./bridge
./host-device
./README.md
./LICENSE
./static
./portmap
./host-local
./firewall
./tuning
./vlan
./ptp
./macvlan
./loopback
./ipvlan
./vrf
./bandwidth
```



The binaries are built statically and should work on any Linux distribution.

#### Step 4: Configuring the `systemd` cgroup driver

To use the `systemd` cgroup driver in `/etc/containerd/config.toml` with `runc`, set

```bash
sudo vim /etc/containerd/config.toml

#for Containerd versions 1.x:
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
  
#for Containerd versions 2.x:
[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc]
  [plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]
    SystemdCgroup = true
```

避坑：

在安装完containerd后，在Redhat系的Linux中，只需以上三行就可以运行，但是Ubuntu系的Linux在只有以上配置的情况下，运行container后，容器非常不稳定，每隔几分钟就会重启，之后排除，将`/etc/containerd/config.toml`补充完整后，容器运行变为稳定。

If you experience container crash loops after the initial cluster installation or after installing a CNI, the containerd configuration provided with the package might contain incompatible configuration parameters. Consider resetting the containerd configuration with `containerd config default > /etc/containerd/config.toml` as specified in [getting-started.md](https://github.com/containerd/containerd/blob/main/docs/getting-started.md#advanced-topics) and then set the configuration parameters specified above accordingly.

If you apply this change, make sure to restart containerd:

```shell
sudo systemctl daemon-reload
sudo systemctl restart containerd
```

## 2.Installing kubeadm on all Nodes.

These instructions are for Kubernetes 1.36.

0. Verify the MAC address and product_uuid are unique for every node 

   ```bash
   #check MAC address and IP address
   ip link
   ip add
   ifconfig -a
   #check the uuid
   sudo cat /sys/class/dmi/id/product_uuid
   ```

1. Add the Kubernetes `yum` repository. The `exclude` parameter in the repository definition ensures that the packages related to Kubernetes are not upgraded upon running `yum update` as there's a special procedure that must be followed for upgrading Kubernetes. Please note that this repository have packages only for Kubernetes 1.36; for other Kubernetes minor versions, you need to change the Kubernetes minor version in the URL to match your desired minor version (you should also check that you are reading the documentation for the version of Kubernetes that you plan to install).

   ```shell
   cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
   [kubernetes]
   name=Kubernetes
   baseurl=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/
   enabled=1
   gpgcheck=1
   gpgkey=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/repodata/repomd.xml.key
   exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
   EOF
   ```

2. Install kubelet, kubeadm and kubectl:

   ```shell
   #verify DNF version
   dnf --version 
   cat /etc/redhat-release 
   
   #For systems with DNF:
   sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
   #For systems with DNF5:
   sudo yum install -y kubelet kubeadm kubectl --setopt=disable_excludes=kubernetes
   
   ```

3. Enable the kubelet service before running kubeadm:

   ```shell
   sudo systemctl enable --now kubelet
   ```

   


## 3.Configuring a cgroup driver on all nodes.

In v1.22 and later, if the user does not set the `cgroupDriver` field under `KubeletConfiguration`, kubeadm defaults it to `systemd`.

## 4.init Kubernetes in first Control-plane node.

```text
sudo kubeadm init \
--apiserver-advertise-address=10.0.0.102 \
--control-plane-endpoint=k8s.imccie.tw  \
--service-cidr=100.1.0.0/16 \
--pod-network-cidr=100.2.0.0/16  \
--upload-certs \
--skip-phases=addon/kube-proxy

  
  
# --skip-phases=addon/kube-proxy 不安装kube-proxy组件，我们使用CNI cilime替代kube-proxy，故初始化时选择不安装kube-proxy
# --upload-certs   标志用来将在所有控制平面实例之间的共享证书上传到集群。然后根据安装提示拷贝 kubeconfig 文件：
# --image-repository string：    这个用于指定从什么位置来拉取镜像（1.13版本才有的），默认值是k8s.gcr.io，
# --kubernetes-version string：  指定kubenets版本号，默认值是stable-1，会导致从https://dl.k8s.io/release/stable-1.txt下载最新的版本号，我们可以将其指定为固定版本来跳过网络请求。
# --apiserver-advertise-address  指明用 Master 的哪个 interface 与 Cluster 的其他节点通信。如果 Master 有多个 interface，建议明确指定，如果不指定，kubeadm 会自动选择有默认网关的 interface。这里的ip为master节点ip，记得更换。
# --pod-network-cidr             指定 Pod 网络的范围。Kubernetes 支持多种网络方案，而且不同网络方案对  –pod-network-cidr有自己的要求，这里设置为100.2.0.0/16 
# --control-plane-endpoint     dlc-aci-k8s-cluster.cisco.com 是映射到该 IP 的自定义 DNS 名称，这里配置hosts映射：10.124.3.11   dlc-aci-k8s-cluster.cisco.com。 这将允许你将 --control-plane-endpoint=dlc-aci-k8s-cluster.cisco.com 传递给 kubeadm init，并将相同的 DNS 名称传递给 kubeadm join。 等待集群部署完成后可以修改 dlc-aci-k8s-cluster.cisco.com 以指向高可用性方案中的负载均衡器的地址10.124.3.10。 注意，如果在初始化是域名不能正常解析，则会初始化失败。
```

【重要提示】kubeadm 不支持将没有 `--control-plane-endpoint` 参数的单个控制平面集群转换为高可用性集群

如果需要重置再初始化，请执行如下两个命令。

```text
kubeadm reset
rm -fr ~/.kube/  /etc/kubernetes/* var/lib/etcd/*
```

for no-root user

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

for root user

```bash
  export KUBECONFIG=/etc/kubernetes/admin.conf
```

Join any number of control-plane nodes running the following command on each as root:

```bash
  sudo kubeadm join k8s.imccie.tw:6443 --token fceiqt.qpdkuhi02dafttfp \
	--discovery-token-ca-cert-hash sha256:bb9a05bff117d3bc048930c542e8222592ff03f3819c6ac8d4fb5a3084c24e10 \
	--control-plane --certificate-key 81e7dae3f9f9b99131a9e3069e09e91823f2977c5ac5922bf5680491a3d0437b
```

Join any number of worker nodes by running the following on each as root:

```bash
sudo kubeadm join k8s.imccie.tw:6443 --token fceiqt.qpdkuhi02dafttfp \
	--discovery-token-ca-cert-hash sha256:bb9a05bff117d3bc048930c542e8222592ff03f3819c6ac8d4fb5a3084c24e10 
```



## 5.other Nodes join the Kubernetes cluster.

### Joining your nodes

The nodes are where your workloads (containers and Pods, etc) run. To add new nodes to your cluster do the following for each machine:

- SSH to the machine

- Become root (e.g. `sudo su -`)

- [Install a runtime](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-runtime) if needed

- Run the command that was output by `kubeadm init`. For example:

  ```bash
  kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>
  ```

If you do not have the token, you can get it by running the following command on the control-plane node:

```bash
kubeadm token list
```

The output is similar to this:

```console
TOKEN                    TTL  EXPIRES              USAGES           DESCRIPTION            EXTRA GROUPS
8ewj1p.9r9hcjoqgajrj4gi  23h  2018-06-12T02:51:28Z authentication,  The default bootstrap  system:
                                                   signing          token generated by     bootstrappers:
                                                                    'kubeadm init'.        kubeadm:
                                                                                           default-node-token
```

By default, tokens expire after 24 hours. If you are joining a node to the cluster after the current token has expired, you can create a new token by running the following command on the control-plane node:

```bash
kubeadm token create
```

The output is similar to this:

```console
5didvk.d09sbcov8ph2amjw
```

## 6.install cilium (only do that on one control node.)



a.Install the helm https://helm.sh/docs/intro/install/

( https://github.com/helm/helm/releases)

```bash
wget https://get.helm.sh/helm-v4.2.0-linux-amd64.tar.gz
sha256sum helm-v4.2.0-linux-amd64.tar.gz
tar -zxvf helm-v4.0.0-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
```



### a.Install the Cilium 

Install the latest version of the Cilium CLI. The Cilium CLI can be used to install Cilium, inspect the state of a Cilium installation, and enable/disable various features (e.g. clustermesh, Hubble).

```bash
helm install cilium oci://quay.io/cilium/charts/cilium \
--version 1.19.4 \
--namespace kube-system \
--set routingMode=native \
--set ipam.mode=kubernetes \
--set ipv4NativeRoutingCIDR=100.2.0.0/16 \
--set autoDirectNodeRoutes=true \
--set loadBalancer.acceleration=native \
--set kubeProxyReplacement=true \
--set hubble.enabled=true \
--set hubble.ui.enabled=true \
--set hubble.relay.enabled=true \
--set hubble.metrics.enabled="{dns,drop,tcp,flow,icmp,http}" \
--set prometheus.enabled=true \
--set operator.prometheus.enabled=true \
--set hubble.metrics.serviceMonitor.enabled=false \
--set bpf.masquerade=true \
--set bpf.hostRouting=true \
--set k8sServiceHost=10.0.0.102 \
--set k8sServicePort=6443
```


这套 Helm 参数已经属于：
bare-metal + kube-proxy replacement + Hubble + Prometheus
的完整生产级配置了。

下面按模块解释每个参数的作用、依赖关系、风险点，以及后续如何访问 Hubble UI 与检查 ServiceMonitor。

---



### 一、整体架构

你的配置实际启用了：

```text id="g8ovrd"
Cilium eBPF Datapath
+ Native Routing
+ kube-proxy replacement
+ Hubble observability
+ Prometheus metrics
```

网络模型大致如下：

```text id="n4q8b0"
Pod
 ↓
eBPF datapath
 ↓
Native Routing (无 VXLAN/Geneve)
 ↓
Direct Node Routing
```

因此性能会非常高。

---

### 二、逐项参数解释

#### 1.

```bash id="5ttaqe"
--set routingMode=native
```

#### 含义

启用：

```text id="4vdc2x"
Native Routing
```

而不是：

* VXLAN
* Geneve Tunnel

#### 效果

Pod 流量：

```text id="h8g9u2"
直接走 Linux routing
```

不封装。

---

## 优点

* 性能最好
* 延迟最低
* MTU 无损耗
* CPU 占用低

---

## 风险

要求：

```text id="p6gukx"
节点之间三层路由必须互通
```

否则：

```text id="7i1o4r"
Pod 无法跨节点通信
```

---

# 2.

```bash id="cjlwmx"
--set ipam.mode=kubernetes
```

## 含义

使用：

```text id="15u8by"
Kubernetes Node CIDR
```

分配 Pod IP。

---

## 即

每个 Node：

```text id="ud6qce"
.spec.podCIDR
```

由：

```text id="mkj6eh"
kube-controller-manager
```

管理。

---

## 优点

简单稳定。

---

# 3.

```bash id="1y5kk6"
--set ipv4NativeRoutingCIDR=100.2.0.0/16
```

## 含义

指定：

```text id="xwwuxn"
整个 Pod CIDR
```

属于：

```text id="fr6c7h"
Native Routing
```

管理范围。

---

## 必须匹配

你的 kubeadm：

```yaml id="8hbgk8"
podSubnet
```

例如：

```yaml id="c4x22g"
--pod-network-cidr=100.2.0.0/16
```

---

## 如果不一致

会出现：

* pod 不通
* 路由错误
* node unreachable

---

# 4.

```bash id="lprcx8"
--set autoDirectNodeRoutes=true
```

## 含义

自动添加：

```text id="y90jqj"
Node 到 Node 的 Pod CIDR 路由
```

---

## 举例

node-1:

```text id="01vjlwm"
100.2.1.0/24
```

node-2:

```text id="t2psh9"
100.2.2.0/24
```

Cilium 自动生成：

```text id="bzbzmb"
ip route add 100.2.2.0/24 via 10.0.0.104
```

---

## 非常重要

这是：

```text id="7fbrl0"
Native Routing 模式核心参数
```

---

# 5.

```bash id="4d8ryc"
--set loadBalancer.acceleration=native
```

## 含义

启用：

```text id="8k8kv7"
XDP acceleration
```

优化 LoadBalancer。

---

## 效果

Service 转发：

```text id="vl3ztg"
直接在 NIC driver 层完成
```

性能极高。

---

## 风险

要求网卡支持：

* XDP
* native mode

RHEL10 + 新内核通常没问题。

---

# 6.

```bash id="t7myyn"
--set kubeProxyReplacement=true
```

## 含义

完全替代：

```text id="1qk3tm"
kube-proxy
```

---

## Service 转发由谁完成

由：

```text id="1txr8u"
eBPF
```

完成。

---

## 优点

* 无 iptables
* 无 ipvs
* 性能极高
* 延迟极低

---

## 风险（你已经遇到了）

节点重启时：

```text id="1pvvf4"
bootstrap deadlock
```

所以：

---

# 7.

```bash id="1dfot6"
--set k8sServiceHost=10.0.0.102
--set k8sServicePort=6443
```

## 作用

强制 Cilium 使用：

```text id="2hkt2g"
真实 APIServer IP
```

而不是：

```text id="lg8z0e"
100.1.0.1
```

---

## 这是你环境最关键参数

否则：

```text id="omqk17"
node reboot 后 cilium 起不来
```

---

# 8.

```bash id="4nmp8y"
--set hubble.enabled=true
```

## 含义

启用：

```text id="h8e7f1"
Hubble observability
```

---

## 功能

可以查看：

* Pod 流量
* DNS
* HTTP
* Drop
* TCP
* ICMP

---

# 9.

```bash id="g1v6e5"
--set hubble.relay.enabled=true
```

## 含义

部署：

```text id="3n8m11"
hubble-relay
```

---

## 作用

聚合：

```text id="b57fkn"
所有 node 的 flow
```

否则 UI 无法工作。

---

# 10.

```bash id="8g0q6w"
--set hubble.ui.enabled=true
```

## 含义

部署：

```text id="mv2v4i"
Hubble Web UI
```

---

# 三、如何访问 Hubble UI

---

# 方法1（推荐）

## Port Forward

执行：

```bash id="o7wpr3"
kubectl port-forward -n kube-system svc/hubble-ui 12000:80
```

浏览器访问：

```text id="55g7uk"
http://localhost:12000
```

---

# 方法2（NodePort）

修改：

```yaml id="zjlwm8"
hubble.ui.service.type=NodePort
```

例如：

```bash id="07q39k"
--set hubble.ui.service.type=NodePort
```

然后：

```bash id="c0d3yd"
kubectl get svc -n kube-system
```

访问：

```text id="2llln2"
http://NODE-IP:NODEPORT
```

---

# 方法3（Ingress）

生产环境推荐：

* NGINX Ingress
* Traefik
* HAProxy

暴露：

```text id="t5uc1j"
hubble-ui
```

---

# 四、Hubble Metrics

---

## 11.

```bash id="l0p7k5"
--set hubble.metrics.enabled="{dns,drop,tcp,flow,icmp,http}"
```

## 含义

启用 flow metrics。

---

## 指标说明

| 指标 | 作用       |
| ---- | ---------- |
| dns  | DNS 查询   |
| drop | 被丢弃流量 |
| tcp  | TCP 状态   |
| flow | 总流量     |
| icmp | ping       |
| http | HTTP 请求  |

---

# Metrics 暴露端口

默认：

```text id="2cgqz7"
9965
```

---

# 五、Prometheus

---

## 12.

```bash id="t61m17"
--set prometheus.enabled=true
```

启用：

```text id="e6zy9j"
cilium-agent metrics
```

---

## 13.

```bash id="0clrm0"
--set operator.prometheus.enabled=true
```

启用：

```text id="vjlwmr"
cilium-operator metrics
```

---

# 六、ServiceMonitor

---

## 14.

```bash id="o3rjyn"
--set hubble.metrics.serviceMonitor.enabled=false
```

## 含义

当前：

```text id="v2htxr"
不会创建 Prometheus Operator 的 ServiceMonitor
```

---

# 七、后续改成 true 后如何检查

你后面会改：

```bash id="gjqjlwm"
--set hubble.metrics.serviceMonitor.enabled=true
```

---

# 升级命令

```bash id="dt16li"
helm upgrade cilium cilium/cilium \
  -n kube-system \
  --reuse-values \
  --set hubble.metrics.serviceMonitor.enabled=true
```

---

# 检查1：确认 ServiceMonitor 创建

```bash id="azjmzn"
kubectl get servicemonitor -A
```

应该看到：

```text id="w7z9od"
cilium-agent
hubble
```

---

# 检查2：查看详情

```bash id="c7czr6"
kubectl describe servicemonitor -n kube-system
```

---

# 检查3：Prometheus 是否发现 Target

如果你使用：

* kube-prometheus-stack

进入：

```text id="fz1qfd"
Prometheus -> Status -> Targets
```

应该看到：

```text id="7wjlwm"
hubble
cilium-agent
```

状态：

```text id="kpm5sr"
UP
```

---

# 检查4：验证 metrics endpoint

```bash id="85c6yb"
kubectl port-forward -n kube-system svc/hubble-metrics 9965:9965
```

然后：

```bash id="94gxzk"
curl localhost:9965/metrics
```

---

# 八、BPF 参数

---

## 15.

```bash id="j5t0xg"
--set bpf.masquerade=true
```

## 含义

使用：

```text id="mu58sj"
eBPF NAT masquerade
```

替代 iptables MASQUERADE。

---

# 16.

```bash id="xjlwm5"
--set bpf.hostRouting=true
```

## 含义

启用：

```text id="k1v8hm"
BPF Host Routing
```

绕过 Linux stack。

---

## 优点

性能极高。

---

## 风险

对：

* kernel
* NIC
* routing

要求较高。

---

# 九、你的配置总体评价

你的配置属于：

```text id="xwqjlwm"
高性能裸金属生产架构
```

特点：

* 无 overlay
* 无 kube-proxy
* eBPF datapath
* Native routing
* XDP acceleration

性能非常强。

---

# 十、我建议增加的参数（重要）

你现在还缺：

---

## 1. 明确 kubeProxyReplacement

新版建议：

```yaml id="hjlwm2"
kubeProxyReplacement: "true"
```

而不是 bool。

---

## 2. 指定 devices

例如：

```yaml id="jlwmw8"
devices: "ens160"
```

否则：

```text id="vjlwm4"
XDP 可能绑定错误网卡
```

---

## 3. 指定 MTU

```yaml id="9t9vqs"
MTU: 1500
```

避免：

```text id="jlwm6o"
jumbo frame 问题
```

---

## 4. 打开 Hubble TLS（生产）

默认 UI 通常未加密。

建议：

```yaml id="kjlwmf"
hubble.tls.enabled=true
```

### b.Install prometheus to monitor cilium

```
helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring
helm upgrade cilium oci://quay.io/cilium/charts/cilium \
--namespace kube-system \
--reuse-values  \
--set hubble.metrics.serviceMonitor.enabled=true
```

### 7.  Install and Enable Loadbance (Use Metallb). (在与Cilium Gateway API 用作Ingress时, Gateway Service从LB获取IP, 此Service并不Select具体的Endpoint, 所以MetalLB 并不会响应此IP的ARP请求,导致流量并不能从外部访问K8S内部)

Download Metallb Yaml to check and back on the K8S.

```bash
wget https://raw.githubusercontent.com/metallb/metallb/v0.16.1/config/manifests/metallb-native.yaml
```

Install and apply to the K8S.

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.16.1/config/manifests/metallb-native.yaml
```

Modify the config file of Metallb.

```yaml
#vim Metallb-IP-pool.yaml

apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.0.0.120-10.0.0.140
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
```

Apply the config file to Metallb.

```bash
kubectl apply -f Metallb-IP-pool.yaml 
```



### 8. Enable LB feature on Cilium and L2 Announced ARP

enable LB-IPAM

``` bash
$ vim cilium_pool.yaml
apiVersion: "cilium.io/v2"
kind: CiliumLoadBalancerIPPool
metadata:
  name: gateway-pool
spec:
  blocks:
#  - cidr: "10.0.10.0/24"
#  - cidr: "2004::0/112"
  - start: "20.0.20.100"
    stop: "20.0.20.200"

$ kubectl apply -f cilium_pool.yaml
```

Verify IP Pool of Cilium.

```bash
$ kubectl get ippools
NAME           DISABLED   CONFLICTING   IPS AVAILABLE   AGE
gateway-pool   false      False         19              15h
$ kubectl describe ippools
Name:         gateway-pool
Namespace:    
Labels:       <none>
Annotations:  <none>
API Version:  cilium.io/v2
Kind:         CiliumLoadBalancerIPPool
Metadata:
  Creation Timestamp:  2026-06-05T17:14:44Z
  Generation:          1
  Resource Version:    4149717
  UID:                 115c3a9f-f893-4386-82c8-5d40acecd740
Spec:
  Blocks:
    Start:   20.0.20.100
    Stop:    20.0.20.200
  Disabled:  false
Status:
  Conditions:
    Last Transition Time:  2026-06-05T17:14:44Z
    Message:               
    Observed Generation:   1
    Reason:                resolved
    Status:                False
    Type:                  cilium.io/PoolConflict
    Last Transition Time:  2026-06-05T17:14:44Z
    Message:               21
    Observed Generation:   1
    Reason:                noreason
    Status:                Unknown
    Type:                  cilium.io/IPsTotal
    Last Transition Time:  2026-06-05T17:14:44Z
    Message:               19
    Observed Generation:   1
    Reason:                noreason
    Status:                Unknown
    Type:                  cilium.io/IPsAvailable
    Last Transition Time:  2026-06-05T17:14:44Z
    Message:               2
    Observed Generation:   1
    Reason:                noreason
    Status:                Unknown
    Type:                  cilium.io/IPsUsed
Events:                    <none>

$ kubectl get ciliumloadbalancerippools.cilium.io 
NAME           DISABLED   CONFLICTING   IPS AVAILABLE   AGE
gateway-pool   false      False         19              15h
```

Enable L2 Announcements on Cilium.

```bash
$ helm upgrade cilium cilium/cilium --version 1.19.4 \
   --namespace kube-system \
   --reuse-values \
   --set l2announcements.enabled=true \
   --set k8sClientRateLimit.qps={QPS} \
   --set k8sClientRateLimit.burst={BURST} \
   --set kubeProxyReplacement=true \
   --set k8sServiceHost=${API_SERVER_IP} \
   --set k8sServicePort=${API_SERVER_PORT}
$ kubectl -n kube-system rollout restart deployment/cilium-operator
$ kubectl -n kube-system rollout restart ds/cilium
```

Apply L2 Announcement polices.

``` bash
$ vim cilium_l2ann.yaml

apiVersion: "cilium.io/v2alpha1"
kind: CiliumL2AnnouncementPolicy
metadata:
  name: policy1
spec:
 # serviceSelector:
 #   matchLabels:
 #     color: blue
  nodeSelector:
    matchExpressions:
      - key: node-role.kubernetes.io/control-plane
        operator: DoesNotExist
  interfaces:
  - ^ens[0-9]+
  externalIPs: true
  loadBalancerIPs: true

$ kubectl apply -f cilium_l2ann.yaml
```

Verify L2 Announcement.

```bash
$ kubectl describe l2announcement
Name:         policy1
Namespace:    
Labels:       <none>
Annotations:  <none>
API Version:  cilium.io/v2alpha1
Kind:         CiliumL2AnnouncementPolicy
Metadata:
  Creation Timestamp:  2026-06-05T17:32:06Z
  Generation:          1
  Resource Version:    4148535
  UID:                 f433b16c-5d63-431d-80c4-b40c8ef8f3d3
Spec:
  External I Ps:  true
  Interfaces:
    ^ens[0-9]+
  Load Balancer I Ps:  true
  Node Selector:
    Match Expressions:
      Key:       node-role.kubernetes.io/control-plane
      Operator:  DoesNotExist
Events:          <none>

$ kubectl -n kube-system exec ds/cilium -- cilium-dbg config --all 
#### Read-only configurations ####
AddressScopeMax                   : 254
AgentNotReadyNodeTaintKey         : node.cilium.io/agent-not-ready
AllocatorListTimeout              : 180000000000
AllowICMPFragNeeded               : true
AllowLocalhost                    : always
AnnotateK8sNode                   : false
AuthMapEntries                    : 524288
AutoCreateCiliumNodeResource      : true
BGPRouterIDAllocationIPPool       : 
BGPRouterIDAllocationMode         : default
BGPSecretsNamespace               : 
BPFCompileDebug                   : 
BPFConntrackAccounting            : false
BPFDistributedLRU                 : false
BPFEventsDefaultBurstLimit        : 0
BPFEventsDefaultRateLimit         : 0
BPFEventsDropEnabled              : true
BPFEventsPolicyVerdictEnabled     : true
BPFEventsTraceEnabled             : true
BPFMapEventBuffers
BPFMapsDynamicSizeRatio           : 0.0025
BPFRoot                           : /sys/fs/bpf
BPFSocketLBHostnsOnly             : false
BootIDFile                        : /proc/sys/kernel/random/boot_id
BpfDir                            : /var/lib/cilium/bpf
BypassIPAvailabilityUponRestore   : false
CGroupRoot                        : /run/cilium/cgroupv2
CTMapEntriesGlobalAny             : 65536
CTMapEntriesGlobalTCP             : 131072
CTMapEntriesTimeoutAny            : 60000000000
CTMapEntriesTimeoutFIN            : 10000000000
CTMapEntriesTimeoutSVCAny         : 60000000000
CTMapEntriesTimeoutSVCTCP         : 8000000000000
CTMapEntriesTimeoutSVCTCPGrace    : 60000000000
CTMapEntriesTimeoutSYN            : 60000000000
CTMapEntriesTimeoutTCP            : 8000000000000
CgroupPathMKE                     : 
CiliumIdentityMaxJitter           : 30000000000
ClockSource                       : 0
ClusterHealthPort                 : 4240
ClusterID                         : 0
ClusterName                       : default
ConfigDir                         : /tmp/cilium/config-map
ConfigFile                        : 
ConnectivityProbeFrequencyRatio   : 0.5
ContainerIPLocalReservedPorts     : auto
CreationTime                      : 2026-06-05T17:40:08.452790664Z
DNSPolicyUnloadOnShutdown         : false
DNSProxyConcurrencyLimit          : 0
DNSProxyEnableTransparentMode     : true
DNSProxyLockCount                 : 131
DNSProxyLockTimeout               : 500000000
DNSProxySocketLingerTimeout       : 10
DatapathMode                      : veth
Debug                             : false
DebugVerbose                      : []
Devices                           : [ens33]
DirectRoutingSkipUnreachable      : false
DisableCiliumEndpointCRD          : false
DisableExternalIPMitigation       : false
DryMode                           : false
EnableAutoDirectRouting           : true
EnableAutoProtectNodePortRange    : true
EnableBGPControlPlane             : false
EnableBGPControlPlaneStatusReport : true
EnableBPFClockProbe               : false
EnableBPFMasquerade               : true
EnableBPFTProxy                   : false
EnableCiliumClusterwideNetworkPolicy: true
EnableCiliumEndpointSlice         : false
EnableCiliumNetworkPolicy         : true
EnableCiliumNodeCRD               : true
EnableEgressGateway               : false
EnableEncryptionStrictModeEgress  : false
EnableEncryptionStrictModeIngress : false
EnableEndpointHealthChecking      : true
EnableEndpointLockdownOnPolicyOverflow: false
EnableEndpointRoutes              : false
EnableEnvoyConfig                 : true
EnableExtendedIPProtocols         : false
EnableHealthChecking              : true
EnableHealthDatapath              : false
EnableHostFirewall                : false
EnableHostLegacyRouting           : false
EnableICMPRules                   : true
EnableIPIPDevices                 : false
EnableIPIPTermination             : false
EnableIPMasqAgent                 : false
EnableIPv4                        : true
EnableIPv4FragmentsTracking       : true
EnableIPv4Masquerade              : true
EnableIPv6                        : false
EnableIPv6FragmentsTracking       : false
EnableIPv6Masquerade              : false
EnableIPv6NDP                     : false
EnableIdentityMark                : true
EnableK8sNetworkPolicy            : true
EnableL2Announcements             : true   >>>>>>>>>>>>>>>>>>>>>>>>>>>>
EnableL7Proxy                     : true
EnableLocalNodeRoute              : true
EnableLocalRedirectPolicy         : false
EnableMKE                         : false
EnableMasqueradeRouteSource       : false
EnableNat46X64Gateway             : false
EnableNodeSelectorLabels          : false
EnableNonDefaultDenyPolicies      : true
EnablePMTUDiscovery               : false
EnablePolicy                      : default
EnableRemoteNodeMasquerade        : false
EnableSCTP                        : false
EnableSRv6                        : false
EnableSocketLBPeer                : true
EnableSocketLBPodConnectionTermination: true
EnableSocketLBTracing             : true
EnableSourceIPVerification        : true
EnableTCX                         : true
EnableTracing                     : false
EnableUnreachableRoutes           : false
EnableVTEP                        : false
EnableXDPPrefilter                : false
EncryptInterface                  : []
EncryptNode                       : false
EncryptionStrictEgressAllowRemoteNodeIdentities: false
EncryptionStrictEgressCIDR        : 
EndpointQueueSize                 : 25
ExcludeLocalAddresses             : <nil>
ExcludeNodeLabelPatterns          : <nil>
ExternalEnvoyProxy                : true
FQDNProxyResponseMaxDelay         : 100000000
FQDNRegexCompileLRUSize           : 1024
FQDNRejectResponse                : refused
FixedIdentityMapping
FixedZoneMapping                  : <nil>
ForceDeviceRequired               : false
FragmentsMapEntries               : 8192
HTTP403Message                    : 
HealthCheckICMPFailureThreshold   : 3
HiveConfig
        LogThreshold              : 100000000
        StartTimeout              : 300000000000
        StopTimeout               : 60000000000
IPAM                              : kubernetes
IPAMCiliumNodeUpdateRate          : 15000000000
IPAMDefaultIPPool                 : default
IPAMMultiPoolPreAllocation
        default                   : 8
IPTracingOptionType               : 0
IPv4NativeRoutingCIDR
        IP                        : 100.2.0.0
        Mask                      : //8AAA==
IPv4NodeAddr                      : auto
IPv4PodSubnets                    : []
IPv4Range                         : auto
IPv4ServiceRange                  : auto
IPv6ClusterAllocCIDR              : f00d::/64
IPv6ClusterAllocCIDRBase          : f00d::
IPv6MCastDevice                   : 
IPv6NAT46x64CIDR                  : 64:ff9b::/96
IPv6NAT46x64CIDRBase              : 64:ff9b::
IPv6NativeRoutingCIDR             : <nil>
IPv6NodeAddr                      : auto
IPv6PodSubnets                    : []
IPv6Range                         : auto
IPv6ServiceRange                  : auto
IdentityAllocationMode            : crd
IdentityChangeGracePeriod         : 5000000000
IdentityRestoreGracePeriod        : 30000000000
InstallIptRules                   : true
InstallNoConntrackIptRules        : false
InstallUplinkRoutesForDelegatedIPAM: false
K8sNamespace                      : kube-system
K8sRequireIPv4PodCIDR             : true
K8sRequireIPv6PodCIDR             : false
K8sSyncTimeout                    : 180000000000
KeepConfig                        : false
KernelHz                          : 1000
L2AnnouncerLeaseDuration          : 15000000000
L2AnnouncerRenewDeadline          : 5000000000
L2AnnouncerRetryPeriod            : 2000000000
LabelPrefixFile                   : 
Labels                            : []
LibDir                            : /var/lib/cilium
LoadBalancerIPIPSockMark          : false
LoadBalancerRSSv4
        IP                        : 
        Mask                      : <nil>
LoadBalancerRSSv4CIDR             : 
LoadBalancerRSSv6
        IP                        : 
        Mask                      : <nil>
LoadBalancerRSSv6CIDR             : 
LocalRouterIPv4                   : 
LocalRouterIPv6                   : 
LogDriver                         : []
LogOpt
LogSystemLoadConfig               : false
MasqueradeInterfaces              : []
MaxConnectedClusters              : 255
MaxControllerInterval             : 0
MonitorAggregation                : medium
MonitorAggregationFlags           : 255
MonitorAggregationInterval        : 5000000000
NATMapEntriesGlobal               : 131072
NeighMapEntriesGlobal             : 131072
NodeLabels                        : []
NodePortAcceleration              : native
NodePortBindProtection            : true
NodePortNat46X64                  : false
PolicyAccounting                  : true
PolicyAuditMode                   : false
PolicyCIDRMatchMode               : []
PolicyDenyResponse                : none
PolicyMapFullReconciliationInterval: 900000000000
PolicyTriggerInterval             : 1000000000
PreAllocateMaps                   : false
ProcFs                            : /host/proc
RestoreState                      : true
ReverseFixedZoneMapping           : <nil>
RouteMetric                       : 0
RoutingMode                       : native
RunDir                            : /var/run/cilium
SRv6EncapMode                     : reduced
ServiceNoBackendResponse          : reject
SizeofCTElement                   : 94
SizeofNATElement                  : 94
SizeofNeighElement                : 24
SizeofSockRevElement              : 52
SocketPath                        : /var/run/cilium/cilium.sock
StateDir                          : /var/run/cilium/state
TCFilterPriority                  : 1
ToFQDNsIdleConnectionGracePeriod  : 0
ToFQDNsMaxDeferredConnectionDeletes: 10000
ToFQDNsMaxIPsPerHost              : 1000
ToFQDNsMinTTL                     : 0
ToFQDNsPreCache                   : 
ToFQDNsProxyPort                  : 0
TracePayloadlen                   : 128
TracePayloadlenOverlay            : 192
VLANBPFBypass                     : []
VtepCidrMask                      : 
k8s-configuration                 : 
k8s-endpoint                      : 
##### Read-write configurations #####
ConntrackAccounting               : Disabled
Debug                             : Disabled
DebugLB                           : Disabled
DropNotification                  : Enabled
MonitorAggregationLevel           : Medium
PolicyAccounting                  : Enabled
PolicyAuditMode                   : Disabled
PolicyTracing                     : Disabled
PolicyVerdictNotification         : Enabled
SourceIPVerification              : Enabled
TraceNotification                 : Enabled
MonitorNumPages                   : 64
PolicyEnforcement                 : default
```

### 9. Use Cilium Gateway API replace Ingress Nginx.

Install CRDs before enable Gateway API.

```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/standard/gateway.networking.k8s.io_gatewayclasses.yaml
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/standard/gateway.networking.k8s.io_gateways.yaml
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/standard/gateway.networking.k8s.io_httproutes.yaml
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/standard/gateway.networking.k8s.io_referencegrants.yaml
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/standard/gateway.networking.k8s.io_grpcroutes.yaml

#add TLSRoute with this snippet.
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/experimental/gateway.networking.k8s.io_tlsroutes.yaml
```

Enable Gateway API  in Cilium.

```bash
$ helm upgrade cilium cilium/cilium --version 1.19.4 \
   --namespace kube-system \
   --reuse-values \
   --set kubeProxyReplacement=true \
   --set gatewayAPI.enabled=true
   
$kubectl -n kube-system rollout restart deployment/cilium-operator
$kubectl -n kube-system rollout restart ds/cilium
```

Verfity Cilium Status and Gateways API.

```bash
$  cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       OK
    \__/       ClusterMesh:        disabled

DaemonSet              cilium                   Desired: 3, Ready: 3/3, Available: 3/3
DaemonSet              cilium-envoy             Desired: 3, Ready: 3/3, Available: 3/3
Deployment             cilium-operator          Desired: 2, Ready: 2/2, Available: 2/2
Deployment             hubble-relay             Desired: 1, Ready: 1/1, Available: 1/1
Deployment             hubble-ui                Desired: 1, Ready: 1/1, Available: 1/1
Containers:            cilium                   Running: 3
                       cilium-envoy             Running: 3
                       cilium-operator          Running: 2
                       clustermesh-apiserver    
                       hubble-relay             Running: 1
                       hubble-ui                Running: 1
Cluster Pods:          14/14 managed by Cilium
Helm chart version:    1.19.4
Image versions         cilium             quay.io/cilium/cilium:v1.19.4@sha256:2eb67991eaa9368ba199c2fac2c573cb0ffdeb79184533344f42fc9a7ff6af3c: 3
                       cilium-envoy       quay.io/cilium/cilium-envoy:v1.36.6-1778235340-b87d1e32f522b33bd51701c6476d199326f01496@sha256:71d4fa0ec45e8d546dbd5604e169dc77fe92be63b799313bff031d00d89762e3: 3
                       cilium-operator    quay.io/cilium/operator-generic:v1.19.4@sha256:1aa2b62735e7d8ab49ee840ae59c346932024c88901579121395c1271b435f71: 2
                       hubble-relay       quay.io/cilium/hubble-relay:v1.19.4@sha256:59af8c0d561e560c2a042e7600a3496bc0367df8fbf868aa68d5834c8ec1a431: 1
                       hubble-ui          quay.io/cilium/hubble-ui-backend:v0.13.5@sha256:fac0c300ae119274edca11fd89b1ad23c788792d8bc4ea2ba631c709e8d3c688: 1
                       hubble-ui          quay.io/cilium/hubble-ui:v0.13.5@sha256:f7d514fc54d784ed6df9d58cf0e97648b143f92b766dd1780ed3fc845bd4c516: 1

$ helm get values cilium -n kube-system -o yaml
autoDirectNodeRoutes: true
bpf:
  hostRouting: true
  masquerade: true
envoy:
  enabled: true  >>>>>>>>>>>>
gatewayAPI:
  enabled: true  >>>>>>>>>>>>
hubble:
  enabled: true
  metrics:
    enabled:
    - dns
    - drop
    - tcp
    - flow
    - icmp
    - http
    serviceMonitor:
      enabled: true
  relay:
    enabled: true
  ui:
    enabled: true
ipam:
  mode: kubernetes
ipv4NativeRoutingCIDR: 100.2.0.0/16
k8sServiceHost: 10.0.0.102
k8sServicePort: 6443
kubeProxyReplacement: true  >>>>>>>>>>>
l2announcements:
  enabled: true   >>>>>>>>>>>>
loadBalancer:
  acceleration: native
operator:
  prometheus:
    enabled: true
prometheus:
  enabled: true
routingMode: native

$ kubectl get gatewayclasses.gateway.networking.k8s.io 
NAME     CONTROLLER                     ACCEPTED   AGE
cilium   io.cilium/gateway-controller   True       40h  <<<<<<<<<<<< True

$ kubectl logs -n kube-system cilium-4r4zj  --tail 10
time=2026-06-05T21:41:48.127683283Z level=info msg="Starting GC of connection tracking" module=agent.datapath.maps.ct-nat-map-gc first=false
time=2026-06-05T21:41:48.164530436Z level=info msg="Conntrack garbage collector interval recalculated" module=agent.datapath.maps.ct-nat-map-gc expectedPrevInterval=1h25m30s actualPrevInterval=1h25m30.030789833s newInterval=2h8m15s deleteRatio=0.035125732421875 adjustedDeleteRatio=0.035125732421875
time=2026-06-05T23:50:03.164856446Z level=info msg="Starting GC of connection tracking" module=agent.datapath.maps.ct-nat-map-gc first=false
time=2026-06-06T00:49:27.16650736Z level=info msg="Fallback node addresses updated" module=agent.datapath.node-address addresses="10.0.0.103 (primary), fe80::250:56ff:fe8b:2d27 (primary)" device=*
time=2026-06-06T01:58:18.215430156Z level=info msg="Starting GC of connection tracking" module=agent.datapath.maps.ct-nat-map-gc first=false
time=2026-06-06T04:06:33.270755753Z level=info msg="Starting GC of connection tracking" module=agent.datapath.maps.ct-nat-map-gc first=false
time=2026-06-06T06:14:48.321348858Z level=info msg="Starting GC of connection tracking" module=agent.datapath.maps.ct-nat-map-gc first=false
time=2026-06-06T06:49:27.166670424Z level=info msg="Fallback node addresses updated" module=agent.datapath.node-address addresses="10.0.0.103 (primary), fe80::250:56ff:fe8b:2d27 (primary)" device=*
time=2026-06-06T08:23:03.371642385Z level=info msg="Starting GC of connection tracking" module=agent.datapath.maps.ct-nat-map-gc first=false
time=2026-06-06T08:43:08.434775821Z level=info msg="Auto-detected local ports to reserve in the container namespace for transparent DNS proxy" module=agent.controlplane.cilium-restapi.config-modification ports=[]

$ kubectl get cm cilium-config -n kube-system -o yaml
apiVersion: v1
data:
  agent-not-ready-taint-key: node.cilium.io/agent-not-ready
  auto-direct-node-routes: "true"
  bpf-distributed-lru: "false"
  bpf-events-drop-enabled: "true"
  bpf-events-policy-verdict-enabled: "true"
  bpf-events-trace-enabled: "true"
  bpf-lb-acceleration: native
  bpf-lb-algorithm-annotation: "false"
  bpf-lb-external-clusterip: "false"
  bpf-lb-map-max: "65536"
  bpf-lb-mode-annotation: "false"
  bpf-lb-sock: "false"
  bpf-lb-source-range-all-types: "false"
  bpf-map-dynamic-size-ratio: "0.0025"
  bpf-policy-map-max: "16384"
  bpf-policy-stats-map-max: "65536"
  bpf-root: /sys/fs/bpf
  cgroup-root: /run/cilium/cgroupv2
  cilium-endpoint-gc-interval: 5m0s
  cluster-id: "0"
  cluster-name: default
  clustermesh-cache-ttl: 0s
  clustermesh-enable-endpoint-sync: "false"
  clustermesh-enable-mcs-api: "false"
  clustermesh-mcs-api-install-crds: "true"
  cni-exclusive: "true"
  cni-log-file: /var/run/cilium/cilium-cni.log
  controller-group-metrics: write-cni-file sync-host-ips sync-lb-maps-with-k8s-services
  custom-cni-conf: "false"
  datapath-mode: veth
  debug: "false"
  default-lb-service-ipam: lbipam
  direct-routing-skip-unreachable: "false"
  dnsproxy-enable-transparent-mode: "true"
  dnsproxy-socket-linger-timeout: "10"
  egress-gateway-reconciliation-trigger-interval: 1s
  enable-auto-protect-node-port-range: "true"
  enable-bpf-clock-probe: "false"
  enable-bpf-masquerade: "true"
  enable-drift-checker: "true"
  enable-dynamic-config: "true"
  enable-endpoint-health-checking: "true"
  enable-endpoint-lockdown-on-policy-overflow: "false"
  enable-envoy-config: "true"            >>>>>>>>>>>>>
  enable-gateway-api: "true"             >>>>>>>>>>
  enable-gateway-api-alpn: "false"
  enable-gateway-api-app-protocol: "false"
  enable-gateway-api-proxy-protocol: "false"
  enable-gateway-api-secrets-sync: "true"
  enable-health-check-loadbalancer-ip: "false"
  enable-health-check-nodeport: "true"
  enable-health-checking: "true"
  enable-hubble: "true"
  enable-hubble-open-metrics: "false"
  enable-ipv4: "true"
  enable-ipv4-big-tcp: "false"
  enable-ipv4-masquerade: "true"
  enable-ipv6: "false"
  enable-ipv6-big-tcp: "false"
  enable-ipv6-masquerade: "true"
  enable-k8s-networkpolicy: "true"
  enable-l2-announcements: "true"
  enable-l2-neigh-discovery: "false"
  enable-l7-proxy: "true"
  enable-lb-ipam: "true"
  enable-masquerade-to-route-source: "false"
  enable-metrics: "true"
  enable-no-service-endpoints-routable: "true"
  enable-node-selector-labels: "false"
  enable-non-default-deny-policies: "true"
  enable-policy: default
  enable-policy-secrets-sync: "true"
  enable-sctp: "false"
  enable-service-topology: "false"
  enable-source-ip-verification: "true"
  enable-tcx: "true"
  enable-vtep: "false"
  enable-well-known-identities: "false"
  enable-xt-socket-fallback: "true"
  envoy-access-log-buffer-size: "4096"
  envoy-base-id: "0"
  envoy-config-retry-interval: 15s
  envoy-keep-cap-netbindservice: "false"
  external-envoy-proxy: "true"
  gateway-api-hostnetwork-enabled: "false"
  gateway-api-hostnetwork-nodelabelselector: ""
  gateway-api-secrets-namespace: cilium-secrets
  gateway-api-service-externaltrafficpolicy: Cluster
  gateway-api-xff-num-trusted-hops: "0"
  health-check-icmp-failure-threshold: "3"
  http-retry-count: "3"
  http-stream-idle-timeout: "300"
  hubble-disable-tls: "false"
  hubble-listen-address: :4244
  hubble-metrics: dns drop tcp flow icmp http
  hubble-metrics-server: :9965
  hubble-metrics-server-enable-tls: "false"
  hubble-network-policy-correlation-enabled: "true"
  hubble-socket-path: /var/run/cilium/hubble.sock
  hubble-tls-cert-file: /var/lib/cilium/tls/hubble/server.crt
  hubble-tls-client-ca-files: /var/lib/cilium/tls/hubble/client-ca.crt
  hubble-tls-key-file: /var/lib/cilium/tls/hubble/server.key
  identity-allocation-mode: crd
  identity-gc-interval: 15m0s
  identity-heartbeat-timeout: 30m0s
  identity-management-mode: agent
  install-no-conntrack-iptables-rules: "false"
  ipam: kubernetes
  ipam-cilium-node-update-rate: 15s
  iptables-random-fully: "false"
  ipv4-native-routing-cidr: 100.2.0.0/16
  k8s-require-ipv4-pod-cidr: "false"
  k8s-require-ipv6-pod-cidr: "false"
  kube-proxy-replacement: "true"
  kube-proxy-replacement-healthz-bind-address: ""
  max-connected-clusters: "255"
  mesh-auth-enabled: "false"
  mesh-auth-gc-interval: 5m0s
  mesh-auth-queue-size: "1024"
  mesh-auth-rotated-identities-queue-size: "1024"
  metrics-sampling-interval: 5m
  monitor-aggregation: medium
  monitor-aggregation-flags: all
  monitor-aggregation-interval: 5s
  nat-map-stats-entries: "32"
  nat-map-stats-interval: 30s
  node-port-bind-protection: "true"
  nodes-gc-interval: 5m0s
  operator-api-serve-addr: 127.0.0.1:9234
  operator-prometheus-serve-addr: :9963
  packetization-layer-pmtud-mode: blackhole
  policy-default-local-cluster: "true"
  policy-deny-response: none
  policy-secrets-namespace: cilium-secrets
  policy-secrets-only-from-secrets-namespace: "true"
  preallocate-bpf-maps: "false"
  procfs: /host/proc
  prometheus-serve-addr: :9962
  proxy-cluster-max-connections: "1024"
  proxy-cluster-max-requests: "1024"
  proxy-connect-timeout: "2"
  proxy-idle-timeout-seconds: "60"
  proxy-initial-fetch-timeout: "30"
  proxy-max-active-downstream-connections: "50000"
  proxy-max-concurrent-retries: "128"
  proxy-max-connection-duration-seconds: "0"
  proxy-max-requests-per-connection: "0"
  proxy-use-original-source-address: "true"
  proxy-xff-num-trusted-hops-egress: "0"
  proxy-xff-num-trusted-hops-ingress: "0"
  remove-cilium-node-taints: "true"
  routing-mode: native
  service-no-backend-response: reject
  set-cilium-is-up-condition: "true"
  set-cilium-node-taints: "true"
  synchronize-k8s-nodes: "true"
  tofqdns-dns-reject-response-code: refused
  tofqdns-enable-dns-compression: "true"
  tofqdns-endpoint-max-ip-per-hostname: "1000"
  tofqdns-idle-connection-grace-period: 0s
  tofqdns-max-deferred-connection-deletes: "10000"
  tofqdns-preallocate-identities: "true"
  tofqdns-proxy-response-max-delay: 100ms
  tunnel-protocol: vxlan
  tunnel-source-port-range: 0-0
  unmanaged-pod-watcher-interval: 15s
  vtep-cidr: ""
  vtep-endpoint: ""
  vtep-mac: ""
  vtep-mask: ""
  write-cni-conf-when-ready: /host/etc/cni/net.d/05-cilium.conflist
kind: ConfigMap
metadata:
  annotations:
    meta.helm.sh/release-name: cilium
    meta.helm.sh/release-namespace: kube-system
  creationTimestamp: "2026-05-15T16:41:22Z"
  labels:
    app.kubernetes.io/managed-by: Helm
  name: cilium-config
  namespace: kube-system
  resourceVersion: "4146348"
  uid: 9c947c4f-a206-40e3-9736-7f5ff1fdb4e8
```



### 10. Test Cilium LB and Cilium Gateway API 

因为想test Https的部分,所以需要先配置生成证书的部分.

Install Cert-Manager use helm.

```bash
$ helm repo add jetstack https://charts.jetstack.io
$ helm repo update
$ helm install cert-manager jetstack/cert-manager   --namespace cert-manager   --create-namespace   --set crds.enabled=true
```

Config Cert-Manager use Cloudfarm DNS.

```bash
$ vim DNS-secure.yaml

apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-secret
  namespace: cert-manager
type: Opaque
stringData:
  api-token: Your Cloudflare API Token (Cloudflare → My Profile → API Tokens)

---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: cloudflare-dns01
spec:
  acme:
    email: admin@test.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-account-key
    solvers:
    - dns01:
        cloudflare:
          apiTokenSecretRef:
            name: cloudflare-api-token-secret
            key: api-token
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-cert
  namespace: kube-system
spec:
  secretName: wildcard-tls
  issuerRef:
    name: cloudflare-dns01
    kind: ClusterIssuer
  dnsNames:
  - "test.com"
  privateKey:
    algorithm: RSA
    size: 4096
    rotationPolicy: Always
  renewBefore: 720h
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: sponsor
  namespace: kube-system
spec:
  secretName: sponsor-tls
  issuerRef:
    name: cloudflare-dns01
    kind: ClusterIssuer
  dnsNames:
  - "sponsor.test.com"
  privateKey:
    algorithm: RSA
    size: 4096
    rotationPolicy: Always
  renewBefore: 720h
  
$ kubectl apply -f DNS-secure.yaml
```

Verify Https cetificate.

```bash
$ kubectl get pods -n cert-manager 
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-5fcb9844ff-dsdd7              1/1     Running   0          41h
cert-manager-cainjector-7969cb9988-wgz5z   1/1     Running   0          41h
cert-manager-webhook-668b969bf7-t6jf8      1/1     Running   0          41h

$ kubectl get clusterissuers.cert-manager.io 
NAME               READY   AGE
cloudflare-dns01   True    41h

$ kubectl get certificates.cert-manager.io -n kube-system
NAME            READY   SECRET         AGE
sponsor         True    sponsor-tls    15h
wildcard-cert   True    wildcard-tls   41h

$ kubectl get secrets sponsor-tls -n kube-system -o yaml 
apiVersion: v1
data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZ
  tls.key: LS0tLS1CRUdJTiBSU0EgUFJ
kind: Secret
metadata:
  annotations:
    cert-manager.io/alt-names: sponsor.imccie.tw
    cert-manager.io/certificate-name: sponsor
    cert-manager.io/common-name: sponsor.imccie.tw
    cert-manager.io/ip-sans: ""
    cert-manager.io/issuer-group: ""
    cert-manager.io/issuer-kind: ClusterIssuer
    cert-manager.io/issuer-name: cloudflare-dns01
    cert-manager.io/uri-sans: ""
  creationTimestamp: "2026-06-05T18:24:48Z"
  labels:
    controller.cert-manager.io/fao: "true"
  name: sponsor-tls
  namespace: kube-system
  resourceVersion: "4158671"
  uid: d4d16546-58a7-4b4e-b078-3810b368f833
type: kubernetes.io/tls

#Export Certificate And Key
$ kubectl get secret sponsor-tls -n kube-system -o jsonpath='{.data.tls.crt}' | base64 -d > tls.crt
$ kubectl get secret sponsor-tls -n kube-system -o jsonpath='{.data.tls.key}' | base64 -d > tls.key

$openssl x509 -in tls.crt -text -noout
```

Deplyment Gateway and HTTPRoute

```bash
$ vim gateway.yaml 

apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
  namespace: kube-system
spec:
  gatewayClassName: cilium
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All

  - name: https
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: wildcard-tls
    allowedRoutes:
      namespaces:
        from: All
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: redirect-http
  namespace: kube-system
spec:
  parentRefs:
  - name: main-gateway
    namespace: kube-system
    sectionName: http

  rules:
  - filters:
    - type: RequestRedirect
      requestRedirect:
        scheme: https
        statusCode: 301

$ vim test_nginx.yaml 

apiVersion: v1
kind: Namespace
metadata:
  name: demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: demo
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80

---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: nginx-route
  namespace: demo
spec:
  parentRefs:
  - name: main-gateway
    namespace: kube-system
    sectionName: https
  hostnames:
  - test.test.com

  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: nginx
      port: 80
      
$ kubectl apply -f gateway.yaml 
$ kubectl apply -f test_nginx.yaml 
```

Verify the website.

```bash
$ arping test.test.com
ARPING 10.0.0.120 from 10.0.0.102 ens33
Unicast reply from 10.0.0.120 [00:50:56:8B:2D:27]  0.832ms
Sent 1 probes (1 broadcast(s))


```



## 7.enable Kubernetes Control-plane HA(on all Control-node).

为 apiserver 提供负载均衡器有很多方法，比如传统的 haproxy+keepalived，或者使用 nginx 代理也可以，这里使用一个比较新颖的工具 kube-vip。

[kube-vip](https://kube-vip.io/) 可以在控制平面节点上提供一个 Kubernetes 原生的 HA 负载均衡，不需要再在外部设置 HAProxy 和 Keepalived 来实现集群的高可用了。d

```
#设置HA的IP，此IP应在DNS中与初始化K8S时使用的--control-plane-endpoint参数对应。
export VIP=192.168.31.10
# 设置网卡名称
export INTERFACE=ens33
#拉取最新版本的kube-vip容器
ctr image pull docker.io/plndr/kube-vip:latest
# 使用下面的容器输出静态Pod资源清单
ctr run --rm --net-host docker.io/plndr/kube-vip:latest vip \
/kube-vip manifest pod \
--interface $INTERFACE \
--vip $VIP \
--controlplane \
--services \
--arp \
--leaderElection | tee  /etc/kubernetes/manifests/kube-vip.yaml
```

上面搭建了3个control-plane 节点的高可用 Kubernetes 集群，接下来验证下高可用是否生效。

首先查看一个 kube-vip 的 Pod 日志：

```
k8s@dlc-aci-k8s01:~$ kubectl logs -f kube-vip-dlc-aci-k8s01 -n kube-system -n kube-system
time="2023-12-13T12:39:03Z" level=info msg="Starting kube-vip.io [v0.6.4]"
time="2023-12-13T12:39:03Z" level=info msg="namespace [kube-system], Mode: [ARP], Features(s): Control Plane:[true], Services:[true]"
time="2023-12-13T12:39:03Z" level=info msg="prometheus HTTP server started"
time="2023-12-13T12:39:03Z" level=info msg="Starting Kube-vip Manager with the ARP engine"
time="2023-12-13T12:39:03Z" level=info msg="beginning services leadership, namespace [kube-system], lock name [plndr-svcs-lock], id [dlc-aci-k8s01]"
I1213 12:39:03.665308       1 leaderelection.go:250] attempting to acquire leader lease kube-system/plndr-svcs-lock...
time="2023-12-13T12:39:03Z" level=info msg="Beginning cluster membership, namespace [kube-system], lock name [plndr-cp-lock], id [dlc-aci-k8s01]"
I1213 12:39:03.665661       1 leaderelection.go:250] attempting to acquire leader lease kube-system/plndr-cp-lock...
E1213 12:39:10.686198       1 leaderelection.go:369] Failed to update lock: etcdserver: request timed out
time="2023-12-13T12:39:10Z" level=info msg="Node [dlc-aci-k8s01] is assuming leadership of the cluster"
```



可以看到 dlc-aci-k8s01现在是我们的 Leader，接下来我们将dlc-aci-k8s01节点关掉，然后观察另外的 kube-vip 的日志变化

```
time="2023-12-15T09:48:35Z" level=info msg="Node [dlc-aci-k8s03] is assuming leadership of the cluster"
time="2023-12-15T09:48:35Z" level=info msg="new leader elected: dlc-aci-k8s03"
```

目前Leader已经切换到dlc-aci-k8s03，而且这个时候集群还是可以正常访问的。（注意在切换过程中，会有2-3秒的中断）

```
k8s@dlc-aci-k8s02:~$ kubectl get pods -A
The connection to the server dlc-aci-k8s-cluster.cisco.com:6443 was refused - did you specify the right host or port?
k8s@dlc-aci-k8s02:~$ kubectl get pods -A
The connection to the server dlc-aci-k8s-cluster.cisco.com:6443 was refused - did you specify the right host or port?
k8s@dlc-aci-k8s02:~$
```

【重要提示】在DLC的lab中，由于MTU的问题，拉取任何镜像都会失败，需要将MTU改到1350或以下，拉取镜像才会成功。