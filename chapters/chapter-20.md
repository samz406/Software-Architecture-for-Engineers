# 第 20 章：DDD：不要神化，也不要忽视

## 1. 从一个真实问题开始

一个做了三年的 SaaS 系统，核心是帮商家管理订单、库存和会员。系统的后端代码主要由 Service 层承载——OrderService 有将近四千行代码，CouponService 三千多行，UserService 两千多行。

这些 Service 的典型写法是这样的：一个公共方法对应一个业务操作，方法里面从头到尾顺序执行：参数校验 → 查数据库 → 业务判断 → 改数据库 → 调外部接口 → 发消息。每个方法几十到几百行。

问题逐渐积累：

- 产品想加一个"会员专属价"功能，需要在下单时判断用户的会员等级。工程师发现 OrderService 里已经有了价格计算逻辑、优惠券逻辑、满减逻辑，现在再加会员价逻辑，整个方法快 300 行了。
- 退款的时候需要回退积分、退还优惠券、释放库存。这些逻辑散落在 RefundService 里，但积分的计算规则在 PointService 里，优惠券的退还规则在 CouponService 里。RefundService 直接调用了它们的内部方法，耦合很深。
- 新来的工程师花了两周才理清"一个订单从创建到完成到底经历了哪些步骤"，因为逻辑分散在七八个 Service 里，没有一个地方能看到全貌。

技术负责人说"我们要用 DDD 重构"。于是团队花了两个月学习 DDD 理论，画了领域模型图，定义了一堆聚合根和领域事件。但重构开始后发现：

- 开发效率短期内下降了，因为大家不习惯新的分层和编码方式；
- 简单的 CRUD 功能也要经过 Application Service → Domain Service → Aggregate → Repository 四层，改一个字段要动四个文件；
- 团队对"限界上下文的边界到底怎么划"争论不休，最后每个人的理解都不一样。

三个月后，部分模块用了 DDD 的方式重构，部分模块还是老的 Service 模式，代码库变成了两种风格混合。

这个故事既说明了 DDD 想要解决的真实问题（Service 层变成垃圾桶、业务逻辑散落各处），也说明了 DDD 落地时常见的困难（学习成本高、过度仪式化、团队理解不一致）。

## 2. 问题为什么会变复杂

### 2.1 传统分层架构的瓶颈

大多数 Java/Go/Python 后端项目使用的是经典的三层架构：Controller → Service → DAO。这种架构在业务简单时很好用：Service 里写业务逻辑，DAO 负责数据访问，清晰明了。

但随着业务复杂度上升，Service 层会膨胀：

- 所有的业务逻辑都堆在 Service 里，Service 变成了"万能方法集合"；
- 业务对象（Order、Coupon、User）退化为纯数据容器，只有 getter/setter，没有行为；
- 业务规则散落在各个 Service 的各个方法里，没有一个集中的地方可以看到"订单可以做什么、不能做什么"；
- Service 之间互相调用，形成网状依赖。

这就是 DDD 社区所说的"贫血模型"问题：对象只有数据没有行为，行为全部外挂在 Service 上。

### 2.2 DDD 想解决的核心问题

DDD（领域驱动设计）的核心诉求是：**让代码结构反映业务结构**。

具体来说：
- 业务中的核心概念应该在代码中有对应的领域对象，而不只是数据库表的映射；
- 业务规则应该内聚在领域对象里，而不是散落在 Service 层；
- 不同的业务子域应该有清晰的边界，而不是混在一个大的 Service 层里；
- 业务语言（产品和工程师用的术语）和代码语言应该尽可能一致。

### 2.3 但 DDD 的落地门槛确实不低

DDD 的理论体系来自 Eric Evans 2003 年的著作，后来又经过了社区大量的扩展和诠释。它引入了一系列概念：领域、子域、限界上下文、实体、值对象、聚合、聚合根、领域服务、领域事件、仓储、应用服务……

这些概念在理论上是自洽的，但对一个没有 DDD 经验的团队来说，消化这些概念并正确地应用到项目中，需要时间和实践。

更关键的是：DDD 不是一个框架，不是引入一个库就能用的。它是一种建模方法和代码组织方式，需要团队达成共识并持续执行。

## 3. 核心概念解释

我尝试用工程师能直接理解的语言来解释 DDD 中最重要的几个概念。

### 3.1 限界上下文（Bounded Context）

限界上下文是 DDD 中最重要的概念，也是最容易被忽略的。

用一个简单的例子解释：在电商系统中，"商品"这个词在不同的场景下含义不同。

