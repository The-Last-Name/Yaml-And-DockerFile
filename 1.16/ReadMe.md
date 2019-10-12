# 使用 kubeadm 安装 kubernetes v1.16

#### 配置要求
对于 Kubernetes 初学者，推荐在阿里云采购如下配置：（您也可以使用自己的虚拟机、私有云等您最容易获得的 Linux 环境）
- 3台 2核4G 的ECS（突发性能实例 t5 ecs.t5-c1m2.large或同等配置，单台约 0.4元/小时，停机时不收费）
- Cent OS 7.6

安装后的软件版本为
- Kubernetes v1.16.0
    - calico 3.8.2
    - nginx-ingress 1.5.5
- Docker 18.09.7

#### 安装后的拓扑图如下：

#### 关于二进制安装

鉴于目前已经有比较方便的办法获得 kubernetes 镜像，我将回避 二进制 安装是否更好的争论。本文采用 kubernetes.io 官方推荐的 kubeadm 工具安装 kubernetes 集群。

#### 检查 centos / hostname
```shell
# 在 master 节点和 worker 节点都要执行
cat /etc/redhat-release

# 此处 hostname 的输出将会是该机器在 Kubernetes 集群中的节点名字
# 不能使用 localhost 作为节点的名字
hostname

# 请使用 lscpu 命令，核对 CPU 信息
# Architecture: x86_64    本安装文档不支持 arm 架构
# CPU(s):       2         CPU 内核数量不能低于 2
lscpu
```

#### 操作系统兼容性
|    CentOS 版本  |   本文档是否兼容   |   备注   |
| ---- | ---- | ---- |
|   7.6  |   😄   |   已验证   |
|   7.5  |   😄   |   已验证   |

修改 hostname
如果您需要修改 hostname，可执行如下指令：
```shell
# 修改 hostname
hostnamectl set-hostname your-new-host-name
# 查看修改结果
hostnamectl status
# 设置 hostname 解析
echo "127.0.0.1   $(hostname)" >> /etc/hosts
```

请确保：
- 我的任意节点 centos 版本在兼容列表中
- 我的任意节点 hostname 不是 localhost
- 我的任意节点 CPU 内核数量大于等于 2

#### 安装 docker / kubelet
使用 root 身份在所有节点执行如下代码，以安装软件：
- docker
- nfs-utils
- kubectl / kubeadm / kubelet
```shell
# 在 master 节点和 worker 节点都要执行
curl -sSL https://kuboard.cn/install-script/v1.16.0/install-kubelet.sh | sh
```
#### 初始化 master 节点
- 以 root 身份在 demo-master-a-1 机器上执行
- 初始化 master 节点时，如果因为中间某些步骤的配置出错，想要重新初始化 master 节点，请先执行 kubeadm reset 操作
- POD_SUBNET 所使用的网段不能与 master节点/worker节点 所在的网段重叠。该字段的取值为一个 CIDR 值，如果您对 CIDR 这个概念还不熟悉，请不要修改这个字段的取值 10.100.0.1/20
```shell
# 只在 master 节点执行
# 替换 x.x.x.x 为 master 节点实际 IP（请使用内网 IP）
# export 命令只在当前 shell 会话中有效，开启新的 shell 窗口后，如果要继续安装过程，请重新执行此处的 export 命令
export MASTER_IP=x.x.x.x
# 替换 apiserver.demo 为 您想要的 dnsName (不建议使用 master 的 hostname 作为 APISERVER_NAME)
export APISERVER_NAME=apiserver.demo
# Kubernetes 容器组所在的网段，该网段安装完成后，由 kubernetes 创建，事先并不存在于您的物理网络中
export POD_SUBNET=10.100.0.1/20
echo "${MASTER_IP}    ${APISERVER_NAME}" >> /etc/hosts
curl -sSL https://kuboard.cn/install-script/v1.16.0/init-master.sh | sh
```
检查 master 初始化结果
```shell
# 只在 master 节点执行

# 执行如下命令，等待 3-10 分钟，直到所有的容器组处于 Running 状态
watch kubectl get pod -n kube-system -o wide

# 查看 master 节点初始化结果
kubectl get nodes -o wide
```
#### 初始化 worker节点
##### 获得 join命令参数
在 master 节点上执行
```shell
# 只在 master 节点执行
kubeadm token create --print-join-command
```
可获取kubeadm join 命令及参数，如下所示
```shell
# kubeadm token create 命令的输出
kubeadm join apiserver.demo:6443 --token mpfjma.4vjjg8flqihor4vt     --discovery-token-ca-cert-hash sha256:6f7a8e40a810323672de5eee6f4d19aa2dbdb38411845a1bf5dd63485c43d303
```
初始化worker
针对所有的 worker 节点执行
```shell
# 只在 worker 节点执行
# 替换 ${MASTER_IP} 为 master 节点实际 IP
# 替换 ${APISERVER_NAME} 为初始化 master 节点时所使用的 APISERVER_NAME
echo "${MASTER_IP}    ${APISERVER_NAME}" >> /etc/hosts

# 替换为 master 节点上 kubeadm token create 命令的输出
kubeadm join apiserver.demo:6443 --token mpfjma.4vjjg8flqihor4vt     --discovery-token-ca-cert-hash sha256:6f7a8e40a810323672de5eee6f4d19aa2dbdb38411845a1bf5dd63485c43d303
```
检查初始化结果
在 master 节点上执行
```shell
# 只在 master 节点执行
kubectl get nodes -o wide
```
输出结果如下所示：
```shell
[root@demo-master-a-1 ~]# kubectl get nodes
NAME     STATUS   ROLES    AGE     VERSION
demo-master-a-1   Ready    master   5m3s    v1.16.0
demo-worker-a-1   Ready    <none>   2m26s   v1.16.0
demo-worker-a-2   Ready    <none>   3m56s   v1.16.0
```
移除 worker 节点
正常情况下，您无需移除 worker 节点，如果添加到集群出错，您可以移除 worker 节点，再重新尝试添加

在准备移除的 worker 节点上执行
```shell
# 只在 worker 节点执行
kubeadm reset
```
在 master 节点 demo-master-a-1 上执行
```shell
# 只在 master 节点执行
kubectl delete node demo-worker-x-x
```
- 将 demo-worker-x-x 替换为要移除的 worker 节点的名字
- worker 节点的名字可以通过在节点 demo-master-a-1 上执行 kubectl get nodes 命令获得
#### 安装 Ingress Controller
在 master 节点上执行
```shell
# 只在 master 节点执行
kubectl apply -f https://kuboard.cn/install-script/v1.16.0/nginx-ingress.yaml
```
配置域名解析

将域名 *.demo.yourdomain.com 解析到 demo-worker-a-2 的 IP 地址 z.z.z.z （也可以是 demo-worker-a-1 的地址 y.y.y.y）

验证配置

在浏览器访问 a.demo.yourdomain.com，将得到 404 NotFound 错误页面

如果您打算将 Kubernetes 用于生产环境，请参考此文档 [Installing Ingress Controller]( https://github.com/nginxinc/kubernetes-ingress/blob/v1.5.3/docs/installation.md )，完善 Ingress 的配置

