# 第 6 章：分层能力：为什么复杂系统一定需要层次？

## 1. 从一个真实问题开始

一个做了两年的订单系统，核心代码在 OrderService 里。打开这个文件，三千多行。随便找一个方法 createOrder()，从头看到尾：

```java
public OrderDTO createOrder(CreateOrderRequest request) {
    // 1. 参数校验（30行）
    if (request.getItems() == null || request.getItems().isEmpty()) { ... }
    if (request.getAddressId() == null) { ... }
    // ... 各种校验
    
    // 2. 查数据库拿商品信息（20行）
    List<Product> products = productMapper.selectByIds(itemIds);
    
    // 3. 算价格（50行）
    BigDecimal totalAmount = BigDecimal.ZERO;
    for (OrderItemRequest item : request.getItems()) {
        Product product = productMap.get(item.getProductId());
        // 阶梯价、会员价、限时折扣...各种价格逻辑
    }
    
    // 4. 查优惠券、算优惠（40行）
    if (request.getCouponId() != null) {
        Coupon coupon = couponMapper.selectById(request.getCouponId());
        // 优惠券使用条件校验、优惠金额计算...
    }
    
    // 5. 锁库存（15行）
    for (OrderItemRequest item : request.getItems()) {
        int affected = inventoryMapper.lockStock(item.getSkuId(), item.getQuantity());
        if (affected == 0) { throw new BusinessException("库存不足"); }
    }
    
    // 6. 冻结优惠券（10行）
    if (request.getCouponId() != null) {
        couponMapper.updateStatus(request.getCouponId(), "FROZEN");
    }
    
    // 7. 创建订单记录（30行）
    Order order = new Order();
    order.setOrderNo(generateOrderNo());
    order.setUserId(request.getUserId());
    // ...设置各种字段
    orderMapper.insert(order);
    
    // 8. 创建订单项记录（20行）
    for (...) { orderItemMapper.insert(orderItem); }
    
    // 9. 创建支付单（15行）
    Payment payment = new Payment();
    // ...
    paymentMapper.insert(payment);
    
    // 10. 发消息通知（10行）
    kafkaTemplate.send("order-created", orderEvent);
    
    // 11. 组装返回值（20行）
    return convertToDTO(order);
}
```

这个方法大概两百多行。参数校验、数据库查询、业务计算、数据持久化、外部调用、消息发送、DTO 转换——全部混在一起。

问题出现了：

产品要改价格计算逻辑（加一种新的折扣方式），工程师在这两百多行里找到价格计算的部分，改了。但不小心影响了后面库存锁定的变量，导致锁库存的数量算错了。

另一个工程师要加一个"下单前检查用户风控状态"的需求。他在方法开头加了一段调用风控服务的代码。但风控服务偶尔超时，导致整个下单流程变慢。

第三个工程师想写单元测试。发现测不了——这个方法直接调用了七八个 Mapper、两个外部服务、一个 Kafka。要 mock 所有依赖才能跑起来，mock 的代码比测试逻辑还长。

这些问题的根源都是同一个：**不同关注点的代码混在了一起，没有分层。**

## 2. 问题为什么会变复杂

### 2.1 "全写在 Service 里"是怎么发生的

项目初期，Controller 接收请求，Service 处理逻辑，DAO 访问数据库。三层架构，清晰明了。

但随着业务变复杂，Service 层成了"万能容器"。因为大家默认"业务逻辑写在 Service 里"，而业务逻辑的范围越来越广：参数校验是业务逻辑、价格计算是业务逻辑、调外部服务也是业务逻辑、发消息也是业务逻辑。

结果是 Service 层承载了所有不属于 Controller 和 DAO 的东西。Service 方法变成了从头到尾的过程式脚本，一个方法几百行，什么都做。

### 2.2 缺乏分层导致的三个典型症状

**改动风险高**：不同关注点的代码交织在一起，改一处可能不小心影响另一处。价格计算和库存锁定写在同一个方法里，改价格逻辑时可能引入库存 bug。

**复用困难**：价格计算逻辑内嵌在 createOrder 方法里，另一个场景（比如购物车预估价格）需要同样的价格计算逻辑，但没法复用——只能复制一份出来。

