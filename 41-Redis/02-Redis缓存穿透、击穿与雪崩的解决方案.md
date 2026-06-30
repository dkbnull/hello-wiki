# 1 缓存的基本使用流程

## 1.1 读取流程（Cache Aside Pattern）

Cache Aside Pattern 是最常用的缓存策略，核心思想是：先查缓存，缓存命中直接返回；缓存未命中则查数据库，然后将数据写入缓存，再返回

读取流程如下：

1. 先查询Redis缓存
2. 缓存命中，直接返回数据
3. 缓存未命中，查询数据库
4. 将数据库查询结果写入Redis缓存（设置过期时间）
5. 返回数据

~~~java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

/**
 * 商品服务类
 * 演示Cache Aside Pattern缓存读取流程
 */
@Service
public class ProductService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Autowired
    private ProductMapper productMapper;

    /**
     * 根据商品ID查询商品信息
     * 先查缓存，缓存未命中再查数据库
     *
     * @param productId 商品ID
     * @return 商品信息
     */
    public Product getProductById(Long productId) {
        //缓存Key
        String cacheKey = "product:info:" + productId;

        //1. 先查缓存
        Object cachedData = redisTemplate.opsForValue().get(cacheKey);
        if (cachedData != null) {
            //缓存命中，直接返回
            return (Product) cachedData;
        }

        //2. 缓存未命中，查询数据库
        Product product = productMapper.selectById(productId);
        if (product != null) {
            //3. 将数据写入缓存，设置30分钟过期时间
            redisTemplate.opsForValue().set(cacheKey, product, 30, TimeUnit.MINUTES);
        }

        return product;
    }
}
~~~

## 1.2 写入流程

写入数据时，应<font color='red'>**先更新数据库，再删除缓存**</font>，而不是更新缓存

为什么不更新缓存而是删除缓存：

- **并发安全问题**：如果同时有写操作和读操作，先更新缓存可能导致缓存与数据库数据不一致
- **性能问题**：有些缓存值是经过复杂计算得到的，如果更新操作频繁，每次都更新缓存会浪费性能
- **数据一致性**：删除缓存后，下次读取时会自动从数据库加载最新数据

~~~java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

/**
 * 商品服务类
 * 演示缓存写入流程：先更新数据库，再删除缓存
 */
@Service
public class ProductService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Autowired
    private ProductMapper productMapper;

    /**
     * 更新商品信息
     * 先更新数据库，再删除缓存
     *
     * @param product 商品信息
     */
    public void updateProduct(Product product) {
        //1. 先更新数据库
        productMapper.updateById(product);

        //2. 再删除缓存
        String cacheKey = "product:info:" + product.getId();
        redisTemplate.delete(cacheKey);
    }
}
~~~

<font color='red'>**如果删除缓存失败，可能导致后续请求读取到旧数据。生产环境中可以通过消息队列重试删除缓存，确保缓存与数据库的最终一致性**</font>

## 1.3 缓存使用流程图

用文字描述完整的缓存使用流程：

```
客户端请求
    |
    v
查询Redis缓存
    |
    +-- 缓存命中 --> 返回缓存数据 --> 结束
    |
    +-- 缓存未命中 --> 查询数据库
                            |
                            +-- 数据存在 --> 写入Redis缓存（设置TTL） --> 返回数据 --> 结束
                            |
                            +-- 数据不存在 --> 返回空 --> 结束（触发缓存穿透问题）
```

写入数据流程：

```
客户端写入请求
    |
    v
更新数据库
    |
    v
删除Redis缓存
    |
    v
返回结果 --> 结束
```

# 2 缓存穿透

## 2.1 什么是缓存穿透

缓存穿透是指客户端请求的数据<font color='red'>**在缓存和数据库中都不存在**</font>，每次请求都无法命中缓存，只能去查询数据库，导致数据库压力剧增

典型场景：

- 恶意攻击者故意请求不存在的ID（如负数ID、超长ID）
- 业务逻辑中查询已被删除的数据
- 非法参数请求（如不存在的用户名）

## 2.2 穿透的危害

- **数据库压力过大**：大量无效请求直接打到数据库，可能导致数据库连接池耗尽
- **正常请求受影响**：数据库被无效请求占满资源后，正常业务请求无法及时处理
- **系统雪崩风险**：数据库宕机后，整个系统不可用

## 2.3 解决方案一：缓存空值

