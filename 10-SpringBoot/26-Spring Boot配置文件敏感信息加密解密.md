> 配置文件是应用的安全软肋：数据库密码、Redis 密码、MQ 密码、第三方密钥等往往以明文形式存放在 `application.yml` 中，随代码一起提交到 Git 仓库，一旦仓库泄露，所有底牌尽出。本文系统讲解 Spring Boot 中配置字段加密、加载时自动解密的完整方案

---

## 一、为什么需要对配置文件加密

### 1.1 明文配置的风险场景

| 场景 | 风险说明 |
|------|---------|
| **代码仓库泄露** | 仓库被拖库、账号被盗、私有仓库误设公开，密码直接暴露 |
| **制品分发** | 打包进 jar 的 `application.yml` 随制品分发，解压即得密码 |
| **运维平台暴露** | 日志采集、监控探针、配置中心面板意外展示 |
| **内部越权** | 能看到服务器文件的人员，等同于拿到生产数据库权限 |

### 1.2 常见的敏感配置项

```yaml
spring:
  datasource:
    password: mysql@123456        # 数据库密码
  data:
    redis:
      password: redis@123456      # Redis 密码
  rabbitmq:
    password: rabbit@123456       # MQ 密码
  mail:
    password: mailbox@123456      # 邮箱授权码
```

加密方案的目标是把这些值替换为密文，Spring 加载配置时自动还原为明文，业务代码零感知。

### 1.3 加密的基本思路

```
开发阶段：明文密码 + 加密密钥 ──加密算法──→ 密文 ──→ 写入 application.yml
运行阶段：密文（配置文件） + 密钥（环境变量/启动参数）──解密──→ 明文 ──→ 注入数据源/Redis 客户端
```

关键原则：**密文可以落盘，密钥绝不落盘**。密文与密钥分离存放，是整套方案安全性的前提。

---

## 二、方案选型对比

| 方案 | 原理 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|---------|
| **jasypt-spring-boot** | 包装 `PropertySource`，读取时检测 `ENC()` 并解密 | 开箱即用、社区成熟 | 引入第三方依赖 | 绝大多数项目（推荐） |
| **自定义 EnvironmentPostProcessor** | 环境准备完成后遍历配置源解密替换 | 零依赖、逻辑可控 | 需自行实现维护 | 不允许引第三方库、有定制需求 |
| **Spring Cloud Config 加解密** | 配置中心服务端统一加解密 | 密钥集中在服务端 | 依赖配置中心架构 | 已使用 Spring Cloud 体系 |
| **Vault / KMS** | 密钥托管到专业密钥管理服务 | 安全等级最高 | 架构复杂、依赖云平台 | 金融等高安全要求场景 |
| **环境变量直传明文** | 不加密，密码直接放环境变量 | 零开发成本 | 未保护密码本身，只是换了存放位置 | 临时方案 |

本文重点讲解前两种——两者都遵循"**配置文件存密文 + 加载时解密**"的模型。

---

## 三、jasypt-spring-boot 快速上手

### 3.1 引入依赖

Spring Boot 3.x（Java 17+）使用 `3.0.5`；Spring Boot 2.x 使用 `3.0.4` 及以下版本。

```xml
<dependency>
    <groupId>com.github.ulisesbocchio</groupId>
    <artifactId>jasypt-spring-boot-starter</artifactId>
    <version>3.0.5</version>
</dependency>
```

> 若使用 `jasypt-spring-boot`（非 starter 版本），需在启动类上额外标注 `@EnableEncryptableProperties`；starter 版本自动配置，无需任何注解。

### 3.2 生成密文

使用纯 Java 工具类生成密文。

```java
import org.jasypt.encryption.pbe.PooledPBEStringEncryptor;
import org.jasypt.encryption.pbe.config.SimpleStringPBEConfig;

/**
 * Jasypt 密文生成工具
 * 仅在开发阶段本地运行，禁止提交到代码仓库
 */
public class JasyptUtil {

    public static void main(String[] args) {
        // 加密密钥：与运行时 jasypt.encryptor.password 保持一致
        String secretKey = "my-jasypt-secret-key";
        // 待加密明文：数据库、Redis 等真实密码，每个密码单独调用一次 encrypt
        PooledPBEStringEncryptor encryptor = buildEncryptor(secretKey);

        System.out.println("MySQL 密文：" + encryptor.encrypt("mysql@123456"));
        System.out.println("Redis 密文：" + encryptor.encrypt("redis@123456"));
    }

    /**
     * 构建加密器：算法与 IV 生成器必须与 jasypt.encryptor 配置完全一致，否则运行时解密失败
     */
    private static PooledPBEStringEncryptor buildEncryptor(String secretKey) {
        PooledPBEStringEncryptor encryptor = new PooledPBEStringEncryptor();
        SimpleStringPBEConfig config = new SimpleStringPBEConfig();
        config.setPassword(secretKey);
        config.setAlgorithm("PBEWITHHMACSHA512ANDAES_256");
        config.setIvGeneratorClassName("org.jasypt.iv.RandomIvGenerator");
        config.setPoolSize("1");
        encryptor.setConfig(config);
        return encryptor;
    }
}
```