- 在商品管理中，"商品"关注的是标题、描述、图片、类目、属性；
- 在库存管理中，"商品"关注的是 SKU、仓库、数量、锁定量；
- 在订单中，"商品"关注的是下单时的快照：名称、价格、规格；
- 在搜索中，"商品"关注的是关键词、标签、排序权重。

如果这四个场景共用一个 Product 对象，这个对象会越来越臃肿，既有商品管理的字段、又有库存的字段、又有搜索的字段。更糟糕的是，修改商品管理的逻辑可能不小心影响到订单或搜索。

限界上下文的核心思想是：**同一个业务名词在不同的上下文中可以有不同的模型**。商品管理有自己的 Product 模型，库存有自己的 StockItem 模型，订单有自己的 OrderItem 模型。它们之间通过明确的接口或事件通信，而不是共享同一个对象。

对于工程师来说，限界上下文可以理解为：**一个独立的、内聚的业务模块，有自己的模型、规则和存储，和其他模块通过接口交互**。它可以是一个独立服务，也可以是单体应用中的一个模块/包。

### 3.2 实体（Entity）和值对象（Value Object）

**实体**是有唯一标识的业务对象。两个实体即使所有属性都一样，只要 ID 不同就是不同的实体。比如两个订单，即使金额、商品、用户都一样，但 order_id 不同就是两个订单。

**值对象**是没有唯一标识的对象，靠值来区分。比如"金额"——100 元和 100 元是同一个值，不需要 ID。再比如"地址"——如果两个地址的省市区街道都一样，就认为是同一个地址。

区分实体和值对象的实际意义在于：
- 实体需要持久化和跟踪生命周期（创建、修改、删除）；
- 值对象可以随意创建和丢弃，不需要跟踪变化。

在代码中，值对象通常设计为不可变的（immutable）。

### 3.3 聚合（Aggregate）和聚合根（Aggregate Root）

聚合是一组紧密关联的实体和值对象的集合，有一个统一的入口——聚合根。

以订单为例：一个订单（Order）包含多个订单项（OrderItem），可能关联一个收货地址（Address）。Order 是聚合根，OrderItem 和 Address 是聚合内部的元素。

聚合的规则是：
- 外部只能通过聚合根来操作聚合内部的对象。不能直接修改 OrderItem，只能通过 Order 来修改；
- 聚合内部的一致性由聚合根负责保证。比如修改订单项后需要重算总价，这个逻辑在 Order 里完成；
- 聚合之间通过 ID 引用，而不是直接持有对象引用。

聚合的实际意义是划定了一个**事务边界**。一个聚合内部的变更在一个事务中完成，聚合之间的协调通过异步事件或应用层协调。

### 3.4 领域服务（Domain Service）

有些业务逻辑不自然地属于任何一个实体。比如"转账"操作涉及两个账户，放在哪个账户的方法里都不合适。这种逻辑可以放在领域服务中。

领域服务和应用服务（Application Service）的区别是：
- 领域服务封装的是**业务规则**（比如转账时的限额检查、手续费计算）；
- 应用服务负责的是**流程编排**（比如接收请求、调用领域对象、发送事件、返回结果），它本身不包含业务规则。

### 3.5 领域事件（Domain Event）

当领域内发生了一件有业务意义的事情时，可以发出一个领域事件。比如"订单已支付""库存已锁定""优惠券已核销"。

领域事件的作用是让不同的限界上下文之间解耦。订单支付成功后，订单上下文发出"OrderPaid"事件，库存上下文监听这个事件来扣减库存，积分上下文监听这个事件来发放积分。订单上下文不需要知道有谁在监听。

## 4. 常见误区

### 4.1 把 DDD 当银弹

有些团队决定"全面 DDD"，所有模块都按 DDD 的方式重写。结果简单的 CRUD 模块（比如系统配置、字典管理）也要经过 Application Service → Domain Service → Aggregate → Repository 四层，代码量膨胀了两三倍，开发效率明显下降。

DDD 适用于**业务逻辑复杂的核心模块**。对于简单的 CRUD 模块，传统的 Service + DAO 模式更高效。在同一个项目中混用两种模式是正常的。

### 4.2 把限界上下文等同于微服务

限界上下文是逻辑边界，微服务是部署边界。一个限界上下文可以部署为一个微服务，也可以作为单体应用中的一个模块。

把每个限界上下文都拆成独立服务，可能会过度增加运维复杂度。合理的做法是先在代码层面划清边界（包、模块），等到确实需要独立部署和扩展时再拆服务。

### 4.3 为了 DDD 而 DDD

有些团队在代码中创建了大量的 Entity、ValueObject、DomainService、Repository 类，但这些类只是换了个名字的 POJO 和 DAO，内部还是贫血模型的写法。

