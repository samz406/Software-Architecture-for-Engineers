# 第 32 章：支付系统架构设计

## 1. 从一个真实问题开始

大促当天，运营群里炸了："大量用户反馈支付后没有收到订单确认，但钱已经扣了！"

技术团队紧急排查。发现微信支付的回调通知因为流量过大被限流了，大量回调请求返回了 429 状态码。微信支付会重试，但重试间隔从一分钟递增到十分钟。也就是说，最晚的回调要十几分钟才能到达。

这十几分钟内，用户的钱已经被微信扣走了，但订单系统还不知道支付成功了。用户看到的订单状态是"待支付"。有些用户以为支付失败了，又重新支付了一次——同一笔订单付了两次钱。

紧急处理后，团队复盘了三个问题：
1. 为什么回调限流了？——没有对回调接口做容量评估，大促流量超出了预期。
2. 回调延迟时用户为什么能重复支付？——创建了新的支付单而不是查询已有支付单的状态。
3. 为什么不能主动查询支付结果？——没有做支付结果的主动轮询机制，完全依赖渠道回调。

**支付系统的特殊性在于：它处理的是钱。** 多扣了用户投诉，少扣了公司亏损，扣了但没记录就是资金差异。容不得半点马虎。

## 2. 问题为什么会变复杂

### 2.1 支付的不确定性

支付过程涉及多个外部系统：用户的银行、支付渠道（微信、支付宝）、商户账户。每个环节都可能出问题：

- 用户发起支付，请求发到微信。微信扣了钱，但回调通知没有发出来（网络故障）。
- 微信发了回调，但我方系统没收到（服务器重启、限流）。
- 我方收到了回调，处理到一半崩溃了（写了支付记录但没更新订单状态）。

这种"操作已经执行但结果不确定"的状态，是支付系统最大的挑战。

### 2.2 和钱相关的零容忍

订单数据丢一条可以人工补录。支付数据差一分钱都要查清楚。

财务对账要求支付系统的每一笔交易都有完整的记录：什么时候发起、什么时候到账、通过什么渠道、金额是多少。任何一个环节的数据缺失或不一致，都会导致对账差异。

### 2.3 多渠道的复杂性

不同的支付渠道（微信、支付宝、银联、Apple Pay）有不同的接口、不同的回调格式、不同的状态码、不同的签名方式。每接入一个新渠道，就要写一套适配逻辑。

如果没有统一的抽象层，渠道相关的代码会散落在系统各处。

## 3. 核心概念解释

### 3.1 支付单的数据模型

支付系统的核心实体是"支付单"——一次支付行为的完整记录。

```sql
-- 支付单
CREATE TABLE pay_order (
    pay_order_id VARCHAR(32) PRIMARY KEY,   -- 支付单号
    biz_order_id VARCHAR(32) NOT NULL,      -- 业务订单号
    biz_type VARCHAR(20) NOT NULL,          -- 业务类型（ORDER/RECHARGE/...）
    
    user_id BIGINT NOT NULL,
    amount DECIMAL(10,2) NOT NULL,          -- 支付金额
    currency VARCHAR(10) DEFAULT 'CNY',
    
    channel VARCHAR(20) NOT NULL,           -- 支付渠道
    channel_order_id VARCHAR(64),           -- 渠道方的交易号
    
    status VARCHAR(20) NOT NULL,            -- CREATED/PAYING/SUCCESS/FAILED/CLOSED
    
    -- 时间
    created_at DATETIME NOT NULL,
    paid_at DATETIME,
    expired_at DATETIME,                    -- 支付超时时间
    
    -- 回调
    notify_url VARCHAR(500),                -- 回调地址
    notify_status VARCHAR(20),              -- PENDING/SUCCESS/FAILED
    notify_count INT DEFAULT 0,
    
    version INT NOT NULL DEFAULT 0,
    
    INDEX idx_biz (biz_order_id, biz_type),
    INDEX idx_channel (channel, channel_order_id),
    INDEX idx_status_time (status, created_at)
);

-- 支付流水（每次状态变更的记录）
CREATE TABLE pay_order_log (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    pay_order_id VARCHAR(32) NOT NULL,
    from_status VARCHAR(20),
    to_status VARCHAR(20) NOT NULL,
    channel_response TEXT,                   -- 渠道返回的原始数据
    operator VARCHAR(50),
    created_at DATETIME NOT NULL,
    INDEX idx_pay_order (pay_order_id)
);

-- 退款单
CREATE TABLE refund_order (
    refund_order_id VARCHAR(32) PRIMARY KEY,
    pay_order_id VARCHAR(32) NOT NULL,      -- 原支付单号
    biz_order_id VARCHAR(32) NOT NULL,
    
    refund_amount DECIMAL(10,2) NOT NULL,
    reason VARCHAR(200),
    
    channel VARCHAR(20) NOT NULL,
    channel_refund_id VARCHAR(64),
    
    status VARCHAR(20) NOT NULL,            -- CREATED/PROCESSING/SUCCESS/FAILED
    
    created_at DATETIME NOT NULL,
    refunded_at DATETIME,
    version INT NOT NULL DEFAULT 0,
    
    INDEX idx_pay_order (pay_order_id)
);
```

