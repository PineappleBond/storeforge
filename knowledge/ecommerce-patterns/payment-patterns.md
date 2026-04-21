# 支付模式 (Payment Patterns)

## 微信支付 / 支付宝 / 抖音支付集成

---

### 触发条件

- 需要接入在线支付的电商订单流程
- 微信小程序/公众号/APP/H5 支付场景
- 需要处理支付回调、退款、对账
- 多支付渠道统一收单

---

### 决策树

```text
用户选择支付方式？
  ├─ 微信小程序 → 小程序支付（wx.requestPayment，服务端统一下单）
  ├─ 微信公众号/H5 → JSAPI 支付 / H5 支付
  ├─ APP（微信/支付宝）→ APP 支付
  ├─ 支付宝 → 支付宝当面扫码 / H5 / APP 支付
  ├─ 抖音小程序 → 抖音支付
  └─ 后台操作
      ├─ 退款 → 原路退款（需商户证书）
      └─ 对账 → 下载账单文件比对
```

---

### 推荐第三方库

**必须使用 SDK，禁止手写 HTTP + RSA 签名代码：**

| 支付渠道               | 推荐库                 | 导入路径                                       |
|--------------------|---------------------|--------------------------------------------|
| 微信支付                | wechatpay-go（官方）     | `github.com/wechatpay-apiv3/wechatpay-go`      |
| 支付宝                 | gopay               | `github.com/go-pay/gopay`                      |
| 抖音支付                | gopay（支持抖音）         | `github.com/go-pay/gopay`                      |
| 统一收单（聚合）            | gopay 多平台封装         | `github.com/go-pay/gopay`                      |

**为什么必须用 SDK：**
- 支付是安全敏感操作，手写 RSA 签名容易引入微妙漏洞（时序攻击、签名串格式错误、证书轮换失败）
- SDK 自动处理：mTLS、证书下载与轮换、请求签名、回调验签、响应解密、重试
- 手写 HTTP + crypto 签名代码增加 300+ 行，SDK 只需 30 行

---

### 代码模板

#### 2.1 微信支付集成（使用 wechatpay-go SDK）

**context key 定义（用于跨 middleware 传递数据）：**

```go
package payment

// contextKey 未导出的 context key 类型，防止键冲突
type contextKey struct{}
var wechatTransactionKey = contextKey{}
```

**服务初始化：**

