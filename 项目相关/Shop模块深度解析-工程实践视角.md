# Shop 模块深度解析 - 工程实践视角

> 从高并发后端工程师的角度，彻底拆解黑马点评中的 Shop 模块
> 适用于：中高级 Java 后端面试、实际项目落地参考

---

## 一、Shop 模块的【业务定位】- 为什么它是缓存优化的第一战场

### 1.1 业务特征分析

在点评类 APP 中，**商铺详情是最高频的读操作**，原因：

```
用户行为路径：
首页浏览 → 点击商铺卡片 → 【查看商铺详情】 → 查看评价 → 下单
          ↑                    ↑
      列表页每次显示10+个      每个商铺平均被点击3-5次
```

**真实数据画像**（参考同类项目）：
- 读写比：**300:1** （300次查询 vs 1次更新）
- 热点商铺：前 5% 的商铺占据 **70%+ 的流量**
- 响应时间要求：< 100ms（超过会明显影响体验）

### 1.2 为什么 Shop 是第一个要优化的模块

**对比其他模块**：

| 模块 | 访问频率 | 数据变化频率 | 缓存优先级 |
|------|---------|------------|----------|
| Shop（商铺详情） | ⭐⭐⭐⭐⭐ | 低（天级） | 🔥 最高 |
| User（用户信息） | ⭐⭐⭐ | 中（小时级） | 高 |
| Blog（笔记） | ⭐⭐⭐⭐ | 高（分钟级） | 中 |
| Voucher（优惠券） | ⭐⭐⭐⭐ | 中（小时级） | 高 |

**Shop 模块的独特优势**：
1. **数据稳定性极高**：商铺地址、营业时间、图片等基本不变
2. **读写严重不对称**：天然适合缓存
3. **无强一致性要求**：延迟几秒钟同步数据用户无感知
4. **热点明显**：网红店、头部商家容易成为热点

### 1.3 不优化会怎样？

**真实场景推演**：
```
假设：每天 10 万用户访问，平均每人查看 20 个商铺
计算：200 万次 MySQL 查询/天 ≈ 23 次/秒

峰值时段（晚 7-9 点）流量翻 5 倍
→ 115 次/秒 MySQL 查询
→ 加上其他业务，数据库压力直接打满
→ 慢查询堆积 → 连接池耗尽 → 服务雪崩
```

---

## 二、【核心查询流程】- 三种典型场景全拆解

### 2.1 场景一：无缓存时的查询流程（初始状态）

```
前端：GET /shop/1

Controller:
  ↓ 调用 shopService.queryByID(1)

ShopServiceImpl:
  ↓ 查询 Redis: cache:shop:1
  ✗ 未命中（第一次访问）
  
  ↓ 查询 MySQL: SELECT * FROM tb_shop WHERE id = 1
  ✓ 返回 Shop 对象
  
  ↓ 写入 Redis: 
     Key = "cache:shop:1"
     Value = JSON(Shop)
     TTL = 30 分钟
  
  ✓ 返回给前端
```

**关键点**：
- **第一次查询必走数据库**（冷启动）
- **查询后立即回填缓存**（Cache-Aside 模式）
- **设置合理过期时间**（防止数据无限堆积）

### 2.2 场景二：有缓存时的查询流程（稳态）

```
前端：GET /shop/1

Controller:
  ↓ 调用 shopService.queryByID(1)

ShopServiceImpl:
  ↓ 查询 Redis: cache:shop:1
  ✓ 命中！直接返回
  
响应时间：< 5ms（纯内存操作）
数据库压力：0
```

**性能对比**：
```
无缓存：MySQL 查询 20-50ms
有缓存：Redis 查询 2-5ms
性能提升：10-25 倍
```

### 2.3 场景三：缓存失效时的查询流程（关键场景）

**2.3.1 正常失效（TTL 过期）**
```
前端：大量请求 GET /shop/1（缓存刚过期）

第一个请求：
  ↓ Redis 未命中
  ↓ 尝试获取锁：SETNX lock:shop:1 1 EX 10
  ✓ 获取成功
  ↓ 查询 MySQL
  ↓ 回填缓存
  ↓ 释放锁
  ✓ 返回数据

第 2-100 个请求（在第一个请求查询期间到达）：
  ↓ Redis 未命中
  ↓ 尝试获取锁
  ✗ 获取失败（锁被占用）
  ↓ 休眠 10ms
  ↓ 递归重试 queryByID(id)
  ↓ 此时缓存已被第一个请求填充
  ✓ 命中缓存，返回数据
```

**关键设计思想**：
- **互斥锁保证只有一个线程查库**（防击穿）
- **其他线程等待并重试**（牺牲少量延迟换稳定性）
- **锁的 TTL 必须设置**（防止死锁）

**2.3.2 热点数据失效（逻辑过期方案 - 已注释但值得关注）**

项目中有完整的逻辑过期实现（虽然当前被注释），核心流程：

```java
// CacheService.queryWithLogicalTTL 方法
查询 Redis:
  ↓ 数据存在（热点数据永不过期）
  ↓ 检查逻辑过期时间
  
如果未过期：
  ✓ 直接返回

如果已过期：
  ↓ 返回旧数据（重点！）
  ↓ 尝试获取锁
    成功：
      ↓ 提交异步任务重建缓存（线程池）
      ↓ 释放锁
    失败：
      ↓ 直接返回旧数据
```

**两种方案对比**：

| 维度 | 互斥锁（当前使用） | 逻辑过期（备选方案） |
|------|----------------|------------------|
| 实现复杂度 | ⭐⭐ 简单 | ⭐⭐⭐⭐ 复杂 |
| 一致性 | 强一致 | 最终一致 |
| 可用性 | 锁等待期间延迟增加 | 始终秒级响应 |
| 数据新鲜度 | 100% 最新 | 可能返回过期数据 |
| 适用场景 | 普通商铺 | 网红店、秒杀商品 |

---

## 三、【缓存设计详解】- 这些细节决定成败

### 3.1 Redis 中存了什么？Key 如何设计？

**当前实现**（第 55 行核心调用）：
```java
Shop result = cacheClient.queryPassThrough(
    CACHE_SHOP_KEY,    // "cache:shop:"
    id,                // 商铺 ID
    Shop.class,
    this::getById,
    CACHE_SHOP_TTL,    // 30 分钟
    TimeUnit.MINUTES
);
```

**Redis 存储结构**：
```redis
Key:   "cache:shop:1"
Value: JSON 字符串
{
  "id": 1,
  "name": "星巴克(中山公园店)",
  "typeId": 1,
  "images": "http://...",
  "area": "长宁区",
  "address": "长宁路...",
  "avgPrice": 50,
  ...
}
TTL:   1800 秒（30分钟）
```

**Key 设计原则**：
```
前缀 + 业务标识 + ID

cache:shop:1
  ↓      ↓    ↓
  |      |    └─ 具体商铺ID（确保唯一性）
  |      └────── 业务模块（便于批量操作）
  └─────────── 用途标识（区分不同类型的key）
```

**为什么这样设计？**
1. **命名空间隔离**：`cache:` 区分业务数据和缓存数据
2. **便于监控**：`redis-cli keys "cache:shop:*"` 查看所有商铺缓存
3. **便于批量删除**：更新商铺分类时可批量清理
4. **避免冲突**：不同模块用不同前缀（`lock:shop:` / `cache:shop:`）

### 3.2 为什么不用 Spring @Cacheable？

**Spring Cache 的痛点**：

```java
// 如果用 @Cacheable，代码会是这样：
@Cacheable(value = "shop", key = "#id", unless = "#result == null")
public Shop getById(Long id) {
    return baseMapper.selectById(id);
}
```

