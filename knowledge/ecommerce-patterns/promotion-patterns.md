# 促销优惠模式 (Promotion Patterns)

## 触发条件

- 需要实现优惠券（单品券/店铺券/平台券）、满减、限时折扣、拼团等促销活动
- 需要处理多种优惠的互斥与叠加规则
- 需要控制促销预算上限（避免超发）
- 需要完整的优惠计算审计追踪

## 推荐第三方库

| 用途 | 推荐库 | 导入路径 |
|------|--------|----------|
| 金额精确计算（促销分摊） | decimal | `github.com/shopspring/decimal` |
| 表达式/规则引擎（可选） | expr | `github.com/expr-lang/expr` |

**为什么用 shopspring/decimal：** 促销按比例分摊优惠时，`int64` 除法会产生精度损失（差 1 分钱）。`decimal` 提供精确的十进制运算，支持 `ROUND_HALF_UP` 舍入。

## 决策树

```
用户下单时选择优惠
  │
  ├─ 优惠类型是什么？
  │   ├─ 单品券 → 直接抵扣商品单价（最高优先级）
  │   ├─ 店铺券 → 抵扣店铺内所有商品（次高优先级）
  │   ├─ 平台券 → 抵扣全场商品（中优先级）
  │   ├─ 满减活动 → 满足门槛后减免（低优先级）
  │   └─ 限时折扣 → 商品打折（最低优先级）
  │
  ├─ 是否与其他优惠互斥？
  │   ├─ 单品券 + 店铺券 → 互斥，只能选一张
  │   ├─ 店铺券 + 平台券 → 互斥，只能选一张
  │   ├─ 优惠券（任何券）+ 满减 → 可叠加
  │   ├─ 优惠券 + 限时折扣 → 可叠加（先折扣后券）
  │   └─ 满减 + 限时折扣 → 可叠加（先折扣后满减）
  │
  ├─ 预算是否充足？
  │   ├─ 预算充足 → 正常计算
  │   └─ 预算不足 → 优惠不可用（前端显示已抢光）
  │
  └─ 计算顺序（从低到高逐层叠加，每层基于前一层折后价）
      1. 限时折扣（P5，最先算，商品级别打折）
      2. 单品券（P4，叠加在折扣后价格上）
      3. 店铺券或平台券（P2/P3 二选一）
      4. 满减（P1，最后算，基于折后总价）
```

## 优惠优先级与互斥规则

```
计算顺序（从先到后，每步基于前一步的折后价）:
  C1: 限时折扣（商品级别打折，最先算）
  C2: 单品券（只针对特定 SKU）
  C3: 店铺券/平台券（二选一，取优惠最大的）
  C4: 满减（基于折后总价，最后算）

互斥矩阵（横竖交叉 = 是否可同时使用）:
              单品券  店铺券  平台券  满减   折扣
  单品券       —      ✗      ✗      ✓      ✓
  店铺券       ✗      —      ✗      ✓      ✓
  平台券       ✗      ✗      —      ✓      ✓
  满减         ✓      ✓      ✓      —      ✓
  折扣         ✓      ✓      ✓      ✓      —

计算顺序: C1 限时折扣 → C2 单品券 → C3 店铺券/平台券(二选一) → C4 满减
最终优惠金额 = 各步骤优惠之和，但不超过商品总价
```

## 代码模板

### 1. 促销类型与规则定义

