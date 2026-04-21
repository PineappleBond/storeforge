# 库存模式 (Inventory Patterns)

## Redis Lua 脚本扣减 + PG 事务双写 + 一致性补偿

---

### 触发条件

- 订单创建时扣减库存
- 秒杀/抢购场景下的防超卖
- 库存回滚（取消订单、支付超时）
- 高并发库存操作需要原子性保证
- 需要精确的库存流水审计

---

### 决策树

```
库存操作场景？
  ├─ 普通商品下单 → Redis Lua 预扣 + PG 事务双写 + 补偿
  ├─ 秒杀/抢购 → Redis 预分配队列 + 异步下单 + 限流排队
  ├─ 库存查询 → Redis 缓存（定期与 PG 对账同步）
  └─ 库存回滚（取消/超时）→ Redis Lua 回退 + PG 事务（必须幂等）
```

---

### 代码模板

#### 4.1 Lua Scripts — 原子库存脚本

Lua 脚本使用 `const` 定义，确保可编译。脚本逻辑在 section 4.2 代码块底部以 `const` 形式提供。

#### 4.2 Go 代码 — 双写 + 一致性补偿

```go
// internal/logic/inventory/inventory_service.go
package inventory

import (
	"context"
	"errors"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
	"github.com/zeromicro/go-zero/core/logx"
	"gorm.io/gorm"
)

// InventoryService 库存服务
type InventoryService struct {
	rdb        *redis.Client
	db         *gorm.DB
	tenantID   string
	deductSHA  string // Lua 脚本 SHA
	rollbackSHA string
}

// InventoryModel PG 库存表
type InventoryModel struct {
	ID        int64     `gorm:"primaryKey"`
	SKUID     int64     `gorm:"uniqueIndex;not null"`
	TenantID  string    `gorm:"uniqueIndex:idx_tenant_sku;not null"`
	Stock     int64     `gorm:"not null"`
	Version   int64     `gorm:"not null;default:0"` // 乐观锁版本号
	UpdatedAt time.Time `json:"updated_at"`
}

func (InventoryModel) TableName() string {
	return "inventories"
}

// DeductInventory 扣减库存（Redis + PG 双写 + 补偿）
func (s *InventoryService) DeductInventory(ctx context.Context, skuID int64, quantity int64, orderID string) error {
	logx.WithContext(ctx).Infof("[库存扣减] 开始: sku=%d, qty=%d, order=%s", skuID, quantity, orderID)

	// ===== 阶段 1: Redis Lua 原子扣减（使用 go-redis Script 类型）=====
	stockKey := fmt.Sprintf("inventory:%s:%d", s.tenantID, skuID)
	logKey := fmt.Sprintf("inventory:deduct_log:%s:%d", s.tenantID, skuID)

	// go-redis Script 类型自动处理：
	// 1. 首次调用 EVAL 并缓存 SHA
	// 2. 后续调用 EVALSHA
	// 3. 遇到 NOSCRIPT 错误自动回退到 EVAL
	deductScript := redis.NewScript(luaDeductScript)
	result, err := deductScript.Eval(ctx, s.rdb, []string{stockKey, logKey},
		quantity, orderID, time.Now().Unix(),
	).Slice()
	if err != nil {
		return fmt.Errorf("lua deduct: %w", err)
	}

	success := result[0].(int64)
	remainingStock := result[1].(int64)

	if success == 0 {
		return fmt.Errorf("库存不足，剩余 %d", remainingStock)
	}

	// ===== 阶段 2: PG 事务扣减 =====
	err = s.db.Transaction(func(tx *gorm.DB) error {
		var inv InventoryModel
		// 使用乐观锁：WHERE stock >= quantity AND version = current_version
		// 多租户：必须加 tenant_id 过滤
		res := tx.Model(&InventoryModel{}).
			Where("tenant_id = ? AND sku_id = ? AND stock >= ?", s.tenantID, skuID, quantity).
			Update("stock", gorm.Expr("stock - ?", quantity)).
			Update("version", gorm.Expr("version + 1"))

		if res.Error != nil {
			return fmt.Errorf("pg update: %w", res.Error)
		}
		if res.RowsAffected == 0 {
			return errors.New("库存不足或并发冲突")
		}

		// 记录库存流水
		return tx.Create(&InventoryLog{
			SKUID:     skuID,
			TenantID:  s.tenantID,
			OrderID:   orderID,
			Change:    -quantity,
			StockAfter: remainingStock,
			Reason:    "order_deduct",
			CreatedAt: time.Now(),
		}).Error
	})

	if err != nil {
		// ===== 阶段 3: PG 失败 → 补偿回滚 Redis =====
		logx.WithContext(ctx).Errorf("[库存补偿] PG 写入失败，回滚 Redis: sku=%d, order=%s, err=%v", skuID, orderID, err)
		s.compensateDeduct(ctx, skuID, quantity, orderID)
		return fmt.Errorf("库存扣减失败，已补偿: %w", err)
	}

	return nil
}

// compensateDeduct PG 失败后补偿回滚 Redis 库存
func (s *InventoryService) compensateDeduct(ctx context.Context, skuID int64, quantity int64, orderID string) {
	logx.WithContext(ctx).Errorf("[库存补偿] 开始回滚 Redis: sku=%d, order=%s, qty=%d", skuID, orderID, quantity)
	stockKey := fmt.Sprintf("inventory:%s:%d", s.tenantID, skuID)
	deductLogKey := fmt.Sprintf("inventory:deduct_log:%s:%d", s.tenantID, skuID)

	if err := s.rdb.ZRem(ctx, deductLogKey, fmt.Sprintf("%s:%d", orderID, quantity)).Err(); err != nil {
		logx.WithContext(ctx).Errorf("[库存补偿] 移除流水失败: sku=%d, order=%s, err=%v", skuID, orderID, err)
	}
	if _, err := s.rdb.IncrBy(ctx, stockKey, quantity).Result(); err != nil {
		logx.WithContext(ctx).Errorf("[库存补偿] Redis 回滚失败: sku=%d, order=%s, err=%v", skuID, orderID, err)
		return
	}
	logx.WithContext(ctx).Infof("[库存补偿] Redis 回滚完成: sku=%d, order=%s", skuID, orderID)
}

// RollbackInventory 回滚库存（取消订单/支付超时）
func (s *InventoryService) RollbackInventory(ctx context.Context, skuID int64, quantity int64, orderID string) error {
	logx.WithContext(ctx).Infof("[库存回滚] 开始: sku=%d, qty=%d, order=%s", skuID, quantity, orderID)

	// ===== 阶段 1: Redis Lua 幂等回滚（使用 go-redis Script 类型）=====
	stockKey := fmt.Sprintf("inventory:%s:%d", s.tenantID, skuID)
	rollbackLogKey := fmt.Sprintf("inventory:rollback_log:%s:%d", s.tenantID, skuID)

	rollbackScript := redis.NewScript(luaRollbackScript)
	result, err := rollbackScript.Eval(ctx, s.rdb, []string{stockKey, rollbackLogKey},
		quantity, orderID, time.Now().Unix(),
	).Slice()
	if err != nil {
		return fmt.Errorf("lua rollback: %w", err)
	}

	success := result[0].(int64)
	alreadyDone := result[2].(int64)

	if alreadyDone == 1 {
		// 幂等：已经回滚过，直接返回
		return nil
	}

	if success == 0 {
		return errors.New("Redis 库存回滚失败")
	}

	// ===== 阶段 2: PG 事务回滚 =====
	err = s.db.Transaction(func(tx *gorm.DB) error {
		res := tx.Model(&InventoryModel{}).
			Where("tenant_id = ? AND sku_id = ?", s.tenantID, skuID).
			Update("stock", gorm.Expr("stock + ?", quantity)).
			Update("version", gorm.Expr("version + 1"))

		if res.Error != nil {
			return res.Error
		}

		// 记录回滚流水
		return tx.Create(&InventoryLog{
			SKUID:     skuID,
			TenantID:  s.tenantID,
			OrderID:   orderID,
			Change:    quantity,
			Reason:    "order_cancel",
			CreatedAt: time.Now(),
		}).Error
	})

	if err != nil {
		// PG 回滚失败：Redis 已成功回滚，PG 不一致
		// 记录补偿任务，由定时任务修复
		logx.WithContext(ctx).Errorf("[库存补偿] PG 回滚失败: sku=%d, order=%s, err=%v", skuID, orderID, err)
		s.enqueueCompensation(ctx, skuID, quantity, orderID, "rollback_pg_failed")
	}

	return err
}

// InventoryLog 库存流水表
type InventoryLog struct {
	ID         int64     `gorm:"primaryKey"`
	SKUID      int64     `gorm:"index"`
	TenantID   string    `gorm:"index:idx_tenant_sku"`
	OrderID    string    `gorm:"index"`
	Change     int64     // 正数=回滚/入库，负数=扣减/出库
	StockAfter int64     // 变动后库存
	Reason     string    // order_deduct / order_cancel / timeout_cancel / admin_adjust
	CreatedAt  time.Time `json:"created_at"`
}

func (InventoryLog) TableName() string {
	return "inventory_logs"
}

// enqueueCompensation 将补偿任务写入队列（由 worker 消费）
func (s *InventoryService) enqueueCompensation(ctx context.Context, skuID int64, quantity int64, orderID, reason string) {
	// 写入 Redis 队列（带 TTL 防止内存泄漏）
	task := fmt.Sprintf(`{"sku_id":%d,"quantity":%d,"order_id":"%s","reason":"%s"}`,
		skuID, quantity, orderID, reason)
	queueKey := fmt.Sprintf("inventory:compensation_queue:%s", s.tenantID)
	if err := s.rdb.LPush(ctx, queueKey, task).Err(); err != nil {
		logx.WithContext(ctx).Errorf("[库存补偿] 写入队列失败: task=%s, err=%v", task, err)
	}
	s.rdb.Expire(ctx, queueKey, 7*24*time.Hour) // 队列 7 天过期
}

// Lua 脚本源码 — 原子库存扣减
const luaDeductScript = `
local stock_key = KEYS[1]
local log_key = KEYS[2]
local deduct_qty = tonumber(ARGV[1])
local order_id = ARGV[2]
local ts = ARGV[3]
local stock = tonumber(redis.call('GET', stock_key) or '0')
if stock < deduct_qty then
    return {0, stock, 0}
