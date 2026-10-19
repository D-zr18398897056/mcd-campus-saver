# workbuddy.md — WorkBuddy 开发对话记录

本项目「麦麦校园省钱搭子（mcd-campus-saver）」的全部开发过程均在腾讯 WorkBuddy 中真实完成，使用官方 mcd-mcp 自定义连接器（端点 `https://mcp.mcd.cn`，Bearer Token 认证）。

本文件为两次开发会话的对话上下文导出，供 WorkBuddy 联动活动奖励核验使用。两段对话均有 WorkBuddy 官方分享链接，可在线核对原文：

| 会话 | 时间 | 内容 | 官方分享链接 |
|---|---|---|---|
| 会话一 | 2026-10-19 23:48 ~ 2026-10-20 00:05 | 需求设计 → 架构文档 → SKILL.md + references/ 七个文件生成 | https://workbuddy.link/p/Dm31tDT2aC907B0yaYeo2J |
| 会话二 | 2026-10-20 00:25 ~ 00:52 | Skill 挂载 → 四模块真实冒烟测试 → README.md / MCP_INTEGRATION.md 生成 | https://workbuddy.link/p/iNQNY2oq0eKPlhOYqSmsIz |

说明：
- 对话中生成文件的完整定稿已收录本仓库（`SKILL.md`、`references/`、`ARCHITECTURE.md`、`README.md`、`MCP_INTEGRATION.md`、`SMOKE_TEST_REPORT.md`），本记录保留对话原文与关键决策。
- 标注「过程消息」处为 WorkBuddy 调用 mcd-mcp 工具的中间过程记录。
- 会话二全部为真实接口调用（未调用 create-order / draw-lottery），原始返回数据见 `SMOKE_TEST_REPORT.md`。

---

# 会话一：Skill 设计与生成

## 用户（2026-10-19 23:48）

> 我要参加麦当劳 MCP 创意开发大赛，开发一个参赛 Skill，名字叫「麦麦校园省钱搭子」（英文名 mcd-campus-saver），目标用户是大学生。
>
> 核心功能设计：
> 1. 校园门店雷达：用户输入学校名或位置关键词，调用工具查询附近麦当劳门店，按距离排序并标出最近门店；
> 2. 外送自动定位：查询已保存的配送地址、添加新地址，查询支持外送的门店，点餐时自动带上位置信息；
> 3. 多人拼餐计算器：输入预算和人数，查询菜单挑选组合，用计价工具算出含优惠的总价和人均价，给出最优拼餐方案；
> 4. 学生优惠聚合：一键查询可领优惠券并自动领券，展示近期活动日历（如买一送一、麦金学期卡等学生专属活动）。
>
> 请基于我已启用的 mcd-mcp 连接器里的工具，帮我完成功能架构设计：每个功能模块调用哪些工具、完整的调用流程。先做设计，先不要写代码。

## WorkBuddy（已完成 7m37s，含多次 mcd-mcp 工具调用——过程消息）

工具摸清了，先说结论：这个 Skill 能不能跑通，关键不在四个功能本身，而在于 **mcd-mcp 的参数链是强耦合的**——`beType` 决定 `orderType`，`beType≠1` 时必须带 `beCode`，团餐还必须带 `gmServiceCode`。所以四个模块必须共用一套会话上下文，否则每次调用都会漏参数报错。

（过程消息：参数矩阵设计）这个矩阵是整个 Skill 的"宪法"——任何一次工具调用的参数都对不齐，接口就直接报错。写进 Skill 时必须放在最显眼的位置。

（过程消息：拼餐链路设计）拼餐计算器是这个 Skill 最能拿分的部分——因为 mcd-mcp 里藏着一套现成的**团餐搭配规则**（≤20 元小食、20-30 汉堡+小食……），还有 `query-promotions` 的满减满折原始数据。这两样凑在一起，才撑得起"输入预算和人数 → 输出最优方案"。

