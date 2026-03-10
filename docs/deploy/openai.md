# OpenAI

## 1. 演进历史

- 第一阶段：技术引爆，把`大模型`从论文变成10亿人可感知的产品
- 第二阶段：生态定义，用一套API`收编`整个行业
- 第三阶段：战略升维，从`卖模型`到`造基建、控全栈`

OpenAI亲手打开了潘多拉的盒子，然后花十年时间，试图成为盒子的唯一保管者。并在所有人都以为它要躺着收钱时，转身押上全部身家，赌下一个十年。

如果说OpenAI做的是`升维`，从技术到标准再到基建，逐步掌控全产业链。那么DeepSeek做的是`破局`，在算力被封锁、巨头已占位的既定格局下，用极致的工程效率把技术霸权拉下神坛，进而重构全球开源AI的权力版图。

> DeepSeek最大的价值不是被膜拜，而是被超越！


## 2. 核心角色及对话流程示例

**四大核心角色**

| 角色          | 发送者      | 用途                                              | 必需性           | 典型内容格式   |
| :------------ | :---------- | :------------------------------------------------ | :--------------- | :------------- |
| **system**    | 开发者/用户 | 设定AI的长期身份、行为准则、上下文规则            | 可选但推荐       | 纯文本         |
| **user**      | 最终用户    | 用户的问题、指令、请求                            | 必需             | 纯文本         |
| **assistant** | AI模型      | 1. 直接回答 2. 请求工具调用 3. 基于工具结果的回答 | 多轮对话中必需   | 见示例 |
| **tool**      | 外部系统    | 工具/函数的执行结果                               | 工具调用场景必需 | JSON字符串     |


**对话流程示例数据**

```json
{
	// 1. 系统设定
	{
		"role": "system",
		"content": "你是天气助手，可根据需要调用get_weather工具。"
	},

	// 2. 用户提问  
	{
		"role": "user",
		"content": "北京今天的天气怎么样？"
	},

	// 3. 模型决定调用工具
	{
		role: "assistant",
		content: null, // ← 必须为null
		tool_calls: [{
			id: "call_123",
			type: "function",
			function: {
				name: "get_weather",
				arguments: "{\"city\":\"北京\"}" // ← JSON字符串
			}
		}]
	},

	// 4. 执行工具，得到结果，以tool消息的形式发送给AI（作为下一次请求的一部分）
	{
		role: "tool",
		tool_call_id: "call_123", // ← 匹配上面的id
		name: "get_weather",
		content: "{\"weather\":\"晴\",\"temperature\":\"25°C\"}"
	},

	// 5. 模型阅读工具结果，给出最终回答
	{
		role: "assistant",
		content: "北京天气晴朗，气温25°C。" // ← 正常文本，无tool_calls
	}
}
```

## 3. 主流大模型的API调用


### 3.1 月之暗面 / Kimi chat

- API key申请地址：https://platform.moonshot.cn/console/api-keys
- API文档地址：https://platform.moonshot.cn/docs
- API定价信息：https://platform.moonshot.cn/docs/price/chat


### 3.2 百度 / 文心一言

- API key申请地址：https://console.bce.baidu.com/qianfan/ais/console/applicationConsole/application
- API文档地址：https://cloud.baidu.com/doc/WENXINWORKSHOP/s/flfmc9do2
- API定价信息：https://cloud.baidu.com/doc/WENXINWORKSHOP/s/Blfmc9dlf


### 3.3 智谱 / GLM

- API key申请地址：https://bigmodel.cn/usercenter/apikeys
- API文档地址：https://bigmodel.cn/dev/api
- API定价信息：https://open.bigmodel.cn/pricing


### 3.4 MiniMax

- API key申请地址：https://platform.minimaxi.com/user-center/basic-information/interface-key
- API文档地址：https://platform.minimaxi.com/document/notice
- API定价信息：https://platform.minimaxi.com/document/price


### 3.5 阿里 / 通义千问 （Qwen）

- API key申请地址：https://dashscope.console.aliyun.com/apiKey
- API文档地址：https://help.aliyun.com/zh/dashscope/developer-reference
- API定价信息：https://dashscope.console.aliyun.com/billing


