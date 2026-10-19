# M4 学生优惠聚合

一键领券，验证哪些券在当前订单真的能用，展示学生相关的近期活动，并管理积分兑换。
**券必须回流进计价**才算闭环。

---

## 1. 入口：一键领券

```
① available-coupons()      // 列出当前可领取的优惠券
② auto-bind-coupons()      // 一键领取所有可领券，无需指定券 ID
```

领取后进入三条支线，按用户需求选择，不必全跑。

---

## 2. 支线 A：券可用性筛选（闭环关键）

```
① query-my-coupons()       // 卡包资产，仅作展示
② query-store-coupons(
     storeCode = <ctx.storeCode>,
     beCode    = <ctx.beCode>,      // beType=1 时不传
     orderType = <ctx.orderType>,
     beType    = <ctx.beType>
   )
   → 这才是「当前门店 + 当前订单类型」下真正能用的券
```

### 关键认知

**卡包里有 ≠ 这单能用。** `query-my-coupons` 明确不做门店、渠道、配送方式的校验，
所以不能拿它来回答「我有什么券能用」。可用性一律以 `query-store-coupons` 为准。

**实测补充（2026-10-20）：团餐场景（beType=6）下 `query-store-coupons` 恒返回空**——
麦麦省券不支持团餐订单。若用户在拼餐链路问券，如实告知「团餐这单用不了这些券」，
引导切麦乐送（beType=2）后再查可用券，不要假装能凑出券优惠。

展示时区分两组：
- **这单能用**（来自 query-store-coupons）→ 进入计价
- **暂不能用**（卡包里有但当前订单不适用）→ 列出但标注不可用

### 回流进计价

```
③ calculate-price(
     ...既有参数,
     items: [{ productCode, quantity, couponId, couponCode }, ...]
   )
```

多张候选券时，**逐张试算，取省得最多的那张**，并告诉用户省了多少。

---

## 3. 支线 B：活动日历

```
① now-time-info()                                  // 取当前日期
② campaign-calendar(specifiedDate = "yyyy-MM-dd")  // 可选，锚定某天
   → 不传则默认返回当月全部活动
```

返回进行中、往期、未来三类活动。

### 学生向筛选

活动是**全国/月度维度**，不按门店区分，所以展示时只需筛选学生相关关键词：

| 关键词 | 典型活动 |
|---|---|
| 1+1 / 随心配 | 1+1 随心配 |
| 买一送一 / 第二份 | 买一送一 |
| 学期卡 / 麦金 | 麦金学期卡 |
| 早餐 | 早餐优惠 |
| 学生 | 学生专属活动 |

筛选后标注「学生党重点关注」，并给出剩余天数。

---

## 4. 支线 C：积分与体验活动

```
① query-my-account()                     // 积分余额
② mall-points-products(catRuleIds = "...")  // 按类目筛选
```

### 类目规则 ID 速查

| catRuleIds | 类目 |
|---|---|
| `1>4` | 商品券（非预付优惠券） |
| `2` | 实物商品 |
| `2>8` | 周边产品 |
| `2>9` | 实物礼品卡 |
| `1>6>20` | 生日类派对 |
| `1>6>21` | 主题类派对 |
| `1>6>22` | 麦麦体验营 |
| `1>6>25` | 品鉴会 |
| `1>6>34` | 读书会 |
| `1>6>40` | 积分兑换活动 |

校园场景优先展示：`1>4`（餐品券）、`1>6>25`（品鉴会）、`1>6>22`（体验营）。

**多类目传法（实测）**：`catRuleIds` 传多个类目用**英文逗号**分隔，如 `1>4,1>6>25`
（实测返回两者并集 31 条）；`1>6>25` 里的 `>` 是层级路径不是分隔符，不要混淆。
另：`1>6>22`（麦麦体验营）当前返回空属正常现象，不是调用错误。

```
③ mall-product-detail(spuId = <用户选的商品>)   // 查看 SKU 规格
```

### 4.1 积分抽奖（严格守规，这是合规重点）

```
① query-lottery-info()
   → 返回 drawDecision：
     - resourceEligible：本次资源是否满足
     - nextConsumption ：本次实际消耗
     - fallbackConsumption：兜底消耗（若为「先次数后积分」，需一并说明）
② 向用户展示 nextConsumption，并等待明确确认
③ draw-lottery()
```

**硬性规则**：

- 必须先展示 `drawDecision.nextConsumption` 并获得**明确确认**才能抽。
  「试试看」「如果可以就抽」**不算确认**。
- `resourceEligible=false` 时**不得调用** `draw-lottery`。
- **严禁**试抽、自动抽、连续抽。
- `availableTimes=0` **不代表不能抽**——混合规则下会改用积分，
  一切以 `drawDecision` 为准，不要自行推断。
- 接口报「操作太频繁」时直接把原文告诉用户，不要改写、不要重试。
- 中奖结果直接展示 `prizes` 与 `validDateInfo`。

### 4.2 体验活动报名（品鉴会 / 体验营 / 派对）

```
① mall-points-products(catRuleIds="1>6>25")   // 拿到 spuId
② mall-product-detail(spuId)                 // 确认 SKU，拿 skuId
③ query-party-city(spuId)                    // 取城市 code
④ query-party-store(code, latitude, longitude, spuId)
⑤ query-party-store-date(...)                // 取可选日期 dateStr
⑥ query-party-store-session(spuId, storeCode, dateStr)
   → 取场次 id / timeStart / timeEnd / leftNum
⑦ party-order-create(
     spuId, skuId, code, storeCode, dateStr,
     id, timeStart, timeEnd, leftNum, count,
     partyType                               // 1=包场，2=拼团
   )
```

`partyType` 判断规则：
- 商品 `partyType=1` → 只能包场
- 商品 `partyType=2` → 只能拼团
- 商品 `partyType=-1` → **必须询问用户**后再定

**每一步都要等用户明确选择再往下走，不得跳步。**

---

## 5. 输出

按 `references/output-templates.md` 的「优惠聚合卡片」渲染。要点：

- 领券结果：共领到几张，这单能用几张。
- 可用券逐张给出「能省多少」，并标出最划算的一张。
- 活动按剩余天数排序，学生相关的打标记。
- 积分给余额 + 能兑的 2-3 样东西。

---

## 6. 常见错误

- **拿 query-my-coupons 当可用券** → 会推荐用户用不了的券。
- **领了券不回流入计价** → 领券就变成没有价值的动作，这是本模块最容易做废的地方。
- **自动抽奖** → 会扣光用户积分，属严重违规。
- **把活动说成某门店专属** → 活动是全国/月度维度，不要按门店归因。
- **编造券 ID / 优惠金额** → 一律来自接口返回。
