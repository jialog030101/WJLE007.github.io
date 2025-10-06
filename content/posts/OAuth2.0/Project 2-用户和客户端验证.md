---
title: Project 2 - 用户和客户端验证
date: 2025-01-15
categories:
  - Java
tags:
  - OAuth
---
# 实验目标

本周我们将实现 OAuth 2.0 授权流程的第一道防线：**身份验证**。完成本实验后，你将能够：

1. ✅ 实现安全的用户密码验证（使用 BCrypt）
2. ✅ 实现账户锁定机制（防止暴力破解）
3. ✅ 实现密码过期策略（强制定期更换密码）
4. ✅ 实现客户端应用验证（验证 `client_id`、`client_secret` 和 `redirect_uri`）
5. ✅ 理解重定向 URI 验证的安全重要性

---

# 理论背景

## 1. 用户认证的安全原则

### 1.1 密码存储

**绝对禁止**：以明文或可逆加密方式存储密码

**推荐方案**：使用强单向哈希算法（BCrypt、Argon2、PBKDF2）

**BCrypt 优势**：

- 内置盐值（Salt），每次哈希结果不同
- 计算成本可调（通过 `work factor` 参数）
- 抗彩虹表攻击

示例：

```java
// 生成密码哈希
String hashedPassword = passwordEncoder.encode("userPassword123");
// 输出类似：$2a$10$N9qo8uLOickgx2ZMRZoMye1J9rY7AVMZ8tPNLPZdRs5u5xP5c5fUa

// 验证密码
boolean matches = passwordEncoder.matches("userPassword123", hashedPassword);
// 返回 true
```

### 1.2 账户锁定策略

**目的**：防止暴力破解攻击

**实现方案**：

- 追踪每个用户的连续登录失败次数
- 达到阈值（如 5 次）后锁定账户一段时间（如 10 分钟）
- 成功登录后清除失败计数器

**使用缓存的原因**：

- 高性能：失败计数不需要持久化到数据库
- 自动过期：利用缓存的 TTL 特性实现锁定时间
- 降低数据库压力

### 1.3 密码过期策略

**目的**：降低长期使用同一密码的风险

**常见策略**：

- 密码有效期：90 天、180 天等
- 强制更新：过期后必须修改密码才能登录
- 密码历史：不允许重复使用最近 N 次的密码

---

## 2. 客户端验证的安全原则

### 2.1 为什么需要验证客户端？

OAuth 2.0 中，客户端应用代表用户请求访问。服务器必须确保：

1. 客户端是已注册的合法应用
2. 客户端提供了正确的密钥
3. 重定向 URI 未被篡改

### 2.2 重定向 URI 验证的重要性

**威胁场景**：

```
攻击者构造恶意链接：
https://sso.example.com/oauth/authorize?
  client_id=legitimate-app&
  redirect_uri=https://attacker.com/steal  ← 恶意 URI

用户登录后，授权码被发送到攻击者的服务器！
```

**防御措施**：

- 客户端注册时预先配置 `redirect_uri`
- 授权请求时，服务器必须验证提供的 URI 与注册的 URI **完全匹配**
- 支持多个 URI 时，必须使用精确匹配（不允许通配符或部分匹配）

---

# 实验任务

## 任务 1：实现用户验证逻辑

### 文件位置

`smart-sso-server/src/main/java/com/smart/sso/server/service/impl/UserServiceImpl.java`

### 实现步骤

#### 1.1 初始化账户锁定缓存

在类的顶部添加：

```java
// TODO: 定义账户锁定缓存
// 提示：
// - 使用 Guava CacheBuilder
// - 设置过期时间（建议 10 分钟）
// - 设置最大容量（建议 10000）
// 思考：为什么使用缓存而不是数据库？

// TODO: 定义最大失败次数常量
// 提示：建议设置为 5 次
```

#### 1.2 实现 `validate(String account, String password)` 方法

