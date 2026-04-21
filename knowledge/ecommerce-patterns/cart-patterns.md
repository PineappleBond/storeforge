# 购物车模式 (Cart Patterns)

## Redis + PostgreSQL 双写购物车

---

### 触发条件

- 需要高性能读写的购物车服务
- 支持未登录游客购物车
- 登录后购物车合并
- 跨设备/多端同步
- 商品失效标记（下架、缺货）

---

### 决策树

```
购物车需求？
  ├─ 仅单端、无需持久化 → Redis only（游客临时购物车）
  ├─ 需要持久化 + 跨端同步 → Redis 缓存 + PG 持久化（双写）
  └─ 需要离线支持（Flutter App）→ Redis + PG + 本地 SQLite 三端同步
```

---

### 代码模板

#### 3.1 数据模型

```go
// model/cart.go
package model

import (
	"time"

	"gorm.io/gorm"
)

// CartItem PostgreSQL 持久化的购物车条目
type CartItem struct {
	ID        int64          `gorm:"primaryKey" json:"id"`
	TenantID  string         `gorm:"index:idx_tenant_user_sku" json:"tenant_id"`
	UserID    int64          `gorm:"uniqueIndex:idx_user_sku,where:deleted_at IS NULL;not null" json:"user_id"`
	SKUID     int64          `gorm:"uniqueIndex:idx_user_sku,where:deleted_at IS NULL;not null" json:"sku_id"`
	Quantity  int            `gorm:"not null;check:quantity > 0" json:"quantity"`
	Selected  bool           `gorm:"default:true" json:"selected"`
	IsValid   bool           `gorm:"default:true" json:"is_valid"` // 商品是否有效
	CreatedAt time.Time      `json:"created_at"`
	UpdatedAt time.Time      `json:"updated_at"`
	DeletedAt gorm.DeletedAt `gorm:"index" json:"-"`
}

func (CartItem) TableName() string {
	return "cart_items"
}

// CartItemDTO 返回给前端的 DTO（含商品信息）
type CartItemDTO struct {
	ID          int64  `json:"id"`
	SKUID       int64  `json:"sku_id"`
	ProductName string `json:"product_name"`
	SKUAttrs    string `json:"sku_attrs"` // JSON: {"color":"red","size":"L"}
	Price       int64  `json:"price"`     // 单价（分）
	CurrencyCode string `json:"currency_code"` // ISO 4217: "CNY", "USD"
	ImageURL    string `json:"image_url"`
	Quantity    int    `json:"quantity"`
	Selected    bool   `json:"selected"`
	IsValid     bool   `json:"is_valid"`
	HasStock    bool   `json:"has_stock"`
	Subtotal    int64  `json:"subtotal"` // price * quantity
}

// SKUInfo 商品基本信息（批量查询复用）
type SKUInfo struct {
	ID     int64  `gorm:"column:id"`
	Title  string `gorm:"column:title"`
	Price  int64  `gorm:"column:price"`
	Image  string `gorm:"column:image_url"`
	Stock  int64  `gorm:"column:stock"`
	Status int    `gorm:"column:status"`
}
```

#### 3.2 Redis Key 设计