```go
package promotion

import (
	"errors"
	"fmt"
	"time"

	"github.com/zeromicro/go-zero/core/logx"
)

// PromotionType 促销类型
type PromotionType int

const (
	TypeSingleItemCoupon PromotionType = iota + 1 // 单品券
	TypeStoreCoupon                              // 店铺券
	TypePlatformCoupon                           // 平台券
	TypeFullReduction                            // 满减
	TypeFlashDiscount                            // 限时折扣
)

func (t PromotionType) String() string {
	m := map[PromotionType]string{
		TypeSingleItemCoupon: "single_item_coupon",
		TypeStoreCoupon:      "store_coupon",
		TypePlatformCoupon:   "platform_coupon",
		TypeFullReduction:    "full_reduction",
		TypeFlashDiscount:    "flash_discount",
	}
	if name, ok := m[t]; ok {
		return name
	}
	return fmt.Sprintf("unknown(%d)", t)
}

// Priority 返回促销优先级（数字越小优先级越高）
func (t PromotionType) Priority() int {
	switch t {
	case TypeSingleItemCoupon:
		return 1
	case TypeStoreCoupon:
		return 2
	case TypePlatformCoupon:
		return 3
	case TypeFullReduction:
		return 4
	case TypeFlashDiscount:
		return 5
	default:
		return 99
	}
}

// PromotionStatus 促销状态
type PromotionStatus int

const (
	StatusDisabled PromotionStatus = iota // 0=停用
	StatusEnabled                         // 1=启用
)

// CouponStatus 优惠券状态
type CouponStatus int

const (
	CouponUnused CouponStatus = iota // 0=未使用
	CouponUsed                      // 1=已使用
	CouponExpired                   // 2=已过期
)

// PromotionRule 促销规则
type PromotionRule struct {
	ID               string          `gorm:"primaryKey;type:varchar(64)"`
	TenantID         string          `gorm:"type:varchar(64);not null;index"` // 多租户隔离
	Type             PromotionType   `gorm:"not null;index"`
	Name             string          `gorm:"type:varchar(128);not null"`
	DiscountAmount   int64           `gorm:"not null"`                          // 优惠金额（分），满减/券用
	DiscountRate     int64           `gorm:"default:100"`                       // 折扣后百分比（100=不打折，80=八折即原价的80%）
	ThresholdAmount  int64           `gorm:"default:0"`                         // 满减门槛金额（分）
	TargetSKU        string          `gorm:"type:varchar(64)"`                  // 单品券/折扣的目标 SKU
	TargetStoreID    string          `gorm:"type:varchar(64)"`                  // 店铺券/满减的目标店铺
	MaxDiscount      int64           `gorm:"default:0"`                         // 最大优惠金额（封顶）
	TotalBudget      int64           `gorm:"not null"`                          // 总预算（分）
	UsedBudget       int64           `gorm:"default:0"`                         // 已使用预算（分）
	UserLimit        int             `gorm:"default:1"`                         // 每人限领次数
	StartTime        time.Time       `gorm:"not null;index"`
	EndTime          time.Time       `gorm:"not null;index"`
	Status           PromotionStatus `gorm:"default:1"`                         // StatusEnabled=启用, StatusDisabled=停用
	CreatedAt        time.Time       `gorm:"not null;autoCreateTime"`
}

func (PromotionRule) TableName() string {
	return "promotion_rules"
}

// Coupon 用户领取的优惠券
type Coupon struct {
	ID           string       `gorm:"primaryKey;type:varchar(64)"`
	TenantID     string       `gorm:"type:varchar(64);not null;index"` // 多租户隔离
	UserID       string       `gorm:"type:varchar(64);not null;index"` // 加密存储，展示时脱敏
	RuleID       string       `gorm:"type:varchar(64);not null;index"`
	Status       CouponStatus `gorm:"default:0"`                       // 0=未使用, 1=已使用, 2=已过期
	UsedOrderID  string       `gorm:"type:varchar(64)"`
	UsedAt       time.Time    `gorm:"null"`
	ObtainedAt   time.Time    `gorm:"not null;autoCreateTime"`
}

func (Coupon) TableName() string {
	return "coupons"
}

// ErrPromotion 促销相关错误
var (
	ErrPromotionExpired      = errors.New("promotion: expired")
	ErrPromotionBudgetExceeded = errors.New("promotion: budget exceeded")
	ErrPromotionIncompatible = errors.New("promotion: incompatible promotions")
	ErrPromotionUserLimitReached = errors.New("promotion: user limit reached")
	ErrPromotionThresholdNotMet = errors.New("promotion: threshold not met")
	ErrPromotionInvalidDiscountRate = errors.New("promotion: invalid discount rate (must be 1-100)")
)

// Validate 校验促销规则的有效性
func (r *PromotionRule) Validate() error {
	if r.StartTime.After(r.EndTime) {
		return errors.New("promotion: start_time after end_time")
	}
	if r.Type == TypeFlashDiscount && (r.DiscountRate < 1 || r.DiscountRate > 100) {
		return ErrPromotionInvalidDiscountRate
	}
	if r.TotalBudget < 0 {
		return errors.New("promotion: total_budget must be non-negative")
	}
	return nil
}
```

### 2. 互斥规则引擎

```go
package promotion

// MutuallyExclusiveGroups 互斥组定义
// 同一组内的促销只能选一个
var MutuallyExclusiveGroups = map[string][]PromotionType{
	"coupon_group": {
		TypeSingleItemCoupon,
		TypeStoreCoupon,
		TypePlatformCoupon,
	},
}

// CanCombine 检查两个促销是否可以叠加
func CanCombine(a, b PromotionType) bool {
	if a == b {
		return false // 同类型不叠加
	}
	// 检查是否在同一个互斥组
	for _, group := range MutuallyExclusiveGroups {
		aInGroup := false
		bInGroup := false
		for _, t := range group {
			if t == a {
				aInGroup = true
			}
			if t == b {
				bInGroup = true
			}
		}
		if aInGroup && bInGroup {
			return false // 同组互斥
		}
	}
	return true
}

// FilterCompatible 从候选促销列表中筛选出与已选促销兼容的项
func FilterCompatible(
	selected []PromotionRule,
	candidates []PromotionRule,
) []PromotionRule {
	var result []PromotionRule
	for _, c := range candidates {
		compatible := true
		for _, s := range selected {
			if !CanCombine(s.Type, c.Type) {
				compatible = false
				break
			}
		}
		if compatible {
			result = append(result, c)
		}
	}
	return result
}
```

### 3. 促销计算引擎

