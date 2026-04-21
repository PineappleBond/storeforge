# 订单状态机模式 (Order State Machine)

## 触发条件

- 需要管理订单生命周期中的各种状态流转
- 需要防止非法状态跳转（如从未支付直接跳到已发货）
- 需要记录每次状态变更的原因和操作人
- 需要支持超时自动取消和自动确认
- 需要支持退款、退货等逆向流程

## 推荐第三方库

| 用途 | 推荐库 | 导入路径 |
|------|--------|----------|
| 状态机管理 | fsm | `github.com/looplab/fsm` |

**为什么用 looplab/fsm：** 手写 map-based 状态转换缺乏事件语义、回调钩子、状态变更拦截。`looplab/fsm` 提供事件驱动的状态机、enter/leave 回调、并发安全，且被广泛使用。

## 决策树

```
用户创建订单
  │
  ├─ 初始状态: pending（待支付）
  │
  ├─ 30 分钟内未支付？
  │   ├─ 是 → 自动取消 (pending → cancelled)
  │   └─ 否 → 用户支付成功 → pending → paid（已支付）
  │
  ├─ 已支付 (paid)
  │   ├─ 商家发货 → paid → shipped（已发货）
  │   └─ 用户申请退款 → paid → refunding（退款中）→ refunded（已退款）
  │
  ├─ 已发货 (shipped)
  │   ├─ 用户确认收货 → shipped → delivered（已收货）
  │   ├─ 7 天未确认 → shipped → delivered（自动确认收货）
  │   └─ 用户申请退货 → shipped → returning（退货中）→ returned（已退货）
  │
  ├─ 已收货 (delivered)
  │   └─ 15 天无售后 → delivered → completed（已完成）
  │
  └─ 已取消 (cancelled)
      └─ 终态，不可再流转

正向流程: pending → paid → shipped → delivered → completed
逆向流程: pending → cancelled（超时/手动取消）
退款流程: paid → refunding → refunded
退货流程: shipped → returning → returned
```

## 代码模板

### 1. 状态定义与转换规则

```go
package order

import (
	"context"
	"fmt"

	"github.com/looplab/fsm"
	"github.com/zeromicro/go-zero/core/logx"
)

// OrderStatus 订单状态
type OrderStatus int

const (
	StatusPending   OrderStatus = 1 // 待支付
	StatusPaid      OrderStatus = 2 // 已支付
	StatusShipped   OrderStatus = 3 // 已发货
	StatusDelivered OrderStatus = 4 // 已收货
	StatusCompleted OrderStatus = 5 // 已完成
	StatusCancelled OrderStatus = 6 // 已取消
	StatusRefunding OrderStatus = 7 // 退款中
	StatusRefunded  OrderStatus = 8 // 已退款
	StatusReturning OrderStatus = 9 // 退货中
	StatusReturned  OrderStatus = 10 // 已退货
)

func (s OrderStatus) String() string {
	m := map[OrderStatus]string{
		StatusPending:   "pending",
		StatusPaid:      "paid",
		StatusShipped:   "shipped",
		StatusDelivered: "delivered",
		StatusCompleted: "completed",
		StatusCancelled: "cancelled",
		StatusRefunding: "refunding",
		StatusRefunded:  "refunded",
		StatusReturning: "returning",
		StatusReturned:  "returned",
	}
	if name, ok := m[s]; ok {
		return name
	}
	return fmt.Sprintf("unknown(%d)", s)
}

// IsTerminal 判断是否为终态（不可再流转）
func (s OrderStatus) IsTerminal() bool {
	switch s {
	case StatusCompleted, StatusCancelled, StatusRefunded, StatusReturned:
		return true
	default:
		return false
	}
}

// StatusFromString 从 fsm 状态字符串转换为 OrderStatus
func StatusFromString(s string) OrderStatus {
	m := map[string]OrderStatus{
		"pending":   StatusPending,
		"paid":      StatusPaid,
		"shipped":   StatusShipped,
		"delivered": StatusDelivered,
		"completed": StatusCompleted,
		"cancelled": StatusCancelled,
		"refunding": StatusRefunding,
		"refunded":  StatusRefunded,
		"returning": StatusReturning,
		"returned":  StatusReturned,
	}
	if status, ok := m[s]; ok {
		return status
	}
	return OrderStatus(0)
}

// allowedTransitions 定义所有合法的状态转换（fsm 的 transitions 字段未导出，
// 且 Can() 仅接受 event 参数，无法从任意状态校验，因此自行维护转换表）
var allowedTransitions = map[string]map[string]string{
	// from_state -> event -> to_state
	"pending":   {"pay": "paid", "cancel": "cancelled"},
	"paid":      {"ship": "shipped", "request_refund": "refunding"},
	"shipped":   {"deliver": "delivered", "request_return": "returning"},
	"delivered": {"complete": "completed"},
}

// ValidateTransition 检查 from 状态下 event 是否合法
func ValidateTransition(from, event string) bool {
	toMap, ok := allowedTransitions[from]
	if !ok {
		return false
	}
	_, ok = toMap[event]
	return ok
}

// ValidateTransitionTarget 返回 from 状态下 event 的目标状态
func ValidateTransitionTarget(from, event string) string {
	toMap, ok := allowedTransitions[from]
	if !ok {
		return ""
	}
	return toMap[event]
}

// NewOrderFSM 使用 looplab/fsm 构建订单状态机（回调钩子用于审计日志接入）
func NewOrderFSM() *fsm.FSM {
	return fsm.NewFSM(
		"pending", // 初始状态
		fsm.Events{
			// 正向流程
			{Name: "pay", Src: []string{"pending"}, Dst: "paid"},
			{Name: "ship", Src: []string{"paid"}, Dst: "shipped"},
			{Name: "deliver", Src: []string{"shipped"}, Dst: "delivered"},
			{Name: "complete", Src: []string{"delivered"}, Dst: "completed"},
			// 取消流程
			{Name: "cancel", Src: []string{"pending"}, Dst: "cancelled"},
			// 退款流程
			{Name: "request_refund", Src: []string{"paid"}, Dst: "refunding"},
			{Name: "refund", Src: []string{"refunding"}, Dst: "refunded"},
			// 退货流程
			{Name: "request_return", Src: []string{"shipped"}, Dst: "returning"},
			{Name: "return", Src: []string{"returning"}, Dst: "returned"},
		},
		fsm.Callbacks{
			"enter_state": func(_ context.Context, e *fsm.Event) {
				// 状态变更时记录日志（后续可接入审计系统）
				orderID, _ := e.Args[0].(string)
				logx.Infof("order: state changed to %s for order %s, event: %s", e.Dst, orderID, e.Event)
			},
		},
	)
}

// ErrInvalidTransition 非法状态转换错误
type ErrInvalidTransition struct {
	OrderID string
	Event   string
	From    string
	To      string
}

func (e *ErrInvalidTransition) Error() string {
	return fmt.Sprintf("order: invalid state transition: %s → %s (order: %s, event: %s)", e.From, e.To, e.OrderID, e.Event)
}
```

