# 数据分析与 BI (Analytics & BI)

## 触发条件

- 需要销售报表、用户行为分析、漏斗分析、A/B 测试、经营看板
- 需要 GMV/订单量/转化率/AOV 等核心指标的统计与展示
- 需要按时间维度（日/周/月）或用户维度（新客/老客）进行数据聚合
- 需要导出报表为 Excel/CSV 格式
- 需要实时计数（如今日订单数、今日 GMV）

## 推荐第三方库

| 用途 | 推荐库 | 导入路径 |
|------|--------|----------|
| 时序数据库 | go-redis (TS) / clickhouse-go | `github.com/ClickHouse/clickhouse-go/v2` |
| 数据可视化对接 | 直接返回 JSON | 前端用 Grafana/Superset |
| OLAP 查询 | go-zero MapReduce | `github.com/zeromicro/go-zero/core/mr` |
| 并发控制 | errgroup | `golang.org/x/sync/errgroup` |
| 报表导出 | excelize | `github.com/xuri/excelize/v2` |

## 决策树

```
需要数据分析？
  ├─ 实时指标（今日订单数/GMV）→ Redis INCRBY 计数器 + TTL 自动重置
  ├─ 历史报表（销售趋势/用户留存）→ ClickHouse / PG 只读副本 + GROUP BY
  ├─ 漏斗分析（浏览→加购→下单→支付）→ 事件表 + 窗口函数
  ├─ 实时看板（WebSocket 推送）→ Redis Pub/Sub + 前端 SSE
  ├─ 导出报表（Excel/CSV）→ go-zero MapReduce 并发聚合 → excelize 流式写入
  └─ A/B 测试 → 实验分组 + 指标对比 + 统计显著性检验
```

## 核心指标定义

```
GMV (Gross Merchandise Volume) = 所有已支付订单金额之和（含运费，不含退款）
订单量 = 已支付状态的订单数量
转化率 = 支付订单数 / 访问用户数 × 100%
AOV (Average Order Value) = GMV / 订单量
客单价 = GMV / 支付用户数（去重）
复购率 = 周期内购买 2 次及以上的用户数 / 周期内总支付用户数
退款率 = 退款订单数 / 已支付订单数 × 100%
```

## 代码模板

### 1. 销售大盘指标 (Sales Dashboard Metrics)

```go
package analytics

import (
	"context"
	"fmt"
	"time"

	"github.com/zeromicro/go-zero/core/logx"
)

// contextKey is an unexported type to avoid context key collisions.
type contextKey string

const (
	ctxKeyTenantID contextKey = "tenantID"
)

// TenantIDFromCtx extracts tenantID from context.
func TenantIDFromCtx(ctx context.Context) (string, bool) {
	v, ok := ctx.Value(ctxKeyTenantID).(string)
	return v, ok
}

// CtxWithTenantID returns a new context with tenantID attached.
func CtxWithTenantID(ctx context.Context, tenantID string) context.Context {
	return context.WithValue(ctx, ctxKeyTenantID, tenantID)
}

// DashboardMetric 经营看板核心指标
type DashboardMetric struct {
	GMV            int64   `json:"gmv"`             // GMV（分）
	OrderCount     int64   `json:"order_count"`     // 订单量
	PaidUserCount  int64   `json:"paid_user_count"` // 支付用户数（去重）
	VisitorCount   int64   `json:"visitor_count"`   // 访问用户数
	ConversionRate float64 `json:"conversion_rate"` // 转化率（%）
	AOV            int64   `json:"aov"`             // 客单价（分）
	RefundRate     float64 `json:"refund_rate"`     // 退款率（%）
	RepeatBuyRate  float64 `json:"repeat_buy_rate"` // 复购率（%）
	Period         string  `json:"period"`          // 统计周期：daily/weekly/monthly
	StartTime      string  `json:"start_time"`
	EndTime        string  `json:"end_time"`
}

// DashboardRepo 看板数据仓库接口
type DashboardRepo interface {
	QueryGMV(ctx context.Context, tenantID string, start, end time.Time) (int64, error)
	QueryOrderCount(ctx context.Context, tenantID string, start, end time.Time) (int64, error)
	QueryPaidUserCount(ctx context.Context, tenantID string, start, end time.Time) (int64, error)
	QueryVisitorCount(ctx context.Context, tenantID string, start, end time.Time) (int64, error)
	QueryRefundCount(ctx context.Context, tenantID string, start, end time.Time) (int64, error)
	QueryRepeatBuyRate(ctx context.Context, tenantID string, start, end time.Time) (float64, error)
}

// DashboardService 看板服务
type DashboardService struct {
	repo DashboardRepo
}

// NewDashboardService 创建看板服务
func NewDashboardService(repo DashboardRepo) *DashboardService {
	return &DashboardService{repo: repo}
}

// ConvertPrice 将分转换为元，支持多货币
func ConvertPrice(cents int64, currency string) (float64, error) {
	switch currency {
	case "CNY", "USD", "EUR", "GBP":
		return float64(cents) / 100, nil
	default:
		return 0, fmt.Errorf("unsupported currency: %s", currency)
	}
}

// GetMetrics 获取指定时间段的核心指标
func (s *DashboardService) GetMetrics(
	ctx context.Context,
	tenantID string,
	start, end time.Time,
	period string,
) (*DashboardMetric, error) {
	logx.WithContext(ctx).Infof("calculating dashboard metrics: tenant=%s period=%s", tenantID, period)

	gmv, err := s.repo.QueryGMV(ctx, tenantID, start, end)
	if err != nil {
		return nil, err
	}
	orderCount, err := s.repo.QueryOrderCount(ctx, tenantID, start, end)
	if err != nil {
		return nil, err
	}
	paidUserCount, err := s.repo.QueryPaidUserCount(ctx, tenantID, start, end)
	if err != nil {
		return nil, err
	}
	visitorCount, err := s.repo.QueryVisitorCount(ctx, tenantID, start, end)
	if err != nil {
		return nil, err
	}
	refundCount, err := s.repo.QueryRefundCount(ctx, tenantID, start, end)
	if err != nil {
		return nil, err
	}
	repeatBuyRate, err := s.repo.QueryRepeatBuyRate(ctx, tenantID, start, end)
	if err != nil {
		return nil, err
	}

	metrics := &DashboardMetric{
		GMV:           gmv,
		OrderCount:    orderCount,
		PaidUserCount: paidUserCount,
		VisitorCount:  visitorCount,
		Period:        period,
		StartTime:     start.Format(time.RFC3339),
		EndTime:       end.Format(time.RFC3339),
		RepeatBuyRate: repeatBuyRate,
	}

	// 计算衍生指标
	// 注意：生产环境应从配置读取公式（见反模式 #4），此处为简化示例
	if visitorCount > 0 {
		metrics.ConversionRate = float64(orderCount) / float64(visitorCount) * 100
	}
	if orderCount > 0 {
		metrics.AOV = gmv / orderCount
	}
	if orderCount > 0 {
		metrics.RefundRate = float64(refundCount) / float64(orderCount) * 100
	}
	if paidUserCount > 0 {
		metrics.RepeatBuyRate = repeatBuyRate
	}

	return metrics, nil
}
```