```go
// internal/logic/payment/wechat_pay.go
package payment

import (
  "context"
  "crypto/rsa"
  "fmt"
  "time"

  "github.com/wechatpay-apiv3/wechatpay-go/core"
  "github.com/wechatpay-apiv3/wechatpay-go/core/notify"
  "github.com/wechatpay-apiv3/wechatpay-go/services/payments"
  "github.com/wechatpay-apiv3/wechatpay-go/services/payments/jsapi"
  "github.com/wechatpay-apiv3/wechatpay-go/services/refund"
  refundsSvc "github.com/wechatpay-apiv3/wechatpay-go/services/refunds"
  "gorm.io/gorm"
)

// WechatPayConfig 微信支付配置
type WechatPayConfig struct {
  MchID               string
  CertificateSerialNo string
  PrivateKey          *rsa.PrivateKey
  AppID               string
  NotifyURL           string
  RefundNotifyURL     string
  WechatPayOptions    []core.Option
}

// UnifiedOrderRequest 统一下单请求
type UnifiedOrderRequest struct {
  Description string
  OutTradeNo  string
  OpenID      string // 小程序/公众号支付必填
  TotalFee    int64  // 单位：分
}

// WechatMiniProgramPayParams 小程序支付参数（前端 wx.requestPayment 使用）
type WechatMiniProgramPayParams struct {
  AppID     string
  TimeStamp string
  NonceStr  string
  Package   string
  SignType  string
  PaySign   string
}

// RefundRequest 退款请求
type RefundRequest struct {
  OutTradeNo  string
  OutRefundNo string
  Reason      string
  RefundFee   int64 // 单位：分
  TotalFee    int64 // 单位：分
}

// RefundResponse 退款响应
type RefundResponse struct {
  RefundID    string
  OutRefundNo string
  Status      string
}

// Order 订单模型
type Order struct {
  ID            uint   `gorm:"primaryKey"`
  OutTradeNo    string `gorm:"uniqueIndex"`
  TenantID      string `gorm:"index"`
  Status        string
  TotalFee      int64
  PaidAt        *string
  TransactionID string
}

const (
  OrderStatusPaid = "paid"
)

// AuditLog 审计日志模型
type AuditLog struct {
  ID         uint `gorm:"primaryKey"`
  EntityType string
  EntityID   uint
  Action     string
  NewValue   string
  OperatorID string
  TenantID   string
  Reason     string
}

// Refund 退款记录模型
type Refund struct {
  ID          uint   `gorm:"primaryKey"`
  OutTradeNo  string `gorm:"index"`
  OutRefundNo string `gorm:"uniqueIndex"`
  RefundFee   int64
  TotalFee    int64
  Status      string
  Reason      string
}

// WechatPayService 微信支付服务（基于 wechatpay-go SDK）
type WechatPayService struct {
  client          *core.Client
  db              *gorm.DB // 用于退款前校验和退款记录
  jsapiSvc        *jsapi.NativePayService
  paymentsSvc     *payments.PaymentsApiService
  refundsSvc      *refundsSvc.RefundsApiService
  mchID           string
  appID           string
  notifyURL       string
  refundNotifyURL string
}

// NewWechatPayService 初始化微信支付服务
// SDK 自动处理：mTLS、证书下载与轮换、请求签名、响应解密
func NewWechatPayService(ctx context.Context, db *gorm.DB, opts WechatPayConfig) (*WechatPayService, error) {
  // 使用 SDK 提供的选项模式构建核心 Client
  // wechatpay-go 内置证书下载器（core/downloader）自动处理证书轮换
  coreClient, err := core.NewClient(
    ctx,
    opts.MchID,
    opts.CertificateSerialNo,
    opts.PrivateKey, // *rsa.PrivateKey，商户私钥
    opts.WechatPayOptions..., // 包含 APIv3Key、自动证书下载等选项
  )
  if err != nil {
    return nil, fmt.Errorf("init wechatpay client: %w", err)
  }

  return &WechatPayService{
    client:          coreClient,
    db:              db,
    jsapiSvc:        jsapi.NewNativePayService(coreClient),
    paymentsSvc:     payments.NewPaymentsApiService(coreClient),
    refundsSvc:      refundsSvc.NewRefundsApiService(coreClient),
    mchID:           opts.MchID,
    appID:           opts.AppID,
    notifyURL:       opts.NotifyURL,
    refundNotifyURL: opts.RefundNotifyURL,
  }, nil
}
```

#### 2.2 统一下单（小程序支付）

```go
// Prepay 小程序统一下单（SDK 封装了签名、证书、mTLS）
func (s *WechatPayService) Prepay(ctx context.Context, req UnifiedOrderRequest) (*WechatMiniProgramPayParams, error) {
  // 第三方 API 调用必须设置超时，防止服务不可用时阻塞
  ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
  defer cancel()

  // SDK 的 PrepayWithRequestPayment 方法：
  // 1. 调用微信统一下单 API 获取 prepay_id
  // 2. 自动用商户私钥签名小程序支付参数
  // 3. 返回前端 wx.requestPayment 所需的全部参数
  params, _, err := s.jsapiSvc.PrepayWithRequestPayment(ctx, jsapi.PrepayRequest{
    Appid:       core.String(s.appID),
    Mchid:       core.String(s.mchID),
    Description: core.String(req.Description),
    OutTradeNo:  core.String(req.OutTradeNo),
    NotifyUrl:   core.String(s.notifyURL),
    Amount: &jsapi.Amount{
      Total:    core.Int64(req.TotalFee), // 单位：分
      Currency: core.String("CNY"),
    },
    Payer: &jsapi.Payer{
      Openid: core.String(req.OpenID),
    },
  }, nil) // requestOptions 可选
  if err != nil {
    return nil, fmt.Errorf("wechat prepay: %w", err)
  }

  // params 已包含 appId, timeStamp, nonceStr, package, signType, paySign
  return &WechatMiniProgramPayParams{
    AppID:     params.AppId,
    TimeStamp: params.TimeStamp,
    NonceStr:  params.NonceStr,
    Package:   params.Package,
    SignType:  params.SignType,
    PaySign:   params.PaySign,
  }, nil
}
```