**测试困难**：想测价格计算逻辑是否正确，但没法单独测——它嵌在一个依赖七八个外部组件的大方法里。

### 2.3 分层的本质

分层的本质是**关注点分离**。不同的关注点放在不同的层次里，每一层只处理自己那个层次的问题。

Controller 层关注的是：HTTP 协议、参数解析、返回格式。
应用层关注的是：流程编排——按什么顺序调用哪些能力。
领域层关注的是：业务规则——价格怎么算、优惠券能不能用、订单能不能取消。
基础设施层关注的是：技术实现——数据库怎么读写、消息怎么发送、外部接口怎么调用。

每一层只依赖下一层，不跨层调用。修改某一层的内部实现时，只要接口不变，其他层不受影响。

## 3. 核心概念解释

### 3.1 经典三层架构：简单但不够

大多数项目的起点是 Controller → Service → DAO（或 Repository）三层架构。

这个分层在业务简单时够用。但当业务复杂度上升时，Service 层会承担过多职责：

- 流程编排（先查数据、再算价格、再锁库存、再创建订单）
- 业务规则（价格怎么算、优惠券能不能用）
- 数据转换（DTO 和数据库对象之间的转换）
- 外部调用协调（调用支付服务、库存服务）

这四种职责混在一起，就是 Service 层膨胀的原因。

### 3.2 四层架构：拆分 Service 层

解决 Service 层膨胀的办法是把它拆成两层：**应用层**和**领域层**。

**接口层（Interfaces）**：对外暴露的入口。包括 HTTP Controller、RPC 接口、消息消费者。负责协议适配、参数校验、返回值组装。

**应用层（Application）**：流程编排。接收请求后，按顺序调用领域对象和基础设施来完成一个完整的业务操作。应用层不包含业务规则，只包含"先做什么、后做什么"的流程。

**领域层（Domain）**：核心业务规则。价格计算、优惠校验、订单状态流转、库存扣减规则——这些"纯业务逻辑"住在领域层。领域层不依赖任何框架和基础设施（不直接调数据库、不直接发消息）。

**基础设施层（Infrastructure）**：技术实现细节。数据库访问、缓存操作、消息发送、外部 HTTP 调用。

用前面的 createOrder 例子，重新分层后：

```java
// 接口层：接收HTTP请求，参数校验
@PostMapping("/orders")
public Result<OrderVO> createOrder(@RequestBody CreateOrderRequest request) {
    // 基本参数格式校验
    validate(request);
    OrderDTO result = orderApplicationService.createOrder(request);
    return Result.success(convertToVO(result));
}

// 应用层：流程编排
public class OrderApplicationService {
    public OrderDTO createOrder(CreateOrderRequest request) {
        // 1. 构建领域对象
        List<Product> products = productRepository.findByIds(request.getItemIds());
        Order order = Order.create(request.getUserId(), products, request.getItems());
        
        // 2. 应用优惠（领域逻辑在Order内部）
        if (request.getCouponId() != null) {
            Coupon coupon = couponRepository.findById(request.getCouponId());
            order.applyCoupon(coupon);
        }
        
        // 3. 锁库存
        inventoryService.lockStock(order.getItems());
        
        // 4. 持久化
        orderRepository.save(order);
        
        // 5. 发事件
        eventPublisher.publish(new OrderCreatedEvent(order));
        
        return OrderDTO.from(order);
    }
}

// 领域层：业务规则
public class Order {
    public static Order create(UserId userId, List<Product> products, 
                                List<ItemRequest> items) {
        Order order = new Order();
        order.userId = userId;
        for (ItemRequest item : items) {
            Product product = findProduct(products, item.getProductId());
            order.addItem(product, item.getQuantity());
        }
        order.calculateAmount();  // 价格计算逻辑在这里
        return order;
    }
    
    public void applyCoupon(Coupon coupon) {
        coupon.validateUsable(this);  // 优惠券使用校验在Coupon领域对象里
        this.couponId = coupon.getId();
        this.discountAmount = coupon.calculateDiscount(this.totalAmount);
        this.payableAmount = this.totalAmount.subtract(this.discountAmount);
    }
    
    private void calculateAmount() {
        this.totalAmount = this.items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
        this.payableAmount = this.totalAmount;
    }
}
```

