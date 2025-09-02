---
tags:
  - OAuth
date: 2025-09-02
title: 第2周实验指南：用户和客户端应用程序
---

# 第 2 周实验指南：用户及客户端应用验证
## 1. 学习目标
本周，我们将深入 OAuth 2.0 流程的第一个关键步骤：验证用户凭证和客户端应用的身份。完成本次实验后，你将能够：
-   实现用户认证逻辑，包括密码验证。

-   增加安全增强功能：多次登录失败后锁定账户。

-   实现密码过期策略。

-   实现客户端应用验证，包括客户端密钥和重定向 URI 的检查。
---
## 2. 理论背景

### 2.1 用户认证

在 SSO 服务器授予任何应用访问权限之前，它必须首先验证用户的身份。这通常通过核对数据库中的用户名和密码来完成。为安全起见，密码绝不能以明文形式存储。我们应该使用像 **BCrypt** 这样强大的单向算法来存储密码的**哈希值**。当用户尝试登录时，我们对他们提供的密码进行哈希计算，并与存储的哈希值进行比较。

### 2.2 客户端认证与重定向 URI

在 OAuth 2.0 中，代表用户请求访问的应用被称为**客户端**。服务器也必须验证这个客户端，通常通过检查其 `client_id` 和 `client_secret` 来完成。

一项至关重要的安全措施是验证**重定向 URI**。用户成功登录后，服务器会带着一个授权码将他们送回客户端应用。服务器必须确保只重定向到预先注册的、可信的 URI，以防止攻击者截获授权码。

---

## 3. 实验任务

### 任务 1：实现 `UserServiceImpl`

导航到 `smart-sso-starter-server` 模块中的 `UserServiceImpl.java` 文件。你会在这里找到 `TODO` 注释作为指引。
1.  **`validate(String account, String password)` 方法**：
    -   使用 `userMapper.selectByUsername(account)` 从数据库中获取 `User`。
    -   **检查 1：用户存在性**：如果用户为 `null`，返回 `Result.error("用户不存在")`。
    -   **检查 2：账户锁定**：实现一个缓存（例如 Guava 的 `Cache`）来跟踪每个用户的登录失败次数。如果一个用户连续登录失败 5 次，将其账户锁定 10 分钟。在检查密码之前，先检查缓存。如果已锁定，返回 `Result.error("账号已被锁定...")`。
    -   **检查 3：密码验证**：使用注入的 `passwordEncoder.matches(password, user.getPassword())` 来比较提供的密码和存储的哈希值。如果不匹配，在你的锁定缓存中记录这次失败，并返回 `Result.error("密码错误")`。
    -   **检查 4：用户启用状态**：如果 `user.getIsEnable()` 为 `false`，返回 `Result.error("用户已被禁用")`。
    -   **检查 5：密码过期**：检查 `user.getPasswordLastModified()` 是否已超过 90 天。如果是，返回 `Result.error("密码已过期...")`。
    -   **成功**：如果所有检查都通过，清除该用户的登录失败缓存，并返回 `Result.success(user)`。
1.  **`getTokenUser(Long userId)` 方法**：
    -   通过用户 ID 获取 `User`。
    -   创建一个 `TokenUser` 对象，这是一个简化的用户表示，可以安全地包含在令牌中。
    -   **关键：不要在 `TokenUser` 对象中包含密码**或其他敏感信息。

### 任务 2：实现 `AppServiceImpl`

导航到 `AppServiceImpl.java` 文件。你也会在这里找到 `TODO` 注释。
1.  **`validate(String appId, String appSecret, String redirectUri)` 方法**：
    -   使用 `appMapper.selectByAppId(appId)` 从数据库中获取 `App`。
    -   **检查 1：应用存在性**：如果为 `null`，返回 `false`。
    -   **检查 2：应用启用状态**：如果 `app.getIsEnable()` 为 `false`，返回 `false`。
    -   **检查 3：应用密钥**：比较 `appSecret` 和 `app.getAppSecret()`。如果不匹配，返回 `false`。
    -   **检查 4：重定向 URI**：这是最重要的部分。已注册的 `app.getRedirectUri()` 可能包含多个 URI，以逗号分隔。你必须验证方法调用中提供的 `redirectUri` 与其中一个已注册的 URI **完全匹配**。如果不匹配，返回 `false`。
    -   **成功**：如果所有检查都通过，返回 `true`。

---

## 4. 测试与验证

我们提供了 JUnit 测试来帮助你验证你的实现。

1.  **运行 `UserServiceImplTest.java`**：在 `src/test/java` 目录下打开此文件并运行测试。它将检查密码验证、账户锁定逻辑和密码过期功能的正确性。修复你的实现，直到所有测试都通过。
2.  **运行 `AppServiceImplTest.java`**：打开并运行此测试文件。它将验证你的客户端验证逻辑，特别是重定向 URI 的检查。修复你的实现，直到所有测试都通过。
一旦所有测试都通过，你就成功完成了本周的实验！