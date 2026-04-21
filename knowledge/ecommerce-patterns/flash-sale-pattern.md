# Flash-Sale Pattern（秒杀/抢购模式）

## 触发条件（When to use）

- 限时限量抢购场景（如：双11 秒杀、限量款首发、定时开抢）
- 预期 QPS 远超日常水平（10x~100x 峰值）
- 库存固定且有限，需要防止超卖
- 需要保证公平性（先抢先得，不允许插队）

## 决策树（Decision tree）

```
是否有固定库存上限？
├── 否 → 不使用秒杀模式，走普通下单流程
└── 是 → 预计 QPS 是否超过日常 5 倍？
         ├── 否 → 普通库存扣减 + Redis 缓存即可
         └── 是 → 使用完整秒杀架构
                  │
                  ├─ 是否需要公平排队？
                  │   ├── 是 → Redis Lua 预分配 + MQ 异步下单
                  │   └── 否 → 直接 Redis 扣减 + 同步下单（仅限中等流量）
                  │
                  └─ 是否需要预售 token 分发？
                      ├── 是（万人级）→ 预售 token + 分层限流 + 排队
                      └── 否（千人级）→ 直接 Redis 预热 + 限流
```

## 代码模板（Code template）

### 1. Redis 库存预热 + Lua 预分配脚本

```go
// flashsale/prewarm.go
package flashsale

import (
	"context"
	"fmt"
	"time"

	"github.com/zeromicro/go-zero/core/stores/redis"
)

const (
	// Key builders — all scoped by tenantID for multi-tenant isolation
	StockKeyPrefix    = "flash:{%s}:stock:"     // flash:{tenantID}:stock:{skuID}
	SoldKeyPrefix     = "flash:{%s}:sold:"      // flash:{tenantID}:sold:{skuID}
	QueueKeyPrefix    = "flash:{%s}:queue:"     // flash:{tenantID}:queue:{skuID}
	LimitKeyPrefix    = "flash:{%s}:limit:"     // flash:{tenantID}:limit:{skuID}:{userID}
	StatusKeyPrefix   = "flash:{%s}:status:"    // flash:{tenantID}:status:{skuID}
	TokenKeyPrefix    = "flash:{%s}:token:"     // flash:{tenantID}:token:{skuID}:{userID}
	AllocKeyPrefix    = "flash:{%s}:alloc:"     // flash:{tenantID}:alloc:{token}
	IdempotentKeyPrefix = "flash:{%s}:idempotent:" // flash:{tenantID}:idempotent:{key}
	RetryKeyPrefix      = "flash:{%s}:retry:"      // flash:{tenantID}:retry:{key}
	DegradationKeyPrefix = "flash:{%s}:degradation:" // flash:{tenantID}:degradation:{field}
)

// FlashSalePreWarm 秒杀库存预热服务
type FlashSalePreWarm struct {
	rdb *redis.Redis
}

func NewFlashSalePreWarm(rdb *redis.Redis) *FlashSalePreWarm {
	return &FlashSalePreWarm{rdb: rdb}
}

func (s *FlashSalePreWarm) PrewarmSKU(ctx context.Context, tenantID string, skuID int64, stock int64, ttl time.Duration) error {
	stockKey := fmt.Sprintf(StockKeyPrefix, tenantID) + fmt.Sprintf("%d", skuID)
	soldKey := fmt.Sprintf(SoldKeyPrefix, tenantID) + fmt.Sprintf("%d", skuID)
	statusKey := fmt.Sprintf(StatusKeyPrefix, tenantID) + fmt.Sprintf("%d", skuID)

	ttlSec := int(ttl.Seconds())
	// Use pipeline for atomic batch write
	pipe := s.rdb.Pipeline()
	pipe.SetnxExCtx(ctx, stockKey, stock, ttlSec)
	pipe.SetnxExCtx(ctx, soldKey, 0, ttlSec)
	pipe.SetnxExCtx(ctx, statusKey, 0, ttlSec)
	_, err := pipe.ExecCtx(ctx)
	if err != nil {
		return fmt.Errorf("pipeline prewarm sku %d: %w", skuID, err)
	}
	return nil
}

// BatchPrewarm 批量预热多个 SKU — 使用 Pipeline 减少 RTT
func (s *FlashSalePreWarm) BatchPrewarm(ctx context.Context, tenantID string, items []struct {
	SKU   int64
	Stock int64
}, ttl time.Duration) error {
	ttlSec := int(ttl.Seconds())
	pipe := s.rdb.Pipeline()

	for _, item := range items {
		stockKey := fmt.Sprintf(StockKeyPrefix, tenantID) + fmt.Sprintf("%d", item.SKU)
		soldKey := fmt.Sprintf(SoldKeyPrefix, tenantID) + fmt.Sprintf("%d", item.SKU)
		statusKey := fmt.Sprintf(StatusKeyPrefix, tenantID) + fmt.Sprintf("%d", item.SKU)

		pipe.SetnxExCtx(ctx, stockKey, item.Stock, ttlSec)
		pipe.SetnxExCtx(ctx, soldKey, 0, ttlSec)
		pipe.SetnxExCtx(ctx, statusKey, 0, ttlSec)
	}

	_, err := pipe.ExecCtx(ctx)
	if err != nil {
		return fmt.Errorf("batch prewarm pipeline exec: %w", err)
	}

	return nil
}
```