### 2. 漏斗分析 (Funnel Analysis)

```go
package analytics

import (
	"context"
	"fmt"
	"time"

	"github.com/zeromicro/go-zero/core/logx"
)

// FunnelStep 漏斗步骤
type FunnelStep struct {
	StepName   string `json:"step_name"`   // 步骤名称：view / cart / checkout / pay / complete
	UserCount  int64  `json:"user_count"`  // 到达该步骤的用户数
	OrderCount int64  `json:"order_count"` // 到达该步骤的订单数（仅 checkout 之后）
	DropOff    int64  `json:"drop_off"`    // 流失用户数
	DropOffPct float64 `json:"drop_off_pct"` // 流失率（%）
	ConvRate   float64 `json:"conv_rate"`   // 转化率（相对上一步）
	TotalRate  float64 `json:"total_rate"`  // 总转化率（相对第一步）
}

// FunnelResult 漏斗分析结果
type FunnelResult struct {
	Steps     []FunnelStep `json:"steps"`
	StartTime string       `json:"start_time"`
	EndTime   string       `json:"end_time"`
}

// FunnelRepo 漏斗数据仓库接口
type FunnelRepo interface {
	// QueryStepUsers 查询到达某个步骤的独立用户数
	QueryStepUsers(ctx context.Context, tenantID, step string, start, end time.Time) (int64, error)
	// QueryStepOrders 查询到达某步骤的订单数（支付及以后）
	QueryStepOrders(ctx context.Context, tenantID, step string, start, end time.Time) (int64, error)
}

// 漏斗步骤定义（按顺序），使用私有变量 + 拷贝防止外部修改
var funnelSteps = []string{"view", "cart", "checkout", "pay", "complete"}

// FunnelSteps 返回漏斗步骤的只读副本
func FunnelSteps() []string {
	result := make([]string, len(funnelSteps))
	copy(result, funnelSteps)
	return result
}

// FunnelService 漏斗分析服务
type FunnelService struct {
	repo FunnelRepo
}

// NewFunnelService 创建漏斗分析服务
func NewFunnelService(repo FunnelRepo) *FunnelService {
	return &FunnelService{repo: repo}
}

// Analyze 执行漏斗分析
func (s *FunnelService) Analyze(
	ctx context.Context,
	tenantID string,
	start, end time.Time,
) (*FunnelResult, error) {
	logx.WithContext(ctx).Infof("funnel analysis: tenant=%s", tenantID)
	result := &FunnelResult{
		StartTime: start.Format(time.RFC3339),
		EndTime:   end.Format(time.RFC3339),
	}

	var firstUserCount int64

	for _, step := range FunnelSteps() {
		users, err := s.repo.QueryStepUsers(ctx, tenantID, step, start, end)
		if err != nil {
			return nil, err
		}

		fs := FunnelStep{
			StepName:  step,
			UserCount: users,
		}

		// 支付步骤之后统计订单数
		if step == "pay" || step == "complete" {
			orders, err := s.repo.QueryStepOrders(ctx, tenantID, step, start, end)
			if err != nil {
				return nil, err
			}
			fs.OrderCount = orders
		}

		// 计算流失（相对上一步）
		if len(result.Steps) > 0 {
			prev := result.Steps[len(result.Steps)-1]
			fs.DropOff = prev.UserCount - users
			if prev.UserCount > 0 {
				fs.DropOffPct = float64(fs.DropOff) / float64(prev.UserCount) * 100
				fs.ConvRate = float64(users) / float64(prev.UserCount) * 100
			}
		}

		// 计算总转化率（相对第一步）
		if firstUserCount == 0 {
			firstUserCount = users
		}
		if firstUserCount > 0 {
			fs.TotalRate = float64(users) / float64(firstUserCount) * 100
		}

		result.Steps = append(result.Steps, fs)
	}

	return result, nil
}

// SQL 参考（ClickHouse 物化视图表查询）
// 假设已有 event_log 表记录了用户行为事件：
//
// SELECT step, uniq(user_id) AS users
// FROM event_log
// WHERE event_date BETWEEN ? AND ?
//   AND step IN ('view','cart','checkout','pay','complete')
// GROUP BY step
// ORDER BY min(event_time)
//
// 漏斗数据也可以通过会话分析来计算：
//
// SELECT
//     countIf(step = 'view')         AS view_users,
//     countIf(step = 'cart')         AS cart_users,
//     countIf(step = 'checkout')     AS checkout_users,
//     countIf(step = 'pay')          AS pay_users,
//     countIf(step = 'complete')     AS complete_users
// FROM (
//     SELECT user_id, arrayJoin(['view','cart','checkout','pay','complete']) AS step
//     FROM event_log
//     WHERE event_date BETWEEN ? AND ?
//     GROUP BY user_id, step
// )
```

### 3. 用户留存分析 (User Cohort Analysis)