```go
// pkg/cartkey/cart.go
package cartkey

import (
	"fmt"
	"strconv"
	"strings"
)

// Redis key 设计：
//   cart:{tenant_id}:{user_id}      → Hash, field=sku_id, value=quantity:selected:timestamp
//   cart:guest:{session_id}         → Hash, 游客购物车
//   cart:invalid:{tenant_id}:{user_id} → Set, 失效 SKU 列表
//   cart:dirty:{tenant_id}:{user_id}   → Set, 需要回写 PG 的 SKU 列表
//   cart:lock:{tenant_id}:{user_id}    → 分布式锁 key
//   cart:ttl:{tenant_id}:{user_id}     → 购物车 TTL key

func CartKey(tenantID string, userID int64) string {
	return fmt.Sprintf("cart:%s:%d", tenantID, userID)
}

func CartItemKey(tenantID string, userID int64, skuID int64) string {
	return fmt.Sprintf("cart:%s:%d:item:%d", tenantID, userID, skuID)
}

func CartTTLKey(tenantID string, userID int64) string {
	return fmt.Sprintf("cart:ttl:%s:%d", tenantID, userID)
}

func CartDirtyKey(tenantID string, userID int64) string {
	return fmt.Sprintf("cart:dirty:%s:%d", tenantID, userID)
}

func GuestCartKey(sessionID string) string {
	return fmt.Sprintf("cart:guest:%s", sessionID)
}

func InvalidSKUKey(tenantID string, userID int64) string {
	return fmt.Sprintf("cart:invalid:%s:%d", tenantID, userID)
}

func DirtySKUKey(tenantID string, userID int64) string {
	return fmt.Sprintf("cart:dirty:%s:%d", tenantID, userID)
}

func CartLockKey(tenantID string, userID int64) string {
	return fmt.Sprintf("cart:lock:%s:%d", tenantID, userID)
}

// CartFieldValue 存储在 Redis Hash 中的值格式
// "quantity:selected:updated_at_unix"
func EncodeCartField(quantity int, selected bool, ts int64) string {
	sel := "0"
	if selected {
		sel = "1"
	}
	return fmt.Sprintf("%d:%s:%d", quantity, sel, ts)
}

func DecodeCartField(val string) (quantity int, selected bool, ts int64, ok bool) {
	parts := strings.Split(val, ":")
	if len(parts) != 3 {
		return 0, false, 0, false
	}
	qty, err1 := strconv.Atoi(parts[0])
	sel := parts[1] == "1"
	ts, err2 := strconv.ParseInt(parts[2], 10, 64)
	if err1 != nil || err2 != nil {
		return 0, false, 0, false
	}
	return qty, sel, ts, true
}
```

#### 3.3 购物车增删改（Redis + PG 双写）

