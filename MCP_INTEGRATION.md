# MCP 集成说明

> 本文说明 mcd-campus-saver 依赖的 MCP Server、用到的全部 Tool、M1–M4 各自的调用流程，以及这套集成带来的业务价值。
>
> 文中所有标注「实测」的数据，均来自 2026-10-20 的真实联调，未做任何构造。完整过程见 [SMOKE_TEST_REPORT.md](./SMOKE_TEST_REPORT.md)。

---

## 1. MCP Server

| 项 | 值 |
|---|---|
| Server 名称 | `mcd-mcp` |
| 端点 | `https://mcp.mcd.cn` |
| 传输类型 | `streamablehttp` |
| 认证方式 | HTTP Header `Authorization: Bearer <token>` |
| 令牌环境变量 | `MCD_MCP_TOKEN` |
| 配置文件 | `~/.workbuddy/mcp.json`（WorkBuddy）；模板见 `mcp-config.example.json` |
| Tool 总数 | 35 个（本 Skill 实际使用 14 个，其余为链路预留） |

### 配置示例

```json
{
  "mcpServers": {
    "mcd-mcp": {
      "type": "streamablehttp",
      "url": "https://mcp.mcd.cn",
      "headers": {
        "Authorization": "Bearer ${MCD_MCP_TOKEN}"
      },
      "disabled": false
    }
  }
}
```

令牌通过环境变量注入，不进版本库。写入配置后需在连接器管理页面点「信任」启用，MCP 不会自动激活。

---

## 2. Tool 清单及用途

### 2.1 本 Skill 实际调用的 Tool（14 个，全部经过真实联调）

| # | Tool | 用途 | 所属模块 | 关键入参 | 关键出参 |
|---|---|---|---|---|---|
| 1 | `now-time-info` | 取服务器当前时间，判断门店营业状态、活动锚定日期 | M1 / M4 | 无 | `date`、`datetime`、`dayOfWeek`、`timestamp` |
| 2 | `query-nearby-stores` | 按位置搜索到店场景可点餐门店 | M1 | `searchType=2`、`city`、`keyword`、`beType`（1 到店 / 5 得来速） | `storeCode`、`storeName`、`address`、**`distance`（米）**、`businessStatus`、`reservationTimeOptions` |
| 3 | `delivery-query-addresses` | 拉取用户已有配送地址列表 | M2 | 无 | `addresses[].addressId` / `contactName` / `fullAddress` / `phone` |
| 4 | `delivery-create-address` | 新建配送地址 | M2 | `city`、`contactName`、`gender`、`phone`(11 位)、`address`、`addressDetail` | `addressId`、`fullAddress`（系统补全行政区划） |
| 5 | `delivery-query-stores` | 按地址查可配送门店，**beType 不同返回的 beCode 不同** | M2 / M3 | `addressId`、`beType`（2 麦乐送 / 6 团餐） | `storeCode`、`beCode`、`reservationTimeOptions` |
| 6 | `query-meal-assistance` | 团餐专属：查可选助餐服务 | M3 | `storeCode`、`beCode`、`orderType=2`、`beType=6` | `gmServiceCode`、`gmServiceName`、`enable`、`unusableReason` |
| 7 | `query-promotions` | 团餐专属：查当前门店有效的满折规则原始数据 | M3 | `storeCode`、`beCode`、`orderType=2`、`beType=6` | `promotionId`、`promotionType`、`ruleCategory`、`ruleDetail.orderDiscount`、`gmServiceCode` |
| 8 | `query-meals` | 查餐品列表与价格，取商品 code | M3 | `storeCode`、`beCode`、`orderType`、`beType` | `categories[]`、`meals{code}.name/currentPrice`（单位：元） |
| 9 | `calculate-price` | 试算含优惠的价格，团餐必需 `gmServiceCode` | M3 / M4 | `items[]`、`gmServiceCode`、`beCode`、`orderType`、`beType` | `price`(分)、`originalPrice`(分)、`discount`(分)、`enjoyed`、`enjoyable`、`takeWayList` |
| 10 | `available-coupons` | 列出当前可领取的优惠券 | M4 | 无 | 券名、状态、图片 |
| 11 | `auto-bind-coupons` | 一键领取全部可领券，无需指定券 ID | M4 | 无 | 成功/失败数、`couponId`、`couponCode` |
| 12 | `campaign-calendar` | 查当月营销活动日历（全国/月度维度，不按门店区分） | M4 | `specifiedDate`（可选，`yyyy-MM-dd`） | 往期 / 今日 / 未来三组的活动标题与介绍（**纯文本，无结构化字段**） |
| 13 | `query-store-coupons` | **真正的可用性校验**：指定门店 + 订单类型下能用的券 | M4 | `storeCode`、`beCode`、`orderType`、`beType` | `title`、`couponId`、`couponCode`、`tradeDateTime`、`products[]` |
| 14 | `mall-points-products` | 积分商城商品列表，按类目筛选 | M4 | `catRuleIds`（逗号分隔多值） | `spuId`、`spuName`、`point`、`price`、`catName`、`upTime`/`downTime` |