```go
package analytics

import (
	"context"
	"sort"
	"time"

	"github.com/zeromicro/go-zero/core/logx"
)

// CohortRow 一个留存 cohort 行
type CohortRow struct {
	CohortMonth   string  `json:"cohort_month"`  // 注册月份，如 2025-01
	TotalUsers    int64   `json:"total_users"`    // 该月注册用户总数
	Month0        float64 `json:"month_0"`        // 当月留存（通常 100%）
	Month1        float64 `json:"month_1"`        // 次月留存
	Month2        float64 `json:"month_2"`        // 第三个月留存
	Month3        float64 `json:"month_3"`
	Month6        float64 `json:"month_6"`
	Month12       float64 `json:"month_12"`
}

// CohortRepo 留存数据仓库接口
type CohortRepo interface {
	// QueryCohortData 查询某月注册用户在第 N 月的活跃用户数
	// 返回 []struct{CohortMonth string, ActiveMonth int, UserCount int64}
	QueryCohortData(ctx context.Context, tenantID, startMonth, endMonth string) (
		[]CohortDataPoint, error,
	)
}

// CohortDataPoint 留存数据点
type CohortDataPoint struct {
	CohortMonth string
	ActiveMonth int // 0 = 当月, 1 = 次月, ...
	UserCount   int64
}

// CohortService 留存分析服务
type CohortService struct {
	repo CohortRepo
}

// NewCohortService 创建留存分析服务
func NewCohortService(repo CohortRepo) *CohortService {
	return &CohortService{repo: repo}
}

// Analyze 生成留存矩阵
func (s *CohortService) Analyze(
	ctx context.Context,
	tenantID string,
	startMonth, endMonth string,
) ([]CohortRow, error) {
	logx.WithContext(ctx).Infof("cohort analysis: tenant=%s", tenantID)
	data, err := s.repo.QueryCohortData(ctx, tenantID, startMonth, endMonth)
	if err != nil {
		return nil, err
	}

	// 按 cohort 月份聚合
	cohortMap := make(map[string]*CohortRow)
	for _, dp := range data {
		row, ok := cohortMap[dp.CohortMonth]
		if !ok {
			row = &CohortRow{CohortMonth: dp.CohortMonth}
			cohortMap[dp.CohortMonth] = row
		}
		if dp.ActiveMonth == 0 {
			row.TotalUsers = dp.UserCount
		}

		// 计算留存百分比（保护：ActiveMonth != 0 但 TotalUsers 为 0 时跳过）
		if row.TotalUsers == 0 && dp.ActiveMonth != 0 {
			continue
		}
		pct := float64(dp.UserCount) / float64(row.TotalUsers) * 100
		switch dp.ActiveMonth {
		case 0:
			row.Month0 = pct
		case 1:
			row.Month1 = pct
		case 2:
			row.Month2 = pct
		case 3:
			row.Month3 = pct
		case 6:
			row.Month6 = pct
		case 12:
			row.Month12 = pct
		}
	}

	// 排序输出
	var rows []CohortRow
	for _, row := range cohortMap {
		rows = append(rows, *row)
	}
	// 按 cohort 月份排序
	sort.Slice(rows, func(i, j int) bool {
		return rows[i].CohortMonth < rows[j].CohortMonth
	})

	return rows, nil
}

// SQL 参考（ClickHouse / MySQL 留存分析）
// 基于订单表计算按月留存：
//
// WITH signup_cohort AS (
//     SELECT
--         user_id,
--         DATE_FORMAT(created_at, '%Y-%m') AS cohort_month
--     FROM users
--     WHERE created_at BETWEEN ? AND ?
-- ),
-- activity AS (
--     SELECT
--         sc.cohort_month,
--         sc.user_id,
--         PERIOD_DIFF(
--             DATE_FORMAT(o.paid_at, '%Y-%m'),
--             sc.cohort_month
--         ) AS active_month
--     FROM signup_cohort sc
--     JOIN orders o ON o.user_id = sc.user_id AND o.status = 'paid'
-- )
-- SELECT
--     cohort_month,
--     active_month,
--     COUNT(DISTINCT user_id) AS user_count
-- FROM activity
-- GROUP BY cohort_month, active_month
-- ORDER BY cohort_month, active_month;
```

### 4. 商品排行 (Product Ranking)

```go
package analytics

import (
	"context"
	"sort"
	"time"

	"github.com/zeromicro/go-zero/core/logx"
)

// ProductRankItem 商品排行项
type ProductRankItem struct {
	Rank       int     `json:"rank"`
	SKU        string  `json:"sku"`
	ProductName string `json:"product_name"`
	Category   string  `json:"category"`
	Revenue    int64   `json:"revenue"`     // 销售额（分）
	Quantity   int64   `json:"quantity"`    // 销量
	Margin     int64   `json:"margin"`      // 毛利（分）
	MarginRate float64 `json:"margin_rate"` // 毛利率（%）
}

// ProductRankService 商品排行服务
type ProductRankService struct {
	repo ProductRankRepo
}

// ProductRankRepo 商品排行数据仓库接口
type ProductRankRepo interface {
	QueryProductStats(
		ctx context.Context,
		tenantID string,
		start, end time.Time,
	) ([]ProductStatsRaw, error)
}

// ProductStatsRaw 原始商品统计数据
type ProductStatsRaw struct {
	SKU         string
	ProductName string
	Category    string
	Revenue     int64
	Cost        int64 // 成本
	Quantity    int64
}

// SortKeyFn defines how to extract a sortable value from a ProductRankItem.
// Return larger values for descending rank.
type SortKeyFn func(item *ProductRankItem) int64

// Common sort key functions.
var (
	SortByRevenue  SortKeyFn = func(i *ProductRankItem) int64 { return i.Revenue }
	SortByQuantity SortKeyFn = func(i *ProductRankItem) int64 { return i.Quantity }
	SortByMargin   SortKeyFn = func(i *ProductRankItem) int64 { return i.Margin }
)

// RankBy 商品排行（统一的排序方法，通过传入排序函数实现不同维度排行）
func (s *ProductRankService) RankBy(
	ctx context.Context,
	tenantID string,
	start, end time.Time,
	limit int,
	sortKey SortKeyFn,
) ([]ProductRankItem, error) {
	logx.WithContext(ctx).Infof("product ranking: tenant=%s limit=%d", tenantID, limit)
	raw, err := s.repo.QueryProductStats(ctx, tenantID, start, end)
	if err != nil {
		return nil, err
	}

	items := make([]ProductRankItem, 0, len(raw))
	for _, r := range raw {
		item := ProductRankItem{
			SKU:         r.SKU,
			ProductName: r.ProductName,
			Category:    r.Category,
			Revenue:     r.Revenue,
			Quantity:    r.Quantity,
			Margin:      r.Revenue - r.Cost,
		}
		if r.Revenue > 0 {
			item.MarginRate = float64(item.Margin) / float64(r.Revenue) * 100
		}
		items = append(items, item)
	}

	// 按指定 key 降序
	sort.Slice(items, func(i, j int) bool {
		return sortKey(&items[i]) > sortKey(&items[j])
	})

	// 截断并设置排名
	if limit > 0 && limit < len(items) {
		items = items[:limit]
	}
	for i := range items {
		items[i].Rank = i + 1
	}

	return items, nil
}
```

### 5. 实时计数器 (Real-time Counter)

```go
package analytics

import (
	"context"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
)

// RealTimeCounter 实时计数器（基于 Redis INCRBY，按日自动重置）
type RealTimeCounter struct {
	rdb      *redis.Client
	tenantID string
	ttl      time.Duration
}

// NewRealTimeCounter 创建实时计数器
func NewRealTimeCounter(rdb *redis.Client, tenantID string, ttl time.Duration) *RealTimeCounter {
	if ttl <= 0 {
		ttl = 90 * 24 * time.Hour // 默认 90 天
	}
	return &RealTimeCounter{rdb: rdb, tenantID: tenantID, ttl: ttl}
}

// key 格式: analytics:counter:{tenantID}:{metric}:{YYYYMMDD}
func (c *RealTimeCounter) makeKey(metric string, date time.Time) string {
	return fmt.Sprintf("analytics:counter:%s:%s:%s", c.tenantID, metric, date.Format("20060102"))
}

// incrWithTTL Lua 脚本：原子性执行 INCRBY + EXPIRE（仅首次创建时设置 TTL）
var incrWithTTL = redis.NewScript(`
local v = redis.call('INCRBY', KEYS[1], ARGV[1])
if redis.call('TTL', KEYS[1]) < 0 then
    redis.call('EXPIRE', KEYS[1], ARGV[2])