**看似简单，但无法解决**：
1. **缓存穿透**：`@Cacheable` 不会缓存 null 值（unless 配置复杂）
2. **缓存击穿**：无法加分布式锁
3. **缓存雪崩**：无法自定义 TTL 随机化
4. **监控困难**：无法记录缓存命中率
5. **降级困难**：Redis 挂了整个服务不可用

**手动控制的优势**：
```java
// 当前实现（CacheService.queryPassThrough）
public <R,ID> R queryPassThrough(...) {
    // 1. 查缓存
    String json = stringRedisTemplate.opsForValue().get(key);
    
    // 2. 命中判断（包括空值）
    if (!StrUtil.isBlank(json)) {
        return JSONUtil.toBean(json, type);
    }
    if (json != null && json.isEmpty()) {  // 空值缓存
        return null;
    }
    
    // 3. 查数据库
    R res = dataBaseCallback.apply(id);
    
    // 4. 缓存空值（防穿透）
    if (res == null) {
        set(key, "", time, unit);  // 存空字符串
        return null;
    }
    
    // 5. 缓存正常数据
    set(key, res, time, unit);
    return res;
}
```

**这段代码的工程价值**：
- **明确的空值处理**：`""` 表示数据库中不存在
- **统一的序列化方式**：JSON 便于跨语言
- **灵活的回调机制**：`this::getById` 可替换为任何查询逻辑

### 3.3 为什么 TTL 不能统一？

**表面上看**：
```java
CACHE_SHOP_TTL = 30L  // 所有商铺都是 30 分钟
```

**实际生产中会这样优化**：
```java
// 根据商铺热度动态调整
Long ttl = calculateTTL(shop);

private Long calculateTTL(Shop shop) {
    if (shop.getSold() > 10000) {  // 热门商铺
        return 60L;  // 1 小时
    } else if (shop.getSold() > 1000) {
        return 30L;  // 30 分钟
    } else {
        return 10L;  // 冷门商铺只缓存 10 分钟
    }
}

// 加上随机因素防止雪崩
Long finalTTL = ttl + RandomUtil.randomLong(0, 5);
```

**为什么要这样？**

1. **防止缓存雪崩**
```
所有商铺 TTL 都是 30 分钟
→ 假设早上 10 点批量预热缓存
→ 10:30 分全部过期
→ 瞬间 1000+ 个商铺同时查库
→ 数据库瞬间压力 100 倍
```

2. **节省内存**
```
Redis 内存有限（假设 4GB）
→ 热门商铺缓存 1 小时（高频访问）
→ 冷门商铺缓存 10 分钟（低频访问）
→ 自动淘汰不活跃数据
```

3. **提高命中率**
```
热门商铺：TTL 长 → 命中率 99%
冷门商铺：TTL 短 → 命中率 60%（但访问频率低，影响小）
整体命中率：95%+（帕累托法则）
```

---

## 四、【三大缓存问题的具体落地】- 理论到代码的映射

### 4.1 缓存穿透：恶意攻击的防御

**如何发生？**
```
黑客脚本：
for i in [999999, 999998, 999997, ...]:  # 不存在的商铺ID
    GET /shop/{i}
```

**没有防护时的链路**：
```
请求 /shop/999999
  ↓ Redis 查询 cache:shop:999999 → 不存在
  ↓ MySQL 查询 SELECT * FROM tb_shop WHERE id = 999999 → 不存在
  ↓ 返回 404
  ↓ 缓存未写入（因为是 null）
  
下一个请求 /shop/999998
  ↓ 又查 MySQL...
  ↓ 又是 null...
  
结果：每个恶意请求都打到数据库，缓存完全失效
```

**当前方案：缓存空值**
```java
// CacheService.queryPassThrough 第 48-50 行
if (res == null) {
    set(key, "", time, unit);  // 关键：存空字符串
}

// 下次查询时（第 41-44 行）
if (json != null && json.isEmpty()) {  // 识别空值
    Result.fail("信息不存在！");
    return null;
}
```

**Redis 中的表现**：
```redis
Key:   "cache:shop:999999"
Value: ""  （空字符串）
TTL:   1800 秒
```

**工程细节**：
1. **空值 TTL 应该更短**
   ```java
   // 当前代码问题：空值 TTL 也是 30 分钟，浪费内存
   // 生产建议：
   if (res == null) {
       set(key, "", CACHE_NULL_TTL, unit);  // 2 分钟即可
   }
   ```

2. **空值的副作用**
   - **内存占用**：1 万个恶意 ID = 1 万个空值缓存
   - **数据延迟**：新增商铺后，空值还在缓存中（要等到过期）
   
3. **生产级方案组合拳**
   ```java
   // 方案1：布隆过滤器（项目中未实现，但面试必问）
   if (!bloomFilter.mightContain(id)) {
       return Result.fail("商铺不存在");
   }
   
   // 方案2：ID 格式校验
   if (id < 1 || id > 999999) {
       return Result.fail("非法ID");
   }
   
   // 方案3：缓存空值（兜底）
   ```

### 4.2 缓存击穿：热点数据的生死时刻

**什么是击穿？**
```
星巴克(ID=1)是超级热点商铺
→ 每秒 1000 次查询
→ 缓存 TTL 到期
→ 这 1 秒内的 1000 个请求同时发现缓存不存在
→ 1000 个线程同时查 MySQL
→ 数据库瞬间死亡
```

**当前方案：互斥锁（注释代码中的逻辑）**

```java
// ShopServiceImpl 第 203-209 行（已注释但值得研究）
public boolean tryLock(long id) {
    Boolean result = stringRedisTemplate.opsForValue()
        .setIfAbsent(LOCK_SHOP_KEY + id, "1", 10, TimeUnit.SECONDS);
    return BooleanUtil.isTrue(result);
}
```

**Redis 命令原理**：
```redis
SET lock:shop:1 1 NX EX 10

NX: 只有 key 不存在时才设置（Not eXists）
EX 10: 10 秒后自动删除（防止死锁）
```

**完整流程演示**：
```
时刻 T0: 缓存过期

线程1: 
  ↓ Redis GET cache:shop:1 → null
  ↓ SET lock:shop:1 1 NX → OK（获取锁成功）
  ↓ 查询 MySQL（耗时 50ms）
  ↓ 写入缓存
  ↓ DEL lock:shop:1（释放锁）

线程2（晚 5ms 到达）:
  ↓ Redis GET cache:shop:1 → null
  ↓ SET lock:shop:1 1 NX → FAIL（锁已被线程1占用）
  ↓ sleep(10ms)
  ↓ 递归重试 queryByID(1)
  ↓ Redis GET cache:shop:1 → 命中！（线程1已填充）

线程3-1000: 同线程2
```

**互斥锁的代价**：
- **延迟增加**：等待锁的线程响应时间 +10ms
- **重试风暴**：1000 个线程递归重试，CPU 压力
- **锁超时风险**：如果 MySQL 查询超过 10 秒，锁自动释放

**逻辑过期方案对比**（CacheService 第 55-97 行）

```java
// 核心思想：永不删除缓存，只更新过期标记
{
  "data": { "id": 1, "name": "星巴克" },
  "expireTime": "2024-01-23 10:00:00"  // 逻辑过期时间
}

// 查询逻辑
if (expire.isBefore(LocalDateTime.now())) {  // 过期了
    if (tryLock(id)) {  // 尝试获取锁
        // 提交异步任务重建（不阻塞当前请求）
        CACHE_REBUILD_EXECUTOR.submit(() -> {
            // 查库 + 更新缓存
        });
    }
    return result;  // 直接返回旧数据
}
```

**方案选择建议**：
```
互斥锁：
  ✓ 适用：普通商铺、一致性要求高
  ✗ 不适用：极热数据（等待时间不可接受）

逻辑过期：
  ✓ 适用：秒杀商品、网红店（可接受短暂不一致）
  ✗ 不适用：价格敏感数据（返回旧价格可能违规）
```