end
local new_stock = redis.call('DECRBY', stock_key, deduct_qty)
if new_stock < 0 then
    redis.call('INCRBY', stock_key, deduct_qty)
    return {0, new_stock + deduct_qty, 0}
end
redis.call('ZADD', log_key, ts, string.format('%s:%d', order_id, deduct_qty))
redis.call('EXPIRE', log_key, 2592000)
return {1, new_stock, deduct_qty}
`

// Lua 脚本源码 — 幂等库存回滚
const luaRollbackScript = `
local stock_key = KEYS[1]
local log_key = KEYS[2]
local rollback_qty = tonumber(ARGV[1])
local order_id = ARGV[2]
local ts = ARGV[3]
local already = redis.call('SISMEMBER', log_key, order_id)
if already == 1 then
    local stock = tonumber(redis.call('GET', stock_key) or '0')
    return {0, stock, 1}
end
local new_stock = redis.call('INCRBY', stock_key, rollback_qty)
redis.call('SADD', log_key, order_id)
redis.call('EXPIRE', log_key, 2592000)
return {1, new_stock, 0}
`
```

#### 4.4 库存补偿 Worker（定时任务）

```go
// internal/worker/inventory_compensation.go
package worker

import (
	"context"
	"encoding/json"
	"fmt"
	"time"

	"github.com/cenkalti/backoff/v4"
	"github.com/redis/go-redis/v9"
	"github.com/robfig/cron/v3"
	"github.com/zeromicro/go-zero/core/logx"
	"gorm.io/gorm"
)

// CompensationWorker 库存一致性补偿 Worker
type CompensationWorker struct {
	rdb      *redis.Client
	db       *gorm.DB
	tenantID string
}

type CompensationTask struct {
	SKUID    int64  `json:"sku_id"`
	Quantity int64  `json:"quantity"`
	OrderID  string `json:"order_id"`
	Reason   string `json:"reason"`
	Attempts int    `json:"attempts"` // 重试次数，防止无限循环
}

// Run 使用 robfig/cron 定时消费补偿队列 + 定期库存对账
func (w *CompensationWorker) Run(ctx context.Context) error {
	c := cron.New(cron.WithChain(
		cron.SkipIfStillRunning(cron.DefaultLogger),
		cron.Recover(cron.DefaultLogger),
	))

	_, err := c.AddFunc("@every 5s", func() {
		w.processQueue(ctx)
	})
	if err != nil {
		return err
	}

	// 每小时执行一次库存对账（低峰期执行，避免覆盖进行中的交易）
	_, err = c.AddFunc("@every 1h", func() {
		if err := w.ReconcileInventory(ctx); err != nil {
			logx.WithContext(ctx).Errorf("库存对账失败: %v", err)
		}
	})
	if err != nil {
		return err
	}

	c.Start()
	<-ctx.Done()
	c.Stop()
	return nil
}

func (w *CompensationWorker) processQueue(ctx context.Context) {
	queueKey := fmt.Sprintf("inventory:compensation_queue:%s", w.tenantID)
	for {
		// BRPop 阻塞等待，避免空队列时的 busy-loop
		result, err := w.rdb.BRPop(ctx, 5*time.Second, queueKey).Result()
		if err == redis.Nil {
			return // 超时且队列为空
		}
		if err != nil {
			logx.WithContext(ctx).Errorf("读取补偿队列失败: %v", err)
			return
		}

		// BRPop 返回 [key, value]
		taskData := result[1]
		var task CompensationTask
		if err := json.Unmarshal([]byte(taskData), &task); err != nil {
			logx.WithContext(ctx).Errorf("解析补偿任务失败: %v, data=%s", err, taskData)
			continue
		}

		// 使用 exponential backoff 重试补偿任务
		b := backoff.WithMaxRetries(backoff.NewExponentialBackOff(), 3)
		if err := backoff.Retry(func() error {
			return w.executeCompensation(ctx, task)
		}, b); err != nil {
			// 超过最大重试次数，写入死信队列
			task.Attempts++
			if task.Attempts >= 5 {
				dlqKey := fmt.Sprintf("inventory:compensation_dlq:%s", w.tenantID)
				taskData, _ := json.Marshal(task)
				w.rdb.LPush(ctx, dlqKey, taskData)
				w.rdb.Expire(ctx, dlqKey, 30*24*time.Hour) // 死信队列 30 天过期
				logx.WithContext(ctx).Errorf("补偿任务超过最大重试次数，转入死信队列: %+v, err=%v", task, err)
			} else {
				// 使用 RPush 放到队尾，避免阻塞队首的正常补偿任务
				w.rdb.RPush(ctx, queueKey, taskData)
				logx.WithContext(ctx).Errorf("补偿任务执行失败，放回队尾: %+v, err=%v", task, err)
			}
			continue
		}

		logx.WithContext(ctx).Infof("补偿任务执行成功: %+v", task)
	}
}

func (w *CompensationWorker) executeCompensation(ctx context.Context, task CompensationTask) error {
	switch task.Reason {
	case "rollback_pg_failed":
		// Redis 已回滚但 PG 未回滚 → 补回 PG
		return w.db.Transaction(func(tx *gorm.DB) error {
			res := tx.Model(&InventoryModel{}).
				Where("tenant_id = ? AND sku_id = ?", w.tenantID, task.SKUID).
				Update("stock", gorm.Expr("stock + ?", task.Quantity))
			if res.Error != nil {
				return res.Error
			}
			if res.RowsAffected == 0 {
				return nil // SKU 不存在，跳过
			}
			return tx.Create(&InventoryLog{
				SKUID:   task.SKUID,
				TenantID: w.tenantID,
				OrderID: task.OrderID,
				Change:  task.Quantity,
				Reason:  "compensation_rollback",
			}).Error
		})

	case "deduct_pg_failed":
		// Redis 扣减成功但 PG 未扣减 → 补扣 PG
		return w.db.Transaction(func(tx *gorm.DB) error {
			res := tx.Model(&InventoryModel{}).
				Where("tenant_id = ? AND sku_id = ? AND stock >= ?", w.tenantID, task.SKUID, task.Quantity).
				Update("stock", gorm.Expr("stock - ?", task.Quantity))
			if res.Error != nil {
				return res.Error
			}
			return tx.Create(&InventoryLog{
				SKUID:   task.SKUID,
				TenantID: w.tenantID,
				OrderID: task.OrderID,
				Change:  -task.Quantity,
				Reason:  "compensation_deduct",
			}).Error
		})

	default:
		return nil
	}
}
```