### 2. Redis Lua 库存预分配脚本（防超卖核心）

```go
// flashsale/alloc.go
package flashsale

import (
	"context"
	"errors"
	"fmt"
	"strings"

	"github.com/zeromicro/go-zero/core/stores/redis"
)

var (
	ErrStockEmpty            = errors.New("库存不足")
	ErrUserLimited           = errors.New("请求过于频繁，请稍后重试")
	ErrPurchaseLimitExceeded = errors.New("已达限购数量")
	ErrSaleNotStart          = errors.New("秒杀未开始")
	ErrSaleEnded             = errors.New("秒杀已结束")
	ErrTokenInvalid          = errors.New("预售 token 无效")
)

// LuaAllocScript 原子化库存预分配 Lua 脚本
// 参数: KEYS[1]=stock, KEYS[2]=sold, KEYS[3]=limit, KEYS[4]=status, KEYS[5]=alloc
//       ARGV[1]=skuID, ARGV[2]=userID, ARGV[3]=quantity, ARGV[4]=maxPerUser
// 返回值: 成功时返回 token 字符串; 失败时返回错误码 "1"=库存不足, "2"=限购, "3"=未开始, "4"=已结束
const LuaAllocScript = `
local stock = tonumber(redis.call('GET', KEYS[1]))
local sold  = tonumber(redis.call('GET', KEYS[2]))
local limit = tonumber(redis.call('GET', KEYS[3]) or "0")
local status = tonumber(redis.call('GET', KEYS[4]))

-- 状态检查
if status == 0 then return "3" end
if status == 2 then return "4" end

-- 库存检查
local remaining = stock - sold
if remaining < tonumber(ARGV[3]) then return "1" end

-- 限购检查
local maxPerUser = tonumber(ARGV[4])
if limit + tonumber(ARGV[3]) > maxPerUser then return "2" end

-- 原子扣减
redis.call('INCRBY', KEYS[2], ARGV[3])
redis.call('INCRBY', KEYS[3], ARGV[3])

-- 记录预分配 token（用于后续异步下单校验），返回完整 token 供调用方使用
local token = ARGV[1] .. ":" .. ARGV[2] .. ":" .. tostring(redis.call('INCR', KEYS[5] .. ":alloc_seq"))
redis.call('SETEX', KEYS[5] .. ":" .. token, 300, ARGV[3])

return token
`

// FlashSaleAllocator 秒杀库存分配器
type FlashSaleAllocator struct {
	rdb      *redis.Redis
	allocSHA string
}

func NewFlashSaleAllocator(rdb *redis.Redis) *FlashSaleAllocator {
	return &FlashSaleAllocator{rdb: rdb}
}

// LoadScript 预加载 Lua 脚本到 Redis
func (a *FlashSaleAllocator) LoadScript(ctx context.Context) error {
	sha, err := a.rdb.ScriptLoadCtx(ctx, LuaAllocScript)
	if err != nil {
		return fmt.Errorf("load lua script failed: %w", err)
	}
	a.allocSHA = sha
	return nil
}

// AllocateResult 分配结果
type AllocateResult struct {
	Success bool
	Token   string
	Code    int // 0=成功, 1=库存不足, 2=限购, 3=未开始, 4=已结束
}

// Allocate 执行库存预分配
func (a *FlashSaleAllocator) Allocate(ctx context.Context, tenantID string, skuID, userID int64, quantity, maxPerUser int64) (*AllocateResult, error) {
	stockKey := fmt.Sprintf(StockKeyPrefix, tenantID) + fmt.Sprintf("%d", skuID)
	soldKey := fmt.Sprintf(SoldKeyPrefix, tenantID) + fmt.Sprintf("%d", skuID)
	limitKey := fmt.Sprintf(LimitKeyPrefix, tenantID) + fmt.Sprintf("%d:%d", skuID, userID)
	statusKey := fmt.Sprintf(StatusKeyPrefix, tenantID) + fmt.Sprintf("%d", skuID)
	allocKey := fmt.Sprintf(AllocKeyPrefix, tenantID)

	result, err := a.rdb.EvalShaCtx(ctx, a.allocSHA, []string{stockKey, soldKey, limitKey, statusKey, allocKey},
		skuID, userID, quantity, maxPerUser,
	).Text()
	if err != nil {
		// 脚本未加载，fallback 到 load + eval
		if strings.Contains(err.Error(), "NOSCRIPT") {
			if loadErr := a.LoadScript(ctx); loadErr != nil {
				return nil, loadErr
			}
			return a.Allocate(ctx, tenantID, skuID, userID, quantity, maxPerUser)
		}
		return nil, fmt.Errorf("allocate failed: %w", err)
	}

	// Lua 返回 token 字符串表示成功，数字字符串表示错误码
	ret := &AllocateResult{Code: 0, Token: result}
	switch result {
	case "1":
		ret.Code = 1
		ret.Token = ""
	case "2":
		ret.Code = 2
		ret.Token = ""
	case "3":
		ret.Code = 3
		ret.Token = ""
	case "4":
		ret.Code = 4
		ret.Token = ""
	default:
		ret.Success = true
	}

	return ret, nil
}
```