### 4.3 缓存雪崩：在这个项目中真的会发生吗？

**雪崩场景**：
```
凌晨 3 点运维重启 Redis
→ 所有缓存清空
→ 早上 8 点用户上班开始使用
→ 所有请求全部 Miss
→ MySQL 瞬间承受全量流量
→ 连接池耗尽 → 服务挂掉
```

**当前项目的"防御"**：
```java
// RedisConstants.java 第 11 行
CACHE_SHOP_TTL = 30L  // 固定 30 分钟

// 问题：所有商铺 TTL 相同，仍有雪崩风险
```

**真实的雪崩触发条件**：
1. **批量过期**：定时任务批量预热缓存（所有 TTL 对齐）
2. **Redis 重启**：运维操作、机器故障
3. **内存淘汰**：Redis 内存满，LRU 算法批量删除

**生产级防御方案**：

**方案1：TTL 随机化**
```java
// 当前项目缺失，面试要提到
Long randomTTL = CACHE_SHOP_TTL + RandomUtil.randomLong(0, 5);
stringRedisTemplate.opsForValue().set(key, json, randomTTL, TimeUnit.MINUTES);

// 效果：
// 商铺1: 30分钟
// 商铺2: 32分钟
// 商铺3: 34分钟
// → 过期时间分散，不会同时失效
```

**方案2：多级缓存**
```java
// 本地缓存 + Redis
// Caffeine/Guava Cache
@Cacheable(value = "local-shop", unless = "#result == null")
public Shop queryWithLocalCache(Long id) {
    // 先查本地缓存（JVM 内存）
    // 未命中再查 Redis
    // 仍未命中再查 MySQL
}

// 优势：Redis 挂了，本地缓存还能扛一波
```

**方案3：Redis 集群 + 持久化**
```yaml
# redis.conf
appendonly yes  # AOF 持久化
save 900 1      # RDB 定期备份

# 主从 + 哨兵
sentinel monitor mymaster 127.0.0.1 6379 2
```

**这个项目的实际情况**：
- **小规模**：商铺数量 < 1 万，全量缓存压力小
- **热点明显**：20% 商铺占 80% 流量（即使雪崩也只影响长尾）
- **降级容易**：直接查 MySQL 也能抗（商铺表结构简单、有索引）

**结论**：
- **教学项目**：雪崩风险较小（数据量、并发量都不高）
- **生产环境**：必须加 TTL 随机化 + 监控预警

---

## 五、【关键代码逻辑拆解】- 每一行都有理由

### 5.1 为什么是 "查缓存 → 查数据库 → 回填缓存"？

**Cache-Aside Pattern（旁路缓存模式）**

```java
// CacheService.queryPassThrough 完整流程
public <R,ID> R queryPassThrough(...) {
    String key = Prefix + id;  // 1️⃣ 构造缓存 key
    
    // 2️⃣ 查缓存
    String json = stringRedisTemplate.opsForValue().get(key);
    if (!StrUtil.isBlank(json)) {
        return JSONUtil.toBean(json, type);  // 命中直接返回
    }
    
    // 3️⃣ 判断是否空值缓存
    if (json != null && json.isEmpty()) {
        return null;
    }
    
    // 4️⃣ 查数据库（回调函数）
    R res = dataBaseCallback.apply(id);  // this::getById
    
    // 5️⃣ 回填缓存（无论有无数据）
    if (res == null) {
        set(key, "", time, unit);  // 空值
    } else {
        set(key, res, time, unit);  // 正常数据
    }
    
    return res;
}
```

**为什么不是其他模式？**

**❌ Read-Through（读穿透）**
```java
// 缓存层自动加载数据
Shop shop = cache.get(id, key -> {
    return shopMapper.selectById(key);  // 缓存自动调用
});

// 问题：
// 1. 无法精细控制缓存逻辑
// 2. 无法处理缓存穿透
// 3. 无法加分布式锁
```

**❌ Write-Through（写穿透）**
```java
// 更新数据时同步更新缓存
cache.put(id, shop);  // 自动写 Redis + MySQL

// 问题：
// 1. 读多写少场景，写操作变慢
// 2. 并发写冲突难处理
```

**✅ Cache-Aside 优势**：
- **应用层控制**：可灵活处理各种异常
- **懒加载**：只缓存被访问过的数据
- **更新简单**：直接删除缓存（项目中第 156 行）

### 5.2 互斥锁在代码中的体现

**完整带锁的查询逻辑**（注释代码复原）：

```java
// ShopServiceImpl 原始逻辑（第 54-99 行注释部分）
public Result queryByID(Long id) {
    // 1. 查缓存
    Map<Object,Object> map = stringRedisTemplate.opsForHash()
        .entries(CACHE_SHOP_KEY + id);
    
    // 2. 命中空值
    if (map.containsKey("empty")) {
        return Result.fail("无相关商家信息！");
    }
    
    // 3. 命中数据
    if (!map.isEmpty()) {
        Shop result = BeanUtil.fillBeanWithMap(map, new Shop(), false);
        return Result.ok(result);
    }
    
    // 4. 未命中，尝试获取锁（防击穿）
    boolean flag = tryLock(id);
    if (!flag) {
        // 4.1 获取失败，等待 10ms 后重试
        Thread.sleep(10);
        return queryByID(id);  // 递归调用
    }
    
    // 5. 获取锁成功，查询数据库
    Shop result = getById(id);
    
    // 6. 处理空值
    if (result == null) {
        Map<String, String> emptyMap = new HashMap<>();
        emptyMap.put("empty", "true");
        stringRedisTemplate.opsForHash().putAll(CACHE_SHOP_KEY + id, emptyMap);
        stringRedisTemplate.expire(CACHE_SHOP_KEY + id, 30, TimeUnit.MINUTES);
        return Result.fail("404");
    }
    
    // 7. 写入缓存
    Map<String,Object> cached_map = BeanUtil.beanToMap(result, 
        new HashMap<>(),
        CopyOptions.create()
            .setIgnoreNullValue(true)
            .setFieldValueEditor((filedName, fieldValue) -> 
                fieldValue != null ? fieldValue.toString() : null
            )
    );
    stringRedisTemplate.opsForHash().putAll(CACHE_SHOP_KEY + id, cached_map);
    stringRedisTemplate.expire(CACHE_SHOP_KEY + id, 30, TimeUnit.MINUTES);
    
    // 8. 释放锁（注意：原代码缺失 unlock 调用，这是个 Bug！）
    // unlock(id);  // 应该有这行
    
    return Result.ok(result);
}
```

**⚠️ 原代码的一个 Bug**：
```java
// 获取锁后查询数据库，但没有 finally 块释放锁
if (!flag) {
    Thread.sleep(10);
    return queryByID(id);  // 递归重试
}
// 缺少 unlock(id)！

// 正确写法：
boolean flag = false;
try {
    flag = tryLock(id);
    if (!flag) {
        Thread.sleep(10);
        return queryByID(id);
    }
    // ... 查询逻辑
} finally {
    if (flag) {
        unlock(id);  // 确保释放
    }
}
```

### 5.3 哪些代码是"为了并发安全而存在"的

**标注每一行的用途**：

