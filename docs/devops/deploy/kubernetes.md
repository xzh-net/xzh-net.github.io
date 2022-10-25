# Kubernetes

## 1. 集群搭建

### 1.1 集群规划

| 主机名称 | IP地址 | 配置 |
| ------- | ------- | ------- |
| k8s-master | 192.168.2.201 | 2CPU，2G内存，20G硬盘 |
| k8s-node1 | 192.168.2.202 | 2CPU，2G内存，20G硬盘 |
| k8s-node2 | 192.168.2.203 | 2CPU，2G内存，20G硬盘 |

### 1.2 基础环境准备(三台机器)

#### 1.2.1 修改主机名

```bash
hostnamectl set-hostname k8s-master
hostnamectl set-hostname k8s-node01 
hostnamectl set-hostname k8s-node02
```

#### 1.2.2 主机名解析

```bash
vim /etc/hosts
# 添加
192.168.2.201 k8s-master
192.168.2.202 k8s-node01 
192.168.2.203 k8s-node02
```

#### 1.2.3 时间同步

```bash
systemctl start chronyd
systemctl enable chronyd
```

#### 1.2.4 关闭防火墙

```bash
systemctl stop firewalld
systemctl disable firewalld
systemctl stop iptables
systemctl disable iptables
```

#### 1.2.5 禁用selinux

```bash
sed -i 's/SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

#### 1.2.6 禁用swap分区

```bash
vi /etc/fstab 
# 注释掉以下字段
/dev/mapper/cl-swap swap swap defaults 0 0
```

#### 1.2.7 修改linux内核参数

1. 添加网桥过滤和地址转发

```bash
vi /etc/sysctl.d/kubernetes.conf
# 添加内容
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
```

```bash
sysctl -p   # 重载配置
```

2. 加载网桥模块

```bash
modprobe br_netfilter       # 加载br_netfilter模块
lsmod | grep br_netfilter   # 查看是否加载
```

#### 1.2.8 配置ipvs功能

1. 安装ipset和ipvsadm

```bash
yum -y install ipset ipvsadm
```

2. 添加需要加载的模块写入脚本文件 

```bash
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
```

3. 为脚本文件添加执行权限

```bash
chmod +x /etc/sysconfig/modules/ipvs.modules
```

4. 执行脚本文件

```bash
/bin/bash /etc/sysconfig/modules/ipvs.modules
```

5. 查看对应的模块是否加载成功

```bash
lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```

### 1.3 安装docker(三台机器)

1. 在线安装

```bash
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum install --setopt=obsoletes=0 docker-ce-18.06.3.ce-3.el7 -y
systemctl start docker  
chkconfig docker on # 开机启动
```

2. 修改镜像源

```bash
vi /etc/docker/daemon.json
```

```conf
{
    "registry-mirrors":["https://docker.mirrors.ustc.edu.cn"],
    "exec-opts":["native.cgroupdriver=systemd"],
    "data-root": "/data/docker"
}
```

3. 启动服务

```bash
sudo systemctl daemon-reload 
sudo systemctl restart docker 
```


### 1.4 安装kubernetes组件(三台机器)

1. 修改镜像源

```bash
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

2. 安装kubeadm、kubelet和kubectl

```bash
yum install --setopt=obsoletes=0 kubeadm-1.17.4-0 kubelet-1.17.4-0 kubectl-1.17.4-0 -y
```

3.  配置kubelet的cgroup

```bash
vi /etc/sysconfig/kubelet
# 添加配置，为了实现docker使用的cgroupdriver与kubelet使用的cgroup的一致性
KUBELET_CGROUP_ARGS="--cgroup-driver=systemd"
KUBE_PROXY_MODE="ipvs"
```

4. 设置kubelet开机自启

```bash
systemctl enable kubelet
```

### 1.5 准备集群镜像(三台机器)

在安装kubernetes集群之前，必须要提前准备好集群需要的镜像，所需镜像可以通过下面命令查看

```bash
kubeadm config images list --kubernetes-version v1.17.4
```

此镜像在kubernetes的仓库中,由于网络原因,无法连接，下面提供了一种替代方案

```bash
images=(
  kube-apiserver:v1.17.4
  kube-controller-manager:v1.17.4
  kube-scheduler:v1.17.4
  kube-proxy:v1.17.4
  pause:3.1
  etcd:3.4.3-0
  coredns:1.6.5
)

for imageName in ${images[@]} ; do
  docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
  docker tag  registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
  docker rmi  registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
done
```


### 1.6 集群初始化

#### 1.6.1 Master节点创建集群

```bash
kubeadm init --kubernetes-version=v1.17.4 --apiserver-advertise-address=192.168.2.201 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 
kubeadm token create --print-join-command   # 查看集群加入命令
```

创建配置文件

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### 1.6.2 Node节点加入集群