### 3.3 MySQL 与 Redis 密码加密示例

将密文用 `ENC(...)` 包裹后写入配置：

```yaml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://127.0.0.1:3306/demo?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai&useSSL=false
    username: root
    # 明文 mysql@123456 加密后的密文
    password: ENC(S2pCV0Y5QXZvRnRNVzF3NE5OaWxWdz09PQ==)
  data:
    redis:
      host: 127.0.0.1
      port: 6379
      # 明文 redis@123456 加密后的密文
      password: ENC(U0RCdE9YRnNXblJqVjFwTFlXcGhUQT09)

# algorithm 与 iv-generator-classname 为 3.x 默认值，显式声明便于排查
jasypt:
  encryptor:
    # 密钥从环境变量读取，配置文件中不出现密钥本身
    password: ${JASYPT_ENCRYPTOR_PASSWORD}
    algorithm: PBEWITHHMACSHA512ANDAES_256
    iv-generator-classname: org.jasypt.iv.RandomIvGenerator
```

> Spring Boot 2.x 中 Redis 配置前缀为 `spring.redis.*`，3.x 调整为 `spring.data.redis.*`。

### 3.4 指定密钥启动

```bash
# 方式一：环境变量（推荐）
export JASYPT_ENCRYPTOR_PASSWORD=my-jasypt-secret-key
java -jar app.jar

# 方式二：JVM 系统属性（-D 必须写在 -jar 之前，否则被当作应用参数）
java -Djasypt.encryptor.password=my-jasypt-secret-key -jar app.jar
```

IDEA 本地启动则在 Run/Debug Configurations 的 Environment variables 中填入 `JASYPT_ENCRYPTOR_PASSWORD=my-jasypt-secret-key`。

多环境场景下各环境密钥独立，密文只对所属环境有效：`application-dev.yml` 写 dev 密钥加密的密文，`application-prod.yml` 写 prod 密钥加密的密文，部署时注入各自密钥：

```bash
export JASYPT_ENCRYPTOR_PASSWORD=prod环境密钥
java -jar app.jar --spring.profiles.active=prod
```

### 3.5 常用配置项

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `jasypt.encryptor.password` | 无 | 加解密密钥（必填），生产从环境变量注入 |
| `jasypt.encryptor.algorithm` | `PBEWITHHMACSHA512ANDAES_256`（3.x） | 2.x 默认为不安全的 `PBEWithMD5AndDES`，务必显式指定强算法 |
| `jasypt.encryptor.iv-generator-classname` | `RandomIvGenerator`（3.x） | 配合 AES 类算法必须为 `RandomIvGenerator` |
| `jasypt.encryptor.property-detector-classname` | `DefaultPropertyDetector` | 自定义密文识别规则（默认识别 `ENC()` 包裹） |

### 3.6 验证

启动后通过 `environment.getProperty("spring.datasource.password")` 读到的已是明文；数据库连接池初始化成功、Redis 读写正常即代表链路生效。可用临时 Runner 验证连通性：

```java
import lombok.RequiredArgsConstructor;
import org.springframework.boot.CommandLineRunner;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.Statement;

/**
 * 数据库与 Redis 连通性验证
 */
@Component
@RequiredArgsConstructor
public class ConnectionCheckRunner implements CommandLineRunner {

    private final DataSource dataSource;
    private final StringRedisTemplate stringRedisTemplate;

    @Override
    public void run(String... args) throws Exception {
        // 验证数据库连接（密码解密失败会在获取连接时抛出异常）
        try (Connection connection = dataSource.getConnection();
             Statement statement = connection.createStatement()) {
            statement.execute("SELECT 1");
            System.out.println("数据库连接成功");
        }

        // 验证 Redis 连接
        stringRedisTemplate.opsForValue().set("config:encrypt:test", "ok");
        System.out.println("Redis 连接成功：" + stringRedisTemplate.opsForValue().get("config:encrypt:test"));
    }
}
```