```go
// internal/logic/cart/cart_service.go
package cart

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"time"

	cartkeypkg "storeforge/pkg/cartkey"

	"github.com/redis/go-redis/v9"
	"github.com/zeromicro/go-zero/core/logx"
	"gorm.io/gorm"
	"gorm.io/gorm/clause"
)

var (
	ErrCartFull     = errors.New("cart is full")
	ErrItemNotFound = errors.New("item not found in cart")
)

const (
	CartItemTTL    = 30 * 24 * time.Hour // 购物车 30 天过期
	MaxCartItems   = 100                 // 单用户购物车上限
	MaxPerSKU      = 99                  // 单 SKU 最大数量
	FlushInterval  = 5 * time.Second     // 脏数据回写间隔
)

type CartService struct {
	rdb *redis.Client
	db  *gorm.DB
}

// AddItem 添加商品到购物车
func (s *CartService) AddItem(ctx context.Context, tenantID string, userID int64, skuID int64, quantity int) error {
	if quantity <= 0 {
		return fmt.Errorf("quantity must be positive")
	}
	if quantity > MaxPerSKU {
		quantity = MaxPerSKU
	}

	cartKey := cartkeypkg.CartKey(tenantID, userID)

	// 1. 检查购物车容量
	itemCount, err := s.rdb.HLen(ctx, cartKey).Result()
	if err != nil {
		logx.WithContext(ctx).Errorf("redis hlen cart: %v", err)
		return fmt.Errorf("redis hlen: %w", err)
	}
	if int(itemCount) >= MaxCartItems {
		// 检查是否已存在该 SKU
		exists, err := s.rdb.HExists(ctx, cartKey, fmt.Sprintf("%d", skuID)).Result()
		if err != nil {
			logx.WithContext(ctx).Errorf("redis hexists cart: %v", err)
			return fmt.Errorf("redis hexists: %w", err)
		}
		if !exists {
			return ErrCartFull
		}
	}

	// 2. 写入 Redis（Hash 结构，原子 HSET）
	field := fmt.Sprintf("%d", skuID)
	value := cartkeypkg.EncodeCartField(quantity, true, time.Now().Unix())
	if err := s.rdb.HSet(ctx, cartKey, field, value).Err(); err != nil {
		logx.WithContext(ctx).Errorf("redis write cart: %v", err)
		return fmt.Errorf("redis write: %w", err)
	}

	// 3. 刷新过期时间
	if err := s.rdb.Expire(ctx, cartKey, CartItemTTL).Err(); err != nil {
		logx.WithContext(ctx).Errorf("redis expire cart: %v", err)
	}

	// 4. 标记为脏数据（异步回写 PG）
	dirtyKey := cartkeypkg.DirtySKUKey(tenantID, userID)
	if err := s.rdb.SAdd(ctx, dirtyKey, field).Err(); err != nil {
		logx.WithContext(ctx).Errorf("redis sadd dirty: %v", err)
	}
	s.rdb.Expire(ctx, dirtyKey, CartItemTTL)

	// 5. 从失效列表移除
	invalidKey := cartkeypkg.InvalidSKUKey(tenantID, userID)
	s.rdb.SRem(ctx, invalidKey, field)

	return nil
}

// UpdateItem 更新购物车商品数量
func (s *CartService) UpdateItem(ctx context.Context, tenantID string, userID int64, skuID int64, quantity int) error {
	if quantity <= 0 {
		return s.DeleteItem(ctx, tenantID, userID, skuID)
	}
	if quantity > MaxPerSKU {
		quantity = MaxPerSKU
	}

	cartKey := cartkeypkg.CartKey(tenantID, userID)
	field := fmt.Sprintf("%d", skuID)

	// 1. 检查商品是否存在
	exists, err := s.rdb.HExists(ctx, cartKey, field).Result()
	if err != nil {
		logx.WithContext(ctx).Errorf("redis hexists cart: %v", err)
		return fmt.Errorf("redis hexists: %w", err)
	}
	if !exists {
		return ErrItemNotFound
	}

	// 2. 更新 Redis
	value := cartkeypkg.EncodeCartField(quantity, true, time.Now().Unix())
	if err := s.rdb.HSet(ctx, cartKey, field, value).Err(); err != nil {
		logx.WithContext(ctx).Errorf("redis hset cart: %v", err)
		return fmt.Errorf("redis hset: %w", err)
	}

	// 3. 标记脏数据
	dirtyKey := cartkeypkg.DirtySKUKey(tenantID, userID)
	if err := s.rdb.SAdd(ctx, dirtyKey, field).Err(); err != nil {
		logx.WithContext(ctx).Errorf("redis sadd dirty: %v", err)
	}

	return nil
}

// DeleteItem 从购物车删除商品
func (s *CartService) DeleteItem(ctx context.Context, tenantID string, userID int64, skuID int64) error {
	cartKey := cartkeypkg.CartKey(tenantID, userID)
	field := fmt.Sprintf("%d", skuID)

	// 1. Redis 删除
	if err := s.rdb.HDel(ctx, cartKey, field).Err(); err != nil {
		logx.WithContext(ctx).Errorf("redis hdel cart: %v", err)
		return fmt.Errorf("redis hdel: %w", err)
	}

	// 2. PG 删除（立即删除，不等异步回写）
	if err := s.db.WithContext(ctx).Where("tenant_id = ? AND user_id = ? AND sku_id = ?", tenantID, userID, skuID).Delete(&CartItem{}).Error; err != nil {
		logx.WithContext(ctx).Errorf("pg delete cart: %v", err)
		return fmt.Errorf("pg delete: %w", err)
	}

	return nil
}

// GetCart 获取购物车（优先读 Redis）
func (s *CartService) GetCart(ctx context.Context, tenantID string, userID int64) ([]CartItemDTO, error) {
	cartKey := cartkeypkg.CartKey(tenantID, userID)

	// 1. 从 Redis 读取
	all, err := s.rdb.HGetAll(ctx, cartKey).Result()
	if err != nil {
		logx.WithContext(ctx).Errorf("redis read cart: %v", err)
		return nil, fmt.Errorf("redis read: %w", err)
	}

	// 2. Redis 为空则回查 PG 并回填 Redis
	if len(all) == 0 {
		return s.loadFromDB(ctx, tenantID, userID)
	}

	// 3. 解析 Redis 数据
	var skuIDs []int64
	skuData := make(map[int64]struct {
		quantity int
		selected bool
	})
	for skuIDStr, val := range all {
		var skuID int64
		fmt.Sscanf(skuIDStr, "%d", &skuID)
		qty, selected, _, ok := cartkeypkg.DecodeCartField(val)
		if !ok {
			logx.WithContext(ctx).Errorf("decode cart field %s: invalid format", val)
			continue
		}
		skuIDs = append(skuIDs, skuID)
		skuData[skuID] = struct {
			quantity int
			selected bool
		}{quantity: qty, selected: selected}
	}

	// 批量查询商品信息，避免 N+1
	var skus []SKUInfo
	if err := s.db.WithContext(ctx).Table("skus").Where("id IN (?)", skuIDs).Find(&skus).Error; err != nil {
		logx.WithContext(ctx).Errorf("batch query skus: %v", err)
		return nil, fmt.Errorf("query sku info: %w", err)
	}

	skuMap := make(map[int64]*SKUInfo, len(skus))
	for i := range skus {
		skuMap[skus[i].ID] = &skus[i]
	}

	// 4. 组装 DTO
	var items []CartItemDTO
	for _, skuID := range skuIDs {
		data := skuData[skuID]
		dto := CartItemDTO{SKUID: skuID}
		if sku, ok := skuMap[skuID]; ok {
			dto.ProductName = sku.Title
			dto.Price = sku.Price
			dto.ImageURL = sku.Image
			dto.IsValid = sku.Status == 1
			dto.HasStock = sku.Stock > 0
		} else {
			dto.IsValid = false
		}
		dto.Quantity = data.quantity
		dto.Selected = data.selected
		dto.Subtotal = dto.Price * int64(data.quantity)
		items = append(items, dto)
	}

	return items, nil
}

// FlushDirty 将脏数据回写 PG（定时任务调用，带分布式锁）
func (s *CartService) FlushDirty(ctx context.Context, tenantID string, userID int64) error {
	dirtyKey := cartkeypkg.DirtySKUKey(tenantID, userID)

	// 获取分布式锁，使用唯一值校验所有权
	lockKey := cartkeypkg.CartLockKey(tenantID, userID)
	lockVal := fmt.Sprintf("flush:%d:%d", userID, time.Now().UnixNano())
	locked, err := s.rdb.SetNX(ctx, lockKey, lockVal, 10*time.Second).Result()
	if err != nil || !locked {
		return nil // 其他 goroutine 正在回写，跳过
	}

	// 使用 Lua 脚本原子释放锁（防止误删其他持有者的锁）
	unlockScript := `
		if redis.call("get", KEYS[1]) == ARGV[1] then
			return redis.call("del", KEYS[1])
		end
		return 0
	`
	defer s.rdb.Eval(ctx, unlockScript, []string{lockKey}, lockVal)

	// 获取所有脏 SKU
	skuIDs, err := s.rdb.SMembers(ctx, dirtyKey).Result()
	if err != nil {
		return err
	}
	if len(skuIDs) == 0 {
		return nil
	}

	cartKey := cartkeypkg.CartKey(tenantID, userID)

	// 批量构建 upsert 数据
	var upsertItems []CartItem
	var failedSKUIDs []string
	for _, skuIDStr := range skuIDs {
		var skuID int64
		fmt.Sscanf(skuIDStr, "%d", &skuID)

		val, err := s.rdb.HGet(ctx, cartKey, skuIDStr).Result()
		if err == redis.Nil {
			// Redis 中已删除 → PG 中也删除
			if err := s.db.Unscoped().WithContext(ctx).
				Where("tenant_id = ? AND user_id = ? AND sku_id = ?", tenantID, userID, skuID).
				Delete(&CartItem{}).Error; err != nil {
				logx.WithContext(ctx).Errorf("pg unscoped delete cart: %v", err)
				failedSKUIDs = append(failedSKUIDs, skuIDStr)
			}
			continue
		}
		if err != nil {
			logx.WithContext(ctx).Errorf("hget cart item: %v", err)
			failedSKUIDs = append(failedSKUIDs, skuIDStr)
			continue
		}

		qty, selected, _, ok := cartkeypkg.DecodeCartField(val)
		if !ok {
			logx.WithContext(ctx).Errorf("decode cart field %s: invalid format", val)
			failedSKUIDs = append(failedSKUIDs, skuIDStr)
			continue
		}
		upsertItems = append(upsertItems, CartItem{
			TenantID: tenantID,
			UserID:   userID,
			SKUID:    skuID,
			Quantity: qty,
			Selected: selected,
			IsValid:  true,
		})
	}

	// 批量 Upsert（避免 N+1）
	if len(upsertItems) > 0 {
		if err := s.db.WithContext(ctx).Clauses(clause.OnConflict{
			Columns:   []clause.Column{{Name: "tenant_id"}, {Name: "user_id"}, {Name: "sku_id"}},
			DoUpdates: clause.AssignmentColumns([]string{"quantity", "selected", "is_valid", "updated_at"}),
		}).Create(&upsertItems).Error; err != nil {
			logx.WithContext(ctx).Errorf("batch upsert cart: %v", err)
			// 全部 upsert 失败，不清除脏标记，等待下次重试
			return fmt.Errorf("batch upsert: %w", err)
		}
	}

	// 仅清除成功回写的 SKU 的脏标记
	if len(failedSKUIDs) == 0 {
		// 全部成功，清除整个脏集合
		if err := s.rdb.Del(ctx, dirtyKey).Err(); err != nil {
			logx.WithContext(ctx).Errorf("redis del dirty: %v", err)
		}
	} else {
		// 部分失败，仅移除成功的 SKU
		for _, skuIDStr := range skuIDs {
			found := false
			for _, f := range failedSKUIDs {
				if f == skuIDStr {
					found = true
					break
				}
			}
			if !found {
				s.rdb.SRem(ctx, dirtyKey, skuIDStr)
			}
		}
	}
	return nil
}

// loadFromDB 从 PG 加载购物车并回填 Redis
func (s *CartService) loadFromDB(ctx context.Context, tenantID string, userID int64) ([]CartItemDTO, error) {
	var items []CartItem
	if err := s.db.WithContext(ctx).Where("tenant_id = ? AND user_id = ? AND is_valid = true", tenantID, userID).Find(&items).Error; err != nil {
		logx.WithContext(ctx).Errorf("load cart from db: %v", err)
		return nil, fmt.Errorf("load cart from db: %w", err)
	}

	cartKey := cartkeypkg.CartKey(tenantID, userID)
	pipe := s.rdb.Pipeline()
	for _, item := range items {
		field := fmt.Sprintf("%d", item.SKUID)
		value := cartkeypkg.EncodeCartField(item.Quantity, item.Selected, time.Now().Unix())
		pipe.HSet(ctx, cartKey, field, value)
	}
	pipe.Expire(ctx, cartKey, CartItemTTL)
	if _, err := pipe.Exec(ctx); err != nil {
		logx.WithContext(ctx).Errorf("redis pipeline exec: %v", err)
	}

	// 批量查询商品信息
	var skuIDs []int64
	for _, item := range items {
		skuIDs = append(skuIDs, item.SKUID)
	}
	skuMap := make(map[int64]SKUInfo)
	if len(skuIDs) > 0 {
		var skus []SKUInfo
		if err := s.db.WithContext(ctx).Table("skus").Where("id IN (?)", skuIDs).Find(&skus).Error; err != nil {
			logx.WithContext(ctx).Errorf("batch query skus in loadFromDB: %v", err)
		} else {
			for _, sku := range skus {
				skuMap[sku.ID] = sku
			}
		}
	}

	// 转换为 DTO
	var dtos []CartItemDTO
	for _, item := range items {
		dto := CartItemDTO{SKUID: item.SKUID}
		if sku, ok := skuMap[item.SKUID]; ok {
			dto.ProductName = sku.Title
			dto.Price = sku.Price
			dto.ImageURL = sku.Image
			dto.IsValid = sku.Status == 1
			dto.HasStock = sku.Stock > 0
		} else {
			dto.IsValid = false
		}
		dto.Quantity = item.Quantity
		dto.Selected = item.Selected
		dto.Subtotal = dto.Price * int64(item.Quantity)
		dtos = append(dtos, dto)
	}

	return dtos, nil
}

// querySKUInfo 查询 SKU 信息（带缓存）
func (s *CartService) querySKUInfo(ctx context.Context, skuID int64) CartItemDTO {
	// 优先从 Redis 缓存读取
	cacheKey := fmt.Sprintf("sku:info:%d", skuID)
	if cached, err := s.rdb.Get(ctx, cacheKey).Result(); err == nil {
		var dto CartItemDTO
		if json.Unmarshal([]byte(cached), &dto) == nil {
			return dto
		}
	}

	// 缓存未命中，查询数据库
	var sku SKUInfo
	if err := s.db.WithContext(ctx).Table("skus").Where("id = ?", skuID).First(&sku).Error; err != nil {
		return CartItemDTO{SKUID: skuID, IsValid: false}
	}

	dto := CartItemDTO{
		SKUID:       skuID,
		ProductName: sku.Title,
		Price:       sku.Price,
		ImageURL:    sku.Image,
		IsValid:     sku.Status == 1,
		HasStock:    sku.Stock > 0,
	}

	// 回填缓存，TTL 1 小时
	if b, err := json.Marshal(dto); err == nil {
		s.rdb.Set(ctx, cacheKey, b, 1*time.Hour)
	}

	return dto
}
```