end
return v
`)

// Incr 对某指标加 1（如新增订单），原子性设置 TTL 防止内存泄漏
func (c *RealTimeCounter) Incr(ctx context.Context, metric string) (int64, error) {
	return c.IncrBy(ctx, metric, 1)
}

// IncrBy 对某指标加指定值（如增加 GMV 金额），原子性设置 TTL 防止内存泄漏
func (c *RealTimeCounter) IncrBy(ctx context.Context, metric string, amount int64) (int64, error) {
	key := c.makeKey(metric, time.Now().UTC())
	ttlSeconds := int(c.ttl.Seconds())
	return incrWithTTL.Run(ctx, c.rdb, []string{key}, amount, ttlSeconds).Int64()
}

// Get 获取某指标当日当前值
func (c *RealTimeCounter) Get(ctx context.Context, metric string) (int64, error) {
	key := c.makeKey(metric, time.Now().UTC())
	return c.rdb.Get(ctx, key).Int64()
}

// GetAtDate 获取某指标指定日期的值
func (c *RealTimeCounter) GetAtDate(ctx context.Context, metric string, date time.Time) (int64, error) {
	key := c.makeKey(metric, date)
	return c.rdb.Get(ctx, key).Int64()
}

// Expire 设置过期时间（防止 Redis 内存无限增长）
// 建议：实时计数器保留 90 天
func (c *RealTimeCounter) Expire(ctx context.Context, metric string, ttl time.Duration) error {
	key := c.makeKey(metric, time.Now().UTC())
	return c.rdb.Expire(ctx, key, ttl).Err()
}

// 常用指标名常量
const (
	MetricOrderCount   = "order_count"    // 今日订单数
	MetricGMV          = "gmv"            // 今日 GMV（分）
	MetricNewUsers     = "new_users"      // 今日新注册用户
	MetricPaidUsers    = "paid_users"     // 今日支付用户数（HyperLogLog 更合适）
	MetricPageViews    = "page_views"     // 今日 PV
	MetricUniqueVisitors = "unique_visitors" // 今日 UV（用 HyperLogLog）
)

// HyperLogLogCounter 基于 HyperLogLog 的去重计数器（用于 UV）
type HyperLogLogCounter struct {
	rdb      *redis.Client
	tenantID string
	ttl      time.Duration
}

// NewHyperLogLogCounter 创建去重计数器
func NewHyperLogLogCounter(rdb *redis.Client, tenantID string, ttl time.Duration) *HyperLogLogCounter {
	if ttl <= 0 {
		ttl = 90 * 24 * time.Hour // 默认 90 天
	}
	return &HyperLogLogCounter{rdb: rdb, tenantID: tenantID, ttl: ttl}
}

func (h *HyperLogLogCounter) makeKey(metric string, date time.Time) string {
	return fmt.Sprintf("analytics:hll:%s:%s:%s", h.tenantID, metric, date.Format("20060102"))
}

// addWithTTL Lua 脚本：原子性执行 PFADD + EXPIRE
var addWithTTL = redis.NewScript(`
redis.call('PFADD', KEYS[1], ARGV[1])
if redis.call('TTL', KEYS[1]) < 0 then
    redis.call('EXPIRE', KEYS[1], ARGV[2])
end
return 1
`)

// Add 添加一个唯一 ID（如 user_id），原子性设置 TTL 防止内存泄漏
func (h *HyperLogLogCounter) Add(ctx context.Context, metric string, id string) error {
	key := h.makeKey(metric, time.Now().UTC())
	ttlSeconds := int(h.ttl.Seconds())
	return addWithTTL.Run(ctx, h.rdb, []string{key}, id, ttlSeconds).Err()
}

// Count 获取去重后的总数
func (h *HyperLogLogCounter) Count(ctx context.Context, metric string) (int64, error) {
	key := h.makeKey(metric, time.Now().UTC())
	return h.rdb.PFCount(ctx, key).Result()
}

// 使用示例：
//
// counter := NewRealTimeCounter(redisClient, "tenant-1", 90*24*time.Hour)
// // 订单创建成功后：
// counter.Incr(ctx, analytics.MetricOrderCount)
// counter.IncrBy(ctx, analytics.MetricGMV, order.TotalAmount)
//
// // 简单计数（不去重，适用于单次事件）
// counter.IncrBy(ctx, analytics.MetricPaidUsers, 1)
//
// // 去重计数（同一用户多次支付只计一次）
// hll := NewHyperLogLogCounter(redisClient, "tenant-1", 90*24*time.Hour)
// hll.Add(ctx, analytics.MetricPaidUsers, userID)
//
// // 用户访问页面时：
// hll.Add(ctx, analytics.MetricUniqueVisitors, userID)
// // 查询今日 UV：
// uv, _ := hll.Count(ctx, analytics.MetricUniqueVisitors)
```

### 6. 批量报表聚合（go-zero MapReduce 并发）