### 3.7 工作原理

jasypt 的核心机制是**在配置读取出口做拦截**：

```
application.yml（password: ENC(S2pCV0Y5...)）
        │
        ▼
jasypt 自动配置注册 BeanFactoryPostProcessor
将每个 PropertySource 包装为 EncryptablePropertySourceWrapper
        │
        ▼
业务代码任意方式读取（@Value / @ConfigurationProperties / Environment）
        │
        ├── 值不匹配 ^ENC\((.*)\)$ ──→ 原样返回
        └── 匹配 → 提取密文 → StringEncryptor.decrypt() → 返回明文
```

关键结论：

1. **对业务完全透明**：`@Value`、`@ConfigurationProperties`、`Environment` 直读拿到的都是明文，MySQL（HikariCP）、Redis（Lettuce/Jedis）等组件自动生效
2. **解密是惰性的**：属性被读取时才解密，不使用的属性即使密文损坏也不报错
3. **包装发生在容器刷新前**：`BeanFactoryPostProcessor` 阶段早于普通 Bean 创建，数据源初始化时已能读到明文

---

## 四、不依赖第三方库：自定义 EnvironmentPostProcessor

如果不希望引入三方库，可以通过 `EnvironmentPostProcessor` 实现：它在**环境准备完成后、容器刷新前**被回调，此时解密 `PropertySource` 中的密文属性，后续所有组件读到的就是明文。

```
SpringApplication.run()
    ├── 加载 application.yml → 组装 Environment
    ├── 触发 ApplicationEnvironmentPreparedEvent
    ├── EnvironmentPostProcessor 回调 ◄── 在这里遍历配置源、解密 ENC() 属性
    └── 容器刷新、Bean 创建（此时读到的已是明文）
```

### 4.1 AES 加解密工具类

采用 AES-GCM 认证加密，自带完整性校验，密文被篡改会解密失败。

```java
import javax.crypto.Cipher;
import javax.crypto.spec.GCMParameterSpec;
import javax.crypto.spec.SecretKeySpec;
import java.nio.charset.StandardCharsets;
import java.security.SecureRandom;
import java.util.Base64;

/**
 * AES-GCM 加解密工具类
 * 密文格式：Base64(IV + 密文)，IV 每次随机生成，同一明文每次加密结果不同
 */
public class AesUtils {

    /** GCM IV 长度（字节），NIST 推荐 12 字节 */
    private static final int IV_LENGTH = 12;
    /** GCM 认证标签长度（位） */
    private static final int TAG_BITS = 128;

    /**
     * 加密
     *
     * @param plainText 待加密明文
     * @param secretKey Base64 编码的密钥
     * @return Base64(IV + 密文)
     * @throws Exception 加密失败
     */
    public static String encrypt(String plainText, String secretKey) throws Exception {
        byte[] key = Base64.getDecoder().decode(secretKey);
        byte[] iv = new byte[IV_LENGTH];
        new SecureRandom().nextBytes(iv);

        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        cipher.init(Cipher.ENCRYPT_MODE, new SecretKeySpec(key, "AES"), new GCMParameterSpec(TAG_BITS, iv));
        byte[] cipherText = cipher.doFinal(plainText.getBytes(StandardCharsets.UTF_8));

        // IV 拼接在密文头部，随密文一起存储
        byte[] result = new byte[IV_LENGTH + cipherText.length];
        System.arraycopy(iv, 0, result, 0, IV_LENGTH);
        System.arraycopy(cipherText, 0, result, IV_LENGTH, cipherText.length);
        return Base64.getEncoder().encodeToString(result);
    }

    /**
     * 解密
     *
     * @param encryptedText Base64(IV + 密文)
     * @param secretKey Base64 编码的密钥
     * @return 明文
     * @throws Exception 解密失败（密钥错误或密文被篡改）
     */
    public static String decrypt(String encryptedText, String secretKey) throws Exception {
        byte[] key = Base64.getDecoder().decode(secretKey);
        byte[] data = Base64.getDecoder().decode(encryptedText);

        byte[] iv = new byte[IV_LENGTH];
        byte[] cipherText = new byte[data.length - IV_LENGTH];
        System.arraycopy(data, 0, iv, 0, IV_LENGTH);
        System.arraycopy(data, IV_LENGTH, cipherText, 0, cipherText.length);

        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        cipher.init(Cipher.DECRYPT_MODE, new SecretKeySpec(key, "AES"), new GCMParameterSpec(TAG_BITS, iv));
        return new String(cipher.doFinal(cipherText), StandardCharsets.UTF_8);
    }

    /**
     * 生成 Base64 编码的随机密钥
     *
     * @param keyBits 密钥位数（128 / 192 / 256）
     * @return Base64 密钥字符串
     */
    public static String generateKey(int keyBits) {
        byte[] key = new byte[keyBits / 8];
        new SecureRandom().nextBytes(key);
        return Base64.getEncoder().encodeToString(key);
    }
}
```

