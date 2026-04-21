# 内容管理与国际化 (CMS & I18n)

## 内容多语言与富文本处理

---

### 触发条件

- 需要首页配置、Banner 管理
- 富文本内容（商品详情、活动页、帮助文档）
- 多语言商品描述、分类名称
- 货币转换、多币种定价

---

### 推荐第三方库

| 用途 | 推荐库 | 导入路径 |
|------|--------|----------|
| HTML 内容过滤 | bluemonday | `github.com/microcosm-cc/bluemonday` |
| 国际化翻译 | go-i18n | `github.com/nicksnyder/go-i18n/v2` |
| 汇率获取 | go-resty | `github.com/go-resty/resty/v3` |
| 图片处理 | imaging | `github.com/disintegration/imaging` |

## 决策树

```
需要多语言/内容管理？
  ├─ 富文本内容（商品详情/活动页）→ bluemonday 过滤 → 存 DB → CDN 缓存
  ├─ 界面国际化（按钮/提示/错误）→ go-i18n + Accept-Language 解析 → fallback 链
  ├─ 商品多语言描述 → 独立翻译表，按 language_code 隔离，优先取对应语言
  ├─ 多币种定价 → 基准价(CNY) + 汇率表 + 各币种精度舍入
  ├─ 图片处理 → 上传时自动裁剪多尺寸 + WebP 转换 → CDN 分发
  └─ 汇率更新 → 定时任务从外部 API 获取 → 更新汇率表 + 缓存失效
```

---

### 代码模板

#### 3.1 Banner 模型

```go
// model/banner.go
package model

import (
	"time"

	"gorm.io/gorm"
)

// Banner 首页/活动页轮播 Banner
type Banner struct {
	ID        uint           `gorm:"primaryKey" json:"id"`
	TenantID  int64          `gorm:"index:idx_tenant_id;not null" json:"tenant_id"` // 多租户隔离
	Title     string         `gorm:"size:255;not null" json:"title"`
	ImageURL  string         `gorm:"size:1024;not null" json:"image_url"`
	LinkURL   string         `gorm:"size:1024" json:"link_url"`
	StartTime time.Time      `gorm:"not null" json:"start_time"`
	EndTime   time.Time      `gorm:"not null" json:"end_time"`
	SortOrder int            `gorm:"default:0" json:"sort_order"`
	Status    int8           `gorm:"default:1;check:status IN (0,1)" json:"status"` // 0=下架, 1=上架
	Language  string         `gorm:"size:10;default:'zh-CN'" json:"language"`
	CreatedAt time.Time      `json:"created_at"`
	UpdatedAt time.Time      `json:"updated_at"`
	DeletedAt gorm.DeletedAt `gorm:"index" json:"-"`
}

func (Banner) TableName() string {
	return "banners"
}
```

#### 3.2 CMS 页面模型

```go
// model/cms_page.go
package model

import (
	"time"

	"gorm.io/gorm"
)

// CMSPage CMS 内容页面（帮助页、活动页、关于我们等）
type CMSPage struct {
	ID        uint           `gorm:"primaryKey" json:"id"`
	TenantID  int64          `gorm:"index:idx_tenant_id;not null" json:"tenant_id"` // 多租户隔离
	Slug      string         `gorm:"uniqueIndex:idx_tenant_slug_lang,where:deleted_at IS NULL;size:255;not null" json:"slug"`      // URL 路径标识
	Title     string         `gorm:"size:255;not null" json:"title"`
	Content   string         `gorm:"type:mediumtext" json:"content"`                  // HTML 内容
	MetaTags  string         `gorm:"type:text" json:"meta_tags"`                      // JSON: {"keywords":[], "description":""}
	Language  string         `gorm:"size:10;uniqueIndex:idx_tenant_slug_lang,where:deleted_at IS NULL;not null" json:"language"`
	Version   int            `gorm:"default:1" json:"version"`                        // 乐观锁 / 版本号
	Status    int8           `gorm:"default:0;check:status IN (0,1)" json:"status"`   // 0=草稿, 1=发布
	CreatedAt time.Time      `json:"created_at"`
	UpdatedAt time.Time      `json:"updated_at"`
	DeletedAt gorm.DeletedAt `gorm:"index" json:"-"`
}

func (CMSPage) TableName() string {
	return "cms_pages"
}
```

