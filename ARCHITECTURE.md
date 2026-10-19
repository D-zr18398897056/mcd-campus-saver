# 麦麦校园省钱搭子（mcd-campus-saver）架构设计

> 参赛 Skill · 基于 mcd-mcp 连接器 · 目标用户：大学生
> 状态：**设计阶段**，未写代码

---

## 0. 设计判断（先说结论）

这个 Skill 能不能跑通，瓶颈不在四个功能本身，而在 **mcd-mcp 的参数链强耦合**：

`beType` 决定 `orderType`，`beType ≠ 1` 时必须带 `beCode`，团餐还必须带 `gmServiceCode`，到店场景又必须带 `takeWayCode`。
漏一个就报错，多传一个（比如到店传 beCode）也报错。

所以架构的第一性设计不是「四个功能」，而是一套 **会话上下文总线**：四个模块共用同一份状态，任何模块写入，其他模块读取。这是整个 Skill 的地基。

第二个判断：**拼餐计算器应该用「企业团餐（beType=6）」实现，而不是麦乐送**。
理由：只有团餐场景能调 `query-promotions` 拿满减满折规则、能调 `query-meal-assistance` 拿助餐服务，而 `query-meals` 还内置了一套团餐搭配档位规则。这三样凑齐，才撑得起「输入预算+人数 → 输出最优方案」。用麦乐送做拼餐会退化成普通多点几份，没有差异化。

---

## 1. 会话上下文协议（核心基建）

每次会话维护一份状态，四个模块共享：

```
SessionContext {
  scene        : 到店自取 | 得来速 | 麦乐送 | 团餐
  beType       : 1 | 5 | 2 | 6
  orderType    : 1 | 2              // 由 beType 推导
  city         : string             // 学校所在城市
  storeCode    : string
  beCode       : string | null      // beType=1 时必须为 null
  addressId    : string | null      // 外送才有
  gmServiceCode: string | null      // 团餐才有
  takeWayCode  : string | null      // 到店才有
  partySize    : number             // 拼餐人数
  budget       : number             // 总预算（元）
  cart         : Item[]             // 商品 code + quantity
  bestCoupon   : { couponId, couponCode } | null
}
```

### 参数传递矩阵（写进 SKILL.md 置顶位置）

| 场景 | beType | orderType | beCode | addressId | gmServiceCode | takeWayCode | 门店来源 |
|---|---|---|---|---|---|---|---|
| 到店自取 | 1 | 1 | **禁止传** | — | — | 必传 | query-nearby-stores |
| 得来速 | 5 | 1 | 必传 | — | — | 必传 | query-nearby-stores |
| 麦乐送 | 2 | 2 | 必传 | 必传 | — | **禁止传** | delivery-query-stores |
| 企业团餐 | 6 | 2 | 必传 | 必传 | 必传 | **禁止传** | delivery-query-stores |

**参数来源链（必须逐级传递，不得凭空构造）：**

```
storeCode  ← query-nearby-stores（到店） 或 delivery-query-stores（外送）
beCode     ← 同上，与 storeCode 配对出现
addressId  ← delivery-query-addresses / delivery-create-address
gmServiceCode ← query-meal-assistance（团餐专属）
takeWayCode   ← calculate-price 返回的 data.takeWayList[].code
couponId/couponCode ← query-store-coupons
```

---

## 2. M1 校园门店雷达

**输入**：学校名 / 校区 / 地标关键词（如「武汉大学」「松江大学城」）

**工具链**
```
① now-time-info                    // 可选，确认当前时间用于营业状态判断
② query-nearby-stores(
     searchType = 2,               // 按位置搜索，必填
     city       = <城市>,          // searchType=2 时必填
     keyword    = <学校名/地标>,    // searchType=2 时必填
     beType     = 1 | 5            // 必填
   )
③ campaign-calendar                // 可选：给最近门店挂上当月活动标签
```

**关键设计点**

- **城市必须显式确认**。`city` 是必填项，而学生说「我们学校」往往不给城市。
  设计成：先尝试从上下文推断 → 推断不出就**一句话反问**「你在哪个城市？」，不要猜测。
- **beType 由用户意图决定**，不是默认 1：
  - 「走去 / 骑车去 / 到店吃」→ beType=1
  - 「开车去 / 打车去 / 得来速」→ beType=5
  - 「送到宿舍 / 外卖」→ 直接转 M2，不走 M1
- **距离排序**：接口返回的门店列表本身按位置相关性排列。
  Skill 内部按返回顺序呈现并标注「离你最近：XXX」。
  ⚠️ 若返回字段中**不含距离数值**，禁止编造「距离 800 米」这类数据，改用「最近 / 次近」的相对表述。
