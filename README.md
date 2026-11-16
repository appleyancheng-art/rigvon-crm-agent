# 睿光 CRM · Agent 接入包

将 MCP 连接与业务 Skill 一起交给你的 Agent。仓库仅包含公开接入资料，不包含 CRM 源码、客户数据或真实 Token。

在 CRM 的 **工作首页上方的「AI 能力」 → 复制接入指令** 复制说明，发送给支持工具配置的 Agent。由它读取最新版 [AGENT_SETUP.md](AGENT_SETUP.md)，完成客户端检查、Skill 安装和连接配置。首次授权仍可能需要你在连接器设置中填写 Token；这段指令不是保证所有客户端全自动完成的安装命令。

**Grok Bot 桌面端已于 2026-11-14 实测接通并调用 crm_capabilities。** 接入路径为“连接应用 → 添加自定义连接器 → 远程 HTTPS”，Token 填在该连接器的 HTTP 标头中。安全输入框单独显示“已保存”不代表认证已经配置，具体步骤见 [Grok Bot 接入与排错](rigvon-crm/references/connection.md)。版本或界面不同需重新核对，不能把其他模型 API 的支持情况当作桌面客户端支持情况。

- MCP：`https://crm.rigvon.cn/api/mcp`，Streamable HTTP / Bearer Token。
- 业务 Skill：[rigvon-crm/SKILL.md](rigvon-crm/SKILL.md)。需连同 references 目录安装。
- 自动安装说明：[AGENT_SETUP.md](AGENT_SETUP.md)。
- 机器可读清单：[manifest.json](manifest.json)。
- 手动接入与排错：[接入说明.md](接入说明.md)。

客户端必须支持远程 MCP 和自定义认证 Header。具备文件/配置工具的 Agent 可以自动安装；普通聊天窗口和受限 Bot 不一定具有这些能力。Skill 安装与 MCP 连接分别验证，不能只读了文档就报告成功。

分享给别人时只分享本仓库或不含凭据的接入指令。由 CRM 管理员为获授权的使用者/连接创建独立 Token，不分享自己的 Token。Grok Bot 同一账号的多个 Bot 可能共享连接器，不能仅凭 Bot 名字断言凭据和权限已隔离。

能力由 CRM 服务端实时返回。开放全业务操作，禁止源码、SQL、部署、账号及 Token 管理；关键操作仍需 CRM 管理员确认。接入测试只读取能力清单，不改业务数据。

## 当前接入包 v1.3.0 · 2026-11-16

新增[报价配置与图纸盘点](rigvon-crm/references/quoting.md)：有图按空间盘点，无图按需求做明确标注的初步配置；按系统/楼层归类，检查协议与配套，保存后回读核对。费用和成本继续由 CRM 处理。

已接入的 WorkBuddy 或其他 Agent：在 CRM「AI 能力」点击「复制更新指令」，更新完整 Skill（含 references/quoting.md）并只读验证 crm_capabilities。MCP 业务能力仍为 v1.0.0，地址、Token 和权限不变，无需重新生成 Token。
