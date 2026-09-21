# CoinClock API 接口契约

版本：v1 · 日期：2026-09-21 · Base URL 示例：https://api.example.com

HTTP 契约服务于五个入口：客户端使用同一套版本化 REST/WS 能力，平台差异只出现在 OIDC 回调、通知、钱包深链和本地能力。OpenAPI 草案见 [openapi.yaml](openapi.yaml)，本文示例更完整；两者不一致时以评审后的 OpenAPI 为生成依据。

## 1. 通用约定

### 1.1 请求头

~~~http
Authorization: Bearer <access_token>
Content-Type: application/json
Accept: application/json
X-Client-Platform: ios|android|windows|macos|web
X-Client-Version: 0.1.0
X-Request-Id: 01J...
Idempotency-Key: 01J...             # POST/支付/交易/AI 建议必填
If-Match: "7"                       # 编辑用户资源时使用 version
~~~

X-Request-Id 若客户端不传由网关生成，响应原样返回。服务器时间为 UTC ISO 8601；金额、价格、数量、费率、额度统一使用十进制字符串，避免 JavaScript number 精度问题。枚举未知值客户端应保留兼容显示，不因新增枚举崩溃。

### 1.2 响应与错误

成功响应统一包含 request_id；列表使用 items、next_cursor、has_more。错误结构：

~~~json
{
  "error": {
    "code": "ALERT_VERSION_CONFLICT",
    "message": "提醒已在另一台设备被修改，请重新加载",
    "details": {"resource_id": "01J...", "current_version": 8},
    "retryable": false
  },
  "request_id": "01J9..."
}
~~~

| 状态 | 语义 | 客户端行为 |
|---:|---|---|
| 400 | schema、精度、枚举或业务参数错误 | 修正表单，不重试 |
| 401 | token 缺失/过期 | 刷新一次，失败重新登录 |
| 403 | 无权限、地区/政策不允许 | 显示原因，不循环重试 |
| 404 | 资源不属于当前用户或不存在 | 显示已删除/不存在 |
| 409 | CAS、幂等键冲突、订单状态竞争 | 拉取状态或复用原响应 |
| 422 | 行情过期、额度不足等业务不可执行 | 显示 code 与修复动作 |
| 429 | 限流/额度 | 遵守 Retry-After |
| 500/503 | 服务或依赖暂时失败 | 只对安全读/幂等请求退避 |

服务端不把内部栈、SQL、RPC token、签名和完整钱包地址放进错误。message 面向用户，code 稳定供客户端分支。

### 1.3 鉴权与权限

- Native 使用 OIDC Authorization Code + PKCE，API 只接受 access token；Web 用 BFF 的 HttpOnly Cookie 或同等安全模式，避免长期 token 放 localStorage。
- 钱包绑定先获取 challenge，用户在钱包签名后提交；challenge 的 purpose、domain、chain_id 必须匹配。
- API 通过 token 的 sub 映射 user_id 取所有者，不信任请求体 user_id。
- 交易、平台币付款、NFT 权益同步等敏感接口再次核验资源、签名、policy version 和风险状态。

## 2. 接口目录

