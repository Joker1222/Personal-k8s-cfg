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

**常见问题:** https://github.com/kubernetes-retired/external-storage/issues/978
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
**一键卸载(如有需要)**

```bash
$ helm uninstall nfs-storage -n logs
```

# 4.利用helm安装elasticsearch
> 参考:https://blog.csdn.net/qq_28540443/article/details/106428346

#### 下载Charts源并修改配置
```bash
# 直接从本git库中下载charts,如需获取最新版请自行调用helm pull elasticsearch
$ cd /opt && wget https://github.com/Joker1222/Personal-k8s-cfg/raw/main/k8s_efk/elasticsearch.tgz

$ cd /opt && tar zxvf elasticsearch.tgz         # 解压

$ vim /opt/elasticsearch/value.yaml             # 修改配置(只展示必要配置)
...
replicas: 1                                                     # 注:这里控制es节点数量,本教程采用单节点部署,如需多节点请自行修改并保证集群各节点环境相同
minimumMasterNodes: 1                                           # 注:这里控制es节点数量,本教程采用单节点部署,如需多节点请自行修改并保证集群各节点环境相同
clusterHealthCheckParams: "wait_for_status=yellow&timeout=1s"   # 注:这行主要是避免单节点部署时卡死(参考问题:https://github.com/elastic/helm-charts/issues/783)
...

...
volumeClaimTemplate:    
  accessModes: [ "ReadWriteOnce" ]
  storageClassName: "nfs-storage"                               # 注:这里选择我们上面创建好的storageClass动态卷存储即可(注意名称匹配)
  resources:
    requests:
      storage: 100Gi                                            # 注:这里根据自己物理机实际磁盘大小控制容量
...

...
protocol: http
httpPort: 9200                                                  # 注:集群内部接口,一般不用修改
transportPort: 9300

service:
  labels: {}
  labelsHeadless: {}
  type: NodePort                                                # 注:这里采用NodePort方式访问 即NodeIP:Port
  nodePort: "30920"                                             # 注:对外暴露的端口,如果es集群无法访问,有可能是端口冲突了(没有报错信息很坑),请确保端口无冲突！
  annotations: {}
  httpPortName: http
  transportPortName: transport
  loadBalancerIP: ""
  loadBalancerSourceRanges: []
  externalTrafficPolicy: ""
...
```
#### 安装
```bash
$ cd /opt && helm install elasticsearch elasticsearch -n logs   # 可能需要等待一会
```

#### 检查
```bash
# 查看节点所在节点位置
$ kubectl get po -n logs -o wide                                
NAME                                                  READY   STATUS    RESTARTS   AGE    IP             NODE    NOMINATED NODE   READINESS GATES
elasticsearch-master-0                                1/1     Running   0          153m   10.233.90.17   node1   <none>           <none>
nfs-storage-nfs-client-provisioner-75959887d5-5gc7b   1/1     Running   0          16h    10.233.90.16   node1   <none>           <none>

# 查看svc端口
$ kubectl get svc -n logs                                       
NAME                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE
elasticsearch-master            NodePort    10.233.9.3      <none>        9200:30920/TCP,9300:31486/TCP   154m
elasticsearch-master-headless   ClusterIP   None            <none>        9200/TCP,9300/TCP               154m

# 可以看到es在node1节点,对外暴露的端口是30920,我们可以curl测试下能否访问 注:别忘了换成你的node节点IP
$ curl 10.94.22.54:30920                                        
{
  "name" : "elasticsearch-master-0",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "WOkf9zbKSFyX4yuTk4ApRA",
  "version" : {
    "number" : "7.11.1",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "ff17057114c2199c9c1bbecc727003a907c0db7a",
    "build_date" : "2021-02-15T13:44:09.394032Z",
    "build_snapshot" : false,
    "lucene_version" : "8.7.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```
#### 一键卸载(如有需要)
```bash
$ helm uninstall elasticsearch -n logs
```

# 5.利用helm部署flunetd
