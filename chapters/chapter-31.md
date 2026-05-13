# 第 31 章：优惠券系统架构设计

## 1. 从一个真实问题开始

营销团队找到技术负责人："下个月我们要做一个大促活动，需要发一批优惠券。满 200 减 30，限新用户使用，每人限领一张，总共发 10 万张。"

工程师一听，不复杂嘛。一张优惠券表，有金额、门槛、使用条件、库存、有效期。领券接口检查条件后发一张，下单时校验并核销。两周搞定。

两周后上线了。然后接下来的六个月，每个月产品都来加新需求：

第一个月："加一种折扣券，打八折。"——改优惠计算逻辑。
第二个月："优惠券要支持指定品类使用，比如只能买食品类。"——加品类过滤。
第三个月："做一个邀请好友送券的活动，分享链接后好友注册就给双方各发一张。"——加发券渠道。
第四个月："支持满减阶梯：满 100 减 10、满 200 减 30、满 500 减 80。"——改优惠计算模型。
第五个月："做一个凑单满减的功能，用户加购后自动计算需要凑多少钱能用券。"——加反向计算逻辑。
第六个月："大促当天同一张券被黄牛脚本刷了八千张，库存一分钟就没了。"——加防刷。

每个月改一次优惠券模块。每次改完都在原来的代码上加 if-else。六个月后优惠券的核心代码变成了一千多行的大泥潭，每次改动都有引入 bug 的风险。

**优惠券系统表面是"发券-领券-用券"三步，实际上是规则最复杂、变化最频繁、和钱直接相关的模块之一。**

## 2. 问题为什么会变复杂

### 2.1 规则的排列组合爆炸

一张优惠券可能有以下维度的规则：

- **类型**：满减、折扣、随机金额、运费减免
- **使用门槛**：无门槛、满 X 元可用
- **适用范围**：全品类、指定品类、指定商品、指定店铺
- **适用人群**：所有用户、新用户、指定等级会员
- **时间限制**：固定有效期、领取后 N 天有效
- **叠加规则**：和其他优惠券能不能叠加、和平台活动能不能叠加

这些维度两两组合就是大量的规则场景。每新增一个维度，规则的复杂度就翻一倍。

### 2.2 和钱直接相关

优惠券的计算错误直接导致资金损失。少算了——用户投诉。多算了——公司亏钱。退款时优惠券的分摊逻辑算错了——财务对不上账。

这种"和钱相关"的特性让优惠券系统的容错空间非常小。

### 2.3 营销节奏快

营销活动的节奏比技术迭代快得多。运营可能提前三天说"后天有个活动要发一批新类型的券"。如果优惠券系统的扩展性不好，每种新券都要改代码，就会变成团队的瓶颈。

## 3. 核心概念解释

### 3.1 优惠券的数据模型

优惠券系统的核心概念是两层结构：**券模板**和**券实例**。

券模板定义一类优惠券的规则。券实例是用户实际持有的那一张券。

```sql
-- 券模板：定义一类券的规则
CREATE TABLE coupon_template (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,              -- "新人满200减30"
    type VARCHAR(20) NOT NULL,               -- FULL_REDUCE/DISCOUNT/RANDOM
    
    -- 优惠内容
    discount_amount DECIMAL(10,2),           -- 满减金额
    discount_rate DECIMAL(4,2),              -- 折扣率（如0.80表示八折）
    max_discount DECIMAL(10,2),              -- 折扣券最大优惠金额
    
    -- 使用门槛
    min_order_amount DECIMAL(10,2),          -- 最低消费金额
    
    -- 适用范围
    scope_type VARCHAR(20) NOT NULL,         -- ALL/CATEGORY/PRODUCT/SHOP
    scope_ids TEXT,                           -- 适用的品类/商品/店铺ID列表
    
    -- 适用人群
    user_type VARCHAR(20) DEFAULT 'ALL',     -- ALL/NEW/MEMBER_LEVEL
    user_condition TEXT,                      -- 人群条件的JSON描述
    
    -- 库存
    total_count INT NOT NULL,                -- 总发行量
    issued_count INT NOT NULL DEFAULT 0,     -- 已发放数量
    
    -- 时间
    valid_type VARCHAR(20) NOT NULL,         -- FIXED（固定时间段）/ RELATIVE（领取后N天）
    valid_start DATETIME,                    -- 固定开始时间
    valid_end DATETIME,                      -- 固定结束时间
    valid_days INT,                          -- 领取后有效天数
    
    -- 限制
    per_user_limit INT DEFAULT 1,            -- 每人限领数量
    
    status VARCHAR(20) NOT NULL DEFAULT 'DRAFT',
    created_at DATETIME NOT NULL,
    version INT NOT NULL DEFAULT 0
);

-- 券实例：用户持有的具体一张券
CREATE TABLE coupon_instance (
    id BIGINT PRIMARY KEY,
    template_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    
    status VARCHAR(20) NOT NULL,             -- AVAILABLE/FROZEN/USED/EXPIRED/RETURNED
    
    valid_start DATETIME NOT NULL,           -- 这张券的有效期开始
    valid_end DATETIME NOT NULL,             -- 这张券的有效期结束
    
    -- 使用信息
    used_order_id VARCHAR(32),               -- 核销时的订单号
    used_at DATETIME,
    
    -- 冻结信息
    frozen_order_id VARCHAR(32),             -- 冻结时的订单号
    frozen_at DATETIME,
    
    created_at DATETIME NOT NULL,
    
    INDEX idx_user_status (user_id, status),
    INDEX idx_template (template_id),
    UNIQUE INDEX uk_freeze (frozen_order_id, template_id)
);
```

