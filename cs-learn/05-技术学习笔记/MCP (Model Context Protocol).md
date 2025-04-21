
https://www.philschmid.de/mcp-introduction
### MCP 基本概念

Model Context Protocol（简称MCP）是一种由Anthropic开源的开放标准协议，旨在规范大型语言模型（LLM）与外部工具、数据源、API等系统之间的动态交互。它通过定义一套统一的通信协议和架构，使得AI模型能够安全、高效、实时地访问和操作外部资源，从而实现更复杂、更灵活的“做事情”能力


### MCP 为什么被设计？为了解决什么痛点？

MCP（Model Context Protocol）被设计的初衷，是为了解决当前大型语言模型（LLM）和AI代理在访问和利用外部数据源、工具及服务时面临的核心痛点。
#### 1. 解决AI模型与外部系统集成的碎片化和复杂性


####  2. 实现动态、实时且上下文感知的数据访问

####  3.提供标准化的上下文管理与模块化提示




![[Pasted image 20250418003342.png]]

### MCP 核心架构与组成


MCP基于经典的客户端-服务器（Client-Server）架构，主要包括以下关键角色：

- **Host（主机）**：运行AI模型的应用程序，如聊天机器人、IDE助手等。Host负责管理多个Client，协调模型与外部系统的交互，处理用户授权和上下文聚合[3](https://composio.dev/blog/what-is-model-context-protocol-mcp-explained/)。
    
- **Client（客户端）**：每个Client与一个MCP服务器建立一对一的持久连接，负责消息路由、能力管理、协议协商及订阅管理。Client是Host与Server之间的桥梁，确保通信的安全和高效[2](https://www.philschmid.de/mcp-introduction)[3](https://composio.dev/blog/what-is-model-context-protocol-mcp-explained/)。
    
- **Server（服务器）**：提供具体的工具（Tools）、资源（Resources）和提示模板（Prompts），供模型调用。工具通常是模型可执行的功能（如调用天气API），资源则是只读的数据源（类似REST API的GET端点），提示模板帮助模型以最佳方式使用工具和资源[2](https://www.philschmid.de/mcp-introduction)[3](https://composio.dev/blog/what-is-model-context-protocol-mcp-explained/)。
    
- **基础协议（Base Protocol）**：定义Host、Client、Server之间的通信规则，采用JSON-RPC 2.0作为消息格式，支持请求（Request）、响应（Response）和通知（Notification）三种消息类型，确保通信的标准化和互操作性[3](https://composio.dev/blog/what-is-model-context-protocol-mcp-explained/)。
- 
![[Pasted image 20250418002154.jpg]]





### MCP 通信机制