| 域 | 方法 | 路径 | 阶段 | 说明 |
|---|---|---|---|---|
| Auth | GET | /v1/auth/config | S0 | OIDC、支持链、客户端策略 |
| Auth | POST | /v1/auth/wallet/challenges | S2 | 获取钱包签名 challenge |
| Auth | POST | /v1/auth/wallet/verify | S2 | 验证签名并绑定/登录 |
| User | GET/PATCH | /v1/me | S0 | 当前用户和偏好 |
| Device | POST/DELETE | /v1/me/devices、/v1/me/devices/{id} | S0 | 注册/撤销设备和通知端点 |
| Market | GET | /v1/markets/instruments | S0 | 币对/场所目录 |
| Market | GET | /v1/markets/quotes | S0 | 最新快照 |
| Market | GET | /v1/markets/candles | S0 | K 线 |
| Market | WS | /v1/stream | S0 | 行情、规则和 AI 状态 |
| Watchlist | GET/POST/PATCH/DELETE | /v1/watchlists... | S0 | 自选分组 |
| Alert | GET/POST/PATCH/DELETE | /v1/alerts... | S0 | 规则管理 |
| Alert | POST | /v1/alerts/{id}/pause、/resume、/test | S0 | 状态控制和测试通知 |
| Notice | GET/PATCH | /v1/notifications... | S0 | 通知历史、确认、回执 |
| Wallet | GET/POST/DELETE | /v1/wallets... | S2 | 绑定地址、权限状态 |
| Fee | GET | /v1/fee-tiers、/v1/me/entitlements | S1/S2 | 费率与权益摘要 |
| Analyst | GET | /v1/analysts | S1 | 分析师目录 |
| Analyst | POST | /v1/analysts/{id}/unlock-quotes | S1/S2 | 固定 quote，过期不可支付 |
| Analyst | POST | /v1/ai/runs | S1 | 创建分析任务 |
| Analyst | GET | /v1/ai/runs/{id} | S1 | 任务状态/报告 |
| Payment | POST | /v1/payments | S2 | 提交付款证明，服务端异步核验 |
| Trade | POST | /v1/order-intents、/v1/orders | S2 | 准备/提交订单 |
| Trade | POST | /v1/orders/{id}/cancel | S2 | 撤单意图 |
| Trade | GET | /v1/orders、/v1/positions | S2 | 订单和持仓镜像 |
| Admin | internal | /internal/v1/... | 运维 | 独立网络、RBAC、审批，不给客户端 |

## 3. 关键接口示例

### 3.1 查询行情

~~~http
GET /v1/markets/quotes?instrument_ids=binance:spot:BTCUSDT,hyperliquid:perp:BTC-USD&price_type=LAST
Authorization: Bearer eyJ...
~~~

~~~json
{
  "items": [
    {
      "instrument_id": "01JQBTCSPOT...",
      "venue": "BINANCE",
      "symbol": "BTC/USDT",
      "market_type": "SPOT",
      "price_type": "LAST",
      "price": "70012.45",
      "source_ts": "2026-09-21T08:00:00.120Z",
      "received_at": "2026-09-21T08:00:00.180Z",
      "age_ms": 60,
      "quality": "FRESH",
      "sequence": "932001"
    }
  ],
  "server_time": "2026-09-21T08:00:00.190Z",
  "request_id": "01JQ..."
}
~~~

quality 为 STALE/GAP/SUSPECT 时仍可展示，但创建/提交交易由业务规则决定是否禁止；客户端必须显示来源和更新时间。

### 3.2 创建提醒

~~~http
POST /v1/alerts
Authorization: Bearer eyJ...
Idempotency-Key: 01JALERTCREATE...
Content-Type: application/json

{
  "instrument_id": "01JQBTCSPOT...",
  "price_type": "LAST",
  "direction": "ABOVE",
  "trigger_mode": "CROSS",
  "threshold": "72000.00",
  "repeat_policy": "REARM",
  "hysteresis_bps": 30,
  "cooldown_seconds": 300,
  "expires_at": null,
  "delivery_policy": {"mode": "ALL_DEVICES", "quiet_hours": "DEFER"}
}
~~~

~~~json
{
  "data": {
    "id": "01JQALERT...",
    "status": "ACTIVE",
    "activation_status": "PENDING",
    "version": 1,
    "cycle_no": 1,
    "baseline_price": null,
    "instrument": {"symbol": "BTC/USDT", "venue": "BINANCE"},
    "semantic_text": "当 Binance BTC/USDT 最新成交价从下方穿越 72000.00 USDT 时提醒",
    "created_at": "2026-09-21T08:05:00Z",
    "updated_at": "2026-09-21T08:05:00Z"
  },
  "request_id": "01JQ..."
}
~~~

服务端收到规则后再建立 baseline。activation_status=PENDING 不代表已经开始判定，客户端监听 alert.activated 或轮询详情。

### 3.3 编辑、暂停、恢复

~~~http
PATCH /v1/alerts/01JQALERT...
If-Match: "1"
Content-Type: application/json

{"threshold":"68000.00","expected_version":1}
~~~

~~~json
{
  "data": {
    "id": "01JQALERT...",
    "version": 2,
    "status": "ACTIVE",
    "activation_status": "PENDING",
    "baseline_price": null
  },
  "request_id": "01JQ..."
}
~~~

~~~http
POST /v1/alerts/01JQALERT.../pause
Idempotency-Key: 01JPAUSE...
~~~

