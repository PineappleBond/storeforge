# 风控系统 (Risk Control & Fraud Prevention)

> **Note**: This is a supplementary pattern extending beyond the core 10 patterns listed in the [StoreForge design doc](../2026-04-20-storeforge-plugin-design.md). Planned for Phase 2 implementation.

## 触发条件

- 防刷单（同一用户/设备短时间内高频下单）
- 异常交易检测（金额突变、收货地址异常、非常规时段交易）
- 反欺诈（盗号、黑产、羊毛党、虚假交易）
- 设备指纹识别与追踪
- 黑名单管理（IP/设备/用户）
- 支付风控（盗刷、拒付、异常支付渠道）

---

## 推荐第三方库

| 用途 | 推荐库 | 导入路径 |
|------|--------|----------|
| 设备指纹 | go-zero builtin | 基于请求特征提取 |
| 规则引擎 | expr | `github.com/expr-lang/expr` |
| IP 地理位置 | geoip2 | `github.com/oschwald/geoip2-golang` |
| 布隆过滤器 | bloom | `github.com/bits-and-blooms/bloom/v3` |
| 限流 | redis_rate | `github.com/go-redis/redis_rate/v10` |

---

## 代码模板

### 风控规则模型

```go
// RiskRule 定义一条风控规则
type RiskRule struct {
	RuleID    string `json:"rule_id"`    // 规则唯一标识，如 "RULE_ORDER_FREQ_001"
	Name      string `json:"name"`       // 规则名称，如 "同地址高频下单"
	Condition string `json:"condition"`  // expr 语言编写的条件表达式
	Action    string `json:"action"`     // block / review / pass
	Weight    int    `json:"weight"`     // 该规则命中后贡献的分数/权重
	Priority  int    `json:"priority"`   // 优先级，数值越大越先执行
	Status    int    `json:"status"`     // 0=禁用, 1=启用
}

// 规则示例
var DefaultRiskRules = []RiskRule{
	{
		RuleID:    "RULE_ORDER_FREQ_001",
		Name:      "同用户同地址短时多单",
		Condition: `user_orders_same_address_1h >= ${threshold.order_freq_count}`,
		Action:    "review",
		Weight:    30,
		Priority:  90,
		Status:    1,
	},
	{
		RuleID:    "RULE_NEW_ACCT_HIGH_001",
		Name:      "新账号高价值异地址订单",
		Condition: `account_age_days < ${threshold.new_account_days} && order_amount > ${threshold.high_order_amount} && shipping_differs_profile == true`,
		Action:    "block",
		Weight:    50,
		Priority:  95,
		Status:    1,
	},
	{
		RuleID:    "RULE_MULTI_ACCT_IP_001",
		Name:      "同 IP 多账号下单",
		Condition: `ip_account_count_24h >= ${threshold.multi_acct_count}`,
		Action:    "block",
		Weight:    60,
		Priority:  99,
		Status:    1,
	},
}
```

### 阈值配置 (Threshold Configuration)

> 规则中的阈值**不应硬编码**，必须通过 `${threshold.xxx}` 变量引用，在求值前
> 由租户配置系统注入。每个租户可以覆盖全局默认阈值，实现差异化风控策略。

```go
// RiskThresholds 租户级风控阈值配置
type RiskThresholds struct {
	TenantID          string  `json:"tenant_id"`
	OrderFreqCount    int     `json:"order_freq_count"`     // 同地址高频下单阈值
	NewAccountDays    int     `json:"new_account_days"`     // 新账号判定天数
	HighOrderAmount   int64   `json:"high_order_amount"`    // 高价值订单金额阈值（单位：分）
	MultiAcctCount    int     `json:"multi_acct_count"`     // 同 IP 多账号判定数量
	BlockScore        int     `json:"block_score"`          // 触发 block 的总分阈值
	ReviewScore       int     `json:"review_score"`         // 触发 review 的总分阈值
}

// DefaultThresholds 全局默认阈值
var DefaultThresholds = RiskThresholds{
	OrderFreqCount:  3,
	NewAccountDays:  7,
	HighOrderAmount: 5000,
	MultiAcctCount:  3,
	BlockScore:      80,
	ReviewScore:     40,
}

// ResolveThresholds 合并租户配置与全局默认值
func ResolveThresholds(tenantCfg *RiskThresholds) RiskThresholds {
	t := DefaultThresholds
	if tenantCfg == nil {
		return t
	}
	if tenantCfg.OrderFreqCount > 0 {
		t.OrderFreqCount = tenantCfg.OrderFreqCount
	}
	if tenantCfg.NewAccountDays > 0 {
		t.NewAccountDays = tenantCfg.NewAccountDays
	}
	if tenantCfg.HighOrderAmount > 0 {
		t.HighOrderAmount = tenantCfg.HighOrderAmount
	}
	if tenantCfg.MultiAcctCount > 0 {
		t.MultiAcctCount = tenantCfg.MultiAcctCount
	}
	if tenantCfg.BlockScore > 0 {
		t.BlockScore = tenantCfg.BlockScore
	}
	if tenantCfg.ReviewScore > 0 {
		t.ReviewScore = tenantCfg.ReviewScore
	}
	return t
}
```

