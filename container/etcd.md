# etcd 说明

> etcd是kubernetes集群中的一个十分重要的组件，用于保存集群所有的 **网络配置和对象的状态信息**
> kubernetes系统中一共有 **两个服务** 需要用到etcd用来 **协同和存储配置**
>  
> 1. 网络插件flannel、对于其它网络插件也需要用到etcd存储网络的配置信息
> 2. kubernetes本身，包括各种对象的状态和元信息配置

## 1 原理

使用的是 **raft一致性算法** 来实现的

## 2 认证

etcd之间的通信需要走证书认证, 保证安全

## 3 etcdctl 使用

类似于 `systemctl` 功能, 提供控制能力, 具体使用方法看 `etcdctl -h`

``` bash
# 检验 etcd 集群是否健康
etcdctl --peers=https://172.17.0.1:2379 cluster-health | grep -q 'cluster is healthy'

etcd_client_url=https://{{ etcd_access_address }}:2379
etcd_peer_url: "https://{{ etcd_access_address }}:2380"
# 校验是否是接入成员是否存在
etcdctl --no-sync --peers={{ etcd_client_url }} member list | grep -q "172.17.0.1"

# 加入etcd成员
etcdctl --no-sync --peers={{ etcd_client_url }} member add {{ etcd_member_name }} {{ etcd_peer_url }}

# 进入 etcd container
docker exec -it etcd1 /bin/sh
```

----
[Etcd在kubernetes集群中的作用](https://blog.csdn.net/bbwangj/article/details/82866927)