对比之前的写法：
- 价格计算逻辑集中在 Order 领域对象里，和数据库操作分开了。要改价格逻辑只需要改 Order 类。
- 应用层只做流程编排，可以一眼看出下单流程有几个步骤。
- 想测价格计算逻辑，直接测 Order 类，不需要 mock 数据库和消息队列。

### 3.3 层与层之间的依赖规则

分层架构最关键的规则是：**依赖方向是单向的，从上到下。**

接口层 → 应用层 → 领域层 → 基础设施层

但这里有一个问题：领域层不应该依赖基础设施层（领域对象不应该直接调用数据库）。但领域层定义的 Repository 接口需要基础设施层来实现。

解决方法是**依赖倒置**：领域层定义 Repository 接口（比如 OrderRepository），基础设施层提供实现（比如 OrderRepositoryImpl 使用 MyBatis）。领域层依赖的是自己定义的接口，不依赖具体的技术实现。

```
领域层定义：  interface OrderRepository { void save(Order order); }
基础设施层实现：class OrderRepositoryImpl implements OrderRepository { ... }
```

这样领域层是"纯"的——不依赖任何框架和技术细节，只包含业务逻辑。

### 3.4 不是所有系统都需要四层

四层架构适合业务逻辑复杂的系统。对于简单的 CRUD 系统（管理后台、配置中心），三层架构足够用。强行分四层只会增加无谓的间接层。

判断标准：如果一个模块的 Service 方法大多数是"查出来、改一下、存回去"的简单流程，三层就够了。如果 Service 方法里有大量的业务判断、规则计算、状态校验，说明需要抽出领域层。

在同一个项目中，核心业务模块用四层，简单模块用三层，这是完全合理的。

## 4. 常见误区

### 4.1 "分层就是分包"

有些团队把代码按 controller/service/dao 分了三个包就觉得分好层了。但如果 Service 里直接 new 了 DAO 的实现类，或者 Controller 里直接调用了 DAO，分包就是摆设。

分层的关键不是包结构，而是**依赖方向**和**职责边界**。每一层只依赖自己下面的层，不跨层调用，不反向依赖。

### 4.2 "分层太多影响性能"

有人担心分层会影响性能——多了几层方法调用的开销。

在 Java、Go 这类编译型语言里，一层方法调用的开销是纳秒级的。一个 HTTP 请求的延迟是毫秒级甚至秒级。多几层方法调用对性能的影响可以忽略不计。

分层影响的不是性能，而是代码的组织清晰度和可维护性。这个收益远大于那点微乎其微的性能开销。

### 4.3 "每个操作都要经过所有层"

有些团队执行分层规范太僵化：Controller 必须调 Application Service，Application Service 必须调 Domain Service，Domain Service 必须调 Repository。哪怕一个简单的"按 ID 查订单"也要过四层。

如果某一层在某个操作中没有实质性的工作，可以跳过它。查询操作可以从应用层直接调 Repository，不需要经过领域层。分层是为了组织复杂性，不是为了增加仪式感。

### 4.4 领域层依赖了基础设施

最常见的分层污染是领域对象里出现了数据库操作：

```java
// 反模式：领域对象里直接用了Mapper
public class Order {
    @Autowired
    private OrderMapper orderMapper;
    
    public void cancel() {
        this.status = CANCELLED;
        orderMapper.updateStatus(this.id, CANCELLED);  // 不应该在领域对象里操作数据库
    }
}
```

领域对象应该只包含业务规则，数据持久化由应用层或基础设施层负责。

### 4.5 应用层变成了另一个"胖 Service"

有些团队引入了应用层，但应用层的方法里还是有大量的业务判断逻辑。应用层没有真正做到"只编排不判断"，变成了改了个名字的 Service。

检验方法：看应用层的代码是否可以被一个不懂业务的人读懂。如果他能理解"先做A再做B最后做C"的流程，但具体的业务规则在领域对象里，说明分层是对的。如果他必须理解复杂的业务条件才能读懂应用层代码，说明业务规则泄漏到了应用层。

## 5. 设计方法