### 风控事件模型

```go
// RiskEvent 记录一次风控评估事件
type RiskEvent struct {
	EventID            string    `json:"event_id"`
	TenantID           string    `json:"tenant_id"`          // 多租户隔离标识
	EventType          string    `json:"event_type"`           // order_create / payment / login / register
	UserID             int64     `json:"user_id"`
	OrderID            string    `json:"order_id,omitempty"`
	IP                 string    `json:"ip"`
	DeviceIdentifier   string    `json:"device_identifier"`    // 服务端基础设备标识，非真实设备指纹
	RiskScore          int       `json:"risk_score"`           // 0-100，越高风险越大
	ActionTaken        string    `json:"action_taken"`         // block / review / pass
	TriggeredRuleIDs   []string  `json:"triggered_rule_ids"`   // 命中的规则 ID 列表
	RuleScores         map[string]int `json:"rule_scores"`     // 每条规则的贡献分数
	Timestamp          time.Time `json:"timestamp"`
}
```

### 设备标识符 (Device Identifier)

> **WARNING**: `DeviceIdentifier` 是一个**服务端基础标识符**（基于 IP + UA + header hash），
> **不是**真正的反欺诈设备指纹。生产环境必须引入客户端 SDK，采集 Canvas/WebGL/TCP 指纹、
> 硬件序列号、浏览器插件列表等深度信号，并结合第三方反欺诈服务（如 FingerprintJS Pro、
> Sift、Forter）才能实现可靠的设备级追踪。此外，需实现指纹轮换检测策略
> （fingerprint rotation strategy）：当同一用户在短时间内产生多个不同设备标识时，
> 应标记为可疑行为（可能是反指纹工具或黑产轮换设备）。

```go
import (
	"crypto/sha256"
	"fmt"
	"net"
	"net/http"
	"strings"
)

// DeviceIdentifier 从 HTTP 请求中提取基础设备标识
// 注意：这是服务端简单 hash，不是真正的反欺诈设备指纹
func DeviceIdentifier(r *http.Request) string {
	ip := extractClientIP(r)
	ua := r.UserAgent()

	// 从自定义 header 或 cookie 中获取前端上报的设备信息
	screenRes := r.Header.Get("X-Device-Screen")      // 如 "1920x1080"
	timezone := r.Header.Get("X-Device-Timezone")     // 如 "Asia/Shanghai"

	raw := fmt.Sprintf("%s|%s|%s|%s", ip, ua, screenRes, timezone)
	hash := sha256.Sum256([]byte(raw))
	return fmt.Sprintf("%x", hash[:16]) // 取前 16 字节作为标识
}

// extractClientIP 提取真实客户端 IP（考虑代理/X-Forwarded-For）
func extractClientIP(r *http.Request) string {
	xff := r.Header.Get("X-Forwarded-For")
	if xff != "" {
		// 取第一个 IP（最接近客户端）
		parts := strings.Split(xff, ",")
		return strings.TrimSpace(parts[0])
	}
	if xri := r.Header.Get("X-Real-IP"); xri != "" {
		return xri
	}
	host, _, err := net.SplitHostPort(r.RemoteAddr)
	if err != nil {
		return r.RemoteAddr // fallback for non-standard format
	}
	return host
}
```

### 订单欺诈检测规则（基于 expr 引擎）

