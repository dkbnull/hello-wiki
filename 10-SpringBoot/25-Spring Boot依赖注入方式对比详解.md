> 依赖注入（Dependency Injection，DI）是 Spring 框架的核心特性之一，也是实现 IoC（控制反转）的具体方式。Spring 提供了多种依赖注入方式，每种方式各有优劣。本文系统讲解构造器注入、字段注入、Setter 注入、方法注入的用法与区别，并涵盖 `@Autowired` 与 `@Resource` 的差异、循环依赖问题、最佳实践等内容。

---

## 一、依赖注入的基本概念

### 1.1 什么是依赖注入

依赖注入是指对象不再自行创建其依赖项，而是由容器（Spring IoC 容器）在运行时将依赖项注入到对象中。

```
传统方式（手动创建依赖）：
    Controller ──new──→ Service ──new──→ Repository
    对象自己控制依赖的创建，耦合度高

依赖注入（容器管理依赖）：
    Container ──注入──→ Controller
    Container ──注入──→ Service
    Container ──注入──→ Repository
    容器统一管理对象的创建和装配，对象只声明需要的依赖
```

### 1.2 依赖注入的三个角色

| 角色 | 说明 | 示例 |
|------|------|------|
| **依赖者** | 需要其他组件的对象 | `UserController` |
| **依赖项** | 被需要的组件 | `UserService` |
| **注入者** | 负责将依赖项注入依赖者 | Spring IoC 容器 |

### 1.3 Spring 依赖注入方式总览

| 注入方式 | 注解 / 配置 | 特点 |
|---------|------------|------|
| **构造器注入** | 构造方法 + `@Autowired`（可省略） | 官方推荐，不可变，可保证完全初始化 |
| **字段注入** | `@Autowired` 标注在字段上 | 简洁，但不推荐 |
| **Setter 注入** | `@Autowired` 标注在 Setter 方法上 | 可选依赖，可重新注入 |
| **方法注入** | `@Autowired` 标注在普通方法上 | 灵活，可注入多个依赖 |
| **Lookup 方法注入** | `@Lookup` | 解决单例注入原型的问题 |

---

## 二、构造器注入

### 2.1 基本用法

通过构造方法将依赖传入类中。Spring 4.3+ 如果类只有一个构造方法，`@Autowired` 可以省略。

```java
import org.springframework.stereotype.Service;

/**
 * 订单服务
 */
@Service
public class OrderService {

    private final UserService userService;
    private final OrderRepository orderRepository;

    /**
     * 构造器注入（Spring 4.3+ 单构造方法可省略 @Autowired）
     */
    public OrderService(UserService userService, OrderRepository orderRepository) {
        this.userService = userService;
        this.orderRepository = orderRepository;
    }

    public void createOrder(Long userId) {
        // 使用注入的依赖
        userService.getUser(userId);
        orderRepository.save(null);
    }
}
```

### 2.2 结合 Lombok 简化

实际项目中最常见的写法，使用 `@RequiredArgsConstructor` 自动生成构造方法：

```java
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

/**
 * 订单服务
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class OrderService {

    private final UserService userService;           // final 字段
    private final OrderRepository orderRepository;   // final 字段

    public void createOrder(Long userId) {
        userService.getUser(userId);
        orderRepository.save(null);
    }
}
```

> **关键**：`@RequiredArgsConstructor` 只为 `final` 字段和 `@NonNull` 标注的字段生成构造方法。非 final 字段不会包含在构造方法中。

### 2.3 多构造方法场景

当类有多个构造方法时，必须用 `@Autowired` 指定主构造方法：

```java
@Service
public class OrderService {

    private final UserService userService;
    private final OrderRepository orderRepository;
    private final LogService logService;  // 可选依赖

    /**
     * 主构造方法（Spring 使用此构造方法实例化）
     */
    @Autowired
    public OrderService(UserService userService, OrderRepository orderRepository) {
        this.userService = userService;
        this.orderRepository = orderRepository;
        this.logService = null;
    }

    /**
     * 辅助构造方法（不参与 Spring 注入）
     */
    public OrderService(UserService userService, OrderRepository orderRepository,
                        LogService logService) {
        this.userService = userService;
        this.orderRepository = orderRepository;
        this.logService = logService;
    }
}
```