### 3.2 支付状态机

```
创建(CREATED)
  → 支付中(PAYING)      [用户发起支付，调用渠道下单接口]
  → 已关闭(CLOSED)      [超时未支付]

支付中(PAYING)
  → 支付成功(SUCCESS)   [收到渠道回调或主动查询确认]
  → 支付失败(FAILED)    [渠道明确返回失败]
  → 已关闭(CLOSED)      [超时未完成]

支付成功(SUCCESS)
  → 退款中(REFUNDING)   [发起退款]
```

关键规则：
- **PAYING 状态只能通过渠道结果来流转**。不能因为超时就直接标为 FAILED——可能渠道已经扣款了。超时应该先主动查询渠道，确认后再决定状态。
- **SUCCESS 是终态**（除非退款）。一旦标记为 SUCCESS，就不能再改回 PAYING 或 FAILED。

### 3.3 幂等和防重

支付系统必须处理两种重复场景：

**用户重复发起支付**：用户点了"支付"按钮后页面没反应，又点了一次。两次请求不能创建两笔支付。

解决方法：同一个业务订单号只允许有一笔"进行中"的支付单。发起支付时先查是否已有 PAYING 状态的支付单，有的话返回已有的支付链接。

**渠道重复回调**：微信支付可能对同一笔交易发送多次回调通知。

解决方法：支付回调处理必须幂等。收到回调后先检查支付单状态，如果已经是 SUCCESS 就直接返回成功，不重复处理。

```java
@Transactional
public void handlePayCallback(PayCallbackRequest callback) {
    PayOrder payOrder = payOrderRepository.findByPayOrderId(callback.getPayOrderId());
    
    // 幂等检查：已经处理过了
    if (payOrder.getStatus() == PayStatus.SUCCESS) {
        return; // 直接返回，不重复处理
    }
    
    // 状态机检查：只有 PAYING 状态可以流转到 SUCCESS
    if (payOrder.getStatus() != PayStatus.PAYING) {
        log.warn("支付单状态不对: {}, 当前状态: {}", payOrder.getPayOrderId(), payOrder.getStatus());
        return;
    }
    
    // 验签：确认回调确实来自渠道方
    if (!channelAdapter.verifySignature(callback)) {
        throw new SecurityException("回调验签失败");
    }
    
    // 金额校验：回调金额和支付单金额一致
    if (!payOrder.getAmount().equals(callback.getAmount())) {
        log.error("金额不一致: 支付单={}, 回调={}", payOrder.getAmount(), callback.getAmount());
        throw new BusinessException("金额不一致");
    }
    
    // 更新状态（乐观锁）
    payOrder.setStatus(PayStatus.SUCCESS);
    payOrder.setChannelOrderId(callback.getChannelOrderId());
    payOrder.setPaidAt(callback.getPaidAt());
    int updated = payOrderRepository.updateWithVersion(payOrder);
    if (updated == 0) {
        throw new ConcurrentModifyException("并发更新冲突");
    }
    
    // 通知业务方（异步）
    eventPublisher.publish(new PaySuccessEvent(payOrder));
}
```