#### 2.3 支付回调验签（go-zero 集成 + SDK 自动处理）

```go
// internal/middleware/wechat_pay_callback.go
package middleware

import (
  "context"
  "net/http"

  "github.com/wechatpay-apiv3/wechatpay-go/core/notify"
  "github.com/wechatpay-apiv3/wechatpay-go/services/payments"
  "github.com/zeromicro/go-zero/core/logx"
)

// WechatPayCallbackVerify 微信支付回调验签中间件（go-zero 集成）
// 职责：仅做签名验证，将解密后的 transaction 注入 context
// 下游 HandleWechatPayCallback 从 context 读取 transaction，不再重复解析
func WechatPayCallbackVerify(
  decryptor notify.RSAAESEventDecryptor, // SDK 提供的解密器
) func(http.HandlerFunc) http.HandlerFunc {
  return func(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
      ctx := r.Context()
      logx.WithContext(ctx).Infof("wechat pay callback received, method=%s, url=%s", r.Method, r.URL.Path)

      // 1. 使用 SDK 验签 + 解密，一行代码完成所有校验
      transaction, err := decryptor.DecryptNotify(ctx, r)
      if err != nil {
        logx.WithContext(ctx).Errorf("wechat pay callback verify failed: %v", err)
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusForbidden)
        w.Write([]byte(`{"code":"FAIL","message":"签名验证失败"}`))
        return
      }

      // 2. 将解密后的 transaction 注入 context，供下游业务逻辑读取
      ctx = context.WithValue(ctx, wechatTransactionKey, transaction.(*payments.Transaction))
      next(w, r.WithContext(ctx))
    }
  }
}

// HandleWechatPayCallback 处理微信支付回调（业务逻辑）
// 从 context 读取已解密的 transaction，不再重新解析请求体
func (h *PaymentCallbackHandler) HandleWechatPayCallback(
  w http.ResponseWriter, r *http.Request,
) {
  ctx := r.Context()
  transaction, ok := ctx.Value(wechatTransactionKey).(*payments.Transaction)
  if !ok || transaction == nil {
    logx.WithContext(ctx).Error("transaction not found in context, middleware may have been skipped")
    http.Error(w, "internal error", http.StatusInternalServerError)
    return
  }

  // 从 transaction 获取业务字段
  outTradeNo := *transaction.OutTradeNo
  transactionID := *transaction.TransactionId
  totalFee := transaction.Amount.Total

  logx.WithContext(ctx).Infof(
    "processing payment callback: out_trade_no=%s, transaction_id=%s, total_fee=%d",
    outTradeNo, transactionID, totalFee,
  )

  // 后续业务逻辑：幂等检查 -> 金额校验 -> 更新订单状态
  // 调用 h.HandleCallback(ctx, outTradeNo, transactionID, totalFee)

  // 微信支付回调必须返回纯 "success" 字符串（不是 JSON），否则微信会重复发送回调
  w.Header().Set("Content-Type", "text/plain")
  w.Write([]byte("success"))
}
```

#### 2.4 支付宝集成（使用 gopay SDK）