```go
package analytics

import (
	"context"
	"fmt"
	"time"

	"golang.org/x/sync/errgroup"

	"github.com/zeromicro/go-zero/core/logx"
	"github.com/zeromicro/go-zero/core/mr"
)

// DailyReport 日报表数据
type DailyReport struct {
	Date       string `json:"date"`
	GMV        int64  `json:"gmv"`
	Orders     int64  `json:"orders"`
	NewUsers   int64  `json:"new_users"`
	Revenue    int64  `json:"revenue"`
	Cost       int64  `json:"cost"`
	Margin     int64  `json:"margin"`
}

// ReportAggregator 报表聚合器
type ReportAggregator struct {
	repo ReportRepo
}

// ReportRepo 报表数据仓库接口
type ReportRepo interface {
	QueryDailyGMV(ctx context.Context, tenantID, date string) (int64, error)
	QueryDailyOrders(ctx context.Context, tenantID, date string) (int64, error)
	QueryDailyNewUsers(ctx context.Context, tenantID, date string) (int64, error)
	QueryDailyRevenue(ctx context.Context, tenantID, date string) (int64, error)
	QueryDailyCost(ctx context.Context, tenantID, date string) (int64, error)
}

// NewReportAggregator 创建报表聚合器
func NewReportAggregator(repo ReportRepo) *ReportAggregator {
	return &ReportAggregator{repo: repo}
}

// GenerateDailyReports 批量生成日报表（并发聚合）
// 使用 go-zero MapReduce 并发查询多天的数据
func (a *ReportAggregator) GenerateDailyReports(
	ctx context.Context,
	tenantID string,
	startDate, endDate string,
) ([]DailyReport, error) {
	// 生成日期列表
	dates, err := generateDateRange(startDate, endDate)
	if err != nil {
		return nil, err
	}

	// MapReduce 并发查询
	type dateResult struct {
		Date   string
		Report *DailyReport
		Err    error
	}

	// Generate: 产出每个日期
	generate := func(source chan<- string) {
		for _, d := range dates {
			source <- d
		}
	}

	// Mapper: 并发查询某天的所有指标（使用 errgroup 避免 goroutine 泄漏）
	mapper := func(ctx context.Context, date string, writer mr.Writer[dateResult], cancel func(error)) {
		report := &DailyReport{Date: date}

		// 使用 errgroup 管理并发查询，确保所有 goroutine 都能正确退出
		g, gctx := errgroup.WithContext(ctx)

		var gmv, orders, users, revenue, cost int64
		g.Go(func() error {
			v, err := a.repo.QueryDailyGMV(gctx, tenantID, date)
			gmv = v
			return err
		})
		g.Go(func() error {
			v, err := a.repo.QueryDailyOrders(gctx, tenantID, date)
			orders = v
			return err
		})
		g.Go(func() error {
			v, err := a.repo.QueryDailyNewUsers(gctx, tenantID, date)
			users = v
			return err
		})
		g.Go(func() error {
			v, err := a.repo.QueryDailyRevenue(gctx, tenantID, date)
			revenue = v
			return err
		})
		g.Go(func() error {
			v, err := a.repo.QueryDailyCost(gctx, tenantID, date)
			cost = v
			return err
		})

		if err := g.Wait(); err != nil {
			cancel(err)
			return
		}

		report.GMV = gmv
		report.Orders = orders
		report.NewUsers = users
		report.Revenue = revenue
		report.Cost = cost
		report.Margin = report.Revenue - report.Cost
		writer.Write(dateResult{Date: date, Report: report})
	}

	// Reducer: 收集所有结果
	var reports []DailyReport
	reduce := func(source <-chan dateResult, writer mr.Writer[[]DailyReport], cancel func(error)) {
		var results []DailyReport
		for r := range source {
			results = append(results, *r.Report)
		}
		writer.Write(results)
	}

	// Add timeout for the entire MapReduce operation
	mrCtx, cancel := context.WithTimeout(ctx, 30*time.Second)
	defer cancel()

	result, err := mr.MapReduce(mrCtx, generate, mapper, reduce)
	if err != nil {
		return nil, err
	}

	return result, nil
}

// generateDateRange 生成日期范围列表（限制最大 365 天）
func generateDateRange(startDate, endDate string) ([]string, error) {
	start, err := time.Parse("2006-01-02", startDate)
	if err != nil {
		return nil, fmt.Errorf("parse start date %q: %w", startDate, err)
	}
	end, err := time.Parse("2006-01-02", endDate)
	if err != nil {
		return nil, fmt.Errorf("parse end date %q: %w", endDate, err)
	}
	if end.Before(start) {
		return nil, fmt.Errorf("end date %q is before start date %q", endDate, startDate)
	}
	if end.Sub(start) > 365*24*time.Hour {
		return nil, fmt.Errorf("date range exceeds maximum of 365 days")
	}

	var dates []string
	for d := start; !d.After(end); d = d.AddDate(0, 0, 1) {
		dates = append(dates, d.Format("2006-01-02"))
	}
	return dates, nil
}
```

### 7. Excel 报表导出（带格式化表头和条件格式）