### 2. 状态变更日志模型

```go
package order

import "time"

// StatusChangeLog 状态变更日志（只读追加表）
type StatusChangeLog struct {
	ID         int64       `gorm:"primaryKey;autoIncrement"`
	TenantID   string      `gorm:"type:varchar(64);not null;index:idx_tenant"`
	OrderID    string      `gorm:"type:varchar(64);not null;index:idx_order_seq"`
	Sequence   int         `gorm:"not null;index:idx_order_seq"` // 变更序号，单调递增
	FromStatus OrderStatus `gorm:"not null"`
	ToStatus   OrderStatus `gorm:"not null"`
	Reason     string      `gorm:"type:varchar(256);not null"` // 变更原因
	OperatorID string      `gorm:"type:varchar(64)"`           // 操作人（用户ID/系统）
	OperatorType string    `gorm:"type:varchar(32)"`           // system/user/admin
	CreatedAt  time.Time   `gorm:"not null;autoCreateTime;index"`
}

func (StatusChangeLog) TableName() string {
	return "order_status_change_logs"
}

// TransitionEvent 状态转换事件（用于内部传递）
type TransitionEvent struct {
	OrderID      string
	FromStatus   OrderStatus
	ToStatus     OrderStatus
	Reason       string
	OperatorID   string
	OperatorType string
	ExtraData    map[string]any // 扩展数据
}
```

### 3. Go 状态机实现