```bash
kubeadm join 192.168.2.201:6443 --token jgqcwh.2rmnf494r16285og \
    --discovery-token-ca-cert-hash sha256:8cf337eed08d91bcf574c0602abe1409e0dbec6717f7477ae42f3ab5a866efad
```

Master节点查看集群状态

```bash
kubectl get nodes
```

### 1.7 网络插件安装

kubernetes支持多种网络插件，比如flannel、calico、canal等等，任选一种使用即可，本次选择flannel

下载地址：https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

https://github.com/xzh-net/InstallHelper/blob/main/k8s/flannel/kube-flannel.yml

```bash
kubectl apply -f kube-flannel.yml   # 安装插件
kubectl get pods --all-namespaces -o wide
kubectl get pods -n kube-system -o wide
kubectl get nodes                   # 验证插件是否安装成功
kubectl describe pod coredns-6955765f44-c6fr2 -n kube-system  # 如果容器报错，进行查看
```

### 1.8 服务部署

```bash
# 部署nginx
kubectl create deployment nginx --image=nginx:1.22.1
# 端口暴漏 NodePort表示集群外浏览器访问
kubectl expose deployment nginx --port=80 --type=NodePort
# 查看服务状态
kubectl get pods,service
```

访问地址：http://192.168.2.201:port/


#### 1.3.2 ingress-nginx

[ingress-nginx](/file/k8s/mandatory)

```bash
kubectl apply -f mandatory.yaml
kubectl get pods -n ingress-nginx -o wide
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
    - name: https
      port: 443
      targetPort: 443
      protocol: TCP
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
```

```bash
kubectl apply -f service-nodeport.yaml
kubectl get svc -n ingress-nginx -o wide
```

#### 1.3.3 metrics-server

下载 https://github.com/kubernetes-sigs/metrics-server/releases/tag/v0.3.6

修改metrics-server-deployment.yaml

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metrics-server
  namespace: kube-system
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    k8s-app: metrics-server
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  template:
    metadata:
      name: metrics-server
      labels:
        k8s-app: metrics-server
    spec:
      hostNetwork: true
      serviceAccountName: metrics-server
      volumes:
      # mount in tmp so we can safely use from-scratch images and/or read-only containers
      - name: tmp-dir
        emptyDir: {}
      containers:
      - name: metrics-server
        image: registry.cn-hangzhou.aliyuncs.com/google_containers/metrics-server-amd64:v0.3.6
        imagePullPolicy: Always
        args:
        - --kubelet-insecure-tls
        - --kubelet-preferred-address-types=InternalIP,Hostname,InternalDNS,ExternalDNS,ExternalIP
        volumeMounts:
        - name: tmp-dir
          mountPath: /tmp
```

```bash
mkdir /home/k8s/metrics-server-0.3.6
kubectl apply -f ./             # 安装metrics-server
kubectl api-versions
kubectl describe svc metrics-server -n kube-system
kubectl get pod -n kube-system  # 查看pod运行情况
kubectl top pod -n kube-system  # 查看资源使用情况
kubectl top nodes