```go
// internal/logic/payment/alipay.go
package payment

import (
  "context"
  "fmt"
  "net/http"
  "time"

  "github.com/go-pay/gopay"
  "github.com/go-pay/gopay/alipay"
  "github.com/shopspring/decimal"
)

// AlipayService 支付宝服务（基于 gopay SDK）
type AlipayService struct {
  client    *alipay.Client
  appID     string
  notifyURL string
}

// NewAlipayService 初始化支付宝服务
// gopay 自动处理：RSA2 签名、参数排序、证书校验
func NewAlipayService(appID, privateKey, appCertPath, notifyURL string) (*AlipayService, error) {
  client, err := alipay.NewClient(appID, privateKey, true)
  if err != nil {
    return nil, fmt.Errorf("init alipay client: %w", err)
  }

  // 设置证书模式（生产推荐）
  err = client.SetCertSnByPath(appCertPath, "", "")
  if err != nil {
    return nil, fmt.Errorf("set alipay cert: %w", err)
  }

  return &AlipayService{
    client:    client,
    appID:     appID,
    notifyURL: notifyURL,
  }, nil
}

// TradePreCreate 支付宝当面付/扫码下单
func (s *AlipayService) TradePreCreate(ctx context.Context, req UnifiedOrderRequest) (string, error) {
  // 第三方 API 调用必须设置超时
  ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
  defer cancel()

  bm := make(gopay.BodyMap)
  bm.Set("subject", req.Description)
  bm.Set("out_trade_no", req.OutTradeNo)
  bm.Set("total_amount", decimal.NewFromInt(req.TotalFee).Div(decimal.NewFromInt(100)).StringFixed(2))
  bm.Set("notify_url", s.notifyURL)

  // gopay 自动处理签名和参数排序
  resp, err := s.client.TradePreCreate(ctx, bm)
  if err != nil {
    return "", fmt.Errorf("alipay precreate: %w", err)
  }
  if resp.Code != alipay.Success {
    return "", fmt.Errorf("alipay error: %s", resp.SubMsg)
  }

  return resp.QrCode, nil
}

// VerifyCallback 支付宝回调验签（SDK 一行完成）
func (s *AlipayService) VerifyCallback(ctx context.Context, r *http.Request) (gopay.BodyMap, error) {
  // gopay 自动处理：参数排序、签名验证、证书校验
  return alipay.VerifySignWithCert(r.Form)
}
```

#### 2.5 退款流程（SDK 封装）

```go
// WechatRefund 微信退款（SDK 封装了 mTLS、签名、证书）
func (s *WechatPayService) WechatRefund(ctx context.Context, req RefundRequest) (*RefundResponse, error) {
  // 第三方 API 调用必须设置超时
  ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
  defer cancel()

  // 退款前校验：累计退款不能超过订单总额（坑 4 的代码实现）
  var totalRefunded int64
  if err := s.db.WithContext(ctx).Model(&Refund{}).
    Where("out_trade_no = ? AND status = ?", req.OutTradeNo, "success").
    Select("COALESCE(SUM(refund_fee), 0)").
    Row().Scan(&totalRefunded); err != nil {
    return nil, fmt.Errorf("query total refunded: %w", err)
  }
  if totalRefunded + req.RefundFee > req.TotalFee {
    return nil, fmt.Errorf("refund amount %d exceeds remaining order amount %d (total=%d, alreadyRefunded=%d)",
      req.RefundFee, req.TotalFee-totalRefunded, req.TotalFee, totalRefunded)
  }

  // SDK 的 Refund 方法：
  // 1. 自动使用商户证书（mTLS）
  // 2. 自动请求签名
  // 3. 自动处理响应解密
  refundResult, _, err := s.refundsSvc.Create(ctx, refundsSvc.CreateRequest{
    OutTradeNo:  core.String(req.OutTradeNo),
    OutRefundNo: core.String(req.OutRefundNo),
    Reason:      core.String(req.Reason),
    NotifyUrl:   core.String(s.refundNotifyURL),
    Amount: &refund.RefundAmount{
      Refund:   core.Int64(req.RefundFee),
      Total:    core.Int64(req.TotalFee),
      Currency: core.String("CNY"),
    },
  }, nil)
  if err != nil {
    return nil, fmt.Errorf("wechat refund: %w", err)
  }

  return &RefundResponse{
    RefundID:    *refundResult.RefundId,
    OutRefundNo: *refundResult.OutRefundNo,
    Status:      *refundResult.Status,
  }, nil
}
```

#### 2.6 支付回调幂等处理（redsync + 多租户 + 可观测性）

