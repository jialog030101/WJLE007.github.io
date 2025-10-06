---
title: Project 3 - 全局会话管理 (TGT)
date: 2025-07-22
categories:
  - Java
tags:
  - OAuth
 
---
# 实验目标

本周我们将实现单点登录的核心机制：**票据授予票据（TGT）**管理。完成本实验后，你将能够：

1. ✅ 理解 TGT 在 SSO 系统中的关键作用
2. ✅ 使用缓存实现会话的创建、检索和移除
3. ✅ 实现基于 TTL 的会话自动过期机制
4. ✅ 实现滑动窗口过期策略（提升用户体验）
5. ✅ 限制每个用户的并发会话数量（安全增强）
6. ✅ 理解会话级联失效机制（为单点登出做准备）

---

# 理论背景

## 1. 什么是票据授予票据 (TGT)？

### 1.1 TGT 的定义

TGT（Ticket Granting Ticket）是 SSO 系统中的**全局会话令牌**，类似于传统 Web 应用中的 Session ID，但具有更强的安全性和跨应用能力。

**关键特性**：

- **唯一性**：每个 TGT 对应一个用户会话
- **安全性**：通常使用 UUID 或密码学随机数生成
- **时效性**：具有明确的过期时间（如 30 分钟无操作）
- **跨域性**：可在多个子应用间共享

### 1.2 TGT 在 OAuth 2.0 流程中的作用

```
用户首次登录流程：
1. 用户访问应用 A → 重定向到 SSO 登录页
2. 用户输入账号密码 → SSO 验证成功
3. SSO 创建 TGT → 将 TGT ID 写入浏览器 Cookie
4. SSO 颁发授权码 → 应用 A 获取访问令牌

用户再次访问应用 B（单点登录体现）：
1. 用户访问应用 B → 重定向到 SSO
2. 浏览器自动携带 TGT Cookie → SSO 验证 TGT 有效
3. SSO 直接颁发授权码 → 应用 B 获取访问令牌
   （无需用户再次输入密码！）
```

---

## 2. 会话管理策略

### 2.1 生存时间 (TTL) 策略

**固定过期时间**：

```java
// 创建会话时设定过期时间
TGT tgt = new TGT();
tgt.setExpireTime(now + 30 minutes);

// 30 分钟后无论用户是否活跃，会话一律过期
```

**优点**：

- 实现简单
- 可预测的资源回收时间

**缺点**：

- 用户体验差（可能在操作中途被登出）
- 不适合长时间交互的应用

### 2.2 滑动窗口过期策略

**动态刷新过期时间**：

```java
// 每次用户操作时刷新过期时间
public void refresh(String tgt) {
    TGTContent content = cache.get(tgt);
    cache.put(tgt, content);  // 重新写入，重置 TTL
}
```

**优点**：

- 用户体验好（活跃用户不会被登出）
- 兼顾安全性（非活跃会话仍会过期）

**实现要点**：

- 使用支持 `expireAfterWrite` 的缓存（如 Guava Cache）
- 每次 `put` 操作都会重置过期计时器

### 2.3 并发会话限制

**安全场景**：

```
用户在办公室电脑登录 → 创建会话 1
用户在家里电脑登录   → 创建会话 2
...
用户在第 6 台设备登录 → 踢掉最早的会话 1
```

**实现方案**：

- 为每个用户维护一个 TGT 队列（FIFO）
- 队列满时，移除最旧的 TGT
- 使用 `ConcurrentLinkedQueue` 保证线程安全

---

## 3. 会话级联失效机制

### 3.1 为什么需要级联失效？

```
场景：用户从应用 A 登出

如果不级联失效：
- 应用 A 的访问令牌失效
- 但应用 B、C 的访问令牌仍然有效 ❌

实现级联失效后：
- TGT 被移除
- 所有关联的访问令牌和刷新令牌全部失效 ✅
- 真正的"单点登出"
```

### 3.2 实现机制