1. 创建对话请求

```bash
curl -X POST https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions \
-H "Authorization: Bearer $DASHSCOPE_API_KEY" \
-H "Content-Type: application/json" \
-d '{
    "model": "qwen-plus",
    "messages": [
        {
            "role": "system",
            "content": "You are a helpful assistant."
        },
        {
            "role": "user",
            "content": "你是谁？"
        }
    ]
}'
```

2. 创建向量请求

```bash
curl --location 'https://dashscope.aliyuncs.com/compatible-mode/v1/embeddings' \
--header "Authorization: Bearer $DASHSCOPE_API_KEY" \
--header 'Content-Type: application/json' \
--data '{
    "model": "text-embedding-v4",
    "input": "风急天高猿啸哀，渚清沙白鸟飞回，无边落木萧萧下，不尽长江滚滚来",  
    "dimensions": "1024",  
    "encoding_format": "float"
}'
```

3. 创建重排序请求

```bash
curl --request POST \
  --url https://dashscope.aliyuncs.com/compatible-api/v1/reranks \
  --header "Authorization: Bearer $DASHSCOPE_API_KEY" \
  --header "Content-Type: application/json" \
  --data '{
        "model": "qwen3-rerank",
        "documents": [
                "文本排序模型广泛用于搜索引擎和推荐系统中，它们根据文本相关性对候选文本进行排序",
                "量子计算是计算科学的一个前沿领域",
                "预训练语言模型的发展给文本排序模型带来了新的进展"
        ],
        "query": "什么是文本排序模型",
        "top_n": 2,
        "instruct": "Given a web search query, retrieve relevant passages that answer the query."
}'
```


### 3.6 科大讯飞 / 讯飞星火 （Spark）

- API key申请地址：https://console.xfyun.cn/services/cbm
- API文档地址：https://www.xfyun.cn/doc/spark/Web.html
- API定价信息：https://xinghuo.xfyun.cn/sparkapi


### 3.7 DeepSeek（深度求索）

- API key申请地址：https://platform.deepseek.com/api_keys
- API文档地址：https://platform.deepseek.com/api-docs/zh-cn/
- API定价信息：https://platform.deepseek.com/api-docs/zh-cn/pricing


### 3.8 硅基流动

- API key申请地址：https://cloud.siliconflow.cn/me/account/ak
- API文档地址：https://docs.siliconflow.cn/cn/api-reference/chat-completions/chat-completions
- API定价信息：https://www.siliconflow.cn/pricing

1. 创建对话请求（Anthropic）

```bash
curl --request POST \
  --url https://api.siliconflow.cn/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "model": "Pro/zai-org/GLM-4.7",
    "messages": [
      {"role": "system", "content": "你是一个有用的助手"},
      {"role": "user", "content": "你好，请介绍一下你自己"}
    ]
  }'
```

2. 创建对话请求（Anthropic）

```bash
curl --request POST \
  --url https://api.siliconflow.cn/v1/messages \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "model": "Pro/zai-org/GLM-4.7",
    "messages": [
      {"role": "system", "content": "你是一个有用的助手"},
      {"role": "user", "content": "你好，请介绍一下你自己"}
    ]
  }'
```

3. 创建嵌入请求

```bash
curl --request POST \
  --url https://api.siliconflow.cn/v1/embeddings \
  --header 'Authorization: Bearer <token>' \
  --header 'Content-Type: application/json' \
  --data '
{
  "model": "BAAI/bge-large-zh-v1.5",
  "input": "Silicon flow embedding online: fast, affordable, and high-quality embedding services. come try it out!"
}
'
```

4. 创建重排序请求

```bash
curl --request POST \
  --url https://api.siliconflow.cn/v1/rerank \
  --header 'Authorization: Bearer <token>' \
  --header 'Content-Type: application/json' \
  --data '
{
  "model": "BAAI/bge-reranker-v2-m3",
  "query": "Apple",
  "documents": [
    "apple",
    "banana",
    "fruit",
    "vegetable"
  ]
}
'
```