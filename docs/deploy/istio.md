# Istio 1.6.8

- 下载地址：https://github.com/istio/istio/releases/download/1.6.8/istio-1.6.8-linux-amd64.tar.gz

## 1. 安装

### 1.1 上传解压

```bash
mkdir -p /opt/software
cd /opt/software
tar -zxvf istio-1.6.8-linux-amd64.tar.gz -C /opt
mv /opt/istio-1.6.8 /opt/istio
```

### 1.2 安装

```bash           
mv /opt/istio/bin/istioctl /usr/local/bin
istioctl manifest apply --set profile=demo    # 执行安装
kubectl get crd -n istio-system | wc -l       # 统计个数
kubectl get pod,svc -n istio-system           # 查看核心组件资源
```

### 1.3 卸载

```bash
/opt/istio/samples/bookinfo/platform/kube/cleanup.sh    # 清空应用，输入命名空间 bookinfo
istioctl manifest generate --set profile=demo | kubectl delete --ignore-not-found=true -f -
kubectl delete -f /opt/istio/samples/bookinfo/platform/kube/bookinfo.yaml 
kubectl delete namespace istio-system             # 删除命名空间
kubectl label namespace default istio-injection-  # 关闭自动注入
```

## 2. bookinfo案例

### 2.1 准备工作

```bash
kubectl create ns bookinfo    # 创建命名空间
kubectl label namespace bookinfo istio-injection=enabled    # 开启自动注入
istioctl kube-inject -f first-istio.yaml | kubectl apply -f -   # 可选操作，不必执行
```

### 2.2 执行安装

```bash
kubectl apply -f /opt/istio/samples/bookinfo/platform/kube/bookinfo.yaml -n bookinfo
kubectl get pod,svc -n bookinfo
```

### 2.3 通过ingress访问

查看productpage应用service暴漏端口

```bash
kubectl get svc -o wide -n bookinfo
```

将productpage绑定Ingress

```yaml
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
kubectl apply -f productpageIngress.yaml -n bookinfo
kubectl get Ingress -n bookinfo             # 查看创建绑定
```

在hosts文件里面增加ip域名的映射关系

```bash
192.168.2.201 productpage.istio.qy.com
```

> 访问地址：http://productpage.istio.qy.com:30080 k8s中将ingress的80映射为30080

### 2.4 通过istio的ingressgateway访问

```bash
kubectl apply -f /opt/istio/samples/bookinfo/networking/bookinfo-gateway.yaml -n bookinfo
```

确认网关和访问地址

```bash
kubectl get gateways -n bookinfo
kubectl get virtualservices -n bookinfo
kubectl get svc -n istio-system|grep istio-ingressgateway

# 获取网关访问地址
kubectl get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}'
# 获取网关访问端口
kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}'
```

> 访问地址：http://192.168.2.203:31561/productpage

### 2.5 流量管理

#### 2.5.1 放开自定义权限

先执行这个文件之后gateway路由规则才可以自定义

```bash
kubectl apply -f /opt/istio/samples/bookinfo/networking/destination-rule-all.yaml  -n bookinfo
kubectl get DestinationRule -n bookinfo       # 查看路由规则
```

#### 2.5.2 基于版本控制

所有的路由的流量全部都切换到v3版本也就是全部都是红星的版本

```bash
kubectl apply -f /opt/istio/samples/bookinfo/networking/virtual-service-reviews-v3.yaml -n bookinfo
kubectl delete -f /opt/istio/samples/bookinfo/networking/virtual-service-reviews-v3.yaml -n bookinfo   # 删除版本控制
```

#### 2.5.3 基于权重控制

此时会把所有的路由的流量会在v1和v3之间进行切换，各占50%

```bash
kubectl apply -f /opt/istio/samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml -n bookinfo
kubectl delete -f /opt/istio/samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml -n bookinfo    # 删除权重控制
```

#### 2.5.4 基于用户控制

在登录的时候会在header头部增加一个jason，如果是jason登录那么会访问v2版本，其它的人访问的是v3

```bash
kubectl apply -f /opt/istio/samples/bookinfo/networking/virtual-service-reviews-jason-v2-v3.yaml -n bookinfo
kubectl delete -f /opt/istio/samples/bookinfo/networking/virtual-service-reviews-jason-v2-v3.yaml -n bookinfo    # 删除用户控制
```

### 2.6 故障注入

在访问的的时候会在header头部增加一个jason，如果是jason访问那么会访问v2版本，其它的人访问的是v3。 访问v3版本的人会注入一个50%几率的延迟2S请求访问

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
kubectl apply -f test.yaml -n bookinfo
```

### 2.7 流量迁移

一个常见的用例是将流量从一个版本的微服务逐渐迁移到另一个版本。在 Istio 中，您可以通过配置一系列规则来实现此目标， 这些规则将一定百分比的流量路由到一个或另一个服务。在本任务中，您将会把 50％ 的流量发送到 `reviews:v1`，另外 50％ 的流量发送到 `reviews:v3`。然后，再把 100％ 的流量发送到 `reviews:v3` 来完成迁移。

1. 让所有的流量都到v1

```bash
kubectl apply -f virtual-service-all-v1.yaml
```

2. 将v1的50%流量转移到v3

```bash
kubectl apply -f virtual-service-reviews-50-v3.yaml
```

3. 确保v3版本没问题之后，可以将流量都转移到v3

```bash
kubectl apply -f virtual-service-reviews-v3.yaml
```

## 3. 监控

Prometheus存储服务的监控数据，数据来自于istio组件mixer上报，Grafana开源数据可视化工具，展示Prometheus收集到的监控数据
 
### 3.1 Prometheus

```bash
kubectl get all -n istio-system   # 查看istio自带组件
```

其实istio已经默认帮我们安装好了grafana和prometheus，只是对应的Service类型是clusterIP类型，表示集群内部可以访问，如果我们需要能够通过浏览器访问，我们只需要ingress访问规则即可

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

```bash
kubectl apply -f prometheus-ingress.yaml
kubectl get ingress -n istio-system
```

在hosts文件里面增加ip域名的映射关系

```
192.168.2.201 prometheus.istio.qy.com
```

访问地址：prometheus.istio.qy.com:30080（k8s中将ingress的80映射为30080）

### 3.2 Grafana

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
kubectl apply -f grafana-ingress.yaml
kubectl get ingress -n istio-system
```

在hosts文件里面增加ip域名的映射关系

```
192.168.2.201 grafana.istio.qy.com
```

访问地址：grafana.istio.qy.com:30080（k8s中将ingress的80映射为30080）