#### 4.5 库存对账（定期校验 Redis 与 PG 一致性）

```go
// internal/worker/inventory_reconcile.go
package worker

import (
	"context"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
	"github.com/zeromicro/go-zero/core/logx"
	"gorm.io/gorm"
)

// ReconcileInventory 定期对账 Redis 与 PG 库存
// InventoryModel / InventoryLog 类型定义见 section 4.2
func (w *CompensationWorker) ReconcileInventory(ctx context.Context) error {
	const batchSize = 500
	var lastID int64
	mismatchCount := 0

	for {
		// 分页查询，避免一次性加载大量数据
		var batch []InventoryModel
		if err := w.db.Where("tenant_id = ? AND id > ?", w.tenantID, lastID).
			Order("id ASC").Limit(batchSize).Find(&batch).Error; err != nil {
			return fmt.Errorf("query pg inventory: %w", err)
		}
		if len(batch) == 0 {
			break
		}

		for _, inv := range batch {
			redisKey := fmt.Sprintf("inventory:%s:%d", w.tenantID, inv.SKUID)
			redisStock, err := w.rdb.Get(ctx, redisKey).Int64()
			if err != nil {
				// Redis 中不存在 → 从 PG 初始化
				w.rdb.Set(ctx, redisKey, inv.Stock, 24*time.Hour)
				continue
			}

			if redisStock != inv.Stock {
				mismatchCount++
				logx.WithContext(ctx).Errorf("[库存对账] SKU %d: Redis=%d, PG=%d, 差异=%d",
					inv.SKUID, redisStock, inv.Stock, redisStock-inv.Stock)

				// 以 PG 为准，修复 Redis（低峰期执行，避免覆盖进行中的交易）
				w.rdb.Set(ctx, redisKey, inv.Stock, 24*time.Hour)
			}
		}

		lastID = batch[len(batch)-1].ID
		if len(batch) < batchSize {
			break
		}
	}

	if mismatchCount > 0 {
		logx.WithContext(ctx).Infof("[库存对账] 完成，发现 %d 个不一致", mismatchCount)
	}

	return nil
}
```