### 2.4 优缺点

| 优点 | 缺点 |
|------|------|
| ✅ 依赖不可变（`final` 字段），线程安全 | ❌ 依赖较多时构造方法参数过多 |
| ✅ 对象创建即完全初始化，不会出现 NPE | ❌ 无法注入循环依赖（但这本身是好事） |
| ✅ 易于单元测试（可直接 new 传入 mock） | ❌ 需要配合 Lombok 减少样板代码 |
| ✅ 明确声明所有必需依赖 | |
| ✅ Spring 官方推荐 | |

---

## 三、字段注入

### 3.1 基本用法

直接在字段上标注 `@Autowired`，Spring 通过反射注入依赖：

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

/**
 * 订单服务（字段注入 - 不推荐）
 */
@Service
public class OrderService {

    @Autowired
    private UserService userService;

    @Autowired
    private OrderRepository orderRepository;

    public void createOrder(Long userId) {
        userService.getUser(userId);
        orderRepository.save(null);
    }
}
```

### 3.2 优缺点

| 优点 | 缺点 |
|------|------|
| ✅ 代码简洁，最少样板代码 | ❌ 无法使用 `final`，依赖可变，不安全 |
| ✅ 不需要构造方法或 Setter | ❌ 单元测试困难（无法通过构造方法传入 mock） |
| | ❌ 隐藏了依赖关系（类表面看不到依赖） |
| | ❌ 违反单一职责（加依赖太容易，容易过度膨胀） |
| | ❌ 依赖注入发生在对象创建之后，可能存在半初始化状态 |
| | ❌ Spring 官方不推荐 |

> **Spring 官方建议**：从 Spring 4.0 开始，官方文档明确推荐构造器注入，不推荐字段注入。

### 3.3 为什么字段注入仍被广泛使用

尽管不推荐，字段注入在实际项目中仍很常见，原因：

- 代码量少，开发效率高
- 早期 Spring 教程普遍使用字段注入
- 依赖较少时，缺点不明显

但这是**技术债务**，建议在新项目中避免使用。

---

## 四、Setter 注入

### 4.1 基本用法

通过 Setter 方法注入依赖：

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

/**
 * 订单服务（Setter 注入）
 */
@Service
public class OrderService {

    private UserService userService;
    private OrderRepository orderRepository;

    @Autowired
    public void setUserService(UserService userService) {
        this.userService = userService;
    }

    @Autowired
    public void setOrderRepository(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    public void createOrder(Long userId) {
        userService.getUser(userId);
        orderRepository.save(null);
    }
}
```

### 4.2 可选依赖

通过 `required = false` 标记可选依赖，当容器中找不到对应 Bean 时不报错：

```java
@Autowired(required = false)
public void setLogService(LogService logService) {
    this.logService = logService;
}
```

> **注意**：构造器注入不支持 `required = false`。如果需要可选依赖，必须使用 Setter 注入。

### 4.3 优缺点

| 优点 | 缺点 |
|------|------|
| ✅ 支持可选依赖（`required = false`） | ❌ 依赖可变（非 final），不保证不变性 |
| ✅ 可以在对象创建后重新注入（热替换） | ❌ 对象可能处于未完全初始化状态 |
| ✅ 单元测试方便（可调用 Setter 传入 mock） | ❌ 代码量比字段注入多 |
| ✅ 解决循环依赖（与构造器注入配合） | |

---

## 五、方法注入

### 5.1 基本用法

在任意方法上标注 `@Autowired`，可以一次注入多个依赖：

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

/**
 * 订单服务（方法注入）
 */
@Service
public class OrderService {

    private UserService userService;
    private OrderRepository orderRepository;

    @Autowired
    public void configureDependencies(UserService userService, OrderRepository orderRepository) {
        this.userService = userService;
        this.orderRepository = orderRepository;
    }

