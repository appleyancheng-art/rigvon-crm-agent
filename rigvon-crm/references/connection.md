# 连接与凭据

MCP 地址：`https://crm.rigvon.cn/api/mcp`。传输：Streamable HTTP。认证：`Authorization: Bearer <CRM Token>`，Bearer 后为一个半角空格。当前不是 OAuth 服务，不填写 OAuth 客户端 ID/Secret。

## 先准备，再提供 Token

首次接入先安装完整 Skill、确认当前客户端的 MCP 配置权限和认证绑定方式；此时不需要 Token。已有可用连接直接复用。准备好且缺少有效凭据时，向用户提示“已准备好，请发送这个 AI 对应的 CRM Token”。用户可在当前获授权的 AI 对话中单独提供最新 Token，由 Agent 完成连接配置。没有可用配置接口时先说明限制和最少的手动步骤，不提前索要无法使用的凭据。

只使用用户本次提供的 Token，或已配置连接的有效凭据；不从旧聊天找 Token，不擅自启用已停用助手。Token 来自 CRM 中已启用、未过期的助手，重置后旧值立即失效。已接入用户更新 Skill、执行业务或下载时不必再提供 Token。CRM 服务端凭据的创建、启停及重置仍由管理员在 AI 能力页管理。

凭据优先存入能够实际绑定 Authorization Header 的客户端凭据存储。没有可用绑定但支持私有用户配置时，可将本次提供的值写入当前用户专用配置文件；Unix 权限设为 0600，其他系统使用等价的用户访问控制。按结构化格式合并并保留其他配置，备份也应私有。不写进共享项目配置、仓库、Skill、URL、命令行参数、日志或回复；报告只说明位置与状态，检查输出需脱敏。用户主动在本次对话提供 Token 可用于此次安装，但不能据此承诺平台不会保留聊天记录。

只有实际运行客户端能读取的环境变量才可作为认证来源；在另一个终端临时 export 不代表桌面客户端已获得 Token。Secret/Vault 显示“已保存”也不证明认证 Header 已绑定。配置已保存但工具尚未调用通过时，报告“待验证”。完整客户端安装步骤见官方仓库的 [AGENT_SETUP.md](https://github.com/Appleyancheng-art/rigvon-crm-agent/blob/main/AGENT_SETUP.md)。

## Grok Bot 桌面端实测路径

2026-11-14 已实测连接并实际调用 crm_capabilities。以下为当天桌面端界面；未来版本以实际可见入口和官方说明为准。

1. 打开“连接应用 / Connect apps”，检查“你的插件”中的已有 CRM 连接。可用则复用，避免同名或重复连接。
2. 新连接使用“添加自定义连接器 / Add a custom connector”。名称可用“睿光 CRM”，类型选择“远程 HTTPS / Remote HTTPS”。
3. 服务器 URL 填 `https://crm.rigvon.cn/api/mcp`。
4. HTTP 标头的“键”填 `Authorization`，“值”填 `Bearer ` 加最新 CRM Token。Agent 有配置能力时自动填写；平台要求用户完成时才交由用户填写。客户端 ID 留空。不要把 `RIGVON_CRM_TOKEN`、`${RIGVON_CRM_TOKEN}` 或 `<Token>` 原样填进桌面端的值字段；这里未验证环境变量展开功能。
5. 保存后确认连接器显示“已连接”且工具已加载。给 Bot 发消息，明确使用这个连接器实际调用 crm_capabilities。必要时开始新消息；不需要为填错或停用 Token 反复重启。
6. 核对返回的 Agent 名称与授权助手一致，分别报告 MCP 与 Skill 的验证结果。成功前只做接入检查，不改业务数据。

**安全输入框与认证 Header 是两处配置。** Grok Bot 的安全输入框/Vault 显示“已保存”，并不能证明连接器会发送该密钥。若 Agent 无法绑定认证 Header，应先说明限制并准备好上述非敏感配置，让用户只完成剩余字段；不要先索要无法使用的 Token，不要反复开安全输入框或编造 Secret 引用格式。

已有连接凭据失效时，优先使用当前版本提供的编辑/重新配置入口。若界面没有 Header 编辑能力，向用户说明需要重新配置；未经同意不移除其他连接。确需另建临时连接时用明确名称，验证后交代旧连接仍存在，不把重复项冒充已清理。

Grok Bot 同一账号的多个 Bot 可共享连接器；这不是逐 Bot 隔离的证明。不同人员应使用各自获授权的账号/连接及独立 CRM Token。

官方参考：[Grok Bot：使用电脑和应用](https://docs.x.ai/grok-bot/computer-and-apps)。

## 成功标准

- 在实际使用的客户端加载到了 CRM 工具，不能只凭保存配置、读取 Skill 或单独 curl 成功。
- 实际调用 crm_capabilities 成功，核对返回的 Agent 名称、模块和边界。
- Skill 的加载/发现单独验证，安装目录需保留 references；不支持 Skill 的客户端应明确报告该限制。

## 排错顺序

- **401**：检查 Header 拼写、Bearer 后空格、Token 是否为最新、助手是否启用、是否过期。核对 CRM 显示的助手名称和凭据前缀即可，不输出完整 Token。过期时重置不会延长期限；由管理员处理。
- **405**：先区分客户端最终提示和服务器实际响应。一次 Grok Bot 实测中，客户端显示 405，服务端最先收到的 POST 却是 401；更换有效 Token 后即连接成功。因此不能仅凭 405 判断要改 SSE、OAuth 或服务器代码。真正对本入口发送 GET 时，当前 JSON 模式可返回 405；正常 MCP 请求用 POST。无法查看服务端日志时保留不确定性，先检查凭据，不盲目改接口。
- **403**：可能为授权角色变化、来源不符或业务操作缺少审批；核对原因，不扩大权限绕过。
- **429**：降低调用频率，稍后再试。
- **409**：重新读取记录版本并合并改动，不能覆盖他人的修改。
- **工具不可用/0 个工具**：连接未完成，不能宣称 crm_capabilities 验证通过。先修复连接配置，再做只读核验。

完整接入包：https://github.com/Appleyancheng-art/rigvon-crm-agent ，根目录 AGENT_SETUP.md 说明各客户端安装方式。新用户需由 CRM 管理员授予独立访问凭据，分享接入包不等于授权访问。


## 下载时不需要读取连接器 Token（能力 v1.1.0 起）

Grok Bot 等客户端常把 MCP 的认证 Header 保存在连接器内部，普通下载工具无法读取它，这是正常的。已接通 MCP 后调用 crm_export_link，服务器会返回完整 HTTPS 下载链接和 expiresAt。直接访问 url，无需 Bearer Header 或 CRM 登录；不要再索要 Token。ticket 仅授权本次指定文件、最长 15 分钟，可用于重试与断点接收；停用、删除或重置助手后失效。过期重新调用 crm_export_link。

旧版返回相对地址并要求 Bearer 时，先刷新 MCP 工具列表/更新 Skill，核对 crm_capabilities 是否为 v1.1.0 或更高。只有在实际客户端必须重新加载工具时才提示重载。不要把“无法看见 Token”当作“不能使用已认证 MCP”。