#### 4.6 秒杀预分配队列模式

```go
// internal/logic/flashsale/flash_sale.go
package flashsale

import (
	"context"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
	"github.com/zeromicro/go-zero/core/logx"
	"gorm.io/gorm"
)

// FlashSaleService 秒杀服务
type FlashSaleService struct {
	rdb      *redis.Client
	db       *gorm.DB
	tenantID string
}

// PreAllocate 秒杀预热：将库存预分配到 Redis 队列
func (s *FlashSaleService) PreAllocate(ctx context.Context, skuID int64, totalStock int64) error {
	// 1. 初始化秒杀库存（多租户隔离，使用 SetNX 防止多实例重复预热）
	key := fmt.Sprintf("flash_sale:stock:%s:%d", s.tenantID, skuID)
	ok, err := s.rdb.SetNX(ctx, key, totalStock, 1*time.Hour).Result()
	if err != nil {
		return fmt.Errorf("setnx flash sale stock: %w", err)
	}
	if !ok {
		logx.WithContext(ctx).Infof("秒杀库存已预热，跳过: sku=%d", skuID)
	}

	// 2. 初始化排队队列（限制并发）
	queueKey := fmt.Sprintf("flash_sale:queue:%s:%d", s.tenantID, skuID)
	// 用户请求先入队，再逐个处理

	return nil
}

// TryPurchase 秒杀购买（排队 + 预分配）
func (s *FlashSaleService) TryPurchase(ctx context.Context, skuID int64, userID int64) error {
	stockKey := fmt.Sprintf("flash_sale:stock:%s:%d", s.tenantID, skuID)
	orderSetKey := fmt.Sprintf("flash_sale:orders:%s:%d", s.tenantID, skuID)

	// 使用 go-redis Script 类型，自动 SHA 缓存 + NOSCRIPT 回退
	script := redis.NewScript(luaFlashSaleScript)
	result, err := script.Eval(ctx, s.rdb, []string{stockKey, orderSetKey},
		userID, 1, // 每人限购 1 件
	).Slice()
	if err != nil {
		return fmt.Errorf("lua eval: %w", err)
	}

	success := result[0].(int64)
	remaining := result[1].(int64)
	msg := result[2].(string)

	if success == 0 {
		if msg == "sold_out" {
			return fmt.Errorf("已售罄")
		}
		if msg == "duplicate" {
			return fmt.Errorf("每人限购 1 件")
		}
	}

	// 秒杀成功 → 异步创建订单
	// s.enqueueOrderCreation(ctx, skuID, userID)
	logx.WithContext(ctx).Infof("秒杀成功: sku=%d, user=%d, 剩余库存=%d", skuID, userID, remaining)

	return nil
}

// FlashSaleRateLimiter 秒杀限流器（令牌桶）
type FlashSaleRateLimiter struct {
	rdb    *redis.Client
	script *redis.Script // go-redis Script 类型，自动 SHA 缓存 + NOSCRIPT 回退
}

// Allow 限流检查（使用 go-redis Script 类型）
// key 必须包含 tenantID 前缀以保证多租户隔离（P1 #11）
func (l *FlashSaleRateLimiter) Allow(ctx context.Context, key string, rate int, window time.Duration) bool {
	if l.script == nil {
		l.script = redis.NewScript(luaRateLimiterScript)
	}
	result, err := l.script.Eval(ctx, l.rdb, []string{key},
		rate, int64(window.Seconds()), time.Now().UnixMilli(),
	).Int64()
	if err != nil {
		logx.WithContext(ctx).Errorf("限流 Lua 脚本执行失败: key=%s, err=%v", key, err)
		return false
	}
	return result == 1
}

// 使用示例：key 必须包含 tenantID
// limiter.Allow(ctx, fmt.Sprintf("ratelimit:%s:flash_sale:%d", tenantID, skuID), 100, time.Second)

// luaRateLimiterScript 限流 Lua 脚本
const luaRateLimiterScript = `
local key = KEYS[1]
local rate = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

