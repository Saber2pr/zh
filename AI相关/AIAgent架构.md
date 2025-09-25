![https://saber2pr.top/MyWeb/resource/image/agent-v3.drawio.svg](https://saber2pr.top/MyWeb/resource/image/agent-v3.drawio.svg)

相关技术：
1. RAG：qdrant，@qdrant/js-client-rest
2. MCP：@modelcontextprotocol/sdk
3. Agent UI：@assistant-ui/react

--- 
1. 这里 Agent UI 一般和 MCP Server 为本地部署，方便执行本地原生操作
2. RAG 一般为服务部署，建设私有知识库

---
完整做的事情：
1. 先搭建 Chat UI，用 @assistant-ui/react 快速出一个对话框，对接一个 LLM 接口
2. 搭建知识库平台，用 qdrant 镜像起一个服务，用 @qdrant/js-client-rest 实现文档增删查改接口，用 LLM 实现文本向量化
3. Chat UI 接入知识库召回，自动携带背景知识
4. 搭建 MCP Server，实现 知识库工具，和各种原生工具
5. Chat UI 接入 MCP Server，先请求工具列表，然后加入到提示词里，同时约束 LLM 输出为 json。json里包含 response 对话内容 + tool_call 执行工具的json。

---
这样基本上就实现了一个“私有”的 Chatgpt，同时还能不断补充私有知识，还可以执行一些工具