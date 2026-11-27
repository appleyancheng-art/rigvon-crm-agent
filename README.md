# 睿光 CRM · Agent 接入包

将 MCP 连接与业务 Skill 一起交给你的 Agent。仓库仅包含公开接入资料，不包含 CRM 源码、客户数据或真实 Token。

在 CRM「AI 能力」点击 **「复制一键接入指令」**，先把不含 Token 的指令发给新 AI。它会读取官方接入包，检查客户端、安装 Skill 并准备连接。等它提示“已准备好，请发送这个 AI 对应的 CRM Token”时，你再创建对应助手或使用已有有效凭据，把 Token 单独发进同一个对话框；AI 完成配置后实际调用 `crm_capabilities` 验证。

具备配置能力的 AI 会自动完成这些步骤；平台要求用户手动建立连接时，AI 会先说明限制和剩余步骤，不提前索要无法使用的 Token。详细流程见 [AGENT_SETUP.md](AGENT_SETUP.md)。已接入的 AI 使用「复制更新指令」，保留原连接和有效 Token，无需重新部署。

- MCP：`https://crm.rigvon.cn/api/mcp`，Streamable HTTP / Bearer Token。
- 业务 Skill：[rigvon-crm/SKILL.md](rigvon-crm/SKILL.md)。需连同 references 目录安装。
- 自动安装说明：[AGENT_SETUP.md](AGENT_SETUP.md)。
- 机器可读清单：[manifest.json](manifest.json)。
- 手动接入与排错：[接入说明.md](接入说明.md)。

客户端必须支持远程 MCP 和自定义认证 Header。具备文件/配置工具的 Agent 可以自动安装；普通聊天窗口和受限 Bot 不一定具有这些能力。Skill 安装与 MCP 连接分别验证，不能只读了文档就报告成功。

分享给别人时只分享本仓库或不含凭据的接入指令。由 CRM 管理员为获授权的使用者/连接创建独立 Token，不分享自己的 Token。Grok Bot 同一账号的多个 Bot 可能共享连接器，不能仅凭 Bot 名字断言凭据和权限已隔离。

能力由 CRM 服务端实时返回。开放全业务操作，禁止源码、SQL、部署、账号及 Token 管理；删除、清空、作废和移除已有明细仍需 CRM 管理员确认。接入测试只读取能力清单，不改业务数据。

## v1.3.0 历史更新 · 2026-11-16

新增[报价配置与图纸盘点](rigvon-crm/references/quoting.md)：有图按空间盘点，无图按需求做明确标注的初步配置；按系统/楼层归类，检查协议与配套，保存后回读核对。费用和成本继续由 CRM 处理。

已接入的 WorkBuddy 或其他 Agent：在 CRM「AI 能力」点击「复制更新指令」，更新完整 Skill（含 references/quoting.md）并只读验证 crm_capabilities。该次仅更新报价指南，未调整权限；后续 v1.4.0 的接入与下载方式见下方。


## v1.4.0 历史更新（2026-11-16）

配套 MCP 能力 v1.1.0：日常业务直接执行，删除类操作继续确认；下载报价/PDF/Excel 使用服务器签发的短时文件链接，无需获取连接器 Token。v1.3.0 的报价配置与图纸盘点指南完整保留。

已接入的 Agent 使用 CRM「复制更新指令」更新 Skill 并刷新 MCP 工具列表；无需重新生成 Token。只读连接验证不自动制作测试单据。

## v1.5.0 历史更新（2026-11-16）

配套 MCP 能力 v1.2.0：新增报价整单批量分组与局部移组。建组、分配和排序原子执行；只修改分组和顺序，保留数量、价格、折扣及服务费。整单默认完整覆盖，局部保留未指定条目，重复请求沿用幂等机制。原有便捷下载和日常业务权限继续保留。

在 CRM「AI 能力」复制更新指令给 Agent，更新完整 Skill，并只读验证 crm_capabilities 与两个分组操作的 schema；旧 Token 继续使用。

## 历史版本：Skill v1.6.0 / MCP v1.3.0（2026-11-22）

补齐台账收款数组与联系人、字段校验定位、结算一次建单与按ID局部更新、稳定明细ID、执行回执最新版本/计算结果、审批原因与撤回、跨组批量添加、库存转报价产品、图片URL转存和手动分类保留。搜索区分命中/全库统计；保留下载短时链接、现有权限和审批规则。

CRM 同步优化字段级 AI 来源、手机列表、深色显示与系统保存文件入口；标记仅内部显示。复制“更新指令”给 Agent，刷新工具后实际调用 crm_capabilities 验证 v1.3.0，再只读 describe 验证 patchSettlementAction、createHistoricalSaleAction、updateProductAction 等新契约。已有 Token 继续使用，无需重置。

## Skill v1.7.0 / MCP v1.4.0（2026-11-23）

新增独立收据管理模块：查询收据及实际收款来源、关联开具、无需客户或台账的独立开具、作废、删除、PDF/Excel 下载。见 [收据指南](rigvon-crm/references/receipts.md)。开具不重复记账，删除不影响原收款且需 CRM 审批。导出标题统一为“收据”。复用现有连接与 Token，更新完整 Skill 后，只读验证 crm_capabilities 与收据操作 schema。

## Skill v1.8.0 / MCP v1.5.0

