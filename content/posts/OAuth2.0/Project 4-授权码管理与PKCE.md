---
title: Project 4 - 授权码管理与 PKCE
date: 2025-01-29
categories:
  - Java
tags:
  - OAuth
---
# 实验目标

本周我们将实现 OAuth 2.0 的核心：**授权码流程**。完成本实验后，你将能够：

1. ✅ 理解授权码在 OAuth 2.0 流程中的关键作用
2. ✅ 实现授权码的创建和一次性使用验证
3. ✅ 理解授权码被截获的安全威胁
4. ✅ 实现 PKCE（Proof Key for Code Exchange）安全增强
5. ✅ 掌握 SHA-256 哈希和 Base64 URL 编码

---

# 理论背景

## 1. 授权码流程详解

### 1.1 为什么需要授权码？

OAuth 2.0 使用**两步验证**而非直接颁发访问令牌，主要出于安全考虑：

```
❌ 不安全的单步流程（隐式授权）：
用户登录 → SSO 直接返回访问令牌到浏览器
问题：访问令牌暴露在浏览器历史记录、日志中

✅ 安全的两步流程（授权码）：
步骤 1：用户登录 → SSO 返回临时授权码到浏览器
步骤 2：客户端后端用授权码 + 密钥换取访问令牌
优势：访问令牌只在后端传输，永不经过浏览器
```

### 1.2 授权码的生命周期

```
时间轴：
T0: 用户完成登录，SSO 生成授权码
    授权码特性：
    - 随机生成（UUID 或密码学随机数）
    - 生命周期极短（5 分钟）
    - 一次性使用（use once and destroy）

T0+30s: 客户端用授权码换取令牌
        授权码被立即销毁

T0+5min: 授权码自动过期
         （即使未被使用）
```

---

## 2. 授权码截获攻击与 PKCE

### 2.1 授权码截获攻击场景

```
攻击场景：恶意应用截获授权码

正常流程：
1. 合法 App 发起授权请求
2. 用户登录，SSO 重定向到 callback URL
3. 恶意 App 拦截重定向（通过注册相同的 URL Scheme）
4. 恶意 App 获取授权码
5. 恶意 App 用授权码 + 客户端密钥换取令牌 ← 问题！

在移动应用中，客户端密钥无法安全存储
（反编译即可获取）
```

### 2.2 PKCE 工作原理

PKCE（读作 "pixy"）通过动态密钥解决此问题：

```
客户端侧（每次授权请求都生成新密钥）：
1. 生成随机字符串 code_verifier（43-128 个字符）
   例如：dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk

2. 计算哈希 code_challenge = BASE64URL(SHA256(code_verifier))
   例如：E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM

3. 授权请求携带：
   - code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
   - code_challenge_method=S256

服务器侧：
1. 存储授权码时保存 code_challenge 和 method

2. 令牌请求时验证：
   - 客户端提供原始 code_verifier
   - 服务器重新计算哈希
   - 比对是否与存储的 code_challenge 一致

安全性：
✅ 即使恶意 App 截获授权码，也无法获取 code_verifier
✅ code_verifier 只存在于合法 App 的内存中
✅ 无需客户端密钥，适合移动/SPA 应用
```

### 2.3 PKCE 实现细节

**关键算法**：

```java
// 1. 生成 code_verifier（客户端实现）
SecureRandom random = new SecureRandom();
byte[] bytes = new byte[32];
random.nextBytes(bytes);
String codeVerifier = Base64.getUrlEncoder()
    .withoutPadding()
    .encodeToString(bytes);

// 2. 计算 code_challenge（客户端实现）
MessageDigest digest = MessageDigest.getInstance("SHA-256");
byte[] hash = digest.digest(codeVerifier.getBytes(StandardCharsets.US_ASCII));
String codeChallenge = Base64.getUrlEncoder()
    .withoutPadding()
    .encodeToString(hash);

// 3. 验证（服务器端实现）
public boolean verifyPKCE(String codeVerifier, String storedChallenge) {
    String computedChallenge = computeChallenge(codeVerifier);
    return computedChallenge.equals(storedChallenge);
}
```

**Base64 URL 编码注意事项**：

```java
// ❌ 标准 Base64（RFC 4648 Section 4）
Base64.getEncoder().encodeToString(bytes);
// 可能包含 +、/、= 字符，不适合 URL

// ✅ Base64 URL（RFC 4648 Section 5）
Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
// 使用 -、_ 替代 +、/，去除 = 填充
```

