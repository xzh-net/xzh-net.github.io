# Istio 1.6.8

## 1. 安装

### 1.1 上传解压

下载地址：https://github.com/istio/istio/releases/download/1.6.8/istio-1.6.8-linux-amd64.tar.gz

```bash
mkdir -p /opt/software
cd /opt/software
tar -zxvf istio-1.6.8-linux-amd64.tar.gz -C /opt
mv /opt/istio-1.6.8 /opt/istio
```

### 1.2 执行安装

```bash
tar -xzf istio-1.6.8-linux-amd64.tar.gz
cd /home/k8s/istio-1.6.8                    # 进入安装目录
mv /home/k8s/istio-1.6.8/bin/istioctl /usr/local/bin
istioctl manifest apply --set profile=demo  # 执行安装
kubectl get ns
kubectl get crd -n istio-system | wc -l     # 统计个数
kubectl get pod,svc -n istio-system         # 查看核心组件资源
```

bookinfo安装
```bash
kubectl label namespace default istio-injection=enabled      # 自动注入
istioctl kube-inject -f istio-demo.yaml | kubectl apply -f - # 手动注入
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl get pod
kubectl get svc
```

通过ingressgateway访问
```bash
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml # 
kubectl get svc istio-ingressgateway -n istio-system
kubectl get gateway

kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}'  # 输出ratings运行时pod
kubectl exec -it $(kubectl get pod -l app=ratings  -o jsonpath='{.items[0].metadata.name}') -c ratings  -- curl productpage:9080/productpage | grep -o "<title>.*</title>"
kubectl get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}'
kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}'
# curl 192.168.3.201:31666/productpage
```

通过ingress方式访问

新建productpageIngress.yaml
```yaml
#ingress
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
 name: productpage-ingress
spec:
 rules:
 - host: productpage.istio.qy.com
   http:
     paths:
     - path: /
       backend:
         serviceName: productpage
         servicePort: 9080
```

```bash
kubectl apply -f productpageIngress.yaml # 
kubectl get Ingress --all-namespaces     # 查看创建绑定
kubectl get svc -n ingress-nginx         # 查看端口
```

无法注入问题排查
```bash
cd /etc/kubernetes/manifests
vi kube-apiserver.yaml
- --enable-aggregator-routing=true # 添加本行
kubectl get namespace -L istio-injection
kubectl describe deployment productpage
kubectl describe replicaset productpage-v1-6987489c74
```

### 5.2 卸载

```bash
/home/k8s/istio-1.6.8/samples/bookinfo/platform/kube/cleanup.sh
cd /home/k8s/istio-1.6.8
kubectl delete -f samples/bookinfo/platform/kube/bookinfo.yaml
istioctl manifest generate --set profile=demo | kubectl delete --ignore-not-found=true -f -
kubectl delete namespace istio-system
kubectl label namespace default istio-injection-
```

### 5.3 监控

prometheus和grafana

配置prometheus-ingress.yaml和grafana-ingress.yaml配置文件

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: prometheus-ingress
  namespace: istio-system
spec:
  rules:
  - host: prometheus.istio.qy.com
    http:
      paths:
      - path: /
        backend:
          serviceName: prometheus
          servicePort: 9090
```

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: istio-system
spec:
  rules:
  - host: grafana.istio.qy.com
    http:
      paths:
      - path: /
        backend:
          serviceName: grafana
          servicePort: 3000
```

```bash
kubectl apply -f prometheus-ingress.yaml grafana-ingress.yaml
kubectl get ingress -n istio-system
```

域名配置
```
192.168.187.201    prometheus.istio.qy.com
192.168.187.201    grafana.istio.qy.com
```


## 4. k8s组件回顾

### 4.1 deployment

pod版本管理工具

```yaml
apiVersion: apps/v1 ## 定义了一个版本
kind: Deployment ##k8s资源类型是Deployment
metadata: ## metadata这个KEY对应的值为一个Maps
  name: nginx-deployment ##资源名字 nginx-deployment
  labels: ##将新建的Pod附加Label
    app: nginx ##一个键值对为key=app,valuen=ginx的Label。
spec: #以下其实就是replicaSet的配置
  replicas: 3 ##副本数为3个，也就是有3个pod
  selector: ##匹配具有同一个label属性的pod标签
    matchLabels: ##寻找合适的label，一个键值对为key=app,value=nginx的Labe
      app: nginx
  template: #模板
    metadata:
      labels: ##将新建的Pod附加Label
        app: nginx
    spec:
      containers:  ##定义容器
      - name: nginx ##容器名称
        image: nginx:1.7.9 ##镜像地址
        ports:
        - containerPort: 80 ##容器端口
```