    public void createOrder(Long userId) {
        userService.getUser(userId);
        orderRepository.save(null);
    }
}
```

### 5.2 优缺点

| 优点 | 缺点 |
|------|------|
| ✅ 灵活，可一次注入多个依赖 | ❌ 不如构造器注入清晰 |
| ✅ 方法名可自定义，语义更灵活 | ❌ 使用场景少 |
| ✅ 可以在方法中添加初始化逻辑 | |

> **使用场景**：较少使用，通常构造器注入或 Setter 注入已足够。适合需要同时设置多个相关依赖并执行初始化逻辑的场景。

---

## 六、@Autowired 与 @Resource 的区别

`@Autowired` 和 `@Resource` 都用于依赖注入，但来源和行为不同。

### 6.1 基本对比

| 特性 | `@Autowired` | `@Resource` |
|------|-------------|-------------|
| **来源** | Spring（`org.springframework.beans.factory.annotation`） | JSR-250 标准（`jakarta.annotation`） |
| **默认匹配方式** | 按类型（byType） | 按名称（byName） |
| **找不到时的行为** | `required = false` 可设为可选 | 无法设为可选，找不到直接报错 |
| **配合注解** | `@Qualifier` 指定 Bean 名称 | `name` 属性直接指定 |
| **适用位置** | 构造器、字段、Setter、方法 | 字段、Setter（不支持构造器） |
| **框架依赖** | 仅 Spring | 标准 JSR-250，其他框架也支持 |

### 6.2 @Autowired 匹配流程

```
@Autowired
private UserService userService;

匹配流程：
1. 按类型查找 UserService 的 Bean
2. 找到唯一 → 注入 ✅
3. 找到多个 → 按字段名 "userService" 匹配 Bean 名称
4. 仍不唯一 → 报错 NoUniqueBeanDefinitionException
5. 配合 @Qualifier("beanName") → 按指定名称注入
```

示例：

```java
// 多个实现类场景
public interface PaymentService {
    void pay(String orderId);
}

@Service("alipayService")
public class AlipayServiceImpl implements PaymentService {
    @Override
    public void pay(String orderId) {
        System.out.println("支付宝支付: " + orderId);
    }
}

@Service("wechatPayService")
public class WechatPayServiceImpl implements PaymentService {
    @Override
    public void pay(String orderId) {
        System.out.println("微信支付: " + orderId);
    }
}
```

多实现场景的注入方式：

```java
// 方式1：@Autowired + @Qualifier（按名称指定）
@Autowired
@Qualifier("alipayService")
private PaymentService paymentService;

// 方式2：字段名与 Bean 名称一致（Spring 自动匹配）
@Autowired
private PaymentService alipayService;  // 字段名 "alipayService" 匹配 Bean 名称

// 方式3：@Primary 标注默认实现
@Service
@Primary  // 多个实现时，@Autowired 优先注入标注了 @Primary 的
public class AlipayServiceImpl implements PaymentService { }
```

### 6.3 @Resource 匹配流程

```
@Resource
private UserService userService;

匹配流程：
1. 如果指定了 name → 只按名称查找
2. 如果未指定 name → 先按字段名 "userService" 查找
3. 找不到 → 再按类型查找
4. 仍找不到 → 报错 NoSuchBeanDefinitionException
```

示例：

```java
// 方式1：默认按字段名匹配
@Resource
private PaymentService alipayService;  // 按名称 "alipayService" 查找

// 方式2：指定 name
@Resource(name = "wechatPayService")
private PaymentService paymentService;

// 方式3：指定 type
@Resource(type = AlipayServiceImpl.class)
private PaymentService paymentService;
```

### 6.4 选型建议

| 场景 | 推荐 |
|------|------|
| Spring 项目，构造器注入 | `@Autowired`（`@Resource` 不支持构造器） |
| 需要按名称注入 | `@Autowired` + `@Qualifier` |
| 跨框架兼容 | `@Resource`（JSR-250 标准） |
| 简单项目 | 均可，团队统一即可 |

> **Spring 官方推荐**：优先使用 `@Autowired`，构造器注入时 `@Autowired` 可省略。

---

## 七、@Autowired 的 required 属性

### 7.1 必需依赖 vs 可选依赖

```java
// 必需依赖（默认 required = true）
// 容器中找不到对应 Bean 会启动失败
@Autowired
private UserService userService;

