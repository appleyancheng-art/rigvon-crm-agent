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

- 灯具固定符号：`spot` 圆点、`linearLight` 短条、`wallLight` 三角、`ceilingLight` 方块；灯带 path 是连续发光线，产品关联和数量不变。`mounting` 可选 `ceiling/wall/floor`（顶面/立面/底位），只决定安装面标识，不改变计价。
- 分类底图示例：`floors[0].pageBackgrounds = [{"pageId":"ceiling","assetId":"<本方案已上传底图ID>","width":2000,"height":1500,"metersPerPixel":null}]`。仅更新目标条目，保留其他分类。未设置继承楼层；`assetId:""` 表示该分类空白；移除条目恢复楼层底图。基础楼层换图不改已有独立底图。
- 替换/删除独立底图，保留该分类设备的归一化坐标，先按旧图尺寸及比例计算现有 autoLength 数量，再置 autoLength=false、metersPerPixel=null；其他分类不动。恢复继承也先冻结旧长度。分类图按独立宽高和比例计算，重新校准后再启用自动计米。
- 旋转设备使用 `rotation`（0–360度）；线路先按有效图纸 width/height 转像素坐标，围绕路径包围盒中心旋转角度差，再归一化并同步首节点 x/y。保留数量和长度，不将超界节点强行裁到0或1。
- 清空设备：移除全部 points 及引用动作，保留底图和 annotations。清空图纸与设备：另将各层 assetId 置空、pageBackgrounds 清空、metersPerPixel=null，并清空 annotations；保留楼层、房间和场景本身。删除、清空或移除图纸/标注走 MCP 审批，不能以保存方式绕过。
- 网页可对互不重叠的编辑自动合并；MCP 保留 revision 与 preconditions 校验。发生冲突须回读后按原意图合并，只重新提交这次修改，不能把旧全量 document 配新 revision 直接覆盖。

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

灯具点位固定图标：`spot` 圆形射灯、`ceilingLight` 方形格栅灯、`linearLight` 长方形明装灯、`trackLight` 细条轨道灯。`magneticTrack` 为磁吸轨道线路，layer=`light`，path 至少两个节点；缩放和旋转同步更新 path 与 x/y。米制产品校准后计米，按根/件销售的轨道沿用手工数量。磁吸轨道电源、驱动和连接件不是轨道线路。

### 机柜、配套设备与他方供货（1.12.0）

网页方案编辑器采用图纸缩略图导航：左侧切换楼层和分类图纸，桌面顶部集中绘图与编辑工具，右侧独立属性栏；手机和平板使用右侧工具列和绘图弹层，缩放与复位可直接点击。保存时自动合并不冲突的页面修改；同一字段冲突时由用户选择保留哪一方。MCP 仍须读取最新 revision 后保存，不绕过版本校验。

- `points[].placement` 缺省 `plan`，表示图纸点位；`cabinet` 是柜内设备，`accessory` 是配套设备。这两类保留在 document.points 中作为单一数据源，不另建重复报价项，画布和图例不显示。
- 机柜点位为 `symbol: "rack", layer: "equipment"`。机柜模型可不绑定产品，此时本体不报价；绑定机柜产品才计入机柜本体价格。
- 柜内设备必须 `parentPointId` 指向同一楼层的机柜点位。配套设备可关联同层灯带/轨道等点位。两类清单设备需要 product、手工 quantity、layer=`equipment`、symbol=`equipment`、path=[]、autoLength=false，不使用 pendingScale。数量不自动猜测。
- `supplyMode` 缺省 `ours`。仅当用户说明不由我方供货时，将指定点位设为 `external`；仍显示点位及“他方供货”图例，不计入设备预算、导出报价与新建报价。此字段只影响本方案指定设备，不修改产品库、其他方案、安装面颜色或关联电源的供货范围。
- 例如他方柜体灯带设 external，与它关联的我方驱动、电源以 accessory、parentPointId=灯带点位id、supplyMode=ours 单独录入。柜内模块同理只算一次，不在图纸上重复摆放。
- 删除机柜/关联点位时，保留的柜内/配套设备清除 parentPointId，并转为 accessory；不偷偷删掉其报价明细。整方案清空时，图纸点位、柜内及配套设备一并清空，并移除场景引用，仍需 CRM 确认。


## 方案直接编辑（Skill 1.13.0 / MCP 1.9.0）