### 3. go-zero 限流器（core/limit）

```go
// flashsale/ratelimiter.go
package flashsale

import (
	"context"
	"fmt"

	"github.com/zeromicro/go-zero/core/limit"
	"github.com/zeromicro/go-zero/core/stores/redis"
)

// RateLimiter 基于 go-zero core/limit 的全局 + 用户级双重限流
type RateLimiter struct {
	redisLimit *limit.PeriodLimit // 全局限流
	userLimit  *limit.PeriodLimit // 用户限流
	rdb        *redis.Redis
	tenantID   string
}

func NewRateLimiter(rdb *redis.Redis, tenantID string, opts RateLimitConfig) *RateLimiter {
	return &RateLimiter{
		redisLimit: limit.NewPeriodLimit(1, opts.GlobalQPS, rdb, fmt.Sprintf("flash:%s:ratelimit:global", tenantID)),
		userLimit:  limit.NewPeriodLimit(1, opts.UserQPS, rdb, fmt.Sprintf("flash:%s:ratelimit:user", tenantID)),
		rdb:        rdb,
		tenantID:   tenantID,
	}
}

// RateLimitConfig 限流配置（按租户配置，避免硬编码）
type RateLimitConfig struct {
	GlobalQPS int // 全局限流 QPS，按租户差异化配置
	UserQPS   int // 用户限流 QPS
}

// DefaultRateLimitConfig 默认限流配置
func DefaultRateLimitConfig() RateLimitConfig {
	return RateLimitConfig{GlobalQPS: 10000, UserQPS: 5}
}

// Allow 检查是否允许请求通过（全局 + 用户级双重限流）
func (rl *RateLimiter) Allow(ctx context.Context, skuID, userID int64) (allowed bool, err error) {
	// 全局限流：每 SKU 每秒最多配置的 QPS 次
	globalKey := fmt.Sprintf("flash:%s:ratelimit:global:%d", rl.tenantID, skuID)
	code, err := rl.redisLimit.TakeCtx(ctx, globalKey)
	if err != nil {
		return false, fmt.Errorf("global rate limit check failed: %w", err)
	}
	if code != limit.Allowed {
		return false, nil
	}

	// 用户限流：每用户每秒最多 5 次
	userKey := fmt.Sprintf("flash:%s:ratelimit:user:%d:%d", rl.tenantID, skuID, userID)
	code, err = rl.userLimit.TakeCtx(ctx, userKey)
	if err != nil {
		return false, fmt.Errorf("user rate limit check failed: %w", err)
	}
	if code != limit.Allowed {
		return false, nil
	}

	return true, nil
}
```

### 4. MQ 异步订单处理器

