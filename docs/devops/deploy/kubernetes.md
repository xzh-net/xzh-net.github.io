# Kubernetes 1.17.4

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
       successThreshold: 0          #连续探测成功多少次才被认定为成功。默认是1
       failureThreshold: 0          #连续探测失败多少次才被认定为失败。默认是3。最小值是1
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

 容器中的程序要运行，肯定是要占用一定资源的，比如cpu和内存等，如果不对某个容器的资源做限制，那么它就可能吃掉大量资源，导致其它容器无法运行。针对这种情况，kubernetes提供了对内存和cpu的资源进行配额的机制，这种机制主要通过resources选项实现，他有两个子选项：
- limits：用于限制运行时容器的最大占用资源，当容器占用资源超过limits时会被终止，并进行重启
- requests ：用于设置容器需要的最小资源，如果环境资源不够，容器将无法启动

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

#### 2.3.7 钩子函数

钩子函数能够感知自身生命周期中的事件，并在相应的时刻到来时运行用户指定的程序代码。

kubernetes在主容器的启动之后和停止之前提供了两个钩子函数：
- post start：容器创建之后执行，如果失败了会重启容器
- pre stop  ：容器终止之前执行，执行完成之后容器将成功终止，在其完成之前会阻塞删除容器的操作

钩子处理器支持使用下面三种方式定义动作：

1. Exec命令

在容器内执行一次命令

```yaml
lifecycle:
  postStart: 
    exec:
      command:
      - cat
      - /tmp/healthy
```

案例

```bash
vi pod-hook-exec.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-hook-exec
  namespace: dev
spec:
  containers:
  - name: main-container
    image: nginx:1.22.1
    ports:
    - name: nginx-port
      containerPort: 80
    lifecycle:
      postStart: 
        exec: # 在容器启动的时候执行一个命令，修改掉nginx的默认首页内容
          command: ["/bin/sh", "-c", "echo postStart... > /usr/share/nginx/html/index.html"]
      preStop:
        exec: # 在容器停止之前停止nginx服务
          command: ["/usr/sbin/nginx","-s","quit"]
```

```bash
kubectl create -f pod-hook-exec.yaml            # 创建pod
kubectl get pods  pod-hook-exec -n dev -o wide  # 查看pod
curl 10.244.2.48                                # 访问
```

2. TCPSocket

在当前容器尝试访问指定的socket

```yaml
lifecycle:
    postStart:
      tcpSocket:
        port: 8080
```

3. HTTPGet

在当前容器中向某url发起http请求

```yaml
lifecycle:
    postStart:
      httpGet:
        path: / #URI地址
        port: 80 #端口号
        host: 127.0.0.1 #主机地址
        scheme: HTTP #支持的协议，http或者https
```

#### 2.3.8 容器探测

容器探测用于检测容器中的应用实例是否正常工作，是保障业务可用性的一种传统机制。如果经过探测，实例的状态不符合预期，那么kubernetes就会把该问题实例" 摘除 "，不承担业务流量。kubernetes提供了两种探针来实现容器探测，分别是：
- liveness probes：存活性探针，用于检测应用实例当前是否处于正常运行状态，如果不是，k8s会重启容器
- readiness probes：就绪性探针，用于检测应用实例当前是否可以接收请求，如果不能，k8s不会转发流量

?> livenessProbe 决定是否重启容器，readinessProbe 决定是否将请求转发给容器。

上面两种探针目前均支持三种探测方式：

1. Exec命令

在容器内执行一次命令，如果命令执行的退出码为0，则认为程序正常，否则不正常

```yaml
livenessProbe:
    exec:
      command:
      - cat
      - /tmp/healthy
```

案例

```bash
vi pod-liveness-exec.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-liveness-exec
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.22.1
    ports: 
    - name: nginx-port
      containerPort: 80
    livenessProbe:
      exec:
        command: ["/bin/cat","/tmp/hello.txt"] # 执行一个查看文件的命令
```

```bash
kubectl create -f pod-liveness-exec.yaml    # 创建Pod
kubectl get pods -n dev -o wide             # 查看Pod，RESTARTS一直增长说明钩子生效
kubectl describe pods pod-liveness-exec -n dev    # 查看Pod详情，提示找不到/tmp/hello.txt
```

2. TCPSocket

将会尝试访问一个用户容器的端口，如果能够建立这条连接，则认为程序正常，否则不正常