- **写入上下文**：用户选定门店后，把 `storeCode` + `beCode`（beType=5 时）+ `beType` + `city` 写入 SessionContext。

**输出模板**
```
📍 离你最近的 3 家麦当劳
1. 【最近】XX 餐厅 · XX 路 XX 号   ← 步行最近
2. XX 餐厅 · XX 商场 B1
3. XX 餐厅 · XX 校区南门

🏷️ 最近门店本月活动：1+1 随心配 / 麦金学期卡
👉 回复「到店自取」直接点，或说「送到宿舍」切外送
```

**异常处理**：接口报错或无结果 → 引导用户补充更具体的地标（如「XX 大学南门」），并提示改用外送按地址定位。

---

## 3. M2 外送自动定位

**输入**：宿舍楼地址 / 「我在 XX 宿舍」

**工具链**
```
① delivery-query-addresses()                  // 无参数，拉已有地址列表
   ├─ 用户选已有 → 取 addressId
   └─ 要新增 ↓
② delivery-create-address(
     city          = <真实城市名>,              // 必填，如「南京市」
     contactName   = <姓名>,                   // 必填
     gender        = 先生 | 女士,               // 必填（可从称呼推断）
     phone         = <11 位纯数字>,             // 必填，如 18616646686
     address       = <楼栋>,                   // 必填，如「XX 大学 7 号宿舍楼」
     addressDetail = <门牌>,                   // 必填，如「B 座 512 寝室」
   )
③ delivery-query-stores(
     addressId = <上一步的 addressId>,
     beType    = 2（麦乐送）| 6（团餐）         // 必填
   )
```

**关键设计点**

- **「自动定位」的真正含义**：不是每次都问地址，而是**首次问一次，之后整个会话自动携带**。
  M1 已经问过城市 → M2 直接复用 `city`，只补楼栋和门牌。
- **校园地址映射**：学生的口语表达要翻译成接口格式。
  「我在桂园 5 舍 302」→ address=「XX 大学桂园 5 舍」，addressDetail=「302 室」。
- **手机号校验**：必须是 11 位纯数字、1 开头。**不满足就反问，不要尝试修正**（比如别自作主张补 0）。
- **beType 分支**：这一步就决定了后面是普通外卖还是团餐拼单。
  用户在 M3 说「我们 5 个人拼」时，若当前是 beType=2，则**重新以 beType=6 调用 delivery-query-stores 取新的 beCode**（beCode 随场景变化，不能复用）。
- **写入上下文**：`addressId` + `storeCode` + `beCode` + `beType` + `orderType`。

**输出模板**
```
📦 配送到：XX 大学 7 号宿舍楼 B 座 512
🍔 该地址可配送门店 3 家，最近的是 XX 餐厅
👉 宿舍拼单直接说「5 个人拼，预算 150」
```

---

## 4. M3 多人拼餐计算器（核心亮点）

**输入**：人数 + 总预算（或人均预算）

**工具链**
```
① 收集输入
   人均预算 = 总预算 ÷ 人数

② query-promotions(
     storeCode = <ctx>, beCode = <ctx>,
     orderType = 2, beType = 6
   )
   → 拿满减满折规则（promotionType: 31=满额减, 33=订单折扣）
   → 推算「凑单门槛线」，例如「满 120 减 20」

③ query-meals(
     storeCode = <ctx>, beCode = <ctx>,
     orderType = 2, beType = 6
   )
   → 拿到全部餐品 + 商品 code + 价格
   → 按人均档位生成 3 套候选方案（见下方搭配规则）

④ query-meal-assistance(
     storeCode = <ctx>, beCode = <ctx>,
     orderType = 2, beType = 6
   )
   → 展示助餐服务让用户选 → 取 gmServiceCode（团餐必传）

⑤ calculate-price(...) × N 套方案      // 并行试算
   items = [{ productCode, quantity }]
   + gmServiceCode + beCode + orderType=2 + beType=6
   → 返回价格单位为「分」，必须 ÷100 转元

⑥ 比价选优 → create-order(...)
   + addressId + gmServiceCode + beCode + beType=6 + orderType=2
```

**团餐搭配档位规则（来自 query-meals 官方规则，Skill 必须遵循）**

| 人均预算 | 搭配模板 |
|---|---|
| < 20 元 | 小食 |
| 20-30 元 | 汉堡 + 小食 或 汉堡 + 饮料 |
| 30-40 元 | 汉堡 + 薯条/小食 + 饮料 |
| 40-50 元 | 汉堡 + 薯条 + 小食 + 饮料 |
| > 50 元 | 丰富组合，优先不重复小食 |

**三套候选方案生成策略**

