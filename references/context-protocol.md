# 会话上下文协议

mcd-mcp 的参数链是强耦合的：`beType` 决定 `orderType`，`beType≠1` 必须带 `beCode`，
团餐额外要求 `gmServiceCode`，到店额外要求 `takeWayCode`。
一旦参数对不齐，接口直接报错。所以四个模块必须共用一份会话状态。

---

## 1. SessionContext 定义

```
SessionContext {
  scene         : 到店自取 | 得来速 | 麦乐送 | 团餐
  beType        : 1 | 5 | 2 | 6
  orderType     : 1 | 2                    // 由 beType 推导，不独立设置
  city          : string                   // 学校所在城市，如「南京市」
  storeCode     : string
  beCode        : string | null            // beType=1 时必须为 null
  addressId     : string | null            // 仅外送场景
  gmServiceCode : string | null            // 仅团餐
  takeWayCode   : string | null            // 仅到店（含得来速）
  partySize     : number                   // 拼餐人数
  budget        : number                   // 总预算（元）
  cart          : Item[]                   // { productCode, quantity }
  bestCoupon    : { couponId, couponCode } | null
}
```

### beType → orderType 推导（禁止手写，必须按此推导）

```
beType=1 → orderType=1
beType=5 → orderType=1
beType=2 → orderType=2
beType=6 → orderType=2
```

---

## 2. 参数矩阵

| 场景 | beType | orderType | beCode | addressId | gmServiceCode | takeWayCode |
|---|---|---|---|---|---|---|
| 到店自取 | 1 | 1 | **禁止传** | — | — | 必传 |
| 得来速 | 5 | 1 | 必传 | — | — | 必传 |
| 麦乐送 | 2 | 2 | 必传 | 必传 | — | **禁止传** |
| 企业团餐 | 6 | 2 | 必传 | 必传 | 必传 | **禁止传** |

### 灰色地带速查

| 工具 | 到店(1) | 得来速(5) | 麦乐送(2) | 团餐(6) |
|---|---|---|---|---|
| query-meals | orderType=1，不传 beCode | orderType=1，传 beCode | orderType=2，传 beCode | orderType=2，传 beCode |
| query-meal-detail | 同上 | 同上 | 同上 | 同上 |
| calculate-price | 同上 | 同上 | 同上 | 同上 + gmServiceCode |
| create-order | + takeWayCode | + takeWayCode | + addressId | + addressId + gmServiceCode |
| query-store-coupons | orderType=1，不传 beCode | 传 beCode | 传 beCode | 传 beCode |
| query-promotions | 不可用 | 不可用 | 不可用 | **仅此场景可用** |
| query-meal-assistance | 不可用 | 不可用 | 不可用 | **仅此场景可用** |

---

## 3. 参数来源链

```
city           ← 用户明确提供（首次必问，之后复用）
storeCode      ← query-nearby-stores（到店）
               ← delivery-query-stores（外送，需先有 addressId）
beCode         ← 与 storeCode 同源同次返回，二者配对使用
addressId      ← delivery-query-addresses（选已有）| delivery-create-address（新建）
gmServiceCode  ← query-meal-assistance 返回的助餐服务 code
takeWayCode    ← calculate-price 返回的 data.takeWayList[].code
couponId/couponCode ← query-store-coupons 返回的当前门店可用券
productCode    ← query-meals 返回的餐品 code
```

**链路顺序（外送场景的完整依赖）**

```
city → addressId → (storeCode + beCode) → 餐品/促销/助餐 → 计价 → 下单
```

任何一环缺失，必须先补齐再往下走，不要跳过。

---

## 4. 上下文写入时机

| 时机 | 写入字段 |
|---|---|
| 用户说出城市 | city |
| 用户选定门店 | storeCode, beCode, beType, orderType, scene |
| 用户选定/新建地址 | addressId |
| 用户给出人数和预算 | partySize, budget |
| 拿到促销规则 | 凑单门槛（内部使用，用于方案生成） |
| 用户选定助餐服务 | gmServiceCode |
| 计价完成 | takeWayCode, 实付金额 |
| 选定最优惠券 | bestCoupon |

---

## 5. 场景切换规则

### 5.1 beType 变化必须重取 beCode

`beCode` 是「门店 + 业务场景」的联合编码，场景一变，值就变。

```
错误做法：beType 2 → 6，沿用旧 beCode 调用 calculate-price
正确做法：beType 2 → 6 时，重新调用
         delivery-query-stores(addressId, beType=6) 取回新 beCode
```

**典型触发场景**：用户先说"送到宿舍"（beType=2），接着说"我们 5 个人拼"（切 beType=6）。
此时必须重取 beCode，不能直接切参数继续用。

### 5.2 缺参时的补问顺序

按依赖顺序补，一次只问一组，不要连环追问：

1. 缺 city → "你在哪个城市？"
2. 缺门店 → 走 M1 查门店
3. 缺 addressId → 走 M2 查/建地址
4. 缺人数/预算 → "几个人？总预算多少？"
5. 缺助餐服务 → 展示选项让用户选

---

## 6. 单位与格式规范

| 项 | 规范 |
|---|---|
| 价格 | 接口返回单位为**分**，展示前 ÷100 转元 |
| 人均 | 实付金额 ÷ 人数，保留 1 位小数 |
| 手机号 | 11 位纯数字、1 开头，格式为 `18616646686`（无空格无连字符） |
| 城市名 | 真实城市全称，如「南京市」「上海市」，不接受「南京」以外的简称推断 |
| 预约时间 | 格式 `yyyy-MM-dd HH:mm`，仅预约场景传，非预约场景一律不传 |
| 订单号 | 仅来自接口返回或用户提供，禁止编造 |

---

## 7. 上下文失效处理

- 接口报「参数错误」→ 优先怀疑 beCode/beType 不匹配，重新走一遍门店查询。
- 接口报「门店不存在」→ storeCode 可能已失效，重新 `query-nearby-stores` 或 `delivery-query-stores`。
- 用户换了城市/学校 → 清空 city/storeCode/beCode/addressId，重新开始。
- 用户换了宿舍地址 → 只清 addressId 及其派生的 storeCode/beCode，city 保留。
