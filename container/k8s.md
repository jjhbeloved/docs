# K8S使用文档

## 1 CNI(Container Network Interface)容器网络接口

CNI **仅** 关心容器 **创建时** 的网络分配，和当 **容器被删除时** 释放网络资源

## 2 依赖组建

1. docker
2. etcd | calico | cni
3. nginx
4. kube-proxy | kube-node
5. hyperkube | kubelet
6. br_netfilter
7. kube-apiserver | kube-controller-manager | kube-scheduler

``` bash
# api-server check
curl http://127.0.0.1:8080//healthz
# scheduler check
curl http://localhost:10251/healthz
# controller check
curl http://localhost:10252/healthz
```

## 3. 保证运行

### 3.1 可能存在服务端口被客户端随机端口占用

为了保证 k8s 分配端口是可用的, 需要在 `/etc/sysctl.conf` 文件下加入 `net.ipv4.ip_local_reserved_ports=30000-3276` 保证端口可用

服务监听的端口以逗号分隔全部添加到 **ip_local_reserved_ports** 中，TCP/IP协议栈从 **ip_local_port_range** 中随机选取源端口时，会 **排除** ip_local_reserved_ports中定义的端口，因此就不会出现端口被占用了服务无法启动

### 3.2 保证br_netfilter(透明防火墙)存在

开启 **内核ipv4转发** 需要加载br_netfilter模块

要支持 `ipv4, ipv6, arp`, 需要是设置 `/etc/sysctl.conf` 中内容为

``` bash
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-arptables=1
net.bridge.bridge-nf-call-ip6tables=1
```

使用 `modinfo br_netfilter` 来校验, 使用 `modprobe br_netfilter` 来新增

在网桥设备上加入防火墙功能. 透明防火墙具有部署能力强、隐蔽性好、安全性高的优点

### 3.3 节约时间

使用脚本启用 shell 自动补全功能，可以节省大量输入

``` bash
"{{ bin_dir }}/kubectl completion bash >/etc/bash_completion.d/kubectl.sh"
```

----
[cni参考文档](https://jimmysong.io/kubernetes-handbook/concepts/cni.html)
