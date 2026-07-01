# 电子发票（数电票）对接技术说明 · 百望云 SMB 开放平台

> 关联 Issue：[#103 用户端会员订单自助开票（数电票，对接百望云）](https://github.com/YZSGO/feedback/issues/103)
> 读者：开发
> 状态：调研 / 方案初稿（待评审）
> 最后更新：2026-07-01

本文整理接入百望云数电票开放平台的**已知信息与初步方案**，作为后续详细设计与排期的依据。凭据、报价等敏感信息**不落公开仓库**，见内部渠道。

---

## 1. 结论速览

- 服务商：**百望云**，产品线 **SMB / 小微**（接口方法名前缀 `baiwang.s.outputinvoice.*` 的 `s` 即 SMB）。一步到位对接**数电票**，不走传统电子发票老路。
- 开票是**异步**的：调开具接口只同步返回受理流水号，真正的开票结果（状态 / PDF 链接 / 发票号）通过**回调**拿到，`query` 仅作兜底对账。
- 成本对量不敏感：报价里**无「按张 / 按开票量」计费**，只有一次性接口费 + 按销方税号计年费。主体数量固定则成本近似封顶（详见内部商务记录）。
- 我们要自建的核心是：**开票单状态机 + 幂等 + 回调接收器 + 归档(OSS) + 通知**，税控能力全部交给百望云。

---

## 2. 环境与接入信息

| 项 | 值 |
|---|---|
| 沙箱网关 | `https://sandbox-openapi.baiwang.com/router/rest` |
| 生产网关 | `https://openapi.baiwang.com/router/rest` |
| 沙箱开放平台 | `https://open-pre.baiwang.com/openplatform-ui/home`（接口在左侧栏「小微产品」下） |
| 沙箱业务后台 | `https://work-pre.baiwang.com/smb/sales/write` |
| 沙箱账号 / appKey / appSecret / 盐值 / 密码 | **不入公开仓库** —— 见内部渠道，落地放 cn Infisical 按环境注入 |

**认证模型**（对应 SDK `PasswordLoginClient`）：

```
appKey + appSecret + userSalt + username + password
        │  密码登录（login）
        ▼
   accessToken（有服务端需缓存 + 到期刷新）
        │  每次业务调用带上 token
        ▼
   invoice / query / queryPage / fastRed ...
```

> 落地要点：token 统一由服务端管理，缓存 + 过期自动重登；appSecret / 盐值 / 密码仅存服务端 Infisical，全程 HTTPS，日志脱敏。

---

## 3. 接口清单

来源：《开票查询冲红接口最新》压缩包（SDK ≥ 3.4.676 / 部分 3.4.629）+ 销售提供的常用接口列表。

### 3.1 核心接口

| 用途 | method | 类型 | 说明 |
|---|---|---|---|
| 发票开具 | `baiwang.s.outputinvoice.invoice` | 写 | 开蓝票 / 红票（`invoiceType` 1=蓝 2=红）。同步返回受理流水号，结果走回调。 |
| 快捷红冲 | `baiwang.s.outputinvoice.fastRed` | 写 | 退款场景红冲。`fastIssueRedType`：0=仅生成红字信息表/确认单，1=生成后自动开红票。 |
| 开票结果查询 | `baiwang.s.outputinvoice.query` | 读 | 按销方税号 + 流水号/开票单号精确查单张状态，返回 `status` / `pdfUrl` / `xmlUrl` / `ofdUrl` / 发票号。 |
| 发票分页查询 | `baiwang.s.outputinvoice.queryPage` | 读 | 分页列表，适合「我的-发票」历史记录页。 |

### 3.2 辅助接口（压缩包内，用途已确认，字段待细读）

| 用途 | 场景 |
|---|---|
| 确认单查询和下载 | 数电红字确认单的查询 / 下载 |
| 确认单确认和撤销 | 数电**专票**红冲需购方在税局确认时使用 |
| 发票再次交付 | 历史记录里「重新发送到邮箱/短信」 |
| 开票失败重推开具 | 开票失败后的手动/自动重试 |

### 3.3 回调（推荐主路径，文档待获取）⚠️

销售明确：**开具和红冲结果推荐用回调实现**，参见《百望云-销项回调接口文档 1.0.6》。
**该文档尚未拿到**，它决定 webhook 收到的字段、验签方式、重试与幂等约定。**待补齐后本节展开。**

---

## 4. 关键字段与业务映射（对应 Issue #103 业务规则）

| Issue 规则 | 接口字段 / 取值 |
|---|---|
| 个人开普票 | `invoiceTypeCode=02`（数电普票）；自然人 `naturalMark=1` + 姓名 + `buyerCredentialsType/No`（身份证） |
| 企业一般纳税人开专票 | `invoiceTypeCode=01`（数电专票）+ `buyerTaxNo` |
| 商品名统一「信息技术服务-会员服务费」 | 明细 `goodsName` + `goodsCode`（税收分类编码，需财务确认具体编码） |
| 税率 6% / 小规模 1% | 明细 `goodsTaxRate`；小规模优惠可能涉 `preferentialMark` / `vatSpecialManagement`（待财务确认） |
| 金额=实付含税、不含券 | 明细 `goodsTotalPriceTax`(含税)/`goodsTotalPrice`(不含税)/`goodsTotalTax`(税额)**建议全传**避免自动反算误差；`priceTaxMark=1` |
| 交付邮箱/短信 | `pushEmail` / `pushPhone`（也可自建交付，用回调拿 `pdfUrl` 后自行发送） |
| 幂等：同订单不重复开 | 业务侧生成唯一 `orderNo`（开票单号）作幂等键 |
| 退款红冲 / 部分红冲 | `fastRed` + 原票引用（原流水号/单号或原发票号）+ `details`（部分红冲传明细，金额为负） |
| 归档 ≥10 年 | 回调/查询得到 `pdfUrl`/`ofdUrl`/`xmlUrl` 后转存 OSS：`{业务类型}/{日期}/{订单号}.pdf` |

> 注：`orderNo` 是**我们生成**的开票业务主键，贯穿开具→查询→冲红→回调，务必全局唯一且幂等。

---

## 5. 架构与主链路（回调式）

```
会员购买成功 / 用户在「发票管理」申请
        │
        ▼
┌───────────────────────────────────────────────────────────┐
│  开票模块（建议在 commerce-backend 内，与订单/退款同库耦合） │
│                                                            │
│  ① 校验：抬头/税号格式 · 30 天有效期 · 该订单是否已开(幂等)   │
│  ② 生成 orderNo(唯一) → 落库 status=待开票                   │
│  ③ 调 invoice(callBackUrl=我方 webhook)                     │
│     ← 同步返回 serialNo → 落库 status=开票中                 │
└───────────────────────────────────────────────────────────┘
        │ (异步)                                ▲
        ▼                                       │ 回调: status/pdfUrl/发票号
┌────────────────────┐            ┌────────────────────────────────┐
│   百望云 SMB        │──────────▶│  webhook 接收器                 │
│  开票 → 上传税局    │   回调     │  验签 → 幂等(serialNo/orderNo)   │
└────────────────────┘            │  → 更新 status=完成/失败         │
        │                         │  → pdfUrl 转存 OSS               │
        │                         │  → 邮件 + 站内信通知用户          │
        │                         └────────────────────────────────┘
        ▼
   定时 query / queryPage 兜底对账（补偿丢失/超时的回调）
```

**红冲链路**（退款审核通过触发）为镜像：

```
退款通过 → 检测原单是否已开票 → 已开 → fastRed
   → 数电普票: 直接生成红票 → 回调 → 原单标记「已冲红」→ 通知
   → 数电专票且需购方确认: 生成红字确认单 → (确认单确认/撤销) → 开红票 → ...
```

---

## 6. 开票单状态机

```
                     ┌────────── 重推开具 ──────────┐
                     ▼                              │
 待开票 ──调invoice──▶ 开票中 ──回调/query──▶ 开票完成 ──┘(失败时)
                                     │
                                     ├─▶ 开票失败（含 errorMessage，可重推）
                                     │
   开票完成 ──退款触发 fastRed──▶ 冲红中 ──回调──▶ 已冲红
```

对应接口返回的 `status`：`00 开票中 / 01 开票完成 / 02 开票失败 / 03 发票已作废 / 04 发票作废中`。

---

## 7. 数据模型（建议，待评审）

**发票单 `invoice_order`**（一订单一票，红冲生成关联子记录）

| 字段 | 说明 |
|---|---|
| `order_no` | 开票业务主键（幂等键），全局唯一 |
| `biz_order_id` | 关联会员订单 |
| `seller_tax_no` | 销方税号（多主体时用于路由） |
| `serial_no` | 百望返回受理流水号 |
| `invoice_type_code` | 01 专票 / 02 普票 / … |
| `title_type` | 个人 / 企业 |
| `buyer_name` / `buyer_tax_no` | 抬头 |
| `amount` / `tax` / `amount_with_tax` | 金额（实付含税） |
| `status` | 状态机（见 §6） |
| `pdf_url` / `ofd_url` / `xml_url` / `oss_key` | 交付与归档 |
| `red_of_order_no` | 若为红票，指向原蓝票 |
| `created_at` / `status_updated_at` | 时间 |

**抬头记忆 `invoice_title`**（I-03，P1）：用户维度保存历史抬头，下次快捷选择。

---

## 8. 待确认 / 待办

- [ ] **获取《百望云-销项回调接口文档 1.0.6》** —— 回调是主路径，缺此文档无法定 webhook 验签/幂等/字段（§3.3）。
- [ ] **销方税号数量**：1 个还是多个？决定年费与是否需要多主体路由（开票时 `taxNo` / `invoiceTerminalCode`）。
- [ ] **确认无「按张/按量」隐藏计费与调用上限**（向销售书面确认）。
- [ ] **财务确认**：会员服务费的**税收分类编码 `goodsCode`**、一般纳税人 6% / 小规模 1% 的优惠标识填法（`preferentialMark` / `vatSpecialManagement`）。
- [ ] **归属服务确认**：建议放 commerce-backend 独立发票模块（与订单/退款同库）——待拍板。
- [ ] 税号实时校验（企查查/天眼查）是否 MVP 就上，还是仅做格式校验。

---

## 9. 与 Issue #103 原描述的差异（供开发注意）

Issue 正文「技术对接概要」用的是**占位接口名**（`invoice.create` / `invoice.query` / `invoice.red` / `invoice.download`）。**实际百望云 SMB 接口**为：

| Issue 占位 | 实际接口 |
|---|---|
| `invoice.create` | `baiwang.s.outputinvoice.invoice` |
| `invoice.query` | `baiwang.s.outputinvoice.query` / `queryPage` |
| `invoice.red` | `baiwang.s.outputinvoice.fastRed` |
| `invoice.download` | 无单独下载接口；下载链接(`pdfUrl`/`ofdUrl`)由**回调或 query 返回** |
| 回调通知 | 《销项回调接口文档 1.0.6》，**推荐作为主路径**（非可选） |