```go
package promotion

import (
	"context"
	"fmt"
	"sort"
	"time"

	"github.com/shopspring/decimal"
	"github.com/zeromicro/go-zero/core/logx"
	"gorm.io/gorm"
	"gorm.io/gorm/clause"
)

// CartItem 购物车商品项
type CartItem struct {
	SKU       string
	StoreID   string
	Price     int64 // 原价（分）
	Quantity  int
}

// PromotionResult 单个商品行的优惠结果
type PromotionResult struct {
	RuleID       string
	RuleName     string
	RuleType     PromotionType
	OriginalPrice int64 // 原价（分）
	DiscountPrice int64 // 折后价（分）
	Discount      int64 // 优惠金额（分）
}

// OrderPromotionResult 订单级优惠汇总
type OrderPromotionResult struct {
	Items        []PromotionResult // 每个商品行的优惠明细
	TotalOriginal int64            // 总原价
	TotalDiscount int64            // 总优惠
	TotalPayable  int64            // 应付金额
	AppliedRules  []string         // 已应用的规则 ID 列表
	AuditTrail    []AuditEntry     // 审计追踪
}

// AuditEntry 审计条目
type AuditEntry struct {
	Step       string    `json:"step"`
	RuleID     string    `json:"rule_id"`
	RuleType   string    `json:"rule_type"`
	BeforeAmt  int64     `json:"before_amt"`
	AfterAmt   int64     `json:"after_amt"`
	Discount   int64     `json:"discount"`
	Timestamp  time.Time `json:"timestamp"`
}

// Calculator 促销计算器
type Calculator struct {
	db *gorm.DB
}

// NewCalculator 创建计算器
func NewCalculator(db *gorm.DB) *Calculator {
	return &Calculator{db: db}
}

// Calculate 计算订单优惠（仅用于试算预览，实际预算扣减须在订单创建事务中通过 ConsumeBudget 执行）
func (c *Calculator) Calculate(
	ctx context.Context,
	tenantID string,
	items []CartItem,
	selectedCouponIDs []string, // 用户选择的优惠券 ID
) (*OrderPromotionResult, error) {
	ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
	defer cancel()

	now := time.Now()
	result := &OrderPromotionResult{
		Items: make([]PromotionResult, len(items)),
	}

	// 初始化：每个商品的原始金额
	var totalOriginal int64
	for i, item := range items {
		amt := item.Price * int64(item.Quantity)
		totalOriginal += amt
		result.Items[i] = PromotionResult{
			OriginalPrice: item.Price,
			DiscountPrice: item.Price, // 初始折后价 = 原价
		}
	}
	result.TotalOriginal = totalOriginal

	// 步骤 1: 加载并应用限时折扣（C1，最先计算）
	discountRules, err := c.loadActiveRules(ctx, tenantID, TypeFlashDiscount, now)
	if err != nil {
		return nil, err
	}
	c.applyFlashDiscount(result, items, discountRules, now)

	// 步骤 2: 加载并应用用户选择的优惠券（过滤已使用和已过期）
	var selectedCoupons []Coupon
	if len(selectedCouponIDs) > 0 {
		if err := c.db.WithContext(ctx).
			Where("tenant_id = ? AND id IN ? AND status = ?", tenantID, selectedCouponIDs, CouponUnused).
			Joins("JOIN promotion_rules pr ON pr.tenant_id = coupons.tenant_id AND pr.id = coupons.rule_id").
			Where("pr.tenant_id = ? AND pr.end_time > ? AND pr.status = ?", tenantID, now, StatusEnabled).
			Find(&selectedCoupons).Error; err != nil {
			return nil, fmt.Errorf("promotion: failed to load coupons: %w", err)
		}
	}

	// 加载券对应的规则（批量加载，避免 N+1）
	ruleIDs := make([]string, 0, len(selectedCoupons))
	for _, coupon := range selectedCoupons {
		ruleIDs = append(ruleIDs, coupon.RuleID)
	}
	var ruleMap map[string]*PromotionRule
	if len(ruleIDs) > 0 {
		var err error
		ruleMap, err = c.loadRulesBatch(ctx, tenantID, ruleIDs)
		if err != nil {
			return nil, fmt.Errorf("promotion: failed to load rules: %w", err)
		}
	}

	// 按优先级排序：单品券 > 店铺券 > 平台券
	sortedCouponIDs := sortCouponsByPriority(selectedCoupons, ruleMap)

	// 步骤 3: 应用单品券
	c.applyCoupons(result, items, sortedCouponIDs, ruleMap, TypeSingleItemCoupon, now)

	// 步骤 4: 应用店铺券/平台券（二选一，取优惠金额最大的）
	c.applyBestStoreOrPlatformCoupon(result, items, sortedCouponIDs, ruleMap, now)

	// 步骤 5: 应用满减
	fullReductionRules, err := c.loadActiveRules(ctx, tenantID, TypeFullReduction, now)
	if err != nil {
		return nil, err
	}
	c.applyFullReduction(result, items, fullReductionRules, now)

	// 计算最终应付金额
	result.calculateTotals()

	return result, nil
}

// loadActiveRules 加载某类型的有效促销规则
func (c *Calculator) loadActiveRules(
	ctx context.Context,
	tenantID string,
	pType PromotionType,
	now time.Time,
) ([]PromotionRule, error) {
	ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	var rules []PromotionRule
	err := c.db.WithContext(ctx).
		Where(
			"tenant_id = ? AND type = ? AND status = ? AND start_time <= ? AND end_time >= ?",
			tenantID, int(pType), StatusEnabled, now, now,
		).
		Find(&rules).Error
	return rules, err
}

// loadRule 加载单个规则（带租户隔离）
func (c *Calculator) loadRule(ctx context.Context, tenantID, ruleID string) (*PromotionRule, error) {
	ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	var rule PromotionRule
	err := c.db.WithContext(ctx).
		Where("tenant_id = ? AND id = ?", tenantID, ruleID).
		First(&rule).Error
	if err != nil {
		return nil, err
	}
	return &rule, nil
}

// loadRulesBatch 批量加载规则（避免 N+1 查询）
func (c *Calculator) loadRulesBatch(ctx context.Context, tenantID string, ruleIDs []string) (map[string]*PromotionRule, error) {
	ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	var rules []PromotionRule
	if err := c.db.WithContext(ctx).
		Where("tenant_id = ? AND id IN ?", tenantID, ruleIDs).
		Find(&rules).Error; err != nil {
		return nil, err
	}
	ruleMap := make(map[string]*PromotionRule, len(rules))
	for i := range rules {
		ruleMap[rules[i].ID] = &rules[i]
	}
	return ruleMap, nil
}

// applyFlashDiscount 应用限时折扣（P5）
func (c *Calculator) applyFlashDiscount(
	result *OrderPromotionResult,
	items []CartItem,
	rules []PromotionRule,
	now time.Time,
) {
	ruleBySKU := make(map[string]*PromotionRule)
	for i := range rules {
		r := &rules[i]
		ruleBySKU[r.TargetSKU] = r
	}

	for i, item := range items {
		rule, ok := ruleBySKU[item.SKU]
		if !ok {
			continue
		}
		// 使用 decimal 精确计算：price * rate / 100
		price := decimal.NewFromInt(item.Price)
		rate := decimal.NewFromInt(rule.DiscountRate)
		hundred := decimal.NewFromInt(100)
		newPriceDecimal := price.Mul(rate).Div(hundred).Round(0)
		newPrice := newPriceDecimal.IntPart()
		if newPrice < 1 {
			newPrice = 1 // 保底：单价至少 1 分
		}

		quantity := decimal.NewFromInt(int64(item.Quantity))
		totalBefore := price.Mul(quantity).IntPart()
		totalAfter := newPriceDecimal.Mul(quantity).IntPart()
		discount := totalBefore - totalAfter

		if discount > 0 {
			if rule.MaxDiscount > 0 && discount > rule.MaxDiscount {
				discount = rule.MaxDiscount
				totalAfter = totalBefore - discount
				newPrice = decimal.NewFromInt(totalAfter).Div(quantity).Round(0).IntPart()
			}

			result.Items[i].DiscountPrice = newPrice
			result.Items[i].RuleID = rule.ID
			result.Items[i].RuleName = rule.Name
			result.Items[i].RuleType = rule.Type
			result.Items[i].Discount = discount

			result.AuditTrail = append(result.AuditTrail, AuditEntry{
				Step:      "flash_discount",
				RuleID:    rule.ID,
				RuleType:  rule.Type.String(),
				BeforeAmt: totalBefore,
				AfterAmt:  totalAfter,
				Discount:  discount,
				Timestamp: now,
			})
		}
	}
}

// applyCoupons 应用指定类型的优惠券
func (c *Calculator) applyCoupons(
	result *OrderPromotionResult,
	items []CartItem,
	sortedIDs []string,
	ruleMap map[string]*PromotionRule,
	filterType PromotionType,
	now time.Time,
) {
	for _, couponID := range sortedIDs {
		rule, ok := ruleMap[couponID]
		if !ok || rule.Type != filterType {
			continue
		}

		// 检查预算
		if rule.UsedBudget+rule.DiscountAmount >= rule.TotalBudget {
			continue // 预算不足，跳过
		}

		// 找到匹配的商品
		for i, item := range items {
			if rule.TargetSKU != "" && item.SKU != rule.TargetSKU {
				continue
			}
			if rule.TargetStoreID != "" && item.StoreID != rule.TargetStoreID {
				continue
			}

			discount := rule.DiscountAmount * int64(item.Quantity)
			currentTotal := result.Items[i].DiscountPrice * int64(item.Quantity)
			if discount > currentTotal {
				discount = currentTotal // 优惠不超过当前金额
			}

			// 使用 decimal 精确分摊优惠到单价
			discountDecimal := decimal.NewFromInt(discount)
			quantityDecimal := decimal.NewFromInt(int64(item.Quantity))
			currentPriceDecimal := decimal.NewFromInt(result.Items[i].DiscountPrice)
			newPrice := currentPriceDecimal.Sub(discountDecimal.Div(quantityDecimal).Round(0)).IntPart()

			result.Items[i].DiscountPrice = newPrice
			result.Items[i].Discount += discount
			result.Items[i].RuleID = rule.ID
			result.Items[i].RuleName = rule.Name
			result.Items[i].RuleType = rule.Type

			result.AuditTrail = append(result.AuditTrail, AuditEntry{
				Step:      "coupon_apply",
				RuleID:    rule.ID,
				RuleType:  rule.Type.String(),
				BeforeAmt: currentTotal,
				AfterAmt:  newPrice * int64(item.Quantity),
				Discount:  discount,
				Timestamp: now,
			})
			break // 每张券只应用一次
		}
	}
}

// applyBestStoreOrPlatformCoupon 店铺券和平台券二选一，取优惠最大的
func (c *Calculator) applyBestStoreOrPlatformCoupon(
	result *OrderPromotionResult,
	items []CartItem,
	sortedIDs []string,
	ruleMap map[string]*PromotionRule,
	now time.Time,
) {
	var bestCouponID string
	var bestDiscount int64

	for _, couponID := range sortedIDs {
		rule, ok := ruleMap[couponID]
		if !ok {
			continue
		}
		if rule.Type != TypeStoreCoupon && rule.Type != TypePlatformCoupon {
			continue
		}

		// 计算该券可产生的优惠
		discount := c.calculateCouponDiscount(result, items, rule)
		if discount > bestDiscount {
			bestDiscount = discount
			bestCouponID = couponID
		}
	}

	if bestCouponID != "" && bestDiscount > 0 {
		rule := ruleMap[bestCouponID]
		// 应用最佳券（复用 applyCoupons，传入单张券）
		c.applyCoupons(result, items, []string{bestCouponID}, ruleMap, rule.Type, now)
	}
}

// calculateCouponDiscount 计算某张券的预计优惠金额
func (c *Calculator) calculateCouponDiscount(
	result *OrderPromotionResult,
	items []CartItem,
	rule *PromotionRule,
) int64 {
	var total int64
	for i, item := range items {
		if rule.TargetSKU != "" && item.SKU != rule.TargetSKU {
			continue
		}
		if rule.TargetStoreID != "" && item.StoreID != rule.TargetStoreID {
			continue
		}
		currentTotal := result.Items[i].DiscountPrice * int64(item.Quantity)
		d := rule.DiscountAmount * int64(item.Quantity)
		if d > currentTotal {
			d = currentTotal
		}
		total += d
	}
	return total
}

// applyFullReduction 应用满减（C4，最后计算）
func (c *Calculator) applyFullReduction(
	result *OrderPromotionResult,
	items []CartItem,
	rules []PromotionRule,
	now time.Time,
) {
	// 找到最优的满减规则（优惠金额最大）
	var bestRule *PromotionRule
	var bestDiscount int64
	var bestEligibleTotal int64

	for i := range rules {
		rule := &rules[i]
		if rule.UsedBudget+rule.DiscountAmount >= rule.TotalBudget {
			continue // 预算不足
		}

		// 计算符合条件的商品总额（result.Items 与 items 是平行切片，直接用索引）
		var eligibleTotal int64
		for i, item := range items {
			if rule.TargetStoreID != "" && item.StoreID != rule.TargetStoreID {
				continue
			}
			eligibleTotal += result.Items[i].DiscountPrice * int64(item.Quantity)
		}

		if eligibleTotal < rule.ThresholdAmount {
			continue // 未达到门槛
		}

		discount := rule.DiscountAmount
		if rule.MaxDiscount > 0 && discount > rule.MaxDiscount {
			discount = rule.MaxDiscount
		}
		if discount > eligibleTotal {
			discount = eligibleTotal
		}

		if discount > bestDiscount {
			bestDiscount = discount
			bestRule = rule
			bestEligibleTotal = eligibleTotal
		}
	}

	// 应用最优满减规则
	if bestRule == nil || bestDiscount <= 0 {
		return
	}

	rule := bestRule
	discount := bestDiscount

	// 使用 decimal 按比例精确分摊到每个商品
	discountDecimal := decimal.NewFromInt(discount)
	eligibleTotalDecimal := decimal.NewFromInt(bestEligibleTotal)
	var totalAllocated int64
	eligibleIndices := make([]int, 0)

	for i, item := range items {
		if rule.TargetStoreID != "" && item.StoreID != rule.TargetStoreID {
			continue
		}
		eligibleIndices = append(eligibleIndices, i)
		itemTotal := result.Items[i].DiscountPrice * int64(item.Quantity)
		itemTotalDecimal := decimal.NewFromInt(itemTotal)

		// 按比例精确计算，ROUND_HALF_UP
		proportionalDiscount := discountDecimal.Mul(itemTotalDecimal).Div(eligibleTotalDecimal).Round(0).IntPart()
		if proportionalDiscount > 0 {
			quantity := int64(item.Quantity)
			unitDiscount := decimal.NewFromInt(proportionalDiscount).Div(decimal.NewFromInt(quantity)).Round(0).IntPart()
			newPrice := result.Items[i].DiscountPrice - unitDiscount
			if newPrice < 1 {
				newPrice = 1
			}
			result.Items[i].DiscountPrice = newPrice
			result.Items[i].Discount += proportionalDiscount
			totalAllocated += proportionalDiscount
		}
	}

	// 处理舍入差异：最后一个符合条件的商品吸收差额
	remainder := discount - totalAllocated
	if remainder != 0 && len(eligibleIndices) > 0 {
		lastIdx := eligibleIndices[len(eligibleIndices)-1]
		result.Items[lastIdx].Discount += remainder
		itemTotal := result.Items[lastIdx].DiscountPrice * int64(items[lastIdx].Quantity)
		newTotal := itemTotal - remainder
		if newTotal < int64(items[lastIdx].Quantity) {
			newTotal = int64(items[lastIdx].Quantity) // 保底：每件商品至少 1 分
		}
		result.Items[lastIdx].DiscountPrice = newTotal / int64(items[lastIdx].Quantity)
	}

	result.AuditTrail = append(result.AuditTrail, AuditEntry{
		Step:      "full_reduction",
		RuleID:    rule.ID,
		RuleType:  rule.Type.String(),
		BeforeAmt: bestEligibleTotal,
		AfterAmt:  bestEligibleTotal - discount,
		Discount:  discount,
		Timestamp: now,
	})
}

// sortCouponsByPriority 按优先级排序优惠券
func sortCouponsByPriority(
	coupons []Coupon,
	ruleMap map[string]*PromotionRule,
) []string {
	type couponPriority struct {
		ID       string
		Priority int
	}
	var cps []couponPriority
	for _, c := range coupons {
		if rule, ok := ruleMap[c.ID]; ok {
			cps = append(cps, couponPriority{ID: c.ID, Priority: rule.Type.Priority()})
		}
	}
	sort.Slice(cps, func(i, j int) bool {
		return cps[i].Priority < cps[j].Priority
	})
	ids := make([]string, len(cps))
	for i, cp := range cps {
		ids[i] = cp.ID
	}
	return ids
}

// calculateTotals 计算最终汇总
func (r *OrderPromotionResult) calculateTotals() {
	var totalDiscount int64
	seen := make(map[string]struct{})
	for _, item := range r.Items {
		totalDiscount += item.Discount
		if item.RuleID != "" {
			if _, ok := seen[item.RuleID]; !ok {
				seen[item.RuleID] = struct{}{}
				r.AppliedRules = append(r.AppliedRules, item.RuleID)
			}
		}
	}
	r.TotalDiscount = totalDiscount
	r.TotalPayable = r.TotalOriginal - totalDiscount
	if r.TotalPayable < 0 {
		r.TotalPayable = 0
	}
}
```