#### 3.3 多语言商品描述

```go
// model/product_i18n.go
package model

import "time"

// ProductI18n 商品多语言翻译表
type ProductI18n struct {
	ID                  uint      `gorm:"primaryKey" json:"id"`
	TenantID            int64     `gorm:"uniqueIndex:idx_tenant_product_lang;not null" json:"tenant_id"` // 多租户隔离
	ProductID           int64     `gorm:"uniqueIndex:idx_tenant_product_lang;not null" json:"product_id"`
	LanguageCode        string    `gorm:"size:10;uniqueIndex:idx_tenant_product_lang;not null" json:"language_code"` // zh-CN, en-US, ja-JP
	TranslatedTitle     string    `gorm:"size:500;not null" json:"translated_title"`
	TranslatedDescription string  `gorm:"type:mediumtext" json:"translated_description"` // 富文本 HTML
	CreatedAt           time.Time `json:"created_at"`
	UpdatedAt           time.Time `json:"updated_at"`
}

func (ProductI18n) TableName() string {
	return "product_i18n"
}

// SupportedLanguage 支持的语言列表
var SupportedLanguage = []string{
	"zh-CN", // 简体中文
	"en-US", // 英文（美国）
	"ja-JP", // 日文
	"ko-KR", // 韩文
	"th-TH", // 泰文
	"vi-VN", // 越南文
	"ar-SA", // 阿拉伯文（沙特）
}
```

#### 3.4 货币转换模型

```go
// model/currency.go
package model

import "time"

// Currency 币种定义
type Currency struct {
	Code         string  `gorm:"primaryKey;size:3" json:"code"`           // ISO 4217: CNY, USD, JPY
	Name         string  `gorm:"size:50;not null" json:"name"`
	Symbol       string  `gorm:"size:10" json:"symbol"`                   // ¥, $, Rp
	DecimalPlace int     `gorm:"not null" json:"decimal_place"`           // CNY=2, JPY=0, KWD=3
	IsActive     bool    `gorm:"default:true" json:"is_active"`
}

func (Currency) TableName() string {
	return "currencies"
}

// ExchangeRate 汇率表（以 CNY 为基准）
type ExchangeRate struct {
	ID          uint      `gorm:"primaryKey" json:"id"`
	TenantID    int64     `gorm:"uniqueIndex:idx_tenant_from_to;not null" json:"tenant_id"` // 多租户隔离
	FromCode    string    `gorm:"size:3;not null" json:"from_code"` // 基准币种，固定为 CNY
	ToCode      string    `gorm:"size:3;uniqueIndex:idx_tenant_from_to;not null" json:"to_code"`
	Rate        float64   `gorm:"type:decimal(18,8);not null" json:"rate"` // 1 CNY = ? TO（NOTE: float64 is acceptable for exchange rates — not monetary amounts. DB column should be decimal(18,8). Intermediate calculations for display prices are OK.)
	LastUpdated time.Time `gorm:"not null" json:"last_updated"`
	CreatedAt   time.Time `json:"created_at"`
	UpdatedAt   time.Time `json:"updated_at"`
}

func (ExchangeRate) TableName() string {
	return "exchange_rates"
}

// DecimalPrecision 各币种小数精度
var DecimalPrecision = map[string]int{
	"JPY": 0, "KRW": 0, "VND": 0,        // 0 位
	"CNY": 2, "USD": 2, "EUR": 2, "GBP": 2, "THB": 2, // 2 位（主流）
	"KWD": 3, "BHD": 3, "OMR": 3,        // 3 位
}
```

#### 3.5 go-i18n 集成