```go
package order

import (
	"context"
	"fmt"
	"time"

	"github.com/looplab/fsm"
	"github.com/zeromicro/go-zero/core/stores/redis"
	"gorm.io/gorm"
	"gorm.io/gorm/clause"
)

// StateMachine 订单状态机（基于 looplab/fsm）
type StateMachine struct {
	db  *gorm.DB
	fsm *fsm.FSM
	rds *redis.Redis // 分布式锁，多实例安全
}

// NewStateMachine 创建状态机实例
func NewStateMachine(db *gorm.DB, rds *redis.Redis) *StateMachine {
	return &StateMachine{
		db:  db,
		fsm: NewOrderFSM(),
		rds: rds,
	}
}

// TransitionRequest 状态转换请求
type TransitionRequest struct {
	TenantID     string                 `json:"tenant_id"`
	OrderID      string                 `json:"order_id"`
	Event        string                 `json:"event"` // fsm event name: "pay", "ship", etc.
	Reason       string                 `json:"reason"`
	OperatorID   string                 `json:"operator_id"`
	OperatorType string                 `json:"operator_type"` // system / user / admin
	ExtraData    map[string]any `json:"extra_data,omitempty"`
}

// Transition 执行状态转换（带事务和审计日志）
// fsm 仅用于校验状态转换合法性，不追踪内部状态
func (sm *StateMachine) Transition(ctx context.Context, req TransitionRequest) error {
	// 分布式锁：多实例安全，防止同一订单并发状态变更
	lockKey := fmt.Sprintf("order:lock:%s", req.OrderID)
	lock := redis.NewRedisLock(sm.rds, lockKey)
	ok, err := lock.AcquireCtx(ctx)
	if err != nil {
		return fmt.Errorf("order: failed to acquire lock for order %s: %w", req.OrderID, err)
	}
	if !ok {
		return fmt.Errorf("order: lock not acquired for order %s", req.OrderID)
	}
	defer func() {
		releaseCtx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
		defer cancel()
		_, _ = lock.ReleaseCtx(releaseCtx)
	}()

	return sm.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
		// 1. 加载当前订单（行锁 + 租户隔离）
		var order Order
		if err := tx.Clauses(clause.Locking{Strength: "UPDATE"}).
			Where("id = ? AND tenant_id = ?", req.OrderID, req.TenantID).
			First(&order).Error; err != nil {
			return fmt.Errorf("order: failed to load order: %w", err)
		}

		// 2. 使用 allowedTransitions 验证状态转换合法性（只读检查）
		currentStatus := order.Status.String()
		toStatusStr, allowed := allowedTransitions[currentStatus][req.Event]
		if !allowed {
			// 检查是否为未知事件
			known := false
			for _, m := range allowedTransitions {
				if _, eok := m[req.Event]; eok {
					known = true
					break
				}
			}
			if !known {
				return fmt.Errorf("order: unknown event: %s", req.Event)
			}
			// 找到目标状态用于错误信息
			target := "unknown"
			for _, m := range allowedTransitions {
				if t, ok := m[req.Event]; ok {
					target = t
					break
				}
			}
			return &ErrInvalidTransition{OrderID: req.OrderID, Event: req.Event, From: currentStatus, To: target}
		}
		toStatus := StatusFromString(toStatusStr)

		// 3. 获取下一个变更序号
		var maxSeq int
		tx.Model(&StatusChangeLog{}).
			Where("order_id = ? AND tenant_id = ?", req.OrderID, req.TenantID).
			Select("COALESCE(MAX(sequence), 0)").
			Scan(&maxSeq)

		// 4. 更新订单状态
		now := time.Now().UTC()
		updates := map[string]any{
			"status":     toStatus,
			"updated_at": now,
		}
		switch toStatus {
		case StatusPaid:
			updates["paid_at"] = now
		case StatusShipped:
			updates["shipped_at"] = now
		case StatusDelivered:
			updates["delivered_at"] = now
		case StatusCompleted:
			updates["completed_at"] = now
		case StatusCancelled:
			updates["cancelled_at"] = now
		case StatusRefunded:
			updates["refunded_at"] = now
		case StatusReturned:
			updates["returned_at"] = now
		}

		result := tx.Model(&Order{}).
			Where("id = ? AND tenant_id = ? AND status = ?", order.ID, req.TenantID, order.Status).
			Updates(updates)
		if result.Error != nil {
			return fmt.Errorf("order: failed to update status: %w", result.Error)
		}
		if result.RowsAffected == 0 {
			return fmt.Errorf("order: concurrent status change detected for order %s", req.OrderID)
		}

		// 5. 写入状态变更日志
		return tx.Create(&StatusChangeLog{
			TenantID:     req.TenantID,
			OrderID:      req.OrderID,
			Sequence:     maxSeq + 1,
			FromStatus:   order.Status,
			ToStatus:     toStatus,
			Reason:       req.Reason,
			OperatorID:   req.OperatorID,
			OperatorType: req.OperatorType,
			CreatedAt:    now,
		}).Error
	})
}

// CanTransition 查询某事件在当前状态下是否合法（只读，不修改 DB）
func (sm *StateMachine) CanTransition(currentStatus OrderStatus, event string) bool {
	toMap, ok := allowedTransitions[currentStatus.String()]
	if !ok {
		return false
	}
	_, ok = toMap[event]
	return ok
}

// GetStatusHistory 获取订单状态变更历史
func (sm *StateMachine) GetStatusHistory(
	ctx context.Context,
	tenantID, orderID string,
) ([]StatusChangeLog, error) {
	var logs []StatusChangeLog
	err := sm.db.WithContext(ctx).
		Where("tenant_id = ? AND order_id = ?", tenantID, orderID).
		Order("sequence ASC").
		Find(&logs).Error
	return logs, err
}
```