### 4. 预算扣减与审计记录

```go
package promotion

import (
	"context"
	"fmt"
	"time"

	"github.com/zeromicro/go-zero/core/logx"
	"gorm.io/gorm"
)

// PromotionAuditLog 促销使用审计日志（只读追加表）
type PromotionAuditLog struct {
	ID        int64     `gorm:"primaryKey;autoIncrement"`
	TenantID  string    `gorm:"type:varchar(64);not null;index"` // 多租户隔离
	OrderID   string    `gorm:"type:varchar(64);not null;index"`
	UserID    string    `gorm:"type:varchar(64);not null;index"` // 加密存储，展示时脱敏
	RuleID    string    `gorm:"type:varchar(64);not null;index"`
	RuleType  PromotionType `gorm:"not null"`
	CouponID  string    `gorm:"type:varchar(64)"`
	Discount  int64     `gorm:"not null"` // 优惠金额（分）
	CreatedAt time.Time `gorm:"not null;autoCreateTime"`
}

func (PromotionAuditLog) TableName() string {
	return "promotion_audit_logs"
}

// BudgetManager 促销预算管理器
type BudgetManager struct {
	db *gorm.DB
}

// NewBudgetManager 创建预算管理器
func NewBudgetManager(db *gorm.DB) *BudgetManager {
	return &BudgetManager{db: db}
}

// ConsumeBudget 消耗预算（带行锁，防止超扣）
func (bm *BudgetManager) ConsumeBudget(
	ctx context.Context,
	tenantID string,
	ruleID string,
	amount int64,
) error {
	return bm.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
		var rule PromotionRule
		if err := tx.Clauses(clause.Locking{Strength: "UPDATE"}).
			Where("tenant_id = ? AND id = ?", tenantID, ruleID).
			First(&rule).Error; err != nil {
			return fmt.Errorf("promotion: failed to load rule: %w", err)
		}

		if rule.UsedBudget+amount > rule.TotalBudget {
			return ErrPromotionBudgetExceeded
		}

		result := tx.Model(&PromotionRule{}).
			Where("tenant_id = ? AND id = ? AND used_budget + ? <= total_budget", tenantID, ruleID, amount).
			UpdateColumn("used_budget", gorm.Expr("used_budget + ?", amount))
		if result.Error != nil {
			return result.Error
		}
		if result.RowsAffected == 0 {
			return ErrPromotionBudgetExceeded
		}

		return nil
	})
}

// RecordPromotionUsage 记录促销使用（订单创建后调用）
func (bm *BudgetManager) RecordPromotionUsage(
	ctx context.Context,
	tenantID string,
	orderID string,
	userID string,
	result *OrderPromotionResult,
	coupons []Coupon,
) error {
	return bm.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
		// 记录审计日志
		for _, item := range result.Items {
			if item.RuleID == "" || item.Discount == 0 {
				continue
			}
			log := PromotionAuditLog{
				TenantID: tenantID,
				OrderID:  orderID,
				UserID:   userID,
				RuleID:   item.RuleID,
				RuleType: item.RuleType,
				Discount: item.Discount,
			}
			// 查找对应的 coupon ID
			for _, c := range coupons {
				if c.RuleID == item.RuleID {
					log.CouponID = c.ID
					break
				}
			}
			if err := tx.Create(&log).Error; err != nil {
				logx.WithContext(ctx).Errorw("promotion: failed to write audit log",
					logx.Field("order_id", orderID),
					logx.Field("rule_id", item.RuleID),
					logx.Field("error", err),
				)
				return fmt.Errorf("promotion: failed to write audit log: %w", err)
			}
		}

		// 标记优惠券为已使用
		for _, c := range coupons {
			result := tx.Model(&Coupon{}).
				Where("tenant_id = ? AND id = ? AND status = ?", tenantID, c.ID, CouponUnused).
				Updates(map[string]any{
					"status":       CouponUsed,
					"used_order_id": orderID,
					"used_at":      time.Now(),
				})
			if result.Error != nil {
				logx.WithContext(ctx).Errorw("promotion: failed to update coupon",
					logx.Field("coupon_id", c.ID),
					logx.Field("error", result.Error),
				)
				return fmt.Errorf("promotion: failed to update coupon %s: %w", c.ID, result.Error)
			}
		}

		return nil
	})
}
```