### 3.4 主动轮询机制

不能完全依赖渠道的回调通知。回调可能延迟、丢失或被限流。需要有主动轮询作为补充。

```java
// 定时任务：每分钟扫描处于 PAYING 状态超过 5 分钟的支付单
@Scheduled(fixedDelay = 60000)
public void pollPayingOrders() {
    List<PayOrder> payingOrders = payOrderRepository
        .findByStatusAndCreatedBefore(PayStatus.PAYING, 
            LocalDateTime.now().minusMinutes(5));
    
    for (PayOrder order : payingOrders) {
        try {
            // 主动查询渠道支付结果
            ChannelQueryResult result = channelAdapter.queryPayResult(
                order.getChannel(), order.getPayOrderId());
            
            if (result.isSuccess()) {
                handlePayCallback(result.toCallbackRequest());
            } else if (result.isFailed()) {
                closePayOrder(order);
            }
            // 如果渠道返回"处理中"，不做操作，等下次轮询
        } catch (Exception e) {
            log.warn("轮询支付结果失败: {}", order.getPayOrderId(), e);
        }
    }
}
```

回调 + 轮询双保险：回调是主要的通知方式（实时性好），轮询是兜底（可靠性高）。

### 3.5 渠道抽象

不同支付渠道的接口差异很大，需要一个统一的抽象层：

```java
public interface PayChannelAdapter {
    // 创建支付（返回支付链接或支付参数）
    CreatePayResult createPay(CreatePayRequest request);
    
    // 查询支付结果
    ChannelQueryResult queryPayResult(String payOrderId);
    
    // 验证回调签名
    boolean verifySignature(PayCallbackRequest callback);
    
    // 发起退款
    RefundResult refund(RefundRequest request);
    
    // 查询退款结果
    RefundQueryResult queryRefundResult(String refundOrderId);
}
```

每个渠道实现这个接口：WechatPayAdapter、AlipayAdapter、UnionPayAdapter。业务代码不直接和渠道打交道，只和抽象接口打交道。

### 3.6 对账

对账是支付系统的"最后一道防线"——确保我方记录的支付数据和渠道方的数据一致。

每天定时从渠道下载前一天的交易明细（账单文件），和我方的支付记录逐笔比对：

- **我方有、渠道有**：正常。金额一致就通过。
- **我方有、渠道没有**：我方可能误记了一笔成功的支付。需要人工核查。
- **我方没有、渠道有**：可能是回调丢失了。需要补录支付成功记录。
- **金额不一致**：严重问题，需要立即排查。

对账差异应该有告警机制，发现差异后及时处理。

## 4. 常见误区

### 4.1 完全依赖回调

"渠道会发回调通知我们的。"——但回调可能延迟、丢失、被限流。必须有主动轮询作为兜底。

### 4.2 不做金额校验

收到回调后直接更新状态，不校验回调中的金额和支付单的金额是否一致。攻击者可能伪造回调，用一分钱"支付"一千元的订单。

每次处理回调都要做两件事：验签（确认是渠道发的）+ 金额校验（确认金额正确）。

### 4.3 支付超时直接标失败

用户发起支付后，五分钟没有收到回调，就把支付单标为失败。但渠道可能已经扣款了——五分钟后回调到达时，发现支付单已经是"失败"状态，回调无法处理。用户的钱扣了但订单没有变成"已支付"。

超时后应该先查询渠道确认结果，再决定是关闭还是标为成功。

### 4.4 退款没有独立管理

把退款逻辑放在订单服务里，退款状态用订单状态字段表示。导致一个订单只能退一次款，且退款和订单状态纠缠在一起。

退款应该有独立的退款单和状态机。一个订单可以有多笔退款（部分退、多次退）。退款的状态独立管理，不影响订单状态。