- `document.points[].iconScale` 可选，0.25–4，缺省 1。图标放大至 150% 写 `1.5`；普通产品、机柜和灯具均适用，线路仅改变线宽和光晕。画布和 PDF 使用同一倍率，不改变产品数量、路径或报价。完整保存必须保留未修改点位的倍率。
- 延长灯带/轨道时固定另一端，更新 `path` 及 `x/y=path[0]`；物理长度变化才重算计米。`iconScale` 不得代替线路长度。
- 灯光默认顶面黄色、中位/立面橙色、低位/底位蓝色。按 `mounting` 或缺省图层取色；改变灯带施工类别时清除旧安装面覆盖或同步正确 `mounting`。
- 文字再编辑保留 annotation 的 `id`、`floorId`、`pageId`、`kind`、`nodes`，只修改 `text`，不重复新增。
- 网页在当前标签页打开方案。按住鼠标中键平移；选择工具后弹出对应产品，灯带和轨道可先选直线/圆弧/整圆/折线。显式选择产品会启用其图纸类别。设备双击换型，文字双击编辑，旋转手柄可直接拖动；吸附可在图纸设置关闭或按 Command/Ctrl 临时跳过。iPhone 同桌面使用横向可滑动长条工具栏。

更新已有方案前读取最新 revision/version 和完整 document。保存使用 crm_prepare_operation → crm_execute_operation，回读确认 `iconScale`；大小调整不应改变 bill of materials。现有 Token 继续使用，刷新工具并 describe(saveDesign) 验证新字段。

## 分类图纸与 Keynote 操作（Skill 1.14.0 / MCP 1.9.1）

- 楼层总图页已取消。保留 `floors[].assetId` 作为各分类共用的默认底图，`pageBackgrounds` 可独立替换或移除当前分类底图。不要删除旧楼层、设备、共享标注或原始附件来模拟取消总图。
- `hiddenPageIds` 只控制本层分类图纸与图例显示。PDF 仅输出勾选且存在底图、点位或标注的分类图，不插入楼层总图。隐藏设备仍按原供货范围进入清单和报价。存量 `annotations[].pageId=":all"` 继续作为共享标注。
- PDF 导入改进嵌套 CAD 外框与标题栏识别；用户应在确认导入前复核裁剪预览，可恢复整页或手动调整。慢网底图会显示加载状态，短暂失败重试，最终失败可点重新加载；不能把失败提示当作新图纸保存。
- Mac：Command 拖控制点旋转，Shift 按45°步进；Option+Command 围绕对侧控制点旋转。普通拖控制点等比例缩放，默认固定对侧；Option 改为中心缩放。Option 拖设备复制，Command+D 原位偏移复制，方向键微移1像素、Shift+方向键10像素。
- 缩放：Command+Shift+＞/＜，Command+0 实际大小，Option+Command+0 适合窗口，Shift+Command+0 适合所选点位，Option+Shift+Command+0 适合全部内容。仍支持普通鼠标滚轮缩放、中键按住拖动画布。Command/Ctrl 拖设备临时跳过吸附；Shift 拖动约束方向。
- iPhone/iPad：双指开合缩放、双指滑动画布；先选中点位，再在该点位上双指扭转旋转，可松开一指继续调整。连接键盘时沿用 Command 快捷键。网页无法接管的浏览器保留键可使用相应工具按钮；文字输入时不执行画布快捷键。
- 保存字段及 Token、权限不变。已有 Agent 发送复制更新指令更新 Skill，刷新工具并实际调用 `crm_capabilities`，确认版本1.9.1。


## 客户定稿、归档与合同附件（Skill 1.15.0 / MCP 1.10.0）

只有用户明确选定版本时才定稿，不能把最新一版或金额最高的一版当作定稿。`crm_get_record(customers,id)` 返回 `finalQuoteId`、`finalDesignId`、`finalizedAt` 和 `proposalOptions`。允许同时选一份报价和一份方案，也允许只选其中一种；其他未删除报价和方案都会分别归档，不删除、不改变报价业务状态。

先 describe `finalizeCustomerProposalsAction`，再 prepare / execute：
```json
{"operation":"finalizeCustomerProposalsAction","input":{"customerId":"<客户ID>","form":{"quoteId":"<报价ID或null>","designId":null,"expectedUpdatedAt":"<客户详情updatedAt>"}},"preconditions":[{"module":"customers","id":"<客户ID>","version":"<最新version>"}],"requestKey":"<新的定稿意图键>","reason":"按用户选择定稿并归档其他版本"}
```
`quoteId` 和 `designId` 至少一个非空；字段空值用 JSON null，不传文字“null”。prepare 会展示选定版本与归档对象，execute 原子保存；随后回读客户以及归档列表。`crm_search` 的 quotes/designs 默认只列未归档，`view:"archived"` 列归档，`view:"deleted"` 列回收站；可增加 `customerId`。移回目录用 `unarchiveQuoteAction` / `unarchiveDesignAction`，顶层参数 `id`，提供对应记录 preconditions。恢复不会自动定稿。