（过程消息：优惠聚合设计）学生优惠聚合最容易做成"只列券"的平庸功能。要做出区分度，得把**领券 → 验证可用 → 回流入计价**做成闭环，再挂上活动日历和积分兑换两条支线。

设计文档已完成。有两个判断想先跟你对齐，因为它们会影响评委看到的东西：

**判断一：架构地基是「会话上下文总线」，不是四个功能。**

mcd-mcp 的参数链是强耦合的——`beType` 决定 `orderType`，`beType≠1` 必带 `beCode`，团餐还要 `gmServiceCode`，到店还要 `takeWayCode`，而且到店场景传 `beCode` 会直接报错。四个模块如果不共享同一份 `SessionContext`，每次调用都会漏参。所以 SKILL.md 的置顶内容应该是参数矩阵，而不是功能介绍。

**判断二：拼餐计算器必须走「企业团餐 beType=6」，不能用麦乐送。**

这是最能拉开差距的一点：只有团餐场景能调 `query-promotions` 拿满减满折原始数据、能调 `query-meal-assistance` 拿助餐服务，而 `query-meals` 还内置了一套人均档位搭配规则。这三样凑齐，「输入预算+人数 → 输出最优方案」才成立；用麦乐送做拼餐只会退化成"多点几份"。

基于这两点，四个模块的工具链是：

- **M1 门店雷达**：`query-nearby-stores(searchType=2, city, keyword, beType)` → 按返回序标注最近门店 → `campaign-calendar` 挂当月活动。注意 `city` 必填，学生说"我们学校"通常不给城市，要**反问而不是猜**。
- **M2 外送定位**：`delivery-query-addresses` → 无则 `delivery-create-address`（5 个必填项，phone 必须 11 位纯数字、不合规就反问别自己修）→ `delivery-query-stores` 拿 storeCode+beCode。切到团餐时 **beCode 会变，必须重取**。
- **M3 拼餐计算器**：`query-promotions`（算凑单门槛）→ `query-meals`（按人均档位出 3 套方案）→ `query-meal-assistance`（取 gmServiceCode）→ `calculate-price × N` 并行比价 → `create-order`。价格返回单位是**分**，一律 ÷100。
- **M4 优惠聚合**：`available-coupons` → `auto-bind-coupons` → **`query-store-coupons` 过滤**（卡包里有 ≠ 这单能用，这条是工具明说的）→ 券 ID 回流入 `calculate-price` 重算。支线挂 `campaign-calendar`、`mall-points-products`、抽奖。

三个容易被忽略但评委一定会看的点：

1. **抽奖合规**：必须先展示 `drawDecision.nextConsumption` 并拿到明确确认才能调 `draw-lottery`，且 `availableTimes=0` 不代表不能抽（可能改用积分）。这条要写死。
2. **凑单反差**：别只算总价，要主动给出 C 方案——"多买一件反而更便宜"。这才是"省钱搭子"该有的样子，也是这个 Skill 的记忆点。
3. **禁止编造**：距离、价格、券 ID 拿不到就说拿不到。

文档最后列了 **5 项待实测验证**（门店接口是否返回距离字段、团餐起送门槛、活动是否含学生标签、`catRuleIds` 多类目是否生效、大学城关键词召回率），建议联调时优先跑这几个，它们会直接改动各模块的实现细节。

另外提一句：你 `~/.workbuddy/skills/` 下有个 `mcd-deals-radar`，SKILL.md 还是模板 TODO 状态。它跟这个参赛项目是什么关系？如果是早期草稿，建议直接清理掉，免得跟 `mcd-campus-saver` 混淆。

设计先停在这里，你确认方向没问题（尤其是团餐那条），我再动手写 SKILL.md 和 references。

📎 产出：`ARCHITECTURE.md`（16.8 KB，含 10 个章节 + 第 10 节 5 项待验证清单）→ 已入库 [ARCHITECTURE.md](./ARCHITECTURE.md)

## 用户（2026-10-20 00:01）