核心思路：当查询数据库发现数据不存在时，将<font color='red'>**null值缓存到Redis中**</font>，并设置较短的过期时间（如5分钟），这样下次相同的请求就能命中缓存，不会打到数据库

~~~java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

/**
 * 商品服务类
 * 演示缓存空值方案解决缓存穿透
 */
@Service
public class ProductService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Autowired
    private ProductMapper productMapper;

    /**
     * 根据商品ID查询商品信息（缓存空值方案）
     * 当数据库中不存在该数据时，缓存null值，设置短过期时间
     *
     * @param productId 商品ID
     * @return 商品信息，不存在返回null
     */
    public Product getProductByIdWithCacheNull(Long productId) {
        //缓存Key
        String cacheKey = "product:info:" + productId;

        //1. 先查缓存
        Object cachedData = redisTemplate.opsForValue().get(cacheKey);
        if (cachedData != null) {
            //缓存命中
            //如果是空值标记对象，直接返回null
            if (cachedData instanceof String && "NULL_VALUE".equals(cachedData)) {
                return null;
            }
            return (Product) cachedData;
        }

        //2. 缓存未命中，查询数据库
        Product product = productMapper.selectById(productId);

        if (product != null) {
            //数据存在，写入缓存，设置30分钟过期时间
            redisTemplate.opsForValue().set(cacheKey, product, 30, TimeUnit.MINUTES);
        } else {
            //数据不存在，缓存空值标记，设置5分钟过期时间
            //避免恶意请求反复穿透到数据库
            redisTemplate.opsForValue().set(cacheKey, "NULL_VALUE", 5, TimeUnit.MINUTES);
        }

        return product;
    }
}
~~~

<font color='red'>**空值的过期时间不宜过长，建议设置在5分钟以内。如果设置过长，当数据库新增了该数据后，缓存中的空值还未过期，会导致短时间内无法读取到新数据**</font>

## 2.4 解决方案二：布隆过滤器

布隆过滤器（Bloom Filter）是一种空间效率很高的概率型数据结构，用于判断<font color='red'>**一个元素是否可能存在于集合中**</font>

核心原理：

- 布隆过滤器使用多个哈希函数将元素映射到位数组中
- 判断元素是否存在时，如果所有哈希位都为1，则元素**可能存在**（存在误判率）
- 如果有任何一个哈希位为0，则元素**一定不存在**
- <font color='red'>**布隆过滤器只能保证"一定不存在"，不能保证"一定存在"**</font>

~~~xml
<!-- 引入Guava依赖 -->
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>32.1.3-jre</version>
</dependency>
~~~

~~~java
import com.google.common.hash.BloomFilter;
import com.google.common.hash.Funnels;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;
import java.util.List;
import java.util.concurrent.TimeUnit;

/**
 * 商品服务类
 * 演示布隆过滤器方案解决缓存穿透
 */
@Service
public class ProductService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Autowired
    private ProductMapper productMapper;

    /**
     * 布隆过滤器
     * 预期插入数据量：10000
     * 误判率：0.01（1%）
     */
    private BloomFilter<Long> bloomFilter;

    /**
     * 初始化布隆过滤器
     * 在服务启动时，将数据库中所有商品ID加载到布隆过滤器中
     */
    @PostConstruct
    public void initBloomFilter() {
        //创建布隆过滤器，预期数据量10000，误判率1%
        bloomFilter = BloomFilter.create(
                Funnels.longFunnel(),
                10000,
                0.01
        );

        //从数据库中加载所有商品ID
        List<Long> allProductIds = productMapper.selectAllIds();
        for (Long productId : allProductIds) {
            bloomFilter.put(productId);
        }
    }

    /**
     * 根据商品ID查询商品信息（布隆过滤器方案）
     * 先通过布隆过滤器判断商品ID是否存在，不存在直接返回
     *
     * @param productId 商品ID
     * @return 商品信息，不存在返回null
     */
    public Product getProductByIdWithBloomFilter(Long productId) {
        //1. 先通过布隆过滤器判断
        //布隆过滤器判断不存在，则一定不存在，直接返回
        if (!bloomFilter.mightContain(productId)) {
            return null;
        }

        //2. 布隆过滤器判断可能存在，查缓存
        String cacheKey = "product:info:" + productId;
        Object cachedData = redisTemplate.opsForValue().get(cacheKey);
        if (cachedData != null) {
            return (Product) cachedData;
        }

        //3. 缓存未命中，查询数据库
        Product product = productMapper.selectById(productId);
        if (product != null) {
            //数据存在，写入缓存
            redisTemplate.opsForValue().set(cacheKey, product, 30, TimeUnit.MINUTES);
        }

        return product;
    }
}
~~~

