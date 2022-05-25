# 传统 CRUD 与 DDD 架构对比速查表

## 📋 目录结构对比

### 传统 CRUD 项目
```
src/main/java/com/example/
├── controller/
│   └── UserController.java      # HTTP请求处理
├── service/
│   └── UserService.java          # 业务逻辑
├── repository/
│   └── UserRepository.java       # 数据访问
└── entity/
    └── User.java                 # JPA实体（数据库表映射）
```

### DDD 项目
```
src/main/java/io/pillopl/library/
├── lending/                      # 有界上下文
│   ├── patron/                   # 读者聚合
│   │   ├── model/                # 领域模型（核心）
│   │   │   └── Patron.java       # 聚合根（包含业务逻辑）
│   │   ├── application/          # 应用服务（编排）
│   │   │   └── hold/
│   │   │       └── PlacingOnHold.java
│   │   └── infrastructure/       # 基础设施
│   │       └── PatronsDatabaseRepository.java
│   └── book/                     # 图书聚合
└── catalogue/                    # 另一个有界上下文
```

---

## 🔄 业务流程对比

### 场景：用户预约图书

#### 传统 CRUD 方式

```java
// 1. Controller 层
@RestController
public class PatronController {
    @Autowired
    private PatronService patronService;
    
    @PostMapping("/patrons/{id}/holds")
    public ResponseEntity placeHold(@PathVariable Long id, @RequestBody HoldRequest request) {
        patronService.placeHold(id, request.getBookId());
        return ResponseEntity.ok().build();
    }
}

// 2. Service 层（包含所有业务逻辑）
@Service
public class PatronService {
    @Autowired
    private PatronRepository patronRepo;
    @Autowired
    private BookRepository bookRepo;
    
    public void placeHold(Long patronId, Long bookId) {
        // 查找实体
        PatronEntity patron = patronRepo.findById(patronId)
            .orElseThrow(() -> new NotFoundException());
        BookEntity book = bookRepo.findById(bookId)
            .orElseThrow(() -> new NotFoundException());
        
        // 业务规则检查（散落在Service中）
        if (patron.getHolds().size() >= 5) {
            throw new BusinessException("最多只能预约5本书");
        }
        if (book.isRestricted() && patron.getType() == REGULAR) {
            throw new BusinessException("普通读者不能预约限制类图书");
        }
        if (patron.getOverdueCheckouts().size() > 2) {
            throw new BusinessException("逾期借阅超过2本，不能预约");
        }
        
        // 修改状态
        HoldEntity hold = new HoldEntity();
        hold.setPatronId(patronId);
        hold.setBookId(bookId);
        hold.setCreatedAt(LocalDateTime.now());
        holdRepository.save(hold);
        
        book.setStatus(ON_HOLD);
        bookRepo.save(book);
    }
}

// 3. Entity 层（贫血模型）
@Entity
public class PatronEntity {
    @Id
    private Long id;
    private String name;
    private PatronType type;
    @OneToMany
    private List<HoldEntity> holds;
    // 只有getter/setter，没有业务方法
}
```

**问题**：
- ❌ 业务逻辑分散在 Service 层
- ❌ Entity 只是数据容器（贫血模型）
- ❌ 业务规则难以测试和复用
- ❌ 容易违反单一职责原则

#### DDD 方式