```go
package analytics

import (
	"context"
	"fmt"
	"time"

	"github.com/xuri/excelize/v2"
)

// ExcelExporter Excel 报表导出器
type ExcelExporter struct{}

// NewExcelExporter 创建导出器
func NewExcelExporter() *ExcelExporter {
	return &ExcelExporter{}
}

// ExportDashboardReport 导出经营看板报表
func (e *ExcelExporter) ExportDashboardReport(
	metrics []DashboardMetric,
) ([]byte, error) {
	f := excelize.NewFile()
	defer f.Close()

	sheetName := "经营看板"
	index, err := f.NewSheet(sheetName)
	if err != nil {
		return nil, fmt.Errorf("excel: create sheet: %w", err)
	}
	f.SetActiveSheet(index)
	f.DeleteSheet("Sheet1") // 删除默认 sheet

	// 定义表头样式（蓝色背景、白色粗体）
	headerStyle, err := f.NewStyle(&excelize.Style{
		Font: &excelize.Font{
			Bold:  true,
			Size:  12,
			Color: "FFFFFF",
		},
		Fill: excelize.Fill{
			Type:    "pattern",
			Pattern: 1,
			Color:   []string{"4472C4"},
		},
		Alignment: &excelize.Alignment{
			Horizontal: "center",
			Vertical:   "center",
		},
		Border: []excelize.Border{
			{Type: "left", Color: "D0D7E5", Style: 1},
			{Type: "top", Color: "D0D7E5", Style: 1},
			{Type: "right", Color: "D0D7E5", Style: 1},
			{Type: "bottom", Color: "D0D7E5", Style: 1},
		},
	})
	if err != nil {
		return nil, err
	}

	// 定义数据行样式
	dataStyle, err := f.NewStyle(&excelize.Style{
		Alignment: &excelize.Alignment{
			Horizontal: "right",
			Vertical:   "center",
		},
		Border: []excelize.Border{
			{Type: "left", Color: "D0D7E5", Style: 1},
			{Type: "right", Color: "D0D7E5", Style: 1},
			{Type: "bottom", Color: "D0D7E5", Style: 1},
		},
		NumFmt: 4, // 千分位数字格式
	})
	if err != nil {
		return nil, err
	}

	// 百分比样式
	pctStyle, err := f.NewStyle(&excelize.Style{
		Alignment: &excelize.Alignment{
			Horizontal: "right",
			Vertical:   "center",
		},
		NumFmt: 10, // 百分比格式
		Border: []excelize.Border{
			{Type: "left", Color: "D0D7E5", Style: 1},
			{Type: "right", Color: "D0D7E5", Style: 1},
			{Type: "bottom", Color: "D0D7E5", Style: 1},
		},
	})
	if err != nil {
		return nil, err
	}

	// 写入表头
	headers := []string{
		"日期", "GMV(元)", "订单量", "支付用户数",
		"访问用户数", "转化率", "客单价(元)", "退款率",
	}
	for i, h := range headers {
		cell, _ := excelize.CoordinatesToCellName(i+1, 1)
		f.SetCellValue(sheetName, cell, h)
		f.SetCellStyle(sheetName, cell, cell, headerStyle)
	}

	// 写入数据行
	for rowIdx, m := range metrics {
		row := rowIdx + 2 // 从第 2 行开始（第 1 行是表头）

		gmvYuan := float64(m.GMV) / 100
		aovYuan := float64(m.AOV) / 100

		data := []any{
			m.StartTime,              // 日期
			gmvYuan,                  // GMV
			m.OrderCount,             // 订单量
			m.PaidUserCount,          // 支付用户数
			m.VisitorCount,           // 访问用户数
			m.ConversionRate / 100,   // 转化率（Excel 百分比）
			aovYuan,                  // 客单价
			m.RefundRate / 100,       // 退款率（Excel 百分比）
		}

		for colIdx, val := range data {
			cell, _ := excelize.CoordinatesToCellName(colIdx+1, row)
			f.SetCellValue(sheetName, cell, val)

			// 百分比列用 pctStyle
			if colIdx == 5 || colIdx == 7 {
				f.SetCellStyle(sheetName, cell, cell, pctStyle)
			} else {
				f.SetCellStyle(sheetName, cell, cell, dataStyle)
			}
		}
	}

	// 自动调整列宽
	for i := 1; i <= len(headers); i++ {
		col, _ := excelize.ColumnNumberToName(i)
		f.SetColWidth(sheetName, col, col, 18)
	}

	// 添加条件格式：GMV 低于 10 元标红（动态范围）
	lastRow := len(metrics) + 1
	f.AddConditionalFormat(sheetName, fmt.Sprintf("B2:B%d", lastRow),
		[]excelize.ConditionalFormatOptions{
			{
				Type:     "cell",
				Criteria: "less than",
				Value:    "10", // GMV < 10 元时标红
				Format:   &excelize.Style{Font: &excelize.Font{Color: "FF0000", Bold: true}},
			},
		},
	)

	// 添加条件格式：转化率低于 1% 标黄（动态范围）
	f.AddConditionalFormat(sheetName, fmt.Sprintf("F2:F%d", lastRow),
		[]excelize.ConditionalFormatOptions{
			{
				Type:     "cell",
				Criteria: "less than",
				Value:    "0.01",
				Format:   &excelize.Style{Fill: excelize.Fill{Type: "pattern", Pattern: 1, Color: []string{"FFC000"}}},
			},
		},
	)

	// 冻结首行
	f.SetPanes(sheetName, &excelize.Panes{
		Freeze:      true,
		Split:       false,
		XSplit:      0,
		YSplit:      1,
		TopLeftCell: "A2",
		ActivePane:  "bottomLeft",
	})

	// 输出为字节流
	buf, err := f.WriteToBuffer()
	if err != nil {
		return nil, fmt.Errorf("excel: write to buffer: %w", err)
	}

	return buf.Bytes(), nil
}

// ExportStreaming 流式导出大数据集（逐行写入，不全部加载到内存）
// 适用于万级以上数据量的报表
func (e *ExcelExporter) ExportStreaming(
	ctx context.Context,
	query func(ctx context.Context, offset, limit int) ([]DashboardMetric, error),
	totalCount int,
) ([]byte, error) {
	f := excelize.NewFile()
	defer f.Close()

	sheetName := "数据报表"
	if _, err := f.NewSheet(sheetName); err != nil {
		return nil, fmt.Errorf("excel: create sheet: %w", err)
	}

	batchSize := 500
	row := 2 // 从第 2 行开始写入

	// 逐批次查询并写入
	for offset := 0; offset < totalCount; offset += batchSize {
		// 支持 context 取消
		select {
		case <-ctx.Done():
			return nil, ctx.Err()
		default:
		}

		batch, err := query(ctx, offset, batchSize)
		if err != nil {
			return nil, err
		}

		for _, m := range batch {
			f.SetCellValue(sheetName, fmt.Sprintf("A%d", row), m.StartTime)
			f.SetCellValue(sheetName, fmt.Sprintf("B%d", row), float64(m.GMV)/100)
			f.SetCellValue(sheetName, fmt.Sprintf("C%d", row), m.OrderCount)
			f.SetCellValue(sheetName, fmt.Sprintf("D%d", row), m.PaidUserCount)
			row++
		}
	}

	buf, err := f.WriteToBuffer()
	if err != nil {
		return nil, err
	}
	return buf.Bytes(), nil
}
```

## 反模式

### 1. 在生产库上跑 OLAP 查询

```go
// 错误：直接在主库上执行重型聚合查询
func DailyReport(db *gorm.DB) error {
    // 全表扫描 + GROUP BY + 多表 JOIN，会锁表影响线上交易
    return db.Raw(`
        SELECT DATE(o.created_at), COUNT(*), SUM(o.total_amount)
        FROM orders o
        JOIN order_items oi ON oi.order_id = o.id
        JOIN products p ON p.id = oi.product_id
        GROUP BY DATE(o.created_at)
    `).Error
}
```

**问题**：全表扫描和复杂 JOIN 会消耗大量 CPU/IO，阻塞线上交易写入。

**正确做法**：使用只读副本或 ClickHouse 独立实例执行分析查询。

```go
// 正确：走 ClickHouse OLAP 实例，带 tenant_id 过滤
func DailyReportCH(ctx context.Context, ch clickhouse.Conn, tenantID string) error {
    return ch.Query(ctx, `
        SELECT
            toDate(created_at) AS day,
            count() AS orders,
            sum(total_amount) AS gmv
        FROM orders_distributed
        WHERE tenant_id = ?
        GROUP BY day
        ORDER BY day
    `, tenantID)
}
```

### 2. 在应用层循环计算指标

```go
// 错误：在 Go 代码里循环累加
func CalcGMV(db *gorm.DB, start, end time.Time) (int64, error) {
    var orders []Order
    db.Where("paid_at BETWEEN ? AND ?", start, end).Find(&orders)

    var total int64
    for _, o := range orders {
        total += o.TotalAmount // N 条记录，全部加载到内存后逐条加
    }
    return total, nil
}
```

**问题**：把聚合逻辑放在应用层，需要将全量数据加载到内存，浪费内存和带宽。

**正确做法**：让数据库做聚合（SUM/COUNT/GROUP BY）。

```go
// 正确：数据库层聚合，带 tenant_id 过滤
func CalcGMV(db *gorm.DB, tenantID string, start, end time.Time) (int64, error) {
    var total int64
    err := db.Model(&Order{}).
        Where("tenant_id = ? AND paid_at BETWEEN ? AND ?", tenantID, start, end).
        Select("COALESCE(SUM(total_amount), 0)").
        Scan(&total).Error
    return total, err
}
```

### 3. 看板查询不缓存

```go
// 错误：每次刷新看板都直接查询数据库
func GetDashboard(w http.ResponseWriter, r *http.Request) {
    metrics, _ := dashboardService.GetMetrics(ctx, start, end) // 每次都重算
    json.NewEncoder(w).Encode(metrics)
}
```

**问题**：经营看板通常多人频繁刷新，每次都重算会给数据库带来不必要的压力。

**正确做法**：加短 TTL 缓存（1-5 分钟）。

