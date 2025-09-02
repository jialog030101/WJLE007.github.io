---
tags:
  - OAuth
date: 2025-09-02
title: 第 5 周实验指南：令牌颁发与刷新令牌轮换
---

# 第 5 周实验指南：令牌颁发与刷新令牌轮换
## 1. 学习目标

这是重要的一周！我们终于要用一个真正的实现来替换 `DummyTokenManager`。我们将构建 OAuth 2.0 服务器的引擎，负责创建、验证和刷新令牌。完成本次实验后，你将能够：

-   实现访问令牌和刷新令牌的创建。

-   理解访问令牌和刷新令牌的不同生命周期。

-   为增强安全性实现刷新令牌轮换（Refresh Token Rotation）。

-   实现刷新令牌盗窃检测。

  

---

  

## 2. 理论背景

  

### 2.1 访问令牌 vs. 刷新令牌

  
-   **访问令牌 (AT)**：这是客户端应用用来访问受保护资源（例如 API）的令牌。访问令牌的**生命周期很短**（例如 15 分钟），以限制泄露时造成的损害。它们会随每个 API 请求一起发送。

-   **刷新令牌 (RT)**：这是一种特殊的令牌，用于在旧的访问令牌过期时获取新的访问令牌。刷新令牌的**生命周期很长**（例如 30 天），但通常只能在令牌端点使用一次。它们由客户端安全存储，不会随每个 API 请求发送。

  

这种分离在安全性和用户体验之间取得了很好的平衡。用户不必每 15 分钟登录一次，但功能强大的令牌（RT）暴露的频率远低于 AT。

### 2.2 刷新令牌轮换

当客户端使用刷新令牌获取新的访问令牌时，该刷新令牌应该如何处理？

-   **标准方法**：服务器返回一个新的访问令牌，同一个刷新令牌可以稍后再次使用。

-   **轮换方法**：服务器返回一个新的访问令牌和**一个新的刷新令牌**。旧的刷新令牌立即失效。

轮换更安全。如果刷新令牌被泄露，攻击者可以用它无限期地生成新的访问令牌。通过轮换，攻击者只能使用一次。当合法用户下次使用它时，服务器会发现旧令牌被再次使用，意识到它被盗了，并可以使整个会话失效。

  

---
## 3. 实验任务


导航到 `smart-sso-starter-server` 模块中的 `LocalTokenManager.java` 文件。你将在这里实现完整的令牌生命周期。

### 任务 1：配置 Spring 使用 `LocalTokenManager`

首先，回到 `SmartSsoServerConfiguration.java` 文件。注释掉 `return new DummyTokenManager();` 并启用 `LocalTokenManager` 的实现。
### 任务 2：初始化缓存

在 `LocalTokenManager` 的构造函数中，初始化三个缓存：

1.  **`accessTokenCache`**：按访问令牌 `String` 索引存储 `TokenContent`。它应该有一个较短的超时时间（`accessTokenTimeout`）。

2.  **`refreshTokenCache`**：按刷新令牌 `String` 索引存储 `TokenContent`。它应该有一个较长的超时时间（`refreshTokenTimeout`）。

3.  **`tgtCache`**：这是一个反向查找缓存。它按 TGT `String` 索引存储一个 `Set<String>` 的刷新令牌。这对于单点登出至关重要：当 TGT 被撤销时，我们可以使用此缓存找到并使所有关联的刷新令牌失效。
### 任务 3：实现核心方法

1.  **`create(String refreshToken, TokenContent content)` 方法**：

    -   使用 `content.getAccessToken()` 作为键，将内容放入 `accessTokenCache`。

    -   使用 `refreshToken` 作为键，将内容放入 `refreshTokenCache`。

    -   将 `refreshToken` 添加到 `tgtCache` 中与 TGT 关联的令牌集合中。

1.  **`getByAccessToken(String accessToken)` 方法**：

    -   从 `accessTokenCache` 中检索内容。

1.  **`get(String refreshToken)` 方法**：

    -   从 `refreshTokenCache` 中检索内容。

1.  **`refresh(String refreshToken)` 方法（轮换的核心）**：

    -   **第 1 步：获取并移除**：原子地从 `refreshTokenCache` 中获取并移除令牌内容。如果找不到，说明 RT 无效或已被使用。返回 `null`。

        -   `TokenContent content = refreshTokenCache.getIfPresent(refreshToken);`

        -   `if (content == null) { ... }`

        -   `refreshTokenCache.invalidate(refreshToken);`

    -   **第 2 步：盗窃检测**：移除旧 RT 后，检查它是否属于因疑似被盗而被标记为撤销的已知令牌家族。如果是，则撤销整个家族并返回 `null`。

    -   **第 3 步：使旧访问令牌失效**：使关联的旧访问令牌失效：`accessTokenCache.invalidate(content.getAccessToken());`

    -   **第 4 步：生成新令牌**：创建一个新的访问令牌和一个新的刷新令牌。

        -   `String newAccessToken = createAccessToken();`

        -   `String newRefreshToken = createRefreshToken();`

    -   **第 5 步：更新内容**：用新的令牌值更新 `content` 对象。

    -   **第 6 步：重建关联**：调用你自己的 `create(newRefreshToken, content)` 方法，将新令牌及其关联存储在缓存中。

    -   **第 7 步：返回新内容**：返回更新后的 `content` 对象，它现在包含了新的令牌。

---

  

## 4. 测试与验证


1.  **运行 `LocalTokenManagerTest.java`**：这是一个复杂的测试套件。它验证：

    -   令牌的正确创建和检索。

    -   访问令牌和刷新令牌的独立过期。

    -   **刷新令牌轮换**：刷新时会颁发新的 AT 和 RT，并且旧的会失效。

    -   **盗窃检测**：使用已轮换（旧的）刷新令牌会失败，并导致整个令牌家族被撤销。

  

2.  **调试与修复**：这是迄今为止最具挑战性的实现。在 `refresh` 操作期间，广泛使用调试器来跟踪你三个缓存的状态。确保你的逻辑在必要时是原子的，以防止竞争条件。