### 3.2 券的生命周期

```
创建(由系统发放或用户领取)
  → 可用(AVAILABLE)
    → 冻结(FROZEN)    [用户下单时冻结]
      → 已使用(USED)   [支付成功后核销]
      → 可用(AVAILABLE) [取消订单后解冻]
    → 已过期(EXPIRED)  [到达有效期]
  → 已退回(RETURNED)   [退款后退还]
```

冻结状态的意义：用户下单时选择了优惠券，但还没支付。此时券进入冻结状态——不能被其他订单使用，但也没有真正核销。如果用户取消订单或支付超时，券解冻回到可用状态。

### 3.3 优惠计算的设计

优惠计算是优惠券系统最复杂的部分。推荐用策略模式处理不同的券类型：

```java
public interface CouponCalculator {
    CouponType getType();
    CalculateResult calculate(Order order, CouponInstance coupon, CouponTemplate template);
}

public class FullReduceCalculator implements CouponCalculator {
    public CouponType getType() { return CouponType.FULL_REDUCE; }
    
    public CalculateResult calculate(Order order, CouponInstance coupon, 
                                     CouponTemplate template) {
        // 1. 筛选适用的商品
        List<OrderItem> applicableItems = filterByScope(order.getItems(), template);
        
        // 2. 计算适用商品的总金额
        Money applicableAmount = sumAmount(applicableItems);
        
        // 3. 判断是否满足门槛
        if (applicableAmount.lt(template.getMinOrderAmount())) {
            return CalculateResult.notMet("未满足最低消费金额");
        }
        
        // 4. 计算优惠金额（不超过适用商品总金额）
        Money discount = Money.min(template.getDiscountAmount(), applicableAmount);
        
        // 5. 按商品金额比例分摊优惠
        Map<Long, Money> itemDiscounts = allocateDiscount(applicableItems, discount);
        
        return CalculateResult.success(discount, itemDiscounts);
    }
}
```

**优惠分摊**是一个容易被忽视的关键设计。一张"满 200 减 30"的券，订单有三件商品（100 元、60 元、40 元）。优惠的 30 元怎么分摊到每件商品上？

按金额比例分摊：100 元的商品分摊 15 元，60 元的分摊 9 元，40 元的分摊 6 元。

为什么要分摊？因为退款时需要知道退这件商品应该退多少优惠。如果用户退了那件 100 元的商品，应该退 15 元的优惠额度，而不是 30 元。

### 3.4 防刷设计

优惠券是高频被黄牛攻击的目标。常见的防刷手段：

**频率限制**：同一用户/IP 每秒最多领取 N 次。

**库存扣减的原子性**：用 Redis 的原子操作或数据库的乐观锁扣减库存，防止超发。

```java
// Redis 原子扣减库存
String key = "coupon:stock:" + templateId;
Long remaining = redisTemplate.opsForValue().decrement(key);
if (remaining < 0) {
    // 库存不足，回滚
    redisTemplate.opsForValue().increment(key);
    throw new BusinessException("券已领完");
}
// Redis 扣减成功后，再做数据库操作（异步或同步）
```

**领取资格校验**：检查用户是否已经领过、是否是新用户、设备指纹是否可疑。

**验证码**：对高价值券的领取加人机验证。

## 4. 常见误区

### 4.1 规则硬编码在代码里

每加一种新的使用限制就加一个 if 判断。十种限制条件就是十个 if 的嵌套。

应该把使用规则做成可配置的：模板中定义规则类型和参数，代码中用规则引擎或策略模式解析执行。新增一种限制条件是加一个策略类，不是改核心代码。

### 4.2 不做优惠分摊

下单时只记录"这个订单用了 30 元优惠"，不记录每件商品分摊了多少。退款时不知道怎么退。

应该在下单时就计算并记录每件商品的优惠分摊金额。

### 4.3 冻结和核销不分

下单时直接核销优惠券（标记为已使用）。用户取消订单后要把券改回可用。但如果取消操作失败了，券就丢了。