---

# 实验任务

## 文件位置

`smart-sso-server/src/main/java/com/smart/sso/server/code/LocalCodeManager.java`

## 任务 1：初始化缓存

### 实现构造函数

```java
public LocalCodeManager(int timeout) {
    this.timeout = timeout;
  
    // TODO: 初始化授权码缓存
    // 提示：
    // - 使用 CacheBuilder.newBuilder()
    // - 设置过期时间（expireAfterWrite），建议 5 分钟
    // - 设置最大容量（maximumSize），防止内存溢出
    // - 添加 RemovalListener 记录授权码被移除的日志
    // 思考：为什么授权码的过期时间要远短于 TGT？
  
    // TODO: 记录初始化日志
    // 提示：记录过期时间
}
```

**设计要点**：

- 授权码过期时间应远短于 TGT（建议 5 分钟）
- 使用 `RemovalListener` 记录审计日志

---

## 任务 2：实现核心方法

### 2.1 实现 `create` 方法

```java
@Override
public void create(String code, CodeContent content) {
    // TODO 1: 验证参数
    // 提示：检查 code 和 content 是否为 null 或空
  
    // TODO 2: 存储到缓存
    // 提示：使用 codeCache.put() 方法
  
    // TODO 3: 记录日志
    // 提示：
    // - 记录授权码（部分）、用户 ID、应用 ID
    // - 记录是否使用了 PKCE（通过检查 codeChallenge 是否为 null）
}
```

**安全考虑**：

- 授权码应该是不可预测的随机字符串
- 建议使用 UUID 或 `SecureRandom`

---

### 2.2 实现 `getAndRemove` 方法（关键！）

```java
@Override
public CodeContent getAndRemove(String code) {
    // TODO 1: 原子性获取并移除
    // 提示：
    // - 先使用 getIfPresent() 获取内容
    // - 立即调用 invalidate() 使授权码失效
    // 思考：为什么必须先 get 再 invalidate？
  
    // TODO 2: 处理授权码不存在的情况
    // 提示：
    // - 如果内容为 null，记录警告日志
    // - 返回 null
  
    // TODO 3: 记录使用日志
    // 提示：记录授权码已被使用并销毁
  
    // TODO 4: 返回内容
}
```

**关键原则**：

```
一次性使用（Use Once）：
- getAndRemove 被调用后，授权码立即失效
- 第二次调用返回 null
- 防止重放攻击（Replay Attack）
```

**竞态条件思考**：

```java
// 场景：两个请求同时使用同一个授权码
// 请求 1: getIfPresent(code) → 返回 content
// 请求 2: getIfPresent(code) → 返回 content ← 问题！

// 如何解决这个问题？
// 提示：考虑 Guava Cache 的原子性操作
```

---

### 2.3 实现 `validatePKCE` 方法

```java
@Override
public boolean validatePKCE(CodeContent content, String codeVerifier) {
    // TODO 1: 检查是否需要验证 PKCE
    // 提示：
    // - 获取存储的 codeChallenge 和 codeChallengeMethod
    // - 如果 codeChallenge 为 null 或空，说明未使用 PKCE
    // - 返回 true（向后兼容，允许不使用 PKCE）
  
    // TODO 2: 验证 verifier 不能为空
    // 提示：
    // - 如果授权码要求 PKCE，但未提供 code_verifier
    // - 记录警告日志
    // - 返回 false
  
    // TODO 3: 根据 method 计算 challenge
    // 提示：
    // - 如果 method 是 "S256"，使用 SHA-256 哈希
    // - 如果 method 是 "plain"，直接使用 verifier
    // - 其他 method 不支持，返回 false
  
    // TODO 3.1: 实现 S256 方法
    // 提示：
    // - 使用 MessageDigest.getInstance("SHA-256")
    // - 对 codeVerifier 的 ASCII 字节进行哈希
    // - 使用 Base64.getUrlEncoder().withoutPadding() 编码
    // - 处理 NoSuchAlgorithmException
  
    // TODO 3.2: 实现 plain 方法
    // 提示：直接将 codeVerifier 作为 computedChallenge
  
    // TODO 4: 比对 challenge
    // 提示：
    // - 使用 equals() 比较 computedChallenge 和 storedChallenge
    // - 记录验证成功或失败的日志
    // - 返回比对结果
}
```

**算法提示**：