```go
// i18n/i18n.go
// NOTE: Package name `i18n` shadows `github.com/nicksnyder/go-i18n/v2/i18n` in imports.
// This is intentional — callers qualify the import (e.g., `goi18n "github.com/nicksnyder/go-i18n/v2/i18n"`)
// to avoid collision. Alternatively, rename this package to `translation` to eliminate shadowing.
package i18n

import (
	"context"
	"embed"
	"fmt"
	"net/http"
	"sync"

	"github.com/BurntSushi/toml"
	"github.com/nicksnyder/go-i18n/v2/i18n"
	"github.com/zeromicro/go-zero/core/logx"
	"golang.org/x/text/language"
)

//go:embed locales/*.toml
var localeFS embed.FS

var (
	once   sync.Once
	bundle *i18n.Bundle
)

// Init 加载翻译文件
func Init() error {
	var err error
	once.Do(func() {
		bundle = i18n.NewBundle(language.Chinese) // 默认语言：中文
		bundle.RegisterUnmarshalFunc("toml", toml.Unmarshal)

		// 加载各语言翻译文件
		files := map[string]string{
			"zh-CN": "zh-CN.toml",
			"en-US": "en-US.toml",
			"ja-JP": "ja-JP.toml",
		}
		for tag, file := range files {
			_, err = bundle.LoadMessageFileFS(localeFS, "locales/"+file)
			if err != nil {
				err = fmt.Errorf("load i18n file %s: %w", tag, err)
				return
			}
		}
		logx.WithContext(context.Background()).Infof("i18n initialized with %d languages", len(files))
	})
	return err
}

// ParseAcceptLanguage 从 Accept-Language 请求头解析首选语言
func ParseAcceptLanguage(header string) string {
	tags, _, err := language.ParseAcceptLanguage(header)
	if err != nil || len(tags) == 0 {
		return "zh-CN" // fallback 默认
	}

	supported := map[string]bool{
		"zh-CN": true, "zh-TW": true,
		"en-US": true, "en-GB": true,
		"ja-JP": true,
		"ko-KR": true,
		"th-TH": true,
		"vi-VN": true,
		"ar-SA": true,
	}

	for _, tag := range tags {
		base := tag.String()
		if supported[base] {
			return base
		}
	}
	return "zh-CN"
}

// Localize 获取翻译文本（带 fallback 链）
// Fallback 链：请求语言 → 默认语言(zh-CN) → 原始内容(key 本身)
func Localize(messageID string, lang string, args ...any) string {
	loc := i18n.NewLocalizer(bundle, lang, "zh-CN")
	msg, err := loc.Localize(&i18n.LocalizeConfig{
		MessageID: messageID,
	})
	if err != nil {
		// fallback: 返回 messageID 本身作为兜底
		return messageID
	}
	return msg
}

// GetLocalizer 为 HTTP 请求创建 localizer（含 fallback 链）
func GetLocalizer(lang string) *i18n.Localizer {
	return i18n.NewLocalizer(bundle, lang, "zh-CN")
}
```

翻译文件示例 (`en-US.toml`)：

```toml
[product.not_found]
other = "Product not found"

[product.add_to_cart]
other = "Add to Cart"

[currency.price]
other = "Price: {{.Price}} {{.Currency}}"

[error.validation_failed]
other = "Validation failed: {{.Errors}}"
```

#### 3.6 富文本消毒（bluemonday）

