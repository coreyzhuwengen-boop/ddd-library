# DDD 架构学习指南

## 🎯 学习目标

本指南帮助你从传统的 Spring Web CRUD 项目思维转换到 DDD（领域驱动设计）架构思维，通过这个图书馆项目理解 DDD 的核心概念和实践方法。

---

## 📊 传统 CRUD vs DDD 架构对比

### 传统 CRUD 项目结构

```
controller/
  └── UserController.java        # 处理HTTP请求
service/
  └── UserService.java           # 业务逻辑
repository/
  └── UserRepository.java         # 数据访问
entity/
  └── User.java                   # 数据库实体（JPA）
```

**特点**：
- 按技术层次组织（Controller → Service → Repository）
- 实体类直接对应数据库表
- 业务逻辑分散在 Service 层
- 容易变成"贫血模型"（Anemic Domain Model）

### DDD 项目结构

```
lending/                          # 有界上下文（Bounded Context）
  ├── patron/                     # 读者聚合
  │   ├── model/                  # 领域模型（核心业务逻辑）
  │   │   └── Patron.java         # 聚合根
  │   ├── application/            # 应用服务（编排）
  │   │   └── hold/
  │   │       └── PlacingOnHold.java
  │   └── infrastructure/        # 基础设施（数据库、框架）
  │       └── PatronsDatabaseRepository.java
  └── book/                       # 图书聚合
```

**特点**：
- 按业务领域组织（按聚合/上下文）
- 领域模型包含业务逻辑（富领域模型）
- 基础设施依赖领域，而非相反
- 清晰的边界和职责划分

---

## 🗺️ 学习路径（7个阶段）

### 阶段 1：理解项目整体结构（30分钟）

**目标**：了解项目的宏观架构，不要深入代码细节

#### 1.1 阅读项目文档
- ✅ 阅读 `README.md`（英文原版，了解项目背景）
- ✅ 阅读 `项目说明.md`（中文说明，理解整体架构）
- ✅ 浏览 `docs/` 目录下的设计文档

#### 1.2 理解包结构
打开 IDE，查看 `src/main/java/io/pillopl/library/` 目录：

```
library/
├── catalogue/          # 图书目录上下文（简单CRUD）
├── commons/            # 公共组件
└── lending/           # 借阅上下文（复杂业务逻辑）
    ├── book/          # 图书聚合
    ├── patron/        # 读者聚合 ⭐ 重点学习
    ├── dailysheet/    # 每日工作表（读模型）
    └── patronprofile/ # 读者档案（读模型）
```

**关键理解**：
- `catalogue` 是简单的 CRUD，没有复杂业务逻辑
- `lending` 是核心业务领域，采用六边形架构
- 每个聚合都有自己的 `model`、`application`、`infrastructure` 层

#### 1.3 查看应用入口
**文件**：`src/main/java/io/pillopl/library/LibraryApplication.java`

```java
new SpringApplicationBuilder()
    .parent(LibraryApplication.class)
    .child(LendingConfig.class).web(WebApplicationType.SERVLET)
    .sibling(CatalogueConfiguration.class).web(WebApplicationType.NONE)
    .run(args);
```

**理解点**：
- 每个有界上下文有独立的 Spring 应用上下文
- 这是模块化单体的体现，未来可以拆分为微服务

---

### 阶段 2：理解最简单的聚合 - Book（1小时）

**目标**：通过 Book 聚合理解 DDD 的核心概念：聚合、实体、值对象、状态模型

#### 2.1 理解 Book 的状态模型（类型系统）

**文件**：`src/main/java/io/pillopl/library/lending/book/model/`

阅读顺序：
1. `Book.java` - 接口定义
2. `AvailableBook.java` - 可借阅状态
3. `BookOnHold.java` - 已预约状态
4. `CheckedOutBook.java` - 已借出状态

**关键理解**：
```java
// ❌ 传统方式：用枚举表示状态
public class Book {
    private BookStatus status; // AVAILABLE, ON_HOLD, CHECKED_OUT
    public void placeOnHold() {
        if (status != AVAILABLE) {
            throw new IllegalStateException();
        }
        // ...
    }
}

// ✅ DDD方式：用类型表示状态
public interface Book { }
public class AvailableBook implements Book {
    public BookOnHold placeOnHold(...) { ... }  // 只能对AvailableBook调用
}
public class BookOnHold implements Book {
    public AvailableBook cancelHold() { ... }     // 只能对BookOnHold调用
}
```

