# Higress 2.1.9

面向 LLM，统一代理各主流大模型和自建大模型服务，提供 OpenAI 兼容的访问方式，并提供二次 API KEY 签发、限流、安全防护、观测等治理能力。


## 1. 快速开始

极简方式，基于本地文件做配置存储，创建挂载路径

```bash
mkdir /data/higress/{data,log,proxy} -p
chmod -R /data/higress
```

启动服务

```bash
docker run -dit --name higress-ai \
    -v /data/higress/data:/data \
    -v /data/higress/log:/var/log/higress \
    -v /data/higress/proxy:/var/log/proxy \
    -e O11Y=on \
    -p 8001:8001 -p 8080:8080 -p 8443:8443  \
    higress-registry.cn-hangzhou.cr.aliyuncs.com/higress/all-in-one:2.1.9
```

## 2. 配置

访问地址：http://192.168.1.100:8001

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

## 3. 日志

网关日志已经映射到宿主机的`/data/higress/proxy`路径下，其中`access.log`是每次请求的日志，后面我们基于这个日志汇总每次请求消耗的token数量，生成账单。

前面已经为每个服务添加了消费者，但是统计日志的时候无法获取消费者身份信息，需要自定义日志格式。因为我们是本地配置，可以直接编辑映射出来的配置文件。

```bash
vi /data/higress/data/configmaps/higress-config.yaml
```

访问日志`accessLogFormat`的标准格式如下：

```yaml
{
    
    # AI特征日志
	"ai_log": "%FILTER_STATE(wasm.ai_log:PLAIN)%",
    # 目标服务地址（网关或后端服务的 IP + 端口）
	"authority": "%REQ(X-ENVOY-ORIGINAL-HOST?:AUTHORITY)%",
    # 网关从下游客户端接收的字节数（即用户请求体大小，含提问内容、请求参数等）
	"bytes_received": "%BYTES_RECEIVED%",
    # 网关向上游服务发送 + 向下游客户端返回的总字节数（含 AI 回复内容、响应头、元数据等）
	"bytes_sent": "%BYTES_SENT%",
    # 网关接收下游请求的本地 IP: 端口（即网关对外提供服务的地址，客户端实际连接的网关地址）
	"downstream_local_address": "%DOWNSTREAM_LOCAL_ADDRESS%",
    # 下游客户端的IP: 端口（即发起请求的客户端地址）
	"downstream_remote_address": "%DOWNSTREAM_REMOTE_ADDRESS%",
    # 整个请求的总耗时（单位：毫秒），从网关接收请求到返回响应的完整周期（含网关转发 + 上游处理 + 网络耗时）
	"duration": "%DURATION%",
    # Istio 服务网格的策略执行状态（如认证、授权、限流等），-表示未启用 Istio 策略或无相关策略触发
    "istio_policy_status": "%DYNAMIC_METADATA(istio.mixer:status)%",
    # HTTP 请求方法，默认使用POST
	"method": "%REQ(:METHOD)%",
    # 请求的 URL 路径，LLM chat模型默认端点：/v1/chat/completions
	"path": "%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%",
    # HTTP 协议版本
	"protocol": "%PROTOCOL%",
    # 网关生成的请求唯一 ID（与ai_log.chat_id一致），用于追踪单个请求的全链路（网关→上游→AI 模型）
	"request_id": "%REQ(X-REQUEST-ID)%",
    # TLS 请求的SNI（服务器名称指示），用于多域名共享证书场景，-表示未使用 TLS 或未指定 SNI
	"requested_server_name": "%REQUESTED_SERVER_NAME%",
    # 响应状态码
	"response_code": "%RESPONSE_CODE%",
    # 网关处理响应的标记（如UF表示上游连接失败、NR表示无匹配路由），-表示无异常标记（处理正常）
	"response_flags": "%RESPONSE_FLAGS%",
    # 网关匹配的路由规则名称
	"route_name": "%ROUTE_NAME%",
    # 请求开始处理的时间
	"start_time": "%START_TIME%",
    # 分布式追踪的全局唯一 ID
	"trace_id": "%REQ(X-B3-TRACEID)%",
    # 上游信息
	"upstream_cluster": "%UPSTREAM_CLUSTER%",
	"upstream_host": "%UPSTREAM_HOST%",
	"upstream_local_address": "%UPSTREAM_LOCAL_ADDRESS%",
	"upstream_service_time": "%RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)%",
	"upstream_transport_failure_reason": "%UPSTREAM_TRANSPORT_FAILURE_REASON%",
    # 客户端请求头
	"user_agent": "%REQ(USER-AGENT)%",
    # 客户端的原始 IP 地址
	"x_forwarded_for": "%REQ(X-FORWARDED-FOR)%",
    # 响应明细
	"response_code_details": "%RESPONSE_CODE_DETAILS%"
}
```

增加自定义信息

```yaml
# 自定义请求头
"user_api_key": "%REQ(vjsp-api-key)%",
"client_version": "%REQ(CLIENT-VERSION)%"
```