配套的密文生成入口：

```java
/**
 * 密文生成工具
 * 仅本地运行生成密文，禁止提交到代码仓库
 */
public class EncryptTool {

    public static void main(String[] args) throws Exception {
        String secretKey = "ruD3jYw7XkW2vQ9mZpA0sA==";   // AesUtils.generateKey(128) 生成
        System.out.println("加密结果：" + AesUtils.encrypt("mysql@123456", secretKey));
    }
}
```

### 4.2 实现 EnvironmentPostProcessor

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.env.EnvironmentPostProcessor;
import org.springframework.core.env.ConfigurableEnvironment;
import org.springframework.core.env.EnumerablePropertySource;
import org.springframework.core.env.MapPropertySource;
import org.springframework.core.env.PropertySource;

import java.util.HashMap;
import java.util.Map;

/**
 * 配置解密处理器
 * 在环境准备完成后、容器刷新前，遍历所有配置源，
 * 将形如 ENC(...) 的密文属性解密为明文，以最高优先级注入环境
 */
public class EncryptedEnvironmentPostProcessor implements EnvironmentPostProcessor {

    /** 密文前缀标识 */
    private static final String PREFIX = "ENC(";
    /** 密文后缀标识 */
    private static final String SUFFIX = ")";
    /** 解密后配置源名称，需保证唯一以便重复执行时覆盖 */
    private static final String DECRYPTED_SOURCE_NAME = "decryptedProperties";

    @Override
    public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
        // 解密密钥从环境变量读取，严禁硬编码在代码或配置文件中
        String secretKey = environment.getProperty("CONFIG_ENC_KEY");
        if (secretKey == null || secretKey.isEmpty()) {
            return;
        }

        Map<String, Object> decrypted = new HashMap<>();
        for (PropertySource<?> source : environment.getPropertySources()) {
            // 只能枚举出属性名的配置源才可遍历处理
            if (!(source instanceof EnumerablePropertySource)) {
                continue;
            }
            for (String name : ((EnumerablePropertySource<?>) source).getPropertyNames()) {
                Object value = source.getProperty(name);
                if (value instanceof String && isEncrypted((String) value)) {
                    decrypted.put(name, decryptValue(name, (String) value, secretKey));
                }
            }
        }

        if (!decrypted.isEmpty()) {
            // 固定名称保证幂等：重复触发时覆盖而非叠加
            environment.getPropertySources().addFirst(new MapPropertySource(DECRYPTED_SOURCE_NAME, decrypted));
        }
    }

    /**
     * 判断属性值是否为 ENC(...) 密文格式
     */
    private boolean isEncrypted(String value) {
        return value.startsWith(PREFIX) && value.endsWith(SUFFIX) && value.length() > PREFIX.length() + SUFFIX.length();
    }

    /**
     * 解密单个属性，失败时抛出异常并中断启动，避免密文未解密被当作密码使用
     */
    private String decryptValue(String name, String value, String secretKey) {
        String cipher = value.substring(PREFIX.length(), value.length() - SUFFIX.length());
        try {
            return AesUtils.decrypt(cipher, secretKey);
        } catch (Exception e) {
            throw new IllegalStateException("配置项解密失败：" + name + "，请检查密文与 CONFIG_ENC_KEY 密钥是否匹配", e);
        }
    }
}
```

由于解密结果以 `addFirst` 注入最高优先级，`@Value`、`@ConfigurationProperties`、`environment.getProperty(...)` 读到的都是明文，与 jasypt 方案效果一致。

### 4.3 注册 EnvironmentPostProcessor

在 `src/main/resources/META-INF/spring.factories` 中注册（Spring Boot 2.x 与 3.x 均适用）：

```properties
org.springframework.boot.env.EnvironmentPostProcessor=cn.wbnull.springbootdemo.config.EncryptedEnvironmentPostProcessor
```

> `EnvironmentPostProcessor` 的注册机制在 Spring Boot 3.x 中依然有效（迁移到 `.imports` 文件的是自动配置条目）。

### 4.4 MySQL 与 Redis 密码加密示例

密文同样用 `ENC(...)` 包裹，与 jasypt 方案写法完全一致，仅密文由 `AesUtils` 生成、无需任何 jasypt 配置：

```yaml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://127.0.0.1:3306/demo?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai&useSSL=false
    username: root
    # 明文 mysql@123456，AesUtils.encrypt 生成
    password: ENC(aVpXUGhGM2NYa0pQY2xKTFlXcGhUQT09)
  data:
    redis:
      host: 127.0.0.1
      port: 6379
      # 明文 redis@123456
      password: ENC(aDJGQk5XVkJWbVZxVDBWTlVrTkhWQT09)