kubectl get pod -n kube-system | grep metrics-server
kubectl -n kube-system logs metrics-server-758b8649fc-hb7qc  metrics-server --tail 100 -f
systemctl restart kubelet
```



## 2. 命令

资源管理方式
   - 命令式对象管理：直接使用命令去操作kubernetes资源
      - `kubectl run nginx-pod --image=nginx:1.22.1 --port=80`
   - 命令式对象配置：通过命令配置和配置文件去操作kubernetes资源
      - `kubectl create/patch -f nginx-pod.yaml`
   - 声明式对象配置：通过apply命令和配置文件去操作kubernetes资源
      - `kubectl apply -f nginx-pod.yaml`

| 类型           | 操作对象 | 适用环境 | 优点           | 缺点                             |
| -------------- | -------- | -------- | -------------- | -------------------------------- |
| 命令式对象管理 | 对象     | 测试     | 简单           | 只能操作活动对象，无法审计、跟踪 |
| 命令式对象配置 | 文件     | 开发     | 可以审计、跟踪 | 项目大时，配置文件多，操作麻烦   |
| 声明式对象配置 | 目录     | 开发     | 支持目录操作   | 意外情况下难以调试               |

### 2.1 Kubectl

<table>
	<tr>
	    <th>命令分类</th>
	    <th>命令</th>
		<th>翻译</th>
		<th>命令作用</th>
	</tr>
	<tr>
	    <td rowspan="6">基本命令</td>
	    <td>create</td>
	    <td>创建</td>
		<td>创建一个资源</td>
	</tr>
	<tr>
		<td>edit</td>
	    <td>编辑</td>
		<td>编辑一个资源</td>
	</tr>
	<tr>
		<td>get</td>
	    <td>获取</td>
	    <td>获取一个资源</td>
	</tr>
   <tr>
		<td>patch</td>
	    <td>更新</td>
	    <td>更新一个资源</td>
	</tr>
	<tr>
	    <td>delete</td>
	    <td>删除</td>
		<td>删除一个资源</td>
	</tr>
	<tr>
	    <td>explain</td>
	    <td>解释</td>
		<td>展示资源文档</td>
	</tr>
	<tr>
	    <td rowspan="10">运行和调试</td>
	    <td>run</td>
	    <td>运行</td>
		<td>在集群中运行一个指定的镜像</td>
	</tr>
	<tr>
	    <td>expose</td>
	    <td>暴露</td>
		<td>暴露资源为Service</td>
	</tr>
	<tr>
	    <td>describe</td>
	    <td>描述</td>
		<td>显示资源内部信息</td>
	</tr>
	<tr>
	    <td>logs</td>
	    <td>日志</td>
		<td>输出容器在 pod 中的日志</td>
	</tr>	
	<tr>
	    <td>attach</td>
	    <td>缠绕</td>
		<td>进入运行中的容器</td>
	</tr>	
	<tr>
	    <td>exec</td>
	    <td>执行</td>
		<td>执行容器中的一个命令</td>
	</tr>	
	<tr>
	    <td>cp</td>
	    <td>复制</td>
		<td>在Pod内外复制文件</td>
	</tr>
		<tr>
		<td>rollout</td>
	    <td>首次展示</td>
		<td>管理资源的发布</td>
	</tr>
	<tr>
		<td>scale</td>
	    <td>规模</td>
		<td>扩(缩)容Pod的数量</td>
	</tr>
	<tr>
		<td>autoscale</td>
	    <td>自动调整</td>
		<td>自动调整Pod的数量</td>
	</tr>
	<tr>
		<td rowspan="2">高级命令</td>
	    <td>apply</td>
	    <td>rc</td>
		<td>通过文件对资源进行配置</td>
	</tr>
	<tr>
	    <td>label</td>
	    <td>标签</td>
		<td>更新资源上的标签</td>
	</tr>
	<tr>
		<td rowspan="2">其他命令</td>
	    <td>cluster-info</td>
	    <td>集群信息</td>
		<td>显示集群信息</td>
	</tr>
	<tr>
	    <td>version</td>
	    <td>版本</td>
		<td>显示当前Server和Client的版本</td>
	</tr>
</table>

```bash
kubectl version  
journalctl -u kubelet -f        # 查看日志
kubeadm reset -f                # 重置kubeadm
kubectl delete node k8s-node01  # 删除节点
kubectl api-resources           # api资源
kubectl api-versions            # api版本
kubectl get cs                  # 集群状态
kubectl explain pod             # yaml资源清单
kubectl explain pod.metadata

kubectl taint nodes node1 key=value:effect # 设置污点
kubectl taint nodes node1 key:effect-      # 去除污点
kubectl taint nodes node1 key-             # 去除所有污点
```

### 2.2 Namespace

#### 2.2.1 命令式

```bash
kubectl get namespace               # 查看所有命名空间
kubectl get ns default              # 查看默认命名空间
kubectl get ns default -o yaml      # 查看默认命名空间和yaml格式展示结果
kubectl describe ns default         # 详情
kubectl create ns dev               # 创建命名空间
kubectl delete ns dev               # 删除命名
```

#### 2.2.2 配置方式

```bash
vi ns-dev.yaml
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

```bash
kubectl create -f ns-dev.yaml     # 创建命名空间
kubectl delete -f dev-dev.yaml    # 用资源文件删除命名空间
kubectl delete namespace dev      # 按名字删除命名空间    
```

### 2.3 Pod

#### 2.3.1 参数解释

