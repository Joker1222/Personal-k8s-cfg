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
$ mount -n -o nolock 10.94.22.240:/data/nfs/ /data/nfs/     # 挂载(注:可以设置你自己的MasterIP和共享路径)
$ df -h                                                     # 查看是否挂载成功
Filesystem                       Size  Used Avail Use% Mounted on
10.94.22.240:/opt/nfs/logs-e...  111G   20G   86G  19% /var/lib/kubelet/...
$ cd /data/nfs/ && touch test.txt                           # 创建文件测试是否共享成功，此时回到master节点执行ls /data/nfs/应该能看到test.txt
```
# 2.安装helm
> 参考:http://www.mydlq.club/article/51/

```bash
# 直接从本git仓库下载即可(其他版本移至官方https://github.com/helm/helm/releases)
$ cd /opt/ && wget https://github.com/Joker1222/Personal-k8s-cfg/raw/main/k8s_efk/helm-v3.5.2-linux-amd64.tar.gz

$ cd /opt/ && tar zxvf helm-v3.5.2-linux-amd64.tar.gz

$ cp linux-amd64/helm /usr/local/bin/       # 安装
$ cp linux-amd64/helm /usr/bin/             # /usr/local/bin如果不行直接安装到/usr/bin
$ helm version                              # 查看helm版本
version.BuildInfo{Version:"v3.2.1", GitCommit:"fe51cd1e31e6a202cba7dead9552a6d418ded79a", GitTreeState:"clean", GoVersion:"go1.13.10"}

# 添加Chart
$ helm repo add  elastic    https://helm.elastic.co
$ helm repo add  gitlab     https://charts.gitlab.io
$ helm repo add  harbor     https://helm.goharbor.io
$ helm repo add  bitnami    https://charts.bitnami.com/bitnami
$ helm repo add  incubator  https://kubernetes-charts-incubator.storage.googleapis.com
$ helm repo add  stable     https://kubernetes-charts.storage.googleapis.com
$ helm repo update
```

# 3.利用helm安装nfs-client-provisioner动态存储卷(StorageClass)
> 参考:
> https://blog.csdn.net/qq_28540443/article/details/106428346
> https://juejin.cn/post/6912071173413011470

```bash
$ kubectl create namespace logs                              # 注:首先创建一个命名空间,后续我们所有的容器都放在这个命名空间下
```

```bash
helm install nfs-storage azure/nfs-client-provisioner \
--set nfs.server=10.94.22.240 \                              # 注:这里可以替换成你的master节点IP
--set nfs.path=/data/nfs/ \                                  # 注:这里可以替换成你的共享路径
--set storageClass.name=nfs-storage \
-namespace logs                                              # 注:这里可以替换成你的命名空间
```

```bash
$ kubectl get po -n logs                                     # 查看pod状态
NAME                                                  READY   STATUS    RESTARTS   AGE
nfs-storage-nfs-client-provisioner-75959887d5-5gc7b   1/1     Running   0          16h

$ kubectl get sc -n logs                                     # 查看storageClass
NAME              PROVISIONER                                        RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
nfs-storage       cluster.local/nfs-storage-nfs-client-provisioner   Delete          Immediate              true                   16h
```

常见问题:https://github.com/kubernetes-retired/external-storage/issues/978
```bash
# 如果nfs-provisioner状态异常,可以desc查看详情
$ kubectl describe po nfs-storage-nfs-client-provisioner-75959887d5-5gc7b -n logs
... 可能会看到类似如下错误
(combined from similar events): MountVolume.SetUp failed for volume "nfs-client-root" : mount failed: exit status 32 
Mounting command: systemd-run Mounting arguments: --description=Kubernetes transient 
mount for /var/lib/kubelet/pods/4509e02c-b2d5-11e8-82cf-0cc47ae2265e/volumes/kubernetes.io~nfs/nfs-client-root --scope -- mount -t nfs 10.8.26.123:/testnfs /var/lib/kubelet/pods/4509e02c-b2d5-11e8-82cf-0cc47ae2265e/volumes/kubernetes.io~nfs/nfs-client-root 
Output: Running scope as unit run-rc8cc3ae4b6b44d15a0ff40788877c617.scope. mount: wrong fs type, bad option, bad superblock on 10.8.26.123:/testnfs, missing codepage or helper program, or other error (for several filesystems (e.g. nfs, cifs) you might need a /sbin/mount.<type> helper program) 
In some cases useful info is found in syslog - try dmesg | tail or so.
...

原因就是node节点没装nfs-common客户端!!!!
```