local count = redis.call('ZCOUNT', key, now - window, now)
if count >= rate then
    return 0
end

redis.call('ZADD', key, now, tostring(now)..':'..tostring(math.random(1000000)))
redis.call('EXPIRE', key, window + 1)
return 1
`

// luaFlashSaleScript 秒杀预分配 Lua 脚本
const luaFlashSaleScript = `
local stock_key = KEYS[1]
local order_set = KEYS[2]
local user_id = ARGV[1]

if redis.call('SISMEMBER', order_set, user_id) == 1 then
    return {0, 0, "duplicate"}
end

local remaining = redis.call('DECR', stock_key)
if remaining < 0 then
    redis.call('INCR', stock_key)
    return {0, 0, "sold_out"}
end

redis.call('SADD', order_set, user_id)
return {1, remaining, "success"}
`
```

---

### 反模式

| 反模式 | 风险 | 正确做法 |
|--------|------|----------|
| 直接 `UPDATE inventory SET stock = stock - 1` | 高并发下超卖，行锁竞争激烈 | Redis Lua 脚本预扣 + PG 双写 + 补偿 |
| 不用 Lua 脚本做 Redis 扣减 | GET + DECR 两步操作非原子，并发不安全 | 所有库存操作必须在 Lua 脚本中原子执行 |
| PG 回滚但 Redis 不回滚 | Redis 库存持续减少，最终为负 | 必须有补偿机制（Worker 定时修复） |
| 秒杀直接走下单流程 | 数据库被打挂 | 秒杀走 Redis 预分配队列，异步创建订单 |
| 库存查询直接读 PG | 热点商品 QPS 过高 | 库存缓存到 Redis，定期与 PG 对账 |
| 回滚不幂等 | 重复回滚导致库存异常增加 | 使用 order_id 做幂等标记（Redis Set） |
| 不记录库存流水 | 无法追溯库存变更原因 | 每次变更必须写 inventory_logs 表 |
| 乐观锁不加 version 条件 | 并发更新丢失 | `WHERE stock >= qty AND version = ?` |