// 可选依赖（required = false）
// 容器中找不到对应 Bean 不报错，注入 null
@Autowired(required = false)
private LogService logService;
```

### 7.2 各注入方式对 required 的支持

| 注入方式 | 支持 `required = false` | 说明 |
|---------|------------------------|------|
| 构造器注入 | ❌ 不支持 | 构造方法参数始终是必需的 |
| 字段注入 | ✅ 支持 | `@Autowired(required = false)` |
| Setter 注入 | ✅ 支持 | `@Autowired(required = false)` |
| 方法注入 | ✅ 支持 | `@Autowired(required = false)` |

### 7.3 Optional 替代方案

Java 8+ 推荐用 `Optional` 声明可选依赖，语义更清晰：

```java
import java.util.Optional;

@Service
public class OrderService {

    private final UserService userService;
    private final Optional<LogService> logService;

    public OrderService(UserService userService, Optional<LogService> logService) {
        this.userService = userService;
        this.logService = logService;
    }

    public void createOrder(Long userId) {
        userService.getUser(userId);
        // 使用前判断是否存在
        logService.ifPresent(service -> service.log("创建订单: " + userId));
    }
}
```

> **注意**：`Optional` 只能用于构造器注入和 Setter 注入，不能用于字段注入。

---

## 八、循环依赖问题

### 8.1 什么是循环依赖

两个或多个 Bean 互相依赖，形成闭环：

```
A 依赖 B，B 依赖 A → 循环依赖

@Service
class A {
    @Autowired 
    B b;  // A 需要 B
}

@Service
class B {
    @Autowired 
    A a;  // B 需要 A
}
```

### 8.2 Spring 如何解决循环依赖

Spring 通过**三级缓存**机制解决 Setter/字段注入的循环依赖：

| 缓存级别 | 名称 | 存储内容 |
|---------|------|---------|
| 一级缓存 | `singletonObjects` | 完全初始化好的 Bean（成品） |
| 二级缓存 | `earlySingletonObjects` | 提前暴露的半成品 Bean（已实例化，未注入依赖） |
| 三级缓存 | `singletonFactories` | Bean 工厂（ObjectFactory，用于生成半成品） |

```
解决循环依赖的流程（A 依赖 B，B 依赖 A）：

1. 创建 A：实例化 A → 将 A 的工厂放入三级缓存
2. 注入 A 的依赖 B → 创建 B
3. 创建 B：实例化 B → 将 B 的工厂放入三级缓存
4. 注入 B 的依赖 A → 查找 A
   → 一级缓存没有
   → 二级缓存没有
   → 三级缓存有 A 的工厂 → 生成 A 的半成品 → 放入二级缓存 → 删除三级缓存
5. B 拿到 A 的半成品 → B 注入完成 → B 放入一级缓存
6. A 拿到 B → A 注入完成 → A 放入一级缓存
```

### 8.3 构造器注入的循环依赖

```
⚠️ 构造器注入的循环依赖无法解决！

A 的构造方法需要 B → 创建 B
B 的构造方法需要 A → 创建 A
A 的构造方法需要 B → ... 死循环