### 2.2 链路中声明、本轮未触发的 Tool（21 个）

这些 Tool 属于完整业务链路，在冒烟测试的四个场景里未被触发（多数涉及真实下单/扣积分，属于高风险操作，测试时主动规避）：

| Tool | 用途 | 所属模块 |
|---|---|---|
| `create-order` | 创建订单（必须先 `calculate-price`） | 结算 |
| `cancel-order` | 取消订单 | 兜底 |
| `order-list` / `query-order` | 订单列表 / 订单详情与配送状态 | 兜底 |
| `query-meal-detail` | 商品详情与特调选项（取 `selectedKey` / `unselectedKey`） | M3 可选 |
| `query-my-coupons` | 卡包资产（**仅展示，不校验可用性**） | M4 |
| `query-my-account` | 积分余额 | M4 |
| `mall-product-detail` | 积分商品 SKU 详情 | M4 |
| `mall-create-order` / `mall-order-list` / `mall-order-detail` | 积分商城下单与订单查询 | M4 |
| `query-lottery-info` / `draw-lottery` | 抽奖信息（先展示消耗）/ 执行抽奖 | M4 |
| `query-my-prizes` | 我的奖品 | M4 |
| `query-survey-coupon` | 问卷领券 | M4 |
| `list-nutrition-foods` | 营养信息查询 | 附加 |
| `query-party-city` / `query-party-store` / `query-party-store-date` / `query-party-store-session` / `party-order-create` | 品鉴会 / 派对报名链路（五步，每步等用户选择） | M4 延伸 |

### 2.3 参数矩阵（调用前必查）

`beType` 决定了一整条参数链怎么传，传错直接报错：

| 场景 | beType | orderType | beCode | addressId | gmServiceCode | takeWayCode | 门店来源 |
|---|---|---|---|---|---|---|---|
| 到店自取 | 1 | 1 | **禁止传** | — | — | 必传 | `query-nearby-stores` |
| 得来速 | 5 | 1 | 必传 | — | — | 必传 | `query-nearby-stores` |
| 麦乐送 | 2 | 2 | 必传 | 必传 | — | **禁止传** | `delivery-query-stores` |
| 企业团餐 | 6 | 2 | 必传 | 必传 | 必传 | **禁止传** | `delivery-query-stores` |

**实测确认的两条关键行为**：

- **beCode 随 beType 变化。** 同一门店 `storeCode=1400235`，`beType=2` 时 `beCode=140023502`，`beType=6` 时 `beCode=140023581`。切场景必须重取，拿旧值调新场景会失败。
- **beType=1 返回的门店确实没有 beCode 字段**，后续一律不传。

---

## 3. M1–M4 调用流程

### M1 校园门店雷达

```
① now-time-info()                          → 当前时间，用于营业状态判断
② query-nearby-stores(
     searchType = 2,                       → 按位置搜索（1 是搜收藏，校园场景不适用）
     city       = "广州市",                 → searchType=2 时必填
     keyword    = "广州大学城",
     beType     = 1                        → 1 到店 / 5 得来速
   )
③ 按 distance 升序排列，取前 3 家
④ （可选）campaign-calendar()               → 挂当月活动标签
```

**实测结果**：命中 5 家，distance 分别为 811 / 998 / 1150 / 1234 / 1235 米，全部位于小谷围街大学城范围内。全部 `reservation=true`，返回 10-20（today=true）与 10-21 两天的预约时段。

**注意**：`beType=1` 时**不要传 `beCode`**；`beType=5`（得来速）时必须取回 `beCode` 并在后续所有调用中携带。

---

### M2 外送自动定位

