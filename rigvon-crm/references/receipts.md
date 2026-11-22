# 收据管理 · Skill v1.7.0 / MCP v1.4.0

收据是收款凭证，不是新增到账交易。查询模块为 `receipts`，不是台账的 `receipts` 明细数组，也不是库存入库单。先 `crm_capabilities`，再 `crm_describe_operation` 读取当前契约。

## 查找与开具

- `crm_search {module:"receipts",q,page,pageSize}` 查找现有收据；`crm_get_record {module:"receipts",id}` 取得详情、version 和 sourceValid。已删除的收据不返回。
- `crm_read_operation {operation:"listReceiptSources",input:{q,kind?,id?}}` 查询已有实收来源，kind 是 deposit（已确认意向收款）、ledger（台账收款）、settlement（结算独立收款）。返回的来源 id 和 sourceHash 用于开具，最多60条，可缩小关键词或传来源id。
- 已有款项用 `issueReceiptAction`，input 为 `{form:{kind,sourceId,sourceHash,customerName,purpose,notes,paidAt}}`。来源已有日期时保持原日期；缺日期时按用户提供的北京时间日历日期填写 YYYY-MM-DD。sourceHash 必须来自刚才读取的来源，不能编造。同一来源只允许一张有效收据，重复开具返回已有单据。
- 不在客户档案或台账中的实收用 `issueTemporaryReceiptAction`，input 为 `{form:{requestId,customerName,projectName,amount,paidAt,method,purpose,notes}}`。requestId 是本次开具意图的 UUID，重试复用。金额传正数字符串、最多两位小数；method 是 BANK_TRANSFER / WECHAT / ALIPAY / CASH / OTHER。可选内容用空字符串。必须有用户提供的收款事实，不能自行推定已到账。
- 独立开具不会创建客户、台账、结算收款或财务收入。已有来源不得再次作为临时收据重复开具。需要重新建立业务档案时另按用户指令处理。

写入统一 `crm_prepare_operation → crm_execute_operation`。READY 可执行；保留 requestKey，禁止为了重试重复创建。执行后回读 records 中的 receipts ID，核对交款人、日期、金额、用途、状态和 sourceValid。AI 来源由服务器记录，不能冒充人工。

## 作废与删除

`voidReceiptAction {id,form:{reason}}` 作废保留列表记录；`deleteReceiptAction {id}` 从业务列表移除并保留后台快照和审计。二者都不退款、不删除原付款、不改变台账金额。操作前必须读取 receipts 的 version 并提供 preconditions，两者均进入 CRM 人工审批，Agent 不可自批。删除后原下载链接不可再下载；需要重开时使用新的开具意图。

## 下载

`crm_export_link {module:"receipts",id,format:"pdf"|"excel"}` 返回15分钟短时绝对链接；直接下载，不需要另取 Token。PDF 与 Excel 均使用“收据”标题，不显示“临时收据”标题。作废、删除或原收款改变导致 sourceValid=false 时不能导出；先核对来源，由用户决定作废/删除与重新开具，不能绕过校验。
