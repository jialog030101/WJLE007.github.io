---
title: Project 6 - 令牌撤销与单点登出
date: 2025-08-12
categories:
  - Java
tags:
  - OAuth
---
# 实验目标

恭喜你来到最后一个实验！本周我们将实现 OAuth 2.0 系统的**收尾工作**：令牌撤销和单点登出。完成本实验后，你将能够：

1. ✅ 理解令牌撤销的重要性和应用场景
2. ✅ 实现符合 RFC 7009 的令牌撤销端点
3. ✅ 理解单点登出（Single Logout, SLO）的工作原理
4. ✅ 实现前端通道登出（Front-Channel Logout）
5. ✅ 理解后端通道登出（Back-Channel Logout）的概念
6. ✅ 完成整个 OAuth 2.0 授权服务器的实现

---

# 实验任务

## 任务 1：完善令牌管理器的撤销逻辑

### 文件位置

`smart-sso-server/src/main/java/com/smart/sso/server/token/LocalTokenManager.java`

### 1.1 完善 `remove` 方法

```java
@Override
public void remove(String refreshToken) {
    // TODO 1: 获取令牌内容
    // 提示：使用 refreshTokenCache.getIfPresent() 方法
    // 思考：如果令牌不存在应该如何处理？（幂等性）
  
    // TODO 2: 从刷新令牌缓存中移除
    // 提示：使用 refreshTokenCache.invalidate() 方法
  
    // TODO 3: 移除关联的访问令牌
    // 提示：从 TokenContent 中获取 accessToken
    // 注意：访问令牌可能为 null，需要判空
  
    // TODO 4: 从 TGT 反向索引中移除
    // 提示：
    // - 获取 TGT 和对应的令牌集合
    // - 从集合中移除当前刷新令牌
    // - 如果集合为空，清理整个 TGT 缓存条目
    // - 否则更新缓存
  
    // TODO 5: 记录日志
    // 提示：记录刷新令牌和用户 ID（用于审计）
}
```

**设计提示**：

- **幂等性原则**：多次调用 `remove` 同一个令牌不应产生副作用
- **级联删除**：考虑令牌之间的关联关系
- **日志记录**：哪些信息对审计有帮助？

---

### 1.2 完善 `removeByTgt` 方法

```java
@Override
public void removeByTgt(String tgt) {
    // TODO 1: 获取该 TGT 关联的所有刷新令牌
    // 提示：使用 tgtCache.getIfPresent() 方法
    // 思考：如果 TGT 不存在或没有令牌应该如何处理？
  
    // TODO 2: 复制集合以避免并发修改异常
    // 提示：为什么需要复制？考虑遍历时集合被修改的场景
  
    // TODO 3: 批量撤销所有刷新令牌
    // 提示：遍历复制的集合，调用 remove() 方法
    // 思考：是否需要计数撤销了多少令牌？
  
    // TODO 4: 清除 TGT 反向索引
    // 提示：使用 tgtCache.invalidate() 方法
  
    // TODO 5: 记录日志
    // 提示：记录 TGT 和撤销数量
}
```

**并发安全思考**：

```java
// 为什么需要复制集合？
// 场景分析：
// 线程 A：遍历 tokens 集合，调用 remove()
// 线程 B：同时修改同一个 tokens 集合
// 结果：ConcurrentModificationException
// 解决：???
```

---

## 任务 2：实现令牌撤销端点

### 文件位置

`smart-sso-server/src/main/java/com/smart/sso/server/controller/TokenController.java`

### 2.1 添加撤销端点