### 4. 定时任务：自动取消和自动确认

```go
package order

import (
	"context"
	"fmt"
	"time"

	"github.com/zeromicro/go-zero/core/logx"
	"github.com/zeromicro/go-zero/core/stores/redis"
	"gorm.io/gorm"
)

// AutoTaskConfig 自动任务配置
type AutoTaskConfig struct {
	AutoCancelTimeout   time.Duration // 未支付自动取消超时（默认 30 分钟）
	AutoConfirmTimeout  time.Duration // 未确认自动确认超时（默认 7 天）
	AutoCompleteTimeout time.Duration // 未完结自动完成超时（默认 15 天）
	BatchSize           int           // 每批处理数量
}

// DefaultAutoTaskConfig 默认配置
func DefaultAutoTaskConfig() AutoTaskConfig {
	return AutoTaskConfig{
		AutoCancelTimeout:   30 * time.Minute,
		AutoConfirmTimeout:  7 * 24 * time.Hour,
		AutoCompleteTimeout: 15 * 24 * time.Hour,
		BatchSize:           100,
	}
}

// AutoTaskRunner 自动状态转换任务执行器
type AutoTaskRunner struct {
	db     *gorm.DB
	sm     *StateMachine
	config AutoTaskConfig
	rds    *redis.Redis   // 分布式锁，多实例 leader election
	tenantID string
}

// NewAutoTaskRunner 创建自动任务执行器
func NewAutoTaskRunner(db *gorm.DB, sm *StateMachine, config AutoTaskConfig, tenantID string, rds *redis.Redis) *AutoTaskRunner {
	return &AutoTaskRunner{db: db, sm: sm, config: config, tenantID: tenantID, rds: rds}
}

// RunAutoCancel 自动取消超时未支付订单
func (atr *AutoTaskRunner) RunAutoCancel(ctx context.Context) error {
	// 分布式锁：防止多实例重复执行
	lock := redis.NewRedisLock(atr.rds, "order:auto_cancel")
	ok, err := lock.AcquireCtx(ctx)
	if err != nil || !ok {
		return nil // 其他实例正在执行，静默跳过
	}
	defer func() {
		releaseCtx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
		defer cancel()
		_, _ = lock.ReleaseCtx(releaseCtx)
	}()

	deadline := time.Now().UTC().Add(-atr.config.AutoCancelTimeout)
	return atr.processBatch(ctx, "auto_cancel_timeout", StatusPending, StatusCancelled, deadline, "created_at", func(tx *gorm.DB, ids []string) error {
		return tx.Model(&Order{}).
			Where("id IN ? AND status = ?", ids, StatusPending).
			Updates(map[string]any{
				"status":       StatusCancelled,
				"cancelled_at": time.Now().UTC(),
			}).Error
	})
}

// RunAutoConfirm 自动确认超时未确认收货订单
func (atr *AutoTaskRunner) RunAutoConfirm(ctx context.Context) error {
	// 分布式锁：防止多实例重复执行
	lock := redis.NewRedisLock(atr.rds, "order:auto_confirm")
	ok, err := lock.AcquireCtx(ctx)
	if err != nil || !ok {
		return nil // 其他实例正在执行，静默跳过
	}
	defer func() {
		releaseCtx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
		defer cancel()
		_, _ = lock.ReleaseCtx(releaseCtx)
	}()

	deadline := time.Now().UTC().Add(-atr.config.AutoConfirmTimeout)
	return atr.processBatch(ctx, "auto_confirm_timeout", StatusShipped, StatusDelivered, deadline, "shipped_at", func(tx *gorm.DB, ids []string) error {
		return tx.Model(&Order{}).
			Where("id IN ? AND status = ?", ids, StatusShipped).
			Updates(map[string]any{
				"status":       StatusDelivered,
				"delivered_at": time.Now().UTC(),
			}).Error
	})
}

// RunAutoComplete 自动完成超时未完结订单
func (atr *AutoTaskRunner) RunAutoComplete(ctx context.Context) error {
	// 分布式锁：防止多实例重复执行
	lock := redis.NewRedisLock(atr.rds, "order:auto_complete")
	ok, err := lock.AcquireCtx(ctx)
	if err != nil || !ok {
		return nil // 其他实例正在执行，静默跳过
	}
	defer func() {
		releaseCtx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
		defer cancel()
		_, _ = lock.ReleaseCtx(releaseCtx)
	}()

	deadline := time.Now().UTC().Add(-atr.config.AutoCompleteTimeout)
	return atr.processBatch(ctx, "auto_complete_timeout", StatusDelivered, StatusCompleted, deadline, "delivered_at", func(tx *gorm.DB, ids []string) error {
		return tx.Model(&Order{}).
			Where("id IN ? AND status = ?", ids, StatusDelivered).
			Updates(map[string]any{
				"status":       StatusCompleted,
				"completed_at": time.Now().UTC(),
			}).Error
	})
}

// processFn 批量处理函数
type processFn func(tx *gorm.DB, ids []string) error

func (atr *AutoTaskRunner) processBatch(
	ctx context.Context,
	reason string,
	from, to OrderStatus,
	deadline time.Time,
	timeField string,
	process processFn,
) error {
	for {
		// 1. 查询符合条件的订单 ID
		var orderIDs []string
		if err := atr.db.WithContext(ctx).
			Model(&Order{}).
			Where("tenant_id = ? AND status = ? AND "+timeField+" < ?", atr.tenantID, from, deadline).
			Order(timeField + " ASC").
			Limit(atr.config.BatchSize).
			Pluck("id", &orderIDs).Error; err != nil {
			return fmt.Errorf("auto_task: failed to query orders: %w", err)
		}

		if len(orderIDs) == 0 {
			break // 没有更多订单需要处理
		}

		// 2. 批量更新状态（带行锁验证）
		var processedCount int
		if err := atr.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
			result := process(tx, orderIDs)
			if result != nil {
				return result
			}

			// 3. 验证实际更新的行数，只记录成功变更的订单
			var affectedIDs []string
			if err := tx.Model(&Order{}).
				Where("id IN ? AND status = ?", orderIDs, to).
				Pluck("id", &affectedIDs).Error; err != nil {
				return fmt.Errorf("auto_task: failed to verify affected rows: %w", err)
			}
			if len(affectedIDs) == 0 {
				return nil // 所有订单已被其他进程处理
			}

			// 4. 批量查询受影响订单的 maxSeq
			type seqResult struct {
				OrderID string
				MaxSeq  int
			}
			var seqResults []seqResult
			if err := tx.Model(&StatusChangeLog{}).
				Select("order_id, COALESCE(MAX(sequence), 0) as max_seq").
				Where("order_id IN ? AND tenant_id = ?", affectedIDs, atr.tenantID).
				Group("order_id").
				Find(&seqResults).Error; err != nil {
				return fmt.Errorf("auto_task: failed to query sequences: %w", err)
			}

			seqMap := make(map[string]int, len(seqResults))
			for _, r := range seqResults {
				seqMap[r.OrderID] = r.MaxSeq
			}

			// 5. 批量构建日志记录
			now := time.Now().UTC()
			var changeLogs []StatusChangeLog
			for _, id := range affectedIDs {
				maxSeq := seqMap[id] // 0 if not found
				changeLogs = append(changeLogs, StatusChangeLog{
					TenantID:     atr.tenantID,
					OrderID:      id,
					Sequence:     maxSeq + 1,
					FromStatus:   from,
					ToStatus:     to,
					Reason:       reason,
					OperatorID:   "system",
					OperatorType: "system",
					CreatedAt:    now,
				})
			}
			if err := tx.Create(&changeLogs).Error; err != nil {
				return fmt.Errorf("auto_task: failed to write change logs: %w", err)
			}
			processedCount = len(affectedIDs)
			return nil
		}); err != nil {
			logx.WithContext(ctx).Errorf("auto_task [%s]: error processing batch: %v", reason, err)
			return err
		}

		logx.WithContext(ctx).Infof("auto_task [%s]: processed %d orders", reason, processedCount)

		if processedCount < atr.config.BatchSize {
			break // 最后一批
		}
	}

	return nil
}
```