DDD 的价值不在于类的命名方式，而在于**业务规则是否真的内聚到了领域对象中**。如果 Order 对象只有 getter/setter，所有规则还是写在 OrderService 里，那只是把文件换了个目录，问题并没有解决。

### 4.4 过度关注战术设计，忽视战略设计

DDD 的概念分为两层：
- **战略设计**：限界上下文、上下文映射、统一语言——这些决定了系统的大结构；
- **战术设计**：实体、值对象、聚合、领域服务——这些决定了代码层面的组织方式。

很多团队直接跳到战术设计，讨论"这个对象是实体还是值对象""聚合根的边界怎么划"，但忽略了更重要的战略设计：整个系统有哪些限界上下文？它们之间的关系是什么？团队怎么和上下文对应？

战略设计做对了，战术设计可以逐步调整。战略设计做错了，战术设计再好也弥补不了。

### 4.5 团队对 DDD 的理解不一致

DDD 的概念有很多细微之处，不同团队成员的理解可能差异很大。比如"聚合的边界应该多大"这个问题，激进派倾向于大聚合（一个订单聚合包含订单、订单项、支付信息、物流信息），保守派倾向于小聚合（订单和支付是不同的聚合）。

如果团队内部对这些概念没有达成共识就开始编码，代码库会出现两种甚至三种风格。

## 5. 设计方法

### 5.1 判断是否需要 DDD

不是所有系统都需要 DDD。以下是判断标准：

**适合用 DDD 的场景：**
- 业务规则复杂，有大量的条件判断和状态流转；
- Service 层代码膨胀严重，核心 Service 超过 2000 行；
- 业务概念的语义在代码中表达不清楚，需要靠注释解释；
- 多个模块之间边界模糊，经常互相影响。

**不需要 DDD 的场景：**
- 简单的 CRUD 系统（管理后台、配置系统）；
- 团队规模小（3 人以下），沟通成本低，Service 层还没膨胀；
- 业务逻辑简单明确，不需要复杂的建模。

### 5.2 渐进式落地 DDD

不建议一次性全面重构为 DDD。推荐渐进式落地：

**第一步：识别核心域**

哪些模块的业务逻辑最复杂、变化最频繁、对业务价值最高？这些模块是 DDD 的优先应用对象。

比如在电商系统中，订单和支付是核心域，而系统配置、用户管理是支撑域，报表和日志是通用域。核心域值得投入精力做 DDD，支撑域和通用域用简单方式就好。

**第二步：划限界上下文**

在核心域内，识别不同的限界上下文。问自己：
- 哪些概念总是一起变化？它们可能属于同一个上下文；
- 哪些概念虽然名字相似但含义不同？它们可能属于不同的上下文。

**第三步：把业务规则从 Service 迁移到领域对象**

这是最核心的一步。找到 Service 中和某个实体强相关的业务规则，把它们移到实体中。

比如 OrderService 中有这段代码：

```java
// Before: 业务规则在 Service 中
public void cancelOrder(Long orderId) {
    Order order = orderRepository.findById(orderId);
    if (order.getStatus() != OrderStatus.PENDING_PAYMENT) {
        throw new BusinessException("只有待支付订单可以取消");
    }
    order.setStatus(OrderStatus.CANCELLED);
    order.setCancelTime(LocalDateTime.now());
    orderRepository.save(order);
}
```

迁移后：

```java
// After: 业务规则在领域对象中
// Order.java
public void cancel() {
    if (this.status != OrderStatus.PENDING_PAYMENT) {
        throw new BusinessException("只有待支付订单可以取消");
    }
    this.status = OrderStatus.CANCELLED;
    this.cancelTime = LocalDateTime.now();
}

// OrderApplicationService.java
public void cancelOrder(Long orderId) {
    Order order = orderRepository.findById(orderId);
    order.cancel();
    orderRepository.save(order);
}
```

Application Service 只负责流程编排（查数据、调领域方法、保存），业务规则（什么状态下可以取消）内聚在 Order 对象中。

**第四步：引入领域事件解耦**

当订单取消后需要释放库存、退还优惠券时，不在 Application Service 里直接调用 InventoryService 和 CouponService，而是发出"OrderCancelled"事件，让其他上下文自行处理。

### 5.3 代码结构组织

DDD 项目的包结构通常是按限界上下文组织，而不是按技术层组织：

```
// 传统的按技术层组织
com.example.controller/
com.example.service/
com.example.dao/
com.example.entity/

// 按限界上下文组织
com.example.order/
    application/    -- 应用服务（流程编排）
    domain/         -- 领域对象、领域服务、领域事件
    infrastructure/ -- 仓储实现、外部接口调用
    interfaces/     -- 对外接口（Controller、消息消费者）
com.example.payment/
    application/
    domain/
    infrastructure/
    interfaces/
```