```go
// flashsale/consumer.go
package flashsale

import (
	"context"
	"encoding/json"
	"fmt"
	"time"

	"github.com/zeromicro/go-zero/core/logx"
	"github.com/zeromicro/go-zero/core/stores/redis"
	"gorm.io/gorm"
)

// FlashSaleOrderQueue returns the MQ queue name scoped to tenant
func FlashSaleOrderQueue(tenantID string) string {
	return fmt.Sprintf("flash.%s.order.create", tenantID)
}

// FlashSaleOrderDLQ returns the DLQ name scoped to tenant
func FlashSaleOrderDLQ(tenantID string) string {
	return fmt.Sprintf("flash.%s.order.dlq", tenantID)
}

// OrderCreateTimeout 订单创建超时
const OrderCreateTimeout = 30 * time.Second

// FlashSaleOrderMessage MQ 消息体
type FlashSaleOrderMessage struct {
	SKUID     int64  `json:"skuId"`
	UserID    int64  `json:"userId"`
	Quantity  int64  `json:"quantity"`
	AllocToken string `json:"allocToken"` // Redis 预分配 token
	IdempotencyKey string `json:"idempotencyKey"`
	Timestamp int64  `json:"timestamp"`
}

// FlashSaleConsumer MQ 异步下单消费者
type FlashSaleConsumer struct {
	rdb        *redis.Redis
	db         *gorm.DB
	producer   MessageProducer // MQ producer 接口
	maxRetries int
	tenantID   string
}

func NewFlashSaleConsumer(rdb *redis.Redis, db *gorm.DB, producer MessageProducer, tenantID string) *FlashSaleConsumer {
	return &FlashSaleConsumer{
		rdb:        rdb,
		db:         db,
		producer:   producer,
		maxRetries: 3,
		tenantID:   tenantID,
	}
}

// MessageProducer MQ 生产者接口（可替换为 RabbitMQ/Kafka/NATS 实现）
type MessageProducer interface {
	Publish(ctx context.Context, queue string, body []byte) error
}

// Consume 消费下单消息（在 MQ consumer goroutine 中调用）
func (c *FlashSaleConsumer) Consume(ctx context.Context, msg *FlashSaleOrderMessage) error {
	// 1. 幂等检查（基于状态，避免失败重试被误判为已处理）
	idempotentKey := fmt.Sprintf(IdempotentKeyPrefix, c.tenantID) + msg.IdempotencyKey
	existingStatus, err := c.rdb.GetCtx(ctx, idempotentKey)
	if err == nil && existingStatus == "done" {
		logx.WithContext(ctx).Infow("duplicate message",
			logx.Field("idempotentKey", msg.IdempotencyKey),
		)
		return nil // 已成功处理，幂等返回
	}

	// 2. 校验预分配 token（确保 token 未过期）
	tokenKey := fmt.Sprintf(AllocKeyPrefix, c.tenantID) + msg.AllocToken
	_, err = c.rdb.GetCtx(ctx, tokenKey)
	if err != nil {
		return fmt.Errorf("alloc token expired or invalid: %s", msg.AllocToken)
	}

	// 3. 在事务中创建订单 + 扣减 PG 库存（仅 DB 操作，不含 Redis）
	ctx, cancel := context.WithTimeout(ctx, OrderCreateTimeout)
	defer cancel()

	err = c.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
		// 3.1 乐观锁扣减 PG 库存
		var stockRecord struct {
			Version int `gorm:"column:version"`
		}
		tx.Table("sku_stock").Select("version").Where("sku_id = ?", msg.SKUID).First(&stockRecord)
		result := tx.Exec(
			"UPDATE sku_stock SET stock = stock - ?, version = version + 1 WHERE sku_id = ? AND stock >= ? AND version = ?",
			msg.Quantity, msg.SKUID, msg.Quantity, stockRecord.Version,
		)
		if result.Error != nil {
			return fmt.Errorf("pg stock update failed: %w", result.Error)
		}
		if result.RowsAffected == 0 {
			return fmt.Errorf("pg stock insufficient for sku %d", msg.SKUID)
		}

		// 3.2 创建订单
		order := &Order{
			UserID:    msg.UserID,
			SKUID:     msg.SKUID,
			Quantity:  msg.Quantity,
			Status:    OrderStatusPending,
			Type:      OrderTypeFlashSale,
			CreatedAt: time.Now(),
		}
		if txErr := tx.Create(order).Error; txErr != nil {
			return fmt.Errorf("create order failed: %w", txErr)
		}

		return nil // DB 事务成功提交
	})

	if err != nil {
		// 4. DB 事务失败 → 重试
		retryKey := fmt.Sprintf(RetryKeyPrefix, c.tenantID) + msg.IdempotencyKey
		retryCount, rerr := c.rdb.IncrCtx(ctx, retryKey)
		if rerr != nil {
			logx.WithContext(ctx).Errorw("retry counter failed",
				logx.Field("error", rerr),
				logx.Field("idempotentKey", msg.IdempotencyKey),
			)
			return fmt.Errorf("incr retry counter: %w", rerr)
		}
		c.rdb.ExpireCtx(ctx, retryKey, 3600)

		if int(retryCount) <= c.maxRetries {
			body, merr := json.Marshal(msg)
			if merr != nil {
				return fmt.Errorf("marshal msg for retry: %w", merr)
			}
			if perr := c.producer.Publish(ctx, FlashSaleOrderQueue(c.tenantID), body); perr != nil {
				return fmt.Errorf("publish retry: %w", perr)
			}
			return fmt.Errorf("message retried, count=%d", retryCount)
		}

		body, merr := json.Marshal(msg)
		if merr != nil {
			return fmt.Errorf("marshal msg for dlq: %w", merr)
		}
		if perr := c.producer.Publish(ctx, FlashSaleOrderDLQ(c.tenantID), body); perr != nil {
			return fmt.Errorf("publish to dlq: %w", perr)
		}
		return fmt.Errorf("message moved to dlq after %d retries", c.maxRetries)
	}

	// 5. DB 事务成功 → Redis 清理（仅在事务提交后执行）
	c.rdb.DelCtx(ctx, tokenKey)
	c.rdb.DelCtx(ctx, fmt.Sprintf(RetryKeyPrefix, c.tenantID)+msg.IdempotencyKey)
	c.rdb.SetExCtx(ctx, idempotentKey, "done", 86400)

	return nil
}

// Order 订单模型（简化）
type Order struct {
	ID        int64     `gorm:"primaryKey;autoIncrement"`
	UserID    int64     `gorm:"not null;index"`
	SKUID     int64     `gorm:"not null;index"`
	Quantity  int64     `gorm:"not null"`
	Status    int       `gorm:"not null;default:0"` // 0=pending, 1=paid, 2=cancelled
	Type      int       `gorm:"not null;default:0"` // 0=normal, 1=flash-sale
	CreatedAt time.Time `gorm:"not null"`
}

const (
	OrderStatusPending = 0
	OrderStatusPaid    = 1
	OrderStatusCancelled = 2
	OrderTypeNormal    = 0
	OrderTypeFlashSale = 1
)
```