### 5. Order 模型

```go
package order

import "time"

// Order 订单主表
type Order struct {
	ID          string      `gorm:"primaryKey;type:varchar(64)"`
	TenantID    string      `gorm:"type:varchar(64);not null;index:idx_tenant"`
	UserID      string      `gorm:"type:varchar(64);not null;index"`
	Status      OrderStatus `gorm:"not null;default:1;index:idx_status_created"`
	TotalAmount int64       `gorm:"not null"` // 总金额（分）
	PayAmount   int64       `gorm:"not null"` // 实付金额（分）
	Discount    int64       `gorm:"default:0"` // 优惠金额（分）

	// 时间戳
	CreatedAt   time.Time  `gorm:"not null;autoCreateTime;index:idx_status_created"`
	UpdatedAt   time.Time  `gorm:"not null;autoUpdateTime"`
	PaidAt      *time.Time `gorm:"null"`
	ShippedAt   *time.Time `gorm:"null"`
	DeliveredAt *time.Time `gorm:"null"`
	CompletedAt *time.Time `gorm:"null"`
	CancelledAt *time.Time `gorm:"null"`
	RefundedAt  *time.Time `gorm:"null"`
	ReturnedAt  *time.Time `gorm:"null"`
}

func (Order) TableName() string {
	return "orders"
}
```