<font color='red'>**布隆过滤器无法删除元素，当数据库中删除数据时，布隆过滤器中对应的元素仍然存在。因此布隆过滤器适合数据变动不频繁的场景，或者需要定期重建布隆过滤器**</font>

## 2.5 解决方案三：请求参数校验

在接口层对请求参数进行校验，拦截明显非法的请求，从源头减少无效请求

~~~java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.HashMap;
import java.util.Map;
import java.util.regex.Pattern;

/**
 * 商品控制器
 * 演示请求参数校验方案，拦截非法请求
 */
@RestController
@RequestMapping("product")
public class ProductController {

    @Autowired
    private ProductService productService;

    /**
     * 根据商品ID查询商品信息
     * 在接口层进行参数校验，拦截非法请求
     *
     * @param productId 商品ID
     * @return 商品信息
     */
    @GetMapping("info/{productId}")
    public Map<String, Object> getProductInfo(@PathVariable String productId) {
        Map<String, Object> result = new HashMap<>();

        //参数校验：ID必须为正整数
        if (!Pattern.matches("^[1-9]\\d*$", productId)) {
            result.put("code", 400);
            result.put("message", "商品ID格式不正确");
            return result;
        }

        //参数校验：ID不能超过合理范围
        long id = Long.parseLong(productId);
        if (id > 100000000L) {
            result.put("code", 400);
            result.put("message", "商品ID超出合理范围");
            return result;
        }

        //校验通过，查询商品
        Product product = productService.getProductById(id);
        if (product != null) {
            result.put("code", 200);
            result.put("message", "查询成功");
            result.put("data", product);
        } else {
            result.put("code", 404);
            result.put("message", "商品不存在");
        }

        return result;
    }
}
~~~

## 2.6 三种方案的对比

| 对比项 | 缓存空值 | 布隆过滤器 | 请求参数校验 |
|--------|---------|-----------|-------------|
| 实现难度 | 低 | 中 | 低 |
| 内存占用 | 较高（缓存大量空值Key） | 很低（位数组） | 无 |
| 可靠性 | 一般（依赖过期时间） | 高（概率型判断） | 一般（只能拦截明显非法请求） |
| 适用场景 | 数据量不大，请求种类少 | 数据量大，请求种类多 | 所有场景，作为第一道防线 |
| 维护成本 | 低（自动过期） | 中（需定期重建） | 低 |
| 缺点 | 占用Redis内存，短时间数据不一致 | 存在误判率，不支持删除 | 只能拦截格式非法的请求 |

<font color='red'>**实际项目中，通常组合使用多种方案：请求参数校验作为第一道防线 + 布隆过滤器拦截不存在的ID + 缓存空值兜底**</font>

# 3 缓存击穿

## 3.1 什么是缓存击穿

缓存击穿是指<font color='red'>**某个热点Key（Hot Key）在过期的瞬间，大量并发请求同时到达，由于缓存刚过期，这些请求都会直接打到数据库**</font>，导致数据库瞬间压力剧增

典型场景：

- 秒杀活动中，某个商品详情页的缓存刚好过期
- 热点新闻的缓存过期，大量用户同时访问
- 热门商品的库存数据缓存过期

## 3.2 击穿与穿透的区别

| 对比项 | 缓存穿透 | 缓存击穿 |
|--------|---------|---------|
| 定义 | 查询不存在的数据，缓存和数据库都没有 | 热点Key过期瞬间，大量并发请求打到数据库 |
| 数据是否存在 | 数据库中不存在 | 数据库中存在 |
| 请求特征 | 持续请求不存在的数据 | 大量并发请求同一个热点Key |
| 影响范围 | 可能影响多个不同的Key | 只影响单个热点Key |
| 解决方向 | 拦截不存在的请求 | 保护热点Key的缓存可用性 |

## 3.3 解决方案一：互斥锁

核心思路：当缓存未命中时，<font color='red'>**只允许一个线程去查询数据库并重建缓存**</font>，其他线程等待，直到缓存重建完成后直接读取缓存

~~~xml
<!-- 引入Redisson依赖 -->
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.24.3</version>
</dependency>
~~~

~~~java
import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

