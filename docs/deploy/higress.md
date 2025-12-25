# Higress 2.1.9

面向 LLM，统一代理各主流大模型和自建大模型服务，提供 OpenAI 兼容的访问方式，并提供二次 API KEY 签发、限流、安全防护、观测等治理能力。

- 官方网站：https://higress.cn/

## 1. 部署

### 1.1 快速启动

基于本地文件做配置存储，创建挂载路径

```bash
mkdir /data/higress/{data,log,proxy} -p
chmod -R 777 /data/higress
```

启动服务

```bash
docker run -dit --name higress-ai \
    -v /data/higress/data:/data \
    -v /data/higress/log:/var/log/higress \
    -v /data/higress/proxy:/var/log/proxy \
    -e O11Y=on \
    -p 8080:8001 -p 80:8080 -p 443:8443  \
    higress-registry.cn-hangzhou.cr.aliyuncs.com/higress/all-in-one:2.1.9
```

### 1.2 独立运行版

下载地址：https://github.com/higress-group/higress-standalone

```bash
mkdir /data && cd /data
# 上传文件解压
unzip higress-standalone-aio-v2.1.9.zip && cd higress-standalone-aio-v2.1.9
# 安装初始化
./bin/configure.sh -a
```

依照命令行提示输入所需要的配置参数，脚本会自动写入配置并启动 Higress。使用 admin 作为用户名进行登录，首次进入会要求设置密码。

## 2. 配置

访问地址：http://192.168.1.100:8080

### 2.1 创建AI服务提供者

![](../../assets/_images/deploy/higress/1.png)

### 2.2 创建消费者

![](../../assets/_images/deploy/higress/2.png)

### 2.3 创建AI路由

![](../../assets/_images/deploy/higress/3.png)

![](../../assets/_images/deploy/higress/4.png)

### 2.4 测试

![](../../assets/_images/deploy/higress/5_1.png)

![](../../assets/_images/deploy/higress/5_2.png)

## 3. 高级功能

所有高级功能插件配置全部基于极简方式部署的路径

### 3.1 网关日志

网关的访问日志输出格式通过 `系统设置 -> 编辑全局配置`中的`accessLogFormat` 字段设置，每次修改需要重启应用。

使用极简方式部署，网关日志已经映射到宿主机的`/data/higress/proxy`路径下，其中`access.log`是每次请求的日志，后面我们基于这个日志汇总每次请求消耗的token数量，生成账单。

前面已经为每个服务添加了消费者，但是统计日志的时候无法获取消费者身份信息，需要自定义日志格式。因为我们是本地配置，可以直接编辑映射出来的配置文件。

```bash
vi /data/higress/data/configmaps/higress-config.yaml
```

访问日志`accessLogFormat`的标准格式如下：