```java
/**
 * 令牌撤销端点 (RFC 7009)
 * POST /oauth/revoke
 */
@PostMapping("/revoke")
public ResponseEntity<Void> revoke(
        @RequestParam("token") String token,
        @RequestParam(value = "token_type_hint", required = false) String tokenTypeHint,
        @RequestParam("client_id") String clientId,
        @RequestParam("client_secret") String clientSecret) {
  
    // TODO 1: 验证客户端身份
    // 提示：调用 appService.validate() 方法
    // 思考：客户端验证失败应该返回什么 HTTP 状态码？
  
    // TODO 2: 根据 token_type_hint 确定令牌类型
    // 提示：
    // - 如果 hint 是 "access_token"，处理访问令牌
    // - 否则默认当作刷新令牌处理
    // 思考：如何从访问令牌找到对应的刷新令牌？
  
    // TODO 2.1: 处理访问令牌撤销
    // 步骤：
    // 1. 通过访问令牌获取 TokenContent
    // 2. 验证令牌是否属于该客户端（调用辅助方法）
    // 3. 获取关联的刷新令牌并撤销
  
    // TODO 2.2: 处理刷新令牌撤销
    // 步骤：
    // 1. 通过刷新令牌获取 TokenContent
    // 2. 验证令牌是否属于该客户端
    // 3. 撤销刷新令牌
  
    // TODO 3: 返回响应
    // 提示：根据 RFC 7009 规范，即使令牌不存在也应返回 200 OK
    // 思考：为什么要这样设计？
}

/**
 * 辅助方法：验证令牌是否属于指定客户端
 * 
 * 提示：
 * - 需要从 TokenContent 中获取客户端信息
 * - 生产环境应该严格验证所有权
 * - 当前简化实现可以暂时允许所有客户端撤销
 */
private boolean isTokenOwnedByClient(TokenContent content, String clientId) {
    // TODO: 实现所有权验证逻辑
    // 思考：如果 TokenContent 中没有存储 appId 怎么办？
}
```

**RFC 7009 合规性检查清单**：

- [ ] 验证客户端身份
- [ ] 支持 `token_type_hint` 参数
- [ ] 实现幂等性（多次撤销返回相同结果）
- [ ] 始终返回 200 OK（不泄露令牌是否存在）

---

## 任务 3：实现登出端点

### 文件位置

`smart-sso-server/src/main/java/com/smart/sso/server/controller/LogoutController.java`

### 3.1 实现前端通道登出

```java
/**
 * 前端通道登出端点
 * GET /oauth/logout?redirect_uri={客户端登出后跳转 URI}
 */
@GetMapping("/logout")
public String logout(
        HttpServletRequest request,
        HttpServletResponse response,
        @RequestParam(value = "redirect_uri", required = false) String redirectUri,
        Model model) {
  
    // TODO 1: 从 Cookie 中获取 TGT
    // 提示：调用辅助方法 getTgtFromCookie()
    // 思考：如果用户未登录应该如何处理？
  
    // TODO 2: 获取 TGT 关联的所有客户端应用
    // 提示：
    // - 从 tgtManager 获取 TGT 内容
    // - 获取已访问的应用 ID 列表
    // - 查询每个应用的登出 URI
    // 思考：如何在 TGT 中记录用户访问过的应用？
  
    // TODO 3: 撤销 TGT（级联撤销所有令牌）
    // 提示：调用 tgtManager.remove() 方法
  
    // TODO 4: 清除 TGT Cookie
    // 提示：调用辅助方法 clearTgtCookie()
  
    // TODO 5: 返回包含 iframe 的 HTML 页面
    // 提示：
    // - 将 logoutUrls 添加到 Model
    // - 将 redirectUri 添加到 Model
    // - 返回视图名称 "logout"
}

/**
 * 辅助方法：从 Cookie 中获取 TGT
 */
private String getTgtFromCookie(HttpServletRequest request) {
    // TODO: 实现从 Cookie 中提取 TGT 的逻辑
    // 提示：
    // - 获取所有 Cookie
    // - 遍历查找名为 "TGT" 的 Cookie
    // - 返回其值（如果存在）
}

/**
 * 辅助方法：清除 TGT Cookie
 */
private void clearTgtCookie(HttpServletResponse response) {
    // TODO: 实现清除 Cookie 的逻辑
    // 提示：
    // - 创建同名 Cookie，值设为空
    // - 设置 MaxAge 为 0
    // - 设置正确的 Path
    // - 设置 HttpOnly 为 true
}

/**
 * 辅助方法：重定向到客户端
 */
private String redirectToClient(String redirectUri) {
    // TODO: 实现重定向逻辑
    // 提示：
    // - 如果 redirectUri 不为空，返回 "redirect:" + redirectUri
    // - 否则返回默认首页 "redirect:/"
}
```

