# AI Agent 运行原理
## AI Agent 是一个后端服务模块通常包含
- 理解和推理模块 (通常是核心 LLM)： 分析输入，理解意图，进行知识检索和推理。
- 规划模块： 制定实现目标的步骤和策略，可能包括决定何时以及如何使用工具。
- Tool Agent (或工具调用模块)： 负责管理可用工具的信息，生成 - tool_call 指令，并与外部工具进行交互。
- 行动模块 (如果适用)： 执行不涉及外部工具的其他操作。
- 生成模块： 根据推理结果和工具的返回，生成最终的输出或回复。
## MCP 正是利用了工具模块统一上下文


举例 以下是AiAgent 与GPT 调用外部数据请求示例

用户提问: "工号是 123 的员工叫什么，邮箱是多少？"
AIAgent 接收需求后

调用LLM 调用格式，如果AIAgent配置了tool agent 就可以每次对话携带上，没有tools就是空，就没有调用外部数据或者工具的能力 
```json
{
  "model": "gpt-4",
  "messages": [
    {"role": "user", "content": "工号是 123 的员工叫什么，邮箱是多少？"}
  ],
  "tools": [
    // 可以定义更多工具
    {
      "type": "function",
      "function": {
        "name": "query_employee_database",
        "description": "根据员工工号查询员工的姓名和邮箱地址。",
        "parameters": {
          "type": "object",
             "properties": {
                "employee_id": {
                    "type": "integer",
                    "description": "要查询的员工的唯一工号。"
                }
            },
          "required": ["employee_id"]
        }
      }
    }
  ],
  "tool_choice": "auto" // 或指定要使用的工具
}
```
LLM会返回类似以下内容，这里返回AiAgent判断有 tool_call 就触发自己的tool_call 功能,就先不把内容返回给用户
```json
{
  "response": "好的，我需要查询一下数据库。",
  "tool_call": {
    "id": "call_abc123",
    "name": "query_employee_database",
    "arguments": {
      "employee_id": 123
    }
  }
}
```
tool_call 就是AiAgent 里面自己要去实现的功能 比如python 
```python
import sqlite3
import json

def query_employee_database(employee_id):
    """查询员工数据库，返回姓名和邮箱。"""
    conn = sqlite3.connect('internal_database.db')  # 假设数据库文件名为 internal_database.db
    cursor = conn.cursor()
    cursor.execute("SELECT name, email FROM employees WHERE id=?", (employee_id,))
    result = cursor.fetchone()
    conn.close()
    if result:
        return {"name": result[0], "email": result[1]}
    else:
        return {"error": "Employee not found"}

这里可以根据tool_call 的name当成函数名直接调用，也可以用作一个key 自己去做对应函数匹配，怎么做都可以
```
tool_call 根据参数 查询好内容后比如结果是 {"name": "张三", "email": "zhang.san@example.com"}
AiAgent 继续调用大模型
```json 
{
  "model": "GPT-4",  // 替换为你要使用的模型名称，例如 "gpt-4", "claude-3-opus", "gemini-pro" 等
  "messages": [
    {
      "role": "user",
      "content": "工号是 123 的员工叫什么，邮箱是多少？"
    },
    {
      "role": "assistant",
      "content": "好的，我需要查询一下数据库。",
      "tool_calls": [
        {
          "id": "call_abc123",
          "type": "function",
          "function": {
            "name": "query_employee_database",
            "arguments": {
              "employee_id": "123"
            }
          }
        }
      ]
    },
    {
      "role": "tool",
      "tool_call_id": "call_abc123",
      "content": "{\"name\": \"张三\", \"email\": \"zhang.san@example.com\"}"
    }
  ],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "query_employee_database",
        "description": "根据员工工号查询员工的姓名和邮箱地址。",
        "parameters": {
          "type": "object",
          "properties": {
            "employee_id": {
              "type": "integer",
              "description": "要查询的员工的唯一工号。"
            }
          },
          "required": ["employee_id"]
        }
      }
    }
    // ... 其他你可能使用的工具定义
  ],
  "temperature": 0.2,  // 可选：控制生成文本的随机性
  "max_tokens": 200,   // 可选：限制模型生成回复的最大长度
  // ... 其他模型特定的参数
}
```
从这里调用可以看出 模型的role 就分user | assistant | tool
大模型最终返回 
{
  "role": "assistant",
  "content": "工号是 123 的员工是张三，他的邮箱是 zhang.san@example.com。"
}
然后AiAgent 再将这个结果返回给用户

也就是用户 问话 得到最终结果，中间过程是AiAgent自己去完成，这个toolAgent 现在就统一MCP Client与Server来完成了，
所以也并不是LLM自身去调用tool_call,只是他根据当前传进去的tools 他会判断用哪个tool 整理好参数后返回，AIAgent再匹配tool call 进行调用后将数据返回给LLM再整理出最终回复


## MCPHOST 里面关于MCP配置
https://github.com/mark3labs/mcphost/blob/main/contribute/conf/demo.json

```json
{
    "mcpServers": {
        "filesystem": {
            "command": "npx",
            "args": [
                "-y",
                "@modelcontextprotocol/server-filesystem",
                "./"
            ]
        }
    }
}
```
我配置了一个关于查看本地文件的mcp server
可以看到 工具的选择

- 判读是否tools回复 https://github.com/mark3labs/mcphost/blob/main/pkg/llm/ollama/provider.go#L54
- LLM请求携带tools https://github.com/mark3labs/mcphost/blob/main/pkg/llm/ollama/provider.go#L125