```go
// 正确：带缓存的看板查询
type CachedDashboard struct {
    svc     *DashboardService
    cache   *RedisCache
    ttl     time.Duration
}

func (c *CachedDashboard) GetMetrics(ctx context.Context, tenantID, period string) (*DashboardMetric, error) {
    cacheKey := fmt.Sprintf("dashboard:metrics:%s:%s", tenantID, period)

    // 先查缓存
    var cached DashboardMetric
    if err := c.cache.Get(ctx, cacheKey, &cached); err == nil {
        return &cached, nil
    }

    // 缓存未命中，查询数据库
    now := time.Now().UTC()
    metrics, err := c.svc.GetMetrics(ctx, tenantID, now.Add(-24*time.Hour), now, period)
    if err != nil {
        return nil, err
    }

    // 写入缓存，TTL 3 分钟
    c.cache.Set(ctx, cacheKey, metrics, 3*time.Minute)
    return metrics, nil
}
```

### 4. 硬编码指标公式

```go
// 错误：公式写死在代码里
func CalcConversionRate(orders, visitors int64) float64 {
    return float64(orders) / float64(visitors) * 100 // 魔法公式
}
```

**问题**：指标定义可能随业务变化（如 GMV 是否含运费、退款是否扣减），硬编码导致修改困难。

**正确做法**：指标公式外部化到配置文件。

```yaml
# config/metrics.yaml
metrics:
  gmv:
    formula: "SUM(total_amount)"
    description: "含运费，不含退款"
    tables: ["orders"]
    conditions:
      - "status = 'paid'"
  conversion_rate:
    formula: "COUNT(DISTINCT order_id) / COUNT(DISTINCT visitor_id) * 100"
    description: "支付订单数 / 访问用户数"
```

## 常见坑

### 1. 日报时区问题

**坑**：服务器用 UTC，业务用 Asia/Shanghai，导致「今日」报表的数据范围偏差 8 小时。

**解决**：
- 所有时间戳统一存储为 UTC
- 在展示层（前端或 API 响应层）才转换为用户所在时区
- 日报的「今天」定义为：`UTC 00:00:00` 到 `UTC 23:59:59`，或按业务定义的中国时间 `08:00:00 UTC` 到次日 `07:59:59 UTC`

```go
// 正确：UTC 存储，展示层转换
func TodayRange() (time.Time, time.Time) {
    // 中国时间的今天 0 点
    cst := time.FixedZone("CST", 8*3600)
    now := time.Now().In(cst)
    start := time.Date(now.Year(), now.Month(), now.Day(), 0, 0, 0, 0, cst)
    end := start.Add(24 * time.Hour)
    // 转换为 UTC 用于数据库查询
    return start.UTC(), end.UTC()
}
```

### 2. 报表查询 N+1 问题

**坑**：先查出所有日期，再逐日查询指标，导致 N 次数据库往返。

```go
// 错误：循环逐日查询
for _, date := range dates {
    gmv := queryDailyGMV(date)     // 1 次查询
    orders := queryDailyOrders(date) // 1 次查询
    // N 天 × 2 指标 = 2N 次查询
}
```

**解决**：
- 一次性查询所有日期的数据，按 GROUP BY 日期返回
- 使用物化视图（Materialized View）或汇总表预聚合

```go
// 正确：单次查询获取多日数据，带 tenant_id 过滤
func QueryMultiDayStats(db *gorm.DB, tenantID string, start, end time.Time) ([]DailyStats, error) {
    var stats []DailyStats
    err := db.Raw(`
        SELECT
            DATE(created_at) AS date,
            COUNT(*) AS order_count,
            COALESCE(SUM(total_amount), 0) AS gmv
        FROM orders
        WHERE tenant_id = ? AND paid_at BETWEEN ? AND ? AND status = 'paid'
        GROUP BY DATE(created_at)
        ORDER BY date
    `, tenantID, start, end).Scan(&stats).Error
    return stats, err
}
```

### 3. 大数据量导出内存溢出

**坑**：导出 10 万行报表时一次性查询全部数据加载到内存。

```go
// 错误：全量加载到内存
var allRows []OrderReport
db.Find(&allRows) // 10 万行 × 每行 200 字节 = 20MB，Excel 序列化后更大
excelize.WriteFile(allRows)
```

**解决**：分批查询 + 流式写入（参考上方 `ExportStreaming` 模板）。

```go
// 正确：分批查询，逐行写入
cursor := 0
batchSize := 1000
for {
    var batch []OrderReport
    db.Limit(batchSize).Offset(cursor).Find(&batch)
    if len(batch) == 0 {
        break
    }
    for _, r := range batch {
        f.SetCellValue(sheet, fmt.Sprintf("A%d", row), r.Date)
        f.SetCellValue(sheet, fmt.Sprintf("B%d", row), r.GMV)
        row++
    }
    cursor += batchSize
}
```

## 测试策略

### 覆盖率目标

| 层级 | 覆盖率要求 | 说明 |
|------|-----------|------|
| L1 Unit | >= 80% | ConvertPrice、generateDateRange、RankBy、实时计数器、GetMetrics |
| L2 Integration | >= 70% | miniredis 实时计数器 + GORM DB 聚合查询 + 多租户隔离 |
| L3 E2E | 核心流程 | 事件上报 → 实时计数 → MapReduce 聚合 → 报表导出 |

### Table-Driven Test 示例（`ConvertPrice`）

```go
func TestConvertPrice(t *testing.T) {
	tests := []struct {
		name      string
		input     int64
		currency  string
		want      float64
		wantErr   bool
	}{
		{"CNY yuan", 1234, "CNY", 12.34, false},
		{"CNY zero", 0, "CNY", 0.0, false},
		{"USD cents", 999, "USD", 9.99, false},
		{"unknown currency", 100, "XYZ", 0, true},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got, err := ConvertPrice(tt.input, tt.currency)
			if (err != nil) != tt.wantErr {
				t.Errorf("ConvertPrice() error = %v, wantErr %v", err, tt.wantErr)
				return
			}
			if got != tt.want {
				t.Errorf("ConvertPrice() = %v, want %v", got, tt.want)
			}
		})
	}
}
```

### Mock Repo Interface 示例