### 6. 使用示例

```go
package main

import (
	"context"
	"storeforge/order"

	"github.com/zeromicro/go-zero/core/logx"
)

func main() {
	db := getDB()      // *gorm.DB
	rds := getRedis()  // *redis.Redis
	sm := order.NewStateMachine(db, rds)
	ctx := context.Background()
	tenantID := "TENANT_001"

	// === 示例 1: 用户支付成功 ===
	err := sm.Transition(ctx, order.TransitionRequest{
		TenantID:     tenantID,
		OrderID:      "ORD20260420001",
		Event:        "pay",
		Reason:       "用户完成微信支付",
		OperatorID:   "USER_12345",
		OperatorType: "user",
	})
	if err != nil {
		logx.WithContext(ctx).Errorf("transition failed: %v", err)
	}

	// === 示例 2: 商家发货 ===
	err = sm.Transition(ctx, order.TransitionRequest{
		TenantID:     tenantID,
		OrderID:      "ORD20260420001",
		Event:        "ship",
		Reason:       "商家通过顺丰发货，运单号 SF1234567890",
		OperatorID:   "ADMIN_001",
		OperatorType: "admin",
	})
	if err != nil {
		logx.WithContext(ctx).Errorf("transition failed: %v", err)
	}

	// === 示例 3: 非法转换会被拒绝 ===
	err = sm.Transition(ctx, order.TransitionRequest{
		TenantID:     tenantID,
		OrderID:      "ORD20260420001",
		Event:        "complete", // 不能从 shipped 直接跳到 completed
		Reason:       "非法跳转测试",
		OperatorID:   "SYSTEM",
		OperatorType: "system",
	})
	if err != nil {
		// 预期错误: order: invalid state transition: shipped → completed
		logx.WithContext(ctx).Errorf("expected error: %v", err)
	}

	// === 示例 4: 查看状态历史 ===
	history, _ := sm.GetStatusHistory(ctx, tenantID, "ORD20260420001")
	for _, entry := range history {
		logx.WithContext(ctx).Infof("[%d] %s → %s, 原因: %s, 操作人: %s(%s)",
			entry.Sequence, entry.FromStatus.String(), entry.ToStatus.String(),
			entry.Reason, entry.OperatorID, entry.OperatorType)
	}

	// === 示例 5: 运行自动取消任务（Cron 每 1 分钟执行）===
	atr := order.NewAutoTaskRunner(db, sm, order.DefaultAutoTaskConfig(), tenantID, rds)
	if err := atr.RunAutoCancel(ctx); err != nil {
		logx.WithContext(ctx).Errorf("auto cancel failed: %v", err)
	}
}
```

## 反模式

### 1. 直接更新状态字段

```go
// 错误：绕过状态机直接更新
db.Model(&Order{}).Where("id = ?", orderID).Update("status", StatusCompleted)
```

**问题**：跳过状态验证，可以从任意状态直接跳到 completed，破坏状态机约束。

**正确做法**：所有状态变更必须通过 `StateMachine.Transition()`，该方法内置转换验证 + 审计日志。

### 2. 缺少转换验证

```go
// 错误：没有验证 from → to 是否合法
func UpdateStatus(orderID string, newStatus OrderStatus) error {
    return db.Model(&Order{}).Where("id = ?", orderID).Update("status", newStatus).Error
}
```

**问题**：无法防止非法状态跳转，例如从 pending 直接到 delivered。

**正确做法**：使用 `ValidateTransition(currentStatus, newStatus)` 验证后再执行更新。

### 3. 不记录变更原因

```go
// 错误：状态变更不记录原因
db.Model(&Order{}).Where("id = ?", orderID).Updates(map[string]any{
    "status": StatusCancelled,
})
// 谁取消的？为什么取消？完全无从追溯
```

**问题**：用户投诉"订单为什么被取消"时无据可查，运营无法追踪异常取消。

**正确做法**：每次状态变更必须写入 `StatusChangeLog`，包含 `reason`、`operator_id`、`operator_type`。

### 4. 并发状态变更无保护

```go
// 错误：没有行锁，并发支付和取消可能同时执行
func Cancel(orderID string) {
    db.Model(&Order{}).Where("id = ? AND status = ?", orderID, StatusPending).
        Update("status", StatusCancelled)
}
```

**问题**：支付回调和超时取消同时触发，可能导致重复状态变更或数据不一致。

**正确做法**：使用 `SELECT ... FOR UPDATE` 行锁 + 乐观锁（`WHERE status = ?` 条件更新），检查 `RowsAffected`。