```go
// service/sanitize.go
package service

import (
	"github.com/microcosm-cc/bluemonday"
)

// RichTextPolicy 富文本安全策略
// 允许：图片、链接、加粗、斜体、下划线、列表、表格、段落
// 禁止：script、style、事件属性、iframe、object
var RichTextPolicy = func() *bluemonday.Policy {
	p := bluemonday.UGCPolicy()

	// 允许图片（含 data URI）
	p.AllowAttrs("src").OnElements("img")
	p.AllowAttrs("alt", "title").OnElements("img")
	p.AllowAttrs("width", "height").Matching(bluemonday.Integer).OnElements("img")

	// 允许链接（限制协议）
	p.AllowAttrs("href").Matching(
		bluemonday.AllowStandardURLs(),
	).OnElements("a")
	p.AllowAttrs("target").Matching(
		bluemonday.Enum("blank", "self"),
	).OnElements("a")
	p.RequireNoFollowOnLinks(true)
	p.RequireNoFollowOnFullyQualifiedLinks(true)

	// 允许表格
	p.AllowElements("table", "thead", "tbody", "tfoot", "tr", "th", "td")

	// 允许常用格式化标签
	p.AllowElements("strong", "em", "u", "s", "br", "hr", "p", "div", "span")
	p.AllowElements("h1", "h2", "h3", "h4", "h5", "h6")
	p.AllowElements("ul", "ol", "li")
	p.AllowElements("blockquote", "pre", "code")

	// 允许内联样式（仅限安全的 CSS 属性）
	p.AllowStyles("color", "background-color", "font-size", "text-align").
		OnElements("span", "div", "p", "h1", "h2", "h3", "h4", "h5", "h6", "td", "th")

	return p
}()

// SanitizeHTML 消毒 HTML 内容
func SanitizeHTML(input string) string {
	return RichTextPolicy.Sanitize(input)
}

// SanitizePlainText 纯文本消毒（strip 所有 HTML 标签）
func SanitizePlainText(input string) string {
	return bluemonday.StrictPolicy().Sanitize(input)
}
```

#### 3.7 图片上传优化

```go
// service/image_processor.go
package service

import (
	"fmt"
	"image"
	_ "image/jpeg"
	_ "image/png"
	"os"
	"path/filepath"

	"github.com/disintegration/imaging"
)

// ImageSize 预定义图片尺寸
type ImageSize struct {
	Name   string
	Width  int
	Height int
}

var presetSizes = []ImageSize{
	{"thumbnail", 200, 200},   // 缩略图：列表页、推荐位
	{"medium", 600, 600},      // 中等：商品详情页主图
	{"large", 1200, 1200},     // 大图：放大查看、营销素材
}

// ProcessUploadedImage 上传图片时生成多种尺寸 + WebP 转换
func ProcessUploadedImage(srcPath string, destDir string) (map[string]string, error) {
	srcFile, err := os.Open(srcPath)
	if err != nil {
		return nil, fmt.Errorf("open source image: %w", err)
	}
	defer srcFile.Close()

	img, _, err := image.Decode(srcFile)
	if err != nil {
		return nil, fmt.Errorf("decode image: %w", err)
	}

	baseName := filepath.Base(srcPath)
	ext := filepath.Ext(baseName)
	name := baseName[:len(baseName)-len(ext)]

	results := make(map[string]string)

	// 生成各尺寸
	for _, size := range presetSizes {
		resized := imaging.Fill(img, size.Width, size.Height, imaging.Center, imaging.Lanczos)

		// 保存 JPEG
		jpegPath := filepath.Join(destDir, fmt.Sprintf("%s_%s.jpg", name, size.Name))
		if err := imaging.Save(resized, jpegPath); err != nil {
			return nil, fmt.Errorf("save %s: %w", size.Name, err)
		}
		results[size.Name+"_jpg"] = jpegPath

		// WebP 转换：imaging.Save 不支持 WebP 输出。
		// 需使用专门的 WebP 编码库，如 github.com/chai2010/webp 或 github.com/davidbyttow/govips。
		// webpPath := filepath.Join(destDir, fmt.Sprintf("%s_%s.webp", name, size.Name))
		// if err := webp.Encode(resized, webpPath, &webp.Options{Quality: 80}); err != nil {
		//     return nil, fmt.Errorf("save %s webp: %w", size.Name, err)
		// }
		// results[size.Name+"_webp"] = webpPath
	}

	return results, nil
}

// ValidateImageUpload 上传图片前校验
func ValidateImageUpload(filePath string) error {
	stat, err := os.Stat(filePath)
	if err != nil {
		return fmt.Errorf("stat file: %w", err)
	}

	// 限制 5MB（避免上传原始大图导致前端加载慢）
	const maxSize = 5 * 1024 * 1024
	if stat.Size() > maxSize {
		return fmt.Errorf("image size %d exceeds limit %d", stat.Size(), maxSize)
	}

	// 验证是否为合法图片
	f, err := os.Open(filePath)
	if err != nil {
		return err
	}
	defer f.Close()

	_, format, err := image.DecodeConfig(f)
	if err != nil {
		return fmt.Errorf("invalid image format: %w", err)
	}

	allowedFormats := map[string]bool{"jpeg": true, "png": true, "webp": true, "gif": true}
	if !allowedFormats[format] {
		return fmt.Errorf("unsupported format: %s, allowed: jpeg/png/webp/gif", format)
	}

	return nil
}
```