```bash
kubectl apply -f nginx_deployment.yaml  # 创建deployment
kubectl get pods                        # 查看pods
kubectl get deployment -o wide          # 查看pods详细
```

### 4.2 namespace

资源隔离

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myns
```

```bash
kubectl get namespace               # 查看命名空间
kubectl apply -f my-namespace.yaml  # 创建命名空间
kubectl delete -f my-namespace.yaml # 用资源文件删除命名空间
kubectl delete namespace myns       # 按名字删除命名空间    
```

### 4.3 service

pod实现了容器内部互通，但是不稳定，通过service让pod拥有固定ip,包括cluterIp（集群内部访问）和NodePort（暴露外部访问）

```yaml
apiVersion: apps/v1 ## 定义了一个版本
kind: Deployment ##资源类型是Deployment
metadata: ## metadata这个KEY对应的值为一个Maps
  name: whoami-deployment ##资源名字
  labels: ##将新建的Pod附加Label
    app: whoami ##key=app:value=whoami
spec: ##资源它描述了对象的
  replicas: 3 ##副本数为1个，只会有一个pod
  selector: ##匹配具有同一个label属性的pod标签
    matchLabels: ##匹配合适的label
      app: whoami
  template: ##template其实就是对Pod对象的定义  (模板)
    metadata:
      labels:
        app: whoami
    spec:
      containers:
      - name: whoami ##容器名字  下面容器的镜像
        image: jwilder/whoami
        ports:
        - containerPort: 8000 ##容器的端口
```

```bash
kubectl get svc
kubectl expose deployment whoami-deployment             # 暴露service
kubectl describe svc whoami-deployment                  # 详细暴露规则
kubectl scale deployment whoami-deployment --replicas=5 # service扩容 
kubectl delete svc whoami-deployment                    # 删除service
kubectl expose deployment whoami-deployment  --type="NodePort" # 按外部访问方式暴露
```

### 4.4 ingress

ngress相当于一个7层的负载均衡器，是k8s对反向代理的一个抽象。大概的工作原理也确实类似于Nginx，可以理解成在 Ingress 里建立一个个映射规则 , ingress Controller 通过监听 Ingress这个api对象里的配置规则并转化成 Nginx 的配置（kubernetes声明式API和控制循环） , 然后对外部提供服务。ingress包括：ingress controller和ingress resources

```yaml
apiVersion: apps/v1 ## 定义了一个版本
kind: Deployment ##资源类型是Deployment
metadata: ## metadata这个KEY对应的值为一个Maps
  name: whoami-deployment ##资源名字
  labels: ##将新建的Pod附加Label
    app: whoami ##key=app:value=whoami
spec: ##资源它描述了对象的
  replicas: 3 ##副本数为1个，只会有一个pod
  selector: ##匹配具有同一个label属性的pod标签
    matchLabels: ##匹配合适的label
      app: whoami
  template: ##template其实就是对Pod对象的定义  (模板)
    metadata:
      labels:
        app: whoami
    spec:
      containers:
      - name: whoami ##容器名字  下面容器的镜像
        image: jwilder/whoami
        ports:
        - containerPort: 8000 ##容器的端口
---
apiVersion: v1
kind: Service
metadata:
  name: whoami-service
spec:
  ports:
  - port: 80   
    protocol: TCP
    targetPort: 8000
  selector:
    app: whoami

```

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: whoami-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules: # 定义规则
  - host: whoami.qy.com  # 定义访问域名
    http:
      paths:
      - path: / # 定义路径规则，/表示没有设置任何路径规则
        backend:
          serviceName: whoami-service  # 把请求转发给service资源，这个service就是我们前面运行的service
          servicePort: 80 # service的端口
```


```bash
kubectl delete deployment  whoami-deployment # 按照名字删除deployment
kubectl delete -f whoami-deployment.yaml     # 按资源文件删除deployment

kubectl apply -f whoami-service.yaml
kubectl apply -f  whoami-ingress.yaml
kubectl get ingress  # 查看ingress资源
kubectl describe ingress whoami-ingress # 查看ingres详细资源
```

注意：

因为Minikube里边内置了Nginx Ingress Controller这个插件， 默认没有启用，所以如果是在Minikube这个单节点集群里实践的话只需要执行下面的命令

```bash
minikube addons enable ingress
minikube addons disabled ingress
```