```yaml
apiVersion: v1     #必选，版本号，例如v1
kind: Pod       　 #必选，资源类型，例如 Pod
metadata:       　 #必选，元数据
  name: string     #必选，Pod名称
  namespace: string  #Pod所属的命名空间,默认为"default"
  labels:       　　  #自定义标签列表
    - name: string      　          
spec:  #必选，Pod中容器的详细定义
  containers:  #必选，Pod中容器列表
  - name: string   #必选，容器名称
    image: string  #必选，容器的镜像名称
    imagePullPolicy: [ Always|Never|IfNotPresent ]  #获取镜像的策略 
    command: [string]   #容器的启动命令列表，如不指定，使用打包时使用的启动命令
    args: [string]      #容器的启动命令参数列表
    workingDir: string  #容器的工作目录
    volumeMounts:       #挂载到容器内部的存储卷配置
    - name: string      #引用pod定义的共享存储卷的名称，需用volumes[]部分定义的的卷名
      mountPath: string #存储卷在容器内mount的绝对路径，应少于512字符
      readOnly: boolean #是否为只读模式
    ports: #需要暴露的端口库号列表
    - name: string        #端口的名称
      containerPort: int  #容器需要监听的端口号
      hostPort: int       #容器所在主机需要监听的端口号，默认与Container相同
      protocol: string    #端口协议，支持TCP和UDP，默认TCP
    env:   #容器运行前需设置的环境变量列表
    - name: string  #环境变量名称
      value: string #环境变量的值
    resources: #资源限制和请求的设置
      limits:  #资源限制的设置
        cpu: string     #Cpu的限制，单位为core数，将用于docker run --cpu-shares参数
        memory: string  #内存限制，单位可以为Mib/Gib，将用于docker run --memory参数
      requests: #资源请求的设置
        cpu: string    #Cpu请求，容器启动的初始可用数量
        memory: string #内存请求,容器启动的初始可用数量
    lifecycle: #生命周期钩子
      postStart: #容器启动后立即执行此钩子,如果执行失败,会根据重启策略进行重启
      preStop: #容器终止前执行此钩子,无论结果如何,容器都会终止
    livenessProbe:  #对Pod内各容器健康检查的设置，当探测无响应几次后将自动重启该容器
      exec:       　 #对Pod容器内检查方式设置为exec方式
        command: [string]  #exec方式需要制定的命令或脚本
      httpGet:       #对Pod内个容器健康检查方法设置为HttpGet，需要制定Path、port
        path: string
        port: number
        host: string
        scheme: string
        HttpHeaders:
        - name: string
          value: string
      tcpSocket:     #对Pod内个容器健康检查方式设置为tcpSocket方式
         port: number
       initialDelaySeconds: 0       #容器启动完成后首次探测的时间，单位为秒
       timeoutSeconds: 0    　　    #对容器健康检查探测等待响应的超时时间，单位秒，默认1秒
       periodSeconds: 0     　　    #对容器监控检查的定期探测时间设置，单位秒，默认10秒一次
       successThreshold: 0
       failureThreshold: 0
       securityContext:
         privileged: false
  restartPolicy: [Always | Never | OnFailure]  #Pod的重启策略
  nodeName: <string> #设置NodeName表示将该Pod调度到指定到名称的node节点上
  nodeSelector: obeject #设置NodeSelector表示将该Pod调度到包含这个label的node上
  imagePullSecrets: #Pull镜像时使用的secret名称，以key：secretkey格式指定
  - name: string
  hostNetwork: false   #是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络
  volumes:   #在该pod上定义共享存储卷列表
  - name: string    #共享存储卷名称 （volumes类型有很多种）
    emptyDir: {}       #类型为emtyDir的存储卷，与Pod同生命周期的一个临时目录。为空值
    hostPath: string   #类型为hostPath的存储卷，表示挂载Pod所在宿主机的目录
      path: string      　　        #Pod所在宿主机的目录，将被用于同期中mount的目录
    secret:            #类型为secret的存储卷，挂载集群与定义的secret对象到容器内部
      scretname: string  
      items:     
      - key: string
        path: string
    configMap:         #类型为configMap的存储卷，挂载预定义的configMap对象到容器内部
      name: string
      items:
      - key: string
        path: string
```

#### 2.3.2 命令式

```bash
kubectl get pods --all-namespaces -o wide                     # 查看所有pods
kubectl run nginx-pod --image=nginx:1.22.1 --port=80 -n dev   # 指定命名空间创建pod
kubectl get pod -n dev                                        # 按namespaces查看pods
kubectl get pods nginx-pod-6fddd965c8-4srb4 -o wide -n dev    # 查看指定pod网络信息
kubectl describe pod nginx-pod-6fddd965c8-4srb4 -n dev        # 查看指定pod具体信息
kubectl get pod nginx-pod-6fddd965c8-4srb4 -n dev -o yaml     # 查看资源文件
kubectl exec -it nginx-pod-6fddd965c8-4srb4 -n dev /bin/sh    # 进入容器
kubectl delete deploy nginx-pod -n dev                        # 删除pod控制器
```

#### 2.3.3 启动命令

```bash
vi pod-command.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-command
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.22.1
    imagePullPolicy: IfNotPresent # 用于设置镜像拉取策略
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","touch /tmp/hello.txt;while true;do /bin/echo $(date +%T) >> /tmp/hello.txt; sleep 3; done;"]
```

```bash
kubectl create -f pod-command.yaml    # 创建pod
kubectl get pods pod-command -n dev   # 查看Pod详情
kubectl exec pod-command -n dev -it -c busybox /bin/sh   # 进入容器
tail -f /tmp/hello.txt
```