**优势**：
- 编译期保证：不能对 `BookOnHold` 调用 `placeOnHold`
- 类型即文档：方法签名就说明了业务规则
- 无需运行时状态检查

#### 2.2 理解 Book 的仓储实现

**文件**：`src/main/java/io/pillopl/library/lending/book/infrastructure/BookDatabaseRepository.java`

**关键点**：
- 仓储接口在 `model` 层：`BookRepository`
- 仓储实现在 `infrastructure` 层：`BookDatabaseRepository`
- 领域模型不依赖基础设施

#### 2.3 理解 Book 如何响应事件

**文件**：`src/main/java/io/pillopl/library/lending/book/application/PatronEventsHandler.java`

**理解点**：
- Book 聚合通过监听 `PatronEvent` 来更新自己的状态
- 这是聚合间通信的方式（事件驱动）

---

### 阶段 3：深入理解核心聚合 - Patron（2-3小时）⭐

**目标**：理解富领域模型、业务规则封装、策略模式

#### 3.1 理解 Patron 聚合根

**文件**：`src/main/java/io/pillopl/library/lending/patron/model/Patron.java`

**阅读重点**：

```java
public class Patron {
    // 1. 聚合包含业务概念，不是简单的数据容器
    private final PatronInformation patron;
    private final List<PlacingOnHoldPolicy> placingOnHoldPolicies;  // 策略列表
    private final OverdueCheckouts overdueCheckouts;
    private final PatronHolds patronHolds;
    
    // 2. 业务方法返回 Either（函数式编程）
    public Either<BookHoldFailed, BookPlacedOnHoldEvents> placeOnHold(AvailableBook book) {
        // 3. 业务规则检查
        Option<Rejection> rejection = patronCanHold(aBook, duration);
        if (rejection.isEmpty()) {
            // 4. 返回领域事件，而不是直接修改状态
            return announceSuccess(events(bookPlacedOnHold));
        }
        return announceFailure(...);
    }
}
```

**对比传统方式**：
```java
// ❌ 传统方式：业务逻辑在Service层
@Service
public class PatronService {
    public void placeOnHold(Long patronId, Long bookId) {
        PatronEntity patron = patronRepo.findById(patronId);
        BookEntity book = bookRepo.findById(bookId);
        
        // 业务规则检查散落在Service中
        if (patron.getHolds().size() >= 5) {
            throw new BusinessException("最多5个预约");
        }
        if (book.isRestricted() && patron.isRegular()) {
            throw new BusinessException("普通读者不能预约限制类图书");
        }
        // ...
    }
}

// ✅ DDD方式：业务逻辑在领域模型中
public class Patron {
    // 业务规则封装在聚合内部
    public Either<BookHoldFailed, BookPlacedOnHoldEvents> placeOnHold(AvailableBook book) {
        // 策略模式：多个业务规则通过策略列表检查
        Option<Rejection> rejection = patronCanHold(aBook, duration);
        // ...
    }
}
```

#### 3.2 理解业务规则策略

**文件**：`src/main/java/io/pillopl/library/lending/patron/model/PlacingOnHoldPolicy.java`

查看策略的实现位置：
- `src/main/java/io/pillopl/library/lending/patron/infrastructure/PatronConfiguration.java`

**关键理解**：
```java
// 策略是函数式接口
@FunctionalInterface
public interface PlacingOnHoldPolicy {
    Either<Rejection, Allowance> apply(AvailableBook toHold, Patron patron, HoldDuration holdDuration);
}

// 具体策略在配置类中定义
PlacingOnHoldPolicy onlyResearcherPatronsCanHoldRestrictedBooksPolicy = 
    (toHold, patron, holdDuration) -> {
        if (toHold.isRestricted() && patron.isRegular()) {
            return left(Rejection.withReason("Regular patrons cannot hold restricted books"));
        }
        return right(new Allowance());
    };
```

**优势**：
- 业务规则可测试、可组合
- 规则清晰可见，易于理解
- 符合开闭原则（新增规则只需添加策略）

#### 3.3 理解值对象（Value Objects）