/**
 * 商品服务类
 * 演示互斥锁方案解决缓存击穿
 * 使用Redisson分布式锁保证只有一个线程重建缓存
 */
@Service
public class ProductService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Autowired
    private RedissonClient redissonClient;

    @Autowired
    private ProductMapper productMapper;

    /**
     * 根据商品ID查询商品信息（互斥锁方案）
     * 缓存过期时，只允许一个线程查询数据库并重建缓存
     *
     * @param productId 商品ID
     * @return 商品信息
     */
    public Product getProductByIdWithMutex(Long productId) {
        //缓存Key
        String cacheKey = "product:info:" + productId;
        //锁Key
        String lockKey = "lock:product:" + productId;

        //1. 先查缓存
        Object cachedData = redisTemplate.opsForValue().get(cacheKey);
        if (cachedData != null) {
            return (Product) cachedData;
        }

        //2. 缓存未命中，获取分布式锁
        RLock lock = redissonClient.getLock(lockKey);
        try {
            //尝试加锁，等待时间5秒，锁持有时间30秒
            boolean acquired = lock.tryLock(5, 30, TimeUnit.SECONDS);
            if (acquired) {
                try {
                    //再次查询缓存（双重检查）
                    //可能其他线程已经重建了缓存
                    cachedData = redisTemplate.opsForValue().get(cacheKey);
                    if (cachedData != null) {
                        return (Product) cachedData;
                    }

                    //查询数据库
                    Product product = productMapper.selectById(productId);
                    if (product != null) {
                        //写入缓存，设置30分钟过期时间
                        redisTemplate.opsForValue().set(cacheKey, product, 30, TimeUnit.MINUTES);
                    }
                    return product;
                } finally {
                    //释放锁
                    lock.unlock();
                }
            } else {
                //获取锁失败，等待一段时间后重试
                Thread.sleep(100);
                return getProductByIdWithMutex(productId);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return null;
        }
    }
}
~~~

<font color='red'>**互斥锁方案能保证数据一致性，但在锁等待期间，其他线程会被阻塞，可能导致请求响应时间变长。适用于对数据一致性要求较高的场景**</font>

## 3.4 解决方案二：逻辑过期

核心思路：在缓存中<font color='red'>**不设置Redis的过期时间，而是在缓存数据中存储一个逻辑过期时间字段**</font>。当读取缓存时发现逻辑已过期，不立即删除缓存，而是异步线程去更新缓存，当前请求先返回旧数据

~~~java
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.io.Serializable;

/**
 * 缓存数据包装类
 * 包含实际数据和逻辑过期时间
 */
@Data
@AllArgsConstructor
@NoArgsConstructor
public class CacheData<T> implements Serializable {

    private static final long serialVersionUID = 1L;

    /**
     * 实际数据
     */
    private T data;

    /**
     * 逻辑过期时间（时间戳，毫秒）
     */
    private Long expireTime;

    /**
     * 判断是否逻辑过期
     *
     * @return true已过期 false未过期
     */
    public boolean isExpired() {
        return System.currentTimeMillis() > expireTime;
    }
}
~~~

~~~java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

/**
 * 商品服务类
 * 演示逻辑过期方案解决缓存击穿
 * 缓存中存储逻辑过期时间，过期后异步更新，当前请求返回旧数据
 */