```lua
特别说明：
    通过上面发现command已经可以完成启动命令和传递参数的功能，为什么这里还要提供一个args选项，用于传递参数呢?这其实跟docker有点关系，kubernetes中的command、args两项其实是实现覆盖Dockerfile中ENTRYPOINT的功能。
 1 如果command和args均没有写，那么用Dockerfile的配置。
 2 如果command写了，但args没有写，那么Dockerfile默认的配置会被忽略，执行输入的command
 3 如果command没写，但args写了，那么Dockerfile中配置的ENTRYPOINT的命令会被执行，使用当前args的参数
 4 如果command和args都写了，那么Dockerfile的配置被忽略，执行command并追加上args参数
```

#### 2.3.4 环境变量

```bash
vi pod-env.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-env
  namespace: dev
spec:
  containers:
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","while true;do /bin/echo $(date +%T);sleep 60; done;"]
    env: # 设置环境变量列表
    - name: "username"
      value: "admin"
    - name: "password"
      value: "123456"
```

```bash
kubectl create -f pod-env.yaml
kubectl exec pod-env -n dev -c busybox -it /bin/sh
echo $username
echo $password
```


#### 2.3.5 端口设置

```
vi pod-ports.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-ports
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.22.1
    ports: # 设置容器暴露的端口列表
    - name: nginx-port
      containerPort: 80
      protocol: TCP
```

```bash
kubectl create -f pod-ports.yaml
kubectl get pod -n dev
kubectl get pod pod-ports -n dev -o yaml
```

#### 2.3.6 资源配额

```bash
vi pod-resources.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-resources
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.22.1
    resources: # 资源配额
      limits:  # 限制资源（上限）
        cpu: "2" # CPU限制，单位是core数
        memory: "10Gi" # 内存限制
      requests: # 请求资源（下限）
        cpu: "1"  # CPU限制，单位是core数
        memory: "10Mi"  # 内存限制
```

```bash
kubectl create -f pod-resources.yaml
kubectl get pod pod-resources -n dev
```

### 2.4 Lable

#### 2.4.1 命令方式

```bash
kubectl label pod nginx version=1.0 -n dev                # 打标签
kubectl label pod nginx version=2.0 -n dev --overwrite    # 更新标签
kubectl get pod nginx  -n dev --show-labels               # 查看标签
kubectl get pod -n dev -l version=2.0  --show-labels      # 筛选标签
kubectl label pod nginx-pod version- -n dev               # 删除标签
```

#### 2.4.2 配置方式

```bash
vi pod-nginx.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: dev
  labels:
    version: "3.0" 
    env: "test"
spec:
  containers:
  - image: nginx:1.22.1
    name: pod
    ports:
    - name: nginx-port
      containerPort: 80
      protocol: TCP
```

```bash
kubectl apply -f pod-nginx.yaml
kubectl get pod nginx  -n dev --show-labels
```


### 2.5 Deployment

#### 2.5.1 命令方式

```bash
kubectl run nginx --image=nginx:1.22.1 --port=80 --replicas=3 -n dev
kubectl get pods -n dev                 # 查看创建的Pod
kubectl get deploy -n dev               # 查看deployment的信息
kubectl describe deploy nginx -n dev    # 查看deployment的详细信息
kubectl delete deploy nginx -n dev      # 删除控制器
```

#### 2.5.2 配置方式

```bash
vi deploy-nginx.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      run: nginx
  template:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - image: nginx:1.22.1
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
```

```bash
kubectl create -f deploy-nginx.yaml
kubectl delete -f deploy-nginx.yaml
```

### 2.6 Service

Service可以看作是一组同类Pod`对外的访问接口`。借助Service，应用可以方便地实现服务发现和负载均衡。

#### 2.6.1 命令方式

1. 创建集群内部可访问的Service

```bash
kubectl expose deploy nginx --name=svc-nginx1 --type=ClusterIP --port=80 --target-port=80 -n dev  # 暴漏service
kubectl get svc svc-nginx1 -n dev -o wide        # 查看service后通过curl访问service暴漏的地址，生命周期中这个地址是不会变
```

2. 创建集群外部也可访问的Service

```bash
kubectl expose deploy nginx --name=svc-nginx2 --type=NodePort --port=80 --target-port=80 -n dev
kubectl get svc -n dev -o wide
kubectl delete svc svc-nginx2 -n dev  # 删除service
```

!> k8s访问svc的clusterip异常的慢

Checksum Offload 是网卡的一个功能选项。如果该选项开启，则网卡层面会计算需要发送或者接收到的消息的校验和，从而节省 CPU 的计算开销。此时，在需要发送的消息到达网卡前，系统会在报头的校验和字段填充一个随机值。但是，尽管校验和卸载能够降低 CPU 的计算开销，但受到计算能力的限制，某些环境下的一些网络卡计算速度不如主频超过 400MHz 的 CPU 快

```bash
ethtool -K flannel.1 tx-checksum-ip-generic off
```

#### 2.6.2 配置方式