#### 3.4 登录时购物车合并

```go
// internal/logic/cart/merge.go
package cart

import (
	"context"
	"fmt"
	"time"

	cartkeypkg "storeforge/pkg/cartkey"
)

// MergeGuestCart 游客登录时合并购物车
// guestCart items 合并到 userCart，相同 SKU 数量相加
func (s *CartService) MergeGuestCart(ctx context.Context, tenantID string, userID int64, sessionID string) error {
	guestKey := cartkeypkg.GuestCartKey(sessionID)
	userKey := cartkeypkg.CartKey(tenantID, userID)

	// 1. 获取游客购物车
	guestItems, err := s.rdb.HGetAll(ctx, guestKey).Result()
	if err != nil || len(guestItems) == 0 {
		return nil // 游客购物车为空，无需合并
	}

	// 2. 获取用户购物车
	userItems, err := s.rdb.HGetAll(ctx, userKey).Result()
	if err != nil {
		userItems = make(map[string]string)
	}

	// 3. 合并策略：相同 SKU 数量相加，上限 MaxCartItems
	pipe := s.rdb.Pipeline()
	for skuID, guestVal := range guestItems {
		guestQty, _, _, ok := cartkeypkg.DecodeCartField(guestVal)
		if !ok {
			logx.WithContext(ctx).Errorf("decode guest cart field %s: invalid format", guestVal)
			continue
		}

		if userVal, exists := userItems[skuID]; exists {
			// 已存在：数量相加（上限 MaxPerSKU）
			userQty, selected, _, _ := cartkeypkg.DecodeCartField(userVal)
			newQty := userQty + guestQty
			if newQty > MaxPerSKU {
				newQty = MaxPerSKU
			}
			pipe.HSet(ctx, userKey, skuID, cartkeypkg.EncodeCartField(newQty, selected, time.Now().Unix()))
		} else {
			// 不存在：直接加入
			pipe.HSet(ctx, userKey, skuID, guestVal)
		}
	}

	pipe.Expire(ctx, userKey, CartItemTTL)

	// 4. 删除游客购物车
	pipe.Del(ctx, guestKey)

	_, err = pipe.Exec(ctx)
	if err != nil {
		return fmt.Errorf("merge cart: %w", err)
	}

	// 5. 标记所有合并的商品为脏数据，触发 PG 回写
	dirtyKey := cartkeypkg.DirtySKUKey(tenantID, userID)
	for skuID := range guestItems {
		s.rdb.SAdd(ctx, dirtyKey, skuID)
	}

	return nil
}
```