Spring 会抛出：BeanCurrentlyInCreationException
```

### 8.4 各注入方式的循环依赖支持

| 注入方式 | 是否支持循环依赖 | 说明 |
|---------|----------------|------|
| 构造器注入 | ❌ 不支持 | 无法实例化任何一方，直接报错 |
| 字段注入 | ✅ 支持 | 通过三级缓存解决 |
| Setter 注入 | ✅ 支持 | 通过三级缓存解决 |
| 方法注入 | ✅ 支持 | 通过三级缓存解决 |

> **关键理解**：构造器注入不支持循环依赖，这其实是**优点**而非缺点。循环依赖通常意味着设计有问题，构造器注入强制你面对并解决它。

### 8.5 循环依赖的解决方案

| 方案 | 说明 | 适用场景 |
|------|------|---------|
| **重构消除循环** | 提取公共逻辑到第三个类 | 首选方案 |
| **改为 Setter 注入** | 将其中一方改为 Setter 注入 | 临时方案，不推荐长期保留 |
| **使用 `@Lazy`** | 延迟注入代理对象 | 临时方案 |
| **事件机制解耦** | 用事件代替直接调用 | 适合通知类场景 |

`@Lazy` 方案：

```java
@Service
public class OrderService {

    private final UserService userService;

    // @Lazy 注入代理对象，首次使用时才真正初始化
    public OrderService(@Lazy UserService userService) {
        this.userService = userService;
    }
}
```

---

## 九、Lookup 方法注入

### 9.1 问题场景

**单例 Bean 注入原型 Bean** 时，原型 Bean 只在注入时创建一次，之后不再创建新的：

```java
@Service
public class OrderService {

    // 希望每次使用都是新的实例，但实际只创建了一次
    @Autowired
    private PrototypeBean prototypeBean;  // 始终是同一个对象 ❌
}

@Scope("prototype")
@Component
public class PrototypeBean {
    // 原型作用域，应该每次获取都创建新实例
}
```

### 9.2 解决方案：@Lookup

```java
import org.springframework.beans.factory.annotation.Lookup;
import org.springframework.stereotype.Service;

/**
 * 订单服务
 */
@Service
public abstract class OrderService {

    /**
     * Lookup 方法注入：每次调用都返回新的 PrototypeBean 实例
     * Spring 通过 CGLIB 动态子类化重写此方法
     */
    @Lookup
    protected abstract PrototypeBean getPrototypeBean();

    public void processOrder(String orderId) {
        // 每次调用 getPrototypeBean() 都返回新实例
        PrototypeBean bean = getPrototypeBean();
        bean.doSomething(orderId);
    }
}
```

> **注意**：`@Lookup` 标注的方法必须是抽象方法（或返回值与 Bean 类型匹配的具体方法），Spring 通过 CGLIB 生成子类重写该方法。因此类不能是 `final`，方法也不能是 `final`/`private`。

### 9.3 替代方案：ObjectFactory

如果不想用抽象类，可以用 `ObjectFactory`：

```java
import org.springframework.beans.factory.ObjectProvider;
import org.springframework.stereotype.Service;

@Service
public class OrderService {

    private final ObjectProvider<PrototypeBean> prototypeBeanProvider;

    public OrderService(ObjectProvider<PrototypeBean> prototypeBeanProvider) {
        this.prototypeBeanProvider = prototypeBeanProvider;
    }

    public void processOrder(String orderId) {
        // 每次调用 getObject() 都返回新实例
        PrototypeBean bean = prototypeBeanProvider.getObject();
        bean.doSomething(orderId);
    }
}
```

> **优势**：`ObjectProvider` 是构造器注入友好的方案，不需要抽象类，推荐使用。

---

## 十、泛型注入

Spring 支持泛型类型的依赖注入，通过类型参数自动匹配对应实现。

### 10.1 泛型基类

```java
/**
 * 通用 Repository 基类
 * @param <T> 实体类型
 */
public abstract class BaseRepository<T> {
    public abstract T findById(Long id);
    public abstract void save(T entity);
}
```

### 10.2 具体实现

```java
import org.springframework.stereotype.Repository;

@Repository
public class UserRepository extends BaseRepository<User> {
    @Override
    public User findById(Long id) {
        return new User(id, "张三");
    }

    @Override
    public void save(User entity) {
        System.out.println("保存用户: " + entity);
    }
}

@Repository
public class OrderRepository extends BaseRepository<Order> {
    @Override
    public Order findById(Long id) {
        return new Order(id, 100.0);
    }