> 架构通过。现在按第 8 节的文件结构，把 Skill 完整写出来：
> 1. SKILL.md：主入口，包含意图路由表（第 6 节）、会话上下文协议+参数矩阵（第 1 节，放在置顶位置）、易错点清单（第 7 节），保持精简；
> 2. references/ 下 6 个文件：context-protocol.md、m1-campus-radar.md、m2-delivery-locate.md、m3-group-meal.md、m4-student-deals.md、output-templates.md，按架构文档对应章节展开成完整流程规范。
>
> 要求：每个文件用独立的 Markdown 代码块完整输出，文件名标在代码块前，方便我直接保存。

## WorkBuddy（已完成 5m14s）

7 个文件已写入 `mcd-campus-saver/`，下面是完整内容，可直接复制保存。

（对话原文此处逐文件完整输出了 `SKILL.md` 与 `references/` 全部 6 个文件，包括 SKILL.md 的 YAML 触发描述、参数矩阵、来源链、意图路由表、12 条易错点清单、语气规范，以及 context-protocol.md 的 SessionContext 定义与灰色地带速查表等。定稿全文见仓库：[SKILL.md](./SKILL.md) 与 [references/](./references/) 六个文件，与对话输出一致。）

---

# 会话二：Skill 挂载与真实冒烟测试

## 用户（2026-10-20 00:25）

> 你应该已识别到我挂载的 Skill「mcd-campus-saver」。请加载它，按顺序做冒烟测试，全程真实调用 mcd-mcp 工具，不要网页搜索，不要编造数据：
> 1. 「我在广州大学城，附近有麦当劳吗」→ 走 M1 门店雷达；
> 2. 「送到宿舍」→ 走 M2 外送定位（个人信息用测试数据：城市「广州市」、联系人「测试同学」、电话 13800000000）；
> 3. 「5 个人拼，预算 150」→ 走 M3 拼餐计算器（切团餐 beType=6，拿到满减规则，走到 calculate-price 展示三套方案就停，禁止调用 create-order 真实下单）；
> 4. 「有什么优惠，帮我领券」→ 走 M4：available-coupons → auto-bind-coupons → campaign-calendar。
>
> 每一步展示：调用了什么工具、返回了什么原始数据、Skill 按输出模板给出了什么卡片。
> 同时顺带验证架构文档第 10 节的 5 个待验证项：①query-nearby-stores 返回有没有距离字段 ②团餐 beType=6 的起送门槛和人数下限 ③campaign-calendar 有没有学生标签字段 ④mall-points-products 的 catRuleIds 传多个值是否生效 ⑤「大学城」类关键词召回率。

## WorkBuddy（已完成 10m37s，全程真实调用 mcd-mcp，未调用 create-order / draw-lottery）

四个模块全跑通了，全部真实调用 mcd-mcp，**没有调 create-order / draw-lottery**。

### 1. M1 门店雷达

`now-time-info` + `query-nearby-stores(searchType=2, city=广州市, keyword=广州大学城, beType=1)`

返回 5 家，**都带 distance**：广工西生活区 811m / 中环西路创信园 998m / 药科大生活区 1150m / 华南理工 1234m / 中环东路创投园 1235m。beType=1 确实无 beCode 字段。现在 00:26，全部 `businessStatus=false`（07:00 才开门）。

```
📍 离你最近的 3 家麦当劳
1. 【最近】广工西生活区餐厅 · 大学城外环西路100号 · 811 米
2. 中环西路创信园餐厅 · 中环西路7号 · 998 米
3. 药科大生活区餐厅 · 大学城外环东路280号 · 1.15 km
👉 回复「到店自取」直接点，或说「送到宿舍」切外送
```

### 2. M2 外送定位

`delivery-query-addresses()` 返回空 → 直接 `delivery-create-address(广州市/测试同学/先生/13800000000/至善园6号宿舍楼/B座512寝室)` → `addressId=1036827600192209995798302260`（系统补全为「番禺区大学城外环东路132号 中山大学东校园 B座512寝室」）。