## 6. 业务案例

### 6.1 案例：订单域的 DDD 建模

以电商订单为例，展示如何用 DDD 的方式建模。

**识别聚合：**

订单聚合（Order Aggregate）：
- 聚合根：Order
- 内部实体：OrderItem（订单项）
- 值对象：Money（金额）、Address（收货地址）、OrderSnapshot（商品快照）

**订单聚合的边界决策：**

支付信息（PaymentInfo）是否属于订单聚合？

分析：支付有自己独立的生命周期（待支付→支付中→支付成功→退款）。支付的变化频率和订单不同。一个订单可能关联多笔支付（分期、换渠道重试）。

结论：支付不属于订单聚合，是独立的聚合。订单和支付通过 order_id 关联。

**领域对象的行为示例：**

```java
public class Order {
    private OrderId id;
    private UserId userId;
    private List<OrderItem> items;
    private Money totalAmount;
    private Money discountAmount;
    private Money payableAmount;
    private OrderStatus status;
    private Address shippingAddress;
    
    // 添加商品
    public void addItem(ProductSnapshot product, int quantity) {
        OrderItem item = new OrderItem(product, quantity);
        this.items.add(item);
        recalculateAmount();
    }
    
    // 应用优惠
    public void applyDiscount(Money discount) {
        if (discount.greaterThan(this.totalAmount)) {
            throw new BusinessException("优惠金额不能超过订单总额");
        }
        this.discountAmount = discount;
        this.payableAmount = this.totalAmount.subtract(discount);
    }
    
    // 确认订单
    public OrderConfirmedEvent confirm() {
        if (this.items.isEmpty()) {
            throw new BusinessException("订单没有商品");
        }
        if (this.shippingAddress == null) {
            throw new BusinessException("收货地址不能为空");
        }
        this.status = OrderStatus.CONFIRMED;
        return new OrderConfirmedEvent(this.id, this.payableAmount);
    }
    
    // 内部方法
    private void recalculateAmount() {
        this.totalAmount = items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
        this.payableAmount = this.totalAmount.subtract(
            this.discountAmount != null ? this.discountAmount : Money.ZERO);
    }
}
```

这个 Order 对象不只是数据容器，它包含了核心的业务规则：添加商品后自动重算金额、优惠不能超过总额、确认前必须有商品和地址。

### 6.2 什么时候不用 DDD

同一个项目中的用户管理模块——功能是用户注册、登录、修改个人信息。业务逻辑简单，没有复杂的状态流转和规则。这种模块用传统的 Service + DAO 模式就够了，没有必要套 DDD 的结构。

判断标准很简单：**如果一个模块的 Service 方法大多数是"查出来、改一下、存回去"的简单流程，不需要 DDD。如果 Service 方法里有大量的业务判断、状态校验、规则计算，值得考虑 DDD。**

## 7. 架构判断清单

- 如果一个 Service 类超过 2000 行且还在持续增长，说明需要考虑用 DDD 来拆分和组织业务逻辑。
- 如果产品描述的业务概念在代码中找不到对应的对象（只有数据库表的映射），说明缺少领域建模。
- 如果修改一个业务规则需要在三个以上的 Service 里同步修改，说明业务规则没有内聚到领域对象中。
- 如果"订单可以做什么"这个问题无法通过看 Order 类来回答，说明业务行为散落在外部了。
- 如果团队要全面推 DDD，但没有人有 DDD 的实践经验，建议先在一个核心模块上试点，而不是全面铺开。
- 如果简单的 CRUD 模块也被要求用 DDD 的方式编写，说明 DDD 被过度应用了。
- 如果团队对"聚合的边界怎么划"没有达成共识就开始编码，大概率会出现风格不一致的问题。

## 8. 本章小结

DDD 解决的是一个真实的工程问题：当业务逻辑变得足够复杂时，传统的 Service 层写法会导致代码膨胀、业务规则散落、模块边界模糊。DDD 通过限界上下文划清边界、通过领域对象内聚业务规则、通过领域事件解耦模块间的协作，提供了一种更清晰的代码组织方式。

但 DDD 不是银弹。它有学习成本，有落地门槛，有过度使用的风险。它适合业务逻辑复杂的核心模块，不适合简单的 CRUD 场景。在同一个项目中，核心域用 DDD、其他模块用传统方式，是完全合理的做法。

对工程师来说，DDD 最有价值的收获可能不是那套概念和分层，而是一种思维方式的转变：**在写代码之前先理解业务、先建模、先划边界。** 这个思维方式即使不用 DDD 的全套术语，也能帮助写出更清晰的代码。
