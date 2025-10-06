---
title: Project 5 - 令牌颁发与刷新令牌轮换
date: 2025-08-05
categories:
  - Java
tags:
  - OAuth
---
# 实验目标

本周是整个项目的**核心部分**！我们将实现 OAuth 2.0 的引擎：令牌管理系统。完成本实验后，你将能够：

1. ✅ 理解访问令牌和刷新令牌的不同生命周期
2. ✅ 实现令牌的创建、验证和存储
3. ✅ 实现刷新令牌轮换（Refresh Token Rotation）
4. ✅ 实现刷新令牌盗窃检测机制
5. ✅ 理解令牌家族（Token Family）的概念
6. ✅ 掌握令牌与 TGT 的关联管理

---

# 理论背景

## 1. 访问令牌 vs. 刷新令牌

### 1.1 核心区别

| 特性               | 访问令牌 (Access Token) | 刷新令牌 (Refresh Token) |
| ------------------ | ----------------------- | ------------------------ |
| **用途**     | 访问受保护的 API 资源   | 获取新的访问令牌         |
| **生命周期** | 短（15 分钟）           | 长（30 天）              |
| **传输频率** | 每次 API 请求都携带     | 仅在刷新时使用           |
| **泄露风险** | 高（频繁传输）          | 低（使用频率低）         |
| **存储位置** | 客户端内存              | 安全存储（加密）         |
| **撤销成本** | 低（自动过期）          | 高（需要主动撤销）       |

### 1.2 为什么需要两种令牌？

**安全与体验的平衡**：

```
场景 1：仅使用访问令牌
问题：访问令牌过期后，用户必须重新登录
结果：用户体验差 ❌

场景 2：访问令牌永不过期
问题：一旦泄露，攻击者可永久访问
结果：安全性差 ❌

场景 3：访问令牌 + 刷新令牌
优势：
- 访问令牌短期有效，降低泄露风险 ✅
- 刷新令牌长期有效，无需频繁登录 ✅
- 刷新令牌可追踪和撤销 ✅
```

---

## 2. 刷新令牌轮换 (Refresh Token Rotation)

### 2.1 传统方法 vs. 轮换方法

**传统方法**（不推荐）：

```
时间 T0: 客户端用授权码换取令牌
        → 获得 AT-1 和 RT-1

时间 T1: AT-1 过期，客户端用 RT-1 刷新
        → 获得 AT-2，RT-1 仍然有效

时间 T2: AT-2 过期，客户端再次用 RT-1 刷新
        → 获得 AT-3，RT-1 仍然有效

问题：RT-1 可以无限次使用，一旦泄露，攻击者可持续获取新令牌 ❌
```

**轮换方法**（推荐）：

```
时间 T0: 客户端用授权码换取令牌
        → 获得 AT-1 和 RT-1

时间 T1: AT-1 过期，客户端用 RT-1 刷新
        → 获得 AT-2 和 RT-2
        → RT-1 立即失效！

时间 T2: AT-2 过期，客户端用 RT-2 刷新
        → 获得 AT-3 和 RT-3
        → RT-2 立即失效！

优势：
- 每个刷新令牌仅能使用一次 ✅
- 刷新令牌泄露的影响被限制在一次刷新周期内 ✅
- 可以检测刷新令牌盗窃（见下文）✅
```

### 2.2 刷新令牌盗窃检测

**攻击场景**：

```
时间 T0: 合法客户端获得 RT-1

时间 T1: 攻击者窃取 RT-1（通过中间人攻击等）

时间 T2: 攻击者使用 RT-1 刷新
        → 服务器返回 AT-2 和 RT-2
        → RT-1 被标记为已使用

时间 T3: 合法客户端尝试使用 RT-1 刷新 ← 检测点！
        → 服务器发现 RT-1 已被使用
        → 判断为盗窃行为
        → 撤销整个令牌家族（RT-1、RT-2 及所有后代）
        → 强制用户重新登录
```

**检测逻辑**：

```java
if (已使用的刷新令牌被再次使用) {
    // 这是盗窃的强信号
    revokeFamilyRecursively(tokenFamily);
    throw new TokenStolenException();
}
```

---

## 3. 令牌家族 (Token Family)

### 3.1 家族概念

令牌家族是指通过刷新操作产生的一系列令牌的集合：

```
授权码换取令牌：
RT-1 (家族根)
  ├─ AT-1
  └─ 刷新后产生 RT-2
       ├─ AT-2
       └─ 刷新后产生 RT-3
            ├─ AT-3
            └─ ...

所有这些令牌属于同一个家族（由初始 RT-1 标识）
```