    @Override
    public void save(Order entity) {
        System.out.println("保存订单: " + entity);
    }
}
```

### 10.3 泛型 Service 注入

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

/**
 * 通用 Service 基类
 * Spring 根据泛型参数自动匹配对应的 Repository
 */
public abstract class BaseService<T> {

    protected BaseRepository<T> repository;

    @Autowired
    public void setRepository(BaseRepository<T> repository) {
        this.repository = repository;
    }

    public T getById(Long id) {
        return repository.findById(id);
    }

    public void save(T entity) {
        repository.save(entity);
    }
}
```

```java
@Service
public class UserService extends BaseService<User> {
    // Spring 自动注入 UserRepository（因为 BaseRepository<User> 匹配 UserRepository）
}

@Service
public class OrderService extends BaseService<Order> {
    // Spring 自动注入 OrderRepository（因为 BaseRepository<Order> 匹配 OrderRepository）
}
```

使用：

```java
@Autowired
private UserService userService;

userService.getById(1L);     // 调用 UserRepository.findById()
userService.save(new User()); // 调用 UserRepository.save()
```

> **优势**：泛型注入实现了代码复用，避免了为每个实体重复编写 CRUD 方法。

---

## 十一、Bean 生命周期回调与注入

### 11.1 注入时机

```
Bean 生命周期中的注入时机：

1. 实例化（Instantiation）       → 构造方法调用
2. 属性填充（Population）        → 依赖注入发生在此阶段
3. 初始化（Initialization）
   ├── @PostConstruct           → 注入完成后执行
   ├── InitializingBean.afterPropertiesSet()
   └── @Bean(initMethod=...)    
4. 使用（In Use）
5. 销毁（Destruction）
   ├── @PreDestroy
   ├── DisposableBean.destroy()
   └── @Bean(destroyMethod=...)
```

### 11.2 @PostConstruct 中使用注入的依赖

```java
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import org.springframework.stereotype.Service;

/**
 * 订单服务
 */
@Service
public class OrderService {

    private final UserService userService;

    public OrderService(UserService userService) {
        this.userService = userService;
        // ⚠️ 此时依赖已注入，但如果通过字段注入则可能还未注入
        System.out.println("1. 构造方法");
    }

    @PostConstruct
    public void init() {
        // ✅ 所有依赖已注入完毕，可以安全使用
        System.out.println("2. @PostConstruct: 依赖已就绪");
        userService.warmUpCache();
    }

    @PreDestroy
    public void cleanup() {
        System.out.println("3. @PreDestroy: 清理资源");
    }
}
```

> **关键**：`@PostConstruct` 在所有依赖注入完成后执行，是执行初始化逻辑的安全位置。**不要在构造方法中调用被注入的依赖的方法**（字段注入场景下依赖可能还未注入）。

---

## 十二、四种注入方式横向对比

### 12.1 全面对比

| 维度 | 构造器注入 | 字段注入 | Setter 注入 | 方法注入 |
|------|----------|---------|------------|---------|
| **不可变性** | ✅ `final` | ❌ | ❌ | ❌ |
| **完全初始化保证** | ✅ | ❌ | ❌ | ❌ |
| **单元测试便利性** | ✅ 可直接 new | ❌ 需反射 | ✅ 可 Setter | ✅ 可调用方法 |
| **可选依赖** | ❌ | ✅ | ✅ | ✅ |
| **循环依赖** | ❌ 不支持 | ✅ 支持 | ✅ 支持 | ✅ 支持 |
| **代码简洁度** | ⭐⭐⭐（Lombok） | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐ |
| **依赖可见性** | ✅ 构造方法明确 | ❌ 隐藏 | ⭐ Setter 明确 | ⭐ 方法明确 |
| **Spring 官方推荐** | ✅ 强烈推荐 | ❌ 不推荐 | ⭐ 可选依赖时使用 | ⭐ 特殊场景 |
| **违反单一职责感知** | ✅ 参数过多有警示 | ❌ 加字段太容易 | ⭐ 中等 | ⭐ 中等 |

### 12.2 适用场景