```yaml
livenessProbe:
    tcpSocket:
      port: 8080
```

案例

```bash
vi pod-liveness-tcpsocket.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-liveness-tcpsocket
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.22.1
    ports: 
    - name: nginx-port
      containerPort: 80
    livenessProbe:
      tcpSocket:
        port: 8080 # 尝试访问8080端口
```

```bash
kubectl create -f pod-liveness-tcpsocket.yaml   # 创建Pod
kubectl get pods -n dev -o wide                 # 查看Pod，RESTARTS一直增长说明钩子生效
kubectl describe pods pod-liveness-tcpsocket -n dev    # 提示 8080: connect: connection refused
```


3. HTTPGet

调用容器内Web应用的URL，如果返回的状态码在200和399之间，则认为程序正常，否则不正常

```yaml
livenessProbe:
    httpGet:
      path: / #URI地址
      port: 80 #端口号
      host: 127.0.0.1 #主机地址
      scheme: HTTP #支持的协议，http或者https
```

案例

```bash
vi pod-liveness-httpget.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-liveness-httpget
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.22.1
    ports:
    - name: nginx-port
      containerPort: 80
    livenessProbe:
      httpGet:  # 其实就是访问http://127.0.0.1:80/hello  
        scheme: HTTP #支持的协议，http或者https
        port: 80 #端口号
        path: /hello #URI地址
```

```bash
kubectl create -f pod-liveness-httpget.yaml
kubectl get pods -n dev -o wide
kubectl describe pod pod-liveness-httpget -n dev    # 提示连接失败：HTTP probe failed with statuscode: 404
```

#### 2.3.9 重启策略

一旦容器探测出现了问题，kubernetes就会对容器所在的Pod进行重启，其实这是由pod的重启策略决定的，pod的重启策略有 3 种，分别如下：

- Always ：容器失效时，自动重启该容器，这也是默认值。
- OnFailure ： 容器终止运行且退出码不为0时重启
- Never ： 不论状态为何，都不重启该容器

重启策略适用于pod对象中的所有容器，首次需要重启的容器，将在其需要时立即进行重启，随后再次需要重启的操作将由kubelet延迟一段时间后进行，且反复的重启操作的延迟时长以此为10s、20s、40s、80s、160s和300s，300s是最大延迟时长。

案例

```bash
vi pod-restartpolicy.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-restartpolicy
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.22.1
    ports:
    - name: nginx-port
      containerPort: 80
    livenessProbe:
      httpGet:
        scheme: HTTP
        port: 80
        path: /hello
  restartPolicy: Never # 设置重启策略为Never
```

```bash
kubectl create -f pod-restartpolicy.yaml
kubectl describe pods pod-restartpolicy  -n dev
kubectl get pods pod-restartpolicy -n dev
```

#### 2.3.10 调度规则

在默认情况下，一个Pod在哪个Node节点上运行，是由Scheduler组件采用相应的算法计算出来的，这个过程是不受人工控制的。但是在实际使用中，这并不满足的需求，因为很多情况下，我们想控制某些Pod到达某些节点上，那么应该怎么做呢？这就要求了解kubernetes对Pod的调度规则，kubernetes提供了四大类调度方式：

- 自动调度：运行在哪个节点上完全由Scheduler经过一系列的算法计算得出
- 定向调度：NodeName、NodeSelector
- 亲和性调度：NodeAffinity、PodAffinity、PodAntiAffinity
- 污点（容忍）调度：Taints、Toleration

##### 2.3.10.1 定向调度

定向调度，指的是利用在pod上声明nodeName或者nodeSelector，以此将Pod调度到期望的node节点上。注意，这里的调度是强制的，这就意味着即使要调度的目标Node不存在，也会向上面进行调度，只不过pod运行失败而已

1. NodeName

用于强制约束将Pod调度到指定的Name的Node节点上。这种方式，其实是直接跳过Scheduler的调度逻辑，直接将Pod调度到指定名称的节点

```bash
vi pod-nodename.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodename
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.22.1
  nodeName: k8s-node01 # 指定调度到k8s-node01节点上
```

```bash
kubectl create -f pod-nodename.yaml
kubectl get pods pod-nodename -n dev -o wide
```

2. NodeSelector