```java
// ===== 并发安全相关 =====
private static final ExecutorService CACHE_REBUILD_EXECUTOR = 
    Executors.newFixedThreadPool(10);  
// 👆 线程池大小 10：防止大量异步任务耗尽系统资源

public boolean tryLock(long id) {
    Boolean result = stringRedisTemplate.opsForValue()
        .setIfAbsent(LOCK_SHOP_KEY + id, "1", 10, TimeUnit.SECONDS);
    // 👆 10 秒 TTL：防止死锁（查询超时也能自动释放）
    return BooleanUtil.isTrue(result);
    // 👆 拆箱空指针保护：Redis 返回 null 时不会 NPE
}

// ===== 数据安全相关 =====
if (json != null && json.isEmpty()) {
    // 👆 空值判断：防止缓存穿透
    return null;
}

if (res == null) {
    set(key, "", time, unit);
    // 👆 缓存空值：防止同一恶意 ID 反复查库
}

// ===== 原子性相关 =====
stringRedisTemplate.opsForValue()
    .setIfAbsent(LOCK_SHOP_KEY + id, "1", 10, TimeUnit.SECONDS);
// 👆 SETNX + EX 是原子操作（单条 Redis 命令）
// 如果分两步：SET + EXPIRE，中间可能宕机导致永久锁
```

**并发测试用例**（面试可能让你写）：
```java
@Test
public void testConcurrentQuery() throws Exception {
    Long shopId = 1L;
    int threadCount = 100;
    
    // 清空缓存，模拟缓存失效
    stringRedisTemplate.delete(CACHE_SHOP_KEY + shopId);
    
    CountDownLatch latch = new CountDownLatch(threadCount);
    AtomicInteger dbQueryCount = new AtomicInteger(0);
    
    // 模拟 100 个并发请求
    for (int i = 0; i < threadCount; i++) {
        new Thread(() -> {
            try {
                latch.countDown();
                latch.await();  // 所有线程同时开始
                shopService.queryByID(shopId);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }).start();
    }
    
    Thread.sleep(2000);
    
    // 验证：加锁的情况下，MySQL 查询应该只有 1 次
    // 不加锁的情况下，MySQL 查询可能有 100 次
    System.out.println("数据库查询次数: " + dbQueryCount.get());
    // 期望输出：1（说明锁生效）
}
```

---

## 六、【设计取舍与反思】- 工程师的思维方式

### 6.1 逻辑过期 vs 互斥锁：各自的代价

**真实场景对比**：

| 场景 | 互斥锁方案 | 逻辑过期方案 | 推荐 |
|------|---------|-----------|-----|
| 普通商铺（日访问 < 100） | 平均延迟 +0ms（基本不锁） | 内存占用 +100%（存过期时间） | 互斥锁 |
| 热门商铺（日访问 1万+） | 峰值延迟 +10ms（锁等待） | 平均延迟 0ms（秒返回） | 逻辑过期 |
| 价格敏感数据（商品价格） | 强一致性 | 可能返回旧价格 | 互斥锁 |
| 秒杀商品（万人抢购） | 响应慢（全在等锁） | 体验好（瞬间响应） | 逻辑过期 |

**成本分析**：

**互斥锁成本**：
```
内存：0 额外成本（只是加个锁key）
CPU：低（等待期间 sleep）
延迟：+10-50ms（等待锁释放）
复杂度：低
一致性：强
```

**逻辑过期成本**：
```
内存：+50%（需存储过期时间字段）
CPU：高（异步线程重建缓存）
延迟：0（直接返回旧数据）
复杂度：高（需线程池、二次检查）
一致性：弱（可能返回过期数据）
```

**代码复杂度对比**：

```java
// 互斥锁：20 行代码
public Shop query(Long id) {
    String json = redis.get(key);
    if (json != null) return parse(json);
    
    if (tryLock(id)) {
        try {
            Shop shop = db.query(id);
            redis.set(key, shop, 30, MINUTES);
            return shop;
        } finally {
            unlock(id);
        }
    } else {
        sleep(10);
        return query(id);  // 重试
    }
}

// 逻辑过期：50+ 行代码
public Shop query(Long id) {
    String json = redis.get(key);
    if (json == null) return null;
    
    RedisData data = parse(json);
    if (data.expireTime.isAfter(now())) {
        return data.shop;  // 未过期
    }
    
    // 过期了，尝试更新
    if (tryLock(id)) {
        // 二次检查
        json = redis.get(key);
        data = parse(json);
        if (data.expireTime.isAfter(now())) {
            unlock(id);
            return data.shop;
        }
        
        // 异步重建
        executor.submit(() -> {
            try {
                Shop shop = db.query(id);
                RedisData newData = new RedisData();
                newData.shop = shop;
                newData.expireTime = now().plusMinutes(30);
                redis.set(key, newData);
            } finally {
                unlock(id);
            }
        });
    }
    
    return data.shop;  // 返回旧数据
}
```

**项目中为什么用互斥锁？**
1. **商铺数据更新频率低**：天级更新，弱一致性没必要
2. **热点不集中**：不是所有商铺都高并发
3. **代码简单**：便于教学和维护

**什么时候必须用逻辑过期？**
- 秒杀场景（库存查询）
- 首页热点数据（轮播图、热门榜单）
- 极高 QPS（单 key 超过 1 万/秒）

### 6.2 如果 Shop 数据更新频繁，这套方案还成立吗？

**假设场景**：引入「实时库存」功能
```java
public class Shop {
    private Long id;
    private String name;
    private Integer stock;  // 新增：实时库存（每笔订单都更新）
    ...
}
```

**现有方案的问题**：

**问题1：缓存一致性崩溃**
```
时刻 T0: 
  用户 A 下单，库存 100 → 99
  执行：updateById(shop)  // 数据库变成 99
  执行：delete(cache:shop:1)  // 删除缓存

时刻 T1（0.1秒后）:
  用户 B 查询商铺
  缓存未命中，查数据库，库存 = 99
  写入缓存

时刻 T2（0.2秒后）:
  用户 C 下单，库存 99 → 98
  updateById(shop)  // 数据库变成 98
  delete(cache:shop:1)  // 删除缓存

时刻 T3（0.3秒后）:
  用户 D 查询商铺
  又要重建缓存...

结果：缓存频繁失效，命中率暴跌！
```

**问题2：数据库压力激增**
```
每秒 100 笔订单
→ 每秒删除 100 次缓存
→ 每秒重建 100 次缓存（查库 100 次）
→ 等于没有缓存！
```

**改进方案**：

**方案1：拆分冷热数据**
```java
// 冷数据（变化少）：缓存 30 分钟
@Data
public class ShopInfo {
    private Long id;
    private String name;
    private String address;
    private String images;
}

// 热数据（频繁变）：缓存 1 分钟或直接查库
@Data
public class ShopStock {
    private Long shopId;
    private Integer stock;
}

// 前端请求
GET /shop/1         → ShopInfo（高命中率）
GET /shop/1/stock   → ShopStock（可接受低命中率）
```

**方案2：使用 Redis 原子操作**
```java
// 不再缓存整个 Shop 对象，只缓存库存
public Integer getStock(Long shopId) {
    String key = "shop:stock:" + shopId;
    String stock = redis.get(key);
    if (stock == null) {
        stock = db.getStock(shopId).toString();
        redis.set(key, stock, 1, TimeUnit.MINUTES);
    }
    return Integer.parseInt(stock);
}

// 下单时直接操作 Redis
public void decrStock(Long shopId) {
    redis.decr("shop:stock:" + shopId);
    // 异步同步到 MySQL
}
```

**方案3：延迟双删**
```java
public void updateShop(Shop shop) {
    // 1. 删除缓存
    redis.delete(CACHE_SHOP_KEY + shop.getId());
    
    // 2. 更新数据库
    updateById(shop);
    
    // 3. 延迟 500ms 再删除一次（防止脏数据）
    Thread.sleep(500);
    redis.delete(CACHE_SHOP_KEY + shop.getId());
}
```

**结论**：
- **当前项目**：Shop 数据基本不变，Cache-Aside 完美
- **高频更新场景**：需要拆分数据 + 引入消息队列 + 最终一致性方案

### 6.3 哪些设计是"教学项目友好，但生产要谨慎"的？

**⚠️ 问题1：递归重试逻辑**
```java
// CacheService 第 69-94 行
if (!tryLock(id)) {
    Thread.sleep(10);
    return queryByID(id);  // 递归调用
}
```