### 3.2 家族标识

```java
class TokenContent {
    String accessToken;      // 当前访问令牌
    String refreshToken;     // 当前刷新令牌（即将被轮换）
    String tokenFamily;      // 家族 ID（从根令牌继承）
    Long userId;
    String tgt;              // 关联的全局会话
    LocalDateTime createdAt;
    LocalDateTime expiresAt;
}
```

**家族 ID 的传递**：

```java
// 初始创建时
TokenContent initial = new TokenContent();
initial.setTokenFamily(UUID.randomUUID().toString());  // 新家族

// 刷新时
TokenContent refreshed = new TokenContent();
refreshed.setTokenFamily(oldContent.getTokenFamily());  // 继承家族
```

---

## 4. 令牌与 TGT 的关联

### 4.1 为什么需要关联？

```
单点登出场景：
用户从应用 A 登出 → TGT 被移除

如果没有关联：
- 应用 B、C 的访问令牌仍然有效 ❌
- 刷新令牌仍然可以获取新的访问令牌 ❌

建立关联后：
- TGT 被移除时触发级联
- 自动撤销所有关联的刷新令牌 ✅
- 访问令牌过期后无法刷新 ✅
- 真正的单点登出 ✅
```

### 4.2 反向索引设计

```java
// 主索引：刷新令牌 → 令牌内容
Cache<String, TokenContent> refreshTokenCache;

// 反向索引：TGT → 刷新令牌集合
Cache<String, Set<String>> tgtCache;
```

**使用场景**：

```java
// 创建令牌时建立关联
public void create(String refreshToken, TokenContent content) {
    // 存储主数据
    refreshTokenCache.put(refreshToken, content);
  
    // 建立反向索引
    String tgt = content.getTgt();
    Set<String> tokens = tgtCache.getOrDefault(tgt, new HashSet<>());
    tokens.add(refreshToken);
    tgtCache.put(tgt, tokens);
}

// TGT 被移除时撤销所有令牌
public void removeByTgt(String tgt) {
    Set<String> tokens = tgtCache.get(tgt);
    for (String refreshToken : tokens) {
        remove(refreshToken);  // 级联删除
    }
}
```

---

# 实验任务

## 文件位置

`smart-sso-server/src/main/java/com/smart/sso/server/token/LocalTokenManager.java`

## 前置任务：切换到真实实现

### 修改配置类

编辑 `SmartSsoServerConfiguration.java`：

```java
// filepath: smart-sso-server/src/main/java/com/smart/sso/server/config/SmartSsoServerConfiguration.java
@Configuration
public class SmartSsoServerConfiguration {
  
    @Bean
    public TokenManager tokenManager(
            @Value("${sso.timeout.access-token}") int accessTokenTimeout,
            @Value("${sso.timeout.refresh-token}") int refreshTokenTimeout) {
      
        // TODO: 注释掉虚拟实现，启用真实实现
        // 提示：
        // - 注释掉 return new DummyTokenManager();
        // - 取消注释 return new LocalTokenManager(...);
    }
  
    // ...existing code...
}
```

---

## 任务 1：初始化缓存

### 实现构造函数

```java
public LocalTokenManager(int accessTokenTimeout, int refreshTokenTimeout) {
    this.accessTokenTimeout = accessTokenTimeout;
    this.refreshTokenTimeout = refreshTokenTimeout;
  
    // TODO 1: 初始化访问令牌缓存
    // 提示：
    // - 使用 CacheBuilder.newBuilder()
    // - 设置过期时间（expireAfterWrite）
    // - 设置最大容量（maximumSize）
    // 思考：为什么访问令牌用分钟，刷新令牌用天？
  
    // TODO 2: 初始化刷新令牌缓存
    // 提示：
    // - 添加 RemovalListener 记录审计日志
    // - 日志应包含：令牌 ID、移除原因（cause）
  
    // TODO 3: 初始化 TGT 反向索引缓存
    // 提示：
    // - 过期时间应该比刷新令牌稍长（为什么？）
    // - 建议：refreshTokenTimeout + 1 天
  
    // TODO 4: 初始化已撤销的令牌家族缓存
    // 提示：
    // - 用于盗窃检测
    // - 过期时间与刷新令牌相同
    // - 最大容量可以小一些（如 10000）
  
    // TODO 5: 记录初始化日志
    // 提示：记录访问令牌和刷新令牌的超时时间
}
```

**思考题**：

1. 为什么 `tgtCache` 的过期时间要比 `refreshTokenCache` 长？
2. `revokedFamilies` 缓存会无限增长吗？

