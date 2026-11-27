# 常用业务映射

所有名称均是 `crm_describe_operation` / `crm_prepare_operation` 的 operation 值，不是单独的 MCP 工具名。只读业务操作通过 crm_read_operation 调用。

| 任务 | 操作 |
|---|---|
| 新客户 / 修改资料 / 跟进 | createCustomerAction / updateCustomerAction / addFollowUpAction |
| 新产品 / 修改产品与图片 | createProductAction / updateProductAction |
| 导入产品 | previewProductImportAction → getProductImportBatch → applyProductImportAction |
| 导入客户 | previewCustomerImportAction → getCustomerImportBatch → applyCustomerImportAction |
| 新报价 / 模板建单 / 复制 | createQuoteFromListAction / createQuoteFromTemplateAction / duplicateQuoteVersionAction |
| 房间、设备 | createQuoteRoomAction / addQuoteItemAction / updateQuoteItemAction / replaceQuoteItemAction |
| 排序 | reorderQuoteRoomsAction / reorderQuoteItemsAction |
| 新结算 / 完整保存 / 局部改单 | createSettlementAction / saveSettlementAction / patchSettlementAction |
| 结算确认 / 状态 | confirmSettlementAction / changeSettlementStatusAction |
| 单笔报销 / 批量 / 进货订单 | createReimbursementAction / createExpenseBatchAction / savePurchaseOrderAction |
| 新方案 / 保存 / 生成报价 | createDesign / saveDesign / createQuoteFromDesign |
| 上传底图 / 保存版本 | uploadDesignAsset / createDesignVersion |
| 合同 | createContractAction / saveContractAction |
| 销售台账 | createHistoricalSaleAction / updateHistoricalSaleAction |
| 售后 / 送货 | createAfterSaleAction / saveAfterSaleAction / saveDeliveryAction |
| 库存入库 | receiveInventoryAction / receiveInventoryBatchAction |
| 业务设置 | updateCompanyProfileAction / updateBudgetRuleAction / saveTemplateEditorAction |

## 报价

配置方法、PDF 盘点、系统分组及保存后的检查见 [报价配置与图纸盘点](quoting.md)。QuoteRoom 是平级分组；空间盘点与报价分组分开处理。已有价格/服务费配置保持不变，金额使用 CRM 计算。

## 新增报销示例

单笔新增、修改、批量录入及进货订单共用三个状态，分别为 `未报销`、`已发送`、`已报销`。资料已提交、仍待报销时选 `已发送`。筛选这类明细时调用 `crm_search`，传入 `{"module":"reimbursements","status":"已发送"}`。总览中的待报销笔数和金额包括未报销与已发送两组。

批量标记时，先读取所选报销记录，调用 `batchReimbursementStatusAction`，在 `form.status` 填写目标状态，`form.records` 传入由记录的 `id` 和原始 `updatedAt` 组成的数组。每批最多 40 条。记录发生变更时，重新读取后再处理。

先搜索同日期、金额、用途的记录排除重复，查看 createReimbursementAction 的描述，然后准备：

```json
{
  "operation":"createReimbursementAction",
  "requestKey":"expense-unique-intent-001",
  "reason":"按用户提供票据登记办公用品费用，未报销",
  "input":{"form":{"name":"办公用品","category":"办公用品","status":"未报销","amount":"120.00","quantity":"1","expenseDate":"2026-11-14","notes":"按用户提供票据登记"}},
  "preconditions":[]
}
```

日期和金额只是格式示例，不可当作用户真实资料写入。

## 修改已有产品

读取 products 模块产品详情取得 version，保留 brand、category、series、model、quoteName、specification、unit、purchasePrice、suggestedSalePrice、defaultDiscount、notes。
准备 updateProductAction，input 为 `{productId,form:{…}}`，只传需修改字段即可，preconditions 为 products 对应 id/version。不传 image 时保持原图；上传字段名以描述为准，不能猜。

## 保存结算

get_record 返回业务详情。items 的快照字段需转成 editor payload 字段：nameSnapshot→name、brandSnapshot→brand、categorySnapshot→category；保留 inventoryItemId、quantity、unitPrice、includeInServiceFee、includeInDiscount、pricingMode、fixedTotal。自定义项目保留 rowKey=id、isCustom 和 purchasePrice。pricingGroups 的 id 映射为 clientId，同时对应行使用 pricingGroupClientId。支付和调整项保留顺序与来源。必须查看 saveSettlementAction 返回的 payloadSchema，不能直接把数据库详情原样塞入 form。

完整保存按提交列表同步明细；现有设备 rowKey=item.id、组合 clientId=group.id、调整项/收款 id 传回时保留持久 ID；未要求删除的行必须保留。新增自定义设备使用唯一 rowKey，并按用户提供资料填写，不能冒充库存设备。

## 方案

createDesign 创建空方案后读取 designs 详情，在 document 基础上修改；保存需要 id、revision、title、customerId、document，并提供 MCP 的 preconditions。服务器拒绝其他项目的底图关联。底图先 uploadDesignAsset，form.file 为附件对象。需要图片视觉分析时读取原始图纸；不要仅靠产品文字编造点位。