检查验证 Nginx Ingress 控制器处于运行状态：

```bash
kubectl get pods -n kube-system --filed-selector=Running
```
访问whoami.qy.com


## 6. 流量管理

### 6.1 放开自定义路由权限

先执行这个文件之后gateway路由规则才可以自定义
```bash
kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml  -n bookinfo-ns
kubectl get DestinationRule -n bookinfo-ns  # 查看路由规则
```

### 6.2 基于版本控制

所有的路由的流量全部都切换到v3版本也就是全部都是红星的版本
```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-v3  -n bookinfo-ns
```

### 6.3 基于权重的流量版本控制

此时会把所有的路由的流量会在v1和v3之间进行切换，各占50%
```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml  -n bookinfo-ns
```

### 6.4 基于用户控制流量

在登录的时候会在header头部增加一个jason，如果是jason登录那么会访问v2版本，其它的人访问的是v3
```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-jason-v2-v3.yaml  -n bookinfo-ns
```

### 6.5 故障注入

在访问的的时候会在header头部增加一个jason，如果是jason访问那么会访问v2版本，其它的人访问的是v3。 访问v3版本的人会注入一个50%几率的延迟2S请求访问

创建故障注入规则-执行:test.yaml
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - fault:
      delay:
        percent: 50
        fixedDelay: 2s
    route:
    - destination:
        host: reviews
        subset: v3
```
```bash
kubectl apply -f test.yaml -n bookinfo-ns
```

### 6.6 流量迁移

一个常见的用例是将流量从一个版本的微服务逐渐迁移到另一个版本。在 Istio 中，您可以通过配置一系列规则来实现此目标， 这些规则将一定百分比的流量路由到一个或另一个服务。在本任务中，您将会把 50％ 的流量发送到 `reviews:v1`，另外 50％ 的流量发送到 `reviews:v3`。然后，再把 100％ 的流量发送到 `reviews:v3` 来完成迁移。

(1)让所有的流量都到v1
```bash
kubectl apply -f virtual-service-all-v1.yaml
```

(2)将v1的50%流量转移到v3
```bash
kubectl apply -f virtual-service-reviews-50-v3.yaml
```

(3)确保v3版本没问题之后，可以将流量都转移到v3
```bash
kubectl apply -f virtual-service-reviews-v3.yaml
```

### 6.7 mixer组件上报

采集指标：自动为Istio生成和收集的应用信息，可以配置的YAML文件

metrics-crd.yaml 

```yaml
# Configuration for metric instances
apiVersion: "config.istio.io/v1alpha2"
kind: instance
metadata:
  name: doublerequestcount
  namespace: istio-system
spec:
  compiledTemplate: metric
  params:
    value: "2" # count each request twice
    dimensions:
      reporter: conditional((context.reporter.kind | "inbound") == "outbound", "client", "server")
      source: source.workload.name | "unknown"
      destination: destination.workload.name | "unknown"
      message: '"twice the fun!"'
    monitored_resource_type: '"UNSPECIFIED"'
---
# Configuration for a Prometheus handler
apiVersion: "config.istio.io/v1alpha2"
kind: prometheus
metadata:
  name: doublehandler
  namespace: istio-system
spec:
  metrics:
  - name: double_request_count # Prometheus metric name
    instance_name: doublerequestcount.instance.istio-system # Mixer instance name (fully-qualified)
    kind: COUNTER
    label_names:
    - reporter
    - source
    - destination
    - message
---
# Rule to send metric instances to a Prometheus handler
apiVersion: "config.istio.io/v1alpha2"
kind: rule
metadata:
  name: doubleprom
  namespace: istio-system
spec:
  actions:
  - handler: doublehandler.prometheus
    instances:
    - doublerequestcount
```

```bash
kubectl apply -f metrics-crd.yaml      # 收集前提安装crd
kubectl get instance -n istio-system   # 检查安装
kubectl get ingress -n istio-system    # 检查普罗米修斯和grafana的ingress存不存在
kubectl apply -f prometheus-ingress.yaml # 创建prometheus的ingress
kubectl apply -f grafana-ingress.yaml    # 创建grafana的ingress
kubectl get svc -o wide -n istio-system  # 查找普罗米修斯ip
```

![](../../assets/_images/java//microservice/servicemesh/observe1.png)

![](../../assets/_images/java//microservice/servicemesh/observe2.png)

![](../../assets/_images/java//microservice/servicemesh/observe3.png)

![](../../assets/_images/java//microservice/servicemesh/observe4.png)