```java
// 1. Controller 层（只负责HTTP）
@RestController
public class PatronProfileController {
    private final PlacingOnHold placingOnHold;  // 应用服务
    
    @PostMapping("/profiles/{patronId}/holds")
    public ResponseEntity placeHold(@PathVariable UUID patronId, @RequestBody PlaceHoldRequest request) {
        Try<Result> result = placingOnHold.placeOnHold(
            new PlaceOnHoldCommand(patronId, request.getBookId(), ...)
        );
        return result.map(success -> ResponseEntity.ok().build())
                     .getOrElse(ResponseEntity.status(500).build());
    }
}

// 2. Application 层（只负责编排）
public class PlacingOnHold {
    private final FindAvailableBook findAvailableBook;
    private final Patrons patronRepository;
    
    public Try<Result> placeOnHold(PlaceOnHoldCommand command) {
        // 查找聚合（通过接口）
        AvailableBook book = findAvailableBook.findAvailableBookBy(command.getBookId())
            .getOrElseThrow(() -> new IllegalArgumentException("Book not found"));
        Patron patron = patronRepository.findBy(command.getPatronId())
            .getOrElseThrow(() -> new IllegalArgumentException("Patron not found"));
        
        // 调用领域方法（业务逻辑在领域模型中）
        Either<BookHoldFailed, BookPlacedOnHoldEvents> result = 
            patron.placeOnHold(book, command.getHoldDuration());
        
        // 发布事件
        return Match(result).of(
            Case($Left($()), this::publishFailure),
            Case($Right($()), this::publishSuccess)
        );
    }
}

// 3. Domain 层（包含业务逻辑）
public class Patron {
    private final PatronInformation patron;
    private final List<PlacingOnHoldPolicy> placingOnHoldPolicies;  // 业务规则策略
    private final OverdueCheckouts overdueCheckouts;
    private final PatronHolds patronHolds;
    
    // 业务方法：返回 Either（成功或失败）
    public Either<BookHoldFailed, BookPlacedOnHoldEvents> placeOnHold(AvailableBook book, HoldDuration duration) {
        // 业务规则检查（通过策略模式）
        Option<Rejection> rejection = patronCanHold(book, duration);
        
        if (rejection.isEmpty()) {
            // 创建领域事件
            BookPlacedOnHold event = bookPlacedOnHoldNow(book.getBookId(), ...);
            return announceSuccess(events(event));
        }
        return announceFailure(bookHoldFailedNow(rejection.get(), ...));
    }
    
    // 业务规则检查（策略模式）
    private Option<Rejection> patronCanHold(AvailableBook book, HoldDuration duration) {
        return placingOnHoldPolicies
            .toStream()
            .map(policy -> policy.apply(book, this, duration))
            .find(Either::isLeft)
            .map(Either::getLeft);
    }
}

// 4. Infrastructure 层（实现仓储）
class PatronsDatabaseRepository implements Patrons {
    private final PatronEntityRepository jpaRepository;
    private final DomainModelMapper mapper;
    private final DomainEvents domainEvents;
    
    @Override
    public Option<Patron> findBy(PatronId patronId) {
        PatronDatabaseEntity entity = jpaRepository.findByPatronId(patronId.getPatronId());
        return Option.of(entity).map(mapper::map);  // 转换为领域模型
    }
    
    @Override
    public Patron publish(PatronEvent event) {
        // 通过事件更新状态（事件溯源）
        PatronDatabaseEntity entity = jpaRepository.findByPatronId(event.patronId().getPatronId());
        entity = entity.handle(event);
        entity = jpaRepository.save(entity);
        
        // 发布事件到事件总线
        domainEvents.publish(event.normalize());
        
        return mapper.map(entity);
    }
}
```

**优势**：
- ✅ 业务逻辑封装在领域模型中（富领域模型）
- ✅ 业务规则通过策略模式可测试、可组合
- ✅ 领域模型不依赖框架，可独立测试
- ✅ 清晰的职责划分

---

## 🎯 核心概念对比

| 概念 | 传统 CRUD | DDD |
|------|----------|-----|
| **组织方式** | 按技术层次（Controller/Service/Repository） | 按业务领域（聚合/有界上下文） |
| **业务逻辑位置** | Service 层 | 领域模型（Domain Model） |
| **实体类** | 贫血模型（只有getter/setter） | 富领域模型（包含业务方法） |
| **状态管理** | 枚举 + 字段 | 类型系统（不同类表示不同状态） |
| **错误处理** | 抛出异常 | 返回 Either/Result |
| **数据访问** | JPA/Hibernate | Spring Data JDBC 或 JdbcTemplate |
| **聚合间通信** | 直接调用 Service | 通过领域事件 |
| **测试** | 需要 Mock 框架 | 领域逻辑可独立测试 |

---

## 📊 代码示例对比

### 1. 状态管理

#### 传统方式
```java
@Entity
public class Book {
    @Enumerated(EnumType.STRING)
    private BookStatus status;  // AVAILABLE, ON_HOLD, CHECKED_OUT
    
    public void placeOnHold() {
        if (status != AVAILABLE) {  // 运行时检查
            throw new IllegalStateException("Book is not available");
        }
        this.status = ON_HOLD;
    }
}
```

#### DDD 方式
```java
// 用类型表示状态
public interface Book { }

public class AvailableBook implements Book {
    public BookOnHold placeOnHold(...) {  // 编译期保证：只能对AvailableBook调用
        return new BookOnHold(...);
    }
}

public class BookOnHold implements Book {
    public AvailableBook cancelHold() {  // 编译期保证：只能对BookOnHold调用
        return new AvailableBook(...);
    }
}
```

---

### 2. 业务规则