## 文件

quotes、settlements、receipts 支持 pdf/excel；contracts、designs 支持 pdf；reimbursements 支持 pdf/excel，传 recordIds（1–40 条），images 按需开启。当前不支持的模块/格式不要构造 URL。客户端下载大文件应等待 HTTP 完成，不要因稍慢反复生成。


## 日常业务和下载

读取、新增、修改、批量添加、入库和用户指定的状态流转不额外要求人工审批。READY 后直接执行，保留版本校验和原业务规则。只有删除、清空、作废或完整保存遗漏已有明细时进入确认；PENDING 不可自行批准。

导出用 crm_export_link，随后直接下载其完整 url。短时链接无需额外 Token；请读取 [下载连接说明](connection.md)，不要让 Grok 用户再次提供密钥。

## 台账收款、联系人与日期

创建至少提供 orderDate（YYYY-MM-DD）、status（实际选项）、contractAmount；可同时提交 `receipts:[{label:"定金",amount:200,paidAt:"2026-11-22",note:"",sourceRange:"",mergedSource:false}]`。这是格式示例，不能写入真实业务。更新时每个已有收款保留 id，省略 receipts 表示不改，显式空数组表示清空并需审批。日期按北京时间日历日期传入；布尔接受 true/false，兼容 yes/no、Y/N、1/0。不得伪造付款凭证。

`customerCompany` 是公司，`contactPerson` 是客户联系人，`ownerSales`（兼容 customerOwner）才是业务员。不要把联系人填进业务员。公司客户可以没有电话；warnings 会提醒联系方式待补。完工日期用 completionDate，局部修改不需要重传收款。

台账 receipts 是实收来源；结算 `paymentSource=LEDGER` 自动带出台账收款，MANUAL 使用单据 payments。不要把同一笔同时加两次。结算确认可能将台账收款冻结成单据快照，这是既有规则。

## 结算局部修改、审批与内部备注

```json
{"operation":"patchSettlementAction","input":{"settlementId":"当前单据ID","form":{"payload":{"items":[{"rowKey":"当前明细ID","quantity":3}]}}},"requestKey":"独立意图唯一编号","reason":"按用户指令调整一行数量","preconditions":[{"module":"settlements","id":"当前单据ID","version":"读取返回的64位版本"}]}
```

以上为结构示例，ID/version不能照抄。patch 省略行保留；新增设备给新 rowKey 与完整必填值，新增组合给新 clientId；现有调整项/payments 按 id 更新，新行不传 id。删除仍用完整保存并审批。数量和单价改变后用回执 computedTotals，不手算回填总额。

`saveSettlementInternalNoteAction` 最小输入 `{settlementId,form:{internalNote:"内部说明"}}`，上限300字，超限拒绝，不是截断；客户可见备注是 `form.payload.notes`，不能混用。`changeSettlementPendingBillingAction` 接受 `{settlementId,form:{pendingBilling:true}}`；只改待开单标记，不等于确认结算，也不扣库、不改收款来源。

prepare 的 changes、approvalReasons 和 stockPreview 描述具体改动与库存预警；确认后 nextAction=execute 才执行。废弃请求使用 crm_cancel_operation，只能撤回自己尚未执行的操作。已成功不能撤回，不用反向写入冒充撤回。

## 价格、库存与多分组

`addQuoteItemAction` 是单件；同一组批量用 addQuoteItemsBatchAction（100行），跨组用 addQuoteItemsToGroupsAction 的 `input:{quoteId,groups:[{roomId,items:[{productId,quantity}]}]}`，最多300行，统一原子提交。整理现有分组仍使用 applyQuoteGroupingAction。

产品库查无物料时查询 inventory；库存 ID 不能冒充 productId。使用 createProductFromInventoryAction（itemId、model、unit，缺失价格必须明确提供）生成报价产品，随后使用返回的产品 ID；此操作不改变库存。同品牌型号已存在会明确拒绝重复建档。

`crm_find_catalog_matches` 按 q、brand、series 查候选，返回型号、单位、规格及当前价格；仅相似不能当可替换。对账先指定原报价 ID/版本作为基线，比较其历史快照；缺型号或单位标为不可比，单价差与总额差分开，禁止自动把 Z1 当 Z3。库存 retailPrice=null 是缺价，不能直接拿进价或其他颜色售价填充。服务费默认偏好沿用适用报价模板（模板 serviceFeeRate 为比例），用户明确后可改模板；不全局改现有单据或推定全店25%。

## 产品图片与类别

新增/更新产品支持 `form.imageUrl` 公开 HTTP(S) JPG/PNG/WebP 图片直链，服务端下载、校验并转存。不接受网页链接、登录态图片、内网地址，最多3MB；下载失败不报成功。不传图片保留原图。替换图片属于修改；removeImage=true 删除图片需审批。本地文件仍可传 image 文件对象，但客户端参数较小时先取得允许公开下载的直链，不要把用户私有图片擅自公开上传。