~~~json
{"data":{"id":"01JQALERT...","status":"PAUSED","version":3},"request_id":"01JQ..."}
~~~

暂停/恢复接口幂等：已经 PAUSED 再暂停返回当前状态，不生成第二次触发。版本冲突返回 ALERT_VERSION_CONFLICT。

### 3.4 注册设备和通知端点

~~~http
POST /v1/me/devices
Authorization: Bearer eyJ...
Idempotency-Key: 01JDEVICE...

{
  "platform": "ios",
  "install_id": "client-generated-random-id",
  "app_version": "0.1.0",
  "timezone": "Asia/Shanghai",
  "notification_endpoints": [
    {"channel":"APNS","token":"<device-token>","permission":"AUTHORIZED"}
  ]
}
~~~

~~~json
{
  "data": {
    "device_id": "01JDEVICE...",
    "status": "ACTIVE",
    "notification_endpoints": [
      {"id":"01JENDPOINT...","channel":"APNS","status":"ACTIVE"}
    ]
  },
  "request_id": "01JQ..."
}
~~~

设备注册只绑定推送能力；permission 是客户端观测值，服务端不能把它当作系统一定会展示通知。

### 3.5 获取权益与费率

~~~http
GET /v1/me/entitlements
Authorization: Bearer eyJ...
~~~

~~~json
{
  "data": {
    "fee_tier": {
      "code":"VIP1",
      "rolling_window_days":30,
      "notional_usd":"125000.00",
      "calculated_at":"2026-09-21T07:55:00Z",
      "policy_version":"fee-2026-09-01-01"
    },
    "entitlements": [
      {"code":"ANALYST_ACCESS","analyst_id":"momentum-v2","remaining":"1","expires_at":"2026-10-21T00:00:00Z"},
      {"code":"AI_USAGE_WAIVER","remaining":"10","unit":"REPORT","source":"GENESIS_NFT"}
    ],
    "limitations":["AI 使用仍受每分钟与每日公平使用限制"]
  },
  "request_id":"01JQ..."
}
~~~

AI_USAGE_WAIVER、TRADING_FEE_DISCOUNT 和 ANALYST_ACCESS 是不同权益。前端不自行根据 VIP1 推断免费规则，应展示服务端 policy version。

### 3.6 创建 AI 分析任务

~~~http
POST /v1/ai/runs
Authorization: Bearer eyJ...
Idempotency-Key: 01JAIRUN...

{
  "analyst_id":"momentum-v2",
  "instrument_id":"01JQBTCSPOT...",
  "lookback":"24h",
  "question":"结合过去 24 小时波动，给出风险和需要观察的价位",
  "client_context":{"language":"zh-CN"}
}
~~~

~~~json
{
  "data": {
    "run_id":"01JAIRUN...",
    "status":"QUEUED",
    "analyst_version":"momentum-v2.3",
    "reserved_credits":"1",
    "estimated_latency_seconds":20,
    "input_snapshot_at":"2026-09-21T08:10:00Z",
    "cost":{"unit":"CREDIT","amount":"1","waived":true}
  },
  "request_id":"01JQ..."
}
~~~

报告完成后，GET /v1/ai/runs/{run_id} 返回 report 和引用快照；报告 URL 是短时签名 URL。AI API 不接受自动下单或任意工具名称。

### 3.7 钱包挑战与验证（S2）

~~~http
POST /v1/auth/wallet/challenges
Content-Type: application/json

{"chain_id":"eip155:42161","address":"0xAbC...123","purpose":"BIND_WALLET"}
~~~

~~~json
{
  "data": {
    "challenge_id":"01JCHALLENGE...",
    "domain":"app.example.com",
    "uri":"https://app.example.com",
    "chain_id":"eip155:42161",
    "nonce":"8f6d...",
    "issued_at":"2026-09-21T08:15:00Z",
    "expiration_time":"2026-09-21T08:20:00Z",
    "message":"app.example.com wants you to sign in..."
  },
  "request_id":"01JQ..."
}
~~~

~~~http
POST /v1/auth/wallet/verify
Idempotency-Key: 01JWALLET...

{
  "challenge_id":"01JCHALLENGE...",
  "address":"0xAbC...123",
  "signature":"0x...",
  "create_session":true
}
~~~