### 5. 完整的秒杀 Handler（go-zero logic 层）

```go
// flashsale/logic.go
package flashsale

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"encoding/json"
	"errors"
	"fmt"
	"time"

	"github.com/zeromicro/go-zero/core/logx"
	"github.com/zeromicro/go-zero/core/stores/redis"
	"gorm.io/gorm"
)

// FlashSaleLogic 秒杀业务逻辑（go-zero logic 层示例）
type FlashSaleLogic struct {
	rdb         *redis.Redis // go-zero Redis wrapper
	db          *gorm.DB
	allocator   *FlashSaleAllocator
	rateLimiter *RateLimiter
	producer    MessageProducer
	tenantID    string
}

func NewFlashSaleLogic(rdb *redis.Redis, db *gorm.DB, producer MessageProducer, tenantID string) *FlashSaleLogic {
	// Script loading is lazy — first Allocate call triggers NOSCRIPT fallback
	return &FlashSaleLogic{
		rdb:         rdb,
		db:          db,
		allocator:   NewFlashSaleAllocator(rdb),
		rateLimiter: NewRateLimiter(rdb, tenantID, DefaultRateLimitConfig()),
		producer:    producer,
		tenantID:    tenantID,
	}
}

// FlashSaleResult 秒杀返回结果
type FlashSaleResult struct {
	OrderID      int64
	AllocToken   string
	IdempotencyKey string
}

// HandleFlashSale 处理秒杀请求的完整流程
func (l *FlashSaleLogic) HandleFlashSale(ctx context.Context, skuID, userID int64, quantity int64) (*FlashSaleResult, error) {
	// Step 1: 全局 + 用户级限流
	allowed, err := l.rateLimiter.Allow(ctx, skuID, userID)
	if err != nil {
		return nil, fmt.Errorf("rate limit check failed: %w", err)
	}
	if !allowed {
		return nil, ErrUserLimited // 触发限流，直接拒绝
	}

	// Step 2: Redis Lua 预分配库存
	result, err := l.allocator.Allocate(ctx, l.tenantID, skuID, userID, quantity, 1)
	if err != nil {
		return nil, fmt.Errorf("stock allocate failed: %w", err)
	}
	if !result.Success {
		switch result.Code {
		case 1:
			return nil, ErrStockEmpty
		case 2:
			return nil, ErrPurchaseLimitExceeded
		case 3:
			return nil, ErrSaleNotStart
		case 4:
			return nil, ErrSaleEnded
		default:
			return nil, errors.New("unknown allocate error")
		}
	}

	// Step 3: 生成幂等键（使用 crypto/rand 保证高并发下不碰撞）
	idempotencyKey := generateIdempotencyKey(skuID, userID)
	msg := &FlashSaleOrderMessage{
		SKUID:          skuID,
		UserID:         userID,
		Quantity:       quantity,
		AllocToken:     result.Token,
		IdempotencyKey: idempotencyKey,
		Timestamp:      time.Now().UnixMilli(),
	}

	body, err := json.Marshal(msg)
	if err != nil {
		// 序列化失败 → 回滚 Redis 库存
		l.rollbackStock(ctx, l.tenantID, skuID, userID, quantity)
		return nil, fmt.Errorf("marshal message failed: %w", err)
	}

	if err := l.producer.Publish(ctx, FlashSaleOrderQueue(l.tenantID), body); err != nil {
		// MQ 发送失败 → 回滚 Redis 库存
		l.rollbackStock(ctx, l.tenantID, skuID, userID, quantity)
		return nil, fmt.Errorf("publish to mq failed: %w", err)
	}

	maskedUID := fmt.Sprintf("u***%04x", userID%0xFFFF)
	logx.WithContext(ctx).Infow("flash-sale allocated",
		logx.Field("skuID", skuID),
		logx.Field("userID", maskedUID),
		logx.Field("token", result.Token),
	)
	return &FlashSaleResult{
		OrderID:        0, // 异步下单，orderID 由消费者写入后轮询获取
		AllocToken:     result.Token,
		IdempotencyKey: idempotencyKey,
	}, nil
}

// rollbackStockLua 原子回滚 Redis 库存（Lua 脚本保证 sold+limit 同时扣减）
// KEYS[1]=sold, KEYS[2]=limit, ARGV[1]=quantity
const rollbackStockLua = `
redis.call('DECRBY', KEYS[1], ARGV[1])
redis.call('DECRBY', KEYS[2], ARGV[1])
return 0
`

// rollbackStock 回滚 Redis 库存（补偿操作）
func (l *FlashSaleLogic) rollbackStock(ctx context.Context, tenantID string, skuID, userID, quantity int64) {
	soldKey := fmt.Sprintf(SoldKeyPrefix, tenantID) + fmt.Sprintf("%d", skuID)
	limitKey := fmt.Sprintf(LimitKeyPrefix, tenantID) + fmt.Sprintf("%d:%d", skuID, userID)

	_, err := l.rdb.EvalCtx(ctx, rollbackStockLua, []string{soldKey, limitKey}, quantity).Int()
	if err != nil {
		logx.WithContext(ctx).Errorw("rollback stock failed",
			logx.Field("skuID", skuID),
			logx.Field("error", err),
		)
	}

	maskedUID := fmt.Sprintf("u***%04x", userID%0xFFFF)
	logx.WithContext(ctx).Infow("flash-sale rollback",
		logx.Field("skuID", skuID),
		logx.Field("userID", maskedUID),
		logx.Field("quantity", quantity),
	)
}

// generateIdempotencyKey 使用 crypto/rand 生成唯一幂等键，避免高并发下碰撞
func generateIdempotencyKey(skuID, userID int64) string {
	b := make([]byte, 16)
	if _, err := rand.Read(b); err != nil {
		// 极低概率失败，fallback 到 UnixNano
		return fmt.Sprintf("flash:%d:%d:%d", skuID, userID, time.Now().UnixNano())
	}
	return fmt.Sprintf("flash:%d:%d:%s", skuID, userID, hex.EncodeToString(b))
}
```