```go
func TestDashboardService_GetMetrics(t *testing.T) {
	// 构造 mock repo
	repo := &mockDashboardRepo{
		gmv:            100000,
		orderCount:     50,
		paidUserCount:  30,
		visitorCount:   500,
		refundCount:    5,
		repeatBuyRate:  0.25,
	}

	svc := NewDashboardService(repo)
	ctx := context.Background()

	metrics, err := svc.GetMetrics(ctx, "tenant-1", time.Now().UTC(), time.Now().UTC().Add(24*time.Hour), "daily")
	if err != nil {
		t.Fatalf("GetMetrics() error = %v", err)
	}

	if metrics.GMV != 100000 {
		t.Errorf("GMV = %d, want 100000", metrics.GMV)
	}
	if metrics.OrderCount != 50 {
		t.Errorf("OrderCount = %d, want 50", metrics.OrderCount)
	}
}

// mockDashboardRepo implements DashboardRepo for testing
type mockDashboardRepo struct {
	gmv, orderCount, paidUserCount, visitorCount, refundCount int64
	repeatBuyRate                                              float64
}

func (m *mockDashboardRepo) QueryGMV(ctx context.Context, tenantID string, start, end time.Time) (int64, error) {
	return m.gmv, nil
}

func (m *mockDashboardRepo) QueryOrderCount(ctx context.Context, tenantID string, start, end time.Time) (int64, error) {
	return m.orderCount, nil
}

func (m *mockDashboardRepo) QueryPaidUserCount(ctx context.Context, tenantID string, start, end time.Time) (int64, error) {
	return m.paidUserCount, nil
}

func (m *mockDashboardRepo) QueryVisitorCount(ctx context.Context, tenantID string, start, end time.Time) (int64, error) {
	return m.visitorCount, nil
}

func (m *mockDashboardRepo) QueryRefundCount(ctx context.Context, tenantID string, start, end time.Time) (int64, error) {
	return m.refundCount, nil
}

func (m *mockDashboardRepo) QueryRepeatBuyRate(ctx context.Context, tenantID string, start, end time.Time) (float64, error) {
	return m.repeatBuyRate, nil
}
```

### 多租户隔离测试

```go
func TestRealTimeCounter_MultiTenantIsolation(t *testing.T) {
	mr := miniredis.RunT(t)
	rdb := redis.NewClient(&redis.Options{Addr: mr.Addr()})
	// tenantID 在构造时传入，确保不同租户的 Redis key 隔离
	counterA := NewRealTimeCounter(rdb, "tenantA", 90*24*time.Hour)
	counterB := NewRealTimeCounter(rdb, "tenantB", 90*24*time.Hour)
	ctx := context.Background()

	// tenantA 和 tenantB 各自独立计数
	counterA.IncrBy(ctx, "gmv", 100)
	counterB.IncrBy(ctx, "gmv", 200)

	valA, _ := counterA.Get(ctx, "gmv")
	valB, _ := counterB.Get(ctx, "gmv")
	if valA != 100 {
		t.Errorf("tenantA gmv = %d, want 100", valA)
	}
	if valB != 200 {
		t.Errorf("tenantB gmv = %d, want 200", valB)
	}
}
```

### MapReduce 并发测试

```go
func TestReportAggregator_GenerateDailyReports_Concurrent(t *testing.T) {
	// 使用 mock repo 测试 MapReduce 并发查询
	repo := &mockReportRepo{gmv: 10000, orders: 50, newUsers: 30, revenue: 8000, cost: 5000}
	agg := NewReportAggregator(repo)
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	reports, err := agg.GenerateDailyReports(ctx, "tenant-1", "2026-01-01", "2026-01-07")
	if err != nil {
		t.Fatalf("GenerateDailyReports: %v", err)
	}
	if len(reports) != 7 {
		t.Errorf("expected 7 reports, got %d", len(reports))
	}
	// 使用 go test -race 检测数据竞争
}

// mockReportRepo implements ReportRepo for testing
type mockReportRepo struct {
	gmv, orders, newUsers, revenue, cost int64
}

func (m *mockReportRepo) QueryDailyGMV(ctx context.Context, tenantID, date string) (int64, error) {
	return m.gmv, nil
}

func (m *mockReportRepo) QueryDailyOrders(ctx context.Context, tenantID, date string) (int64, error) {
	return m.orders, nil
}

func (m *mockReportRepo) QueryDailyNewUsers(ctx context.Context, tenantID, date string) (int64, error) {
	return m.newUsers, nil
}

func (m *mockReportRepo) QueryDailyRevenue(ctx context.Context, tenantID, date string) (int64, error) {
	return m.revenue, nil
}

func (m *mockReportRepo) QueryDailyCost(ctx context.Context, tenantID, date string) (int64, error) {
	return m.cost, nil
}
```

### 集成测试（miniredis + GORM SQLite）

```go
func TestRealTimeCounter_Integration(t *testing.T) {
	mr := miniredis.RunT(t)
	rdb := redis.NewClient(&redis.Options{Addr: mr.Addr()})
	counter := NewRealTimeCounter(rdb, "tenant-1", 90*24*time.Hour)
	ctx := context.Background()

	// 测试连续累加
	counter.IncrBy(ctx, "gmv", 100)
	counter.IncrBy(ctx, "gmv", 200)
	val, _ := counter.Get(ctx, "gmv")
	if val != 300 {
		t.Errorf("gmv = %d, want 300", val)
	}
}
```

## 合规要求 (GDPR)

### 用户 ID 匿名化

事件日志中的 `user_id` 字段在写入前必须做单向哈希（如 SHA-256 + salt），禁止明文存储可识别的个人身份信息（PII）。

```go
import "crypto/sha256"

// AnonymizeUserID 返回用户 ID 的不可逆哈希
func AnonymizeUserID(userID, salt string) string {
	h := sha256.Sum256([]byte(userID + salt))
	return fmt.Sprintf("%x", h)
}
```

### 数据保留策略

分析数据必须配置保留周期（TTL），过期自动清理：

| 数据类型 | 保留周期 | 存储 |
|----------|----------|------|
| 实时计数器 | 90 天 | Redis (TTL) |
| 事件日志 | 180 天 | ClickHouse (PARTITION BY + DROP) |
| 聚合报表 | 365 天 | PG / ClickHouse |

### 删除权支持 (Right to Erasure)

当用户行使 GDPR 删除权时，必须能够：
1. 删除或匿名化该用户的所有事件日志记录
2. 从 HyperLogLog 中无法直接删除，需重建该日期的 HLL（通过 replay 事件日志中其他用户数据）
3. 记录删除操作到审计日志（不含被删数据本身）

```go
// ComplianceService 合规服务
type ComplianceService struct {
	eventRepo EventRepo
	salt      string
}

// EventRepo 事件数据仓库接口
type EventRepo interface {
	DeleteByUserID(ctx context.Context, tenantID, userID string) error
}

// EraseUserData 删除指定用户的所有可识别数据
func (svc *ComplianceService) EraseUserData(ctx context.Context, tenantID, userID string) error {
	// 1. 删除事件日志中的记录
	if err := svc.eventRepo.DeleteByUserID(ctx, tenantID, userID); err != nil {
		return fmt.Errorf("delete events: %w", err)
	}

	// 2. HyperLogLog 无法单独删除元素，需重建（见上方合规说明）
	// 生产环境可通过 replay 事件日志中其他用户数据来重建 HLL

	// 3. 记录审计日志（仅记录操作，不存用户数据）
	logx.WithContext(ctx).Infof("erasure request completed: tenant=%s user_hash=%s",
		tenantID, AnonymizeUserID(userID, svc.salt))

	return nil
}
```