```java
@Override
public Result<User> validate(String account, String password) {
    // TODO 1: 检查账户是否被锁定
    // 提示：
    // - 从 loginFailureCache 获取失败次数
    // - 如果失败次数 >= MAX_LOGIN_FAILURES，返回锁定错误
  
    // TODO 2: 从数据库查询用户
    // 提示：调用 userMapper.selectByUsername() 方法
    // 思考：用户不存在应该返回什么错误信息？
  
    // TODO 3: 验证密码
    // 提示：
    // - 使用 passwordEncoder.matches() 方法
    // - 如果密码错误，增加失败计数
    // - 计算剩余尝试次数并返回友好提示
  
    // TODO 4: 检查账户状态
    // 提示：检查 user.getIsEnable() 字段
  
    // TODO 5: 检查密码是否过期
    // 提示：
    // - 获取 passwordLastModified 字段
    // - 使用 ChronoUnit.DAYS.between() 计算天数
    // - 如果超过 90 天，返回密码过期错误
  
    // TODO 6: 验证成功，清除失败计数
    // 提示：调用 loginFailureCache.invalidate() 方法
  
    // TODO 7: 返回成功结果
}
```

**安全提示**：

```
登录失败时应该返回什么错误信息？

❌ 不推荐："用户名不存在" 或 "密码错误"
   → 泄露用户名是否存在

✅ 推荐："用户名或密码错误"
   → 不泄露具体信息
```

#### 1.3 实现 `getTokenUser(Long userId)` 方法

```java
@Override
public TokenUser getTokenUser(Long userId) {
    // TODO 1: 查询用户
    // 提示：调用 userMapper.selectById() 方法
  
    // TODO 2: 处理用户不存在的情况
    // 提示：返回 null
  
    // TODO 3: 构造 TokenUser
    // 提示：
    // - 创建 TokenUser 对象
    // - 设置 id、username、name、email 等字段
    // - 注意：不要设置 password 字段！
  
    // TODO 4: 返回结果
}
```

**安全警告**：

```java
// ❌ 危险：将完整 User 对象返回给客户端
return user;  // 包含密码哈希！

// ✅ 安全：只返回必要的字段
TokenUser tokenUser = new TokenUser();
tokenUser.setId(user.getId());
// ... 其他非敏感字段
return tokenUser;
```

---

## 任务 2：实现客户端验证逻辑

### 文件位置

`smart-sso-server/src/main/java/com/smart/sso/server/service/impl/AppServiceImpl.java`

### 实现步骤

#### 2.1 实现 `validate(String appId, String appSecret, String redirectUri)` 方法

```java
@Override
public boolean validate(String appId, String appSecret, String redirectUri) {
    // TODO 1: 查询客户端应用
    // 提示：调用 appMapper.selectByAppId() 方法
    // 思考：应用不存在时应该如何处理？
  
    // TODO 2: 检查应用是否启用
    // 提示：检查 app.getIsEnable() 字段
  
    // TODO 3: 验证客户端密钥
    // 提示：使用 equals() 比较密钥
    // 思考：为什么不使用 BCrypt 验证客户端密钥？
  
    // TODO 4: 验证重定向 URI（最关键的安全检查！）
    // 提示：
    // - 获取注册的 redirectUri（可能包含多个，用逗号分隔）
    // - 分割字符串为数组
    // - 遍历检查是否有完全匹配的 URI
    // - 使用 trim() 处理空格
  
    // TODO 5: 所有检查通过
    // 提示：记录成功日志并返回 true
}
```

**重定向 URI 验证提示**：

```java
// 支持配置多个 URI，用逗号分隔
String registeredUris = app.getRedirectUri();  // "http://app1.com/callback,http://app2.com/callback"

// TODO: 实现精确匹配验证
// 注意：
// 1. 必须完全匹配（不允许部分匹配）
// 2. 不允许通配符
// 3. 区分大小写

// 思考：为什么不能使用 contains() 或 startsWith()？
```