服务端校验 message 的 domain/URI/nonce/purpose/chain、签名地址 checksum 和一次性消费。交易订单签名不能复用绑定 challenge。

### 3.8 订单意图、签名与提交（S2）

~~~http
POST /v1/order-intents
Idempotency-Key: 01JORDERINTENT...

{
  "wallet_id":"01JWALLET...",
  "venue":"HYPERLIQUID",
  "instrument_id":"01JQBTCUSD...",
  "side":"BUY",
  "order_type":"LIMIT",
  "quantity":"0.010",
  "limit_price":"69900.00",
  "slippage_bps":"30",
  "reduce_only":false
}
~~~

~~~json
{
  "data": {
    "intent_id":"01JINTENT...",
    "status":"AWAITING_SIGNATURE",
    "expires_at":"2026-09-21T08:16:00Z",
    "payload_hash":"sha256:...",
    "signing_payload":{"protocol":"venue-defined","chain_id":"...","nonce":"..."},
    "fees":{"venue":"0.00045","platform":"0.00010","gas":"0.00000","currency":"USDC"}
  },
  "request_id":"01JQ..."
}
~~~

~~~http
POST /v1/orders
Idempotency-Key: 01JORDER...

{"intent_id":"01JINTENT...","signature":"0x..."}
~~~

~~~json
{
  "data": {
    "order_id":"01JORDER...",
    "status":"SUBMISSION_UNKNOWN",
    "venue_order_id":null,
    "next_action":"POLL_STATUS"
  },
  "request_id":"01JQ..."
}
~~~

SUBMISSION_UNKNOWN 是有意暴露的安全状态：客户端必须查询订单或等待 WS 对账，不能自行再次提交。

### 3.9 平台币支付与 NFT 权益（S2）

先创建固定报价，客户端展示链、代币合约、金额、收款方、用途和过期时间，再由钱包签名/发送链上交易：

```http
POST /v1/payment-quotes
Idempotency-Key: 01JQUOTE...
Authorization: Bearer eyJ...

{
  "purpose":"UNLOCK_ANALYST",
  "analyst_id":"momentum-v2",
  "chain_id":"eip155:42161",
  "token_contract":"0xToken...",
  "currency":"PLATFORM_TOKEN"
}
```

```json
{
  "data": {
    "quote_id":"01JQUOTE...",
    "purpose":"UNLOCK_ANALYST",
    "chain_id":"eip155:42161",
    "token_contract":"0xToken...",
    "atomic_amount":"1250000000000000000",
    "display_amount":"1.25",
    "recipient":"0xTreasury...",
    "nonce":"payment-nonce-...",
    "expires_at":"2026-09-21T08:30:00Z",
    "policy_version":"ai-price-2026-09-01-01"
  },
  "request_id":"01JQ..."
}
```

链上发送完成后只提交交易证明，服务端再按 chain event、金额、代币、收款方和 finality 异步核验：

```http
POST /v1/payments
Idempotency-Key: 01JPAYMENT...

{"quote_id":"01JQUOTE...","tx_hash":"0xTxHash..."}
```

```json
{
  "data": {
    "payment_id":"01JPAYMENT...",
    "status":"PENDING_CONFIRMATION",
    "quote_id":"01JQUOTE...",
    "entitlement_status":"NOT_GRANTED",
    "required_confirmations":12
  },
  "request_id":"01JQ..."
}
```

只有 `CONFIRMED` 才发放 `ANALYST_ACCESS` 或 `AI_USAGE_WAIVER`。过期报价、错链/错币、少付、重复 log 和 reorg 返回 `EXCEPTION_REVIEW`，不自动重复发权益。NFT 查询使用 `GET /v1/me/entitlements`，其中 source 会标明 NFT 合约、token_id 和 finality；转移后由索引事件撤销或冻结权益，客户端不能自行声明“永久免费”。

## 4. WebSocket 协议

### 4.1 连接与订阅

1. 客户端先 POST /v1/stream/ticket，返回单次 ticket，约 30 秒有效。
2. 连接 wss://api.example.com/v1/stream?ticket=...。
3. 首帧发送订阅，服务端校验用户权限并返回 subscribed；同连接最多 50 个行情、5 个私有频道（初始策略）。

~~~json
{"op":"subscribe","channel":"quotes","instrument_ids":["01JQBTCSPOT..."],"price_type":"LAST"}
~~~