### 5. 使用示例

```go
package main

import (
	"context"
	"fmt"
	"log"
	"storeforge/promotion"
	"storeforge/db"
)

func main() {
	database := db.GetDB()
	calc := promotion.NewCalculator(database)
	bm := promotion.NewBudgetManager(database)

	ctx := context.Background()
	tenantID := "TENANT001"

	// 构建购物车
	items := []promotion.CartItem{
		{SKU: "SKU001", StoreID: "STORE01", Price: 9900, Quantity: 2},  // 99 元
		{SKU: "SKU002", StoreID: "STORE01", Price: 5900, Quantity: 1},  // 59 元
		{SKU: "SKU003", StoreID: "STORE02", Price: 12900, Quantity: 1}, // 129 元
	}

	// 用户选择的优惠券
	couponIDs := []string{"COUPON001", "COUPON002"}

	// 计算优惠
	result, err := calc.Calculate(ctx, tenantID, items, couponIDs)
	if err != nil {
		log.Fatal(err)
	}

	fmt.Printf("原价: %d 分\n", result.TotalOriginal)
	fmt.Printf("优惠: %d 分\n", result.TotalDiscount)
	fmt.Printf("应付: %d 分\n", result.TotalPayable)
	fmt.Printf("使用规则: %v\n", result.AppliedRules)

	// 打印审计追踪
	for _, entry := range result.AuditTrail {
		fmt.Printf("[%s] %s: %d → %d (优惠 %d 分)\n",
			entry.RuleType, entry.RuleID,
			entry.BeforeAmt, entry.AfterAmt, entry.Discount,
		)
	}

	// 创建订单后，消耗预算并记录使用
	// bm.ConsumeBudget(ctx, ruleID, discountAmount)
	// bm.RecordPromotionUsage(ctx, orderID, userID, result, coupons)
}
```