### 6. go-zero 降级策略（core/breaker）

```go
// flashsale/degradation.go
package flashsale

import (
	"context"
	"fmt"
	"strconv"

	"github.com/zeromicro/go-zero/core/breaker"
	"github.com/zeromicro/go-zero/core/logx"
	"github.com/zeromicro/go-zero/core/stores/redis"
)

// FlashSaleDegradation 基于 go-zero breaker 的降级策略
type FlashSaleDegradation struct {
	breaker  breaker.Breaker
	rdb      *redis.Redis
	tenantID string
}

func NewFlashSaleDegradation(rdb *redis.Redis, tenantID string) *FlashSaleDegradation {
	return &FlashSaleDegradation{
		breaker:  breaker.NewBreaker(),
		rdb:      rdb,
		tenantID: tenantID,
	}
}

// DoWithFallback 执行秒杀核心逻辑，失败时自动降级
func (d *FlashSaleDegradation) DoWithFallback(ctx context.Context, skuID int64, fn func() (interface{}, error), fallback func() (interface{}, error)) (interface{}, error) {
	return d.breaker.DoWithFallbackCtx(ctx, func() (interface{}, error) {
		result, err := fn()
		if err != nil {
			logx.WithContext(ctx).Errorw("core logic failed",
				logx.Field("skuID", skuID),
				logx.Field("error", err),
			)
		}
		return result, err
	}, func(err error) (interface{}, error) {
		// 熔断触发时的降级逻辑 — 根据 level 执行不同降级策略
		key := fmt.Sprintf(DegradationKeyPrefix, d.tenantID) + "level"
		levelStr, _ := d.rdb.GetCtx(ctx, key)
		level := int64(0)
		if levelStr != "" {
			level = parseInt64(levelStr)
		}
		logx.WithContext(ctx).Errorw("circuit breaker tripped",
			logx.Field("degradationLevel", level),
			logx.Field("skuID", skuID),
		)
		switch level {
		case 1:
			// L1: 返回友好提示
			return nil, fmt.Errorf("服务繁忙，请稍后重试")
		case 2:
			// L2: 返回静态排队页面
			return nil, fmt.Errorf("系统繁忙，已进入排队模式")
		case 3:
			// L3: 直接阻断所有请求
			return nil, fmt.Errorf("秒杀活动暂停中")
		default:
			if fallback != nil {
				return fallback()
			}
			return nil, fmt.Errorf("服务繁忙，请稍后重试")
		}
	})
}

// GetLevel 获取当前降级等级（从 Redis 读取，多实例共享）
func (d *FlashSaleDegradation) GetLevel(ctx context.Context) int64 {
	key := fmt.Sprintf(DegradationKeyPrefix, d.tenantID) + "level"
	levelStr, err := d.rdb.GetCtx(ctx, key)
	if err != nil || levelStr == "" {
		return 0 // 默认正常
	}
	return parseInt64(levelStr)
}

// SetLevel 设置降级等级（手动控制恢复）
func (d *FlashSaleDegradation) SetLevel(ctx context.Context, level int64) error {
	key := fmt.Sprintf(DegradationKeyPrefix, d.tenantID) + "level"
	return d.rdb.SetExCtx(ctx, key, level, int64(300))
}

// Reset 手动重置熔断器（故障恢复后调用）
func (d *FlashSaleDegradation) Reset() {
	d.breaker = breaker.NewBreaker()
}

// parseInt64 helper — go-zero returns string from GetCtx
func parseInt64(s string) int64 {
	n, err := strconv.ParseInt(s, 10, 64)
	if err != nil {
		return 0
	}
	return n
}
```