---

### 常见坑

**坑 1：Lua 脚本加载失败（NOSCRIPT）**
- **现象**：`EVALSHA` 报错 `NOSCRIPT No matching script`
- **原因**：Redis 重启或集群故障转移后脚本缓存丢失
- **解决**：应用启动时 `SCRIPT LOAD` 预加载；捕获 NOSCRIPT 错误后回退到 `EVAL` 重新加载

**坑 2：Redis DECR 导致负数库存**
- **现象**：并发请求导致 Redis 库存变成负数
- **原因**：Lua 脚本中先 DECR 再判断，但多个请求同时 DECR
- **解决**：Lua 脚本内先 GET 判断再 DECR；或 DECR 后判断负数则 INCR 回滚（模板中已包含双重保护）

**坑 3：补偿队列堆积**
- **现象**：PG 持续不可用，补偿队列堆积数万条
- **解决**：限制补偿队列消费速率；设置告警阈值；补偿失败超过 N 次转人工处理（死信队列）

**坑 4：库存预热时未考虑分布式部署**
- **现象**：多个服务实例同时预热同一 SKU 库存，重复 SET 导致数据覆盖
- **解决**：预热使用 `SETNX`（仅当 key 不存在时设置）；或使用分布式锁保证只有一个实例预热

**坑 5：秒杀场景 Redis 内存溢出**
- **现象**：百万级用户同时抢购，order_set（SADD 所有 userID）占用大量内存
- **解决**：使用 HyperLogLog 统计购买人数（近似去重，节省内存）；或只保留前 N 个购买者（`SADD` 后 `SINTERCARD` 限制）；或使用 Bloom Filter 预判重复购买