| 场景 | 推荐方式 | 理由 |
|------|---------|------|
| **必需依赖** | 构造器注入 | 保证不可变和完全初始化 |
| **可选依赖** | Setter 注入 | `required = false` |
| **简单工具类** | 字段注入（可接受） | 代码简洁，但仅限非核心逻辑 |
| **需要多个相关依赖同时设置** | 方法注入 | 可在方法中添加初始化逻辑 |
| **单例注入原型** | `ObjectProvider` / `@Lookup` | 每次获取新实例 |
| **通用泛型基类** | 泛型注入 | 代码复用 |
| **配置类 `@Bean` 方法** | 方法参数注入 | Spring 自动从容器中查找 |

### 12.3 @Bean 方法中的注入

在 `@Configuration` 类中，`@Bean` 方法的参数会自动从容器中注入：

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AppConfig {

    /**
     * 参数 userService 自动从容器中注入
     * 无需标注 @Autowired（Spring 4.3+ 自动注入）
     */
    @Bean
    public OrderService orderService(UserService userService, OrderRepository orderRepository) {
        return new OrderService(userService, orderRepository);
    }
}
```

---

## 十三、最佳实践

### 13.1 项目中的注入规范

```java
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

/**
 * 订单服务 - 推荐写法示范
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class OrderService {

    // ✅ 必需依赖：final + 构造器注入（Lombok 生成构造方法）
    private final UserService userService;
    private final OrderRepository orderRepository;

    // ✅ 可选依赖：ObjectProvider（构造器友好）
    private final ObjectProvider<LogService> logServiceProvider;

    /**
     * 创建订单
     */
    public void createOrder(Long userId, OrderCreateRequest request) {
        // 使用必需依赖
        User user = userService.getById(userId);

        // 使用可选依赖
        logServiceProvider.ifAvailable(service -> 
            service.log("创建订单: userId=" + userId)
        );

        // 业务逻辑
        Order order = new Order();
        order.setUserId(userId);
        order.setAmount(request.getAmount());
        orderRepository.save(order);
    }
}
```

### 13.2 最佳实践清单

| 编号 | 实践 | 说明 |
|------|------|------|
| 1 | **构造器注入优先** | 所有必需依赖用构造器注入，配合 `@RequiredArgsConstructor` |
| 2 | **字段加 `final`** | 保证依赖不可变，线程安全 |
| 3 | **避免字段注入** | 除非是遗留代码或快速原型 |
| 4 | **可选依赖用 `ObjectProvider`** | 比 `@Autowired(required = false)` 语义更清晰 |
| 5 | **多实现用 `@Qualifier`** | 明确指定 Bean 名称，避免歧义 |
| 6 | **接口编程** | 注入接口类型，而非实现类 |
| 7 | **避免循环依赖** | 构造器注入天然避免，如遇循环则重构 |
| 8 | **不在构造方法中调用依赖方法** | 字段注入场景下依赖可能未就绪 |
| 9 | **初始化逻辑放 `@PostConstruct`** | 此时所有依赖已注入 |
| 10 | **团队统一风格** | 全部用构造器注入 + Lombok，保持一致性 |

### 13.3 单元测试对比

构造器注入的测试便利性：

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.mockito.Mockito.*;

/**
 * 构造器注入 → 单元测试极简
 */
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private UserService userService;

    @Mock
    private OrderRepository orderRepository;

    @Test
    void testCreateOrder() {
        // 直接 new 传入 mock，无需 Spring 容器
        OrderService orderService = new OrderService(userService, orderRepository);

        when(userService.getById(1L)).thenReturn(new User(1L, "张三"));

        orderService.createOrder(1L, new OrderCreateRequest());

        verify(orderRepository, times(1)).save(any());
    }
}
```

字段注入的测试困难：

```java
/**
 * 字段注入 → 单元测试困难
 * 需要反射或 Spring TestContext 才能注入 mock
 */
class OrderServiceFieldInjectionTest {

    @Test
    void testCreateOrder() throws Exception {
        OrderService service = new OrderService();

        // 需要通过反射注入私有字段（非常 hacky）
        Field field = OrderService.class.getDeclaredField("userService");
        field.setAccessible(true);
        field.set(service, mock(UserService.class));

        // ... 测试逻辑
    }
}
```

