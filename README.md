# 睿光 CRM · Agent 接入包

将 MCP 连接与业务 Skill 一起交给你的 Agent。仓库仅包含公开接入资料，不包含 CRM 源码、客户数据或真实 Token。

在 CRM 的 **系统管理 → AI 支持 → 复制接入指令** 复制说明，发送给支持工具配置的 Agent。由它读取 [AGENT_SETUP.md](AGENT_SETUP.md) 自动配置；你只需在客户端密钥设置提供 CRM 生成的独立 Token，并在客户端要求时重新加载。

- MCP：`https://crm.rigvon.cn/api/mcp`，Streamable HTTP / Bearer Token。
- 业务 Skill：[rigvon-crm/SKILL.md](rigvon-crm/SKILL.md)。需连同 references 目录安装。
- 自动安装说明：[AGENT_SETUP.md](AGENT_SETUP.md)。
- 机器可读清单：[manifest.json](manifest.json)。
- 手动接入与排错：[接入说明.md](接入说明.md)。

客户端必须支持远程 MCP 和自定义认证 Header。具备文件/配置工具的 Agent 可以自动安装；普通聊天窗口和受限 Bot 不一定具有这些能力。Skill 安装与 MCP 连接分别验证，不能只读了文档就报告成功。

能力由 CRM 服务端实时返回。开放全业务操作，禁止源码、SQL、部署、账号及 Token 管理；关键操作仍需 CRM 管理员确认。接入测试只读取能力清单，不改业务数据。