```go
import (
	"context"
	"fmt"
	"sync"

	"github.com/expr-lang/expr"
	"github.com/expr-lang/expr/vm"
)

// FraudCheckContext 欺诈检测的上下文变量，供 expr 引擎求值
type FraudCheckContext struct {
	TenantID                  string  // 租户 ID，用于审计追踪
	UserID                    int64
	AccountAgeDays            int
	OrderAmount               int64   // 订单金额（单位：分）
	SameAddressOrders1h       int
	IPAccountCount24h         int
	ShippingDiffersProfile    bool
	DeviceInBlacklist         bool
	IPInBlacklist             bool
	PaymentVelocity1h         int     // 1 小时内支付次数
	AmountDeviationRatio     float64 // 偏离用户历史均值的倍数（内部计算用，非金额）
}

// RuleEngine 风控规则引擎（预编译规则缓存，避免每次请求重复编译）
type RuleEngine struct {
	mu       sync.RWMutex
	compiled map[string]*compiledRule // ruleID → 预编译结果
}

type compiledRule struct {
	program *vm.Program
	rule    RiskRule
}

// NewRuleEngine 创建规则引擎，预编译所有启用的规则
func NewRuleEngine(rules []RiskRule) (*RuleEngine, error) {
	engine := &RuleEngine{
		compiled: make(map[string]*compiledRule, len(rules)),
	}
	for _, rule := range rules {
		if rule.Status == 0 {
			continue
		}
		program, err := expr.Compile(rule.Condition, expr.Env(FraudCheckContext{}))
		if err != nil {
			return nil, fmt.Errorf("compile rule %s: %w", rule.RuleID, err)
		}
		engine.compiled[rule.RuleID] = &compiledRule{program: program, rule: rule}
	}
	return engine, nil
}

// ReloadRule 热更新单条规则（运行时动态加载新规则）
func (e *RuleEngine) ReloadRule(rule RiskRule) error {
	if rule.Status == 0 {
		return nil // 禁用的规则不加载
	}
	program, err := expr.Compile(rule.Condition, expr.Env(FraudCheckContext{}))
	if err != nil {
		return fmt.Errorf("compile rule %s: %w", rule.RuleID, err)
	}
	e.mu.Lock()
	defer e.mu.Unlock()
	e.compiled[rule.RuleID] = &compiledRule{program: program, rule: rule}
	return nil
}

// EvaluateAllRules 评估所有预编译规则，返回命中结果
func (e *RuleEngine) EvaluateAllRules(ctx context.Context, fraudCtx *FraudCheckContext) ([]RiskRule, error) {
	e.mu.RLock()
	defer e.mu.RUnlock()

	var triggered []RiskRule
	for _, cr := range e.compiled {
		output, err := expr.Run(cr.program, fraudCtx)
		if err != nil {
			continue
		}
		if matched, ok := output.(bool); matched && ok {
			triggered = append(triggered, cr.rule)
		}
	}
	return triggered, nil
}
```

### 布隆过滤器 — 黑名单快速检测（辅助加速层）

> **注意**: 布隆过滤器仅用于 **"definitely not in blacklist"** 快速路径判断。
> 返回 false 表示"一定不在黑名单中"，可以直接放行（zero-lookup fast-path）。
> 返回 true 表示"可能在黑名单中"（存在误判），必须回查 Redis Set 或数据库确认。
> **实际的黑名单查询应以 Redis Set 为主存储**，布隆过滤器只是可选的内存加速层。
> 必须实现持久化：布隆过滤器重启后会丢失数据，应从 Redis/DB 重建。