@Service
public class ProductService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Autowired
    private ProductMapper productMapper;

    /**
     * 线程池，用于异步更新缓存
     */
    private static final ExecutorService REBUILD_EXECUTOR = Executors.newFixedThreadPool(5);

    /**
     * 根据商品ID查询商品信息（逻辑过期方案）
     * 发现逻辑过期后，异步线程更新缓存，当前请求先返回旧数据
     *
     * @param productId 商品ID
     * @return 商品信息
     */
    public Product getProductByIdWithLogicalExpire(Long productId) {
        //缓存Key
        String cacheKey = "product:info:" + productId;

        //1. 查询缓存
        Object cachedData = redisTemplate.opsForValue().get(cacheKey);
        if (cachedData == null) {
            //缓存不存在，说明是第一次查询，需要预热缓存
            return null;
        }

        //2. 缓存存在，检查是否逻辑过期
        CacheData<Product> cacheData = (CacheData<Product>) cachedData;
        if (!cacheData.isExpired()) {
            //未过期，直接返回
            return cacheData.getData();
        }

        //3. 逻辑已过期，异步重建缓存
        REBUILD_EXECUTOR.submit(() -> {
            rebuildCache(cacheKey, productId);
        });

        //4. 先返回旧数据，保证高可用
        return cacheData.getData();
    }

    /**
     * 异步重建缓存
     *
     * @param cacheKey  缓存Key
     * @param productId 商品ID
     */
    private void rebuildCache(String cacheKey, Long productId) {
        try {
            //再次检查缓存，避免重复重建
            Object cachedData = redisTemplate.opsForValue().get(cacheKey);
            if (cachedData != null) {
                CacheData<Product> cacheData = (CacheData<Product>) cachedData;
                if (!cacheData.isExpired()) {
                    //已经被其他线程更新，无需重复操作
                    return;
                }
            }

            //查询数据库
            Product product = productMapper.selectById(productId);
            if (product != null) {
                //创建缓存数据，设置逻辑过期时间为30分钟后
                CacheData<Product> newCacheData = new CacheData<>(
                        product,
                        System.currentTimeMillis() + 30 * 60 * 1000
                );
                //写入缓存，不设置Redis过期时间
                redisTemplate.opsForValue().set(cacheKey, newCacheData);
            }
        } catch (Exception e) {
            //异常处理，记录日志
            System.err.println("异步重建缓存失败: " + e.getMessage());
        }
    }

    /**
     * 预热缓存（系统启动时或手动调用）
     * 将热点数据提前加载到缓存中，设置逻辑过期时间
     *
     * @param productId 商品ID
     */
    public void warmUpCache(Long productId) {
        Product product = productMapper.selectById(productId);
        if (product != null) {
            String cacheKey = "product:info:" + productId;
            CacheData<Product> cacheData = new CacheData<>(
                    product,
                    System.currentTimeMillis() + 30 * 60 * 1000
            );
            redisTemplate.opsForValue().set(cacheKey, cacheData);
        }
    }
}
~~~

<font color='red'>**逻辑过期方案能保证高可用，但短时间内会返回旧数据，适用于对数据一致性要求不高、但对可用性要求高的场景（如商品详情页浏览量）**</font>

## 3.5 解决方案三：热点数据永不过期

核心思路：对于热点数据，<font color='red'>**设置较长的过期时间（甚至永不过期），并通过后台定时任务定期刷新缓存**</font>，确保缓存中始终有数据

~~~java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.concurrent.TimeUnit;

/**
 * 热点缓存刷新服务
 * 通过定时任务定期刷新热点数据的缓存，避免缓存过期
 */
@Service
public class HotCacheRefreshService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Autowired
    private ProductMapper productMapper;

    /**
     * 热点商品ID列表（实际项目中可从配置中心或数据库中读取）
     */
    private static final List<Long> HOT_PRODUCT_IDS = List.of(1L, 2L, 3L, 5L, 8L, 13L);

    /**
     * 定时刷新热点缓存
     * 每10分钟执行一次，提前刷新热点数据，避免缓存过期
     */
    @Scheduled(fixedRate = 10 * 60 * 1000)
    public void refreshHotCache() {
        for (Long productId : HOT_PRODUCT_IDS) {
            try {
                //查询数据库获取最新数据
                Product product = productMapper.selectById(productId);
                if (product != null) {
                    String cacheKey = "product:info:" + productId;
                    //设置较长的过期时间：1小时
                    redisTemplate.opsForValue().set(cacheKey, product, 1, TimeUnit.HOURS);
                    System.out.println("热点缓存刷新成功: " + productId);
                }
            } catch (Exception e) {
                System.err.println("热点缓存刷新失败: " + productId + ", " + e.getMessage());
            }
        }
    }
}
~~~

<font color='red'>**热点数据永不过期方案简单有效，但需要额外的定时任务维护，且热点数据的识别需要结合业务监控。适用于热点数据明确且数量有限的场景**</font>

## 3.6 三种方案的对比

| 对比项 | 互斥锁 | 逻辑过期 | 热点数据永不过期 |
|--------|--------|---------|----------------|
| 实现难度 | 中 | 较高 | 低 |
| 一致性 | 强一致性 | 最终一致性（短时间返回旧数据） | 最终一致性 |
| 可用性 | 一般（存在阻塞等待） | 高（始终返回数据） | 高 |
| 性能影响 | 锁等待期间性能下降 | 无阻塞，性能好 | 无影响 |
| 额外资源 | Redis分布式锁 | 线程池 + 预热 | 定时任务 |
| 适用场景 | 对数据一致性要求高 | 对可用性要求高 | 热点数据明确且数量少 |
| 缺点 | 可能导致请求阻塞 | 短时间内返回旧数据 | 需要维护热点数据列表 |