```java
// 使用 Guava Cache 的 RemovalListener
CacheBuilder.newBuilder()
    .removalListener(notification -> {
        String tgt = notification.getKey();
        // TGT 被移除时，自动触发
        tokenManager.removeByTgt(tgt);
    })
    .build();
```

---

# 实验任务

## 文件位置

`smart-sso-server/src/main/java/com/smart/sso/server/session/LocalTicketGrantingTicketManager.java`

## 任务 1：初始化缓存

### 1.1 初始化主 TGT 缓存

在构造函数中添加：

```java
public LocalTicketGrantingTicketManager(int timeout, TokenManager tokenManager) {
    this.timeout = timeout;
    this.tokenManager = tokenManager;
  
    // TODO 1: 初始化 TGT 缓存
    // 提示：
    // - 使用 CacheBuilder.newBuilder()
    // - 设置过期时间（expireAfterWrite），单位为分钟
    // - 设置最大容量（maximumSize），建议 100000
    // - 添加 RemovalListener 处理 TGT 被移除时的回调
  
    // TODO 1.1: 实现 RemovalListener 的 onRemoval 方法
    // 提示：
    // - 获取被移除的 TGT 和移除原因（RemovalCause）
    // - 记录日志
    // - 调用 tokenManager.removeByTgt() 实现级联失效
    // 思考：为什么需要检查 tokenManager 不为 null？
}
```

### 1.2 初始化用户会话队列缓存

```java
// TODO 2: 初始化用户 TGT 队列缓存
// 提示：
// - 过期时间应该比 TGT 稍长（timeout + 10 分钟）
// - 设置最大容量（建议 50000）
// 思考：为什么这个缓存的过期时间要比 TGT 长？
```

**设计思考**：

- `userTgtCache` 的过期时间为什么要 > `tgtCache`？
- 提示：考虑并发会话限制的竞态条件

---

## 任务 2：实现核心方法

### 2.1 实现 `create` 方法

```java
@Override
public void create(String tgt, TicketGrantingTicketContent content) {
    // TODO 1: 处理并发会话限制
    // 提示：
    // - 从 content 获取 userId
    // - 从 userTgtCache 获取该用户的 TGT 队列
    // - 如果队列不存在，创建新的 ConcurrentLinkedQueue
  
    // TODO 2: 检查会话数量是否超限
    // 提示：
    // - 检查队列大小是否 >= MAX_SESSIONS_PER_USER
    // - 如果超限，使用 poll() 移除最旧的 TGT
    // - 调用 tgtCache.invalidate() 触发级联失效
    // - 记录警告日志
  
    // TODO 3: 添加新会话到队列
    // 提示：
    // - 使用 offer() 方法添加新 TGT
    // - 更新 userTgtCache
  
    // TODO 4: 存储 TGT
    // 提示：使用 tgtCache.put() 方法
  
    // TODO 5: 记录日志
    // 提示：记录用户 ID、TGT、当前会话数
}
```

**设计提示**：

- 使用 `ConcurrentLinkedQueue` 保证线程安全
- `offer` 和 `poll` 是队列的 FIFO 操作

### 2.2 实现 `get` 方法

```java
@Override
public TicketGrantingTicketContent get(String tgt) {
    // TODO: 从缓存中检索 TGT
    // 提示：使用 getIfPresent() 方法
    // 思考：为什么返回 null 而不抛出异常？
}
```

### 2.3 实现 `remove` 方法

```java
@Override
public void remove(String tgt) {
    // TODO 1: 先获取内容（用于清理用户队列）
    // 提示：使用 getIfPresent() 方法
  
    // TODO 2: 从主缓存中移除
    // 提示：使用 invalidate() 方法，这会触发 RemovalListener
  
    // TODO 3: 从用户队列中移除
    // 提示：
    // - 检查 content 是否为 null
    // - 从 content 获取 userId
    // - 从 userTgtCache 获取队列
    // - 从队列中移除该 TGT
    // - 记录日志
}
```