#### 3.5 购物车商品失效检查

```go
// internal/logic/cart/invalidate.go
package cart

import (
	"context"
	"fmt"

	cartkeypkg "storeforge/pkg/cartkey"
)

// InvalidateSKUs 标记 SKU 为失效（商品下架/库存不足时调用）
func (s *CartService) InvalidateSKUs(ctx context.Context, tenantID string, userID int64, skuIDs []int64) error {
	cartKey := cartkeypkg.CartKey(tenantID, userID)
	invalidKey := cartkeypkg.InvalidSKUKey(tenantID, userID)

	pipe := s.rdb.Pipeline()
	for _, skuID := range skuIDs {
		field := fmt.Sprintf("%d", skuID)
		// 从购物车 Hash 中移除
		pipe.HDel(ctx, cartKey, field)
		// 加入失效集合
		pipe.SAdd(ctx, invalidKey, field)
	}
	pipe.Expire(ctx, invalidKey, 7*24*time.Hour) // 失效商品保留 7 天

	_, err := pipe.Exec(ctx)
	return err
}

// CheckInvalidSKUs 定时任务：批量检查用户购物车中的 SKU 是否仍然有效
//
// 生产实现要点：
//   1. 反向索引——商品下架/库存变更时主动推送失效事件，而非全量扫描
//   2. Lua 脚本原子批量检查每个用户的购物车（避免 N+1）
//   3. 租户感知迭代：按 tenantID 分区扫描 "cart:{tenant_id}:*" key
//   4. Bloom Filter 快速判断 SKU 是否有效
func (s *CartService) CheckInvalidSKUs(ctx context.Context, productService ProductValidityChecker) error {
	// TODO: 迭代所有租户的购物车 key
	// pattern := fmt.Sprintf("cart:%s:*", "*")
	// 或使用租户配置列表逐一扫描

	// TODO: 对每个用户购物车，构建 SKU ID 列表
	// items, err := s.rdb.HGetAll(ctx, cartKey).Result()

	// TODO: 批量查询商品服务获取有效 SKU 列表
	// validSKUs, err := productService.BatchCheckValidity(ctx, skuIDs)

	// TODO: 对失效 SKU 调用 InvalidateSKUs
	// if len(invalidSKUIDs) > 0 {
	//     s.InvalidateSKUs(ctx, tenantID, userID, invalidSKUIDs)
	// }

	_ = productService // suppress unused warning
	return nil
}

// ProductValidityChecker 商品有效性检查接口（可替换实现）
type ProductValidityChecker interface {
	BatchCheckValidity(ctx context.Context, skuIDs []int64) (valid []int64, invalid []int64, err error)
}
```