```go
import (
	"github.com/bits-and-blooms/bloom/v3"
)

// BlacklistBloom 封装布隆过滤器，仅用作"一定不在"快速路径的可选内存加速层
// 主存储必须是 Redis Set（支持精确查找和持久化）
type BlacklistBloom struct {
	filter *bloom.BloomFilter
}

// NewBlacklistBloom 创建布隆过滤器
// expectedItems: 预期元素数量
// falsePositiveRate: 期望误判率，如 0.01 (1%)
func NewBlacklistBloom(expectedItems uint, falsePositiveRate float64) *BlacklistBloom {
	return &BlacklistBloom{
		filter: bloom.NewWithEstimates(expectedItems, falsePositiveRate),
	}
}

// Add 将条目加入布隆过滤器（同时应写入 Redis Set）
func (b *BlacklistBloom) Add(entry string) {
	b.filter.Add([]byte(entry))
}

// MightExist 快速检查是否"可能"在黑名单中
// 返回 false → 一定不在黑名单中，直接放行（fast-path）
// 返回 true  → 可能在黑名单中，必须查询 Redis Set 确认
func (b *BlacklistBloom) MightExist(entry string) bool {
	return b.filter.Test([]byte(entry))
}

// Count 返回已添加的元素近似数量
func (b *BlacklistBloom) Count() uint32 {
	return b.filter.ApproximatedSize()
}

// 使用示例（双层架构）：
//
// 1. 主存储：Redis Set（带 tenant 前缀实现多租户隔离）
//    redis.SAdd(ctx, fmt.Sprintf("risk:tenant:%s:blacklist:ip", tenantID), "192.168.1.100")
//    redis.SIsMember(ctx, fmt.Sprintf("risk:tenant:%s:blacklist:ip", tenantID), userIP)
//
// 2. 辅助层：布隆过滤器（内存加速 "一定不在" fast-path）
//    bloom := NewBlacklistBloom(1_000_000, 0.01)
//    bloom.Add("192.168.1.100")
//
//    if !bloom.MightExist(userIP) {
//        // 一定不在黑名单，直接放行（零 Redis 查询）
//        return "pass"
//    }
//    // 可能在，回查 Redis Set 确认
//    if redis.SIsMember(ctx, fmt.Sprintf("risk:tenant:%s:blacklist:ip", tenantID), userIP) {
//        return "block"
//    }
//
// 3. 持久化：布隆过滤器重启后需从 Redis 全量重建
//    entries := redis.SMembers(ctx, fmt.Sprintf("risk:tenant:%s:blacklist:ip", tenantID))
//    for _, e := range entries { bloom.Add(e) }
```

### 风险评分引擎

> **设计原则**: 每条规则拥有独立的 `Weight` 字段作为命中后贡献的分数。
> 最终动作由**累计总分**决定，而非由最高 priority 或最高 severity 的规则直接覆盖。
> 禁止使用 `ruleScore(action)` 将 action 映射回分数——这会导致因果倒置（Action 是结果，不是输入）。

