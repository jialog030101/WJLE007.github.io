---
tags:
  - OAuth
date: 2025-09-02
title: 第 3 周实验指南：全局会话管理 (TGT)
---

# 第 3 周实验指南：全局会话管理 (TGT)
## 1. 学习目标
本周，我们将实现单点登录的核心：由票据授予票据（TGT）代表的全局会话。完成本次实验后，你将能够：
-   理解 TGT 在 SSO 系统中的作用。
-   实现 TGT 的创建、检索和移除。
-   使用带有生存时间（TTL）的缓存实现会话过期。
-   为会话实现滑动窗口过期策略。
-   限制每个用户的并发会话数量。

---
## 2. 理论背景

### 2.1 什么是票据授予票据 (TGT)？
TGT 是进入 SSO 王国的主钥匙。用户首次登录后，SSO 服务器会创建一个 TGT 并存储它，将其与用户身份关联。然后，服务器在用户的浏览器中设置一个包含 TGT ID 的 Cookie。
当用户访问另一个应用时，他们的浏览器会将此 TGT Cookie 发送到 SSO 服务器。服务器验证 TGT 以确认用户拥有活动会话，如果验证通过，则为新应用颁发访问令牌，而**无需再次请求用户登录**。这就是单点登录的精髓。
### 2.2 会话管理策略
-   **生存时间 (TTL)**：会话不应永久有效。我们定义一个超时时间（例如 30 分钟），非活动会话在此之后自动过期。
-   **滑动窗口过期**：为了改善用户体验，我们可以在用户活动时刷新会话的过期时间。如果用户执行操作，他们 30 分钟的会话计时器将重置。这可以防止他们在仍在使用应用时被登出。
-   **并发会话限制**：出于安全和资源管理的考虑，我们可能希望限制一个用户可以同时登录的设备数量（例如，最多 5 个活动会话）。
---
## 3. 实验任务
导航到 `smart-sso-starter-server` 模块中的 `LocalTicketGrantingTicketManager.java` 文件。你将在这里使用缓存实现管理 TGT 的逻辑。
### 任务 1：初始化缓存
在构造函数中，你需要使用 Guava 的 `CacheBuilder` 初始化两个缓存：
1.  **`tgtCache`**：用于存储 TGT。它的键是 TGT `String`，值是 `TicketGrantingTicketContent`。
    -   将 `expireAfterWrite()` 设置为构造函数传入的 `timeout` 值。这实现了基本的 TTL 过期。
    -   实现一个 `RemovalListener`。当 TGT 因过期或被驱逐而从缓存中移除时，你必须调用 `tokenManager.removeByTgt(tgt)` 以确保所有关联的访问令牌也失效（这是为单点登出做准备，我们稍后会完全实现）。
1.  **`userTgtCache`**：此缓存用于跟踪每个用户的会话以强制执行并发限制。它的键是 `Long` 类型的用户 ID，值是他们活动 TGT 的 `Queue<String>`。
### 任务 2：实现核心方法
1.  **`create(String tgt, TicketGrantingTicketContent content)` 方法**：

    -   首先，处理会话并发限制。从 `userTgtCache` 中获取该用户（`content.getUserId()`）的 TGT 队列。
    -   如果队列大小达到或超过 `MAX_SESSIONS_PER_USER`（例如 5），则从队列中移除最旧的 TGT（`queue.poll()`），并使用 `tgtCache.invalidate(oldestTgt)` 将其从主 `tgtCache` 中移除。
    -   将新的 `tgt` 添加到 `userTgtCache` 中用户的队列里。
    -   最后，将新的 TGT 添加到主 `tgtCache` 中：`tgtCache.put(tgt, content)`。
1.  **`get(String tgt)` 方法**：
    -   很简单：使用 `tgtCache.getIfPresent(tgt)` 从缓存中检索内容。
1.  **`remove(String tgt)` 方法**：
    -   从主缓存中使 TGT 失效：`tgtCache.invalidate(tgt)`。你配置的 `RemovalListener` 将处理后续的清理工作。
1.  **`refresh(String tgt)` 方法**：
    -   此方法实现滑动窗口。要刷新，你只需从缓存中重新读取 TGT，然后再写回去。写入（`put`）操作会重置过期计时器。
    -   获取内容：`TicketGrantingTicketContent content = get(tgt);`
    -   如果存在，则重新写入：`if (content != null) { tgtCache.put(tgt, content); }`

---
## 4. 测试与验证

1.  **运行 `LocalTicketGrantingTicketManagerTest.java`**：从 `src/test/java` 目录打开此测试文件。它包含以下测试：
    -   基本的创建和检索。
    -   自动过期（TTL）。
    -   滑动窗口刷新逻辑。
    -   并发会话限制的强制执行。
1.  **调试与修复**：运行测试。它们最初可能会失败。使用调试器逐步执行你的代码，检查缓存的状态，并修复你的实现，直到所有测试都通过。
通过 `LocalTicketGrantingTicketManagerTest` 中的所有测试意味着你已成功构建了一个强大的全局会话管理器！