**生产环境风险**：
```
假设：MySQL 挂了，查询超时 30 秒
→ 所有请求都获取不到锁
→ 每个请求每 10ms 递归一次
→ 30 秒内递归 3000 次
→ 栈溢出 StackOverflowError
```

**改进方案**：
```java
// 用循环代替递归
public Shop queryByID(Long id) {
    int retryCount = 0;
    while (retryCount < 3) {  // 最多重试 3 次
        String json = redis.get(key);
        if (json != null) return parse(json);
        
        if (tryLock(id)) {
            try {
                return queryAndCache(id);
            } finally {
                unlock(id);
            }
        } else {
            retryCount++;
            Thread.sleep(50);  // 每次等待时间递增
        }
    }
    
    // 重试失败，降级查库
    return getById(id);
}
```

**⚠️ 问题2：固定的线程池大小**
```java
// ShopServiceImpl 第 52 行
private static final ExecutorService CACHE_REBUILD_EXECUTOR = 
    Executors.newFixedThreadPool(10);
```

**生产环境风险**：
```
10 个固定线程
→ 如果 11 个热点 key 同时过期
→ 第 11 个任务会阻塞
→ 可能导致缓存长时间不更新
```

**改进方案**：
```java
// 使用 ThreadPoolExecutor，可配置拒绝策略
private static final ExecutorService CACHE_REBUILD_EXECUTOR = 
    new ThreadPoolExecutor(
        5,   // 核心线程数
        20,  // 最大线程数
        60, TimeUnit.SECONDS,  // 空闲线程存活时间
        new LinkedBlockingQueue<>(100),  // 队列大小
        new ThreadPoolExecutor.CallerRunsPolicy()  // 拒绝策略：调用者执行
    );
```

**⚠️ 问题3：缺少监控和降级**
```java
// 当前代码：没有任何监控埋点
Shop result = cacheClient.queryPassThrough(...);
return Result.ok(result);
```

**生产环境必备**：
```java
public Result queryByID(Long id) {
    long startTime = System.currentTimeMillis();
    boolean cacheHit = false;
    
    try {
        // 查询逻辑
        Shop result = cacheClient.queryPassThrough(...);
        cacheHit = (result != null);
        return Result.ok(result);
    } catch (Exception e) {
        // 降级：Redis 挂了直接查库
        log.error("缓存查询失败，降级查库", e);
        return Result.ok(getById(id));
    } finally {
        // 监控埋点
        long cost = System.currentTimeMillis() - startTime;
        monitorService.record("shop.query", cost, cacheHit);
        
        // 慢查询告警
        if (cost > 100) {
            log.warn("Shop查询慢: id={}, cost={}ms", id, cost);
        }
    }
}
```

**⚠️ 问题4：空值缓存的内存泄漏风险**
```java
// 当前代码：空值 TTL 也是 30 分钟
if (res == null) {
    set(key, "", time, unit);  // time = 30 分钟
}
```

**风险**：
```
恶意攻击：请求 10 万个不存在的商铺 ID
→ Redis 存储 10 万个空值
→ 每个空值占用 ~100 字节
→ 总计 10MB（看似不多）
→ 但如果是图片、文章等大对象，内存占用会爆炸
```

**改进方案**：
```java
// 空值单独设置更短的 TTL
if (res == null) {
    set(key, "", CACHE_NULL_TTL, unit);  // 2 分钟
} else {
    set(key, res, time, unit);  // 30 分钟
}

// 或者用布隆过滤器完全避免缓存空值
```

---

## 七、【面试高频问题】- 考官想听什么

### 7.1 如果 Redis 宕机，Shop 查询会怎样？

**当前代码表现**：
```java
// CacheService.queryPassThrough 第 36 行
String json = stringRedisTemplate.opsForValue().get(key);
// 👆 如果 Redis 宕机，这行会抛异常

// 没有 try-catch，会直接传播到 Controller
// 前端看到：500 Internal Server Error
```

**问题链路**：
```
Redis 宕机
  ↓ stringRedisTemplate.get() 抛异常
  ↓ 异常未捕获
  ↓ 整个查询失败
  ↓ 用户无法访问任何商铺
```

**面试标准答案**（分层次）：

**Level 1：降级方案**
```java
public <R,ID> R queryPassThrough(...) {
    try {
        // 尝试查 Redis
        String json = stringRedisTemplate.opsForValue().get(key);
        if (!StrUtil.isBlank(json)) {
            return JSONUtil.toBean(json, type);
        }
    } catch (Exception e) {
        // Redis 异常，记录日志但不阻断流程
        log.error("Redis查询失败，降级查库: key={}", key, e);
    }
    
    // 直接查库（降级）
    R res = dataBaseCallback.apply(id);
    
    // 尝试写缓存（可能仍失败，但不影响返回）
    try {
        if (res != null) {
            set(key, res, time, unit);
        }
    } catch (Exception e) {
        log.error("Redis写入失败: key={}", key, e);
    }
    
    return res;
}
```

**Level 2：熔断机制**
```java
// 引入 Hystrix 或 Sentinel
@SentinelResource(value = "queryShop", 
                  fallback = "queryShopFallback")
public Shop queryByID(Long id) {
    return cacheClient.queryPassThrough(...);
}

// 降级方法
public Shop queryShopFallback(Long id, Throwable e) {
    log.error("查询商铺失败，触发降级", e);
    return shopMapper.selectById(id);  // 直接查库
}

// 熔断规则
// 1分钟内失败 50% 以上 → 熔断 10 秒
// 熔断期间直接走降级，不再尝试访问 Redis
```

**Level 3：多级缓存**
```java
// 本地缓存 + Redis
public Shop queryByID(Long id) {
    // 1. 查本地缓存（Caffeine）
    Shop shop = localCache.getIfPresent(id);
    if (shop != null) return shop;
    
    // 2. 查 Redis
    try {
        shop = redisCache.get(id);
        if (shop != null) {
            localCache.put(id, shop);  // 回填本地缓存
            return shop;
        }
    } catch (Exception e) {
        log.error("Redis异常", e);
    }
    
    // 3. 查数据库
    shop = shopMapper.selectById(id);
    
    // 4. 回填缓存（尽力而为）
    localCache.put(id, shop);
    try {
        redisCache.put(id, shop);
    } catch (Exception ignored) {}
    
    return shop;
}
```

### 7.2 空值缓存会带来什么副作用？

**面试答题结构**：

**副作用1：内存占用**
```
场景：遭受恶意攻击
攻击者：循环请求不存在的商铺ID（999999-899999）
结果：Redis 存储 10 万个空值
内存占用：10 万 × 100 字节 = 10MB（可接受）

但如果是文章、商品等大对象：
10 万 × 10KB = 1GB（严重！）
```

**应对方案**：
```java
// 1. 空值单独设置更短 TTL
CACHE_NULL_TTL = 2L;  // 2 分钟

// 2. 限制空值缓存数量（使用 LRU）
// Redis 配置：maxmemory-policy allkeys-lru

// 3. 布隆过滤器前置拦截
```

**副作用2：新增数据不可见**
```
时刻 T0:
  用户查询商铺 ID=100（不存在）
  → 缓存空值，TTL=30分钟

时刻 T1（5分钟后）:
  运营新增商铺 ID=100

时刻 T2:
  用户查询商铺 ID=100
  → 命中空值缓存
  → 返回"商铺不存在"
  → 实际上已经存在了！
  
  要等 25 分钟后空值过期才能看到
```

**应对方案**：
```java
// 新增商铺时，主动删除可能存在的空值缓存
public void saveShop(Shop shop) {
    shopMapper.insert(shop);
    
    // 删除空值缓存（如果存在）
    stringRedisTemplate.delete(CACHE_SHOP_KEY + shop.getId());
}
```