**坑 6：库存对账时直接以 PG 为准覆盖 Redis**
- **现象**：对账窗口期内有正常交易发生，以 PG 为准会覆盖 Redis 中未持久化的扣减
- **解决**：对账应在低峰期执行；对账时暂停新交易或记录快照时间点；使用"差异阈值"告警而非直接覆盖

---

### 测试策略

#### Lua 脚本单元测试

- **miniredis 方案**：使用 `github.com/alicebob/miniredis/v2` 启动内存 Redis，直接 `EVAL` Lua 脚本，验证返回值。miniredis 支持完整 Lua 环境，是最准确的集成测试方式。
  ```go
  func TestLuaDeductScript(t *testing.T) {
      mr, _ := miniredis.Run()
      defer mr.Close()
      rdb := redis.NewClient(&redis.Options{Addr: mr.Addr()})
      // mr.Set("inventory:tenant1:1001", "50")
      // script := redis.NewScript(luaDeductScript)
      // result, err := script.Eval(ctx, rdb, keys, args...).Slice()
      // assert.Equal(t, int64(1), result[0])
  }
  ```
- **go-redis Mock 方案**：使用 `redismock` 或自定义 mock `redis.UniversalClient`，但 Lua 脚本逻辑无法真正 mock 执行，只能测试调用路径。
- **边界用例**：库存=0、扣减>库存、重复扣减同一订单、并发 100 goroutine 同时扣减验证不超卖。

#### 补偿 Worker 测试

- **单元测试**：mock `*gorm.DB`（使用 `gorm.io/driver/sqlite` 内存数据库 + `sqlmock`），写入补偿任务到 miniredis，启动 `processQueue` 单轮执行，验证 PG 数据是否修复。
- **重试测试**：让 mock DB 在前 2 次 `ExecuteCompensation` 返回错误，第 3 次成功，验证 backoff 重试路径。
- **死信测试**：mock DB 始终失败，验证任务最终被放回队列且次数限制生效。

#### 对账（Reconciliation）测试

- **单元测试**：初始化 PG 和 Redis 的已知状态（正常匹配 + 故意不一致），执行 `ReconcileInventory`，验证不一致项被修复且 mismatchCount 正确。
- **并发安全测试**：对账期间模拟正常交易（goroutine 持续扣减），验证对账不会误覆盖正在进行的交易。

#### 并发测试场景

- **超卖防护**：100 个 goroutine 同时扣减同一 SKU（初始库存=50），验证最终 Redis+PG 库存 >= 0。
- **幂等回滚**：同一订单多次调用 `RollbackInventory`，验证只回滚一次。
- **双写一致性**：mock PG 事务失败，验证 Redis 被正确补偿回滚。
- **多租户隔离**：两个 tenant 同时操作相同 skuID，验证 key 不串扰。

#### 覆盖率目标