#### 2.2 实现 `getApp(String appId)` 方法

```java
@Override
public App getApp(String appId) {
    // TODO: 从数据库查询应用
    // 提示：调用 appMapper.selectByAppId() 方法
}
```

---

# 测试与验证

## 1. 单元测试

### 1.1 运行用户服务测试

```bash
# 在 IDEA 中右键运行
src/test/java/com/smart/sso/server/service/UserServiceImplTest.java
```

### 1.2 自己编写测试用例

```java
@Test
public void testValidateSuccess() {
    // TODO: 测试正确的用户名和密码
}

@Test
public void testValidateWrongPassword() {
    // TODO: 测试错误的密码
}

@Test
public void testAccountLockout() {
    // TODO: 测试连续失败导致账户锁定
    // 提示：循环调用 5 次 validate()
}

@Test
public void testPasswordExpired() {
    // TODO: 测试密码过期
    // 提示：需要 mock passwordLastModified 字段
}
```

### 1.3 运行客户端服务测试

```bash
# 在 IDEA 中右键运行
src/test/java/com/smart/sso/server/service/AppServiceImplTest.java
```

### 1.4 自己编写客户端测试用例

```java
@Test
public void testValidateSuccess() {
    // TODO: 测试正确的客户端信息
}

@Test
public void testValidateWrongSecret() {
    // TODO: 测试错误的客户端密钥
}

@Test
public void testValidateWrongRedirectUri() {
    // TODO: 测试未注册的重定向 URI
}
```

---

## 2. 手动测试

### 2.1 测试用户验证

```bash
# TODO: 使用 Postman 或 curl 测试
# POST /api/user/validate
# 提示：构造包含 account 和 password 的 JSON 请求体
```

### 2.2 测试客户端验证

```bash
# TODO: 测试客户端验证
# POST /api/app/validate
# 提示：包含 appId、appSecret、redirectUri 参数
```

---

# 思考与拓展

## 深入思考

1. **密码验证安全性**：

   - BCrypt 每次生成的哈希值都不同，如何验证密码？
   - 为什么不直接比较哈希值？
2. **账户锁定策略**：

   - 10 分钟的锁定时间是否合适？
   - 如何防止恶意锁定他人账户？
3. **重定向 URI 验证**：

   - 为什么必须完全匹配？
   - 允许部分匹配会有什么安全风险？

## 拓展实验

1. **实现渐进式锁定**：

   - 第一次失败：立即可重试
   - 第三次失败：锁定 1 分钟
   - 第五次失败：锁定 10 分钟
2. **实现验证码**：

   - 失败 3 次后要求输入验证码
   - 使用 Google reCAPTCHA
3. **实现密码强度检查**：

   - 注册时检查密码强度
   - 禁止使用常见密码

---

# 常见问题 (FAQ)

### Q1：BCrypt 每次生成的哈希值都不同，如何验证密码？

A1：
// TODO: 自己查阅资料并回答
// 提示：BCrypt 的盐值存储在哪里？

### Q2：账户锁定后如何手动解锁？

A2：

```java
// TODO: 实现手动解锁方法
// 提示：使用 loginFailureCache.invalidate()
```

### Q3：为什么重定向 URI 必须完全匹配？

A3：
// TODO: 自己思考并举例说明
// 提示：构造一个攻击场景

---

# 总结

完成本实验后，你应该：

- ✅ 理解密码哈希和验证的原理
- ✅ 掌握使用缓存实现账户锁定
- ✅ 理解重定向 URI 验证的安全重要性
- ✅ 能够通过所有单元测试
- ✅ 为下周的会话管理打下基础

**下周预告**：全局会话管理（TGT）- 单点登录的核心机制！

---

**预计完成时间**：8-10 小时
**难度等级**：⭐⭐⭐☆☆