```go
import (
	"context"
	"fmt"
	"sort"
	"sync"

	"github.com/expr-lang/expr"
	"github.com/expr-lang/expr/vm"
)

// RiskScoreEngine 多规则风险评分引擎
type RiskScoreEngine struct {
	mu     sync.RWMutex
	rules  []RiskRule
	progMu sync.RWMutex
	cache  map[string]*vm.Program // 预编译的 expr 程序缓存
}

// NewRiskScoreEngine 创建评分引擎，初始化缓存
func NewRiskScoreEngine(rules []RiskRule) *RiskScoreEngine {
	return &RiskScoreEngine{
		rules: rules,
		cache: make(map[string]*vm.Program),
	}
}

// Evaluate 评估并返回综合风险结果
// 最终动作由累计总分通过阈值决定，不由单条规则的 Action 覆盖
func (e *RiskScoreEngine) Evaluate(ctx context.Context, fraudCtx *FraudCheckContext, thresholds RiskThresholds) RiskDecision {
	// 按优先级排序
	e.mu.RLock()
	sorted := make([]RiskRule, len(e.rules))
	copy(sorted, e.rules)
	e.mu.RUnlock()

	sort.Slice(sorted, func(i, j int) bool {
		return sorted[i].Priority > sorted[j].Priority
	})

	var totalScore int
	var triggeredIDs []string
	ruleScores := make(map[string]int)

	for _, rule := range sorted {
		if rule.Status == 0 {
			continue
		}
		matched, err := e.evaluateRule(ctx, rule, fraudCtx)
		if err != nil || !matched {
			continue
		}

		// 使用规则自身的 Weight 作为贡献分数（不是由 Action 推导）
		totalScore += rule.Weight
		triggeredIDs = append(triggeredIDs, rule.RuleID)
		ruleScores[rule.RuleID] = rule.Weight
	}

	// 限制总分不超过 100
	if totalScore > 100 {
		totalScore = 100
	}

	// 最终动作由累计总分通过租户阈值决定
	action := decideActionByScore(totalScore, thresholds)

	return RiskDecision{
		RiskScore:      totalScore,
		Action:         action,
		TriggeredRules: triggeredIDs,
		RuleScores:     ruleScores,
	}
}

// evaluateRule 单条规则求值（使用缓存避免重复编译）
func (e *RiskScoreEngine) evaluateRule(ctx context.Context, rule RiskRule, fraudCtx *FraudCheckContext) (bool, error) {
	// 先查缓存
	e.progMu.RLock()
	prog, ok := e.cache[rule.RuleID]
	e.progMu.RUnlock()
	if ok {
		output, err := expr.Run(prog, fraudCtx)
		if err != nil {
			return false, err
		}
		if matched, ok := output.(bool); ok {
			return matched, nil
		}
		return false, nil
	}

	// 缓存未命中，编译并缓存（双重检查锁定）
	prog, err := expr.Compile(rule.Condition, expr.Env(FraudCheckContext{}))
	if err != nil {
		return false, fmt.Errorf("compile rule %s: %w", rule.RuleID, err)
	}
	e.progMu.Lock()
	if existing, found := e.cache[rule.RuleID]; found {
		prog = existing // 使用其他 goroutine 已编译的结果
	} else {
		e.cache[rule.RuleID] = prog
	}
	e.progMu.Unlock()

	output, err := expr.Run(prog, fraudCtx)
	if err != nil {
		return false, err
	}
	if matched, ok := output.(bool); ok {
		return matched, nil
	}
	return false, nil
}

// decideActionByScore 根据累计总分和租户阈值决定最终动作
func decideActionByScore(score int, thresholds RiskThresholds) string {
	switch {
	case score >= thresholds.BlockScore:
		return "block"
	case score >= thresholds.ReviewScore:
		return "review"
	default:
		return "pass"
	}
}

// RiskDecision 最终风控决策
type RiskDecision struct {
	RiskScore      int            `json:"risk_score"`
	Action         string         `json:"action"`          // block / review / pass
	TriggeredRules []string       `json:"triggered_rules"`
	RuleScores     map[string]int `json:"rule_scores"`
}
```

### 支付风控 — 频率检查与金额异常检测

> **多租户隔离**: 所有 Redis key 必须带 `tenant:{tenantID}` 前缀，防止不同租户间数据泄漏。
> 风控规则也应支持租户级配置（thresholds、规则启用/禁用、权重均可按租户覆盖默认值）。