~~~json
{"op":"subscribe","channel":"private","topics":["alerts","notifications","ai_runs","orders"]}
~~~

### 4.2 事件 envelope 示例

~~~json
{
  "type":"alert.triggered",
  "event_id":"01JEVENT...",
  "occurred_at":"2026-09-21T08:20:00.220Z",
  "aggregate_id":"01JQALERT...",
  "sequence":42,
  "data":{
    "trigger_id":"01JTRIGGER...",
    "price":"72001.10",
    "threshold":"72000.00",
    "source_ts":"2026-09-21T08:20:00.120Z",
    "notification_id":"01JNOTICE..."
  }
}
~~~

客户端以 event_id 去重，断线后用 GET /v1/notifications?since= 或详情补偿。行情只提供最新快照，断线不补发每个 tick；规则、订单、AI 等私有事件提供 sequence 或状态查询。

### 4.3 心跳、断线和降级

服务端 20 秒 ping，客户端 10 秒内 pong；连续两次失败断开。客户端退避 1/2/4/8/30 秒并加入随机抖动，认证过期重新获取 ticket。WS 不可用时：读接口轮询 + 系统推送；两者都不可用时，客户端展示离线/未知状态，不声称价格提醒仍正常。

## 5. Provider Webhook / 内部接口

| 接口 | 认证 | 处理要求 |
|---|---|---|
| POST /internal/webhooks/push/{provider} | mTLS/签名、时间窗口 | 原始 body 校验，receipt 唯一，异步处理 |
| POST /internal/webhooks/billing/{provider} | provider 签名 | 收据/支付状态查询核验，不能直接 grant |
| POST /internal/market/{venue}/reconnect | 服务身份 | 运维审计、限时、指定场所 |
| GET /internal/health/live | 内网 | 只检查进程活着 |
| GET /internal/health/ready | 内网 | 检查可接流量的关键依赖 |

内部 API 只在私网，服务身份使用 mTLS/短期 workload token。管理接口另有域名和 RBAC，不能仅以路径字符串区分公网权限。

## 6. 状态机摘要

~~~mermaid
stateDiagram-v2
    [*] --> DRAFT
    DRAFT --> ACTIVE: 保存并激活
    ACTIVE --> PAUSED: 用户暂停
    PAUSED --> ACTIVE: 用户恢复
    ACTIVE --> TRIGGERED: 条件满足
    TRIGGERED --> COMPLETED: ONCE
    TRIGGERED --> WAIT_REARM: REARM
    WAIT_REARM --> ACTIVE: 进入迟滞区且冷却结束
    ACTIVE --> EXPIRED: expires_at
    WAIT_REARM --> EXPIRED: expires_at
    DRAFT --> DELETED: 删除
    ACTIVE --> DELETED: 删除
    PAUSED --> DELETED: 删除
~~~

Order：CREATED → AWAITING_SIGNATURE → SUBMITTING → OPEN / SUBMISSION_UNKNOWN → PARTIALLY_FILLED → FILLED。任何允许状态可转 CANCEL_REQUESTED → CANCELED，但成交事实可使结果为 PARTIALLY_FILLED。REJECTED、EXPIRED、FAILED 为终态；SUBMISSION_UNKNOWN 不允许创建替代订单。

AI Run：QUEUED → RUNNING → COMPLETED 或 FAILED/TIMED_OUT/CANCELED。预留额度在开始前占用；终态结算/释放必须幂等。客户端网络断开不改变任务状态。

## 7. 版本、幂等和兼容

- 路径主版本只在不兼容变化时增加；字段新增默认向后兼容，删除先标记 deprecated 并给迁移期。
- 幂等 key scope 至少为 user_id + method + path + key；同 key 不同 body hash 返回 409。支付/交易保留窗口覆盖外部场所最长未知期，之后仍可查询状态。
- 业务资源 version 是用户配置 CAS；事件 aggregate_version 是发布顺序；Kafka offset 不直接暴露给客户端。
- 客户端发送 X-Client-Version；服务端按 feature_flags 控制新枚举和新 UI。旧客户端至少能读取通知、查看并暂停规则。
- OpenAPI、JSON Schema、事件 schema 在 CI 做兼容性检查；生成 SDK 更新需在五端编译和 contract test 通过后发布。