| 方案 | 策略 | 目标用户 |
|---|---|---|
| A 省钱型 | 贴近凑单门槛下限，优先选满减后实付最低 | 真·贫困大学生 |
| B 均衡型 | 严格按人均档位模板，品类最均衡 | 大多数场景 |
| C 凑满减型 | 主动加一件凑到门槛，用「多买一件反而更便宜」的落差说服 | 差几块钱到门槛时 |

**输出模板（这是最打动评委的一屏）**
```
👥 5 人拼 · 预算 150 元 · 人均 30

方案 A 省钱型   ¥138 → 满减后 ¥118   人均 ¥23.6  ✅ 最省
  麦香鸡腿堡×2 + 麦辣鸡翅×2 + 中薯条×2 + 可乐×5

方案 B 均衡型   ¥150 → 满减后 ¥130   人均 ¥26.0
  板烧鸡腿堡×5 + 中薯条×3 + 可乐×5 + 麦辣鸡翅×2

方案 C 凑单型   ¥152 → 满减后 ¥120   人均 ¥24.0  ⭐ 多花 2 元多一件小食
  方案 B + 玉米杯×2（凑到满 150 减 30）

💡 选 C：比 B 多 2 件小食，总价反而少 10 元
💰 AA 一下：4 人付 ¥24，1 人付 ¥24

👉 回复「下单 C」继续
```

**降级策略**：人数 < 团餐门槛 或 预算过低时，自动降级为 beType=2 麦乐送多份单点，并明确告知用户「人数不够团餐门槛，已切换为麦乐送」。

---

## 5. M4 学生优惠聚合

**入口**：一键领券

**工具链**
```
① available-coupons()          // 列出可领券
② auto-bind-coupons()          // 一键全领（无需指定券 ID）

然后三条支线并行 ↓

A 券可用性筛选
   query-my-coupons()          // 卡包资产，仅展示，不校验可用性
   query-store-coupons(        // 真正的「当前订单能用」
     storeCode=<ctx>, beCode=<ctx>, orderType=<ctx>, beType=<ctx>
   )
   ⚠️ 关键：卡包里有 ≠ 这单能用，必须用 query-store-coupons 过滤

B 活动日历
   now-time-info()             // 取当前日期
   campaign-calendar(specifiedDate = "yyyy-MM-dd")   // 可选
   → 筛选学生相关：1+1 随心配 / 买一送一 / 麦金学期卡 / 早餐优惠

C 积分与体验活动
   query-my-account()                              // 积分余额
   mall-points-products(catRuleIds="1>4")          // 可兑换餐品券
   mall-points-products(catRuleIds="1>6>25")       // 品鉴会
   mall-points-products(catRuleIds="1>6>22")       // 麦麦体验营
   mall-product-detail(spuId)                      // 选 SKU

   延伸·抽奖（严格守规）
   query-lottery-info()        // 必须先展示 drawDecision.nextConsumption
   → 得到用户明确确认 → draw-lottery()
   ⚠️ 严禁自动抽、连续抽；availableTimes=0 不代表不能抽（可能改用积分）

   延伸·活动报名
   query-party-city → query-party-store → query-party-store-date
   → query-party-store-session → party-order-create

④ 回流：query-store-coupons 拿到的 couponId + couponCode
   传入 calculate-price 重新试算 → 取最省的一张
```

**关键设计点**

- **券闭环是差异化关键**：领券只是开始，必须走到「这单能用 → 试算 → 省了多少」才算闭环。
- **学生活动识别**：campaign-calendar 返回的是全量活动，Skill 需要做关键词筛选
  （1+1、买一送一、学期卡、学生、早餐、超值），并在输出中标注「学生党重点关注」。
- **抽奖是高风险操作**：工具明确要求先展示消耗、再获明确确认。
  Skill 必须写死这条规则，否则会连续抽光用户积分——这是评委一定会看的合规项。

**输出模板**
```
🎫 一键领券完成，共领到 6 张
   可用 3 张 · 暂不可用 3 张

✅ 这单能用的券
  · 满 50 减 10（外送专享）→ 用它能省 ¥10 ⭐ 推荐
  · 麦辣鸡翅免费券
  · 中薯条升大薯条

📅 本月学生党活动
  · 1+1 随心配  10/15 - 10/31  还剩 16 天
  · 麦金学期卡  长期有效        ¥9.9/月，含 6 张券
  · 早餐优惠    每日 6:00-10:30

💎 你的积分 1,240 → 可兑 麦香鱼券 / 2 个品鉴会名额
```

---

## 6. 意图路由表（SKILL.md 的入口决策）