应该分两步：下单时冻结（可逆），支付成功后核销（不可逆）。取消订单解冻比"撤销核销"简单得多。

### 4.4 库存扣减不是原子的

先查库存 `SELECT remaining`，再减库存 `UPDATE remaining = remaining - 1`。高并发时一百个人同时查到"还剩一张"，然后都扣减成功——超发了九十九张。

库存扣减要用原子操作：`UPDATE SET remaining = remaining - 1 WHERE remaining > 0`，或者用 Redis 的原子 decrement。

### 4.5 有效期处理不到位

用户领券后一直不用，券到了有效期却还是"可用"状态。导致用户下单时选了过期券，到支付时才报错。

需要定时任务扫描过期券并标记为"已过期"。或者在每次查询用户可用券时实时检查有效期。

## 5. 设计方法

### 5.1 优惠券系统的模块划分

```
优惠券系统
├── 模板管理：创建、编辑、上下架券模板
├── 发券引擎：系统发券、活动发券、定向发券
├── 领券服务：用户主动领取、频率控制、库存扣减
├── 核销服务：冻结、核销、解冻、退回
├── 计算引擎：优惠金额计算、优惠分摊、凑单计算
├── 过期处理：定时任务标记过期券
└── 查询服务：用户可用券列表、可用券推荐
```

### 5.2 与订单系统的交互协议

订单系统和优惠券系统的交互有四个节点：

| 节点 | 操作 | 调用方式 | 失败处理 |
|-----|------|---------|---------|
| 下单 | 冻结优惠券 | 同步 | 冻结失败则下单失败 |
| 支付成功 | 核销优惠券 | 异步事件 | 失败重试 |
| 取消订单 | 解冻优惠券 | 异步事件 | 失败重试 |
| 退款 | 退回优惠券 | 异步事件 | 失败重试 |

冻结是同步的——因为需要立即确认券是否可用。核销、解冻、退回是异步的——即使延迟几秒也不影响用户体验。

## 6. 业务案例

### 6.1 案例：从"一个 if"到"规则引擎"的优惠券系统演进

**第一版（满减券）**：一种券类型，一个 if 判断。

**第二版（多类型）**：加了折扣券、随机金额券。每种类型的计算逻辑不同，用策略模式拆分。

**第三版（使用范围）**：券可以限定品类或商品。在计算前加了一层"适用商品筛选"逻辑。

**第四版（人群限制）**：券可以限定用户群体。在领券和核销时加了"用户资格检查"逻辑。

**第五版（叠加规则）**：一个订单可以用多张券，但有叠加限制（满减券和折扣券不能同时用）。在计算引擎中加了"叠加规则检查"。

到第五版时，使用条件的组合已经非常多。产品提新需求（比如"仅限周末使用""仅限某个渠道下单时使用"）时，需要改代码加条件。

**第六版（规则配置化）**：把使用条件抽象为"规则"，每种规则是一个独立的检查器。模板上通过 JSON 配置需要哪些规则和参数。新增一种使用条件只需要开发一个新的规则检查器并注册，不改核心代码。

```json
{
  "rules": [
    {"type": "MIN_ORDER_AMOUNT", "params": {"amount": 200}},
    {"type": "CATEGORY_LIMIT", "params": {"categoryIds": [101, 102]}},
    {"type": "USER_TYPE", "params": {"type": "NEW"}},
    {"type": "TIME_RANGE", "params": {"weekdays": [6, 7]}}
  ]
}
```

不是一开始就需要做规则引擎。前三版的 if-else 和策略模式完全够用。到第五版规则组合开始爆炸时，才需要规则引擎。

## 7. 架构判断清单

- 如果新增一种优惠券类型需要修改核心计算逻辑，说明计算引擎的扩展性不够。
- 如果退款时不知道优惠金额怎么分摊，说明下单时没有记录分摊明细。
- 如果优惠券库存在高并发时会超发，说明库存扣减不是原子的。
- 如果用户下单选了券但取消后券消失了，说明冻结和核销没有分开。
- 如果优惠券的使用规则散落在多个模块的代码中（订单模块、支付模块都有判断），说明规则检查没有集中管理。
- 如果无法回答"这张券什么时候发的、谁领的、在哪个订单用的、退款后退没退回来"，说明券的生命周期追踪不完整。

## 8. 本章小结

优惠券系统的核心挑战是**规则复杂性**和**与钱相关的精确性**。

数据模型的关键是"模板+实例"两层结构。状态管理的关键是区分冻结和核销。计算引擎的关键是策略模式加优惠分摊。扩展性的关键是规则配置化。

和订单系统类似，优惠券系统也应该从简单开始逐步演进。第一天就做规则引擎是过度设计。从策略模式开始，当规则组合开始爆炸时再引入规则引擎——这个演进路径在大多数团队中都适用。