#### 3.8 货币中间件

```go
// middleware/currency.go
package middleware

import (
	"context"
	"encoding/json"
	"fmt"
	"math"
	"net/http"
	"strings"
	"sync"
	"time"

	"github.com/go-resty/resty/v3"
	"github.com/zeromicro/go-zero/core/breaker"
	"github.com/zeromicro/go-zero/core/logx"
	"github.com/zeromicro/go-zero/core/stores/redis"
	"github.com/zeromicro/go-zero/rest"
)

// ctxKey is an unexported context key type to avoid collisions.
// Using string keys (e.g., context.WithValue(ctx, "currency", ...)) is an anti-pattern —
// any package can collide. Always use unexported custom types as context keys.
type ctxKey struct{}

var currencyCtxKey = ctxKey{}

// CurrencyContext 请求上下文中的货币信息
type CurrencyContext struct {
	Code       string  `json:"code"`
	Symbol     string  `json:"symbol"`
	Rate       float64 `json:"rate"`
	LastUpdate string  `json:"last_update"`
}

// ExchangeRateFetcher 汇率获取器（含 Redis 缓存 + go-zero 熔断）
type ExchangeRateFetcher struct {
	mu        sync.RWMutex
	rates     map[string]float64
	updatedAt time.Time
	client    *resty.Client
	rds       *redis.Redis
	tenantID  int64
	breaker   breaker.Breaker
}

// NewExchangeRateFetcher 创建汇率获取器
func NewExchangeRateFetcher(apiBaseURL string, rds *redis.Redis, tenantID int64) *ExchangeRateFetcher {
	return &ExchangeRateFetcher{
		rates:    make(map[string]float64),
		client:   resty.New().SetBaseURL(apiBaseURL).SetTimeout(10 * time.Second),
		rds:      rds,
		tenantID: tenantID,
		breaker:  breaker.NewBreaker(breaker.WithName("exchange_rate")),
	}
}

// redisKey 返回带租户前缀的 Redis key
func (f *ExchangeRateFetcher) redisKey(field string) string {
	return fmt.Sprintf("storeforge:tenant:%d:currency:%s", f.tenantID, field)
}

// RefreshRates 从外部 API 刷新汇率（含 Redis 缓存 + go-zero 熔断）
func (f *ExchangeRateFetcher) RefreshRates(ctx context.Context) error {
	return f.breaker.DoCtx(ctx, func() error {
		return f.doRefreshRates(ctx)
	})
}

func (f *ExchangeRateFetcher) doRefreshRates(ctx context.Context) error {
	// 先查 Redis 缓存
	cacheKey := f.redisKey("rates")
	if f.rds != nil {
		val, err := f.rds.Get(cacheKey)
		if err == nil && val != "" {
			var cached map[string]float64
			if json.Unmarshal([]byte(val), &cached) == nil {
				f.mu.Lock()
				f.rates = cached
				f.updatedAt = time.Now()
				f.mu.Unlock()
				logx.WithContext(ctx).Infof("exchange rates loaded from Redis cache (%d entries)", len(cached))
				return nil
			}
		}
	}

	// 从 API 拉取
	resp, err := f.client.R().WithContext(ctx).Get("/latest?base=CNY")
	if err != nil {
		return err
	}

	var data struct {
		Rates map[string]float64 `json:"rates"`
	}
	if err := json.Unmarshal(resp.Body(), &data); err != nil {
		return err
	}

	// 成功
	f.mu.Lock()
	f.rates = data.Rates
	f.updatedAt = time.Now()
	f.mu.Unlock()

	// 写入 Redis 缓存（24 小时过期）
	if f.rds != nil {
		payload, _ := json.Marshal(data.Rates)
		f.rds.Setex(cacheKey, string(payload), 86400)
	}

	logx.WithContext(ctx).Infof("exchange rates refreshed from API (%d entries)", len(data.Rates))
	return nil
}

// GetRate 获取指定币种对 CNY 的汇率
func (f *ExchangeRateFetcher) GetRate(code string) (float64, bool) {
	f.mu.RLock()
	defer f.mu.RUnlock()
	rate, ok := f.rates[code]
	return rate, ok
}

// LastUpdated 返回汇率最后更新时间
func (f *ExchangeRateFetcher) LastUpdated() time.Time {
	f.mu.RLock()
	defer f.mu.RUnlock()
	return f.updatedAt
}

// CurrencyMiddleware go-zero rest.Middleware — 货币检测与转换
// 转换为 go-zero 模式：实现 rest.Middleware 接口的 Handle 方法
type CurrencyMiddleware struct {
	fetcher *ExchangeRateFetcher
}

func NewCurrencyMiddleware(fetcher *ExchangeRateFetcher) *CurrencyMiddleware {
	return &CurrencyMiddleware{fetcher: fetcher}
}

func (m *CurrencyMiddleware) Handle(next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		ctx := r.Context()

		// 1. 从 Accept-Language 或查询参数检测目标币种
		targetCurrency := r.URL.Query().Get("currency")
		if targetCurrency == "" {
			targetCurrency = detectCurrencyFromLocale(r.Header.Get("Accept-Language"))
		}

		// 2. 获取汇率
		rate, ok := m.fetcher.GetRate(targetCurrency)
		if !ok {
			targetCurrency = "CNY" // fallback 基准币种
			rate = 1.0
		}

		// 3. 注入上下文（使用 unexported context key type，避免 key 碰撞）
		ctx = context.WithValue(ctx, currencyCtxKey, CurrencyContext{
			Code:       targetCurrency,
			Rate:       rate,
			LastUpdate: m.fetcher.LastUpdated().Format(time.RFC3339),
		})
		r = r.WithContext(ctx)

		logx.WithContext(ctx).Debugw("currency context injected",
			logx.Field("currency", targetCurrency),
			logx.Field("rate", rate),
		)

		next(w, r)
	}
}

// GetCurrencyFromContext 从请求上下文中安全提取货币信息
func GetCurrencyFromContext(ctx context.Context) (CurrencyContext, bool) {
	val := ctx.Value(currencyCtxKey)
	cc, ok := val.(CurrencyContext)
	return cc, ok
}

// detectCurrencyFromLocale 从语言地区推断币种
func detectCurrencyFromLocale(acceptLang string) string {
	localeMap := map[string]string{
		"zh-CN": "CNY", "zh-TW": "TWD",
		"en-US": "USD", "en-GB": "GBP",
		"ja-JP": "JPY", "ko-KR": "KRW",
		"th-TH": "THB", "vi-VN": "VND",
		"ar-SA": "SAR",
	}

	for locale, currency := range localeMap {
		if strings.Contains(acceptLang, locale) {
			return currency
		}
	}
	return "CNY" // fallback
}

// ConvertPrice 价格转换（处理各币种精度）
func ConvertPrice(cnyPrice int64, rate float64, code string) int64 {
	decimal := 2
	if d, ok := DecimalPrecision[code]; ok {
		decimal = d
	}

	multiplier := int64(1)
	for i := 0; i < decimal; i++ {
		multiplier *= 10
	}

	// CNY 存储为分（*100），转换为目标币种后按精度截断
	converted := float64(cnyPrice) * rate / 100.0
	return int64(math.Round(converted * float64(multiplier)))
}
```