---

## 任务 2：实现核心方法

### 2.1 实现 `create` 方法

```java
@Override
public void create(String refreshToken, TokenContent content) {
    // TODO 1: 验证参数
    // 提示：检查 refreshToken 和 content 是否为 null
  
    // TODO 2: 存储访问令牌
    // 提示：从 content 中获取 accessToken，存入 accessTokenCache
  
    // TODO 3: 存储刷新令牌
    // 提示：存入 refreshTokenCache
  
    // TODO 4: 建立 TGT 反向索引
    // 提示：
    // - 从 content 中获取 TGT
    // - 从 tgtCache 获取或创建令牌集合（Set）
    // - 将刷新令牌添加到集合
    // - 更新缓存（为什么需要 put 回去？）
    // 思考：为什么使用 ConcurrentHashMap.newKeySet()？
  
    // TODO 5: 记录日志
    // 提示：记录用户 ID、令牌家族 ID、TGT
}
```

**设计思考**：

- 为什么需要反向索引（TGT → 刷新令牌集合）？
- 如何保证多线程环境下的安全性？

---

### 2.2 实现 `getByAccessToken` 方法

```java
@Override
public TokenContent getByAccessToken(String accessToken) {
    // TODO: 从访问令牌缓存中检索
    // 提示：使用 getIfPresent() 方法
    // 思考：为什么不抛出异常而是返回 null？
}
```

---

### 2.3 实现 `get` 方法

```java
@Override
public TokenContent get(String refreshToken) {
    // TODO: 从刷新令牌缓存中检索
    // 提示：使用 getIfPresent() 方法
}
```

---

### 2.4 实现 `refresh` 方法（核心逻辑！）

```java
@Override
public TokenContent refresh(String refreshToken) {
    // TODO 1: 原子性获取并移除旧刷新令牌
    // 提示：
    // - 使用 getIfPresent() 获取 TokenContent
    // - 立即调用 invalidate() 使其失效
    // 思考：为什么要立即失效？（防止什么攻击？）
  
    // TODO 2: 盗窃检测 - 检查令牌家族是否已被撤销
    // 提示：
    // - 从 TokenContent 获取 tokenFamily
    // - 检查 revokedFamilies 缓存
    // - 如果已撤销，返回 null 并记录错误日志
  
    // TODO 3: 检查旧刷新令牌是否已被使用过（重放攻击检测）
    // 提示：
    // - 如果步骤 1 返回 null，说明令牌已被使用或不存在
    // - 但家族未被撤销，说明是重放攻击
    // - 标记家族为被盗（revokedFamilies.put）
    // - 撤销整个家族（调用 revokeFamily）
    // - 返回 null 并记录严重错误日志
  
    // TODO 4: 使旧访问令牌失效
    // 提示：从 TokenContent 获取 oldAccessToken，invalidate
  
    // TODO 5: 生成新的访问令牌和刷新令牌
    // 提示：调用辅助方法 createAccessToken() 和 createRefreshToken()
  
    // TODO 6: 构造新的令牌内容（继承家族 ID）
    // 提示：
    // - 创建新的 TokenContent 对象
    // - 复制：userId、tgt、tokenFamily（关键！）
    // - 设置：新的访问令牌、刷新令牌
    // - 设置：createdAt、expiresAt
  
    // TODO 7: 存储新令牌
    // 提示：调用 create() 方法
  
    // TODO 8: 从 TGT 反向索引中移除旧刷新令牌
    // 提示：
    // - 获取 TGT 和令牌集合
    // - 从集合中移除旧刷新令牌
    // - 更新缓存
  
    // TODO 9: 记录成功日志
    // 提示：记录用户 ID、家族 ID、旧令牌和新令牌（部分）
  
    // TODO 10: 返回新的 TokenContent
}

// 辅助方法：生成访问令牌
private String createAccessToken() {
    // TODO: 生成安全的随机令牌
    // 提示：使用 UUID 或 SecureRandom + Base64
}

// 辅助方法：生成刷新令牌
private String createRefreshToken() {
    // TODO: 生成安全的随机令牌
}

// 辅助方法：撤销整个令牌家族
private void revokeFamily(String tokenFamily) {
    // TODO: 遍历所有刷新令牌，找到同一家族的并撤销
    // 提示：
    // - 遍历 refreshTokenCache.asMap()
    // - 比较 TokenContent 的 tokenFamily
    // - 调用 remove() 方法
    // - 记录警告日志
    // 思考：这个实现的性能如何？如何优化？
}
```

**关键步骤解析**：