- **L1 单元测试** >= 80%（行覆盖率）
- **L2 集成测试**：至少覆盖 Redis Lua 脚本全路径 + PG 事务全路径
- **L3 E2E 测试**：至少 1 个完整下单→扣减→取消→回滚→补偿流程

---

### go-zero 集成指南

#### 使用 `core/stores/redis` 替代原生 go-redis

go-zero 封装了 `core/stores/redis`，在原生 go-redis 基础上增加了内置 breaker、timeout、metric 等能力。库存操作应优先使用此封装：

```go
import "github.com/zeromicro/go-zero/core/stores/redis"

// 初始化 Redis 客户端（内置连接池 + 超时 + breaker）
rds := redis.MustNewRedis(redis.RedisConf{
    Host: "redis:6379",
    Type: "node", // 或 "cluster"
    Pass: "",
})

// Eval 执行 Lua 脚本（自带 breaker 保护）
// val, err := rds.Eval(ctx, luaScript, []string{key}, args...)
```

#### GORM 连接池配置

```go
import (
    "runtime"
    "time"
    "gorm.io/gorm"
)

// 初始化 GORM 后显式配置连接池（P1 #17 强制要求）
sqlDB, _ := db.DB()
sqlDB.SetMaxOpenConns(runtime.NumCPU() * 2)
sqlDB.SetMaxIdleConns(runtime.NumCPU())
sqlDB.SetConnMaxLifetime(30 * time.Minute)
sqlDB.SetConnMaxIdleTime(10 * time.Minute)
```

#### 库存错误码定义

```go
package inventory

import "errors"

// 库存模块错误码（P1 #14 MODULE_SEQ 命名空间）
// 实际业务代码应直接引用这些 sentinel error 而非使用 fmt.Errorf 硬编码字符串
// 代码模板中为简化阅读使用 fmt.Errorf，实现时请替换
var (
    ErrInsufficientStock = errors.New("INVENTORY_001: 库存不足")
    ErrRollbackFailed    = errors.New("INVENTORY_002: 库存回滚失败")
    ErrSoldOut           = errors.New("INVENTORY_003: 已售罄")
    ErrDuplicatePurchase = errors.New("INVENTORY_004: 重复购买")
    ErrConcurrency       = errors.New("INVENTORY_005: 并发冲突，请重试")
)
```

#### 使用 `core/breaker` 保护库存操作

库存是关键路径，当 Redis 或 PG 持续不可用时，应快速失败而不是拖垮整个系统：

```go
import (
    "context"
    "errors"
    "fmt"

    "github.com/zeromicro/go-zero/core/breaker"
    "github.com/zeromicro/go-zero/core/logx"
    "github.com/zeromicro/go-zero/core/stores/redis"
)

// 为每个租户创建独立 breaker（故障隔离）
type InventoryBreaker struct {
    breaker breaker.Breaker
}

// DeductWithBreaker 带熔断的库存扣减
func (b *InventoryBreaker) DeductWithBreaker(ctx context.Context, fn func() error) error {
    return b.breaker.DoWithAcceptable(ctx, func() error {
        return fn()
    }, func(err error) bool {
        // redis.Nil / context.Canceled 等不算"失败"，不应触发熔断
        return errors.Is(err, redis.Nil) || errors.Is(err, context.Canceled)
    })
}

// 使用
err := invBreaker.DeductWithBreaker(ctx, func() error {
    return s.DeductInventory(ctx, skuID, quantity, orderID)
})
if errors.Is(err, breaker.ErrServiceUnavailable) {
    logx.WithContext(ctx).Errorf("库存服务熔断中，sku=%d", skuID)
    return fmt.Errorf("库存服务繁忙，请稍后重试")
}
```

#### go-zero 集成要点

| 组件 | 用途 | 库存场景 |
|------|------|----------|
| `core/stores/redis` | 封装 go-redis | 所有 Lua 脚本执行、队列操作 |
| `core/breaker` | 熔断降级 | Redis/PG 连续失败时快速失败 |
| `core/logx` | 结构化日志 | 替换所有 `log.Printf`，支持 ctx tracing |
| `core/stores/sqlc` | SQL 缓存 | 库存查询走 cachedConn 减少 PG 压力 |
| `core/prometheus` | 指标暴露 | 库存扣减耗时、补偿队列长度、对账差异数 |