### 2.4 实现 `refresh` 方法（滑动窗口核心）

```java
@Override
public void refresh(String tgt) {
    // TODO 1: 获取当前内容
    // 提示：使用 getIfPresent() 方法
  
    // TODO 2: 处理 TGT 不存在的情况
    // 提示：
    // - 如果 content 为 null，记录警告日志
    // - 直接返回
  
    // TODO 3: 重新写入缓存（重置过期时间）
    // 提示：使用 put() 方法
    // 思考：为什么重新 put 可以刷新过期时间？
  
    // TODO 4: 记录调试日志
}
```

**关键原理**：

```java
// Guava Cache 的 expireAfterWrite 特性
// put(key, value) → 重置该 key 的过期计时器
// 思考：这如何实现滑动窗口？
```

---

# 测试与验证

## 1. 单元测试

### 1.1 运行测试

```bash
# 在 IDEA 中右键运行
src/test/java/com/smart/sso/server/session/LocalTicketGrantingTicketManagerTest.java
```

### 1.2 自己编写测试用例

```java
@Test
public void testCreateAndGet() {
    // TODO: 测试基本创建和检索
}

@Test
public void testExpiration() throws InterruptedException {
    // TODO: 测试自动过期（TTL）
    // 提示：需要等待超过过期时间
}

@Test
public void testRefresh() throws InterruptedException {
    // TODO: 测试滑动窗口刷新
    // 验证：刷新后 TGT 不会过期
}

@Test
public void testConcurrentSessionLimit() {
    // TODO: 测试并发会话限制
    // 场景：创建超过 MAX_SESSIONS_PER_USER 个会话
    // 验证：最旧的会话被踢掉
}
```

---

## 2. 手动测试

### 2.1 测试会话创建

```bash
# TODO: 使用 Postman 或 curl 测试会话创建
# POST /api/session/create
# 提示：构造包含 userId 和 username 的 JSON 请求体
```

### 2.2 测试会话验证

```bash
# TODO: 测试会话验证
# GET /api/session/validate?tgt=YOUR_TGT_HERE
# 预期：有效的 TGT 返回 valid:true
```

---

# 思考与拓展

## 深入思考

1. **滑动窗口机制**：

   - 滑动窗口会导致会话永不过期吗？
   - 如何平衡用户体验和安全性？
2. **并发会话限制**：

   - 为什么需要限制每个用户的会话数？
   - 如何选择合适的限制数量？
3. **缓存过期时间**：

   - 为什么 `userTgtCache` 要比 `tgtCache` 过期时间长？
   - 如果不这样设计会有什么问题？

## 拓展实验

1. **实现设备指纹**：

   - 在 TGT 中记录设备信息
   - 支持"踢掉其他设备"功能
2. **实现分布式会话**：

   - 将 Guava Cache 替换为 Redis
   - 支持集群部署
3. **实现会话活动监控**：

   - 记录每个 TGT 的最后活动时间
   - 统计用户的活跃度

---

# 常见问题 (FAQ)

### Q1：为什么使用 Guava Cache 而不是 Redis？

A1：
// TODO: 自己思考并回答
// 提示：考虑单机部署 vs 分布式部署

### Q2：`RemovalListener` 何时被触发？

A2：
// TODO: 列举触发场景
// 提示：至少包含 3 种情况

### Q3：滑动窗口会导致会话永不过期吗？

A3：
// TODO: 自己思考并回答
// 提示：考虑用户停止操作的情况

---

# 总结

完成本实验后，你应该：

- ✅ 深入理解 TGT 在 SSO 系统中的核心作用
- ✅ 掌握使用 Guava Cache 实现会话管理
- ✅ 理解滑动窗口过期策略的原理和实现
- ✅ 能够实现并发会话限制
- ✅ 理解会话级联失效机制
- ✅ 通过所有单元测试

**下周预告**：授权码管理和 PKCE 安全增强！

---

**预计完成时间**：10-12 小时
**难度等级**：⭐⭐⭐⭐☆