# 4 缓存雪崩

## 4.1 什么是缓存雪崩

缓存雪崩是指<font color='red'>**大量缓存Key在同一时间过期，或者Redis服务宕机**</font>，导致大量请求同时打到数据库，数据库压力剧增甚至宕机，引发整个系统的级联故障

缓存雪崩的两种场景：

- **大量Key同时过期**：批量设置的缓存使用了相同的过期时间，到期时大量请求同时打到数据库
- **Redis宕机**：Redis服务不可用，所有请求都直接打到数据库

## 4.2 解决方案一：过期时间加随机值

核心思路：在设置缓存过期时间时，<font color='red'>**在基础过期时间上加上一个随机偏移量**</font>，使不同Key的过期时间分散开来，避免大量Key同时过期

~~~java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.util.Random;
import java.util.concurrent.TimeUnit;

/**
 * 商品服务类
 * 演示过期时间加随机值方案，避免大量Key同时过期
 */
@Service
public class ProductService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Autowired
    private ProductMapper productMapper;

    /**
     * 随机数生成器
     */
    private static final Random RANDOM = new Random();

    /**
     * 基础过期时间：30分钟
     */
    private static final long BASE_EXPIRE_MINUTES = 30;

    /**
     * 随机偏移量最大值：5分钟（300秒）
     */
    private static final int RANDOM_EXPIRE_SECONDS = 300;

    /**
     * 根据商品ID查询商品信息（过期时间加随机值方案）
     * 在基础过期时间上加上随机偏移量，避免大量Key同时过期
     *
     * @param productId 商品ID
     * @return 商品信息
     */
    public Product getProductByIdWithRandomExpire(Long productId) {
        //缓存Key
        String cacheKey = "product:info:" + productId;

        //先查缓存
        Object cachedData = redisTemplate.opsForValue().get(cacheKey);
        if (cachedData != null) {
            return (Product) cachedData;
        }

        //查询数据库
        Product product = productMapper.selectById(productId);
        if (product != null) {
            //过期时间 = 基础时间 + 随机偏移量
            long expireSeconds = BASE_EXPIRE_MINUTES * 60 + RANDOM.nextInt(RANDOM_EXPIRE_SECONDS);
            redisTemplate.opsForValue().set(cacheKey, product, expireSeconds, TimeUnit.SECONDS);
        }

        return product;
    }
}
~~~

<font color='red'>**随机偏移量不宜过大，一般建议为基础过期时间的10%~20%。偏移量过大会导致部分缓存过早过期，降低缓存命中率**</font>

## 4.3 解决方案二：Redis高可用

通过搭建Redis高可用架构，避免Redis单点故障导致的缓存雪崩

**哨兵模式（Sentinel）**：

- 哨兵节点监控主节点的健康状态
- 主节点故障时自动进行故障转移，选举新的主节点
- 至少需要3个哨兵节点，保证过半数同意才进行故障转移
- 适用于中小规模场景

**集群模式（Cluster）**：

- 数据分片存储在多个主节点上，每个主节点至少有一个从节点
- 至少6个节点（3主3从），支持自动故障转移和数据分片
- 适用于大规模、高并发场景

~~~yml
# Spring Boot配置Redis哨兵模式
spring:
  redis:
    sentinel:
      master: mymaster
      nodes:
        - 192.168.1.101:26379
        - 192.168.1.102:26379
        - 192.168.1.103:26379
~~~

~~~yml
# Spring Boot配置Redis集群模式
spring:
  redis:
    cluster:
      nodes:
        - 192.168.1.101:6379
        - 192.168.1.102:6379
        - 192.168.1.103:6379
        - 192.168.1.104:6379
        - 192.168.1.105:6379
        - 192.168.1.106:6379
~~~

## 4.4 解决方案三：多级缓存

核心思路：在Redis缓存之前增加一层本地缓存（如Caffeine），形成<font color='red'>**本地缓存 + Redis缓存 + 数据库**</font>的多级缓存架构。当Redis不可用时，本地缓存仍能提供部分数据的查询能力

~~~xml
<!-- 引入Caffeine依赖 -->
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
    <version>3.1.8</version>
</dependency>
~~~

~~~java
import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