## 反模式

### 1. 叠加互斥优惠券

```go
// 错误：同时应用店铺券和平台券
total -= storeCoupon.DiscountAmount
total -= platformCoupon.DiscountAmount // 违反了互斥规则！
```

**问题**：同一用户同时使用多张互斥优惠券，导致商家亏损。

**正确做法**：使用 `CanCombine()` 检查兼容性，店铺券和平台券二选一取最优。

### 2. 无预算限制

```go
// 错误：不检查预算直接发券
func IssueCoupon(userID, ruleID string) error {
	coupon := Coupon{UserID: userID, RuleID: ruleID}
	return db.Create(&coupon).Error
}
```

**问题**：超发优惠券，预算用尽后仍可以领取，财务无法控制成本。

**正确做法**：发券和消费时都必须检查 `UsedBudget + amount <= TotalBudget`，使用行锁防止并发超扣。

### 3. 无审计追踪

```go
// 错误：优惠计算完不记录任何日志
func Checkout(cart Cart) (int64, error) {
    total := cart.Total
    total -= applyCoupons(cart)
    total -= applyFullReduction(cart)
    return total, nil // 没有任何记录：用了什么券？优惠了多少？
}
```

**问题**：财务对账无法核对优惠金额，用户投诉时无据可查。

**正确做法**：每步优惠计算都记录 `AuditEntry`，包含规则 ID、类型、前后金额、时间戳。