方案设计支持客户关联、A3 图纸标绘和产品线路计量。保存前读取最新修订，保留标注及点位属性。米制线路校准或确认手工数量后可生成报价。

## Skill v1.9.0 / MCP v1.6.0

报销支持未报销、已发送、已报销三个状态。已发送表示资料已提交，金额仍计入待报销。更新 Skill 并刷新工具后，可通过 `crm_describe_operation` 核对单笔新增和修改的状态选项。

## Skill v1.10.0 / MCP v1.7.0

方案支持点位底图、逐层类别可见性和线路缩放。保存方案前先读取完整 document 和 revision，隐藏类别仍计入设备清单和报价。清空方案点位需确认。已接入的 Agent 用「复制更新指令」更新 Skill 并刷新工具，通过 crm_describe_operation 查看 saveDesign 的图纸字段。

## Skill v1.10.1 / MCP v1.7.1（2026-11-27）

修复项目核算嵌套数量、提成比例及负责人未保存的问题；统一校验、按稳定标识合并明细，省略值保留，保存后在事务内回读校验，失败整单回滚。修复已有送货单更新时缺失版本，自动从通过 preconditions 的详情传递 updatedAt。操作示例见 [业务流程](rigvon-crm/references/workflows.md)。现有 Token 继续使用，在「AI 能力」复制更新指令更新完整 Skill、刷新工具，只读核对 crm_capabilities 为 1.7.1 及两个保存操作的 schema。

## Skill v1.11.0 / MCP v1.7.1（2026-11-27）

接入改为先发送通用指令、AI 准备好后再提供 Token。允许 Agent 使用用户在本次授权对话中提供的 Token 配置当前客户端；优先凭据存储，支持用户私有配置作为备选。分别验证 Skill 加载和实际 MCP 调用。公开指令及下载包不含凭据，CRM 业务权限和服务端 Token 管理保持不变。已有 Agent 复制更新指令即可沿用新指南，无需换 Token 或重建连接。

## Skill v1.12.0 / MCP v1.8.0（2026-11-27）

方案设计增加灯具固定符号与安装面配色、点位旋转、分类独立底图。网页支持普通滚轮缩放、独立属性面板、两种确认清空和保存冲突合并。保存契约及计米规则见 rigvon-crm/references/workflows.md。已有连接和有效 Token 继续使用；复制更新指令更新完整 Skill、刷新 MCP，再只读调用 crm_capabilities 和 crm_describe_operation(saveDesign) 核对新版本与字段。

本版方案支持磁吸轨道画线、机柜模型、柜内/配套设备清单，以及每件设备独立的我方/他方供货范围。默认我方供货；他方供货保留图纸、排除报价，关联电源仍单独计价。图框采用用户提供的睿光智能品牌原图。


## Skill v1.13.0 / MCP v1.9.0（2026-11-27）

方案新增点位 `iconScale`，画布与 PDF 同步尺寸，报价数量不变。网页支持当前标签页打开、中键平移、拖动旋转、单端延长、自动对齐吸附、组件选型及文字再编辑。已有连接和有效 Token 继续使用；复制更新指令更新 Skill，刷新工具并只读核对 crm_capabilities 与 saveDesign schema。

## Skill v1.15.0 / MCP v1.10.0（2026-11-27）

方案改为分类图纸，PDF 不再追加楼层总图；勾选分类决定图纸显示与导出，隐藏不减少设备报价。修复嵌套 CAD 图框裁剪及慢网底图加载，新增 Keynote 式鼠标、键盘与双指操作。已有连接和 Token 继续使用；发送复制更新指令更新 Skill 并刷新工具。详情见 [方案工作流程](rigvon-crm/references/workflows.md)。

## Skill v1.16.0 / MCP v1.11.0（2026-11-27）

报价 PDF/Excel 新增“仅显示折后总额”：保留设备与数量，隐藏单价、小计、原总价及费用拆分；Excel 同时移除内部核算和隐藏价格数据。合同附带报价可逐项选择同组显示选项，保存后应用到附件快照。客户定稿入口保留在客户详情，客户列表不再显示。Option 拖拽复制时原图标保持原位。

已有连接和 Token 继续使用。复制 CRM 的更新指令，更新完整 Skill 并刷新工具；用 `crm_capabilities` 核对 MCP v1.11.0，再调用 `crm_describe_operation(saveContractAction)` 核对 `attachmentQuoteExportOptions`。导出示例见报价指南，合同保存示例见业务流程。

## 当前版本 Skill v1.17.0 / MCP v1.12.0（2026-11-27）

报价 PDF、Excel 及合同报价附件改为独立的正向显示选项：单价、原价、折后总额、设备合价、房间小计、折扣、服务费及调整明细，勾选才显示。Excel 的内部核算另行控制，未选的结构化价格不会写入隐藏单元格。旧 onlyFinalTotal 参数仍兼容并展开为独立选项。横版合同沿用竖版的封面图片、字体和黑金配色；企业合同使用公司注册地址、税号和对公银行账号。

已有 Token 和连接继续使用。更新完整 Skill 并刷新工具；调用 `crm_capabilities` 核对 1.12.0，调用 `crm_describe_operation(saveContractAction)` 核对附件显示字段。详见报价指南及业务流程。