```bash
vi svc-nginx.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-nginx
  namespace: dev
spec:
  clusterIP: 10.109.179.231
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: nginx
  type: ClusterIP
```

```bash
kubectl create -f svc-nginx.yaml
kubectl delete -f svc-nginx.yaml
```

### 2.6 Ingress

#### 2.6.1 创建应用

例子tomcat-nginx.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-deployment
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tomcat-pod
  template:
    metadata:
      labels:
        app: tomcat-pod
    spec:
      containers:
      - name: tomcat
        image: tomcat:8.5-jre10-slim
        ports:
        - containerPort: 8080

---

apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: dev
spec:
  selector:
    app: nginx-pod
  clusterIP: None
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
  namespace: dev
spec:
  selector:
    app: tomcat-pod
  clusterIP: None
  type: ClusterIP
  ports:
  - port: 8080
    targetPort: 8080
```

```bash
kubectl delete ns dev
kubectl create ns dev
kubectl apply -f tomcat-nginx.yaml   # 创建
kubectl get svc -n dev
```

#### 2.6.2 Http代理

创建ingress-http.yaml

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-http
  namespace: dev
spec:
  rules:
  - host: nginx.xuzhihao.net
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-service
          servicePort: 80
  - host: tomcat.xuzhihao.net
    http:
      paths:
      - path: /
        backend:
          serviceName: tomcat-service
          servicePort: 8080
```

```bash
kubectl apply -f ingress-http.yaml
kubectl get ing ingress-http -n dev
kubectl describe ing ingress-http  -n dev
kubectl get svc -n ingress-nginx # 查看端口

#配置host
192.168.3.200 nginx.xuzhihao.net
192.168.3.200 tomcat.xuzhihao.net
```

#### 2.6.3 Https代理

```bash
# 生成证书
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/C=CN/ST=BJ/L=BJ/O=nginx/CN=xuzhihao.net"
# 创建密钥
kubectl create secret tls tls-secret --key tls.key --cert tls.crt
```

创建ingress-https.yaml

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-https
  namespace: dev
spec:
  tls:
    - hosts:
      - nginx.xuzhihao.net
      - tomcat.xuzhihao.net
      secretName: tls-secret # 指定秘钥
  rules:
  - host: nginx.xuzhihao.net
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-service
          servicePort: 80
  - host: tomcat.xuzhihao.net
    http:
      paths:
      - path: /
        backend:
          serviceName: tomcat-service
          servicePort: 8080
```

```bash
kubectl apply -f ingress-https.yaml
kubectl get ing ingress-https -n dev
kubectl describe ing ingress-https -n dev
```

## 3. 数据存储

### 3.1 基本存储

#### 3.1.1 EmptyDir

EmptyDir是最基础的Volume类型，一个EmptyDir就是Host上的一个空目录，EmptyDir是在Pod被分配到Node时创建的，它的初始内容为空，并且无须指定宿主机上对应的目录文件，因为kubernetes会自动分配一个目录，当Pod销毁时， EmptyDir中的数据也会被永久删除

创建一个volume-emptydir.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-emptydir
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - containerPort: 80
    volumeMounts:  # 将logs-volume挂在到nginx容器中，对应的目录为 /var/log/nginx
    - name: logs-volume
      mountPath: /var/log/nginx
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","tail -f /logs/access.log"] # 初始命令，动态读取指定文件中内容
    volumeMounts:  # 将logs-volume 挂在到busybox容器中，对应的目录为 /logs
    - name: logs-volume
      mountPath: /logs
  volumes: # 声明volume， name为logs-volume，类型为emptyDir
  - name: logs-volume
    emptyDir: {}
```

```bash
kubectl create -f volume-emptydir.yaml
kubectl get pods volume-emptydir -n dev -o wide
# curl 192.168.3.200
kubectl logs -f volume-emptydir -n dev -c busybox # # 通过kubectl logs命令查看指定容器的标准输出
```

#### 3.1.2 HostPath

HostPath就是将Node主机中一个实际目录挂在到Pod中，以供容器使用，这样的设计就可以保证Pod销毁了，但是数据依据可以存在于Node主机上

创建一个volume-hostpath.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-hostpath
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - containerPort: 80
    volumeMounts:
    - name: logs-volume
      mountPath: /var/log/nginx
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","tail -f /logs/access.log"]
    volumeMounts:
    - name: logs-volume
      mountPath: /logs
  volumes:
  - name: logs-volume
    hostPath: 
      path: /root/logs
      type: DirectoryOrCreate  # 目录存在就使用，不存在就先创建后使用
```

```
关于type的值的一点说明：
	DirectoryOrCreate 目录存在就使用，不存在就先创建后使用
	Directory	目录必须存在
	FileOrCreate  文件存在就使用，不存在就先创建后使用
	File 文件必须存在	
  Socket	unix套接字必须存在
	CharDevice	字符设备必须存在
	BlockDevice 块设备必须存在
