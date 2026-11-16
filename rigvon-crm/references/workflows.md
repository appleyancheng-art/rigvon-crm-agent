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
| 新结算 / 完整保存 | createSettlementAction / saveSettlementAction |
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
准备 updateProductAction，input 为 `{productId,form:{…}}`，preconditions 为 products 对应 id/version。不传 image 时保持原图；上传字段名以描述为准，不能猜。

## 保存结算

get_record 返回业务详情。items 的快照字段需转成 editor payload 字段：nameSnapshot→name、brandSnapshot→brand、categorySnapshot→category；保留 inventoryItemId、quantity、unitPrice、includeInServiceFee、includeInDiscount、pricingMode、fixedTotal。自定义项目保留 rowKey=id、isCustom 和 purchasePrice。pricingGroups 的 id 映射为 clientId，同时对应行使用 pricingGroupClientId。支付和调整项保留顺序与来源。必须查看 saveSettlementAction 返回的 payloadSchema，不能直接把数据库详情原样塞入 form。

完整保存会替换明细；未要求删除的行必须保留。新增自定义设备使用唯一 rowKey，并按用户提供资料填写，不能冒充库存设备。

## 方案

createDesign 创建空方案后读取 designs 详情，在 document 基础上修改；保存需要 id、revision、title、customerId、document，并提供 MCP 的 preconditions。服务器拒绝其他项目的底图关联。底图先 uploadDesignAsset，form.file 为附件对象。需要图片视觉分析时读取原始图纸；不要仅靠产品文字编造点位。

## 文件

quotes、settlements 支持 pdf/excel；contracts、designs 支持 pdf；reimbursements 支持 pdf/excel，传 recordIds（1–40 条），images 按需开启。当前不支持的模块/格式不要构造 URL。客户端下载大文件应等待 HTTP 完成，不要因稍慢反复生成。