| 用户说 | 路由到 | 首要动作 |
|---|---|---|
| 「我们学校附近有麦吗」 | M1 | query-nearby-stores(beType=1) |
| 「骑车去 / 到店吃」 | M1 | 确认 city + beType=1 |
| 「开车去」 | M1 | beType=5，注意取 beCode |
| 「送到宿舍 / 外卖」 | M2 | delivery-query-addresses |
| 「加个地址」 | M2 | delivery-create-address（5 个必填项） |
| 「5 个人拼，预算 150」 | M3 | 切团餐 → query-promotions |
| 「AA 一下」 | M3 | 人均 = 总价 ÷ 人数 |
| 「有什么优惠 / 帮我领券」 | M4 | available-coupons → auto-bind |
| 「最近有啥活动」 | M4 | now-time-info → campaign-calendar |
| 「我积分多少 / 能兑什么」 | M4 | query-my-account → mall-points-products |
| 「我想抽奖」 | M4 | query-lottery-info（先展示消耗） |
| 「下单 / 就是它了」 | 结算 | calculate-price → create-order |
| 「我的订单」 | 兜底 | order-list → query-order |

---

## 7. 易错点清单（必须写进 SKILL.md，每条都是踩过的坑）

1. **beType=1 到店自取，绝不传 beCode** —— 传了直接报错。
2. **外送场景绝不传 takeWayCode** —— 它只属于 orderType=1。
3. **storeCode 与 beCode 必须配对**，不能张冠李戴，且都来自对应场景的查询工具。
4. **价格单位是「分」**，展示前一律 ÷100。
5. **特调 key 规则**：选中的用 `selectedKey`；未选中但 `unselectedKey` 非空的项**也必须传入**。
6. **create-order 前必须先 calculate-price**（拿 takeWayCode 与准确价格）。
7. **团餐必须先选助餐服务**才能计价下单。
8. **切场景要重取 beCode**：beType 从 2 变 6，beCode 会变，不能复用旧值。
9. **delivery-create-address 的 city 是真实城市名**，phone 是 11 位纯数字。
10. **抽奖必须先展示消耗 + 明确确认**，禁止自动抽。
11. **query-my-coupons ≠ 可用券**，可用性一律以 query-store-coupons 为准。
12. **禁止编造距离、价格、券 ID**，拿不到就说拿不到。

---

## 8. Skill 文件结构（建议）

采用 **Capabilities-Based + Workflow 混合**结构：

```
mcd-campus-saver/
├── SKILL.md                          # 主入口：意图路由 + 上下文协议 + 易错点
└── references/
    ├── context-protocol.md           # 会话上下文 + 参数矩阵（最核心）
    ├── m1-campus-radar.md            # 门店雷达完整流程
    ├── m2-delivery-locate.md         # 外送定位完整流程
    ├── m3-group-meal.md              # 拼餐计算器（含档位规则 + 三方案策略）
    ├── m4-student-deals.md           # 优惠聚合（含券闭环 + 抽奖合规）
    └── output-templates.md           # 输出卡片模板（校园语气）
```

SKILL.md 保持精简（只放路由 + 矩阵 + 易错点），四个模块的细节放进 references 按需加载——
这样既不撑爆上下文，又能保证每个模块都有完整的调用规范。

---

## 9. 差异化亮点（评委视角）

| 亮点 | 说明 |
|---|---|
| **凑单反差** | 不是简单算总价，而是主动算「再买一件反而更便宜」，给出 C 方案——这是真正的"省钱"思维 |
| **团餐满减数据驱动** | 用 query-promotions 的原始数据推算凑单门槛，而不是拍脑袋 |
| **券闭环** | 领券 → 验证可用 → 回流入计价 → 算出实际省了多少，形成完整链路 |
| **AA 直达** | 拼餐场景天然需要分账，输出直接给每人转账金额 |
| **上下文免打扰** | 问过一次地址/城市，全程自动携带，不像通用对话那样反复追问 |

---

## 10. 待确认 / 风险项

设计阶段需要实测验证的点：

1. **query-nearby-stores 返回是否含距离字段** —— 决定 M1 能否做真实的距离排序，
   若不含则改为相对表述（目前设计已按「不确定」保守处理）。
2. **团餐 beType=6 是否有起送门槛/人数下限** —— 决定 M3 的降级策略触发条件。
3. **campaign-calendar 的活动数据是否含学生标签字段** —— 决定 M4 是关键词筛选还是字段筛选。
4. **mall-points-products 的 catRuleIds 复用多个类目是否生效** —— 涉及积分兑换支线的实现。
5. **大学城/校区类关键词的搜索召回率** —— 可能需要提示用户改用具体地标。

以上 5 项建议在实际联调时优先验证，它们会直接影响各模块的实现细节。
