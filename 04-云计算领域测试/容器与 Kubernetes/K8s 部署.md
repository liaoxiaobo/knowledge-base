---
tags:
  - "#实战"
  - "#云计算"
  - "#Kubernetes"
  - "#kubeadm"
  - "#环境搭建"
---


# K8s 部署

> 摘要：使用 kubeadm 在 CentOS 7 上部署 Kubernetes v1.16.4 集群（1 Master + 2 Worker），覆盖环境准备、Docker 与 kubelet 安装、控制面初始化、网络插件部署、节点加入及 Dashboard 安装。可作为测试环境搭建和 K8s 部署验证的参考手册。

**适用场景**：搭建 K8s 测试环境、验证容器平台部署流程、为云原生应用测试准备底层集群。

**关键词**：kubeadm、Kubernetes、Docker、flannel、kubelet、控制面、Worker 节点、Dashboard。

---

## 一、环境准备

### 1.1 节点规划

| IP | 角色 | 安装组件 |
|---|---|---|
| 192.168.0.16 | K8s Master | kube-apiserver、kube-scheduler、kube-controller-manager、kube-proxy、docker、flannel、kubelet、etcd |
| 192.168.0.17 | K8s Node | kubelet、kube-proxy、docker、flannel |
| 192.168.0.18 | K8s Node | kubelet、kube-proxy、docker、flannel |

以下 1.2 ~ 1.6 节操作需要在所有节点执行。

### 1.2 设置主机名

```bash
hostnamectl set-hostname test01  # Master
hostnamectl set-hostname test02  # Worker
hostnamectl set-hostname test03  # Worker
```

### 1.3 关闭防火墙及 SELinux

```bash
# 关闭 SELinux
systemctl stop firewalld && systemctl disable firewalld
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux && setenforce 0

# 设置防火墙默认区域为 trusted（或按需开放 6443、10250 等端口）
firewall-cmd --set-default-zone=trusted
firewall-cmd --complete-reload
```

### 1.4 关闭 swap 分区

K8s 默认要求关闭 swap，避免 kubelet 调度异常：

```bash
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### 1.5 内核网络参数调整

开启 IPv4 转发和 bridge 流量转发，确保 CNI 插件与 iptables 代理正常工作：

```bash
sysctl -w net.ipv4.ip_forward=1
sysctl -w net.bridge.bridge-nf-call-iptables=1
sysctl -w net.bridge.bridge-nf-call-ip6tables=1
```

验证：

```bash
cat /proc/sys/net/ipv4/ip_forward
cat /proc/sys/net/bridge/bridge-nf-call-iptables
cat /proc/sys/net/bridge/bridge-nf-call-ip6tables
```

### 1.6 时间同步

```bash
yum install -y ntpdate
ntpdate time.windows.com
```

---

## 二、安装 Docker

```bash
yum install docker -y
systemctl enable docker
systemctl start docker
docker --version
```

---

## 三、安装 Kubernetes

### 3.1 配置 YUM 源

```bash
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

### 3.2 安装 kubeadm、kubelet、kubectl

```bash
yum install -y kubelet-1.16.4 kubeadm-1.16.4 kubectl-1.16.4
systemctl enable kubelet
```

---

## 四、初始化 Master 节点

### 4.1 执行 kubeadm init

由于默认镜像仓库 `k8s.gcr.io` 国内无法访问，这里指定阿里云镜像仓库：

```bash
kubeadm init \
  --apiserver-advertise-address=192.168.0.16 \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.16.4 \
  --service-cidr=10.1.0.0/16 \
  --pod-network-cidr=10.244.0.0/16
```

关键输出包括：

- 控制面组件（apiserver、scheduler、controller-manager、etcd）以 static Pod 形式启动；
  - 生成 `/etc/kubernetes/admin.conf`；
- 生成 bootstrap token 和 `kubeadm join` 命令。

### 4.2 配置 kubectl

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 4.3 安装网络插件（flannel）

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml
```

执行后会创建：

- podsecuritypolicy、clusterrole、clusterrolebinding、serviceaccount、configmap；
- 针对不同架构（amd64、arm64、arm、ppc64le、s390x）的 DaemonSet。

### 4.4 验证控制面状态

```bash
kubectl get po --all-namespaces -o wide
kubectl get no
```

预期结果：所有控制面 Pod 和 kube-flannel Pod 状态为 `Running`，Master 节点状态为 `Ready`。

---

## 五、添加 Worker 节点

在 Worker 节点执行初始化时输出的 join 命令：

```bash
kubeadm join 192.168.0.16:6443 --token 7e169l.60ykxxr8md7sb8ak \
  --discovery-token-ca-cert-hash sha256:c0b04a34ac7bb55fdf5b04a70233c14c15a11341a2bde0ea33b30579d84c0ce4