```
支持的 code_challenge_method：

1. S256（推荐）：
   步骤：
   1. 将 code_verifier 转换为 ASCII 字节
   2. 使用 SHA-256 哈希
   3. 使用 Base64 URL 编码（无填充）
   
2. plain（不推荐）：
   直接使用 code_verifier
   
思考：为什么 plain 方法不安全？
```

---

# 测试与验证

## 1. 单元测试

### 1.1 运行测试

```bash
# 在 IDEA 中右键运行
src/test/java/com/smart/sso/server/code/LocalCodeManagerTest.java
```

### 1.2 自己编写测试用例

```java
@Test
public void testCreateAndGetOnce() {
    // TODO: 测试基本创建和一次性使用
    // 验证：
    // 1. 第一次使用成功
    // 2. 第二次使用返回 null
}

@Test
public void testCodeExpiration() throws InterruptedException {
    // TODO: 测试授权码自动过期
    // 提示：需要等待超过过期时间
}

@Test
public void testPKCE_S256_Success() throws Exception {
    // TODO: 测试 PKCE S256 验证成功
    // 步骤：
    // 1. 生成 code_verifier
    // 2. 计算 code_challenge（使用 SHA-256）
    // 3. 创建授权码
    // 4. 验证
}

@Test
public void testPKCE_WrongVerifier() {
    // TODO: 测试错误的 verifier
    // 预期：验证失败
}

@Test
public void testNoPKCE() {
    // TODO: 测试不使用 PKCE 的情况
    // 预期：跳过验证，返回 true
}
```

---

## 2. 手动测试

### 2.1 测试完整授权流程

```bash
# 步骤 1：模拟授权请求（带 PKCE）
# TODO: 客户端生成 verifier 和 challenge
# 提示：参考理论背景中的算法示例

# 步骤 2：用户登录后创建授权码
# TODO: POST /api/code/create
# 提示：包含 codeChallenge 和 codeChallengeMethod

# 步骤 3：客户端用授权码换取令牌
# TODO: POST /oauth/token
# 提示：包含 code_verifier 参数

# 步骤 4：再次使用同一授权码
# TODO: 重复步骤 3
# 预期：失败，返回 invalid_grant 错误
```

---

# 思考与拓展

## 深入思考

1. **一次性使用原则**：

   - 为什么授权码必须一次性使用？
   - 如果允许重复使用会有什么安全风险？
2. **PKCE 的必要性**：

   - PKCE 主要解决什么问题？
   - 哪些类型的客户端必须使用 PKCE？
3. **Base64 URL 编码**：

   - 为什么要使用 Base64 URL 而不是标准 Base64？
   - 为什么要去除填充（withoutPadding）？

## 拓展实验

1. **实现授权码绑定**：

   - 将授权码绑定到客户端 IP 地址
   - 防止授权码被其他 IP 使用
2. **实现动态过期时间**：

   - 根据客户端类型调整授权码过期时间
   - 移动应用可以适当延长
3. **实现 PKCE 强制策略**：

   - 特定类型的客户端强制使用 PKCE
   - 拒绝不使用 PKCE 的授权请求

---

# 常见问题 (FAQ)

### Q1：为什么授权码必须一次性使用？

A1：
// TODO: 自己思考并回答
// 提示：考虑重放攻击的场景

### Q2：PKCE 的 S256 和 plain 有什么区别？

A2：
// TODO: 自己思考并回答
// 提示：考虑安全性和适用场景

### Q3：如何生成安全的授权码？

A3：

```java
// TODO: 实现生成授权码的方法
// 提示：
// 方法 1：使用 UUID
// 方法 2：使用 SecureRandom + Base64 URL 编码
```

### Q4：PKCE 是否替代客户端密钥？

A4：
// TODO: 自己思考并回答
// 提示：考虑不同类型客户端的安全需求

---

# 总结

完成本实验后，你应该：

- ✅ 深入理解授权码的作用和生命周期
- ✅ 掌握一次性使用原则的实现
- ✅ 理解 PKCE 如何防止授权码截获
- ✅ 掌握 SHA-256 哈希和 Base64 URL 编码
- ✅ 通过所有单元测试
- ✅ 为下周的令牌颁发做好准备

**下周预告**：令牌颁发和刷新令牌轮换 - OAuth 2.0 的最核心部分！

---

**预计完成时间**：8-10 小时
**难度等级**：⭐⭐⭐⭐☆