合同 `saveContractAction` 使用完整 `form.payload`。`partyType:"PERSONAL"` 为个人合同，甲方使用 `partyAName`；`ENTERPRISE` 为企业合同，甲方使用必填 `partyACompany`，`partyAName` 为联系人，`partyAIdNumber` 为统一社会信用代码。旧客户端省略新增字段会保留已有设置。

`includeAttachments:false` 仅导出竖版合同。`true` 时必须关联客户，并选 `attachmentQuoteId` 和/或 `attachmentDesignId`，来源必须属于该客户。优先采用用户确认的定稿 ID，也可按明确指示选择其他版本。保存成功后固定当时 PDF 快照；源报价/方案之后修改或归档，不会改动已存附件。需要更新附件时重新保存合同。`crm_export_link` 导出一份 PDF，顺序为横版合同、报价、方案，所有部分均横版。导出不包含内部成本和备注；附件保存失败时整单失败，不报告成功。

网页方案操作补充：Option 拖动复制，Option 拖边缘中心缩放；复制/粘贴格式只传递 iconScale，线路不改变长度或数量。可多选粘贴，或应用到本图同款（跳过锁定项）。⌥⌘C/V 沿用 Keynote；Safari 可能优先打开控制台，可用格式按钮或 ⌥⇧C/V。Agent 批量修改尺寸仍需保留完整 document 的其余内容与最新 revision，不更改设备型号、安装面颜色、供货范围或报价数量。


## 合同附带报价的显示选项（Skill 1.17.0 / MCP 1.12.0）

客户定稿入口位于客户详情。合同编辑勾选“附带报价／方案”后，选择报价并逐项勾选导出内容。调用 saveContractAction，在完整 form.payload 中增加以下对象，其余合同字段保留最新详情并提供 preconditions：

```json
{"attachmentQuoteExportOptions":{"showUnitPrice":false,"showOriginalPrice":false,"showFinalTotal":true,"showItemSubtotal":false,"showRoomSubtotal":true,"showDiscount":false,"showFeeBreakdown":false,"showBrand":false,"showProductIntroduction":true,"showCustomer":true,"showCompany":true}}
```

此例只显示折后总额及房间小计，不显示单价、原价、设备合价和费用拆分。所有选项独立，缺省新合同默认 true；旧客户端省略对象或部分字段保留已有选择。旧 onlyFinalTotal:true 保存时展开为独立字段；以后可单独重新开启任一字段。未知字段或字符串布尔值会被拒绝。

保存成功后回读 attachmentQuoteExportOptions 与 attachments，核对已生成快照。合同导出使用保存时的附件，源报价变更不会自动改动；需要更新内容或选项时重新保存合同再导出。选项只控制报价附件，不控制合同正文的应付款。

单独合同竖版，合并附件时横版，两者字体、黑金配色、封面图片保持一致。企业合同采用系统已配置的公司注册地址、纳税人识别号、公司账户和开户行；个人合同使用原个人收款资料。切换类型后保存并回读，核对导出中的双方名称与收款账户。

Option 拖拽时保留原点位，副本预览随鼠标移动；松手仅新增一个副本，撤销/取消不会删除原点位。只改变编辑交互，方案保存 schema 不变。

## 合同系统保修年限（Skill 1.18.0 / MCP 1.13.0）

合同新增可编辑的 KNX 系统保修年限 systemWarrantyYears，1–99 整年，默认 5 年；保存后竖版及横版 PDF 同步使用该值。硬件设备的保修期限及范围按对应生产厂商政策执行；不再承诺所有硬件免费更换固定年限。旧客户端省略该字段时保留已有值。

1. 调用 crm_get_record(module="contracts", id)，读取最新 data 和 version。
2. 保留完整合同内容，将 `form.payload.systemWarrantyYears` 设为用户要求的整数，例如 `8`。不得传字符串、空值或小数。不要用 additionalTerms 覆盖系统保修条款。
3. 调用 crm_prepare_operation(operation="saveContractAction")，携带该合同最新 preconditions，再执行 operationId。存档合同须先按业务流程恢复草稿。
4. 回读合同确认 systemWarrantyYears=8，通过 crm_export_link(module="contracts", id) 导出：单独竖版与附带方案/报价的横版均写“KNX 系统由乙方提供 8 年保修服务”。硬件保修范围和年限始终按对应厂家政策。已下载 PDF 不会被追溯修改，必须重新导出。
