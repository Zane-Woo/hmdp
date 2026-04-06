# Redis存储结构总览（hm-dianping0）

> 依据代码实际读写点整理：`back_end/hm-dianping/src/main/java` 与 `src/main/resources/*.lua`。

## 1. Key 总览表

| 模块 | Key 模式 | Redis类型 | Value/字段结构 | TTL/裁剪 | 主要读写位置 |
|---|---|---|---|---|---|
| 登录验证码 | `login:code:{phone}` | String | 6位验证码字符串 | 2分钟 | `UserServiceImpl.sendCode()` 写；`UserServiceImpl.login()` 读 |
| 登录态 | `{token}`（直接用 token 当 key） | Hash | `UserDTO` 转 Map（值均为字符串） | 30分钟；请求续期 | `UserServiceImpl.login()` 写；`CommonIntercceptors` 读+续期 |
| 店铺缓存 | `cache:shop:{id}` | String | 普通：`Shop` JSON；热点：`RedisData{data,expireTime}` JSON | 普通：30分钟；逻辑过期：无TTL | `ShopServiceImpl.queryByID()`/`CacheService.*` |
| 店铺互斥锁 | `lock:shop:{id}` | String | value=`"1"` | 10秒 | `CacheService.tryLock()/unlock()` |
| 店铺类型列表 | `shopType:` | List | list element：`ShopType` JSON | 30分钟 | `ShopTypeServiceImpl.listType()` |
| 附近商铺 | `shop:geo:{typeId}` | GEO(ZSet) | member=`shopId`，坐标=`x,y` | 无 | `ShopServiceImpl.queryShopByType()` 读 |
| 关注关系 | `{userId}:follows:` | Set | member=`followUserId` | 无 | `FollowServiceImpl.follow()` 写；`commonFollow()` 交集 |
| 关注流Feed | `feed:{userId}:` | ZSet | member=`blogId`，score=时间戳(ms) | 无 | `BlogServiceImpl.saveBlog()` 写；`queryBlogsOfFollow()` 读 |
| 博客点赞 | `blog:liked:{blogId}` | ZSet | member=`userId`，score=点赞时间戳(ms) | 无 | `BlogServiceImpl.likeBlog()/likesQuery()` |
| 秒杀库存 | `seckill:stock:{voucherId}` | String(数字) | 应为库存数字（Lua里 `tonumber(get)`） | 无 | Lua/`VoucherOrderServiceImpl` 读写链路 |
| 秒杀去重 | `seckill:order:{voucherId}` | Set | member=`userId` | 无 | `SeckillVoucherOrder.lua` 里 `SISMEMBER/SADD` |
| 秒杀消息流 | `stream.order` | Stream | fields：`userId`,`voucherId`,`orderId` | 无（可自行裁剪） | `SeckillVoucherOrder.lua` XADD；`VoucherOrderServiceImpl` 消费+ACK |
| 用户签到 | `sign:{userId}:{yyyy/MM}` | String(bitmap) | 第 `day-1` 位表示当月第 `day` 天是否签到 | 无 | `UserServiceImpl.Sign()/signCount()` |
| Redisson锁 | `lock:order{userId}`（锁名） | Redisson内部结构 | Redisson 自动管理（hash/续期等） | Redisson策略 | `VoucherOrderServiceImpl.handleVoucherOrder()` |

## 2. 逐项结构说明

### 2.1 登录验证码（String）
- Key：`login:code:{phone}`
- Value：6 位验证码
- TTL：`LOGIN_CODE_TTL = 2` 分钟
- 代码：`UserServiceImpl.sendCode()` / `UserServiceImpl.login()`

### 2.2 登录态（Hash）
- Key：`{token}`（注意：`RedisConstants.LOGIN_USER_KEY` 定义了 `login:token:` 但当前代码未使用）
- Hash fields：`UserDTO` 字段（如 `id`、`nickName`、`icon` 等），值统一转为字符串
- TTL：30 分钟；在 `CommonIntercceptors` 里每次请求续期 30 分钟
- 代码：
  - 写：`UserServiceImpl.login()` -> `opsForHash().putAll(token, map)` + `expire(token, 30min)`
  - 读：`CommonIntercceptors.preHandle()` -> `opsForHash().entries(uuid)` + `expire(uuid, 30min)`