```

启动时注入密钥（多环境管理与 jasypt 方案相同：各环境密钥独立，部署时注入各自密钥）：

```bash
# 环境变量
export CONFIG_ENC_KEY=ruD3jYw7XkW2vQ9mZpA0sA==
java -jar app.jar

# 或 JVM 参数
java -DCONFIG_ENC_KEY=ruD3jYw7XkW2vQ9mZpA0sA== -jar app.jar
```

### 4.5 jasypt 与自定义方案的差异

| 维度 | jasypt | 自定义 EnvironmentPostProcessor |
|------|--------|-------------------------------|
| 解密时机 | 属性被读取时（惰性） | 环境准备完成后（一次性） |
| 依赖 | 引入第三方 jar | 零依赖 |
| 密文格式 | `ENC(...)`，可自定义 | 完全自定义 |
| 算法 | PBE 系列（口令派生密钥） | 任意（示例为 AES-GCM） |
| 维护成本 | 低 | 需自管加解密与异常处理 |

### 4.6 为什么不用 PropertySourceLoader

`PropertySourceLoader` 同样通过 `META-INF/spring.factories` 注册、也能在加载时处理密文，但两者定位不同：前者是**文件解析层**扩展点（把物理文件变成 `PropertySource`，默认实现即 `YamlPropertySourceLoader`、`PropertiesPropertySourceLoader`，典型用途是引入新配置格式），后者是**环境后置处理层**扩展点（对已就位的 Environment 做统一增强）。

| 维度 | PropertySourceLoader | EnvironmentPostProcessor |
|------|---------------------|---------------------------|
| **生效范围** | 仅"经它加载的那类文件" | Environment 中所有可枚举配置源 |
| **处理配置中心属性** | ❌ Nacos/Apollo 属性不走本地 loader | ✅ 最终都在 Environment 中 |
| **典型用途** | 新配置格式、整文件级加解密 | 属性解密、脱敏、注入全局默认值 |
| **实现复杂度** | 高（后缀冲突、import 机制） | 低（遍历替换） |

解密场景选择 `EnvironmentPostProcessor` 的原因：

1. **覆盖面**：`ENC()` 属性可能来自 profile 文件、命令行覆盖、配置中心下发，loader 只能处理自己声明的后缀对应的文件
2. **后缀冲突不可控**：想"替换"默认 yml loader 行不通——多个 loader 声明同一后缀时由 `spring.factories` 加载顺序决定谁生效；只能自定义新后缀（如 `.enc-yml`），又因不匹配 `application.yml` 命名约定需 `spring.config.import` 显式引入，成本陡增
3. **时机语义**：loader 在解析时就地变换（密文不进入 Environment，隔离更彻底），EnvironmentPostProcessor 是事后覆盖——解密优先全来源覆盖与实现简单

### 4.7 执行顺序：为什么能读到 yml 属性

自定义处理器必须**晚于**加载配置文件的 `ConfigDataEnvironmentPostProcessor`（order = `Ordered.HIGHEST_PRECEDENCE + 10`）执行，否则执行时 yml 尚未加载，遍历不到任何 `ENC()` 属性。

`EnvironmentPostProcessor` 是纯 `@FunctionalInterface`（不继承 `Ordered`），所有实现由 `AnnotationAwareOrderComparator` 排序——**未实现 `Ordered` 且无 `@Order` 的实现默认排最后**，恰好晚于配置文件加载，因此本文示例无需任何额外设置。若确需指定顺序，使用 `@Order(Ordered.LOWEST_PRECEDENCE)`；若发现遍历不到 yml 属性，优先排查是否误设了过高优先级。

---

## 五、密钥管理策略

加密方案的安全性最终取决于密钥管理。**密钥与密文同处一地等于没加密。**

### 5.1 分环境策略

| 环境 | 密钥存放方式 | 说明 |
|------|-------------|------|
| 本地开发 | 环境变量或 IDEA 启动配置 | 密钥仅存本机，不进仓库 |
| 测试 | CI/CD 平台密钥管理（GitLab CI Variables、Jenkins Credentials） | 流水线注入环境变量，密钥不落盘 |
| 生产 | 部署平台环境变量（K8s Secret、云容器环境变量） | 配合最小权限：仅运维与发布系统可见 |
| 高安全要求 | KMS / Vault 动态获取 | 密钥轮转、访问审计、按需下发 |

### 5.2 Kubernetes 场景

```yaml
# Secret 存放密钥
apiVersion: v1
kind: Secret
metadata:
  name: app-crypto-secret
