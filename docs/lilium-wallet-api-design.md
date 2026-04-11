# Lilium 钱包 API 接入规范 v1

适用对象：接入 Lilium 的第三方开发者  
文档状态：草案

## 目录

- [1. 文档说明](#overview)
- [2. 共享平台能力](#shared-platform)
- [3. 钱包入口与认证边界](#wallet-entrypoints)
- [4. 核心概念](#concepts)
- [5. 钱包只读 API](#wallet-read)
- [6. 钱包转账 API](#wallet-transfer)
- [7. 数据契约](#data-contract)
- [8. 错误模型](#errors)
- [9. 幂等要求](#idempotency)
- [10. 安全要求](#security)

<a id="overview"></a>
## 1. 文档说明

Lilium 提供面向第三方业务系统的游戏币钱包查询与转账能力。第三方可以：

- 通过 machine token 查询绑定账户的余额、流水、统计
- 通过 machine token 或 browser token 向其他用户发起钱包转账

本规范仅适用于游戏币场景，不涉及真实货币。

前置文档：

- [Lilium 平台认证与路由规范 v1.1](./lilium-platform-authentication.md)
- [Lilium 开放清算 API 接入规范 v1.1](./lilium-open-clearing-api-design.md)

接入钱包能力前，第三方应先完成共享平台能力接入。

<a id="shared-platform"></a>
## 2. 共享平台能力

本规范只定义钱包能力本身。平台级认证、统一入口、OIDC 与 `client_credentials` token 获取流程，统一见：

- [Lilium 平台认证与路由规范 v1.1](./lilium-platform-authentication.md)

<a id="wallet-entrypoints"></a>
## 3. 钱包入口与认证边界

钱包相关入口为：

- `https://lilium.kuma.homes/api/wallet/balance`
- `https://lilium.kuma.homes/api/wallet/stats`
- `https://lilium.kuma.homes/api/wallet/transactions`
- `https://lilium.kuma.homes/api/wallet/wealth_leaderboard`
- `https://lilium.kuma.homes/api/wallet/transfer`

认证边界：

- 钱包只读 API 使用 `Bearer` token，要求 `wallet:read` scope
- 钱包转账 API 使用 `Bearer` token，要求 `wallet:transfer` scope
- browser token 可调用所有钱包 API（读接口隐式具有权限，转账接口需配合 `Idempotency-Key`）
- `/api/wallet/bank` 为 browser-only 端点，machine token 不可访问

<a id="concepts"></a>
## 4. 核心概念

### 4.1 `effective_user_id`

所有钱包 API 按 machine token 的 `effective_account_user_id` 确定操作账户。

- 本规范中的所有 `*_user_id` 字段均沿用平台 `user_id` 规则：类型为 `string`，最大长度 `255`，调用方不得假设固定格式
- browser token：操作当前登录用户账户
- machine token：操作 token 绑定账户（主账户或子账户）
- machine token 不拥有 owner 级横向钱包可见性

### 4.2 Wallet Transfer 与 Clearing Payout 的区别

| 维度 | Wallet Transfer | Clearing Payout |
| --- | --- | --- |
| 语义 | 通用钱包转账，`from -> to` | 商户结算动作 |
| 范围 | 任意合法钱包主体间 | partner 清算账户向用户发放 |
| 附加结构 | 无 clearing instruction / webhook | 有 clearing instruction、partner reference、webhook、状态机 |
| 端点 | `POST /api/wallet/transfer` | `POST /api/v1/clearing-instructions` |
| scope | `wallet:transfer` | `clearing:basic` |

Transfer 不复用 clearing payout 语义。两者最多只共用底层账务原语。

### 4.3 合法钱包主体

转账目标 `to_user_id` 必须满足：

- 类型同 `user_id`
- 对应一条真实 `User` 记录
- 不是系统内部账户（BANK / futures treasury / insurance treasury / wallet adjustment offset 等）

若 `to_user_id` 没有对应 `User` 记录，返回 `recipient_not_found`。

<a id="wallet-read"></a>
## 5. 钱包只读 API

### 5.1 查询余额

`GET /api/wallet/balance`

认证：`wallet:read`

语义：

- browser token：读取当前登录用户余额
- machine token：读取 `effective_account_user_id` 余额

响应示例：

```json
{
  "user_id": "sub_0123456789abcdef0123",
  "balance": "1000.00"
}
```

### 5.2 查询统计

`GET /api/wallet/stats`

认证：`wallet:read`

语义：

- browser token：当前登录用户统计
- machine token：`effective_account_user_id` 的流水统计

### 5.3 查询流水

`GET /api/wallet/transactions`

认证：`wallet:read`

语义：

- browser token：当前登录用户流水
- machine token：`effective_account_user_id` 流水

查询参数：

- `page`：可选，默认 `1`
- `size`：可选，默认 `20`，最大 `100`
- `type`：可选，按交易类型过滤
- `search`：可选，按描述搜索

分页 / type / search 都只在该账户范围内执行。

### 5.4 查询财富排行榜

`GET /api/wallet/wealth_leaderboard`

认证：`wallet:read`

语义：

- 榜单本身是全局财富榜
- `current_user_id` / nearby context 使用 `identity.effective_user_id`
- browser token 与 machine token 均可访问

查询参数：

- `limit`：可选，默认 `10`，最大 `100`

### 5.5 Bank 余额（browser-only）

`GET /api/wallet/bank`

认证：browser token only（通过 `get_current_user`）

- machine token 不可访问
- 这更像系统 / 运营视图，不适合与 partner subaccount machine token 一起开放

<a id="wallet-transfer"></a>
## 6. 钱包转账 API

### 6.1 发起转账

`POST /api/wallet/transfer`

认证：

- browser token：允许调用
- machine token：需要 `wallet:transfer` scope

所有调用方必须携带 `Idempotency-Key`。

请求体：

| 字段 | 类型 | 必填 | 约束 | 说明 |
| --- | --- | --- | --- | --- |
| `to_user_id` | string | 是 | 同 `user_id` | 收款账户 |
| `amount` | string | 是 | 正数字符串，符合 `DECIMAL(38,2)` | 转账金额 |
| `memo` | string | 否 | 0-200 字符 | 可选备注 |

注意：

- 请求体不包含 `from_user_id`
- 出账账户固定为 `identity.effective_user_id`
- 避免给客户端"可任意指定出账账户"的字段

请求示例：

```json
{
  "to_user_id": "6ba7b810-9dad-11d1-80b4-00c04fd430c8",
  "amount": "12.34",
  "memo": "付款备注"
}
```

成功响应：

| 字段 | 说明 |
| --- | --- |
| `from_user_id` | 出账账户，同 `user_id` |
| `to_user_id` | 收款账户，同 `user_id` |
| `amount` | 确认金额 |
| `from_balance` | 出账方转后余额 |
| `to_balance` | 收款方转后余额 |
| `reference_id` | 对账用参考 ID |
| `created_at` | 创建时间 |

响应示例：

```json
{
  "from_user_id": "sub_0123456789abcdef0123",
  "to_user_id": "6ba7b810-9dad-11d1-80b4-00c04fd430c8",
  "amount": "12.34",
  "from_balance": "987.66",
  "to_balance": "112.34",
  "reference_id": "wallet_transfer_abc123",
  "created_at": "2026-04-12T10:30:00Z"
}
```

### 6.2 基本校验

- `amount > 0`
- `to_user_id != identity.effective_user_id`（不允许自转账）
- 目标账户必须是合法钱包主体
- 不允许向系统内部账户转账
- 余额不足时返回 `insufficient_balance`

### 6.3 交易类型

- 出账方：`TRANSFER_OUT`
- 收款方：`TRANSFER_IN`
- 双边流水共享同一个 `tx_group_id`
- `memo` 进入 transaction memo 字段，不拼入 description

<a id="data-contract"></a>
## 7. 数据契约

除非另有说明，所有字符串字段均使用 UTF-8 编码，且不应包含前后空白。

### 7.1 金额格式

- 使用字符串传输，禁止使用 JSON number
- 最多 2 位小数
- `amount > 0`
- 不额外引入单笔上限

### 7.2 幂等 payload 构造

服务端构造 canonical payload，不是直接拿原始 request body 做 hash：

```python
request_payload = {
    "from_user_id": identity.effective_user_id,
    "to_user_id": body.to_user_id,
    "amount": str(body.amount),
    "memo": body.memo or "",
}
```

这保证：

- `from_user_id` 虽然不在客户端请求体中，但仍然进入 hash
- 不同 effective account 即使复用同一个 `Idempotency-Key`，也不会串单
- `memo = null` 与 `memo = ""` 语义收敛

<a id="errors"></a>
## 8. 错误模型

错误响应统一格式：

```json
{
  "detail": "insufficient_balance"
}
```

钱包转账错误码：

| 场景 | HTTP status | detail |
| --- | --- | --- |
| 缺少幂等 key | 400 | `missing_idempotency_key` |
| 目标不存在 | 404 | `recipient_not_found` |
| 目标为内部账户 | 422 | `invalid_recipient` |
| 自转账 | 422 | `self_transfer` |
| 余额不足 | 422 | `insufficient_balance` |
| 幂等冲突 | 409 | `idempotency_conflict` |
| 金额非法 | 422 | `invalid_amount` |

钱包只读错误码沿用平台认证文档的通用错误码：

- `UNAUTHORIZED_PARTNER`
- `INVALID_SCOPE`

<a id="idempotency"></a>
## 9. 幂等要求

`POST /api/wallet/transfer` 必须携带 `Idempotency-Key`。

规则：

- browser token 与 machine token 统一使用幂等 contract
- 同一 client / browser namespace 对同一 endpoint 使用相同 `Idempotency-Key` 时，服务端返回同一逻辑结果
- 幂等 payload 由服务端构造，包含 canonical `from_user_id`、`to_user_id`、`amount`、`memo`
- 服务端保证相同 `Idempotency-Key` 的结果在 `24` 小时内保持一致

确定性失败 replay：

- 相同 `Idempotency-Key` 重试同一请求时，如果原始请求因 `recipient_not_found` 或 `insufficient_balance` 失败，replay 同样返回该失败结果

<a id="security"></a>
## 10. 安全要求

除共享平台认证文档中的通用安全要求外，钱包能力额外要求：

- machine token 的转账操作始终以 `effective_account_user_id` 为出账账户
- `wallet:transfer` 与 `wallet:read` 独立授权，不互相包含
- 第三方不应把 `wallet:transfer` 授予不需要转账的凭证
- 本规范不引入单笔上限和频率限制；如果第三方需要，应单独申请

## 修订历史

- `2026-04-12`: 初始钱包 API 规范成稿。
- `2026-04-12`: 统一 `user_id` 契约：所有 `*_user_id` 字段均沿用平台 `user_id` 规则（`string`，最大长度 `255`，不假设固定格式）。