```yaml
{
    
    # AI特征日志
	"ai_log":"%FILTER_STATE(wasm.ai_log:PLAIN)%",
    # 目标服务地址（网关或后端服务的 IP + 端口）
	"authority":"%REQ(X-ENVOY-ORIGINAL-HOST?:AUTHORITY)%",
    # 网关从下游客户端接收的字节数（即用户请求体大小，含提问内容、请求参数等）
	"bytes_received":"%BYTES_RECEIVED%",
    # 网关向上游服务发送 + 向下游客户端返回的总字节数（含 AI 回复内容、响应头、元数据等）
	"bytes_sent":"%BYTES_SENT%",
    # 网关接收下游请求的本地 IP:端口（即网关对外提供服务的地址，客户端实际连接的网关地址）
	"downstream_local_address":"%DOWNSTREAM_LOCAL_ADDRESS%",
    # 下游客户端的IP:端口（即发起请求的客户端地址）
	"downstream_remote_address":"%DOWNSTREAM_REMOTE_ADDRESS%",
    # 整个请求的总耗时（单位：毫秒），从网关接收请求到返回响应的完整周期（含网关转发 + 上游处理 + 网络耗时）
	"duration":"%DURATION%",
    # Istio 服务网格的策略执行状态（如认证、授权、限流等），-表示未启用 Istio 策略或无相关策略触发
    "istio_policy_status":"%DYNAMIC_METADATA(istio.mixer:status)%",
    # HTTP 请求方法，默认使用POST
	"method":"%REQ(:METHOD)%",
    # 请求的 URL 路径，LLM chat模型默认端点：/v1/chat/completions
	"path":"%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%",
    # HTTP 协议版本
	"protocol":"%PROTOCOL%",
    # 网关生成的请求唯一 ID（与ai_log.chat_id一致），用于追踪单个请求的全链路（网关→上游→AI 模型）
	"request_id":"%REQ(X-REQUEST-ID)%",
    # TLS 请求的SNI（服务器名称指示），用于多域名共享证书场景，-表示未使用 TLS 或未指定 SNI
	"requested_server_name":"%REQUESTED_SERVER_NAME%",
    # 响应状态码
	"response_code":"%RESPONSE_CODE%",
    # 网关处理响应的标记（如UF表示上游连接失败、NR表示无匹配路由），-表示无异常标记（处理正常）
	"response_flags":"%RESPONSE_FLAGS%",
    # 网关匹配的路由规则名称
	"route_name":"%ROUTE_NAME%",
    # 请求开始处理的时间
	"start_time":"%START_TIME%",
    # 分布式追踪的全局唯一 ID
	"trace_id":"%REQ(X-B3-TRACEID)%",
    # 上游信息
	"upstream_cluster":"%UPSTREAM_CLUSTER%",
	"upstream_host":"%UPSTREAM_HOST%",
	"upstream_local_address":"%UPSTREAM_LOCAL_ADDRESS%",
	"upstream_service_time":"%RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)%",
	"upstream_transport_failure_reason":"%UPSTREAM_TRANSPORT_FAILURE_REASON%",
    # 客户端请求头
	"user_agent":"%REQ(USER-AGENT)%",
    # 客户端的原始 IP 地址
	"x_forwarded_for":"%REQ(X-FORWARDED-FOR)%",
    # 响应明细
	"response_code_details":"%RESPONSE_CODE_DETAILS%"
}
```

增加从请求头获取数据

```yaml
# 请求头
"user_api_key":"%REQ(user-api-key)%",
"client_version":"%REQ(CLIENT-VERSION)%"
```

重启服务


!> 使用独立运行版如果出现修改全局配置`mesh`属性无效的问题，有两种解决办法：

1. 安装之前： `/data/higress-standalone-aio-v2.1.9/compose/scripts/prepare.sh` 修改默认值。

2. 安装之后： `/data/higress-standalone-aio-v2.1.9/compose/volumes/pilot/config/mesh`


### 3.2 调整日志等级

进入 gateway 所在容器的 bash 终端

```bash
docker exec -it higress-ai /bin/bash
```

如果是独立运行版

```bash
docker exec -it higress-gateway-1 /bin/bash
```

执行命令

```bash
# Wasm 插件模块
curl localhost:15000/logging?wasm=debug -X POST
# MCP Server 模块
curl localhost:15000/logging?golang=debug -X POST    
```



日志等级包括：
- trace
- debug
- info
- warning/warn（默认值）
- error
- critical
- off

调整后的日志等级在 gateway 重启后将会自动失效


### 3.3 AI 统计

提供了对 token 用量的统计信息，包括日志、监控以及告警

```yaml
attributes:
- apply_to_log:true
  key:"question"
  value:"messages.@reverse.0.content"
  value_source:"request_body"
- apply_to_log:true
  key:"answer"
  rule:"append"
  value:"choices.0.delta.content"
  value_source:"response_streaming_body"
- apply_to_log:true
  key:"answer"
  value:"choices.0.message.content"
  value_source:"response_body"
```

### 3.4 AI 意图识别

通过大模型对用户提问进行分类，将返回结果通过请求头或者请求体进行分发，实现精确路由。