type: Opaque
stringData:
  jasypt-encryptor-password: my-jasypt-secret-key
```

```yaml
# Deployment 片段：环境变量引用 Secret
spec:
  containers:
    - name: app
      env:
        - name: JASYPT_ENCRYPTOR_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-crypto-secret
              key: jasypt-encryptor-password
```

---

## 六、常见问题

### 6.1 EncryptionOperationNotPossibleException

jasypt 读取属性时报 `DecryptionException: Unable to decrypt: ENC(xxx)`，按以下顺序排查：

| 检查项 | 排查方法 |
|--------|---------|
| **密钥不一致** | 生成密文时与运行时的密钥是否完全一致（注意末尾空白字符） |
| **算法不一致** | 生成密文的算法与 `jasypt.encryptor.algorithm` 是否一致 |
| **IV 生成器缺失** | 使用 AES 类算法但未配置 `RandomIvGenerator` |
| **密文损坏** | 复制丢字符、换行截断、Base64 中 `+` 被 URL 转义 |
| **JDK 算法限制** | `PBEWITHHMACSHA512ANDAES_256` 需要 JDK 8u161+ |
| **未传密钥** | 密钥为空时 jasypt 用默认值 `changeit`，解密必然失败 |

### 6.2 忘记传密钥时快速失败

jasypt 在首次读取 `ENC()` 属性时才解密，失败抛异常导致启动失败，宁启动失败也不要拿着密文去连数据库。可加启动自检：

```java
/**
 * 配置解密自检：密钥未配置时快速失败
 */
@Component
public class CryptoStartupCheck {

    @PostConstruct
    public void check() {
        String key = System.getenv("JASYPT_ENCRYPTOR_PASSWORD");
        if (key == null || key.isBlank()) {
            throw new IllegalStateException("未配置 JASYPT_ENCRYPTOR_PASSWORD，包含加密配置的应用无法启动");
        }
    }
}
```

<p align="center">
    <a href="https://github.com/dkbnull/hello-wiki/blob/main/10-SpringBoot/26-Spring%20Boot%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6%E6%95%8F%E6%84%9F%E4%BF%A1%E6%81%AF%E5%8A%A0%E5%AF%86%E8%A7%A3%E5%AF%86.md" target="_blank">
        <img src="https://img.shields.io/badge/GitHub-访问地址-blue?logo=github">
    </a>
    <a href="https://gitee.com/dkbnull/hello-wiki/blob/main/10-SpringBoot/26-Spring%20Boot%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6%E6%84%9F%E4%BF%A1%E6%81%AF%E5%8A%A0%E5%AF%86%E8%A7%A3%E5%AF%86.md" target="_blank">
        <img src="https://img.shields.io/badge/Gitee-访问地址-red?logo=gitee">
    </a>
    <a href="https://mp.weixin.qq.com/s/Ht57bwd0B09Os6-PZ9jelA" target="_blank">
       <img src="https://img.shields.io/badge/微信公众号-访问地址-brightgreen?logo=wechat">
    </a>
    <a href="https://juejin.cn/post/7686010356545732671" target="_blank">
       <img src="https://img.shields.io/badge/掘金-访问地址-blue?logo=juejin">
    </a>
    <a href="https://zhuanlan.zhihu.com/p/2083690285620642455" target="_blank">
       <img src="https://img.shields.io/badge/知乎-访问地址-blue?logo=zhihu">
    </a>
</p>