```

```bash
kubectl create -f volume-hostpath.yaml
kubectl get pods volume-hostpath -n dev -o wide
# curl 192.168.3.200
ls /root/logs/  # 到Pod所在的节点运行
```


#### 3.1.3 NFS

HostPath可以解决数据持久化的问题，但是一旦Node节点故障了，Pod如果转移到了别的节点，又会出现问题了，此时需要准备单独的网络存储系统，比较常用的用NFS、CIFS

```bash
yum install nfs-utils -y   # master上安装nfs服务
mkdir /root/data/nfs -pv   # 创建共享目录
vi /etc/exports
# 将共享目录以读写权限暴露给192.168.3.0/24网段中的所有主机
/root/data/nfs     192.168.3.0/24(rw,no_root_squash)
systemctl start nfs
showmount -e 192.168.3.200 # 查看NFS共享目录 
# 在node上安装nfs服务，注意不需要启动
```

创建volume-nfs.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-nfs
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - containerPort: 80
    volumeMounts:
    - name: logs-volume
      mountPath: /var/log/nginx
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","tail -f /logs/access.log"] 
    volumeMounts:
    - name: logs-volume
      mountPath: /logs
  volumes:
  - name: logs-volume
    nfs:
      server: 192.168.3.200  #nfs服务器地址
      path: /root/data/nfs #共享文件路径
```

```bash
kubectl create -f volume-nfs.yaml
kubectl get pods volume-nfs -n dev -o wide
ls /root/data/nfs
```

### 3.2 高级存储

- 存储：存储工程师维护
- PV：  kubernetes管理员维护
- PVC：kubernetes用户维护

#### 3.2.1 PV

资源清单
```yaml
apiVersion: v1  
kind: PersistentVolume
metadata:
  name: pv2
spec:
  nfs: # 存储类型，与底层真正存储对应
  capacity:  # 存储能力，目前只支持存储空间的设置
    storage: 2Gi
  accessModes:  # 访问模式
  storageClassName: # 存储类别
  persistentVolumeReclaimPolicy: # 回收策略
```

使用NFS作为存储，创建3个PV，对应NFS中的3个暴露的路径。

```bash
mkdir /root/data/{pv1,pv2,pv3} -pv
vi /etc/exports
# 将共享目录以读写权限暴露给192.168.3.0/24网段中的所有主机
/root/data/pv1     192.168.3.0/24(rw,no_root_squash)
/root/data/pv2     192.168.3.0/24(rw,no_root_squash)
/root/data/pv3     192.168.3.0/24(rw,no_root_squash)
systemctl restart nfs
```

创建pv.yaml
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name:  pv1
spec:
  capacity: 
    storage: 1Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /root/data/pv1
    server: 192.168.3.200

---

apiVersion: v1
kind: PersistentVolume
metadata:
  name:  pv2
spec:
  capacity: 
    storage: 2Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /root/data/pv2
    server: 192.168.3.200
    
---

apiVersion: v1
kind: PersistentVolume
metadata:
  name:  pv3
spec:
  capacity: 
    storage: 3Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /root/data/pv3
    server: 192.168.3.200
```

```bash
kubectl create -f pv.yaml
kubectl get pv -o wide
```

#### 3.2.2 PVC

资源清单
```yaml
apiVersion: v1  
kind: PersistentVolume
metadata:
  name: pv2
spec:
  nfs: # 存储类型，与底层真正存储对应
  capacity:  # 存储能力，目前只支持存储空间的设置
    storage: 2Gi
  accessModes:  # 访问模式
  storageClassName: # 存储类别
  persistentVolumeReclaimPolicy: # 回收策略
```

创建pvc.yaml，申请pv
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc1
  namespace: dev
spec:
  accessModes: 
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
      
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc2
  namespace: dev
spec:
  accessModes: 
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
     
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc3
  namespace: dev
spec:
  accessModes: 
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl create -f pvc.yaml
kubectl get pvc  -n dev -o wide  # pvc和pv有一个自动寻找合适空间绑定的过程
```

创建pods.yaml, 使用pv

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
  namespace: dev
spec:
  containers:
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","while true;do echo pod1 >> /root/out.txt; sleep 10; done;"]
    volumeMounts:
    - name: volume
      mountPath: /root/
  volumes:
    - name: volume
      persistentVolumeClaim:
        claimName: pvc1
        readOnly: false
---
apiVersion: v1
kind: Pod
metadata:
  name: pod2
  namespace: dev
spec:
  containers:
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","while true;do echo pod2 >> /root/out.txt; sleep 10; done;"]
    volumeMounts:
    - name: volume
      mountPath: /root/
  volumes:
    - name: volume
      persistentVolumeClaim:
        claimName: pvc2
        readOnly: false  