#### 3.6 跨端同步机制

```go
// internal/logic/cart/sync.go
package cart

import (
	"context"
	"encoding/json"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
	"github.com/zeromicro/go-zero/core/logx"
	"gorm.io/gorm"
)

// CartSyncService 跨端同步服务
// 适用于：用户同时在手机 App 和 Web 端操作购物车
type CartSyncService struct {
	rdb      *redis.Client
	db       *gorm.DB
	tenantID string
}

// SyncCart 同步购物车（基于版本号/时间戳的乐观同步）
func (s *CartSyncService) SyncCart(ctx context.Context, userID int64, clientSeq int64) (int64, error) {
	syncKey := fmt.Sprintf("cart:sync:%s:%d", s.tenantID, userID)

	// 获取服务端版本号
	serverSeq, err := s.rdb.Get(ctx, syncKey).Int64()
	if err == redis.Nil {
		return clientSeq, nil
	}
	if err != nil {
		return 0, fmt.Errorf("redis get sync seq: %w", err)
	}

	// 客户端版本 < 服务端版本：其他端有修改，需要拉取最新数据
	if clientSeq < serverSeq {
		return serverSeq, nil
	}

	// 客户端版本 >= 服务端版本：本地最新，无需同步
	return clientSeq, nil
}

// WriteCart 写入购物车并递增版本号
func (s *CartSyncService) WriteCart(ctx context.Context, userID int64) (int64, error) {
	syncKey := fmt.Sprintf("cart:sync:%s:%d", s.tenantID, userID)

	// 每次写操作递增版本号
	newSeq, err := s.rdb.Incr(ctx, syncKey).Result()
	if err != nil {
		return 0, fmt.Errorf("incr sync seq: %w", err)
	}

	// 设置过期时间（24 小时无操作则失效）
	if err := s.rdb.Expire(ctx, syncKey, 24*time.Hour).Err(); err != nil {
		logx.WithContext(ctx).Errorf("redis expire sync key: %v", err)
	}

	return newSeq, nil
}

// PublishCartChange 发布购物车变更事件（WebSocket / SSE 推送）
func (s *CartSyncService) PublishCartChange(ctx context.Context, tenantID string, userID int64, skuID int64, action string) error {
	channel := fmt.Sprintf("cart:change:%s:%d", tenantID, userID)
	message, err := json.Marshal(map[string]interface{}{
		"sku_id": skuID,
		"action": action,
		"ts":     time.Now().Unix(),
	})
	if err != nil {
		return fmt.Errorf("json marshal cart change: %w", err)
	}
	return s.rdb.Publish(ctx, channel, message).Err()
}
```