1. 新建一个固定地址的服务：intent-service，服务指向172.17.17.16:80 （即自身网关实例+端口）。
2. 新建一条higress的大模型路由，供插件访问大模型，路由以 /intent 作为前缀。
3. 为路由开启`AI 意图识别`插件功能。

```yaml
llm:
  proxyDomain:"172.17.17.161"
  proxyModel:"Qwen3-Coder-30B-A3B-Instruct"
  proxyPort:"80"
  proxyServiceName:"intent-service.static"
  proxyTimeout:"10000"
  proxyUrl:"http://172.17.17.161:80/intent/compatible-mode/v1/chat/completions"
scene:
  category:"金融|电商|法律|Higress"
  prompt:"你是一个智能类别识别助手，负责根据用户提出的问题和预设的类别，确定问题属于哪个预设的类别，并给出相应的类别。用户提出的问题为:'%s',预设的类别为'%s'，直接返回一种具体类别，如果没有找到就返回'NotFound'。"
```


### 3.5 请求/响应转换

对请求/响应头、请求查询参数、请求/响应体参数进行转换。



#### 3.5.1 实现基于Body参数路由

将body中的 userId 转换到请求头中的 x-user-id 

```yaml
reqRules:
- operate:map
  headers:
  - fromKey:userId
    toKey:x-user-id
  mapSource:body
```