---

## 十四、总结

### 14.1 核心要点速查

1. **构造器注入是官方推荐**：不可变、完全初始化、测试友好
2. **字段注入不推荐**：隐藏依赖、不可变、测试困难
3. **Setter 注入用于可选依赖**：`required = false` 或 `ObjectProvider`
4. **`@Autowired` 按类型匹配**，配合 `@Qualifier` 按名称指定
5. **`@Resource` 按名称匹配**，JSR-250 标准，不支持构造器
6. **多实现场景**：`@Primary` 指定默认 + `@Qualifier` 精确指定
7. **构造器注入不支持循环依赖**：这是优点，强制解决设计问题
8. **Spring 三级缓存解决 Setter/字段注入的循环依赖**
9. **单例注入原型用 `ObjectProvider` 或 `@Lookup`**
10. **泛型注入实现代码复用**：`BaseRepository<T>` 自动匹配实现
11. **`@PostConstruct` 是初始化逻辑的安全位置**
12. **Lombok `@RequiredArgsConstructor` 简化构造器注入**

### 14.2 决策树

```
依赖是否必需？
  ├── 必需 → 构造器注入（final + @RequiredArgsConstructor）
  └── 可选 → ObjectProvider（构造器友好）
            或 Setter 注入（@Autowired(required = false)）

是否有多个实现？
  ├── 是 → @Primary 指定默认 + @Qualifier 精确指定
  └── 否 → 直接注入

是否是单例注入原型？
  ├── 是 → ObjectProvider 或 @Lookup
  └── 否 → 正常注入

是否遇到循环依赖？
  ├── 是 → 重构消除循环（首选）
  │        或 @Lazy 延迟注入（临时）
  └── 否 → 正常注入
```

### 14.3 参考文档

- [Spring Framework - Dependency Injection](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html)
- [Spring Framework - Constructor-based DI](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html#beans-constructor-injection)
- [Spring Framework - Setter-based DI](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html#beans-setter-injection)
- [Spring Framework - @Autowired](https://docs.spring.io/spring-framework/reference/core/beans/annotation-config/autowired.html)
- [Spring Framework - Method Injection](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-method-injection.html)
- [Spring Framework - Circular Dependencies](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html#beans-dependency-resolution)

---

<p align="center">
    <a href="https://github.com/dkbnull/hello-wiki/blob/main/10-SpringBoot/25-Spring%20Boot%E4%BE%9D%E8%B5%96%E6%B3%A8%E5%85%A5%E6%96%B9%E5%BC%8F%E5%AF%B9%E6%AF%94%E8%AF%A6%E8%A7%A3.md" target="_blank">
        <img src="https://img.shields.io/badge/GitHub-访问地址-blue?logo=github">
    </a>
    <a href="https://gitee.com/dkbnull/hello-wiki/blob/main/10-SpringBoot/25-Spring%20Boot%E4%BE%9D%E8%B5%96%E6%B3%A8%E5%85%A5%E6%96%B9%E5%BC%8F%E5%AF%B9%E6%AF%94%E8%AF%A6%E8%A7%A3.md" target="_blank">
        <img src="https://img.shields.io/badge/Gitee-访问地址-red?logo=gitee">
    </a>
    <a href="https://mp.weixin.qq.com/s/AT0SZsxNDlvVzqoLDQ2dbQ" target="_blank">
       <img src="https://img.shields.io/badge/微信公众号-访问地址-brightgreen?logo=wechat">
    </a>
    <a href="https://juejin.cn/post/7665553448478507018" target="_blank">
       <img src="https://img.shields.io/badge/掘金-访问地址-blue?logo=juejin">
    </a>
    <a href="https://zhuanlan.zhihu.com/p/2064130431272870550" target="_blank">
       <img src="https://img.shields.io/badge/知乎-访问地址-blue?logo=zhihu">
    </a>
</p>