## 反模式（Anti-patterns）

| 反模式 | 说明 | 后果 |
|--------|------|------|
| 秒杀直接写 DB | 不经过 Redis 预分配，直接 `UPDATE stock = stock - 1` | PG 连接池被打满，数据库崩溃，超卖 |
| 无限流 | 秒杀接口不配置 rate limit | 恶意刷接口/脚本抢光库存，正常用户无法访问 |
| 无队列异步处理 | 同步创建订单，等 PG 写入完成再返回 | 接口 RT > 5s，大量请求堆积，服务雪崩 |
| Redis 和 DB 不一致无补偿 | Redis 扣了库存但 MQ 发送失败 | 用户看到扣库存成功但订单未创建，客诉 |
| 不限购 | 单用户可以无限购买 | 黄牛脚本秒光库存 |
| 无降级策略 | 故障时继续硬扛 | 整个服务级联崩溃 |
| 秒杀开关不可控 | 无法紧急停止秒杀 | 系统崩溃时无法及时止损 |
| 预热数据不一致 | Redis 库存 ≠ DB 库存 | 超卖或少卖 |

## 常见坑（Common pitfalls + solutions）

### 坑 1：Redis Lua 脚本加载后重启丢失

**现象**：Redis 重启或主从切换后，`EVALSHA` 返回 `NOSCRIPT` 错误。

**解决**：
- 服务启动时预加载 `ScriptLoad`
- 捕获 `NOSCRIPT` 错误时自动 `EVAL` 全量脚本
- 使用 `go-redis` 的 `EvalSha` + fallback `Eval` 模式（如代码模板所示）

### 坑 2：MQ 消息丢失导致少卖

**现象**：Redis 扣了库存，但 MQ 消息丢失，订单没有创建。

**解决**：
- MQ 使用持久化队列（RabbitMQ `delivery_mode=2` / Kafka `acks=all`）
- MQ publish 失败时回滚 Redis 库存（rollbackStock）
- 定时任务扫描 Redis `flash:alloc:*` 中超过 5 分钟的 token，未消费的自动回滚

### 坑 3：超卖 — Redis 和 PG 库存不一致

**现象**：Redis 说还有库存，但 PG 已无库存。

**解决**：
- PG 库存扣减必须加乐观锁（`WHERE stock >= ? AND version = ?`）
- Redis 库存是"预分配"，不是"最终扣减"，真正扣减在 PG 事务中完成
- 定时任务对账：比较 Redis sold 和 PG order count，不一致则告警