**副作用3：误判风险**
```java
// 当前代码的判断逻辑
if (json != null && json.isEmpty()) {
    return null;  // 认为是空值
}

// 问题：如果序列化后的对象真的是空字符串怎么办？
// 例如：Shop 对象所有字段都是 null
// JSONUtil.toJsonStr(shop) 可能返回 "{}"

// 更严谨的做法：
public class CacheValue {
    private boolean isNull;  // 显式标记
    private String data;
}
```

**副作用4：缓存击穿风险转移**
```
空值缓存防止了「穿透」（不存在的数据）
但可能引发「击穿」（空值过期瞬间）

场景：
  10:00  空值缓存写入，TTL=2分钟
  10:02  空值过期
  10:02:00.001  100个请求同时到达
  → 100个请求都要查库
  → 击穿！

解决：空值缓存也要加互斥锁
```

### 7.3 为什么不用本地缓存 + Redis 两级缓存？

**面试考点**：考察对架构演进的理解

**Level 1：技术对比**

| 维度 | 纯 Redis | 本地缓存 + Redis |
|------|---------|----------------|
| 响应时间 | 2-5ms | 0.1ms（本地命中） |
| 一致性 | 强（单一数据源） | 弱（多节点可能不一致） |
| 内存占用 | Redis 服务器 | 每个应用节点 × N |
| 复杂度 | 简单 | 复杂（需考虑失效策略） |

**Level 2：适用场景**

**适合两级缓存的场景**：
```java
// 1. 极高频读取（单机 QPS > 1万）
// 例如：首页配置、字典数据

@Cacheable(cacheNames = "local-config", unless = "#result == null")
public AppConfig getConfig() {
    // 本地缓存 1 分钟
    // Redis 缓存 10 分钟
}

// 2. 数据量小（可全量加载）
// 例如：商铺分类（总共才 20 个）

@PostConstruct
public void loadShopTypes() {
    List<ShopType> types = shopTypeMapper.selectAll();
    localCache.putAll(types);  // 启动时全量加载
}
```

**不适合的场景（Shop 模块）**：
```java
// 1. 数据量大（上万个商铺）
// 每个应用节点缓存所有商铺 → 内存爆炸

// 2. 更新频繁（营业时间、价格调整）
// 更新时要通知所有节点清除本地缓存 → 复杂

// 3. 数据分布不均（80%请求访问20%商铺）
// 大量冷数据占用本地内存 → 浪费
```

**Level 3：引入两级缓存的代价**

**一致性问题**：
```
场景：3 个应用节点（A、B、C）

T0: 节点 A 查询商铺1，写入本地缓存 + Redis
T1: 运营更新商铺1，删除 Redis 缓存
T2: 节点 A 的本地缓存还在（不知道 Redis 已删除）
    用户访问节点 A → 返回旧数据（脏读！）
```

**解决方案**：
```java
// 1. 引入消息队列
// 更新时发送 MQ 消息，所有节点监听并清除本地缓存

public void updateShop(Shop shop) {
    updateById(shop);
    redis.delete(CACHE_SHOP_KEY + shop.getId());
    mq.send("shop.update", shop.getId());  // 通知所有节点
}

// 每个节点监听消息
@RabbitListener(queues = "shop.update")
public void onShopUpdate(Long shopId) {
    localCache.invalidate(shopId);
}

// 2. 设置极短的本地缓存 TTL
localCache.expireAfterWrite(10, TimeUnit.SECONDS);
// 最多 10 秒不一致，可接受

// 3. 使用 Spring Cache + Redis 广播
@CacheEvict(cacheNames = "shop", key = "#shop.id")
public void updateShop(Shop shop) {
    updateById(shop);
}
// Spring Cache 会自动通过 Redis Pub/Sub 通知其他节点
```

**当前项目为什么不用？**
1. **并发量不高**：单机 Redis 能抗
2. **数据量大**：不适合全量加载到本地
3. **保持简单**：教学项目，避免过度设计

**什么时候必须用两级缓存？**
- 单机 QPS > 5 万（Redis 网络 IO 成为瓶颈）
- 数据量小且固定（配置、字典）
- 可接受秒级延迟（弱一致性）

---

## 八、【从零实现 Shop 模块】- 实战攻略

### 8.1 正确实现顺序（防止返工）

**阶段 1：基础查询（0 → 1）**
```java
// 目标：能查到数据即可
@Override
public Result queryByID(Long id) {
    return Result.ok(getById(id));
}

// 验证：
// 1. Postman 测试：GET /shop/1 → 200 OK
// 2. 不存在的ID：GET /shop/999999 → 返回 null
```

**阶段 2：简单缓存（1 → 10）**
```java
// 目标：减少数据库压力
@Override
public Result queryByID(Long id) {
    String key = CACHE_SHOP_KEY + id;
    
    // 1. 查缓存
    String json = stringRedisTemplate.opsForValue().get(key);
    if (StrUtil.isNotBlank(json)) {
        Shop shop = JSONUtil.toBean(json, Shop.class);
        return Result.ok(shop);
    }
    
    // 2. 查数据库
    Shop shop = getById(id);
    if (shop == null) {
        return Result.fail("商铺不存在");
    }
    
    // 3. 写缓存
    stringRedisTemplate.opsForValue().set(
        key, 
        JSONUtil.toJsonStr(shop), 
        30, 
        TimeUnit.MINUTES
    );
    
    return Result.ok(shop);
}

// 验证：
// 1. 第一次查询：观察 MySQL 日志（有查询）
// 2. 第二次查询：观察 MySQL 日志（无查询）
// 3. Redis CLI：GET cache:shop:1 → 有数据
```

**阶段 3：防穿透（10 → 30）**
```java
// 目标：抵御恶意请求
@Override
public Result queryByID(Long id) {
    String key = CACHE_SHOP_KEY + id;
    String json = stringRedisTemplate.opsForValue().get(key);
    
    // 命中缓存
    if (StrUtil.isNotBlank(json)) {
        return Result.ok(JSONUtil.toBean(json, Shop.class));
    }
    
    // 命中空值（新增）
    if (json != null) {  // "" 表示空值
        return Result.fail("商铺不存在");
    }
    
    // 查数据库
    Shop shop = getById(id);
    
    // 不存在，缓存空值（新增）
    if (shop == null) {
        stringRedisTemplate.opsForValue().set(
            key, 
            "",  // 空字符串
            CACHE_NULL_TTL,  // 2 分钟
            TimeUnit.MINUTES
        );
        return Result.fail("商铺不存在");
    }
    
    // 存在，缓存数据
    stringRedisTemplate.opsForValue().set(
        key, 
        JSONUtil.toJsonStr(shop), 
        CACHE_SHOP_TTL, 
        TimeUnit.MINUTES
    );
    
    return Result.ok(shop);
}

// 验证：
// 1. 请求不存在的ID：GET /shop/999999
// 2. Redis CLI：GET cache:shop:999999 → ""
// 3. 再次请求：观察 MySQL 日志（无查询）
```

