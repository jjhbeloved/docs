# Docker使用文档

``` bash
# 配置文件目录
/etc/docker/daemon.json
{
      "registry-mirrors": ["http://registry.sensetime.com"],
      "insecure-registries": [""]
}

sudo docker pull gcr.io/fluentd-elasticsearch/fluentd:v2.5.1
```

## 1 导入导出镜像

``` bash
# 导出 image
sudo docker save busybox-1 > /home/save.tar
# 导入　image
docker load < /home/save.tar
docker push localhost:5000/viper-test/engine-image-process-service:v1.4.0-521c39b
# 删除　image
docker rmi busbox-1
# 重命名　image
docker tag ca1b6b825289 registry.cn-hangzhou.aliyuncs.com/xxxxxxx:v1.0
```

## 2　会出现 device or resource busy

Error response from daemon: Driver overlay failed to remove root filesystem 10016968f1165d0dcb941c8d7fb2f852283f4fa742b5d122dec4e3827cd1330a: remove /var/lib/docker/overlay/f2e06edc8f93feb984d3e2ce412e57485b8e4887778ecc77b9018bda378233a6/merged: device or resource busy

``` bash
grep docker /proc/*/mountinfo | grep f2e06edc8f93feb984d3e2ce412e57485b8e4887778ecc77b9018bda378233a6 | awk -F':' '{print $1}' | awk -F'/' '{print $3}'
# 获取进程id　进行kill
```

## 3 路由转发

``` text
内网<--[网关]-->公网
```

1. 目标地址和源地址是通过网关进行替换, 而屏蔽了信息源
2. 替换地址后, 数据包的路由转发(当有 **多于一块网卡** 的时候需要转发)

在 `/etc/sysctl.conf` 文件下, 添加 `net.ipv4.ip_forward = 1`

## 4 基本 container 命令

``` bash
# --rm 在容器结束时, 会删除对应 文件系统数据, 在stop的时候为了方便调试, 是不会删除的
docker run --rm ...
# cp 会从容器内拷贝对应的目录/文件 -> 宿主机制定目录, 若是容器不存在会返回容器/文件不存在
docker cp <container_id>:/tmp/abc /abc
```