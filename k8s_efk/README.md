# k8s基于nfs存储搭建EFK日志系统
> Kubernetes 中比较流行的日志收集解决方案是 Elasticsearch、Fluentd 和 Kibana（EFK）技术栈，也是官方现在比较推荐的一种方案。
> 本文使用helm一键安装部署EFK集群，NFS作为动态存储卷

| Environment | Version |
|-|-|
| k8s | v1.17.9 |
| helm | v3.2.1 |
| elasticsearch | v7.11.1 |
| kibana | v7.11.1 |
| flunetd | v3.0.4 |
| nfs | v4 |

# 1.在物理机上安装NFS
> 参考:https://www.aikiki.top/hexo/2020/03/17/Debian-10-%E6%90%AD%E5%BB%BA-nfs-%E6%9C%8D%E5%8A%A1%E5%99%A8/