NodeSelector用于将pod调度到添加了指定标签的node节点上。它是通过kubernetes的label-selector机制实现的，也就是说，在pod创建之前，会由scheduler使用MatchNodeSelector调度策略进行label匹配，找出目标node，然后将pod调度到目标节点，该匹配规则是强制约束

首先分别为node节点添加标签

```bash
kubectl label nodes k8s-node01 nodeenv=pro
kubectl label nodes k8s-node02 nodeenv=test
```

创建一个pod-nodeselector.yaml文件

```bash
vi pod-nodeselector.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodeselector
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.22.1
  nodeSelector: 
    nodeenv: test # 指定调度到具有nodeenv=test标签的节点上
```

```bash
kubectl create -f pod-nodeselector.yaml
kubectl get pods pod-nodeselector -n dev -o wide
```

##### 2.3.10.2 亲和性调度

kubernetes还提供了一种亲和性调度（Affinity）。它在NodeSelector的基础之上的进行了扩展，可以通过配置的形式，实现优先选择满足条件的Node进行调度，如果没有，也可以调度到不满足条件的节点上，使调度更加灵活

Affinity主要分为三类：
- nodeAffinity(node亲和性）: 以node为目标，解决pod可以调度到哪些node的问题
- podAffinity(pod亲和性) :  以pod为目标，解决pod可以和哪些已存在的pod部署在同一个拓扑域中的问题
- podAntiAffinity(pod反亲和性) :  以pod为目标，解决pod不能和哪些已存在pod部署在同一个拓扑域中的问题

1. NodeAffinity

硬限制

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodeaffinity-required
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.22.1
  affinity:  #亲和性设置
    nodeAffinity: #设置node亲和性
      requiredDuringSchedulingIgnoredDuringExecution: # 硬限制
        nodeSelectorTerms:
        - matchExpressions: # 匹配env的值在["xxx","yyy"]中的标签
          - key: nodeenv
            operator: In
            values: ["pro","yyy"]
```

软限制

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodeaffinity-preferred
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.22.1
  affinity:  #亲和性设置
    nodeAffinity: #设置node亲和性
      preferredDuringSchedulingIgnoredDuringExecution: # 软限制
      - weight: 1
        preference:
          matchExpressions: # 匹配env的值在["xxx","yyy"]中的标签(当前环境没有)
          - key: nodeenv
            operator: In
            values: ["xxx","yyy"]
```

```lua
NodeAffinity规则设置的注意事项：
    1 如果同时定义了nodeSelector和nodeAffinity，那么必须两个条件都得到满足，Pod才能运行在指定的Node上
    2 如果nodeAffinity指定了多个nodeSelectorTerms，那么只需要其中一个能够匹配成功即可
    3 如果一个nodeSelectorTerms中有多个matchExpressions ，则一个节点必须满足所有的才能匹配成功
    4 如果一个pod所在的Node在Pod运行期间其标签发生了改变，不再符合该Pod的节点亲和性需求，则系统将忽略此变化
```

2. PodAffinity

PodAffinity主要实现以运行的Pod为参照，实现让新创建的Pod跟参照pod在一个区域的功能

硬限制，首先创建一个参照Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-podaffinity-target
  namespace: dev
  labels:
    podenv: pro #设置标签
spec:
  containers:
  - name: nginx
    image: nginx:1.22.1
  nodeName: k8s-node01 # 将目标pod名确指定到k8s-node01上
```

创建目标pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-podaffinity-required
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.22.1
  affinity:  #亲和性设置
    podAffinity: #设置pod亲和性
      requiredDuringSchedulingIgnoredDuringExecution: # 硬限制
      - labelSelector:
          matchExpressions: # 匹配env的值在["xxx","yyy"]中的标签
          - key: podenv
            operator: In
            values: ["pro","yyy"]
        topologyKey: kubernetes.io/hostname
```

topologyKey用于指定调度时作用域
   - kubernetes.io/hostname，是以Node节点为区分范围
   - beta.kubernetes.io/os，以Node节点的操作系统类型来区分


3. PodAntiAffinity

PodAntiAffinity主要实现以运行的Pod为参照，让新创建的Pod跟参照pod不在一个区域中的功能

硬限制，继续使用上一个参照Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-podaffinity-target
  namespace: dev
  labels:
    podenv: pro #设置标签
spec:
  containers:
  - name: nginx
    image: nginx:1.22.1
  nodeName: k8s-node01 # 将目标pod名确指定到k8s-node01上
```

创建目标pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-podantiaffinity-required
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.22.1
  affinity:  #亲和性设置
    podAntiAffinity: #设置pod亲和性
      requiredDuringSchedulingIgnoredDuringExecution: # 硬限制
      - labelSelector:
          matchExpressions: # 匹配podenv的值在["pro"]中的标签
          - key: podenv
            operator: In
            values: ["pro"]
        topologyKey: kubernetes.io/hostname
```

##### 2.3.10.3 污点和容忍

1. 污点

前面的调度方式都是站在Pod的角度上，通过在Pod上添加属性，来确定Pod是否要调度到指定的Node上，其实我们也可以站在Node的角度上，通过在Node上添加`污点`属性，来决定是否允许Pod调度过来

Node被设置上污点之后就和Pod之间存在了一种相斥的关系，进而拒绝Pod调度进来，甚至可以将已经存在的Pod驱逐出去。

污点的格式为：`key=value:effect`, key和value是污点的标签，effect描述污点的作用，支持如下三个选项：
- PreferNoSchedule：kubernetes将尽量避免把Pod调度到具有该污点的Node上，除非没有其他节点可调度
- NoSchedule：kubernetes将不会把Pod调度到具有该污点的Node上，但不会影响当前Node上已存在的Pod
- NoExecute：kubernetes将不会把Pod调度到具有该污点的Node上，同时也会将Node上已存在的Pod驱离

```bash
kubectl taint nodes k8s-node01 key=value:effect  # 设置污点
kubectl taint nodes k8s-node01 key:effect-       # 去除污点
kubectl taint nodes k8s-node01 key-              # 去除所有污点
```

案例1验证PreferNoSchedule

```bash
# 停止node02节点模拟只有一个节点的情况
kubectl taint nodes k8s-node01 tag=xzh:PreferNoSchedule   # 为node1设置污点(PreferNoSchedule)
kubectl describe nodes k8s-node01                         # 查看节点污点
kubectl run taint1 --image=nginx:1.22.1 -n dev            # 创建pod
kubectl get pods -n dev -o wide
```

案例2验证NoSchedule

```bash
kubectl taint nodes k8s-node01 tag:PreferNoSchedule-      # 取消污点
kubectl taint nodes k8s-node01 tag=xzh:NoSchedule         # 修改污点
kubectl run taint2 --image=nginx:1.22.1 -n dev            # 创建pod
kubectl get pods -n dev -o wide
```

案例3验证NoExecute

```bash
kubectl taint nodes k8s-node01 tag:NoSchedule-            # 取消污点
kubectl taint nodes k8s-node01 tag=xzh:NoExecute          # 修改污点
kubectl run taint3 --image=nginx:1.22.1 -n dev            # 创建pod
kubectl get pods -n dev -o wide
```

2. 容忍

上面介绍了污点的作用，我们可以在node上添加污点用于拒绝pod调度上来，但是如果就是想将一个pod调度到一个有污点的node上去，这时候应该怎么做呢？这就要使用到`容忍`。

?> 污点就是拒绝，容忍就是忽略，Node通过污点拒绝pod调度上去，Pod通过容忍忽略拒绝

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-toleration
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.22.1
  tolerations:      # 添加容忍
  - key: "tag"        # 要容忍的污点的key
    operator: "Equal" # 操作符
    value: "xzh"    # 容忍的污点的value
    effect: "NoExecute"   # 添加容忍的规则，这里必须和标记的污点规则相同
```

添加了容忍之后，该pod可以正常运行在有污点的节点上

#### 2.3.11 控制器

Pod控制器是管理pod的中间层，使用Pod控制器之后，只需要告诉Pod控制器，想要多少个什么样的Pod就可以了，它会创建出满足条件的Pod并确保每一个Pod资源处于用户期望的目标状态。如果Pod资源在运行中出现故障，它会基于指定策略重新编排Pod

##### 2.3.11.1 ReplicaSet

##### 2.3.11.1 ReplicaSet

##### 2.3.11.1 ReplicaSet

##### 2.3.11.1 ReplicaSet

##### 2.3.11.1 ReplicaSet

### 2.4 Lable

#### 2.4.1 命令式

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

#### 2.5.1 命令式

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

#### 2.6.1 命令式

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

?> k8s访问svc的clusterip异常的慢

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