### 5.1 从现有代码逐步分层

不需要一次性重构。可以逐步把业务逻辑从 Service 中提取到领域对象中。

**第一步：识别"纯业务逻辑"。** 在一个大的 Service 方法中，找到不依赖数据库、不依赖外部服务的业务判断和计算逻辑。这些就是可以迁移到领域层的候选。

**第二步：把这些逻辑迁移到领域对象的方法中。** 比如把价格计算逻辑从 OrderService 迁移到 Order.calculateAmount()。

**第三步：Service 改为调用领域对象的方法。** Service 变成了"查出领域对象 → 调用领域方法 → 保存领域对象"的编排角色。

**第四步：随着迁移的推进，把 Service 重命名为 ApplicationService，明确它的"编排"职责。**

### 5.2 分层的检验标准

一个好的分层应该满足：

1. **领域层可以独立测试**：不需要启动 Spring 容器、不需要连数据库，直接 new 领域对象就能测业务逻辑。
2. **换一个数据库不需要改领域层**：从 MySQL 换到 PostgreSQL，只需要改基础设施层的 Repository 实现。
3. **换一个接口协议不需要改应用层**：从 HTTP 换到 RPC，只需要改接口层。
4. **应用层的方法可以被不懂业务的人读懂流程**：方法里的每一步都是"做什么"，而不是"怎么做"。

## 6. 业务案例

### 6.1 案例：订单创建流程的分层重构

**重构前（一个方法搞定一切）**：

OrderService.createOrder() 包含：参数校验、查商品、算价格、校验优惠券、锁库存、创建订单记录、创建支付单、发消息。约 250 行。

问题：改价格逻辑时不小心引入了库存 bug。写单测要 mock 8 个依赖。新人两天看不懂这个方法。

**重构后（按关注点分层）**：

接口层：参数格式校验、DTO 转换。约 20 行。
应用层：流程编排——构建订单 → 应用优惠 → 锁库存 → 保存 → 发事件。约 30 行。
领域层：Order.create() 包含价格计算，Order.applyCoupon() 包含优惠校验和计算，Coupon.validateUsable() 包含使用条件检查。每个方法 20-40 行。
基础设施层：OrderRepositoryImpl、InventoryClient、EventPublisher。各自 20-30 行。

总代码量增加了约 30%（因为多了类文件和接口定义），但每个文件都短且职责单一。

收益：
- 改价格逻辑只需要改 Order 类，不影响库存逻辑。
- 测价格计算只需要 new Order() 再调方法，不需要 mock 任何东西。
- 新人先看应用层理解流程（五步），再看领域层理解规则，分步消化。

## 7. 架构判断清单

- 如果一个 Service 方法超过 100 行，说明它可能混合了多个关注点，需要考虑分层。
- 如果测试一个业务规则需要 mock 数据库和外部服务，说明业务逻辑没有从基础设施中分离出来。
- 如果领域对象（如 Order）只有 getter/setter 没有业务方法，说明业务逻辑外泄到了 Service 层。
- 如果应用层的代码里有 if-else 业务判断（而不只是流程编排），说明业务逻辑泄漏到了应用层。
- 如果改了数据库框架（比如从 MyBatis 换到 JPA）需要改业务逻辑代码，说明分层不够干净。
- 如果同一个业务规则在两个 Service 方法里重复出现，说明需要把这个规则收到领域对象中。

## 8. 本章小结

分层的目的不是把代码分散到更多的文件里，而是让不同关注点的代码住在它该住的地方。业务规则住在领域层，流程编排住在应用层，技术实现住在基础设施层。每一层只管自己那一摊事。

分层的收益是：改动影响可控（改价格逻辑不会影响库存）、代码可测试（业务逻辑可以脱离框架独立测试）、代码可理解（看应用层知道流程，看领域层知道规则）。

不是所有系统都需要严格的四层分层。简单的 CRUD 模块用三层就够了。但对于业务逻辑复杂、Service 层已经膨胀的模块，把 Service 拆分为应用层和领域层，是降低复杂度最有效的手段之一。分层不需要一步到位——先从最痛的那个 Service 开始，把里面的业务规则提到领域对象中，逐步推进。