```

```bash
kubectl create -f pods.yaml     # 创建pod
kubectl get pods -n dev -o wide # 查看pod
kubectl get pvc -n dev -o wide  # 查看pvc
kubectl get pv -n dev -o wide   # 查看pv
more /root/data/pv1             # 查看nfs中的文件存储
```

### 3.3 配置存储

#### 3.3.1 ConfigMap

ConfigMap是一种比较特殊的存储卷，它的主要作用是用来存储配置信息的

创建configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: configmap
  namespace: dev
data:
  info: |
    username:admin
    password:123456
```

```bash
kubectl create -f configmap.yaml
kubectl describe cm configmap -n dev
```

创建一个pod-configmap.yaml，将上面创建的configmap挂载进去

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    volumeMounts: # 将configmap挂载到目录
    - name: config
      mountPath: /configmap/config
  volumes: # 引用configmap
  - name: config
    configMap:
      name: configmap
```

```bash
kubectl create -f pod-configmap.yaml
kubectl get pod pod-configmap -n dev
kubectl exec -it pod-configmap -n dev /bin/sh
cd /configmap/config/
ls
```

#### 3.3.2 Secret

一种和ConfigMap非常类似的对象，称为Secret对象。它主要用于存储敏感信息，例如密码、秘钥、证书等等。

```bash
echo -n 'admin' | base64 #准备username
echo -n '123456' | base64 #准备password
```

创建secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret
  namespace: dev
type: Opaque
data:
  username: YWRtaW4=
  password: MTIzNDU2
```

```bash
kubectl create -f secret.yaml
kubectl describe secret secret -n dev
```

创建pod-secret.yaml，将上面创建的secret挂载进去

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    volumeMounts: # 将secret挂载到目录
    - name: config
      mountPath: /secret/config
  volumes:
  - name: config
    secret:
      secretName: secret
```

```bash
kubectl create -f pod-secret.yaml
kubectl get pod pod-secret -n dev
kubectl exec -it pod-secret /bin/sh -n dev
ls /secret/config/
more /secret/config/username
```

## 5. DashBoard

### 5.1 部署

下载文件

```bash
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
```

修改kubernetes-dashboard的Service类型

```bash
vi recommended.yaml
```

```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort  # 新增
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30009  # 新增
  selector:
    k8s-app: kubernetes-dashboard
```

```bash
kubectl create -f recommended.yaml
kubectl get pod,svc -n kubernetes-dashboard  # 查看资源
```

### 5.2 初始化账号

```bash
kubectl create serviceaccount dashboard-admin -n kubernetes-dashboard  # 创建账号
kubectl create clusterrolebinding dashboard-admin-rb --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:dashboard-admin # 授权
kubectl get secrets -n kubernetes-dashboard | grep dashboard-admin  # 获取账号token
kubectl describe secrets [secretsname] -n kubernetes-dashboard      # 查看token
```

### 5.3 证书更换（可选）

```bash
cd /opt/k8s
mkdir key && cd key
#生成证书
openssl genrsa -out dashboard.key 2048 
openssl req -new -out dashboard.csr -key dashboard.key -subj '/CN=192.168.2.201'
openssl x509 -req -in dashboard.csr -signkey dashboard.key -out dashboard.crt 
# 删除原有的证书secret
kubectl delete secret kubernetes-dashboard-certs -n kube-system
# 创建新的证书secret
kubectl create secret generic kubernetes-dashboard-certs --from-file=dashboard.key --from-file=dashboard.crt -n kube-system
# 查看pod
kubectl get pod -n kube-system
# 重启pod
kubectl delete pod <pod name> -n kube-system
```

### 5.3 Dashboard UI

访问地址：https://192.168.2.201:30009
`鼠标点击页面的任意地方，键盘输入： thisisunsafe`

在登录页面上输入上面的token

![](../../assets/_images/devops/deploy/k8s/image1.png)

出现下面的页面代表成功

![](../../assets/_images/devops/deploy/k8s/image2.png)

使用DashBoard，以Deployment为例演示DashBoard的使用

**查看**

选择指定的命名空间`dev`，然后点击`Deployments`，查看dev空间下的所有deployment

![](../../assets/_images/devops/deploy/k8s/image3.png)


**扩缩容**

在`Deployment`上点击`规模`，然后指定`目标副本数量`，点击确定

![](../../assets/_images/devops/deploy/k8s/image4.png)

**编辑**

在`Deployment`上点击`编辑`，然后修改`yaml文件`，点击确定

![](../../assets/_images/devops/deploy/k8s/image5.png)

**查看Pod**

点击`Pods`, 查看pods列表

![](../../assets/_images/devops/deploy/k8s/image6.png)

**操作Pod**

选中某个Pod，可以对其执行日志（logs）、进入执行（exec）、编辑、删除操作

![](../../assets/_images/devops/deploy/k8s/image7.png)