**阶段 4：防击穿（30 → 60）**
```java
// 目标：热点数据失效时不压垮数据库
@Override
public Result queryByID(Long id) {
    String key = CACHE_SHOP_KEY + id;
    String json = stringRedisTemplate.opsForValue().get(key);
    
    if (StrUtil.isNotBlank(json)) {
        return Result.ok(JSONUtil.toBean(json, Shop.class));
    }
    
    if (json != null) {
        return Result.fail("商铺不存在");
    }
    
    // 尝试获取锁（新增）
    String lockKey = LOCK_SHOP_KEY + id;
    boolean locked = false;
    try {
        locked = tryLock(lockKey);
        
        if (!locked) {
            // 未获取锁，等待后重试
            Thread.sleep(50);
            return queryByID(id);  // 递归重试
        }
        
        // 获取锁成功，二次检查缓存（DCL）
        json = stringRedisTemplate.opsForValue().get(key);
        if (StrUtil.isNotBlank(json)) {
            return Result.ok(JSONUtil.toBean(json, Shop.class));
        }
        
        // 查数据库
        Shop shop = getById(id);
        
        // 模拟慢查询
        Thread.sleep(200);
        
        if (shop == null) {
            stringRedisTemplate.opsForValue().set(
                key, "", CACHE_NULL_TTL, TimeUnit.MINUTES
            );
            return Result.fail("商铺不存在");
        }
        
        stringRedisTemplate.opsForValue().set(
            key, 
            JSONUtil.toJsonStr(shop), 
            CACHE_SHOP_TTL, 
            TimeUnit.MINUTES
        );
        
        return Result.ok(shop);
        
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    } finally {
        if (locked) {
            unlock(lockKey);  // 确保释放锁
        }
    }
}

// 工具方法
private boolean tryLock(String key) {
    Boolean result = stringRedisTemplate.opsForValue()
        .setIfAbsent(key, "1", 10, TimeUnit.SECONDS);
    return Boolean.TRUE.equals(result);
}

private void unlock(String key) {
    stringRedisTemplate.delete(key);
}

// 验证：
// 1. JMeter 并发测试：100 线程同时请求同一商铺
// 2. 观察 MySQL 日志：只有 1 次查询
// 3. 观察响应时间：部分请求有 50ms 延迟（等锁）
```

**阶段 5：封装工具类（60 → 80）**
```java
// 目标：复用到其他模块
@Component
public class CacheService {
    @Autowired
    private StringRedisTemplate stringRedisTemplate;
    
    public <R, ID> R queryWithPassThrough(
        String keyPrefix,
        ID id,
        Class<R> type,
        Function<ID, R> dbFallback,
        Long time,
        TimeUnit unit
    ) {
        String key = keyPrefix + id;
        
        // 1. 查缓存
        String json = stringRedisTemplate.opsForValue().get(key);
        if (StrUtil.isNotBlank(json)) {
            return JSONUtil.toBean(json, type);
        }
        
        // 2. 空值判断
        if (json != null) {
            return null;
        }
        
        // 3. 查数据库
        R r = dbFallback.apply(id);
        
        // 4. 写缓存
        if (r == null) {
            stringRedisTemplate.opsForValue().set(key, "", 2L, TimeUnit.MINUTES);
        } else {
            stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(r), time, unit);
        }
        
        return r;
    }
}

// 使用
@Override
public Result queryByID(Long id) {
    Shop shop = cacheService.queryWithPassThrough(
        CACHE_SHOP_KEY,
        id,
        Shop.class,
        this::getById,  // 方法引用
        CACHE_SHOP_TTL,
        TimeUnit.MINUTES
    );
    return shop == null ? Result.fail("商铺不存在") : Result.ok(shop);
}
```

**阶段 6：完善更新逻辑（80 → 100）**
```java
// 目标：数据更新时保证一致性
@Override
public Result updateShop(Shop shop) {
    if (shop.getId() == null) {
        return Result.fail("商铺ID不能为空");
    }
    
    // 1. 更新数据库
    updateById(shop);
    
    // 2. 删除缓存（Cache-Aside 模式）
    stringRedisTemplate.delete(CACHE_SHOP_KEY + shop.getId());
    
    // 3. 删除可能存在的空值缓存
    // （如果之前查询过不存在，现在新增了）
    
    return Result.ok();
}

// 验证：
// 1. 更新商铺：PUT /shop {"id":1, "name":"新名称"}
// 2. Redis CLI：GET cache:shop:1 → nil（已删除）
// 3. 查询商铺：GET /shop/1 → 返回新名称
// 4. Redis CLI：GET cache:shop:1 → 有数据（重建成功）
```

### 8.2 最容易翻车的 3 个点

**翻车点 1：忘记释放锁**
```java
// ❌ 错误代码
boolean locked = tryLock(id);
if (!locked) {
    Thread.sleep(50);
    return queryByID(id);
}

// 查数据库...
// 写缓存...
unlock(id);  // 如果上面抛异常，这行不会执行！

// ✅ 正确代码
boolean locked = false;
try {
    locked = tryLock(id);
    if (!locked) {
        Thread.sleep(50);
        return queryByID(id);
    }
    
    // 查数据库...
    // 写缓存...
} finally {
    if (locked) {
        unlock(id);  // 确保释放
    }
}
```

**翻车现象**：
- 第一次查询正常
- 后续所有查询都卡死（锁永远不释放）
- 要等 10 秒锁自动过期

**翻车点 2：空值判断顺序错误**
```java
// ❌ 错误代码
String json = stringRedisTemplate.opsForValue().get(key);
if (json == null || json.isEmpty()) {
    // 缓存未命中，查数据库...
}

// 问题：无法区分「未缓存」和「缓存了空值」

// ✅ 正确代码
String json = stringRedisTemplate.opsForValue().get(key);

// 1. 有数据
if (StrUtil.isNotBlank(json)) {
    return JSONUtil.toBean(json, Shop.class);
}

// 2. 缓存了空值
if (json != null) {  // "" 不是 null
    return null;
}

// 3. 未缓存
// 查数据库...
```

**翻车现象**：
- 缓存穿透防护失效
- 每次查询不存在的ID都会打到数据库

**翻车点 3：更新时先删缓存再更新数据库**
```java
// ❌ 错误代码
public void updateShop(Shop shop) {
    // 1. 先删缓存
    stringRedisTemplate.delete(CACHE_SHOP_KEY + shop.getId());
    
    // 2. 再更新数据库
    updateById(shop);
}

// 并发问题：
// T0: 线程A 删除缓存
// T1: 线程B 查询（缓存未命中）
// T2: 线程B 查数据库（还是旧数据）
// T3: 线程B 写缓存（写入旧数据）
// T4: 线程A 更新数据库（新数据）
// 结果：缓存是旧数据，数据库是新数据，不一致！

// ✅ 正确代码
public void updateShop(Shop shop) {
    // 1. 先更新数据库
    updateById(shop);
    
    // 2. 再删缓存
    stringRedisTemplate.delete(CACHE_SHOP_KEY + shop.getId());
}
```

**翻车现象**：
- 更新后查询到旧数据
- 要等缓存过期才能看到新数据
- 非常难排查（不是每次都复现）

---

## 九、【总结：面试中如何讲这个模块】

### 9.1 一分钟版本（破冰）

```
面试官：讲讲你项目中的缓存设计。

你：
我们的黑马点评项目中，Shop 商铺查询模块是访问最频繁的功能，
读写比达到 300:1，所以我重点做了缓存优化。

采用 Cache-Aside 模式，先查 Redis，未命中再查 MySQL，
查到后回填缓存，TTL 设置 30 分钟。

为了应对缓存穿透，我缓存了空值，TTL 设置得更短（2分钟）。
为了防止热点数据击穿，用 Redis 的 SETNX 实现了互斥锁，
保证缓存失效时只有一个线程查库，其他线程等待并重试。

最终效果是缓存命中率达到 95%+，接口响应时间从 50ms 降到 5ms，
数据库压力下降了 20 倍。
```

### 9.2 五分钟版本（深入）

**1. 背景和问题（30秒）**
```
商铺查询是点评类 APP 的核心功能，用户浏览列表、查看详情
都要频繁查询商铺信息。我们监控发现高峰期 QPS 达到 200+，
数据库慢查询开始堆积，P95 响应时间超过 100ms。
```