### 2.3 店铺缓存（String，普通TTL/逻辑过期两种）
- Key：`cache:shop:{id}`
- 普通缓存 Value：`Shop` 的 JSON
- 逻辑过期 Value：`RedisData` JSON：
  - `data`: Shop
  - `expireTime`: LocalDateTime
- TTL：
  - 普通：30 分钟（`CACHE_SHOP_TTL`）
  - 逻辑过期：Redis不设TTL，仅用 `expireTime` 控制
- 防穿透空值：`CacheService.queryPassThrough()` 使用空字符串 `""` 表示空值（但你当前传入的 TTL 是 30 分钟，而不是 `CACHE_NULL_TTL=2` 分钟）

### 2.4 店铺互斥锁（String）
- Key：`lock:shop:{id}`
- Value：`"1"`
- TTL：10 秒
- 用途：热点 key 逻辑过期时的缓存重建互斥

### 2.5 店铺类型列表（List）
- Key：`shopType:`
- List element：`ShopType` JSON
- TTL：30 分钟

### 2.6 附近商铺（GEO）
- Key：`shop:geo:{typeId}`
- member：`shopId`（字符串）
- 用法：`GEOSEARCH` + `includeDistance` + `limit`
- 备注：当前代码只看到查询逻辑，未看到导入/预热 GEO 数据的写入逻辑

### 2.7 关注关系（Set）
- Key：`{userId}:follows:`
- member：被关注用户 id（字符串）
- 用法：关注/取关 `SADD/SREM`，共同关注 `SINTER`

### 2.8 Feed 收件箱（ZSet）
- Key：`feed:{userId}:`
- member：`blogId`
- score：毫秒时间戳
- 用法：滚动分页 `ZREVRANGEBYSCORE WITHSCORES` + offset + count

### 2.9 博客点赞（ZSet）
- Key：`blog:liked:{blogId}`
- member：`userId`
- score：点赞时间戳
- 用法：
  - 是否点赞：`ZSCORE`
  - 点赞/取消：`ZADD/ZREM`
  - 点赞列表：`ZRANGE`

### 2.10 秒杀（库存/去重/Stream）
- 库存 Key：`seckill:stock:{voucherId}`
  - Lua 脚本按“数字库存”使用：`tonumber(redis.call('get', stockKey))`
- 去重 Key：`seckill:order:{voucherId}`（Set）
  - Lua 里 `SISMEMBER` 判断一人一单，`SADD` 写入
- Stream：`stream.order`
  - fields：`userId`,`voucherId`,`orderId`
  - 生产：`SeckillVoucherOrder.lua` -> `XADD`
  - 消费：`VoucherOrderServiceImpl` -> `XREADGROUP` + `XACK`；异常读取 pending（从 `0`）

#### 重要备注（你当前代码存在“结构不一致”风险）
`VoucherServiceImpl.addSeckillVoucher()` 把 `seckill:stock:{voucherId}` 写成了 `SeckillVoucher` 的 JSON：
- 但 Lua 脚本把它当“数字库存”使用（`tonumber(get)` + `incrby -1`）。
- 这会导致 Lua 里 `tonumber` 失败或库存扣减异常。

### 2.11 签到（bitmap）
- Key：`sign:{userId}:{yyyy/MM}`
- 结构：用 bit 表示当月每天是否签到
- 写：`SETBIT(key, day-1, true)`
- 读：`BITFIELD` 取低 `dayOfMonth` 位后在 Java 里统计

## 3. 代码入口索引（便于快速定位）
- 登录：`UserServiceImpl`、`CommonIntercceptors`
- 店铺缓存/GEO：`ShopServiceImpl`、`CacheService`
- 店铺类型List：`ShopTypeServiceImpl`
- 关注/共同关注：`FollowServiceImpl`
- 点赞/Feed：`BlogServiceImpl`
- 秒杀Lua+Stream：`SeckillVoucherOrder.lua`、`VoucherOrderServiceImpl`、`VoucherServiceImpl`
- 签到bitmap：`UserServiceImpl`