### 坑 4：限流 key 未设置 TTL 导致内存泄漏

**现象**：`flash:ratelimit:*` key 持续累积，Redis 内存不断增长。

**解决**：
- 滑动窗口限流脚本中必须 `EXPIRE`（代码模板已包含）
- 配置 Redis `maxmemory-policy = allkeys-lru`

### 坑 5：秒杀结束后 Redis key 未清理

**现象**：大量过期 key 占用内存。

**解决**：
- 预热时设置合理的 TTL（秒杀持续时间 + 30 分钟缓冲）
- 秒杀结束后主动 `DEL` 清理（或依赖 Redis 惰性删除）
- 使用 `SCAN` 批量清理：`SCAN 0 MATCH flash:* COUNT 1000`

### 坑 6：高并发下 log 打满磁盘

**现象**：每秒 10 万条日志，磁盘 I/O 成为瓶颈。

**解决**：
- 秒杀期间降低日志级别（仅记录 error）
- 采样日志：每 1000 条记录 1 条 success
- 异步日志写入（使用 zap 等高性能日志库）

### 坑 7：时钟漂移导致秒杀提前/延后

**现象**：多实例时间不一致，有的已经开抢有的还没开始。

**解决**：
- 秒杀时间以 Redis 时间为准：`redis.call('TIME')`
- 所有实例使用 NTP 同步时钟
- 状态存储在 Redis（`flash:{tenantID}:status:{skuID}`），不依赖本地时间

## 测试策略（Test strategy）

### L1 — 单元测试

| 组件 | 工具 | 覆盖范围 |
|------|------|----------|
| LuaAllocScript | `alicebob/miniredis/v2` | 正常分配、库存不足、限购、状态检查、token 返回格式、alloc_seq 递增 |
| AllocateResult 解析 | 标准 testing | token 返回成功/错误码分支覆盖率 |
| generateIdempotencyKey | 标准 testing | 碰撞检测（10000 次无重复）、rand 失败 fallback 路径 |
| parseInt64 | 标准 testing | 有效/无效输入 |

```go
import (
	"testing"
	"github.com/stretchr/testify/assert"
	"github.com/zeromicro/go-zero/core/stores/redis"
	"github.com/alicebob/miniredis/v2"
)

func TestLuaAllocScript_NormalAlloc(t *testing.T) {
	s := miniredis.RunT(t)
	c := redis.New(s.Addr())
	defer c.Close()

	// Setup
	c.Set("flash:test:stock:1001", "10")
	c.Set("flash:test:sold:1001", "0")
	c.Set("flash:test:limit:1001:42", "0")
	c.Set("flash:test:status:1001", "1")

	result, err := c.Eval(LuaAllocScript, []string{
		"flash:test:stock:1001", "flash:test:sold:1001",
		"flash:test:limit:1001:42", "flash:test:status:1001",
		"flash:test:alloc:1001",
	}, "1001", "42", "1", "3").Text()

	assert.NoError(t, err)
	assert.Contains(t, result, "1001:42:") // token starts with sku:userID
}
```

### L2 — 集成测试

| 场景 | 说明 |
|------|------|
| Prewarm → Allocate → Consume 完整链路 | 使用 miniredis + mock DB + mock Producer，验证库存扣减→token 生成→MQ 消息→订单创建全链路 |
| 幂等性 | 发送两条相同 idempotencyKey 消息，第二条应直接返回（setnx 幂等） |
| 重试→死信 | 模拟 DB 事务失败 3 次，验证消息进入 DLQ |
| MQ publish 失败回滚 | 模拟 producer.Publish 返回 error，验证 rollbackStock 被调用、sold/limit 回滚 |
| NOSCRIPT 自动恢复 | 先 LoadScript 后 Redis FlushAll，验证 Allocate 自动重新加载脚本 |

### L3 — 压力测试

| 组件 | 目标 | 说明 |
|------|------|------|
| Allocator | >= 50K QPS | 单 SKU 并发 10000 goroutine 抢购 100 库存，验证无超卖（sold <= stock） |
| 全链路 | >= 10K QPS | Prewarm → 并发 Allocate → 异步 Consume，验证 Redis/DB 一致性 |
| 泄漏检测 | 0 key 泄漏 | 压测后 `DBSIZE` 无异常增长，TTL key 自动过期 |

### 覆盖率目标

- **L1 >= 80%** 行覆盖率（alloc.go, logic.go, consumer.go）
- **Lua 脚本** 100% 分支覆盖（5 条返回路径全部测试）
- 关键路径：幂等检查、重试逻辑、回滚补偿、限流拒绝 必须 100% 覆盖
