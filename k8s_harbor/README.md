# k8s集群拉取私有仓库(Harbor)镜像教程
> Harbor仓库搭建请参考：https://github.com/Joker1222/Harbor-Deploy <br>
> 注:如果harbor服务构建时就指定了默认域名，建议使用域名方式访问harbor，直接使用IP:PORT可能会报错 <br>
> 本教程使用的Harbor仓库IP为10.94.22.240，域名为harbor.123.com

## 0.配置所有节点的Hosts
```bash
$ echo "10.94.22.240 harbor.123.com" >> /etc/hosts #注:按需换成自己harbor的IP地址
```
## 1.配置所有节点docker访问harbor方式为http 
```vim
$ vim /etc/docker/daemon.json              #debian默认安装的Docker的修改方式 
{
  "log-opts": {
    "max-size": "5m",
    "max-file":"3"
  },
  "exec-opts": ["native.cgroupdriver=systemd"],
  "insecure-registries": ["http://harbor.123.com:9010"]   //加入这一行 设置http访问harbor(因为默认docker是https访问的)
}

$ systemctl daemon-reload && systemctl restart docker # 重启Docker
```
## 2.创建k8s Secret秘钥
#### 登录Harbor生成auth令牌
```bash
$ docker login -u username -p password harbor.123.com:9010   
...Login Successful!

$ cat ~/.docker/config.json
{
	"auths": {
		"harbor.123.com:9010": {
			"auth": "xxxxx"    //登陆成功后能查看到这里的令牌 (实际上就是账密字符串进行了一次base64加密)
		}
	},
	"HttpHeaders": {
		"User-Agent": "Docker-Client/19.03.15 (linux)"
	}
}
```
#### base64加密~/.docker/config.json文件，获得k8s的Secret秘钥需要的auth令牌
```bash
$ cat ~/.docker/config.json | base64 -w 0 
ewoJImF1dGhzIjogasdasdasdasdasdsasdzzzzzzzzzzzzzzzzzzzzzxcxcxcxcxcxcxcxcxcQvasdasdadsa==
```

#### 创建Secret
```bash
$ vim secret.yaml #使用配置文件创建secret即可
apiVersion: v1
kind: Secret
metadata:
    name: login  # 自定义Secret名称
type: kubernetes.io/dockerconfigjson
data:
        .dockerconfigjson: ewoJImF1dGhzIjogasdasdasdasdasdsasdzzzzzzzzzzzzzzzzzzzzzxcxcxcxcxcxcxcxcxcQvasdasdadsa==  # 这里使用上面加密后的auth令牌
        
$ kubectl create -f secret.yaml

$ kubectl get secret login #查看信息
```

## 4.创建一个简单地Pod来验证
```vim
$ vim demo_pod.yaml#配置如下
apiVersion: v1
kind: Pod
metadata:
  name: demo_pod
spec:
  containers:
  - name: demo_pod
    image: harbor.123.com:9010/xxx/image  #这里填你自己harbor里面的镜像即可
    imagePullPolicy: Always
  imagePullSecrets:
  - name: login   #这里指定使用哪个秘钥 ，就用刚刚login的即可

$ kubectl create -f demo_pod.yaml

```