/**
 * 商品服务类
 * 演示多级缓存方案：本地缓存（Caffeine） + Redis缓存 + 数据库
 * 当Redis不可用时，本地缓存仍能提供部分数据的查询能力
 */
@Service
public class ProductService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Autowired
    private ProductMapper productMapper;

    /**
     * 本地缓存（Caffeine）
     * 最大容量：1000个
     * 过期时间：5分钟
     */
    private final Cache<Long, Product> localCache = Caffeine.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(5, TimeUnit.MINUTES)
            .build();

    /**
     * 根据商品ID查询商品信息（多级缓存方案）
     * 查询顺序：本地缓存 -> Redis缓存 -> 数据库
     *
     * @param productId 商品ID
     * @return 商品信息
     */
    public Product getProductByIdWithMultiLevelCache(Long productId) {
        //1. 先查本地缓存（Caffeine）
        Product localProduct = localCache.getIfPresent(productId);
        if (localProduct != null) {
            return localProduct;
        }

        //2. 查Redis缓存
        String cacheKey = "product:info:" + productId;
        try {
            Object redisData = redisTemplate.opsForValue().get(cacheKey);
            if (redisData != null) {
                Product product = (Product) redisData;
                //回填本地缓存
                localCache.put(productId, product);
                return product;
            }
        } catch (Exception e) {
            //Redis异常，降级到直接查数据库
            System.err.println("Redis查询异常，降级到数据库: " + e.getMessage());
        }

        //3. 查询数据库
        Product product = productMapper.selectById(productId);
        if (product != null) {
            //回填Redis缓存
            try {
                long expireSeconds = 30 * 60 + (long) (Math.random() * 300);
                redisTemplate.opsForValue().set(cacheKey, product, expireSeconds, TimeUnit.SECONDS);
            } catch (Exception e) {
                System.err.println("Redis写入异常: " + e.getMessage());
            }
            //回填本地缓存
            localCache.put(productId, product);
        }

        return product;
    }
}
~~~

<font color='red'>**多级缓存方案能有效降低Redis的压力，在Redis不可用时本地缓存仍能提供数据。但需要注意本地缓存与Redis缓存之间的数据一致性问题，建议设置较短的本地缓存过期时间**</font>

## 4.5 解决方案四：熔断降级

当数据库压力过大或Redis不可用时，通过熔断降级机制<font color='red'>**直接返回默认值或错误提示，避免数据库被压垮**</font>

使用Sentinel实现熔断降级：

~~~xml
<!-- 引入Sentinel依赖 -->
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-spring-boot-starter</artifactId>
    <version>1.8.6</version>
</dependency>
~~~

~~~java
import com.alibaba.csp.sentinel.annotation.SentinelResource;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

/**
 * 商品服务类
 * 演示Sentinel熔断降级方案
 * 当查询数据库异常率过高时，触发熔断，直接返回降级数据
 */
@Service
public class ProductService {

    @Autowired
    private ProductMapper productMapper;

    /**
     * 根据商品ID查询商品信息（熔断降级方案）
     * 当异常率超过阈值时触发熔断，调用降级方法返回默认数据
     *
     * @param productId 商品ID
     * @return 商品信息
     */
    @SentinelResource(value = "getProductById",
            fallback = "getProductByIdFallback")
    public Product getProductByIdWithSentinel(Long productId) {
        return productMapper.selectById(productId);
    }

    /**
     * 降级方法
     * 当熔断触发时，返回默认的降级数据
     *
     * @param productId 商品ID
     * @return 默认商品信息
     */
    public Product getProductByIdFallback(Long productId) {
        //返回默认的降级数据，避免数据库被压垮
        Product fallbackProduct = new Product();
        fallbackProduct.setId(productId);
        fallbackProduct.setName("系统繁忙，请稍后重试");
        return fallbackProduct;
    }
}
~~~

## 4.6 解决方案五：限流

通过限流算法控制<font color='red'>**单位时间内允许通过的请求数量**</font>，在数据库承载能力范围内提供服务

常见限流算法：

- **令牌桶算法（Token Bucket）**：以固定速率向桶中放入令牌，请求需要从桶中获取令牌才能通过。桶满时多余的令牌被丢弃，桶空时请求被拒绝。允许突发流量
- **漏桶算法（Leaky Bucket）**：请求进入桶中，以固定速率从桶中流出处理。桶满时多余的请求被拒绝。流量平滑输出，不允许突发流量
- **滑动窗口算法**：在固定时间窗口内限制请求数量，窗口滑动更新计数