> **Circuit breaker / degradation guidance**:
>
> - **Implementation**: 使用 `go-zero/core/breaker` 的 `Breaker` 接口，非手动熔断逻辑
> - **Algorithm**: Google SRE 自适应算法（内部 window=10s, buckets=40，不可配置）
> - **Fallback**: 熔断期间使用 Redis 缓存的最后成功汇率；无缓存时 fallback 到 CNY 基准价 (rate=1.0)
> - **Observability**: 熔断状态变更通过 `logx.WithContext(ctx)` 记录 ERROR/WARN 日志，接入告警
> - **Timeout**: resty client 配置 `SetTimeout(10 * time.Second)`，防止外部 API 无限阻塞

---

### 反模式 (Anti-Patterns)

#### 存储 HTML 不做消毒 → XSS 漏洞

```go
// 错误：直接存储用户提交的 HTML
page.Content = req.Content // 可被注入 <script>alert('xss')</script>

// 正确：使用 bluemonday 消毒
page.Content = SanitizeHTML(req.Content)
```

#### 硬编码翻译文本在代码中

```go
// 错误：翻译散落在代码里
msg := "商品已加入购物车" // 无法支持多语言

// 正确：外部化到 YAML/TOML 翻译文件
msg := Localize("product.added_to_cart", lang)
// en-US.toml: [product.added_to_cart] other = "Product added to cart"
```