```
① delivery-query-addresses()
   ├─ 有地址 → 用户选一个，取 addressId
   └─ 列表为空 → 进 ②（直接引导新建，不做「要不要新建」的确认）
② delivery-create-address(
     city="广州市", contactName="测试同学", gender="先生",
     phone="13800000000",
     address="广州大学城 中山大学东校园至善园6号宿舍楼",
     addressDetail="B座512寝室"
   )                                        → addressId
③ delivery-query-stores(addressId, beType=2)   → storeCode + beCode
```

**实测结果**：地址列表为空 → 新建成功，`addressId=1036827600192209995798302260`，接口把地址补全为「广东省广州市番禺区大学城外环东路132号 中山大学(广州校区东校园)B座512寝室」。`beType=2` 返回 3 家可配送门店。

**注意**：外送场景返回的门店**不含 distance 字段**，排序只能按接口返回顺序，不要编造距离。

---

### M3 多人拼餐计算器

```
① 切团餐：delivery-query-stores(addressId, beType=6)   → 新的 beCode（≠ beType=2 的值）
② query-meal-assistance(storeCode, beCode, 2, 6)       → 选 gmServiceCode
③ query-promotions(storeCode, beCode, 2, 6)            → 折扣门槛原始数据
④ query-meals(storeCode, beCode, 2, 6)                 → 餐品 + productCode + 价格
⑤ calculate-price(...) × 3 套方案                       → 实付（分）
⑥ 比价选优 → 展示三方案卡片
⑦ （用户确认后）create-order(...)
```

**实测结果（5 人 / 150 元）**：

| | 方案 A | 方案 B | 方案 C |
|---|---|---|---|
| 商品原价 | ¥140.00 | ¥147.00 | ¥300.00 |
| 商品现价 | ¥140.00 | ¥147.00 | **¥234.00**（触发 7.8 折） |
| 运费 | ¥6.00 | ¥6.00 | ¥6.00 |
| 打包费 | ¥6.30 → ¥3.00 | ¥5.50 → ¥3.00 | ¥13.50 → ¥6.00 |
| 优惠 | ¥3.30 | ¥2.50 | ¥73.50 |
| **实付** | **¥149.00** | **¥156.00** | **¥246.00** |
| 人均 | ¥29.80 | ¥31.20 | ¥49.20 |
| `enjoyable.balance` | 差 ¥160 到 7.8 折 | 差 ¥153 | 差 ¥200 到 7.4 折 |

**折扣门槛（query-promotions 原始规则，12 条全为 `ruleCategory=30` 满折）**：

| gmServiceCode | 300 元 | 500 元 | 1000 元 | 2000 元 |
|---|---|---|---|---|
| GMS001 保鲜速达 | 7.8 折 | 7.4 折 | 7.0 折 | 6.6 折 |
| GMS002 专人分餐 | 8.8 折 | 8.4 折 | 8.0 折 | 7.6 折 |
| GMS003 专人摆台 | 9.3 折 | 8.9 折 | 8.5 折 | 8.1 折 |

**助餐服务时间窗（实测）**：GMS002 / GMS003 的 `enable=false`，原因是「专人服务仅对送达时间为 07:00~20:00 点的订单可享」。凌晨下单只有 GMS001 可选——必须把不可选项过滤掉再给用户选，否则报错。

---

### M4 学生优惠聚合

```
① available-coupons()                       → 可领券列表
② auto-bind-coupons()                       → 一键全领，返回 couponId + couponCode
③ 支线 A：query-store-coupons(storeCode, beCode, orderType, beType)
          → 真正的可用性校验，卡包不作数
          → 可用券回流入 calculate-price 逐张试算，取省得最多的
④ 支线 B：now-time-info() → campaign-calendar(specifiedDate)
          → 关键词筛出学生向活动（学生 / 学期卡 / 1+1 / 买一送一 / 早餐）
⑤ 支线 C：query-my-account() → mall-points-products(catRuleIds)
          → 积分余额 + 可兑换商品；抽奖需先 query-lottery-info 展示消耗并等确认
```

**实测结果**：`available-coupons` 返回 5 张 → `auto-bind-coupons` 成功 5 张失败 0 张 → 但 `query-store-coupons(beType=6)` 返回**空数组**，5 张券一单都用不上。