### 4.5 不做对账

"系统运行正常，数据应该没问题。"——直到某天发现有十几笔交易的状态和渠道不一致。

对账是必须的。不是"怀疑有问题才对账"，而是"每天都对账"。对账是发现问题的手段，不是解决问题后的验证。

## 5. 设计方法

### 5.1 支付系统的模块划分

```
支付系统
├── 支付核心：创建支付单、状态机管理、幂等处理
├── 渠道适配：各支付渠道的接口封装
├── 回调处理：接收渠道回调、验签、分发
├── 轮询补偿：主动查询支付结果的定时任务
├── 退款处理：退款单管理、退款状态跟踪
├── 对账：账单下载、逐笔比对、差异处理
└── 风控：金额校验、频率限制、异常检测
```

### 5.2 支付系统的稳定性设计

| 措施 | 说明 |
|-----|------|
| 渠道隔离 | 每个渠道独立线程池，一个渠道超时不影响其他 |
| 渠道熔断 | 单个渠道失败率超过 50% 时熔断，引导用户选其他渠道 |
| 回调限流 | 回调接口设置限流，防止大促时被打垮 |
| 异步通知 | 支付结果通过事件异步通知业务方，不阻塞支付核心流程 |
| 数据冗余 | 支付相关的操作全量记录流水日志，出问题可追溯 |

## 6. 业务案例

### 6.1 案例：支付系统的三次关键改造

**第一次改造：加主动轮询**

上线初期，完全依赖微信回调。一次网络故障导致回调延迟一小时，几百笔支付的订单状态没有更新。用户看到"待支付"但钱已经扣了。

改造：增加了主动轮询机制——PAYING 状态超过 5 分钟的支付单，每分钟主动查询渠道结果。

改造后，即使回调完全丢失，最多 6 分钟内就能通过轮询获取到支付结果。

**第二次改造：渠道抽象和隔离**

接入支付宝后，发现微信和支付宝的回调处理代码高度重复，且微信渠道超时会影响支付宝的处理（共享线程池）。

改造：抽象出 PayChannelAdapter 接口，每个渠道独立实现。每个渠道独立线程池和熔断器。

改造后，微信渠道故障时，支付宝渠道完全不受影响。新接入银联渠道只需要实现 Adapter 接口，核心代码不用动。

**第三次改造：自动化对账**

运营发现有些退款在渠道方已经成功了，但我方的退款单还是"处理中"。原因是退款回调丢失了，且退款没有做主动轮询。

改造：增加退款轮询机制。同时实现自动化对账——每天凌晨下载各渠道的交易和退款账单，自动逐笔比对，差异记录发送到告警群。

改造后，对账差异从平均每天三到五笔降到零。偶尔出现差异也能在当天发现并处理。

## 7. 架构判断清单

- 如果支付系统没有主动轮询机制，完全依赖渠道回调，说明存在回调丢失后无法自愈的风险。
- 如果支付回调没有做验签和金额校验，存在被伪造回调攻击的风险。
- 如果同一笔业务订单可以创建多笔进行中的支付单，说明防重逻辑有缺陷。
- 如果支付超时后直接标为失败而不先查询渠道结果，可能导致"钱扣了但订单未更新"。
- 如果没有每日对账机制，支付数据的一致性无法保证。
- 如果不同渠道共享线程池，一个渠道的故障会拖垮所有渠道。
- 如果退款没有独立的状态管理，退款流程会和订单状态耦合。

## 8. 本章小结

支付系统的核心挑战是**处理不确定性**。支付过程中的每个环节都可能失败或延迟，系统必须在这些不确定性中保证数据最终一致。

关键的设计原则：幂等（同一操作执行多次结果一致）、兜底（回调+轮询双保险）、对账（每日核对发现差异）、隔离（渠道之间互不影响）。

支付系统的每一个设计决策都围绕一个核心问题：**如果这一步出了问题，钱会不会出错？** 围绕这个问题设计超时、重试、幂等、对账机制，是支付系统架构的根本方法。