#### 传统方式
```java
@Service
public class PatronService {
    public void placeHold(Long patronId, Long bookId) {
        PatronEntity patron = patronRepo.findById(patronId).get();
        BookEntity book = bookRepo.findById(bookId).get();
        
        // 业务规则散落在Service中
        if (patron.getType() == REGULAR && patron.getHolds().size() >= 5) {
            throw new BusinessException("最多5个预约");
        }
        if (book.isRestricted() && patron.getType() == REGULAR) {
            throw new BusinessException("普通读者不能预约限制类图书");
        }
        // ...
    }
}
```

#### DDD 方式
```java
// 业务规则封装在领域模型中
public class Patron {
    private final List<PlacingOnHoldPolicy> policies;  // 策略列表
    
    public Either<BookHoldFailed, BookPlacedOnHoldEvents> placeOnHold(AvailableBook book, HoldDuration duration) {
        // 通过策略检查业务规则
        Option<Rejection> rejection = policies
            .toStream()
            .map(policy -> policy.apply(book, this, duration))
            .find(Either::isLeft)
            .map(Either::getLeft);
        
        if (rejection.isEmpty()) {
            return announceSuccess(events(...));
        }
        return announceFailure(...);
    }
}

// 具体策略（可测试、可组合）
PlacingOnHoldPolicy onlyResearcherCanHoldRestricted = (book, patron, duration) -> {
    if (book.isRestricted() && patron.isRegular()) {
        return left(Rejection.withReason("Regular patrons cannot hold restricted books"));
    }
    return right(new Allowance());
};
```

---

### 3. 错误处理

#### 传统方式
```java
public void placeHold(Long patronId, Long bookId) {
    if (patron.getHolds().size() >= 5) {
        throw new BusinessException("最多5个预约");  // 异常是副作用
    }
    // ...
}
```

#### DDD 方式
```java
// 返回 Either（函数式编程）
public Either<BookHoldFailed, BookPlacedOnHoldEvents> placeOnHold(AvailableBook book) {
    Option<Rejection> rejection = patronCanHold(book, duration);
    if (rejection.isEmpty()) {
        return right(events(...));  // 成功
    }
    return left(bookHoldFailedNow(...));  // 失败（值，不是异常）
}

// 使用方式
Either<BookHoldFailed, BookPlacedOnHoldEvents> result = patron.placeOnHold(book);
result.fold(
    failure -> handleFailure(failure),
    events -> handleSuccess(events)
);
```

---

### 4. 测试

#### 传统方式
```java
@Test
public void testPlaceHold() {
    // 需要 Mock 很多依赖
    PatronService service = new PatronService();
    when(patronRepo.findById(1L)).thenReturn(Optional.of(mockPatron));
    when(bookRepo.findById(1L)).thenReturn(Optional.of(mockBook));
    
    service.placeHold(1L, 1L);
    
    verify(holdRepo).save(any());
}
```

#### DDD 方式
```java
// 领域逻辑可独立测试，无需 Mock
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

---

## 🔍 关键差异总结

### 1. 思维转变

| 传统思维 | DDD 思维 |
|---------|---------|
| "我需要一个 UserService 来处理用户相关操作" | "用户是一个聚合，包含哪些业务概念？" |
| "业务逻辑写在 Service 层" | "业务逻辑封装在领域模型中" |
| "Entity 对应数据库表" | "领域模型反映业务概念，与数据库分离" |
| "用枚举表示状态" | "用类型表示状态" |
| "抛出异常处理错误" | "返回 Either/Result 表示可能失败的操作" |

### 2. 设计原则

**传统 CRUD**：
- 关注数据流转：Controller → Service → Repository → Database
- 按技术层次组织代码
- 实体是数据容器

**DDD**：
- 关注业务概念：聚合、实体、值对象、领域服务
- 按业务领域组织代码
- 领域模型包含业务逻辑

### 3. 依赖方向

**传统 CRUD**：
```
Controller → Service → Repository → Entity
（所有层都依赖数据库实体）
```

**DDD**：
```
Infrastructure → Application → Domain
（基础设施依赖领域，领域不依赖基础设施）
```

---

## 💡 学习建议

1. **先理解业务领域**：不要一开始就看代码，先理解"读者预约图书"这个业务场景
2. **从简单开始**：先看 `Book` 聚合，理解状态模型
3. **追踪完整流程**：选择一个业务流程（如预约），从 Controller 追踪到 Domain
4. **对比学习**：将 DDD 代码与你熟悉的 CRUD 代码对比
5. **动手实践**：尝试添加一个小功能，保持 DDD 风格

---

**记住**：DDD 不是技术框架，而是一种思维方式。核心是**将业务逻辑封装在领域模型中**，让代码更贴近业务语言。