### 4. 优惠金额超过商品总价

```go
// 错误：不限制优惠上限
discount := coupon.Amount * quantity
total -= discount // discount 可能 > total
```

**问题**：大额优惠券可能导致应付金额为负。

**正确做法**：`discount = min(discount, currentTotal)`，最终 `TotalPayable` 最低为 0。

### 5. 计算顺序错误

```go
// 错误：先算满减再算折扣
total = applyFullReduction(total) // 基于原价计算满减
total = applyDiscount(total)       // 再打折
```

**问题**：正确顺序应该是先打折再满减，否则用户可能无法达到满减门槛或优惠金额计算不准确。

## 常见坑

### 1. 并发超扣预算

**坑**：多个用户同时使用同一促销规则，`used_budget` 被并发更新导致超过 `total_budget`。

**解决**：
- 使用 `SELECT ... FOR UPDATE` 行锁（`BudgetManager.ConsumeBudget` 中已实现）
- 使用原子更新 `UPDATE ... SET used_budget = used_budget + ? WHERE used_budget + ? <= total_budget`
- 高并发场景建议预算扣减走 Redis Lua 脚本：

```lua
-- Redis Lua 脚本原子扣减预算
-- KEYS[1] = promotion:budget:{tenantID}:{ruleID}
-- ARGV[1] = amount
-- ARGV[2] = total_budget
local used = tonumber(redis.call('GET', KEYS[1]) or '0')
local amount = tonumber(ARGV[1])
local total = tonumber(ARGV[2])
if used + amount > total then
    return -1  -- 预算不足
end
redis.call('INCRBY', KEYS[1], amount)
return used + amount
```

完整 Go 实现（Redis Lua 版本）：

```go
package promotion

import (
	"context"
	"fmt"

	"github.com/redis/go-redis/v9"
)

// RedisBudgetManager 基于 Redis Lua 脚本的预算管理器
type RedisBudgetManager struct {
	rdb *redis.Client
}

// NewRedisBudgetManager 创建 Redis 预算管理器
func NewRedisBudgetManager(rdb *redis.Client) *RedisBudgetManager {
	return &RedisBudgetManager{rdb: rdb}
}

const budgetLuaScript = `
local used = tonumber(redis.call('GET', KEYS[1]) or '0')
local amount = tonumber(ARGV[1])
local total = tonumber(ARGV[2])
if used + amount > total then
    return -1
end
redis.call('INCRBY', KEYS[1], amount)
return used + amount
`

// ConsumeBudget 消耗预算（Redis 原子操作，适用于高并发场景）
func (bm *RedisBudgetManager) ConsumeBudget(
	ctx context.Context,
	tenantID string,
	ruleID string,
	amount int64,
	totalBudget int64,
) error {
	key := fmt.Sprintf("promotion:budget:%s:%s", tenantID, ruleID)
	result, err := bm.rdb.Eval(ctx, budgetLuaScript, []string{key}, amount, totalBudget).Int64()
	if err != nil {
		return fmt.Errorf("promotion: redis budget consume: %w", err)
	}
	if result == -1 {
		return ErrPromotionBudgetExceeded
	}
	return nil
}
```

### 2. 满减按比例分摊的精度损失

**坑**：满减金额按比例分摊到多个商品时，除不尽导致总和与优惠金额差 1 分钱。

**解决**：最后一个商品使用减法兜底：

```go
// 前 N-1 个商品正常按比例分摊
for i := 0; i < len(items)-1; i++ {
    proportionalDiscount := totalDiscount * itemTotal / eligibleTotal
    itemDiscounts[i] = proportionalDiscount
    remaining -= proportionalDiscount
}
// 最后一个商品 = 总优惠 - 已分摊
itemDiscounts[len(items)-1] = remaining
```

### 3. 优惠券过期时间判断

**坑**：只在领券时检查过期时间，使用时不检查，导致过期券仍可使用。

**解决**：优惠计算时同时检查券的状态和过期时间：

```go
// Calculate 中加载优惠券时过滤
db.Where("id IN ? AND status = 0", couponIDs).
    Joins("JOIN promotion_rules pr ON pr.id = coupons.rule_id").
    Where("pr.end_time > ?", time.Now()).
    Find(&coupons)
```

### 4. 拼团活动库存竞争

**坑**：拼团人数未满时预占库存，拼团失败后库存未及时释放。

**解决**：
- 拼团创建时预占库存（Redis）
- 拼团超时未完成自动释放库存
- 拼团成功后将预占转为实际扣减
- 释放和扣减都必须幂等

### 5. 促销规则的时间边界

**坑**：`start_time <= now <= end_time` 判断在边界时刻可能出现毫秒级误差。

**解决**：
- 数据库查询使用 `start_time <= ? AND end_time >= ?`
- 前端展示提前 1 分钟预告即将开始的活动
- 活动结束后延迟 5 分钟清理（给正在下单的用户缓冲）

### 6. 负数优惠金额

**坑**：折扣率配置错误（如 0 或负数）导致价格为 0 或负数。

**解决**：
- 创建/更新促销规则时校验 `discount_rate` 范围：`1 <= rate <= 100`
- 计算时兜底：`newPrice = max(1, newPrice)`，确保价格至少为 1 分
- 管理后台配置促销规则时增加范围校验

## 测试策略

> 以下为骨架示例，实际项目需根据具体实现补充 helper 函数和 mock。

### 1. 促销计算测试示例