**2. 方案选型（1分钟）**
```
我调研了三种方案：
- Spring @Cacheable：太简单，无法处理缓存击穿和穿透
- Guava 本地缓存：数据量大，每个节点都缓存会浪费内存
- Redis + 手动控制：灵活，能精细处理各种异常情况

最终选择 Redis + 手动控制，因为商铺数据更新频率低、
数据稳定、无强一致性要求，非常适合缓存。
```

**3. 核心实现（2分钟）**
```
我封装了一个 CacheService 工具类，核心方法是 queryPassThrough：

第一步：查 Redis，命中直接返回。这里用的是 JSON 序列化，
      因为跨语言兼容性好，运维用 Redis CLI 也能直接看数据。

第二步：判断是否空值缓存（空字符串），防止缓存穿透。
      之前遇到过恶意攻击，用脚本循环请求不存在的商铺 ID，
      加了这个判断后，第二次请求就直接被拦截了。

第三步：缓存未命中，查数据库。这里用了函数式编程，
      传入方法引用 this::getById，保证工具类通用。

第四步：无论有没有数据都写缓存。有数据就缓存 30 分钟，
      没数据就缓存空值 2 分钟（更短的 TTL）。

针对热点数据击穿，我在注释里保留了逻辑过期方案，
但当前用的是互斥锁：缓存失效时用 SETNX 获取锁，
获取失败的线程等待 10ms 后重试。这样能保证只有一个线程查库。
```

**4. 踩坑和优化（1分钟）**
```
开发过程中踩了几个坑：

1. 忘记释放锁：初版代码没用 try-finally，一旦查询异常，
   锁就永远不释放。后来加了 finally 块，并且给锁设置了 10 秒 TTL 兜底。

2. 递归重试可能栈溢出：如果数据库挂了，每个请求都会递归几千次。
   我在注释里标注了，生产环境应该改成循环 + 最大重试次数。

3. 更新时的一致性问题：初版是先删缓存再更新数据库，容易出现脏数据。
   后来改成先更新数据库再删缓存，虽然仍有极小概率不一致，
   但结合缓存的 TTL，影响范围可控。
```

**5. 效果和反思（30秒）**
```
上线后，缓存命中率稳定在 95% 以上，P95 响应时间降到 10ms 以内，
数据库 QPS 下降了 95%。

如果重新设计，我会加入 Sentinel 熔断降级，防止 Redis 故障影响主流程；
对于更高并发的场景，可以考虑本地缓存 + Redis 两级缓存，
但要处理好节点间的缓存一致性问题。
```

### 9.3 高频追问应答

| 面试官问题 | 标准答案 |
|----------|---------|
| 为什么用 Redis 不用 Memcached？ | 1. Redis 支持更多数据结构（Hash/Geo 后续用到）<br>2. Redis 有持久化，重启不丢数据<br>3. Redis 支持 Lua 脚本，可实现原子操作<br>4. 团队更熟悉 Redis |
| 缓存 TTL 为什么是 30 分钟？ | 1. 商铺数据基本不变，30分钟够用<br>2. 太长（2小时）会导致更新不及时<br>3. 太短（5分钟）缓存频繁失效，失去意义<br>4. 生产环境会加随机值（25-35分钟）防雪崩 |
| 如果要保证强一致性怎么办？ | 1. 不缓存（实时查库）<br>2. 使用 Canal 监听 Binlog，自动同步缓存<br>3. 使用 Redis + MySQL 分布式事务（Seata）<br>4. 但商铺数据没必要强一致 |
| 你们 Redis 是单机还是集群？ | 项目中单机，生产会用主从+哨兵<br>1. 主从复制保证高可用<br>2. 哨兵自动故障转移<br>3. 如果数据量大会用 Cluster 分片 |
| 缓存预热怎么做？ | 1. 定时任务：凌晨查库批量写 Redis<br>2. 启动时预热：@PostConstruct 加载热点数据<br>3. 运营工具：手动触发预热（新店开业） |
| 如果缓存和数据库不一致怎么排查？ | 1. 查日志：确认更新操作是否执行<br>2. 查 Redis：GET cache:shop:1 看数据<br>3. 查 TTL：TTL cache:shop:1 看过期时间<br>4. 加监控：埋点记录缓存更新操作 |

---

## 十、【一张图总结】

```
┌─────────────────────────────────────────────────────┐
│                  Shop 模块全景图                      │
└─────────────────────────────────────────────────────┘

用户请求：GET /shop/1
    ↓
┌───────────────────────────────────────────────────┐
│ Controller: ShopController.queryShopById()        │
└───────────────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────────────┐
│ Service: ShopServiceImpl.queryByID()              │
│  ↓                                                │
│  调用 CacheService.queryPassThrough()             │
└───────────────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────────────┐
│ CacheService 核心逻辑                              │
│                                                   │
│  1️⃣ 查 Redis: GET cache:shop:1                   │
│     ├─ 命中数据 → 返回 ✓                          │
│     ├─ 命中空值 → 返回 null（防穿透）✓             │
│     └─ 未命中 → 继续                              │
│                                                   │
│  2️⃣ 查 MySQL: SELECT * FROM tb_shop WHERE id=1   │
│     ├─ 有数据 → 缓存 30 分钟 ✓                    │
│     └─ 无数据 → 缓存空值 2 分钟（防穿透）✓         │
│                                                   │
│  3️⃣ 返回结果                                     │
└───────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════

三大问题的解决方案：

┌─────────────┬─────────────────────────────────────┐
│ 缓存穿透     │ 缓存空值（TTL=2分钟）                │
│ Cache Miss  │ Redis: "" 标记不存在的数据            │
├─────────────┼─────────────────────────────────────┤
│ 缓存击穿     │ 互斥锁（SETNX）                      │
│ Hotkey Fail │ 只允许一个线程查库，其他等待          │
├─────────────┼─────────────────────────────────────┤
│ 缓存雪崩     │ TTL 随机化（25-35分钟）              │
│ Mass Expire │ + Redis 持久化 + 主从高可用           │
└─────────────┴─────────────────────────────────────┘

═══════════════════════════════════════════════════════

关键指标：

  响应时间：5ms（缓存命中）vs 50ms（查库）
  命中率：  95%+
  并发能力：200+ QPS（单机 Redis）
  数据库压力：下降 95%

═══════════════════════════════════════════════════════

面试金句：

"Shop 模块是典型的读多写少场景，我用 Cache-Aside 模式
 配合 Redis 缓存，命中率达到 95%，响应时间降低 10 倍。

 针对缓存三大问题：穿透用空值，击穿用互斥锁，雪崩用 TTL 随机化。

 这套方案在教学项目中够用，生产环境还需要加熔断降级、
 监控告警和多级缓存等。"
```

---

## 写在最后：工程师的视角

这个 Shop 模块虽然是教学项目，但包含了真实高并发系统的核心思想：

1. **先简单后复杂**：从无缓存 → 简单缓存 → 防穿透 → 防击穿，逐步迭代
2. **先满足需求后优化**：先让功能跑起来，有压力再优化
3. **权衡取舍**：一致性 vs 性能，简单 vs 灵活，要做选择
4. **防御性编程**：互斥锁、空值缓存、TTL 兜底，处处考虑异常

面试时不要只背代码，要讲清楚：
- **为什么**这样设计（业务特点、技术选型）
- **怎么做**的（核心流程、关键代码）
- **有什么问题**（踩过的坑、改进方向）
- **如果是你**会怎么优化（展现技术视野）

最后，记住这句话：
**"代码是写给人看的，只是顺便让机器执行。"**

好的缓存方案不是最复杂的，而是最适合业务的。Shop 模块的成功不在于用了多少技术，而在于用最简单的方式解决了最核心的问题。

---

**文档版本**：v1.0  
**最后更新**：2026-01-23  
**作者视角**：高并发 Java 后端工程师  
**适用场景**：中高级后端面试、项目复盘、技术分享

祝面试顺利！🚀