---

### 反模式

| 反模式 | 风险 | 正确做法 |
|--------|------|----------|
| 只用 Redis 不持久化 | Redis 重启/故障数据全丢 | Redis 缓存 + PG 持久化双写 |
| 登录后不合并游客购物车 | 用户丢失已选商品，体验差 | 登录时自动合并（数量相加） |
| 购物车无限增长 | 内存浪费，加载变慢 | 设置上限（100 项）+ 30 天过期 |
| 不处理失效商品 | 用户看到已下架商品，下单失败 | 定时检查 + 主动标记失效 |
| 跨端不同步 | 多设备数据不一致 | Redis Pub/Sub 推送 + 版本号同步 |
| 直接返回 GORM model 给前端 | 暴露内部字段 | 使用 CartItemDTO 转换 |
| 购物车操作不走分布式锁 | 并发操作导致数据不一致 | 写操作使用 cart:lock:{user_id} |
| 不同步库存状态 | 加购成功但库存已空 | 加购时检查库存，下单前二次检查 |

---

### 常见坑

**坑 1：Redis Hash HGetAll 大 key 问题**
- **现象**：用户购物车商品很多时，HGetAll 返回大量数据导致 Redis 阻塞
- **解决**：设置单用户购物车上限（100 项）；使用 HSCAN 分批获取；监控 big key