```

验证节点加入：

```bash
kubectl get no
```

预期结果：test01 为 master，test02、test03 为 worker，所有节点状态 `Ready`。

---

## 六、测试 K8s 集群

部署一个 nginx 应用验证集群网络和服务暴露是否正常：

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get pods,svc
```

验证服务可访问：

```bash
curl -I 10.1.210.90              # ClusterIP
curl -I 192.168.0.18:30580       # NodePort
```

---

## 七、部署 Kubernetes Dashboard

### 7.1 准备证书并部署

```bash
kubectl create secret generic kubernetes-dashboard-certs --from-file=$HOME/certs -n kubernetes-dashboard
kubectl apply -f dashboard-recommended.yaml
```

### 7.2 绑定集群管理员角色

```bash
kubectl delete clusterrolebinding kubernetes-dashboard
kubectl create clusterrolebinding kubernetes-dashboard \
  --clusterrole=cluster-admin \
  --serviceaccount=kubernetes-dashboard:kubernetes-dashboard
```

### 7.3 获取登录 Token

```bash
kubectl get secret -n kubernetes-dashboard
kubectl describe secret kubernetes-dashboard-token-xxx -n kubernetes-dashboard | grep token:
```

通过 `https://<NodeIP>:30001` 访问 Dashboard，输入 token 登录。

---

## 八、测试关注点

在 K8s 部署和验证过程中，测试人员可以关注以下方面：

| 测试维度 | 关注点 |
|---|---|
| 部署正确性 | kubeadm init 是否成功、控制面 Pod 是否全部 Running、节点是否 Ready |
| 网络连通性 | Pod 之间跨节点通信、Service ClusterIP/NodePort 是否可达、DNS 解析是否正常 |
| 组件高可用 | 控制面单点故障时集群是否可继续提供服务（生产环境建议多 Master） |
| 资源限制 | kubelet、Docker、etcd 的 CPU/内存/磁盘 I/O 是否满足测试负载需求 |
| 日志与监控 | 系统组件日志（kube-apiserver、kube-scheduler、etcd）是否有异常 |
| 安全基线 | 防火墙规则、RBAC 权限、Dashboard 访问权限是否合理 |

---

## 九、常见问题与排查

### 9.1 `kubeadm join` 出现 hostname 不可达警告

```text
[WARNING Hostname]: hostname "test02" could not be reached
[WARNING Hostname]: hostname "test02": lookup test02 on 10.16.140.4:53: no such host
```

**处理**：在 `/etc/hosts` 中配置节点主机名与 IP 的映射，或配置内网 DNS。

### 9.2 kube-proxy 默认使用 iptables 而非 ipvs

```text
Flag proxy-mode="" unknown, assuming iptables proxy
Using iptables Proxier.
```

**处理**：如需使用 ipvs，需提前加载 `ip_vs`、`ip_vs_rr`、`ip_vs_wrr`、`ip_vs_sh` 内核模块。

### 9.3 etcd 出现慢请求日志

```text
wal: sync duration of 1.126809415s, expected less than 1s
etcdserver: read-only range request ... took too long ...
```

**处理**：检查 Master 节点磁盘 I/O 性能，etcd 对磁盘延迟敏感，建议使用 SSD。

### 9.4 apiserver TLS 握手错误

```text
http: TLS handshake error from 192.168.0.10:59610: remote error: tls: bad certificate
```

**处理**：检查客户端证书是否过期、CN/O 是否匹配、是否正确配置了 kubeconfig。

### 9.5 `kubectl get cs` 显示 AGE 为 unknown

这是 K8s v1.16+ 的已知问题，不影响实际功能。可通过以下命令获取组件状态：

```bash
kubectl get cs -o=go-template='{{printf "|NAME|STATUS|MESSAGE|\n"}}{{range .items}}{{$name := .metadata.name}}{{range .conditions}}{{printf "|%s|%s|%s|\n" $name .status .message}}{{end}}{{end}}'
```

---

## 参考链接

- [Kubernetes 网络插件官方文档](https://kubernetes.io/docs/concepts/cluster-administration/addons/)
- [kubeadm 安装参考（腾讯云）](https://cloud.tencent.com/developer/article/1509412)
- [Kubernetes 中文社区](https://www.kubernetes.org.cn/6632.html)