**文件**：
- `src/main/java/io/pillopl/library/lending/patron/model/PatronId.java`
- `src/main/java/io/pillopl/library/lending/patron/model/HoldDuration.java`
- `src/main/java/io/pillopl/library/catalogue/ISBN.java`

**关键理解**：
- 值对象是不可变的
- 值对象封装了验证逻辑
- 值对象通过 `equals` 比较，而非引用

#### 3.4 理解领域事件

**文件**：`src/main/java/io/pillopl/library/lending/patron/model/PatronEvent.java`

**关键理解**：
- 领域事件表示"已发生的事情"
- 事件是不可变的
- 事件用于聚合间通信

查看事件定义：
```java
@Value
class BookPlacedOnHold implements PatronEvent {
    @NonNull UUID eventId;
    @NonNull BookId bookId;
    @NonNull PatronId patronId;
    @NonNull Instant when;
    // ...
}
```

---

### 阶段 4：理解应用服务层（1小时）

**目标**：理解应用服务如何编排领域对象

#### 4.1 理解应用服务的职责

**文件**：`src/main/java/io/pillopl/library/lending/patron/application/hold/PlacingOnHold.java`

**关键理解**：
```java
public class PlacingOnHold {
    // 1. 应用服务依赖领域接口，不依赖具体实现
    private final FindAvailableBook findAvailableBook;  // 接口
    private final Patrons patronRepository;            // 接口
    
    // 2. 应用服务负责编排，不包含业务逻辑
    public Try<Result> placeOnHold(PlaceOnHoldCommand command) {
        // 3. 查找聚合
        AvailableBook availableBook = find(command.getBookId());
        Patron patron = find(command.getPatronId());
        
        // 4. 调用领域方法（业务逻辑在领域模型中）
        Either<BookHoldFailed, BookPlacedOnHoldEvents> result = 
            patron.placeOnHold(availableBook, command.getHoldDuration());
        
        // 5. 发布事件
        return Match(result).of(
            Case($Left($()), this::publishEvents),
            Case($Right($()), this::publishEvents)
        );
    }
}
```

**对比传统方式**：
```java
// ❌ 传统方式：Service包含业务逻辑
@Service
public class PatronService {
    public void placeOnHold(Long patronId, Long bookId) {
        // 业务逻辑都在这里
        if (patron.getHolds().size() >= 5) { ... }
        if (book.isRestricted() && patron.isRegular()) { ... }
        // ...
    }
}

// ✅ DDD方式：Service只负责编排
public class PlacingOnHold {
    public Try<Result> placeOnHold(PlaceOnHoldCommand command) {
        // 查找聚合
        Patron patron = find(command.getPatronId());
        AvailableBook book = find(command.getBookId());
        
        // 调用领域方法（业务逻辑在Patron中）
        Either<...> result = patron.placeOnHold(book, ...);
        
        // 处理结果
        return publishEvents(result);
    }
}
```

#### 4.2 理解命令对象

**文件**：`src/main/java/io/pillopl/library/lending/patron/application/hold/PlaceOnHoldCommand.java`

**关键理解**：
- 命令是不可变的值对象
- 命令包含执行操作所需的所有信息
- 命令可以验证输入数据

---

### 阶段 5：理解基础设施层（1小时）

**目标**：理解如何将领域模型持久化，如何实现仓储

#### 5.1 理解仓储实现

**文件**：`src/main/java/io/pillopl/library/lending/patron/infrastructure/PatronsDatabaseRepository.java`

**关键理解**：

```java
class PatronsDatabaseRepository implements Patrons {
    // 1. 实现领域接口
    @Override
    public Option<Patron> findBy(PatronId patronId) {
        // 2. 从数据库加载实体
        PatronDatabaseEntity entity = patronEntityRepository.findByPatronId(...);
        // 3. 转换为领域模型
        return Option.of(entity).map(domainModelMapper::map);
    }
    
    // 4. 通过事件保存（事件溯源的思想）
    @Override
    public Patron publish(PatronEvent domainEvent) {
        Patron result = Match(domainEvent).of(
            Case($(instanceOf(PatronCreated.class)), this::createNewPatron),
            Case($(), this::handleNextEvent)
        );
        // 5. 发布事件到事件总线
        domainEvents.publish(domainEvent.normalize());
        return result;
    }
}
```