**坑 2：双写不一致**
- **现象**：Redis 写入成功但 PG 写入失败，导致数据不一致
- **解决**：Redis 为主写入，PG 异步回写（dirty key 机制）；定时任务补偿回写失败的数据；回写使用批量 Upsert 减少数据库压力

**坑 3：合并购物车时数量超出限制**
- **现象**：游客购物车 50 件 A 商品 + 用户购物车 60 件 A 商品 = 110 件，超过单 SKU 上限
- **解决**：合并时对每个 SKU 做上限检查（`MaxPerSKU`），超出则截断

**坑 4：游客 session 丢失导致购物车消失**
- **现象**：用户清除浏览器 Cookie/缓存后，游客购物车数据丢失
- **解决**：使用设备指纹 + localStorage 持久化 session ID；提供"恢复购物车"功能；尽早引导用户登录

**坑 5：定时回写与并发写入冲突**
- **现象**：定时任务正在回写 PG，同时用户修改了同一 SKU，回写覆盖了最新数据
- **解决**：回写时使用 Redis 中的最新值（回写前再读一次）；或使用时间戳判断，只回写比 PG 中更新的记录

**坑 6：失效商品检查性能差**
- **现象**：遍历所有用户购物车检查 SKU 有效性，数据量大时超时
- **解决**：反向索引——在商品下架/库存变更时主动推送失效事件到受影响的购物车，而非定时全量扫描；或使用 Bloom Filter 快速判断 SKU 是否有效

---

### 测试策略

#### L1 单元测试场景（目标覆盖率 >= 80%）

| 场景 | 测试点 |
| ------ | ------ |
| **AddItem - 正常添加** | Redis Hash 写入正确、脏标记设置、TTL 刷新 |
| **AddItem - 购物车已满** | 返回 ErrCartFull，超过 MaxCartItems 且 SKU 不存在时拒绝 |
| **AddItem - 重复 SKU** | 已存在时覆盖数量（HSet 语义） |
| **AddItem - 无效数量** | quantity <= 0 返回错误 |
| **UpdateItem - 正常更新** | Redis 值更新、脏标记设置 |
| **UpdateItem - 商品不存在** | 返回 ErrItemNotFound |
| **UpdateItem - 数量为 0** | 触发 DeleteItem 行为 |
| **DeleteItem - 正常删除** | Redis HDel + PG 软删除 |
| **GetCart - 空购物车** | 回退到 loadFromDB 加载 PG 数据 |
| **GetCart - 有缓存数据** | 从 Redis 解析 + 批量查询 SKU 信息 |
| **GetCart - N+1 防护** | 验证使用 IN 查询而非循环查询 |
| **FlushDirty - 正常回写** | Upsert 到 PG、清除脏标记 |
| **FlushDirty - 锁竞争** | SetNX 失败时跳过，不阻塞 |
| **FlushDirty - 锁所有权** | 仅在 lock value 匹配时释放锁 |
| **FlushDirty - Redis 已删除** | PG 中同步删除（Unscoped） |
| **MergeGuestCart - 正常合并** | 游客 SKU 合并到用户购物车，数量相加 |
| **MergeGuestCart - 游客空** | 直接返回 nil |
| **MergeGuestCart - 数量上限** | 合并后单 SKU 数量不超过 MaxPerSKU |
| **InvalidateSKUs - 批量失效** | 从购物车移除 + 加入失效集合 |
| **多租户隔离** | 不同 tenantID 的购物车数据互不干扰 |

#### L2 集成测试

- **使用 miniredis** 测试 Redis 实际写入，验证序列化/反序列化、TTL、Pipeline 批量操作
- Redis + PG 双写一致性：写入 Redis 后触发 PG 回写，验证数据一致
- 游客登录合并流程：创建游客购物车 -> 登录 -> 合并 -> 验证用户购物车
- 分布式锁并发安全：多 goroutine 同时 FlushDirty，验证无数据损坏

#### L3 E2E 测试

- 完整购物车生命周期：加购 -> 改数量 -> 删除 -> 下单 -> 清空
- 多端同步：Web 端加购 -> App 端查看 -> 数据一致