此配置同时支持 application/json 和 application/x-www-form-urlencoded 两种类型的请求Body。复杂 Json 处理，可以根据 [GJSON Path](https://github.com/tidwall/gjson/blob/master/SYNTAX.md?spm=36971b57.35684624.0.0.67a947678O6zqr&file=SYNTAX.md) 语法。

#### 3.5.2 请求头转换

```yaml
reqRules:
# 移除请求头中的 X-remove
- operate:remove
  headers:
  - key:X-remove
# 将请求头中的 X-not-renamed 重命名 X-renamed
- operate:rename
  headers:
  - oldKey:X-not-renamed
    newKey:X-renamed
# 将请求头中的 X-replace 的值更新成 replaced
- operate:replace
  headers:
  - key:X-replace
    newValue:replaced
# 添加请求头中的 X-add-append
- operate:add
  headers:
  - key:X-add-append
    value:host-$1
    host_pattern:^(.*)\.com$
# 追加请求头中的 X-add-append
- operate:append
  headers:
  - key:X-add-append
    appendValue:path-$1
    path_pattern:^.*?\/(\w+)[\?]{0,1}.*$
# 将加请求头中的 X-add-append 转成 X-map
- operate:map
  headers:
  - fromKey:X-add-append
    toKey:X-map
# 将加请求头中的 字段值去重
- operate:dedupe
  headers:
  ## X-dedupe-first 取第一个值
  - key:X-dedupe-first
    strategy:RETAIN_FIRST
  ## X-dedupe-last 取最后一个值  
  - key:X-dedupe-last
    strategy:RETAIN_LAST
  ## X-dedupe-unique 取唯一值    
  - key:X-dedupe-unique
    strategy:RETAIN_UNIQUE
```

数据样例

```bash
curl -v console.higress.io/get -H 'host:foo.bar.com' \
-H 'X-remove:exist' -H 'X-not-renamed:test' -H 'X-replace:not-replaced' \
-H 'X-dedupe-first:1' -H 'X-dedupe-first:2' -H 'X-dedupe-first:3' \
-H 'X-dedupe-last:a' -H 'X-dedupe-last:b' -H 'X-dedupe-last:c' \
-H 'X-dedupe-unique:1' -H 'X-dedupe-unique:2' -H 'X-dedupe-unique:3' \
-H 'X-dedupe-unique:3' -H 'X-dedupe-unique:2' -H 'X-dedupe-unique:1'

# httpbin 响应结果
{
  "args":{},
  "headers":{
    ...
    "X-Add-Append":"host-foo.bar,path-get",
    ...
    "X-Dedupe-First":"1",
    "X-Dedupe-Last":"c",
    "X-Dedupe-Unique":"1,2,3",
    ...
    "X-Map":"host-foo.bar,path-get",
    "X-Renamed":"test",
    "X-Replace":"replaced"
  },
  ...
}
```

#### 3.5.3 请求体参数转换

```yaml
reqRules:
# 将body中 a1 移除
- operate:remove
  body:
  - key:a1
# 将body中 a2 重命名 a2-new
- operate:rename
  body:
  - oldKey:a2
    newKey:a2-new
# 将body中的 temperature 设置新值
  operate:"replace"
- body:
  - key:"temperature"
    newValue:0.79
    value_type:"string"
# 向body中添加新值
- operate:add
  body:
  - key:a1-new
    value:t1-new
    value_type:string
# 向body中追加新值
- operate:append
  body:
  - key:a1-new
    appendValue:t1-$1-append
    value_type:string
    host_pattern:^(.*)\.com$
```

#### 3.5.4 请求查询参数转换

```yaml
reqRules:
# 移除 k1
- operate:remove
  querys:
  - key:k1
# k2 重命名 k2-new
- operate:rename
  querys:
  - oldKey:k2
    newKey:k2-new
# k2-new 设置新值
- operate:replace
  querys:
  - key:k2-new
    newValue:v2-new
# 添加新值 k3
- operate:add
  querys:
  - key:k3
    value:v31-$1
    path_pattern:^.*?\/(\w+)[\?]{0,1}.*$
# 追加新值 k3
- operate:append
  querys:
  - key:k3
    appendValue:v32
# k3 转换成 k4
- operate:map
  querys:
  - fromKey:k3
    toKey:k4
# k4 去重 取第一个值
- operate:dedupe
  querys:
  - key:k4
    strategy:RETAIN_FIRST
```

#### 3.5.5 嵌套


指定的 key 中含有 `.` 表示嵌套含义

```yaml
respRules:
- operate:add
  body:
  - key:foo.bar
    value:value
```

样例数据

```bash
curl -v console.higress.io/get

# httpbin 响应结果
{
 ...
 "foo":{
  "bar":"value"
 },
 ...
}
```


#### 3.5.6 非嵌套转义

```yaml
respRules:
- operate:add
  body:
  - key:foo\.bar
    value:value
```

样例数据


```bash
curl -v console.higress.io/get

# httpbin 响应结果
{
 ...
 "foo.bar":"value",
 ...
}
```

#### 3.5.7 访问元素数组

```json
{
  "users":[
    {
      "123":{ "name":"zhangsan", "age":18 }
    },
    {
      "456":{ "name":"lisi", "age":19 }
    }
  ]
}
```

移除 `user` 第一个元素

```yaml
reqRules:
- operate:remove
  body:
  - key:users.0
```

样例数据

```bash
curl -v -X POST console.higress.io/post \
-H 'Content-Type:application/json' \
-d '{"users":[{"123":{"name":"zhangsan"}},{"456":{"name":"lisi"}}]}'
{
  ...
  "json":{
    "users":[
      {
        "456":{
          "name":"lisi"
        }
      }
    ]
  },
  ...
}
```

#### 3.5.8 遍历数组

!> 该操作目前只能用在 replace 上，请勿在其他转换中尝试该操作，以免造成无法预知的结果

```json
{
  "users":[
    {
      "name":"zhangsan",
      "age":18
    },
    {
      "name":"lisi",
      "age":19
    }
  ]
}
```

```yaml
reqRules:
- operate:replace
  body:
  - key:users.#.age
    newValue:20
```

样例数据

```bash
curl -v -X POST console.higress.io/post \
-H 'Content-Type:application/json' \
-d '{"users":[{"name":"zhangsan","age":18},{"name":"lisi","age":19}]}'
{
  ...
  "json":{
    "users":[
      {
        "age":"20",
        "name":"zhangsan"
      },
      {
        "age":"20",
        "name":"lisi"
      }
    ]
  },
  ...
}
```