**关键点**：
- 仓储接口在 `model` 层定义
- 仓储实现在 `infrastructure` 层
- 需要将数据库实体转换为领域模型
- 这个项目使用了事件溯源的思想（通过事件重建状态）

#### 5.2 理解数据库实体

**文件**：`src/main/java/io/pillopl/library/lending/patron/infrastructure/PatronDatabaseEntity.java`

**关键理解**：
- 数据库实体是持久化模型，与领域模型分离
- 使用 Spring Data JDBC（不是 JPA）
- 实体可以处理事件来更新状态

#### 5.3 理解配置类

**文件**：`src/main/java/io/pillopl/library/lending/patron/infrastructure/PatronConfiguration.java`

**关键理解**：
- 配置类组装所有依赖
- 定义业务规则策略
- 创建工厂对象

---

### 阶段 6：理解事件驱动和CQRS（1-2小时）

**目标**：理解聚合间如何通信，读写分离

#### 6.1 理解事件发布

**文件**：
- `src/main/java/io/pillopl/library/commons/events/publisher/JustForwardDomainEventPublisher.java`
- `src/main/java/io/pillopl/library/commons/events/publisher/StoreAndForwardDomainEventPublisher.java`

**关键理解**：
- **立即一致性**：事件发布后立即处理
- **最终一致性**：事件存储后异步处理
- 项目支持两种模式切换

#### 6.2 理解读模型（CQRS）

**文件**：
- `src/main/java/io/pillopl/library/lending/dailysheet/` - 每日工作表
- `src/main/java/io/pillopl/library/lending/patronprofile/` - 读者档案

**关键理解**：
- **写模型**：`Patron`、`Book` 等聚合，处理命令
- **读模型**：`DailySheet`、`PatronProfile`，优化查询
- 读模型通过监听事件更新

查看读模型如何更新：
- `src/main/java/io/pillopl/library/lending/dailysheet/infrastructure/SheetsReadModel.java`

#### 6.3 理解Web层

**文件**：`src/main/java/io/pillopl/library/lending/patronprofile/web/PatronProfileController.java`

**关键理解**：
- Controller 调用应用服务
- Controller 不包含业务逻辑
- 使用 HATEOAS 提供 RESTful API

---

### 阶段 7：理解测试和架构约束（1小时）

**目标**：理解如何测试领域逻辑，如何保证架构不被破坏

#### 7.1 理解领域测试

**文件**：`src/test/groovy/io/pillopl/library/lending/patron/`

**关键理解**：
- 使用 Spock Framework（BDD风格）
- 测试领域逻辑，无需模拟依赖
- 测试代码接近业务语言

示例：
```groovy
def 'should reject hold when patron has maximum number of holds'() {
    given:
        Patron patron = aRegularPatron() withHolds(maxNumberOfHolds())
    and:
        AvailableBook book = aCirculatingBook()
    when:
        Either result = patron.placeOnHold(book)
    then:
        result.isLeft()
        result.getLeft() instanceof BookHoldFailed
}
```

#### 7.2 理解架构测试

**文件**：`src/test/groovy/io/pillopl/library/lending/architecture/`

**关键理解**：
- 使用 ArchUnit 验证架构约束
- 确保领域模型不依赖基础设施
- 防止架构腐化

---

## 🔍 完整业务流程追踪

### 场景：读者预约图书

让我们追踪一个完整的业务流程，理解各层如何协作：

#### 1. 请求入口
**文件**：`PatronProfileController.java` (line 100-114)

```java
@PostMapping("/profiles/{patronId}/holds")
ResponseEntity placeHold(@PathVariable UUID patronId, @RequestBody PlaceHoldRequest request) {
    Try<Result> result = placingOnHold.placeOnHold(
        new PlaceOnHoldCommand(...)
    );
    return result.map(success -> ResponseEntity.ok().build())
                 .getOrElse(ResponseEntity.status(INTERNAL_SERVER_ERROR).build());
}
```

#### 2. 应用服务编排
**文件**：`PlacingOnHold.java`

```java
public Try<Result> placeOnHold(PlaceOnHoldCommand command) {
    // 查找聚合
    AvailableBook availableBook = find(command.getBookId());
    Patron patron = find(command.getPatronId());
    
    // 调用领域方法
    Either<BookHoldFailed, BookPlacedOnHoldEvents> result = 
        patron.placeOnHold(availableBook, command.getHoldDuration());
    
    // 发布事件
    return Match(result).of(...);
}
```