---

### 3.2 创建登出页面模板

创建 `src/main/resources/templates/logout.html`：

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>登出中...</title>
    <style>
        /* TODO: 添加样式（可参考示例或自定义） */
    </style>
</head>
<body>
    <!-- TODO 1: 添加登出提示信息 -->
  
    <!-- TODO 2: 添加加载动画 -->
  
    <!-- TODO 3: 添加隐藏的 iframe，用于通知各个应用登出 -->
    <!-- 提示：使用 Thymeleaf 的 th:each 遍历 logoutUrls -->
  
    <script th:inline="javascript">
        // TODO 4: 等待所有 iframe 加载完成后重定向
        // 提示：
        // - 使用 setTimeout 等待一段时间（如 2 秒）
        // - 从 Model 中获取 redirectUri
        // - 执行重定向
    </script>
</body>
</html>
```

---

### 3.3 实现后端通道登出（概念性）

```java
/**
 * 后端通道登出通知
 * 注意：这是服务器到服务器的调用
 */
public void notifyBackChannelLogout(String tgt) {
    // TODO 1: 获取 TGT 关联的所有应用
    // 提示：从 tgtManager 获取 TGT 内容和应用 ID 列表
  
    // TODO 2: 为每个应用生成登出令牌
    // 提示：
    // - 遍历应用 ID 列表
    // - 检查应用是否配置了后端登出 URI
    // - 调用辅助方法生成登出令牌
  
    // TODO 3: 异步发送 POST 请求到客户端的登出端点
    // 提示：
    // - 使用 CompletableFuture.runAsync() 异步执行
    // - 使用 HttpClient 发送 POST 请求
    // - 请求体包含 logout_token 参数
    // - 处理成功和失败的响应
    // 思考：如果某个应用通知失败应该如何处理？
}

/**
 * 辅助方法：创建登出令牌
 * 生产环境应使用 JWT 库（如 jjwt）
 */
private String createLogoutToken(Long userId, String appId) {
    // TODO: 实现登出令牌生成逻辑
    // 提示：
    // - 简化实现可以使用 Base64 编码
    // - 包含用户 ID、应用 ID 和时间戳
    // - 生产环境应使用 JWT 并签名
}
```

**思考题**：

1. 前端通道登出和后端通道登出各有什么优缺点？
2. 如果后端通道通知失败应该如何处理？
3. 如何防止登出令牌被重放攻击？

---

## 任务 4：集成 TGT 移除监听器

### 修改 TGT 管理器

```java
// filepath: smart-sso-server/src/main/java/com/smart/sso/server/session/LocalTicketGrantingTicketManager.java