```go
// internal/logic/payment/callback_handler.go
package payment

import (
  "context"
  "fmt"
  "time"

  "github.com/go-redsync/redsync/v4"
  "github.com/go-redsync/redsync/v4/redis/goredis/v9"
  "github.com/redis/go-redis/v9"
  "github.com/zeromicro/go-zero/core/logx"
  "gorm.io/gorm"
)

// PaymentCallbackHandler 支付回调处理器（支持多租户隔离）
type PaymentCallbackHandler struct {
  rdb      *redis.Client
  db       *gorm.DB
  rs       *redsync.Redsync // redsync 分布式锁管理器
  tenantID string           // 租户 ID，用于隔离锁键和数据查询
}

// NewPaymentCallbackHandler 创建回调处理器，初始化 redsync
func NewPaymentCallbackHandler(rdb *redis.Client, db *gorm.DB, tenantID string) *PaymentCallbackHandler {
  pool := goredis.NewPool(rdb)
  rs := redsync.New(pool)
  return &PaymentCallbackHandler{
    rdb:      rdb,
    db:       db,
    rs:       rs,
    tenantID: tenantID,
  }
}

// HandleCallback 处理支付回调（核心幂等逻辑）
func (h *PaymentCallbackHandler) HandleCallback(
  ctx context.Context,
  outTradeNo string,
  transactionID string,
  totalFee int64,
) error {
  logx.WithContext(ctx).Infof(
    "[tenant=%s] processing payment callback: out_trade_no=%s, txn_id=%s",
    h.tenantID, outTradeNo, transactionID,
  )

  // 1. redsync 分布式锁（带重试配置），确保幂等
  // 锁键包含 tenantID，实现多租户隔离
  lockKey := fmt.Sprintf("pay:callback:lock:%s:%s", h.tenantID, outTradeNo)
  mutex := h.rs.NewMutex(lockKey,
    redsync.WithExpiry(30*time.Second),
    redsync.WithTries(3),       // 最多重试 3 次
    redsync.WithRetryDelay(50*time.Millisecond), // 重试间隔
  )

  if err := mutex.LockContext(ctx); err != nil {
    // 获取锁失败：可能是并发请求正在处理，记录日志后返回
    logx.WithContext(ctx).Errorf(
      "[tenant=%s] acquire lock failed for out_trade_no=%s: %v",
      h.tenantID, outTradeNo, err,
    )
    return nil // 正在处理中，幂等返回
  }
  defer func() {
    if _, err := mutex.UnlockContext(ctx); err != nil {
      logx.WithContext(ctx).Errorf("release lock failed: %v", err)
    }
  }()

  // 2. 查询订单（带 tenant_id 过滤，多租户隔离；ctx 透传用于 tracing）
  var order Order
  if err := h.db.WithContext(ctx).
    Where("out_trade_no = ? AND tenant_id = ?", outTradeNo, h.tenantID).
    First(&order).Error; err != nil {
    return fmt.Errorf("query order: %w", err)
  }
  if order.Status == OrderStatusPaid {
    return nil // 已处理，幂等返回
  }

  // 3. 验证金额（int64 分直接比较，无需 decimal 转换）
  if order.TotalFee != totalFee {
    logx.WithContext(ctx).Errorf(
      "[tenant=%s] amount mismatch: out_trade_no=%s, expected=%d cents, got=%d cents",
      h.tenantID, outTradeNo, order.TotalFee, totalFee,
    )
    return fmt.Errorf("amount mismatch for out_trade_no=%s: expected %d cents, got %d cents",
      outTradeNo, order.TotalFee, totalFee)
  }

  // 4. 事务更新：订单状态 + 审计日志（ctx 透传用于 tracing）
  return h.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
    if err := tx.Model(&Order{}).
      Where("out_trade_no = ? AND tenant_id = ?", outTradeNo, h.tenantID).
      Updates(map[string]interface{}{
        "status":         OrderStatusPaid,
        "paid_at":        time.Now().UTC(),
        "transaction_id": transactionID,
      }).Error; err != nil {
      return fmt.Errorf("update order: %w", err)
    }
    return tx.Create(&AuditLog{
      EntityType: "order",
      EntityID:   order.ID,
      Action:     "payment_received",
      NewValue:   string(OrderStatusPaid),
      OperatorID: "system",
      TenantID:   h.tenantID,
      Reason:     fmt.Sprintf("wechat pay callback, txn=%s", transactionID),
    }).Error
  })
}
```

---

### 反模式