手动 category 保留用户填写值（例如安装配件、门禁系统），不会套用批量导入的旧分类归一化规则。规格常见错字只提示不擅改型号；发布前核对官方资料与实际产品。

## 统计、传输和能力边界

搜索命中依据 total/items 与 statistics.matched；statistics.global 是可用的全库统计，没有计算的金额为 null。原生 summary 保留兼容，各字段统计范围看 statisticsScope。金额和库存预警均基于查询时数据；扣库时还会重新校验。

`crm_export_link` 的绝对 HTTPS 链接最多15分钟，保持完整 ticket，无需 Token。TLS EOF/短暂连接失败可在原链接有效期内有限重试；过期重新取链接。不要绕过证书验证。客户端 toolName 变空、tool_call_replaced、connector not available 若未到服务器，CRM 无法修复客户端；缩小参数、串行调用并报告实际失败环节。服务器会对超限请求返回413、字段错误给出定位。

多 Bot 共用一条连接就是共用同一个授权身份，不是人员隔离。不同人员用各自账号/助手；不要为排错复制、重置或扩展权限。Numbers 附件先按其正式解析器提取，避免逐包扫描猜值。

## 收据

使用 receipts 模块与 [收据管理](receipts.md)；先查实收来源，再开具或独立开具。不要将收据操作误当作新增收款。

## 项目核算明细保存（MCP v1.7.1）

先 `crm_get_record({module:"installation-fees",id})` 取得当前 version、items 和 commissions；人员由 `crm_search({module:"people"})` 查询。推荐局部修改：

```json
{
  "operation": "saveInstallationCalculationAction",
  "input": {
    "id": "<核算单ID>",
    "form": {"payload": {
      "items": [{"id": "<灯带明细ID>", "quantity": 80}],
      "commissions": [{"id": "<方案设计明细ID>", "rate": 0.02, "assigneeId": "<人员ID>"}]
    }}
  },
  "preconditions": [{"module": "installation-fees", "id": "<核算单ID>", "version": "<回读version>"}],
  "requestKey": "<本次意图唯一值>",
  "reason": "按用户要求修正灯带数量与方案设计提成归属"
}
```

明细可用持久 id，也可用 `items[].productType` / `commissions[].role` 或原 sortOrder；标识同时提供时必须一致，重排数组不改变匹配。无标识时必须按系统标准顺序完整提交 8 项品类或 7 项角色，局部无标识行拒绝。品类与角色以详情为准；不要把 `installationFee`、`grossProfit`、`commissionTotal` 等计算结果当作输入。

quantity 非负，最多三位小数。嵌套 rate 为 0~1 比例，最多六位小数；5% 传 0.05。`unit`、`unitPrice`、`sortOrder` 可回传原值，不能改变固定规则。省略行和字段保留；空数组不代表清空。不要同时提供 payload 和其他 form 字段，也不要混用嵌套数组与网页索引字段。兼容网页 `quantity-0`、`rate-0` 等字段，但 `rate-0=5` 表示 5%，建议 Agent 统一使用嵌套契约。

手填提成传 `isManualAmount:true,amount:500`，恢复自动计算传 `isManualAmount:false` 并省略 amount。选择系统人员使用有效 assigneeId，姓名以系统记录为准；非系统人员传 `assigneeId:null,assigneeName:"姓名"`；清空归属两者显式传 null。未提交的手填金额和负责人保持原值。关联核算单的名称及客户来自台账，不能借保存修改关联模式。

READY 后 execute，再 get_record 核对每条改动及 computedTotals。服务端逐项验证数量、提成、人员及金额；回读不一致时整个事务回滚并返回错误，不会 SUCCEEDED。自动金额按原有规则计算、逐行四舍五入后汇总。已确认并带日期的提成进入原有月度汇总。旧版错误操作已经 SUCCEEDED 时，重复 execute 只返回旧结果；应核实用户原意图和最新数据，以新 requestKey 提交纠正修改，不重建整单。

## 已有送货单更新（MCP v1.7.1）

先读取 `deliveries` 详情，再 prepare：

```json
{
  "operation": "saveDeliveryAction",
  "input": {"id": "<送货单ID>", "form": {"deliveryDate": "2026-11-21"}},
  "preconditions": [{"module": "deliveries", "id": "<送货单ID>", "version": "<回读version>"}],
  "requestKey": "<本次意图唯一值>",
  "reason": "按用户要求更正送货日期"
}
```

更新时省略 title、assigneeId、status、notes、reviewNote 会保留原值。preconditions.version 是记录哈希；form.updatedAt 是业务保存使用的时间戳，两者不能混用。服务器从已验证的快照自动填入 updatedAt，客户端可省略；如显式传入，必须和详情的 updatedAt 完全一致。prepare 后他人修改会使 execute 拒绝，重新读取后合并，不能绕过校验。新建时 id=null，仍按 schema 提供 title、deliveryDate、assigneeId、status、notes、reviewNote。保存后回读日期及未修改字段。
