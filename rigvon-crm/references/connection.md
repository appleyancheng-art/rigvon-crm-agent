# 连接与凭据

MCP 地址为 https://crm.rigvon.cn/api/mcp ，使用 Streamable HTTP 和 Authorization Bearer Token。Token 从客户端安全配置读取，不能放在聊天、URL、Skill 或共享仓库中。

首次加载调用 crm_capabilities。如果工具不可用，先检查客户端 rigvon-crm MCP 连接与凭据，不能仅凭读到本 Skill 就宣称已连接。401 表示凭据错误、停用或过期；403 表示授权或审批不足；429 应降低频率；409 需重新读取记录版本。

Skill 不管理 Token；凭据在 CRM 的 AI 支持页面由管理员创建、停用和重置。完整接入包见 https://github.com/Appleyancheng-art/rigvon-crm-agent ，安装说明为根目录 AGENT_SETUP.md。