`delivery-query-stores(beType=2)` → 3 家，beCode 140023502 / 140089602 / 140094202；只有中二横路餐厅 24h 营业。外送返回**无 distance 字段**。

### 3. M3 拼餐（切团餐，未下单）

切 beType=6 重取 beCode：`140023581`（≠ 麦乐送的 `140023502`，易错点 8 实测成立）。助餐服务凌晨只有 GMS001 保鲜速达可用（GMS002/003 限 07:00–20:00 送达）。

**关键发现：query-promotions 12 条全是满折（ruleCategory=30），没有满减。门槛 300/500/1000/2000 元，对应 7.8/7.4/7.0/6.6 折。150 块够不着任何一档。**

calculate-price 三套（已 ÷100）：A ¥149.00（人均 29.80，差 ¥160 到门槛）／B ¥156.00（人均 31.20）／C 凑到 300 触发 7.8 折 ¥246.00（人均 49.20）。A/B 省下的 3.3 / 2.5 元其实只是打包费减免。C 凑单反差不成立，没硬推。

### 4. M4 优惠聚合

`available-coupons` 5 张 → `auto-bind-coupons` 成功 5 张失败 0 张（couponId/couponCode 全在报告里）→ `campaign-calendar(2026-10-20)`。

额外验证：团餐场景下 `query-store-coupons` 返回**空数组**——领的 5 张券一单都用不上，正好印证「卡包里有 ≠ 这单能用」。

### 5 个待验证项（看板已出）

①距离字段**有**（整数米）②团餐门槛 **300 元起、折扣制、无人数下限** ③活动日历**无学生标签字段**，只能关键词筛 ④catRuleIds **逗号分隔多值生效**（7+24=31 条并集）⑤「大学城」**召回良好**，5 家全中。

两个建议：文档里 `m1` 3.2 节可以从「不确定」改成「有距离、按米排序」；`m3` 降级策略应该写死 300 元这个数字，否则 150 元预算会误导用户去凑根本够不到的满减。这个 Skill 没有 `agent_created` 标记，我用 SkillManage 改不了，已在报告第 6 节列了 5 条回写建议。

📎 产出：`mcd-campus-saver-smoke-test-report.md`（13.2 KB）→ 已入库 [SMOKE_TEST_REPORT.md](./SMOKE_TEST_REPORT.md)
（报告第 6 节的 5 条回写建议，已由开发者在仓库 references/ 中逐条落实。）

## 用户（00:48 前后）

> 测试完成，规范也已按实测更新。现在基于实测结果，为项目生成两份参赛必交文档：
> 1. README.md：项目介绍（一句话+背景）、功能列表（四大模块）、安装方法（两种用法：①把 SKILL.md+references/ 放进 WorkBuddy skills 目录 ②用 mcp-config.example.json 配置 mcd-mcp 连接器，说明 MCD_MCP_TOKEN 环境变量）、使用示例（用冒烟测试的真实案例：广州大学城查店、5人拼150元、领5张券）、目标用户、目录结构说明（含 ARCHITECTURE.md、references/、SMOKE_TEST_REPORT.md）；
> 2. MCP_INTEGRATION.md：使用的 MCP Server（mcd-mcp，端点 https://mcp.mcd.cn ）、用到的全部 Tool 清单及用途表、M1-M4 各自调用流程、业务价值（学生省钱闭环：真实距离排序、团餐折扣门槛诚实提示、券可用性验证）。
>
> 每个文件用独立 Markdown 代码块完整输出，文件名标在代码块前。

## WorkBuddy（已完成 4m20s）

两份文档已写入项目目录 `mcd-campus-saver/`，下面是完整内容。

（对话原文此处完整输出了 `README.md` 与 `MCP_INTEGRATION.md` 全文，定稿见仓库：[README.md](./README.md)、[MCP_INTEGRATION.md](./MCP_INTEGRATION.md)。）