### 5. 终态可再次流转

```go
// 错误：允许终态继续流转
// cancelled → paid （已取消的订单被支付？）
```

**问题**：终态订单不应该再接受任何状态变更。

**正确做法**：在 `AllowedTransitions` 中不定义任何以终态为 `From` 的转换规则，`CanTransition` 会返回 false。

## 常见坑

### 1. 自动取消和支付回调的竞争条件

**坑**：订单在 30 分钟边界时刻，自动取消任务和支付回调同时执行。

**解决**：
- 使用 `SELECT ... FOR UPDATE` 行锁确保串行化
- 支付回调先检查 `status == StatusPending`，否则拒绝处理
- 自动取消同理，只处理 `status == StatusPending AND created_at < deadline` 的订单
- 使用数据库事务保证状态更新和日志写入的原子性

### 2. 自动任务扫描性能问题

**坑**：`WHERE status = 1 AND created_at < ?` 在大量订单下全表扫描。

**解决**：
- 在 `status` 和 `created_at` 上建立联合索引：`INDEX idx_status_created (status, created_at)`
- 批量处理（每批 100 条），避免一次锁太多行
- 使用延迟队列（RabbitMQ delayed exchange / Redis sorted set）替代轮询扫描

### 3. 状态变更日志无限增长

**坑**：`order_status_change_logs` 表随时间无限增长，影响查询性能。

**解决**：
- 按月分表或使用 PostgreSQL 分区表
- 定期归档 1 年以上的日志到冷存储
- 高频查询场景可考虑只保留最近 N 条，历史数据走归档表

### 4. 退款流程与支付回调的冲突

**坑**：用户在退款中（refunding），但支付回调延迟到达，尝试将订单设为 paid。

**解决**：
- 支付回调处理前先检查当前状态，如果已经是 refunding/refunded，直接拒绝并触发退款
- 支付回调必须是幂等的：同一个回调调用多次，结果一致
- 支付回调中增加状态校验：只接受 `pending → paid` 转换

### 5. 时区问题导致自动取消不准确

**坑**：`created_at` 存储时使用了本地时区，但服务器时区变化导致自动取消时间错误。

**解决**：
- 所有时间戳使用 UTC 存储
- `time.Now()` 统一使用 `time.Now().UTC()`
- 数据库连接配置时区：`?timezone=UTC`

### 6. 状态机规则变更后的兼容性

**坑**：新增状态（如 `StatusPartiallyShipped`）后，历史订单的数据迁移和规则兼容。

**解决**：
- 新增状态时同时添加 `AllowedTransitions` 规则
- 历史数据迁移脚本将旧状态映射到新状态
- 状态转换规则应该支持动态配置（从数据库或配置中心加载），而不是硬编码

### 7. 分布式环境下多实例并发执行自动任务

**坑**：多个服务实例同时运行 `RunAutoCancel`，重复处理同一批订单。

**解决**：
- 使用分布式锁（Redis `SETNX` / PostgreSQL `pg_advisory_lock`）
- 只有一个实例获得锁后执行自动任务
- 锁的 TTL 应小于任务执行间隔，防止死锁

## 测试策略

### 测试覆盖率目标

| 层级 | 覆盖率要求 | 说明 |
|------|-----------|------|
| L1 Unit | >= 80% | FSM 状态转换、非法转换拒绝、并发冲突检测 |
| L2 Integration | >= 70% | DB 事务 + 审计日志写入 + 行锁 |
| L3 E2E | 核心流程 | 完整订单生命周期 + 自动取消/确认 |

### 1. FSM 状态转换单元测试

```go
func TestFSM_Transition(t *testing.T) {
	f := order.NewOrderFSM()

	tests := []struct {
		name       string
		from       string
		event      string
		wantOK     bool
		wantTarget string
	}{
		{"pay: pending -> paid", "pending", "pay", true, "paid"},
		{"ship: paid -> shipped", "paid", "ship", true, "shipped"},
		{"deliver: shipped -> delivered", "shipped", "deliver", true, "delivered"},
		{"complete: delivered -> completed", "delivered", "complete", true, "completed"},
		{"cancel: pending -> cancelled", "pending", "cancel", true, "cancelled"},
		{"skip: paid -> completed (illegal)", "paid", "complete", false, ""},
		{"skip: pending -> shipped (illegal)", "pending", "ship", false, ""},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			// 使用 order.ValidateTransition 替代 f.Can(event, from)（fsm.Can 仅接受 event）
			got := order.ValidateTransition(tt.from, tt.event)
			if got != tt.wantOK {
				t.Errorf("ValidateTransition(%q, %q) = %v, want %v", tt.from, tt.event, got, tt.wantOK)
			}
			if tt.wantOK {
				target := order.ValidateTransitionTarget(tt.from, tt.event)
				if target != tt.wantTarget {
					t.Errorf("transition %s from %s destination = %s, want %s", tt.event, tt.from, target, tt.wantTarget)
				}
			}
		})
	}
}
```

