---
tags:
  - OAuth
date: 2025-09-02
title: 第 6 周实验指南：令牌撤销与单点登出
---


# 第 6 周实验指南：令牌撤销与单点登出

## 1. 学习目标

在我们的最后一次实验中，我们将实现令牌撤销和单点登出（SSO）这两个至关重要的安全功能。完成本次实验后，你将能够：

  

-   理解明确撤销令牌的重要性。

-   实现一个用于令牌撤销的端点（遵循 RFC 7009）。

-   理解后端通道、异步单点登出的概念。

-   实现当用户会话终止时通知客户端应用的逻辑。

  

---

  

## 2. 理论背景

  

### 2.1 令牌撤销

  

当用户从一个应用明确登出时会发生什么？客户端应用应该销毁其本地会话，并通知 SSO 服务器使令牌失效。这可以防止即使用户已登出，泄露的刷新令牌仍被使用。RFC 7009 为此定义了一个标准端点。客户端发送其 `client_id`、`client_secret` 和要撤销的 `token`。

  

### 2.2 单点登出 (SSO)

  

单点登录很棒，但它有一个对应物：单点登出。如果用户从一个应用登出，他们是否应该自动从在该 SSO 会话期间访问的所有其他应用中登出？为了安全，是的。

  

-   **前端通道登出**：涉及复杂的浏览器重定向。它不可靠，因为如果其中一个应用宕机，它可能会失败。

-   **后端通道登出**：SSO 服务器直接与每个客户端应用的后端通信，通知它们用户的会话已结束。这更可靠。服务器向每个客户端预先注册的登出 URI 发送一个特殊的“登出令牌”。然后，客户端验证此令牌并在其端终止相应的用户会话。

  

---

  

## 3. 实验任务

我们将向 `LocalTokenManager.java` 和主要的 `LogoutController` 中添加最后的逻辑。
### 任务 1：在 `LocalTokenManager` 中实现令牌移除逻辑
1.  **`remove(String refreshToken)` 方法**：

    -   此方法将由撤销端点调用。

    -   使用 `refreshToken` 从 `refreshTokenCache` 中检索 `TokenContent`。

    -   如果找到，则使刷新令牌（`refreshTokenCache.invalidate(refreshToken)`）和关联的访问令牌（`accessTokenCache.invalidate(content.getAccessToken())`）都失效。
    -   同时，从反向查找的 `tgtCache` 中移除刷新令牌，以清理 TGT 到 RT 的映射。

2.  **`removeByTgt(String tgt)` 方法**：

    -   这是单点登出的核心。当 TGT 过期或被手动移除时，它由我们 TGT 缓存上的 `RemovalListener`（来自第 3 周）调用。
    -   使用 `tgtCache` 获取与此 TGT 关联的所有刷新令牌的 `Set<String>`。
    -   遍历此集合，并为每个刷新令牌调用你自己的 `remove(refreshToken)` 方法。这将产生级联效应，使与主 TGT 会话关联的所有令牌都失效。
### 任务 2：实现撤销端点
在 `LogoutController.java`（或类似的控制器）中，实现 `/oauth/revoke` 端点。

1.  该端点应接受一个带 `token`、`client_id` 和 `client_secret` 参数的 `POST` 请求。

2.  首先，使用 `appService` 验证客户端的凭据（`client_id` 和 `client_secret`）。

3.  如果客户端有效，则调用 `tokenManager.remove(token)`
  

### 任务 3：实现异步登出通知（概念性）

实现完整的后端通道通知很复杂，因为它需要运行客户端应用。对于本次实验，我们将专注于服务器端的逻辑。

当调用 `removeByTgt` 时，在清除令牌后，服务器应：

1.  检索用户在此会话期间已认证的所有客户端应用的列表（此信息应存储在 `TicketGrantingTicketContent` 中）。

2.  对于每个已注册 `logout_uri` 的客户端应用：

3.  创建一个包含 `user_id` 和 `event` 声明的特殊 JWT（登出令牌）（`{ "http://schemas.openid.net/event/backchannel-logout": {} }`）。

4.  向客户端的 `logout_uri` 发送一个异步 `POST` 请求，并将 `logout_token` 作为参数。

---

  

## 4. 测试与验证
1.  **运行 `LocalTokenManagerTest.java`**：现有的测试套件已包含令牌撤销（`testRemove`）和通过 TGT 移除实现单点登出（`testRemoveByTgt`）的测试。确保这些测试在你的新实现下通过。


2.  **手动测试（可选）**：如果你有客户端应用，可以使用 Postman 或 `curl` 等工具手动测试撤销端点。

    -   首先，获取一个刷新令牌。

    -   然后，向 `/oauth/revoke` 发送一个带令牌的 `POST` 请求。

    -   尝试再次使用该刷新令牌；它应该会失败。

恭喜！你现在已经实现了一个完整的、安全的 OAuth 2.0 服务器，具备用户和客户端验证、会话管理、刷新令牌轮换和单点登出功能。