```go
package promotion_test

import (
	"context"
	"sync"
	"sync/atomic"
	"testing"

	"storeforge/promotion"

	"github.com/stretchr/testify/assert"
)

// newTestCalculator: 使用内存 SQLite 或 mock DB
// func newTestCalculator(t *testing.T) *promotion.Calculator { ... }

func TestCalculate_CouponStacking(t *testing.T) {
	// 测试：单品券 + 满减可叠加，店铺券 + 平台券互斥
	calc := newTestCalculator(t)
	ctx := context.Background()
	tenantID := "TENANT001"

	// setupTestData 需在 helper 中预置促销规则和优惠券
	setupTestData(t, calc, tenantID)

	items := []promotion.CartItem{
		{SKU: "SKU001", StoreID: "STORE01", Price: 10000, Quantity: 1},
	}

	// 验证：单品券 + 满减可叠加
	result, err := calc.Calculate(ctx, tenantID, items, []string{"SINGLE_COUPON"})
	assert.NoError(t, err)
	assert.True(t, len(result.AppliedRules) >= 1)
}

func TestCalculate_MutualExclusion(t *testing.T) {
	// 测试：店铺券和平台券互斥，只取优惠金额大的
	calc := newTestCalculator(t)
	ctx := context.Background()
	tenantID := "TENANT001"
	setupTestData(t, calc, tenantID)

	result, err := calc.Calculate(ctx, tenantID, []promotion.CartItem{
		{SKU: "SKU001", StoreID: "STORE01", Price: 10000, Quantity: 1},
	}, []string{"STORE_COUPON", "PLATFORM_COUPON"})

	assert.NoError(t, err)
	// 验证只应用了一张券（优惠金额大的）
	storeApplied := false
	platformApplied := false
	for _, ruleID := range result.AppliedRules {
		if ruleID == "STORE_RULE" {
			storeApplied = true
		}
		if ruleID == "PLATFORM_RULE" {
			platformApplied = true
		}
	}
	assert.False(t, storeApplied && platformApplied, "store and platform coupons should be mutually exclusive")
}
```

### 2. 预算并发测试

```go
func TestBudgetManager_ConcurrentConsume(t *testing.T) {
	// 测试：并发扣减预算不超限
	db := setupTestDB(t)
	bm := NewBudgetManager(db)
	ctx := context.Background()
	tenantID := "TENANT001"

	// 创建总预算 1000 分的规则
	createTestRule(db, tenantID, "RULE001", 1000)

	var wg sync.WaitGroup
	successCount := atomic.Int64{}
	for i := 0; i < 20; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			err := bm.ConsumeBudget(ctx, tenantID, "RULE001", 100)
			if err == nil {
				successCount.Add(1)
			}
		}()
	}
	wg.Wait()

	// 最多 10 次成功（1000/100）
	assert.LessOrEqual(t, successCount.Load(), int64(10))
}
```

### 3. 覆盖率目标

| 层级 | 目标 | 说明 |
|------|------|------|
| L1 单元测试 | >= 80% | 促销计算、互斥规则、预算扣减 |
| L2 集成测试 | >= 70% | DB 交互、Redis 预算 Lua 脚本（miniredis） |
| L3 E2E 测试 | 关键路径 | 下单全流程 |

#### 多租户隔离测试

```go
func TestPromotion_MultiTenantIsolation(t *testing.T) {
	// tenantA 的促销规则对 tenantB 不可见
	calc := newTestCalculator(t)
	ctx := context.Background()

	// tenantA 创建专属规则
	setupTenantData(t, calc, "TENANT_A")

	// tenantB 使用 tenantA 的优惠券应失败
	result, err := calc.Calculate(ctx, "TENANT_B", []promotion.CartItem{
		{SKU: "SKU001", StoreID: "STORE01", Price: 10000, Quantity: 1},
	}, []string{"TENANT_A_COUPON"})
	assert.NoError(t, err)
	assert.Zero(t, len(result.AppliedRules), "tenantB should not see tenantA's coupon")
}
```

#### Lua 脚本预算扣减测试

```go
func TestBudgetManager_ConsumeBudget_Lua(t *testing.T) {
	mr := miniredis.RunT(t)
	rdb := redis.NewClient(&redis.Options{Addr: mr.Addr()})
	ctx := context.Background()

	bm := NewRedisBudgetManager(rdb)
	key := "promotion:budget:TENANT001:RULE001"
	rdb.Set(ctx, key, 0, 0) // 初始化预算为 0
	totalBudget := int64(1000)

	// 并发扣减 20 次，每次 100 分
	var success int
	for i := 0; i < 20; i++ {
		err := bm.ConsumeBudget(ctx, "TENANT001", "RULE001", 100)
		if err == nil {
			success++
		}
	}
	if success != 10 { // 最多 10 次（1000/100）
		t.Errorf("success=%d, want 10", success)
	}
}
```

#### 边界条件测试

```go
func TestCanCombine_MutualExclusive(t *testing.T) {
	assert.False(t, promotion.CanCombine(promotion.TypeStoreCoupon, promotion.TypePlatformCoupon))
	assert.True(t, promotion.CanCombine(promotion.TypeSingleItemCoupon, promotion.TypeFullReduction))
	assert.True(t, promotion.CanCombine(promotion.TypeFlashDiscount, promotion.TypeFullReduction))
	assert.False(t, promotion.CanCombine(promotion.TypeSingleItemCoupon, promotion.TypeSingleItemCoupon))
}
```

#### 预算 TOCTOU 竞态说明

```go
// Calculate 中的预算检查（rule.UsedBudget）是试算快照，
// 基于 Calculate 开始时刻的内存数据，与 ConsumeBudget 之间存在 TOCTOU 窗口。
// 正确流程：
//   1. Calculate() — 试算预览，告知用户优惠金额（预算数据可能已过期）
//   2. 用户确认下单
//   3. ConsumeBudget() — 带行锁/Lua 脚本的原子扣减（唯一可信的预算扣减点）
//   4. RecordPromotionUsage() — 记录审计日志 + 更新券状态
// 如果步骤 3 返回 ErrPromotionBudgetExceeded，订单创建失败，返回用户"优惠券已抢光"。
```