| 反模式 | 风险 | 正确做法 |
|--------|------|----------|
| 手写 HTTP + RSA 签名调用支付 API | 签名格式错误、证书轮换失败、安全漏洞 | 使用 wechatpay-go / gopay SDK |
| 不验签直接处理回调 | 攻击者伪造回调，零元购 | SDK 内置验签器，一行完成 |
| 回调不做幂等处理 | 同一回调多次到达导致重复发货 | 使用分布式锁 + 订单状态检查确保幂等 |
| 存储用户银行卡/支付信息 | PCI-DSS 合规风险，数据泄漏 | 绝不存储，使用支付平台 Token 化方案 |
| 回调不验证金额 | 攻击者修改金额字段 | 对比回调金额与订单金额，不一致则拒绝 |
| 手动管理支付平台证书 | 证书轮换导致验签突然失败 | SDK 内置证书下载器，自动轮换（见坑 3） |
| 退款不幂等 | 重复退款，资金损失 | 退款单号作为幂等键，同一单号只退一次 |
| 回调日志不记录 | 出问题无法追溯 | 所有回调请求完整记录（headers + body + 验签结果） |
| 小程序 code 在前端直接换 token | code 泄漏导致账号被盗 | code 必须发送到服务端，服务端调用 code2Session |

---

### 常见坑

**坑 1：手写签名串格式错误**
- **现象**：签名总是验证失败
- **原因**：签名串末尾多/少换行，URL 路径包含 query 参数
- **解决**：使用 SDK（wechatpay-go）自动构造签名串，不手写

**坑 2：支付宝回调返回 "success" 字符串**
- **现象**：支付宝一直重复发送回调
- **原因**：回调必须返回纯文本 `success`（不是 JSON），否则支付宝认为处理失败
- **解决**：`fmt.Fprint(w, "success")`，不要返回 JSON

**坑 3：证书轮换导致验签突然失败**
- **现象**：某天起所有回调验签失败
- **原因**：微信/支付宝平台证书已轮换
- **解决**：使用 SDK 内置证书下载器（core/downloader），自动检测并下载新证书

**坑 4：退款金额超过原订单金额**
- **现象**：微信返回 "退款金额超过订单金额"
- **原因**：部分退款累计金额超过订单总额
- **解决**：退款前查询已退款总额，确保 `已退 + 本次退 <= 订单总额`

**坑 5：高并发下回调与主动查询竞态**
- **现象**：回调还没处理，用户刷新页面触发主动查询，查到已支付状态后发货；随后回调到达再次发货
- **解决**：主动查询和回调都走同一个幂等处理逻辑，使用分布式锁保证只有一个路径执行发货

---

### 测试模板

#### 3.1 使用 httptest 模拟微信支付回调