```go
import (
	"context"
	"fmt"
	"math"
	"time"

	"github.com/go-redis/redis_rate/v10"
	"github.com/redis/go-redis/v9"
)

// PaymentRiskChecker 支付风控检查器
type PaymentRiskChecker struct {
	rateLimiter *redis_rate.Limiter
	rdb         *redis.Client
}

// VelocityCheckResult 频率检查结果
type VelocityCheckResult struct {
	LimitExceeded bool
	CurrentCount  int
	Limit         int
	Window        time.Duration
}

// CheckPaymentVelocity 检查用户支付频率（滑动窗口限流）
// Redis key 带 tenantID 前缀以实现多租户隔离
func (c *PaymentRiskChecker) CheckPaymentVelocity(ctx context.Context, tenantID string, userID int64) (*VelocityCheckResult, error) {
	key := fmt.Sprintf("risk:tenant:%s:payment_velocity:user:%d", tenantID, userID)

	// 1 小时内最多允许 5 次支付（该阈值应从租户配置读取）
	result, err := c.rateLimiter.Allow(ctx, key, redis_rate.PerHour(5))
	if err != nil {
		return nil, fmt.Errorf("rate limit check: %w", err)
	}

	return &VelocityCheckResult{
		LimitExceeded: result.Remaining == 0,
		CurrentCount:  result.Limit.Rate - result.Remaining,
		Limit:         result.Limit.Rate,
		Window:        time.Hour,
	}, nil
}

// DetectAmountAnomaly 检测金额异常（基于 Z-Score）
// 将当前订单金额与用户历史均值的偏离度做比较
// 参数使用 int64（单位：分），内部转为 float64 做统计计算
func DetectAmountAnomaly(currentAmount, historyAvg, historyStdDev int64) (bool, float64) {
	if historyStdDev == 0 {
		// 无历史数据或历史金额一致，无法计算偏离度
		return false, 0
	}

	zScore := math.Abs(float64(currentAmount-historyAvg)) / float64(historyStdDev)
	// Z-Score > 3 视为异常（99.7% 置信区间外）
	return zScore > 3.0, zScore
}

// PaymentRiskCheck 综合支付风控
type PaymentRiskCheck struct {
	TenantID        string  // 租户 ID，用于多租户隔离
	UserID          int64
	Amount          int64   // 订单金额（单位：分）
	HistoryAvg      int64   // 历史均值（单位：分）
	HistoryStdDev   int64   // 历史标准差（单位：分）
}

func (c *PaymentRiskChecker) EvaluatePayment(ctx context.Context, check *PaymentRiskCheck, thresholds RiskThresholds) RiskDecision {
	var triggered []string
	ruleScores := make(map[string]int)
	totalScore := 0

	// 1. 频率检查
	velocity, err := c.CheckPaymentVelocity(ctx, check.TenantID, check.UserID)
	if err == nil && velocity.LimitExceeded {
		totalScore += 40
		ruleScores["PAYMENT_VELOCITY_EXCEEDED"] = 40
		triggered = append(triggered, "PAYMENT_VELOCITY_EXCEEDED")
	}

	// 2. 金额异常检测
	anomaly, zScore := DetectAmountAnomaly(check.Amount, check.HistoryAvg, check.HistoryStdDev)
	if anomaly {
		score := 30
		if zScore > 5.0 {
			score = 50 // 极度异常
		}
		totalScore += score
		ruleScores["PAYMENT_AMOUNT_ANOMALY"] = score
		triggered = append(triggered, "PAYMENT_AMOUNT_ANOMALY")
	}

	// 阈值决策（使用租户配置）
	action := decideActionByScore(totalScore, thresholds)

	return RiskDecision{
		RiskScore:      totalScore,
		Action:         action,
		TriggeredRules: triggered,
		RuleScores:     ruleScores,
	}
}
```

---

## Anti-Patterns（反模式）

| 反模式 | 正确做法 |
|--------|----------|
| 边界情况直接 block，没有人工审核环节 | 使用 "review" 动作，给审核人员介入空间，不要只有 block/pass 二元选择 |
| 没有风控审计日志 | 每次决策必须记录 rule_id、risk_score、触发规则列表，便于追溯和复盘 |
| 规则变更需要重新部署代码 | 使用 expr 引擎做动态规则求值，规则存数据库，支持热更新 |
| 只检查单一维度（只看 IP 或只看设备） | 多维度评分：IP + 设备指纹 + 行为模式 + 历史画像，综合决策 |

```go
// 反模式：二元 block/pass，没有 review 中间态
func AntiPattern_Decide(score int) string {
	if score > 60 {
		return "block" // 太粗暴
	}
	return "pass"
}

// 正确：三级决策
func Correct_Decide(score int) string {
	switch {
	case score >= 80:
		return "block"    // 高危，直接拦截
	case score >= 40:
		return "review"   // 中危，转人工审核
	default:
		return "pass"     // 低危，放行
	}
}
```

---

## 常见踩坑点

| 踩坑场景 | 解决方案 |
|----------|----------|
| 误杀正常用户 | 渐进式评分而非二元拦截；提供申诉通道；review 状态优先于 block |
| 风控检查拖慢主链路性能 | Bloom Filter 预计算、规则结果缓存、非关键规则异步执行 |
| NAT/代理导致多用户共享 IP | 使用设备指纹（IP + UA + 屏幕分辨率 + 时区 hash），不能仅依赖 IP |
| GDPR/数据合规要求 | 风控数据超过保留期后必须匿名化；日志中不存储原始设备指纹 |