**积分商城类目（实测多值生效）**：`catRuleIds` 用**逗号**分隔传多个值，返回并集——`1>6>25` 得 7 条、`1>4` 得 24 条、`1>4,1>6>25` 得 **31 条**。（值内部的 `>` 是层级，不是分隔符。）

---

## 4. 业务价值：学生省钱闭环

这套 MCP 集成的价值不在于「能下单」——任何点餐助手都能下单。价值在于**把省钱的每一步都做成可验证的**，而不是靠话术让用户以为自己省了。

### 4.1 真实距离排序 —— 不编造，也不需要编造

M1 直接消费 `query-nearby-stores` 的 `distance` 字段（整数米），按升序排并展示真实数字：811 米、998 米、1.15 km。

这件事的意义有两层。一是**可信**：学生能据此判断「多走 200 米值不值」，而不是对着「距离较近」这种模糊描述猜。二是**安全**：架构设计阶段曾担心接口不含距离字段、需要退回「最近 / 次近」的相对表述，实测证明字段存在，所以 Skill 里严禁编造距离的这条规矩有了正向的落点——不需要编，就绝不编。

### 4.2 团餐折扣门槛诚实提示 —— 最难也最值钱的一条

团餐是唯一能拿到折扣规则的场景，这也是 M3 选 beType=6 而不是麦乐送的原因。但折扣门槛是 **300 元起**，而学生拼单的典型预算是 150 元——**够不着**。

这时候最省事的做法是假装门槛不存在，或者硬推一个「凑单更划算」的方案。实测数据说明那样做是骗人的：

- 预算 150 元，离 300 元门槛差 150 元
- 硬凑到 300 的方案 C，实付 ¥246，比不凑的方案 B（¥156）**贵 ¥90**

所以 Skill 的处理是：**先看 `calculate-price` 返回的 `enjoyable.balance`**（还差多少分到门槛），差额过大就直接放弃 C 方案，并明确告诉用户「团餐折扣 300 元起，150 块够不着，A/B 省下的 3.3 / 2.5 元只是打包费减免，不是餐品折扣」。

这条规矩已经写进 `references/m3-group-meal.md` 的降级策略。**省钱助手的第一品质是准确，不是乐观**——一个会承认「这次没折扣」的助手，才敢让人相信它说「这次有折扣」的时候。

### 4.3 券可用性验证 —— 领券不是终点，能用才是

`available-coupons` + `auto-bind-coupons` 能在两秒内领到 5 张券，屏幕上会显示「成功 5 张」。但 `query-store-coupons` 在同一个门店、同一个订单类型下返回的是**空数组**——这 5 张券在团餐场景一张都用不了。

如果 Assistant 停在「已帮你领到 5 张券」，用户会带着错误预期走到结算页，最后发现用不了，体验比没领还差。

所以本 Skill 把券的可用性定义为**以 `query-store-coupons` 为准，卡包（`query-my-coupons`）一律不作数**，并要求可用券必须回流入 `calculate-price` 重算，算出「用这张券实际省了多少」才算闭环。领到 5 张、能用 0 张，是一个诚实的坏消息，但比一个虚假的好消息有用得多。

### 4.4 闭环总结

```
查店（真实距离）  →  定位（一次问清，全程携带）  →  拼餐（真实折扣门槛 + AA 分账）
                                                          ↓
                                              领券 → 验证可用 → 回流入计价 → 算出省了多少
```

四个模块共用一份会话上下文，`beCode` 这类随场景变化的关键参数在切换时自动重取，用户从头到尾只需要回答一次城市和一次宿舍地址。省下的每一分钱，都有一个可以追溯的接口字段作为依据。

---

## 5. 已知边界

| 边界 | 说明 |
|---|---|
| 团餐折扣门槛高 | 300 元起，5 人以下小预算拼单通常触发不了，此时团餐相比麦乐送无价格优势 |
| 团餐场景不支持麦麦省券 | 实测 `query-store-coupons(beType=6)` 恒为空，用券需切 beType=2 |
| 助餐服务有时间窗 | GMS002 / GMS003 仅 07:00–20:00 送达可用 |
| 活动日历无结构化字段 | 返回纯文本，学生向筛选依赖关键词，可能漏掉不含关键词的活动 |
| 外送门店无距离字段 | 只有到店场景（`query-nearby-stores`）返回 distance |
| 价格两套单位 | `query-meals` 的 `currentPrice` 是元字符串；`calculate-price` 全部是分整数，展示前必须 ÷100 |