```go
// internal/logic/payment/callback_handler_test.go
package payment

import (
  "bytes"
  "encoding/json"
  "net/http"
  "net/http/httptest"
  "testing"

  "github.com/alicebob/miniredis/v2"
  "github.com/redis/go-redis/v9"
  "github.com/zeromicro/go-zero/core/logx"
  "gorm.io/driver/sqlite"
  "gorm.io/gorm"
)

func init() {
  logx.DisableStat() // 测试环境关闭统计日志
}

// WechatCallbackFixture 微信支付回调测试 fixture
type WechatCallbackFixture struct {
  ID           string                   `json:"id"`
  CreateTime   string                   `json:"create_time"`
  ResourceType string                   `json:"resource_type"`
  EventType    string                   `json:"event_type"`
  Summary      string                   `json:"summary"`
  Resource     *WechatCallbackResource  `json:"resource"`
}

// WechatCallbackResource 回调资源
type WechatCallbackResource struct {
  OriginalType   string `json:"original_type"`
  Algorithm      string `json:"algorithm"`
  Ciphertext     string `json:"ciphertext"`
  AssociatedData string `json:"associated_data"`
  Nonce          string `json:"nonce"`
}

// TestHandleWechatPayCallback_Success 测试正常回调处理
func TestHandleWechatPayCallback_Success(t *testing.T) {
  // 1. 构建 golden fixture JSON（模拟微信支付回调 payload）
  fixture := WechatCallbackFixture{
    ID:             "EV-202604201234567890",
    CreateTime:     "2026-04-20T12:34:56+08:00",
    ResourceType:   "transaction",
    EventType:      "TRANSACTION.SUCCESS",
    Resource: &WechatCallbackResource{
      OriginalType:    "transaction",
      Algorithm:       "AEAD_AES_256_GCM",
      Ciphertext:      "", // 测试环境可直接使用明文或 mock 解密器
      AssociatedData:  "微信支付回调",
      Nonce:           "abc123",
    },
  }

  body, _ := json.Marshal(fixture)
  req := httptest.NewRequest(http.MethodPost, "/pay/callback/wechat", bytes.NewReader(body))
  req.Header.Set("Content-Type", "application/json")
  req.Header.Set("Wechatpay-Serial", "FAKE_SERIAL_NO")
  req.Header.Set("Wechatpay-Signature", "FAKE_SIGNATURE")
  req.Header.Set("Wechatpay-Timestamp", "1713590096")
  req.Header.Set("Wechatpay-Nonce", "abc123")

  w := httptest.NewRecorder()

  // 2. 调用 handler（测试环境使用 mock 的 validator，始终通过验签）
  mockRDB := miniredis.RunT(t)
  rdb := redis.NewClient(&redis.Options{Addr: mockRDB.Addr()})
  mockDB, _ := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{})
  mockDB.AutoMigrate(&Order{}, &AuditLog{})

  handler := NewPaymentCallbackHandler(rdb, mockDB, "tenant-001")
  handler.HandleWechatPayCallback(w, req)

  // 3. 断言响应
  resp := w.Result()
  if resp.StatusCode != http.StatusOK {
    t.Errorf("expected status 200, got %d", resp.StatusCode)
  }
  if w.Body.String() != "success" {
    t.Errorf("expected body 'success', got %q", w.Body.String())
  }
}
```

#### 3.2 Golden Fixture JSON（微信支付回调标准格式）

```json
{
  "id": "EV-202604201234567890",
  "create_time": "2026-04-20T12:34:56+08:00",
  "resource_type": "transaction",
  "event_type": "TRANSACTION.SUCCESS",
  "summary": "支付成功",
  "resource": {
    "original_type": "transaction",
    "algorithm": "AEAD_AES_256_GCM",
    "ciphertext": "<加密后的交易数据>",
    "associated_data": "微信支付回调",
    "nonce": "abc123def456"
  }
}
```

**说明**：

- `ciphertext` 是 AES-256-GCM 加密的 JSON，解密后为 `payments.Transaction` 对象
- 测试时使用 mock 解密器直接返回明文 Transaction，跳过真实 AES 解密

构造测试用例时覆盖以下场景：

- 正常支付成功（SUCCESS）
- 金额不匹配（构造错误的 total_fee）
- 重复回调（相同 out_trade_no，锁已被持有）
- 验签失败（修改 Wechatpay-Signature header）
- 多租户隔离（tenantA 的回调不应影响 tenantB 的订单）

### 覆盖率目标

| 层级 | 覆盖率要求 | 说明 |
|------|-----------|------|
| L1 Unit | >= 80% | 金额校验、幂等检查、分布式锁、支付状态机 |
| L2 Integration | >= 70% | miniredis 锁 + GORM SQLite 订单更新 + 多租户隔离 |
| L3 E2E | 核心流程 | 用户下单 → 支付 → 回调验签 → 订单状态更新 |

### 并发支付回调测试

```go
func TestPaymentCallback_ConcurrentCallback(t *testing.T) {
	// 使用 miniredis + redsync 测试并发回调
	// 同一 out_trade_no 被多个支付渠道同时回调
	// 验证幂等性：只有一个请求能成功更新订单状态
	// 其余请求被分布式锁拒绝或检测到已支付后返回 success
}
```

### 支付超时边界测试

```go
func TestPayment_TimeoutBoundary(t *testing.T) {
	// 订单在支付超时前 1 秒发起支付
	// 回调在超时后 1 秒到达
	// 验证订单状态一致性（支付成功应覆盖超时状态）
}
```

### 多租户隔离测试

```go
func TestPayment_MultiTenantIsolation(t *testing.T) {
	// tenantA 的支付回调不应修改 tenantB 的订单
	// 分布式锁 key 包含 tenantID：pay:callback:lock:{tenantID}:{outTradeNo}
}
```