#### 使用过期汇率做货币转换

```go
// 错误：硬编码汇率，永不更新
rate := 6.5 // CNY to USD，可能已经是半年前的汇率

// 正确：定时刷新 + 展示"最后更新时间"
fetcher.RefreshRates(ctx) // 每 30 分钟执行，传入 ctx
ctx.LastUpdate // 返回给前端展示 "汇率更新于 2026-04-20 14:30"
```

#### 图片不压缩/不缩放直接存储

```go
// 错误：前端 200px 缩略图加载 5MB 原始图
// 用户流量浪费，页面加载慢

// 正确：上传时生成多尺寸
results, _ := ProcessUploadedImage(srcPath, destDir)
// 返回: {"thumbnail": ".../thumb.jpg", "medium": ".../med.jpg", "large": ".../lg.jpg"}
// 列表页用 thumbnail，详情页用 medium，放大查看用 large
```

---

### 常见陷阱 (Common Pitfalls)

#### Fallback 链（统一策略）

所有多语言场景（翻译 key 缺失、语言不支持、汇率不可用）遵循统一的三级降级策略：

```
权威 Fallback 链：
  1. 请求的语言 / 请求的币种 (Accept-Language: ja-JP 或 ?currency=THB)
  2. 默认语言 / 基准币种 (zh-CN / CNY)
  3. 原始内容 / 翻译 key 本身 / rate = 1.0
```

```go
// go-i18n 的 fallback 通过 NewLocalizer 的多参数实现
loc := i18n.NewLocalizer(bundle, "ja-JP", "zh-CN")
// 如果 ja-JP 没有该 key，自动 fallback 到 zh-CN
// Localize 函数实现见 3.5 节
```

#### 货币舍入精度

不同币种有不同小数位数，不能统一用 2 位：

| 币种 | 精度 | 示例 |
|------|------|------|
| JPY  | 0 位 | ¥1,234 |
| CNY  | 2 位 | ¥12.34 |
| USD  | 2 位 | $12.34 |
| KWD  | 3 位 | KWD 12.345 |

```go
// 错误：所有货币统一保留 2 位小数
price := fmt.Sprintf("%.2f", converted) // JPY 显示为 ¥1234.00 → 错误

// 正确：使用 DecimalPrecision 映射
decimal := DecimalPrecision[code] // JPY=0, CNY=2, KWD=3
price := fmt.Sprintf("%.*f", decimal, converted)
```

#### RTL 语言（阿拉伯文/希伯来文）布局方向

```go
// 内容必须标记语言方向
type LocalizedContent struct {
	Language    string `json:"language"`
	Direction   string `json:"direction"` // "ltr" 或 "rtl"
	Content     string `json:"content"`
}

func GetDirection(lang string) string {
	rtlLanguages := map[string]bool{
		"ar-SA": true, "ar-EG": true, "ar-AE": true,
		"he-IL": true, "fa-IR": true, "ur-PK": true,
	}
	if rtlLanguages[lang] {
		return "rtl"
	}
	return "ltr"
}
```

前端 CSS 处理：

```css
/* LTR 默认布局 */
.product-card { direction: ltr; }

/* RTL 语言自动翻转 */
[dir="rtl"] .product-card {
  direction: rtl;
  text-align: right;
}

/* 图标/箭头等需手动翻转 */
[dir="rtl"] .arrow-icon {
  transform: scaleX(-1);
}
```