```go
// RiskEventAnonymizer 定义风控事件匿名化仓库接口
type RiskEventAnonymizer interface {
	// AnonymizeExpired 在 DB 层匿名化过期数据（非内存操作）
	AnonymizeExpired(ctx context.Context, tenantID string, cutoff time.Time) (int64, error)
}

// AnonymizeExpiredEvents 执行 GDPR 合规的数据匿名化
//
// 重要警告：
// 1. 仅在内存中修改 RiskEvent 对象 **不能** 满足 GDPR 要求。
//    GDPR 要求持久化存储中的个人数据也必须被删除或匿名化。
//    必须通过 DB-level UPDATE/DELETE 操作实际修改数据库记录。
// 2. 内存中的修改仅用于防止当前运行中的内存快照泄漏。
// 3. 必须同步处理日志、备份、导出副本中的过期数据。
func AnonymizeExpiredEvents(ctx context.Context, repo RiskEventAnonymizer, tenantID string, retentionDays int) (int64, error) {
	cutoff := time.Now().AddDate(0, 0, -retentionDays)
	count, err := repo.AnonymizeExpired(ctx, tenantID, cutoff)
	if err != nil {
		return 0, fmt.Errorf("anonymize expired events: %w", err)
	}
	// DB 层匿名化 SQL 示例：
	// UPDATE risk_events
	// SET ip = '0.0.0.0', device_identifier = 'ANONYMIZED',
	//     user_id = 0, order_id = 'ANONYMIZED'
	// WHERE tenant_id = ? AND timestamp < ?
	return count, nil
}
```

## 测试策略

### 覆盖率目标

| 层级 | 覆盖率要求 | 说明 |
|------|-----------|------|
| L1 Unit | >= 85% | 规则预编译、expr 评估、布隆过滤器、金额异常检测、评分引擎 |
| L2 Integration | >= 75% | 规则引擎端到端（规则命中→评分→等级判断）、Bloom Filter 与 Redis 黑名单联动 |
| L3 E2E | 核心流程 | 用户下单 → 风控检查 → 评分 → 等级判定（放行/验证/拦截） |

### RuleEngine 预编译缓存测试

```go
import (
	"context"
	"testing"
)

func TestRuleEngine_PrecompiledCache(t *testing.T) {
	rules := []RiskRule{
		{RuleID: "R001", Condition: "OrderAmount > 10000", Status: 1},
		{RuleID: "R002", Condition: "AccountAgeDays < 7", Status: 1},
		{RuleID: "R003", Condition: "invalid expr !!!", Status: 1},
	}

	// 无效规则应返回编译错误
	_, err := NewRuleEngine(rules)
	if err == nil {
		t.Fatal("expected compile error for invalid rule")
	}

	// 有效规则应预编译成功
	validRules := rules[:2]
	engine, err := NewRuleEngine(validRules)
	if err != nil {
		t.Fatalf("NewRuleEngine: %v", err)
	}

	// EvaluateAllRules 不应再调用 expr.Compile
	ctx := context.Background()
	fraudCtx := &FraudCheckContext{OrderAmount: 15000, AccountAgeDays: 10}
	triggered, err := engine.EvaluateAllRules(ctx, fraudCtx)
	if err != nil {
		t.Fatalf("EvaluateAllRules: %v", err)
	}
	if len(triggered) != 1 {
		t.Errorf("expected 1 triggered rule (OrderAmount > 10000), got %d", len(triggered))
	}
}
```

### 布隆过滤器误判率测试

```go
import (
	"fmt"
	"testing"
)

func TestBlacklistBloom_FalsePositiveRate(t *testing.T) {
	bf := NewBlacklistBloom(10000, 0.01) // 1 万容量, 1% 误判率
	// 添加 1000 个黑名单 IP
	for i := 0; i < 1000; i++ {
		bf.Add(fmt.Sprintf("10.0.%d.%d", i/256, i%256))
	}
	// 测试 1000 个非黑名单 IP
	falsePositives := 0
	for i := 0; i < 1000; i++ {
		ip := fmt.Sprintf("192.168.%d.%d", i/256, i%256)
		if bf.MightExist(ip) {
			falsePositives++
		}
	}
	rate := float64(falsePositives) / 1000
	if rate > 0.05 { // 实测误判率应接近理论值 1%
		t.Errorf("false positive rate %.2f exceeds threshold 5%%", rate)
	}
}
```

### 评分引擎边界测试

```go
import "testing"

func TestDecideActionByScore_BoundaryConditions(t *testing.T) {
	thresholds := DefaultThresholds // BlockScore=80, ReviewScore=40

	tests := []struct {
		name       string
		score      int
		wantAction string
	}{
		{"score=0 -> pass", 0, "pass"},
		{"score=39 -> pass", 39, "pass"},
		{"score=40 -> review", 40, "review"},
		{"score=79 -> review", 79, "review"},
		{"score=80 -> block", 80, "block"},
		{"score=100 -> block", 100, "block"},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := decideActionByScore(tt.score, thresholds)
			if got != tt.wantAction {
				t.Errorf("score=%d: got %q, want %q", tt.score, got, tt.wantAction)
			}
		})
	}
}
```

