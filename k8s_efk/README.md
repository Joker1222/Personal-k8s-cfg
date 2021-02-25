# k8s基于nfs存储搭建EFK日志系统
> Kubernetes 中比较流行的日志收集解决方案是 Elasticsearch、Fluentd 和 Kibana（EFK）技术栈，也是官方现在比较推荐的一种方案。
> 本文使用helm一键安装部署EFK集群，NFS作为动态存储卷

| Environment | Version |
|-|-|
| debian | 4.9.130-2 |
| k8s | v1.17.9 |
| helm | v3.2.1 |
| elasticsearch | v7.11.1 |
| kibana | v7.11.1 |
| flunetd | v3.0.4 |
| nfs | v4 |

本教程k8s集群分布
- master:  IP=10.94.22.240
- node1: IP=10.94.22.54

# 1.在物理机上安装NFS
> 参考:https://www.aikiki.top/hexo/2020/03/17/Debian-10-%E6%90%AD%E5%BB%BA-nfs-%E6%9C%8D%E5%8A%A1%E5%99%A8/

注意: master节点安装NFS-Server,其他全部Node节点务必安装NFS-Common(客户端)，否则storageClass将创建失败！

#### master物理机安装nfs-server
```bash
$ apt-get install nfs-kernel-server                                                           # 安装
$ mkdir -p /data/nfs                                                                          # 创建共享目录
$ echo "/data/nfs  *(rw,no_subtree_check,root_squash,no_all_squash,insecure)" >> /etc/exports # 设置共享权限
$ source /etc/exports                                                                         # 检查配置(可能会报一些奇怪的错,可以考虑忽略)
$ /etc/init.d/nfs-kernel-server restart                                                       # 重启
```

#### node物理机安装nfs-common
```bash
$ apt-get install nfs-common                                # 安装
$ mkdir -p /data/nfs                                        # 创建共享目录
$ mount -n -o nolock 10.94.22.240:/data/nfs/ /data/nfs/     # 挂载
$ df -h                                                     # 查看是否挂载成功
Filesystem                       Size  Used Avail Use% Mounted on
10.94.22.240:/opt/nfs/logs-e...  111G   20G   86G  19% /var/lib/kubelet/...
$ cd /data/nfs/ && touch test.txt                           # 创建文件测试是否共享成功，此时回到master节点执行ls /data/nfs/应该能看到test.txt
```