public LocalTicketGrantingTicketManager(int timeout, TokenManager tokenManager) {
    // ...existing code...
  
    this.tgtCache = CacheBuilder.newBuilder()
        .expireAfterWrite(timeout, TimeUnit.MINUTES)
        .maximumSize(100000)
        .removalListener(new RemovalListener<String, TicketGrantingTicketContent>() {
            @Override
            public void onRemoval(RemovalNotification<String, TicketGrantingTicketContent> notification) {
                // TODO 1: 记录日志
                // 提示：记录 TGT、移除原因（cause）
              
                // TODO 2: 级联失效 - 撤销所有关联的令牌
                // 提示：调用 tokenManager.removeByTgt() 方法
                // 思考：为什么需要检查 tokenManager 不为 null？
              
                // TODO 3: 可选 - 触发后端通道登出通知
                // 提示：
                // - 仅在显式移除时触发（cause == RemovalCause.EXPLICIT）
                // - 调用 notifyBackChannelLogout() 方法
            }
        })
        .build();
}
```

**设计思考**：

- 为什么使用 `RemovalListener` 而不是手动调用撤销方法？
- 自动过期和手动删除的 `RemovalCause` 有什么区别？
- 级联失效可能导致什么性能问题？如何优化？

---

# 测试与验证

## 1. 单元测试

### 1.1 运行令牌管理器测试

```bash
# 在 IDEA 中右键运行
src/test/java/com/smart/sso/server/token/LocalTokenManagerTest.java
```

**测试用例提示**：

- [ ] `remove()` 方法的幂等性测试
- [ ] 撤销刷新令牌时是否级联删除了访问令牌
- [ ] `removeByTgt()` 是否批量撤销了所有令牌
- [ ] TGT 被移除时是否自动触发了令牌撤销

**自己编写测试用例**：

```java
@Test
public void testRemoveIdempotence() {
    // TODO: 测试多次调用 remove 同一个令牌不会产生错误
}

@Test
public void testRemoveCascade() {
    // TODO: 测试撤销刷新令牌时访问令牌也被删除
}
```

---

## 2. 手动测试

### 2.1 测试令牌撤销端点

```bash
# 步骤 1：获取令牌
# TODO: 构造 POST 请求到 /oauth/token
# 提示：使用授权码模式获取令牌

# 步骤 2：撤销刷新令牌
# TODO: 构造 POST 请求到 /oauth/revoke
# 提示：包含 token、token_type_hint、client_id、client_secret 参数

# 步骤 3：尝试使用已撤销的令牌刷新
# TODO: 构造 POST 请求到 /oauth/token
# 提示：使用 refresh_token 授权类型
# 预期结果：返回 invalid_grant 错误
```

### 2.2 测试登出流程

```bash
# 步骤 1：在浏览器访问登出端点
# TODO: 访问 http://localhost:8080/oauth/logout?redirect_uri=...

# 步骤 2：观察页面
# TODO: 检查是否显示"登出中"页面

# 步骤 3：检查 Cookie
# TODO: 打开开发者工具，确认 TGT Cookie 已被删除

# 步骤 4：验证单点登出
# TODO: 尝试访问需要登录的页面，应该被重定向到登录页
```

---

# 思考与拓展

## 深入思考

1. **令牌撤销的安全性**：

   - 为什么即使令牌不存在也要返回 200 OK？
   - 如何防止恶意客户端频繁调用撤销端点？
2. **单点登出的可靠性**：

   - 前端通道登出可能失败的场景有哪些？
   - 如何提高后端通道登出的成功率？
3. **性能优化**：

   - `removeByTgt` 遍历所有令牌的性能如何？
   - 如何优化大量令牌的批量撤销？

## 拓展实验

1. **实现重试机制**：

   - 后端通道通知失败时自动重试
   - 使用指数退避算法
2. **实现撤销历史记录**：

   - 记录所有令牌撤销操作
   - 支持管理员查询撤销历史
3. **实现设备管理**：

   - 在 TGT 中记录设备信息
   - 支持"除当前设备外全部登出"

---

# 总结

完成本实验后，你应该能够：

- ✅ 实现符合 RFC 7009 的令牌撤销端点
- ✅ 理解单点登出的工作原理
- ✅ 实现前端通道和后端通道登出
- ✅ 掌握级联撤销机制
- ✅ **完成整个 OAuth 2.0 授权服务器的实现！**

**恭喜你完成所有实验！** 🎉

---

**预计完成时间**：10-12 小时
**难度等级**：⭐⭐⭐⭐☆