### 多租户隔离测试

```go
import "testing"

func TestRiskControl_MultiTenantIsolation(t *testing.T) {
	// Test 1: ResolveThresholds 返回默认值当租户配置为 nil
	defaults := ResolveThresholds(nil)
	if defaults.OrderFreqCount != DefaultThresholds.OrderFreqCount {
		t.Errorf("nil tenant should return defaults")
	}

	// Test 2: 租户覆盖不修改全局默认值
	tenantA := &RiskThresholds{OrderFreqCount: 10}
	resolvedA := ResolveThresholds(tenantA)
	if resolvedA.OrderFreqCount != 10 {
		t.Errorf("tenantA override not applied")
	}
	defaults2 := ResolveThresholds(nil)
	if defaults2.OrderFreqCount != DefaultThresholds.OrderFreqCount {
		t.Errorf("tenantA override leaked to global defaults")
	}

	// Test 3: 零值字段不被当作覆盖
	tenantB := &RiskThresholds{BlockScore: 0, ReviewScore: 50}
	resolvedB := ResolveThresholds(tenantB)
	if resolvedB.BlockScore != DefaultThresholds.BlockScore {
		t.Errorf("zero BlockScore should not override default")
	}
	if resolvedB.ReviewScore != 50 {
		t.Errorf("non-zero ReviewScore should be applied")
	}
}
```

### 设备标识符边缘测试

```go
import (
	"net/http/httptest"
	"testing"
)

func TestExtractClientIP(t *testing.T) {
	tests := []struct {
		name   string
		xff    string
		xri    string
		remote string
		wantIP string
	}{
		{"xff single", "1.2.3.4", "", "[::1]:5555", "1.2.3.4"},
		{"xff multiple take first", "1.2.3.4, 5.6.7.8, 9.10.11.12", "", ":8080", "1.2.3.4"},
		{"xff with spaces", " 1.2.3.4 , 5.6.7.8 ", "", ":8080", "1.2.3.4"},
		{"xri fallback", "", "10.0.0.1", ":3000", "10.0.0.1"},
		{"remote fallback", "", "", "192.168.1.1:12345", "192.168.1.1:12345"},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			r := httptest.NewRequest("GET", "/", nil)
			if tt.xff != "" {
				r.Header.Set("X-Forwarded-For", tt.xff)
			}
			if tt.xri != "" {
				r.Header.Set("X-Real-IP", tt.xri)
			}
			r.RemoteAddr = tt.remote
			got := extractClientIP(r)
			if got != tt.wantIP {
				t.Errorf("got %q, want %q", got, tt.wantIP)
			}
		})
	}
}

func TestDeviceIdentifier_Deterministic(t *testing.T) {
	r1 := httptest.NewRequest("GET", "/", nil)
	r2 := httptest.NewRequest("GET", "/", nil)
	id1 := DeviceIdentifier(r1)
	id2 := DeviceIdentifier(r2)
	if id1 != id2 {
		t.Errorf("same request should produce same identifier: %s vs %s", id1, id2)
	}
}
```

### 金额异常检测测试

```go
import "testing"

func TestDetectAmountAnomaly(t *testing.T) {
	tests := []struct {
		name        string
		current     int64
		avg         int64
		stdDev      int64
		wantAnomaly bool
	}{
		{"zero stddev -> no anomaly", 100, 50, 0, false},
		{"within 3 sigma", 100, 50, 30, false},
		{"just over 3 sigma", 140, 50, 30, true},
		{"extreme > 5 sigma", 300, 50, 30, true},
		{"negative deviation", 1, 100, 20, true},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			anomaly, _ := DetectAmountAnomaly(tt.current, tt.avg, tt.stdDev)
			if anomaly != tt.wantAnomaly {
				t.Errorf("current=%d avg=%d stdDev=%d: got anomaly=%v, want %v",
					tt.current, tt.avg, tt.stdDev, anomaly, tt.wantAnomaly)
			}
		})
	}
}
```