---

### 测试策略 (Test Strategy)

#### ConvertPrice 边界用例

```go
func TestConvertPrice(t *testing.T) {
	tests := []struct {
		name     string
		cnyPrice int64
		rate     float64
		code     string
		want     int64
	}{
		{"JPY zero decimal", 10000, 1.5, "JPY", 1500},    // 100.00 CNY → JPY (0 decimals)
		{"CNY standard", 10000, 1.0, "CNY", 1000},        // no conversion
		{"USD 2 decimal", 10000, 0.14, "USD", 140},       // 100.00 CNY → USD
		{"KWD 3 decimal", 10000, 0.043, "KWD", 43},       // 100.00 CNY → KWD
		{"unknown code defaults to 2", 10000, 1.0, "XXX", 1000}, // unknown → 2 decimal
		{"zero price", 0, 1.5, "USD", 0},                  // zero price edge case
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := ConvertPrice(tt.cnyPrice, tt.rate, tt.code)
			if got != tt.want {
				t.Errorf("ConvertPrice() = %d, want %d", got, tt.want)
			}
		})
	}
}

func TestConvertPrice_NoOverflow(t *testing.T) {
	// Large price to verify no int64 overflow
	result := ConvertPrice(1_000_000_000, 100.0, "JPY") // 10M CNY → JPY
	if result < 0 {
		t.Errorf("overflow detected: result = %d (negative)", result)
	}
}
```

#### SanitizeHTML XSS 防护

```go
func TestSanitizeHTML_XSS(t *testing.T) {
	tests := []struct {
		name  string
		input string
		check func(string) bool // returns true if sanitized (dangerous content removed)
	}{
		{
			"script injection",
			"<script>alert('xss')</script><p>Hello</p>",
			func(s string) bool { return !strings.Contains(s, "<script>") },
		},
		{
			"onerror event handler",
			`<img src=x onerror="alert(1)">`,
			func(s string) bool { return !strings.Contains(s, "onerror") },
		},
		{
			"javascript: protocol in href",
			`<a href="javascript:alert(1)">click</a>`,
			func(s string) bool { return !strings.Contains(s, "javascript:") },
		},
		{
			"iframe injection",
			`<iframe src="https://evil.com"></iframe>`,
			func(s string) bool { return !strings.Contains(s, "<iframe>") },
		},
		{
			"data URI script",
			`<img src="data:text/html,<script>alert(1)</script>">`,
			func(s string) bool { return !strings.Contains(s, "<script>") },
		},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			result := SanitizeHTML(tt.input)
			if !tt.check(result) {
				t.Errorf("SanitizeHTML(%q) = %q — XSS payload not sanitized", tt.input, result)
			}
		})
	}
}

func TestSanitizeHTML_AllowsValid(t *testing.T) {
	input := `<p>Hello <strong>world</strong></p>`
	result := SanitizeHTML(input)
	if !strings.Contains(result, "<strong>") {
		t.Errorf("valid HTML stripped: got %q", result)
	}
}
```

#### ParseAcceptLanguage 各种请求头

```go
func TestParseAcceptLanguage(t *testing.T) {
	tests := []struct {
		name   string
		header string
		want   string
	}{
		{"single language", "en-US", "en-US"},
		{"multiple with quality", "ja-JP;q=0.9, en-US;q=0.8, zh-CN;q=0.7", "ja-JP"},
		{"unsupported falls back", "fr-FR, de-DE", "zh-CN"},
		{"empty header", "", "zh-CN"},
		{"wildcard", "*", "zh-CN"},
		{"zh-CN first", "zh-CN, en-US;q=0.9", "zh-CN"},
		{"unsupported prefers default", "ru-RU, ar-SA;q=0.5", "ar-SA"}, // ar-SA is supported
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := ParseAcceptLanguage(tt.header)
			if got != tt.want {
				t.Errorf("ParseAcceptLanguage(%q) = %q, want %q", tt.header, got, tt.want)
			}
		})
	}
}
```
