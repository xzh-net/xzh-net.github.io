# OpenAI

API角色

| 角色          | 发送者      | 用途                                              | 必需性           | 典型内容格式   |
| :------------ | :---------- | :------------------------------------------------ | :--------------- | :------------- |
| **system**    | 开发者/用户 | 设定AI的长期身份、行为准则、上下文规则            | 可选但推荐       | 纯文本         |
| **user**      | 最终用户    | 用户的问题、指令、请求                            | 必需             | 纯文本         |
| **assistant** | AI模型      | 1. 直接回答 2. 请求工具调用 3. 基于工具结果的回答 | 多轮对话中必需   | 见示例 |
| **tool**      | 外部系统    | 工具/函数的执行结果                               | 工具调用场景必需 | JSON字符串     |

决策树

```lua
收到用户问题
    ↓
是否需要外部工具/数据？
    ├── 否 → 格式1：纯文本回复
    │       {"role":"assistant","content":"直接回答"}
    │
    └── 是 → 格式2：工具调用
            {"role":"assistant","content":null,"tool_calls":[...]}
                ↓
            执行工具，返回格式：
            {"role":"tool","tool_call_id":"...","content":"{...}"}
                ↓
            AI处理结果，返回格式3：
            {"role":"assistant","content":"整合结果..."}
```

对话流程示例数据

```json
{
	// 1. 系统设定
	{
		"role": "system",
		"content": "你是一个天气助手，帮助用户查询天气。请根据工具返回的信息回答用户。"
	},

	// 2. 用户提问  
	{
		"role": "user",
		"content": "北京今天的天气怎么样？"
	},

	// 3. AI返回工具调用请求（assistant消息，含tool_calls）
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

	// 5. AI基于结果回复
	{
		role: "assistant",
		content: "北京天气晴朗，气温25°C。" // ← 正常文本，无tool_calls
	}
}
```