1. **原子性获取并移除**：为什么要立即失效？
2. **盗窃检测**：家族被标记为被盗后会发生什么？
3. **重放检测**：如何区分正常过期和重放攻击？
4. **家族继承**：为什么新令牌要继承旧令牌的家族 ID？

---

### 2.5 实现 `remove` 方法

```java
@Override
public void remove(String refreshToken) {
    // TODO 1: 获取内容（用于清理反向索引）
    // 提示：使用 getIfPresent()
  
    // TODO 2: 移除刷新令牌
    // 提示：使用 invalidate()
  
    // TODO 3: 移除关联的访问令牌
    // 提示：从 TokenContent 获取 accessToken，需要判空
  
    // TODO 4: 从 TGT 反向索引中移除
    // 提示：
    // - 获取 TGT 和令牌集合
    // - 从集合中移除刷新令牌
    // - 如果集合为空，清理整个条目
    // - 否则更新缓存
  
    // TODO 5: 记录日志
}
```

---

### 2.6 实现 `removeByTgt` 方法（单点登出核心）

```java
@Override
public void removeByTgt(String tgt) {
    // TODO 1: 获取该 TGT 关联的所有刷新令牌
    // 提示：从 tgtCache 获取令牌集合
  
    // TODO 2: 遍历并移除所有刷新令牌
    // 提示：
    // - 复制集合（避免并发修改异常）
    // - 遍历复制的集合
    // - 调用 remove() 方法
    // - 计数撤销数量
  
    // TODO 3: 清除 TGT 反向索引
    // 提示：使用 invalidate()
  
    // TODO 4: 记录日志
    // 提示：记录 TGT 和撤销数量
}
```

---

# 测试与验证

## 1. 单元测试

### 1.1 运行测试

```bash
# 在 IDEA 中右键运行
src/test/java/com/smart/sso/server/token/LocalTokenManagerTest.java
```

### 1.2 自己编写测试用例

```java
@Test
public void testCreateAndGet() {
    // TODO: 测试基本创建和检索功能
}

@Test
public void testRefreshTokenRotation() {
    // TODO: 测试刷新令牌轮换
    // 验证：
    // 1. 旧刷新令牌失效
    // 2. 新刷新令牌有效
    // 3. 家族 ID 继承
}

@Test
public void testTokenTheftDetection() {
    // TODO: 测试刷新令牌盗窃检测
    // 场景：
    // 1. 第一次刷新成功
    // 2. 再次使用旧令牌（重放）
    // 3. 验证家族被撤销
}

@Test
public void testRemoveByTgt() {
    // TODO: 测试 TGT 级联失效
}
```

---

## 2. 手动测试

### 2.1 测试完整令牌流程

```bash
# 步骤 1：获取授权码
# TODO: 使用 Postman 请求 /oauth/authorize

# 步骤 2：用授权码换取令牌
# TODO: POST /oauth/token
# 参数：grant_type, code, client_id, client_secret

# 步骤 3：使用访问令牌访问 API
# TODO: GET /api/userinfo
# Header: Authorization: Bearer {access_token}

# 步骤 4：刷新令牌
# TODO: POST /oauth/token
# 参数：grant_type=refresh_token, refresh_token

# 步骤 5：验证旧令牌失效
# TODO: 再次使用旧刷新令牌，应该失败
```

---

# 思考与拓展

## 深入思考

1. **刷新令牌轮换的成本收益**：

   - 轮换增加了多少服务器开销？
   - 安全性提升了多少？
2. **盗窃检测的误报率**：

   - 哪些情况可能导致误报？
   - 如何降低误报率？
3. **令牌家族的管理**：

   - `revokeFamily()` 的性能如何？
   - 如何优化家族撤销？

## 拓展实验

1. **实现宽限期（Grace Period）**：

   - 允许旧刷新令牌在短时间内仍可使用
   - 适用于网络不稳定的场景
2. **实现移动应用的特殊策略**：

   - 检测客户端类型
   - 移动应用延长宽限期
3. **实现令牌使用统计**：

   - 记录每个令牌的使用次数
   - 分析异常使用模式

---

# 总结

完成本实验后,你应该能够：

- ✅ 实现令牌的创建、检索和刷新
- ✅ 掌握刷新令牌轮换机制
- ✅ 实现令牌盗窃检测
- ✅ 理解令牌家族的概念
- ✅ 掌握令牌与 TGT 的关联管理

**下周预告**：令牌撤销与单点登出 - 完成整个 OAuth 2.0 系统！

---

**预计完成时间**：12-15 小时
**难度等级**：⭐⭐⭐⭐⭐