### 2. 非法转换拒绝测试

```go
func TestStateMachine_IllegalTransition(t *testing.T) {
	db := setupTestDB(t)
	rds := setupTestRedis(t)
	sm := order.NewStateMachine(db, rds)
	ctx := context.Background()

	// Create a shipped order
	createOrder(t, db, "ORD-001", order.StatusShipped)

	// Attempt illegal transition: shipped -> completed (should fail)
	err := sm.Transition(ctx, order.TransitionRequest{
		TenantID:     "TENANT-1",
		OrderID:      "ORD-001",
		Event:        "complete", // requires delivered, not shipped
		Reason:       "illegal test",
		OperatorID:   "SYSTEM",
		OperatorType: "system",
	})
	if err == nil {
		t.Fatal("expected error for illegal transition, got nil")
	}
	var illegalErr *order.ErrInvalidTransition
	if !errors.As(err, &illegalErr) {
		t.Fatalf("expected ErrInvalidTransition, got %T: %v", err, err)
	}
	if illegalErr.From != "shipped" || illegalErr.To != "completed" {
		t.Errorf("got from=%s, to=%s; want from=shipped, to=completed", illegalErr.From, illegalErr.To)
	}
}
```

### 3. 并发状态转换测试

```go
func TestStateMachine_ConcurrentTransitions(t *testing.T) {
	db := setupTestDB(t)
	rds := setupTestRedis(t)
	sm := order.NewStateMachine(db, rds)
	ctx := context.Background()

	createOrder(t, db, "ORD-CONCURRENT", order.StatusPending)

	var wg sync.WaitGroup
	var mu sync.Mutex
	var errors []error

	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			err := sm.Transition(ctx, order.TransitionRequest{
				TenantID:     "TENANT-1",
				OrderID:      "ORD-CONCURRENT",
				Event:        "pay",
				Reason:       "concurrent test",
				OperatorID:   "SYSTEM",
				OperatorType: "system",
			})
			mu.Lock()
			errors = append(errors, err)
			mu.Unlock()
		}()
	}
	wg.Wait()

	// Exactly 1 success, 9 failures
	successCount := 0
	for _, err := range errors {
		if err == nil {
			successCount++
		}
	}
	if successCount != 1 {
		t.Errorf("expected exactly 1 successful transition, got %d", successCount)
	}
}
```

### 4. 自动取消幂等性测试

```go
func TestAutoTaskRunner_RunAutoCancel_Idempotent(t *testing.T) {
	db := setupTestDB(t)
	rds := setupTestRedis(t)
	sm := order.NewStateMachine(db, rds)
	atr := order.NewAutoTaskRunner(db, sm, order.AutoTaskConfig{
		AutoCancelTimeout: 30 * time.Minute,
		BatchSize:         50,
	}, "TENANT-1", rds)
	ctx := context.Background()

	// Create an expired pending order
	createExpiredOrder(t, db, "ORD-EXPIRED", order.StatusPending, time.Now().UTC().Add(-1*time.Hour))

	// First run should cancel
	if err := atr.RunAutoCancel(ctx); err != nil {
		t.Fatalf("first RunAutoCancel failed: %v", err)
	}
	assertOrderStatus(t, db, "ORD-EXPIRED", order.StatusCancelled)

	// Second run should be idempotent (no change, no error)
	if err := atr.RunAutoCancel(ctx); err != nil {
		t.Fatalf("second RunAutoCancel failed (idempotency): %v", err)
	}
	assertOrderStatus(t, db, "ORD-EXPIRED", order.StatusCancelled)
}
```

### 5. 多租户隔离测试

```go
func TestStateMachine_MultiTenantIsolation(t *testing.T) {
	db := setupTestDB(t)
	rds := setupTestRedis(t)
	sm := order.NewStateMachine(db, rds)
	ctx := context.Background()

	// Create orders in different tenants
	createOrder(t, db, "ORD-A", order.StatusPending) // TENANT-A
	createOrder(t, db, "ORD-B", order.StatusPending) // TENANT-B

	// Tenant A cannot transition Tenant B's order
	err := sm.Transition(ctx, order.TransitionRequest{
		TenantID:     "TENANT-A",
		OrderID:      "ORD-B",
		Event:        "pay",
		Reason:       "cross-tenant test",
		OperatorID:   "SYSTEM",
		OperatorType: "system",
	})
	if err == nil {
		t.Fatal("expected error for cross-tenant transition, got nil")
	}
}
```