使用Sentinel配置限流规则：

~~~java
import com.alibaba.csp.sentinel.slots.block.flow.FlowRule;
import com.alibaba.csp.sentinel.slots.block.flow.FlowRuleManager;

import java.util.ArrayList;
import java.util.List;

/**
 * Sentinel限流规则配置类
 * 在系统启动时加载限流规则
 */
public class SentinelRuleConfig {

    /**
     * 初始化限流规则
     * 限制getProductById接口每秒最多通过100个请求
     */
    public static void initFlowRules() {
        List<FlowRule> rules = new ArrayList<>();

        FlowRule rule = new FlowRule();
        rule.setResource("getProductById");
        //限流阈值：每秒100个请求
        rule.setGrade(com.alibaba.csp.sentinel.slots.block.flow.FlowRuleConstant.FLOW_GRADE_QPS);
        rule.setCount(100);
        //超过阈值后直接拒绝
        rule.setLimitApp("default");
        rules.add(rule);

        FlowRuleManager.loadRules(rules);
    }
}
~~~

# 5 总结：三大问题的对比

## 5.1 缓存穿透 vs 击穿 vs 雪崩对比表格

| 对比项 | 缓存穿透 | 缓存击穿 | 缓存雪崩 |
|--------|---------|---------|---------|
| 定义 | 查询不存在的数据，缓存和数据库都没有 | 热点Key过期瞬间，大量并发请求打到数据库 | 大量Key同时过期或Redis宕机，大量请求打到数据库 |
| 根本原因 | 数据不存在 | 单个热点Key过期 | 批量Key过期或Redis不可用 |
| 影响范围 | 多个不同的Key | 单个热点Key | 大量Key |
| 解决方案 | 缓存空值、布隆过滤器、参数校验 | 互斥锁、逻辑过期、永不过期 | 随机过期时间、高可用、多级缓存、熔断降级、限流 |

## 5.2 实际项目中的最佳实践组合

在实际项目中，通常需要组合多种方案来应对缓存问题：

**基础防护（所有项目必备）**：

- 缓存过期时间加随机值，防止缓存雪崩
- 接口层参数校验，拦截明显非法的请求
- 缓存空值，防止缓存穿透

**进阶防护（中高并发项目）**：

- 布隆过滤器，拦截不存在的数据请求
- 互斥锁或逻辑过期，保护热点Key
- Redis哨兵或集群模式，保证Redis高可用

**高级防护（高并发大流量项目）**：

- 多级缓存（Caffeine + Redis），降低Redis压力
- 熔断降级（Sentinel），保护数据库不被压垮
- 限流（Sentinel），控制请求总量
- 热点数据自动识别与预热

至此，Redis缓存穿透、击穿与雪崩的解决方案讲解完毕。

---

<p align="center">
  <a href="https://github.com/dkbnull/hello-wiki/blob/main/41-Redis/02-Redis%E7%BC%93%E5%AD%98%E7%A9%BF%E9%80%8F%E3%80%81%E5%87%BB%E7%A9%BF%E4%B8%8E%E9%9B%AA%E5%B4%A9%E7%9A%84%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88.md" target="_blank">
     <img src="https://img.shields.io/badge/GitHub-访问地址-blue?logo=github">
  </a>
  <a href="https://gitee.com/dkbnull/hello-wiki/blob/main/41-Redis/02-Redis%E7%BC%93%E5%AD%98%E7%A9%BF%E9%80%8F%E3%80%81%E5%87%BB%E7%A9%BF%E4%B8%8E%E9%9B%AA%E5%B4%A9%E7%9A%84%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88.md" target="_blank">
     <img src="https://img.shields.io/badge/Gitee-访问地址-red?logo=gitee">
  </a>
    <a href="https://mp.weixin.qq.com/s/Bnnj8L14QdOhT9SKJG9u8w" target="_blank">
       <img src="https://img.shields.io/badge/微信公众号-访问地址-brightgreen?logo=wechat">
    </a>
    <a href="https://juejin.cn/post/7657063330017148937" target="_blank">
       <img src="https://img.shields.io/badge/掘金-访问地址-blue?logo=juejin">
    </a>
    <a href="https://zhuanlan.zhihu.com/p/2055413842428143177" target="_blank">
       <img src="https://img.shields.io/badge/知乎-访问地址-blue?logo=zhihu">
    </a>
</p>