#### 3. 领域逻辑执行
**文件**：`Patron.java` (line 48-58)

```java
public Either<BookHoldFailed, BookPlacedOnHoldEvents> placeOnHold(AvailableBook aBook, HoldDuration duration) {
    // 业务规则检查（策略模式）
    Option<Rejection> rejection = patronCanHold(aBook, duration);
    
    if (rejection.isEmpty()) {
        // 创建事件
        BookPlacedOnHold bookPlacedOnHold = bookPlacedOnHoldNow(...);
        // 返回事件
        return announceSuccess(events(bookPlacedOnHold));
    }
    return announceFailure(...);
}
```

#### 4. 事件发布和处理
- 事件发布到事件总线
- `Book` 聚合监听 `BookPlacedOnHold` 事件，更新状态
- `DailySheet` 读模型监听事件，更新查询视图

---

## 📝 学习检查清单

完成每个阶段后，检查自己是否理解：

### 阶段 1-2：基础理解
- [ ] 理解项目为什么按有界上下文组织，而不是按技术层次
- [ ] 理解 Book 为什么用不同类型表示状态，而不是枚举
- [ ] 理解为什么领域模型不依赖 Spring

### 阶段 3：核心理解
- [ ] 理解 Patron 为什么是"富领域模型"（包含业务逻辑）
- [ ] 理解策略模式如何封装业务规则
- [ ] 理解为什么业务方法返回 `Either` 而不是 `void` 或抛出异常
- [ ] 理解领域事件的作用

### 阶段 4-5：架构理解
- [ ] 理解应用服务和领域模型的职责划分
- [ ] 理解为什么仓储接口在 model 层，实现在 infrastructure 层
- [ ] 理解数据库实体和领域模型的区别

### 阶段 6-7：高级理解
- [ ] 理解事件驱动如何实现聚合间通信
- [ ] 理解 CQRS 的读写分离
- [ ] 理解如何测试领域逻辑
- [ ] 理解架构测试如何防止架构腐化

---

## 🎓 实践建议

### 1. 动手实践
尝试添加一个新功能，比如"图书归还"：

1. 在 `Patron` 聚合中添加 `returnBook` 方法
2. 创建 `ReturnBookCommand`
3. 创建应用服务 `ReturningBook`
4. 在 Controller 中添加端点
5. 编写测试

### 2. 对比学习
找一个传统的 CRUD 项目，尝试用 DDD 方式重构：

- 识别聚合边界
- 提取业务规则到领域模型
- 使用事件驱动替代直接调用

### 3. 阅读相关书籍
- 《领域驱动设计》- Eric Evans
- 《实现领域驱动设计》- Vaughn Vernon
- 《领域驱动设计精粹》- Scott Millett

---

## ❓ 常见问题

### Q1: 为什么不用 JPA？
**A**: 这个项目使用 Spring Data JDBC，因为：
- 不需要延迟加载（聚合较小）
- 不需要缓存
- 需要更精确的 SQL 控制
- 减少对象-关系阻抗不匹配

### Q2: 为什么领域模型不依赖 Spring？
**A**: 为了：
- 领域逻辑可独立测试
- 领域模型可移植（不绑定框架）
- 清晰的依赖方向（基础设施依赖领域）

### Q3: 为什么用 `Either` 而不是异常？
**A**: `Either` 是函数式编程的方式：
- 异常是副作用，`Either` 是值
- `Either` 在类型系统中，编译期可见
- 更符合函数式编程风格

### Q4: 这个项目是微服务吗？
**A**: 不是，这是**模块化单体**：
- 每个有界上下文是独立的模块
- 有独立的 Spring 应用上下文
- 未来可以拆分为微服务

---

## 🚀 下一步

完成以上学习后，你可以：

1. **深入理解事件溯源**：研究如何通过事件重建聚合状态
2. **学习 Saga 模式**：处理跨聚合的长事务
3. **学习限界上下文映射**：理解上下文间的关系
4. **实践微服务拆分**：将模块化单体拆分为微服务

---

**祝你学习愉快！** 🎉

如有问题，建议：
1. 先自己思考，尝试理解设计意图
2. 查看测试代码，测试往往是最好的文档
3. 阅读项目文档和设计图
4. 参考 DDD 相关书籍

