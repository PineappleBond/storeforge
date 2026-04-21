# 物流配送模式 (Shipping Patterns)

## 触发条件

- 需要对接第三方物流服务商（顺丰/申通/圆通/中通等）
- 需要计算运费（按重量、按地区、按商品）
- 需要生成电子面单
- 需要查询物流轨迹
- 需要处理发货、签收、异常等物流状态

## 推荐第三方库

| 用途 | 推荐库 | 导入路径 |
|------|--------|----------|
| HTTP 客户端（物流 API 调用） | go-resty | `github.com/go-resty/resty/v3` |
| 重试与退避 | backoff | `github.com/cenkalti/backoff/v4` |
| 限流 | rate limiter | `golang.org/x/time/rate` |
| 字符串操作 | Go 标准库 | `strings` 包（禁止手写 findSubstring 等函数） |

**为什么用 resty + backoff：** 手写 HTTP 重试循环容易出错（线性退避 vs 指数退避、忘记重试幂等判断）。`resty` 内置重试和超时配置，`backoff` 提供指数退避 + jitter。

## 决策树

```
需要发货功能
  │
  ├─ 运费计算
  │   ├─ 统一运费？→ 固定运费模板
  │   ├─ 按重量计费？→ 重量 × 单价（首重 + 续重）
  │   ├─ 按地区计费？→ 地区运费矩阵
  │   └─ 组合计费？→ 重量 + 地区 + 商品类型
  │
  ├─ 物流公司选择
  │   ├─ 顺丰 → 高价值/急件（费用高，速度快）
  │   ├─ 中通/圆通/申通 → 普通件（费用低，速度中等）
  │   └─ 京东物流 → 京东自营/仓配一体
  │
  ├─ 面单生成方式
  │   ├─ 电子面单 API → 实时获取运单号 + 打印面单
  │   └─ 手动录入运单号 → 线下打单，线上录入
  │
  └─ 物流轨迹查询
      ├─ 主动轮询 → 定时调用物流公司 API（每 2 小时）
      └─ 回调推送 → 物流公司 webhook 通知状态变更
```

## 代码模板

### 1. 运费计算模型

```go
package shipping

import (
	"context"
	"errors"
	"fmt"
)

// ShippingFeeCalculator 运费计算器接口
type ShippingFeeCalculator interface {
	Calculate(ctx context.Context, req FeeCalculateRequest) (FeeCalculateResult, error)
}

// FeeCalculateRequest 运费计算请求
type FeeCalculateRequest struct {
	ProvinceCode string  `json:"province_code"` // 省份编码（如 "110000" 北京）
	CityCode     string  `json:"city_code"`     // 城市编码
	DistrictCode string  `json:"district_code"` // 区县编码
	WeightGram   int64   `json:"weight_gram"`   // 总重量（克）
	VolumeCm3    int64   `json:"volume_cm3"`    // 总体积（立方厘米）
	ItemCount    int     `json:"item_count"`    // 商品件数
	TenantID     string  `json:"tenant_id"`     // 所属租户/店铺
	CarrierCode  string  `json:"carrier_code"`  // 指定物流商（可选）
}

// FeeCalculateResult 运费计算结果
type FeeCalculateResult struct {
	Fee         int64  `json:"fee"`          // 运费（分）
	CarrierCode string `json:"carrier_code"` // 物流商编码
	FirstWeight int64  `json:"first_weight"` // 首重（克）
	FirstFee    int64  `json:"first_fee"`    // 首重费用（分）
	ExtraWeight int64  `json:"extra_weight"` // 续重（克）
	ExtraFee    int64  `json:"extra_fee"`    // 续重费用（分）
	Breakdown   string `json:"breakdown"`    // 费用明细（用于展示）
}

// ShippingTemplate 运费模板
type ShippingTemplate struct {
	ID           string `gorm:"primaryKey;type:varchar(64)"`
	TenantID     string `gorm:"type:varchar(64);not null;index"` // 多租户隔离
	Name         string `gorm:"type:varchar(128);not null"`
	CarrierCode  string `gorm:"type:varchar(32);not null"`  // 物流商
	ChargeType   string `gorm:"type:varchar(32);not null"`  // by_weight / by_volume / by_piece / fixed
	FirstWeight  int64  `gorm:"not null"`                   // 首重（克）
	FirstFee     int64  `gorm:"not null"`                   // 首重费用（分）
	ExtraWeight  int64  `gorm:"not null"`                   // 续重单位（克）
	ExtraFee     int64  `gorm:"not null"`                   // 续重费用（分）
	FreeThreshold int64 `gorm:"default:0"`                  // 满 X 元包邮（分），0 表示不包邮
	MaxWeight    int64  `gorm:"default:0"`                  // 最大可寄送重量（克），0 表示无限制
	CreatedAt    string `gorm:"not null;autoCreateTime"`
}

func (ShippingTemplate) TableName() string {
	return "shipping_templates"
}

// RegionFee 地区运费规则（特殊地区加价/不配送）
type RegionFee struct {
	ID           int64  `gorm:"primaryKey;autoIncrement"`
	TemplateID   string `gorm:"type:varchar(64);not null;index"`
	ProvinceCode string `gorm:"type:varchar(16);not null"`
	CityCode     string `gorm:"type:varchar(16)"`
	ExtraFee     int64  `gorm:"not null;default:0"` // 额外运费（分）
	NoDelivery   bool   `gorm:"default:false"`      // 不配送
}

func (RegionFee) TableName() string {
	return "shipping_region_fees"
}

// 错误定义
var (
	ErrNoDelivery          = errors.New("shipping: no delivery to this region")
	ErrWeightExceeded      = errors.New("shipping: weight exceeded maximum")
	ErrTemplateNotFound    = errors.New("shipping: template not found")
	ErrCarrierNotSupported = errors.New("shipping: carrier not supported")
)
```

### 2. Go 运费计算器

```go
package shipping

import (
	"fmt"

	"gorm.io/gorm"
)

// DefaultCalculator 默认运费计算器
type DefaultCalculator struct {
	db *gorm.DB
}

// NewDefaultCalculator 创建默认运费计算器
func NewDefaultCalculator(db *gorm.DB) *DefaultCalculator {
	return &DefaultCalculator{db: db}
}

// ceilDiv 整数除法向上取整：(a + b - 1) / b
func ceilDiv(a, b int64) int64 {
	if b <= 0 {
		return 0
	}
	return (a + b - 1) / b
}

// Calculate 计算运费
func (c *DefaultCalculator) Calculate(ctx context.Context, req FeeCalculateRequest) (FeeCalculateResult, error) {
	result := FeeCalculateResult{
		CarrierCode: req.CarrierCode,
	}

	// 1. 获取运费模板
	template, err := c.getTemplate(ctx, req.TenantID, req.CarrierCode)
	if err != nil {
		return result, err
	}
	result.CarrierCode = template.CarrierCode

	// 2. 检查重量限制
	if template.MaxWeight > 0 && req.WeightGram > template.MaxWeight {
		return result, fmt.Errorf("%w: max %dg, got %dg",
			ErrWeightExceeded, template.MaxWeight, req.WeightGram)
	}

	// 3. 检查地区是否配送
	regionFee, noDelivery, err := c.getRegionFee(ctx, template.ID, req.ProvinceCode, req.CityCode)
	if err != nil {
		return result, err
	}
	if noDelivery {
		return result, fmt.Errorf("%w: %s-%s", ErrNoDelivery, req.ProvinceCode, req.CityCode)
	}

	// 4. 检查包邮条件
	// 包邮在订单层处理（根据订单金额判断），运费计算器只计算基础运费

	// 5. 计算运费
	var fee int64
	switch template.ChargeType {
	case "fixed":
		// 固定运费
		fee = template.FirstFee

	case "by_weight":
		// 按重量：首重 + 续重
		firstWeight := template.FirstWeight
		firstFee := template.FirstFee
		extraWeightUnit := template.ExtraWeight
		extraFeeUnit := template.ExtraFee

		if req.WeightGram <= firstWeight {
			fee = firstFee
		} else {
			excess := req.WeightGram - firstWeight
			// 续重向上取整
			extraUnits := ceilDiv(excess, extraWeightUnit)
			fee = firstFee + int64(extraUnits)*extraFeeUnit
		}

	case "by_volume":
		// 按体积计费（逻辑类似按重量）
		firstVol := template.FirstWeight // 复用字段存储首体积
		firstFee := template.FirstFee
		extraVolUnit := template.ExtraWeight
		extraFeeUnit := template.ExtraFee

		if req.VolumeCm3 <= firstVol {
			fee = firstFee
		} else {
			excess := req.VolumeCm3 - firstVol
			extraUnits := ceilDiv(excess, extraVolUnit)
			fee = firstFee + int64(extraUnits)*extraFeeUnit
		}

	case "by_piece":
		// 按件数计费
		firstPieces := int(template.FirstWeight) // 复用字段存储首件数
		firstFee := template.FirstFee
		extraPieceUnit := int(template.ExtraWeight) // 复用字段存储续件单位
		extraFeeUnit := template.ExtraFee

		if req.ItemCount <= firstPieces {
			fee = firstFee
		} else {
			excess := req.ItemCount - firstPieces
			extraUnits := ceilDiv(excess, extraPieceUnit)
			fee = firstFee + int64(extraUnits)*extraFeeUnit
		}

	default:
		return result, fmt.Errorf("shipping: unknown charge type: %s", template.ChargeType)
	}

	// 6. 加上地区附加费
	fee += regionFee

	result.Fee = fee
	result.FirstWeight = template.FirstWeight
	result.FirstFee = template.FirstFee
	result.ExtraWeight = template.ExtraWeight
	result.ExtraFee = template.ExtraFee
	result.Breakdown = fmt.Sprintf("首重%dg %d分 + 续重 = %d分 + 地区附加 %d分 = %d分",
		template.FirstWeight/1000, template.FirstFee/100,
		(fee-regionFee)/100, regionFee/100, fee/100)

	return result, nil
}

// getTemplate 获取运费模板
func (c *DefaultCalculator) getTemplate(ctx context.Context, tenantID, carrierCode string) (*ShippingTemplate, error) {
	var template ShippingTemplate
	query := c.db.WithContext(ctx).Where("tenant_id = ?", tenantID)
	if carrierCode != "" {
		query = query.Where("carrier_code = ?", carrierCode)
	}
	err := query.Order("id ASC").First(&template).Error
	if err != nil {
		return nil, fmt.Errorf("%w: tenant=%s, carrier=%s", ErrTemplateNotFound, tenantID, carrierCode)
	}
	return &template, nil
}

// getRegionFee 获取地区附加费
func (c *DefaultCalculator) getRegionFee(
	ctx context.Context, templateID, provinceCode, cityCode string,
) (fee int64, noDelivery bool, err error) {
	var regionFees []RegionFee
	err = c.db.WithContext(ctx).Where("template_id = ? AND province_code = ?", templateID, provinceCode).
		Find(&regionFees).Error
	if err != nil {
		return 0, false, err
	}

	for _, rf := range regionFees {
		// 优先匹配城市级别规则
		if rf.CityCode != "" && rf.CityCode == cityCode {
			return rf.ExtraFee, rf.NoDelivery, nil
		}
		// 其次匹配省级规则
		if rf.CityCode == "" {
			fee = rf.ExtraFee
			noDelivery = rf.NoDelivery
		}
	}
	return fee, noDelivery, nil
}
```

### 3. 物流公司适配器

```go
package shipping

import (
	"context"
	"crypto/hmac"
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"sort"
	"strings"
	"time"

	"github.com/cenkalti/backoff/v4"
	"github.com/go-resty/resty/v3"
	"golang.org/x/time/rate"
)

// CarrierCode 物流公司编码
type CarrierCode string

const (
	CarrierSF  CarrierCode = "SF"  // 顺丰
	CarrierSTO CarrierCode = "STO" // 申通
	CarrierYTO CarrierCode = "YTO" // 圆通
	CarrierZTO CarrierCode = "ZTO" // 中通
)

// Carrier 物流服务商接口
type Carrier interface {
	Code() CarrierCode
	Name() string
	CreateWaybill(ctx context.Context, req WaybillRequest) (WaybillResult, error)
	QueryTracking(ctx context.Context, trackingNo string) ([]TrackingEvent, error)
	CancelWaybill(ctx context.Context, trackingNo string) error
}

// WaybillRequest 面单创建请求
type WaybillRequest struct {
	OrderID       string `json:"order_id"`
	SenderName    string `json:"sender_name"`
	SenderPhone   string `json:"sender_phone"`
	SenderAddr    string `json:"sender_addr"`
	SenderProvince string `json:"sender_province"`
	SenderCity    string `json:"sender_city"`
	SenderDistrict string `json:"sender_district"`
	ReceiverName  string `json:"receiver_name"`
	ReceiverPhone string `json:"receiver_phone"`
	ReceiverAddr  string `json:"receiver_addr"`
	ReceiverProvince string `json:"receiver_province"`
	ReceiverCity  string `json:"receiver_city"`
	ReceiverDistrict string `json:"receiver_district"`
	WeightGram    int64  `json:"weight_gram"`
	ItemName      string `json:"item_name"`
	ItemCount     int    `json:"item_count"`
	Value         int64  `json:"value"` // 保价金额（分）
}

// WaybillResult 面单创建结果
type WaybillResult struct {
	TrackingNo   string `json:"tracking_no"`   // 运单号
	WaybillURL   string `json:"waybill_url"`   // 电子面单 URL
	LabelURL     string `json:"label_url"`     // 打印标签 URL
	CarrierCode  string `json:"carrier_code"`  // 物流商编码
	EstimateFee  int64  `json:"estimate_fee"`  // 预估费用（分）
	CreateTime   time.Time `json:"create_time"`
}

// TrackingEvent 物流轨迹事件
type TrackingEvent struct {
	TrackingNo string    `json:"tracking_no"`
	Status     string    `json:"status"`      // 揽收/运输中/派送中/已签收/异常
	Detail     string    `json:"detail"`      // 详细描述
	Location   string    `json:"location"`    // 当前位置
	TrackTime  time.Time `json:"time"`        // 时间
}

// BaseCarrier 物流商基础实现（使用 resty HTTP 客户端）
type BaseCarrier struct {
	code      CarrierCode
	name      string
	baseURL   string
	appKey    string
	appSecret string
	client    *resty.Client
	limiter   *rate.Limiter
}

func (c *BaseCarrier) Code() CarrierCode { return c.code }
func (c *BaseCarrier) Name() string      { return c.name }

// signRequest 生成请求签名（使用 HMAC-SHA256，替代不安全的 MD5）
func (c *BaseCarrier) signRequest(params map[string]string) string {
	keys := make([]string, 0, len(params))
	for k := range params {
		keys = append(keys, k)
	}
	sort.Strings(keys)

	var buf strings.Builder
	for _, k := range keys {
		buf.WriteString(k)
		buf.WriteString("=")
		buf.WriteString(params[k])
		buf.WriteString("&")
	}
	buf.WriteString("secret=")
	buf.WriteString(c.appSecret)

	// HMAC-SHA256 签名（替代 MD5，符合 P0#5 安全与隐私原则）
	// 注意：各物流商签名算法不同（顺丰 HMAC-SHA256、中通 MD5、韵达 RSA），
	// 生产环境应在各 Carrier 适配器中实现各自的签名算法，此处 BaseCarrier
	// 默认使用更安全的 HMAC-SHA256。
	mac := hmac.New(sha256.New, []byte(c.appSecret))
	mac.Write([]byte(buf.String()))
	return strings.ToUpper(hex.EncodeToString(mac.Sum(nil)))
}

// doRequest 通用请求（使用 resty + 指数退避重试 + 限流）
func (c *BaseCarrier) doRequest(
	ctx context.Context,
	method string,
	params map[string]string,
) ([]byte, error) {
	params["app_key"] = c.appKey
	params["timestamp"] = fmt.Sprintf("%d", time.Now().UnixMilli())
	params["sign"] = c.signRequest(params)

	// 等待令牌桶可用（限流防护）
	if c.limiter != nil {
		if err := c.limiter.Wait(ctx); err != nil {
			return nil, fmt.Errorf("shipping: rate limiter wait: %w", err)
		}
	}

	var result []byte
	// 使用 backoff 库实现指数退避重试
	b := backoff.WithMaxRetries(backoff.NewExponentialBackOff(), 3)
	err := backoff.Retry(func() error {
		resp, err := c.client.R().
			SetContext(ctx).
			SetQueryParams(params).
			Get(c.baseURL + "/" + method)
		if err != nil {
			return err
		}
		if resp.StatusCode() >= 500 {
			// Retry only on server errors (5xx)
			return fmt.Errorf("shipping: API error %d: %s", resp.StatusCode(), string(resp.Body()))
		}
		if resp.StatusCode() != 200 {
			// 4xx client errors: return immediately without retry
			return backoff.Permanent(fmt.Errorf("shipping: client error %d: %s", resp.StatusCode(), string(resp.Body())))
		}
		result = resp.Body()
		return nil
	}, b)
	if err != nil {
		return nil, fmt.Errorf("shipping: request failed after retries: %w", err)
	}
	return result, nil
}
```

### 4. 顺丰适配器实现

```go
package shipping

import (
	"context"
	"encoding/json"
	"fmt"
	"strings"
	"time"

	"github.com/go-resty/resty/v3"
)

// SFCarrier 顺丰物流适配器
type SFCarrier struct {
	BaseCarrier
}

// NewSFCarrier 创建顺丰适配器
func NewSFCarrier(appKey, appSecret string) *SFCarrier {
	return &SFCarrier{
		BaseCarrier: BaseCarrier{
			code:      CarrierSF,
			name:      "顺丰速运",
			baseURL:   "https://bsp-oisp.sf-express.com/bsp-oisp/sfexpressService",
			appKey:    appKey,
			appSecret: appSecret,
			client:    resty.New().SetTimeout(10 * time.Second),
			limiter:   rate.NewLimiter(rate.Every(100*time.Millisecond), 10), // 10 req/s
		},
	}
}

// CreateWaybill 创建顺丰电子面单
func (c *SFCarrier) CreateWaybill(
	ctx context.Context,
	req WaybillRequest,
) (WaybillResult, error) {
	params := map[string]string{
		"method":          "orderService.orders",
		"partnerID":       c.appKey,
		"requestID":       req.OrderID,
		"language":        "zh-CN",
		"orderType":       "1", // 1=正常下单
		"cargoType":       "0", // 0=普通货物
		"payMethod":       "1", // 1=寄付
		"sendStartCity":   req.SenderCity,
		"sendStartAddress": req.SenderAddr,
		"sendContact":     req.SenderName,
		"sendMobilePhone": req.SenderPhone,
		"receiveCity":     req.ReceiverCity,
		"receiveAddress":  req.ReceiverAddr,
		"receiveContact":  req.ReceiverName,
		"receiveMobilePhone": req.ReceiverPhone,
		"cargoTotalWeight": fmt.Sprintf("%.2f", float64(req.WeightGram)/1000.0),
		"cargoCount":      fmt.Sprintf("%d", req.ItemCount),
		"remark":          req.ItemName,
	}

	body, err := c.doRequest(ctx, "orders", params)
	if err != nil {
		return WaybillResult{}, err
	}

	// 解析顺丰响应
	var sfResp struct {
		APIResultData struct {
			APIStatus   string `json:"apiStatus"`
			APIErrorMsg string `json:"apiErrorMsg"`
			MsgData     struct {
				MasterWaybillNo string `json:"masterWaybillNo"`
				WaybillNoInfo1  string `json:"waybillNoInfo1"`
				FilterResult    string `json:"filterResult"`
				EstimateFee     int64  `json:"estimateFee"`
			} `json:"msgData"`
		} `json:"apiResultData"`
	}
	if err := json.Unmarshal(body, &sfResp); err != nil {
		return WaybillResult{}, fmt.Errorf("shipping: SF response parse error: %w", err)
	}

	if sfResp.APIResultData.APIStatus != "100" {
		return WaybillResult{}, fmt.Errorf("shipping: SF API error: %s",
			sfResp.APIResultData.APIErrorMsg)
	}

	return WaybillResult{
		TrackingNo:  sfResp.APIResultData.MsgData.MasterWaybillNo,
		WaybillURL:  sfResp.APIResultData.MsgData.WaybillNoInfo1,
		CarrierCode: string(CarrierSF),
		CreateTime:  time.Now(),
	}, nil
}

// QueryTracking 查询顺丰物流轨迹
func (c *SFCarrier) QueryTracking(
	ctx context.Context,
	trackingNo string,
) ([]TrackingEvent, error) {
	params := map[string]string{
		"method":        "routeService.routePush",
		"trackingType":  "1", // 1=运单号查询
		"trackingNo":    trackingNo,
		"methodType":    "1", // 1=查询
	}

	body, err := c.doRequest(ctx, "route", params)
	if err != nil {
		return nil, err
	}

	var sfResp struct {
		APIResultData struct {
			APIStatus   string `json:"apiStatus"`
			APIErrorMsg string `json:"apiErrorMsg"`
			MsgData     []struct {
				AcceptAddress string `json:"acceptAddress"`
				AcceptRemark  string `json:"acceptRemark"`
				AcceptTime    string `json:"acceptTime"`
			} `json:"msgData"`
		} `json:"apiResultData"`
	}
	if err := json.Unmarshal(body, &sfResp); err != nil {
		return nil, fmt.Errorf("shipping: SF tracking parse error: %w", err)
	}

	if sfResp.APIResultData.APIStatus != "100" {
		return nil, fmt.Errorf("shipping: SF tracking error: %s",
			sfResp.APIResultData.APIErrorMsg)
	}

	var events []TrackingEvent
	for _, item := range sfResp.APIResultData.MsgData {
		t, err := time.Parse("2006-01-02 15:04:05", item.AcceptTime)
		if err != nil {
			return nil, fmt.Errorf("shipping: SF track time parse error %q: %w", item.AcceptTime, err)
		}
		event := TrackingEvent{
			TrackingNo: trackingNo,
			Status:     parseSFStatus(item.AcceptRemark),
			Detail:     item.AcceptRemark,
			Location:   item.AcceptAddress,
			TrackTime:  t,
		}
		events = append(events, event)
	}
	return events, nil
}

// CancelWaybill 取消顺丰面单
func (c *SFCarrier) CancelWaybill(ctx context.Context, trackingNo string) error {
	params := map[string]string{
		"method":      "orderService.ordersCancel",
		"trackingNo":  trackingNo,
		"cancelType":  "1",
	}

	body, err := c.doRequest(ctx, "cancel", params)
	if err != nil {
		return err
	}

	var sfResp struct {
		APIResultData struct {
			APIStatus string `json:"apiStatus"`
		} `json:"apiResultData"`
	}
	if err := json.Unmarshal(body, &sfResp); err != nil {
		return err
	}

	if sfResp.APIResultData.APIStatus != "100" {
		return fmt.Errorf("shipping: SF cancel failed")
	}
	return nil
}

// parseSFStatus 从顺丰轨迹描述中解析状态
func parseSFStatus(detail string) string {
	switch {
	case strings.Contains(detail, "揽收") || strings.Contains(detail, "已取件") || strings.Contains(detail, "取件"):
		return "picked_up"
	case strings.Contains(detail, "运输中") || strings.Contains(detail, "发往") || strings.Contains(detail, "到达") || strings.Contains(detail, "中转"):
		return "in_transit"
	case strings.Contains(detail, "派送") || strings.Contains(detail, "派件中"):
		return "delivering"
	case strings.Contains(detail, "签收") || strings.Contains(detail, "已签收"):
		return "delivered"
	case strings.Contains(detail, "问题") || strings.Contains(detail, "异常") || strings.Contains(detail, "拒签"):
		return "exception"
	default:
		return "unknown"
	}
}
```

### 5. 物流服务编排器（带重试和轨迹存储）

```go
package shipping

import (
	"context"
	"fmt"
	"time"

	"github.com/cenkalti/backoff/v4"
	"github.com/zeromicro/go-zero/core/logx"
	"gorm.io/gorm"
	"gorm.io/gorm/clause"
)
type ShippingService struct {
	db         *gorm.DB
	calculator ShippingFeeCalculator
	carriers   map[CarrierCode]Carrier
}

// NewShippingService 创建物流服务
func NewShippingService(
	db *gorm.DB,
	calculator ShippingFeeCalculator,
) *ShippingService {
	return &ShippingService{
		db:         db,
		calculator: calculator,
		carriers:   make(map[CarrierCode]Carrier),
	}
}

// RegisterCarrier 注册物流商
func (s *ShippingService) RegisterCarrier(carrier Carrier) {
	s.carriers[carrier.Code()] = carrier
}

// CalculateFee 计算运费
func (s *ShippingService) CalculateFee(
	ctx context.Context,
	req FeeCalculateRequest,
) (FeeCalculateResult, error) {
	result, err := s.calculator.Calculate(ctx, req)
	if err != nil {
		logx.WithContext(ctx).Errorw("shipping: calculate fee failed",
			logx.Field("tenant_id", req.TenantID),
			logx.Field("carrier", req.CarrierCode),
			logx.Field("error", err))
		return result, err
	}
	logx.WithContext(ctx).Infow("shipping: fee calculated",
		logx.Field("tenant_id", req.TenantID),
		logx.Field("fee", result.Fee))
	return result, nil
}

// CreateShipment 创建发货记录并获取面单
func (s *ShippingService) CreateShipment(
	ctx context.Context,
	tenantID string,
	orderID string,
	carrierCode CarrierCode,
	req WaybillRequest,
) (*Shipment, error) {
	carrier, ok := s.carriers[carrierCode]
	if !ok {
		return nil, fmt.Errorf("%w: %s", ErrCarrierNotSupported, carrierCode)
	}

	// 调用物流商创建面单（重试逻辑已在 BaseCarrier.doRequest 内部处理）
	result, err := carrier.CreateWaybill(ctx, req)
	if err != nil {
		return nil, fmt.Errorf("shipping: failed to create waybill: %w", err)
	}

	// 在同一事务中保存发货记录，确保原子性
	var shipment *Shipment
	err = s.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
		shipment = &Shipment{
			TenantID:     tenantID,
			OrderID:      orderID,
			CarrierCode:  string(carrierCode),
			CarrierName:  carrier.Name(),
			TrackingNo:   result.TrackingNo,
			WaybillURL:   result.WaybillURL,
			Status:       "created",
			EstimateFee:  result.EstimateFee,
			LastSyncAt:   time.Now(),
		}
		return tx.Create(shipment).Error
	})
	if err != nil {
		return nil, fmt.Errorf("shipping: failed to save shipment: %w", err)
	}

	logx.WithContext(ctx).Infow("shipping: shipment created",
		logx.Field("tenant_id", tenantID),
		logx.Field("order_id", orderID),
		logx.Field("tracking_no", shipment.TrackingNo))

	return shipment, nil
}

// QueryTracking 查询物流轨迹
func (s *ShippingService) QueryTracking(
	ctx context.Context,
	trackingNo string,
	carrierCode CarrierCode,
) ([]TrackingEvent, error) {
	carrier, ok := s.carriers[carrierCode]
	if !ok {
		return nil, fmt.Errorf("%w: %s", ErrCarrierNotSupported, carrierCode)
	}

	events, err := carrier.QueryTracking(ctx, trackingNo)
	if err != nil {
		return nil, err
	}

	// 存储轨迹到本地数据库（批量插入，ON CONFLICT 去重）
	if len(events) > 0 {
		// 从 Shipment 获取 TenantID，确保多租户隔离
		var shipment Shipment
		if err := s.db.WithContext(ctx).Select("tenant_id").
			Where("tracking_no = ?", trackingNo).First(&shipment).Error; err != nil {
			return nil, fmt.Errorf("shipping: shipment not found for tracking %s: %w", trackingNo, err)
		}

		records := make([]TrackingRecord, 0, len(events))
		for _, event := range events {
			records = append(records, TrackingRecord{
				TenantID:    shipment.TenantID,
				TrackingNo:  trackingNo,
				CarrierCode: string(carrierCode),
				Status:      event.Status,
				Detail:      event.Detail,
				Location:    event.Location,
				TrackTime:   event.TrackTime,
			})
		}
		err := s.db.WithContext(ctx).
			Clauses(clause.OnConflict{
				Columns: []clause.Column{{Name: "tracking_no"}, {Name: "track_time"}, {Name: "detail"}},
				DoNothing: true,
			}).
			CreateInBatches(&records, 100).Error
		if err != nil {
			return nil, fmt.Errorf("shipping: failed to save tracking records: %w", err)
		}
	}

	// 更新发货状态
	latestStatus := ""
	if len(events) > 0 {
		latestStatus = events[0].Status
	}
	if latestStatus != "" {
		if err := s.db.WithContext(ctx).Model(&Shipment{}).
			Where("tracking_no = ?", trackingNo).
			Updates(&Shipment{
				Status:     latestStatus,
				LastSyncAt: time.Now(),
			}).Error; err != nil {
			logx.WithContext(ctx).Errorw("shipping: failed to update shipment status",
				logx.Field("tracking_no", trackingNo),
				logx.Field("error", err))
		}
	}

	return events, nil
}

// CancelShipment 取消发货
func (s *ShippingService) CancelShipment(
	ctx context.Context,
	trackingNo string,
	carrierCode CarrierCode,
) error {
	carrier, ok := s.carriers[carrierCode]
	if !ok {
		return fmt.Errorf("%w: %s", ErrCarrierNotSupported, carrierCode)
	}

	err := carrier.CancelWaybill(ctx, trackingNo)
	if err != nil {
		return err
	}

	err = s.db.WithContext(ctx).Model(&Shipment{}).
		Where("tracking_no = ?", trackingNo).
		Update("status", "cancelled").Error
	if err != nil {
		return fmt.Errorf("shipping: failed to update shipment status: %w", err)
	}

	logx.WithContext(ctx).Infow("shipping: shipment cancelled",
		logx.Field("tracking_no", trackingNo))

	return nil
}

// Shipment 发货记录
type Shipment struct {
	ID           int64     `gorm:"primaryKey;autoIncrement"`
	TenantID     string    `gorm:"type:varchar(64);not null;index"` // 多租户隔离
	OrderID      string    `gorm:"type:varchar(64);not null;index"`
	CarrierCode  string    `gorm:"type:varchar(32);not null"`
	CarrierName  string    `gorm:"type:varchar(64)"`
	TrackingNo   string    `gorm:"type:varchar(64);not null;uniqueIndex"`
	WaybillURL   string    `gorm:"type:varchar(512)"`
	Status       string    `gorm:"type:varchar(32);not null;default:'created'"`
	EstimateFee  int64     `gorm:"default:0"`
	LastSyncAt   time.Time `gorm:"not null"`
	CreatedAt    time.Time `gorm:"not null;autoCreateTime"`
}

func (Shipment) TableName() string {
	return "shipments"
}

// TrackingRecord 物流轨迹记录
type TrackingRecord struct {
	ID          int64     `gorm:"primaryKey;autoIncrement"`
	TenantID    string    `gorm:"type:varchar(64);not null;index"` // 多租户隔离
	TrackingNo  string    `gorm:"type:varchar(64);not null;index:idx_track_no_time"`
	CarrierCode string    `gorm:"type:varchar(32);not null"`
	Status      string    `gorm:"type:varchar(32)"`
	Detail      string    `gorm:"type:varchar(512)"`
	Location    string    `gorm:"type:varchar(256)"`
	TrackTime   time.Time `gorm:"not null;index:idx_track_no_time"`
	CreatedAt   time.Time `gorm:"not null;autoCreateTime"`
}

func (TrackingRecord) TableName() string {
	return "tracking_records"
}
```

### 6. 物流轨迹定时同步任务

```go
package shipping

import (
	"context"
	"sync"
	"time"

	"github.com/zeromicro/go-zero/core/logx"
	"gorm.io/gorm"
)

// TrackingSyncer 物流轨迹同步器
type TrackingSyncer struct {
	db         *gorm.DB
	service    *ShippingService
	interval   time.Duration
	batchSize  int
	maxConcurrent int // 最大并发 API 查询数
}

// NewTrackingSyncer 创建轨迹同步器
func NewTrackingSyncer(
	db *gorm.DB,
	service *ShippingService,
	interval time.Duration,
) *TrackingSyncer {
	if interval <= 0 {
		interval = 2 * time.Hour // 默认每 2 小时同步一次
	}
	return &TrackingSyncer{
		db:            db,
		service:       service,
		interval:      interval,
		batchSize:     50,
		maxConcurrent: 5, // 默认 5 并发，避免触发物流 API 限流
	}
}

// SyncPending 同步待更新的物流轨迹
func (ts *TrackingSyncer) SyncPending(ctx context.Context) error {
	// 先获取所有有未同步运单的租户（多租户隔离）
	var tenantIDs []string
	err := ts.db.WithContext(ctx).Model(&Shipment{}).
		Where("status NOT IN ?", []string{"delivered", "cancelled", "returned"}).
		Where("last_sync_at < ?", time.Now().Add(-ts.interval)).
		Distinct("tenant_id").Pluck("tenant_id", &tenantIDs).Error
	if err != nil {
		return err
	}

	// 按租户逐一处理，确保查询始终带 tenant_id 过滤
	for _, tenantID := range tenantIDs {
		var shipments []Shipment
		err := ts.db.WithContext(ctx).
			Where("tenant_id = ?", tenantID).
			Where("status NOT IN ?", []string{"delivered", "cancelled", "returned"}).
			Where("last_sync_at < ?", time.Now().Add(-ts.interval)).
			Order("last_sync_at ASC").
			Limit(ts.batchSize).
			Find(&shipments).Error
		if err != nil {
			logx.WithContext(ctx).Errorw("sync pending shipments query failed",
				logx.Field("tenant_id", tenantID),
				logx.Field("error", err))
			continue
		}

		// 使用信号量控制并发数，避免触发物流 API 限流
		sem := make(chan struct{}, ts.maxConcurrent)
		var wg sync.WaitGroup
		for _, s := range shipments {
			wg.Add(1)
			sem <- struct{}{} // 获取信号量
			go func(shipment Shipment) {
				defer wg.Done()
				defer func() { <-sem }() // 释放信号量

				carrierCode := CarrierCode(shipment.CarrierCode)
				events, err := ts.service.QueryTracking(ctx, shipment.TrackingNo, carrierCode)
				if err != nil {
					logx.WithContext(ctx).Errorw("tracking sync failed",
						logx.Field("tracking_no", shipment.TrackingNo),
						logx.Field("error", err))
					return
				}

				// 检查是否已签收
				for _, e := range events {
					if e.Status == "delivered" {
						ts.db.WithContext(ctx).Model(&Shipment{}).
							Where("id = ?", shipment.ID).
							Updates(&Shipment{
								Status:     "delivered",
								LastSyncAt: time.Now(),
							})
						break
					}
				}

				logx.WithContext(ctx).Infow("tracking synced",
					logx.Field("tracking_no", shipment.TrackingNo),
					logx.Field("events", len(events)))
			}(s)
		}
		wg.Wait()
	}

	return nil
}
```

### 7. 使用示例

```go
package main

import (
	"context"
	"fmt"
	"time"
	"storeforge/shipping"

	"github.com/zeromicro/go-zero/core/logx"
)

func main() {
	db := getDB() // *gorm.DB

	// 创建运费计算器
	calc := shipping.NewDefaultCalculator(db)

	// 创建物流服务
	svc := shipping.NewShippingService(db, calc)

	// 注册物流商
	sf := shipping.NewSFCarrier("your-app-key", "your-app-secret")
	svc.RegisterCarrier(sf)
	// svc.RegisterCarrier(shipping.NewZTOCarrier(...))
	// svc.RegisterCarrier(shipping.NewYTOCarrier(...))
	// svc.RegisterCarrier(shipping.NewSTOCarrier(...))

	ctx := context.Background()

	// === 示例 1: 计算运费 ===
	fee, err := svc.CalculateFee(ctx, shipping.FeeCalculateRequest{
		ProvinceCode: "440000", // 广东
		CityCode:     "440300", // 深圳
		WeightGram:   2500,     // 2.5kg
		TenantID:     "STORE01",
		CarrierCode:  "SF",
	})
	if err != nil {
		logx.WithContext(ctx).Errorw("calculate fee failed", logx.Field("error", err))
	}
	fmt.Printf("运费: %d 分 (%s)\n", fee.Fee, fee.Breakdown)

	// === 示例 2: 创建发货 ===
	shipment, err := svc.CreateShipment(ctx, "STORE01", "ORD20260420001", shipping.CarrierSF, shipping.WaybillRequest{
		OrderID:        "ORD20260420001",
		SenderName:     "张三",
		SenderPhone:    "13800138000",
		SenderAddr:     "XX区XX路XX号",
		SenderCity:     "440300",
		ReceiverName:   "李四",
		ReceiverPhone:  "13900139000",
		ReceiverAddr:   "YY区YY路YY号",
		ReceiverCity:   "110100",
		WeightGram:     2500,
		ItemName:       "iPhone 15 Pro",
		ItemCount:      1,
	})
	if err != nil {
		logx.WithContext(ctx).Errorw("create shipment failed", logx.Field("error", err))
	}
	fmt.Printf("运单号: %s, 面单URL: %s\n", shipment.TrackingNo, shipment.WaybillURL)

	// === 示例 3: 查询物流轨迹 ===
	events, err := svc.QueryTracking(ctx, shipment.TrackingNo, shipping.CarrierSF)
	if err != nil {
		logx.WithContext(ctx).Warnw("tracking query failed", logx.Field("error", err))
	} else {
		for _, e := range events {
			fmt.Printf("[%s] %s - %s (%s)\n",
				e.TrackTime.Format("2006-01-02 15:04"),
				e.Status, e.Detail, e.Location)
		}
	}

	// === 示例 4: 定时同步轨迹 ===
	syncer := shipping.NewTrackingSyncer(db, svc, 2*time.Hour)
	// 每 2 小时同步一次
	// ticker := time.NewTicker(2 * time.Hour)
	// go func() {
	//     for range ticker.C {
	//         syncer.SyncPending(ctx)
	//     }
	// }()
}
```

## 反模式

### 1. 硬编码运费

```go
// 错误：运费写死在代码里
const SHIPPING_FEE = 1000 // 10 元
func CalculateShipping() int64 {
    return SHIPPING_FEE
}
```

**问题**：不同地区、不同重量、不同物流商运费不同，硬编码无法适应业务变化。

**正确做法**：使用 `ShippingTemplate` + `RegionFee` 配置化计算，支持按重量/体积/件数/地区差异化计费。

### 2. 不记录物流轨迹

```go
// 错误：调完物流 API 就不管了，不存储轨迹
func Ship(orderID string) {
    waybill := carrier.CreateWaybill(req)
    db.Update("tracking_no", waybill.TrackingNo)
    // 轨迹呢？用户怎么知道货到哪了？
}
```

**问题**：用户无法追踪物流状态，客服也无法回答"我的货到哪了"。

**正确做法**：定时调用物流 API 查询轨迹，存储到 `tracking_records` 表，前端可展示完整物流时间线。

### 3. 物流 API 失败不重试

```go
// 错误：一次失败就报错
func CreateWaybill(req WaybillRequest) (string, error) {
    result, err := sfClient.CreateWaybill(req)
    if err != nil {
        return "", err // 网络抖动就失败，用户体验极差
    }
    return result.TrackingNo, nil
}
```

**问题**：物流 API 偶发超时/失败，用户下单后无法发货。

**正确做法**：实现指数退避重试（3 次，间隔 1s/2s/4s），重试仍失败则记录到重试队列异步处理。

### 4. 不处理物流状态异常

```go
// 错误：只关心"已签收"，不管异常状态
func UpdateStatus(trackingNo string) {
    events := carrier.QueryTracking(trackingNo)
    latest := events[0]
    if latest.Status == "签收" {
        updateOrderStatus(delivered)
    }
    // 拒签？丢失？破损？不管了？
}
```

**问题**：物流异常（拒签、丢失、破损）需要及时处理，否则影响用户体验和售后。

**正确做法**：监控物流异常状态（`exception`），触发告警和售后流程。

### 5. 面单号重复使用

```go
// 错误：不校验运单号唯一性
db.Create(&Shipment{TrackingNo: trackingNo, ...})
// 如果 trackingNo 已存在会怎样？
```

**问题**：同一个运单号被多次使用，物流查询混乱。

**正确做法**：`tracking_no` 字段加 `uniqueIndex`，创建前检查唯一性。

## 常见坑

### 1. 物流公司 API 签名方式不同

**坑**：顺丰、中通、圆通、申通的 API 签名方式各不相同（有的 MD5，有的 HMAC-SHA256，有的需要 RSA）。

**解决**：
- 为每个物流公司实现独立的 `Carrier` 适配器
- 签名逻辑封装在各自适配器内部，外部通过统一接口调用
- 签名参数（appKey、appSecret）配置化，不要硬编码

### 2. 物流轨迹查询频率限制

**坑**：物流 API 有频率限制（如顺丰 100 次/分钟），大量订单同时查询会被限流。

**解决**：
- 定时同步时限制每批处理数量（`batchSize = 50`）
- 使用令牌桶限流（`rate.Limiter`）控制 API 调用频率
- 优先同步即将超时的订单（已发货 7 天以上未签收的）
- 已签收的订单不再轮询

### 3. 重量单位混淆

**坑**：运费模板用"千克"，物流 API 用"克"，前端用"斤"，换算出错。

**解决**：
- 系统内部统一使用克（`WeightGram int64`）
- 对外展示时转换：`fmt.Sprintf("%.2fkg", float64(grams)/1000.0)`
- 前端输入时明确标注单位

### 4. 电子面单打印兼容性

**坑**：不同物流商的面单格式不同（顺丰热敏纸 100x150mm，中通 100x180mm），打印机设置不对会错位。

**解决**：
- 使用物流商提供的标准面单 URL（PDF 格式）
- 后端存储面单 URL，前端通过 iframe 或新窗口打开打印
- 打印机预设好对应物流商的面单模板

### 5. 偏远地区不配送

**坑**：某些物流公司不配送偏远地区（西藏、新疆部分地区），但运费模板没有配置，导致下单成功但无法发货。

**解决**：
- 运费模板配置 `RegionFee.NoDelivery = true` 标记不配送地区
- 下单前校验收货地址是否在配送范围内
- 前端选择收货地址时实时校验并提示

### 6. 物流 webhook 回调验签

**坑**：物流公司推送的 webhook 回调没有验签，恶意请求可以伪造物流状态。

**解决**：
- 所有 webhook 回调必须验证签名（物流公司通常提供 HMAC 签名）
- 验签失败直接拒绝，记录安全日志
- 回调接口配置 IP 白名单

### 7. 物流信息脱敏

**坑**：物流轨迹中包含收件人手机号、姓名等 PII 信息，直接返回前端可能泄露。

**解决**：
- 物流详情中的手机号脱敏：`138****1234`
- 地址只显示到区/县级别
- 敏感信息加密存储（手机号 AES-256-GCM）

### 8. 发货后库存未扣减

**坑**：订单创建时预扣库存，发货后未转为实际扣减，取消时重复释放。

**解决**：
- 订单创建：`inventory_reserve`（预扣）
- 订单支付：保持预扣
- 订单发货：`inventory_reserve → inventory_deduct`（预扣转实际扣减）
- 订单取消/退款：`inventory_reserve → inventory_release`（释放预扣）
- 所有库存操作必须幂等

## 测试策略

### L1 单元测试（覆盖率目标 >= 80%）

#### Carrier Adapter Mock Test

```go
func TestCarrierAdapter_CreateWaybill(t *testing.T) {
    // Use httptest.NewServer to mock carrier API responses
    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(200)
        w.Write([]byte(`{"apiResultData":{"apiStatus":"100","apiErrorMsg":"","msgData":{"masterWaybillNo":"SF123456","waybillNoInfo1":"https://..."}}}`))
    }))
    defer srv.Close()

    carrier := shipping.NewSFCarrier("test-key", "test-secret")
    // Override baseURL via reflection or constructor injection
    // ...

    result, err := carrier.CreateWaybill(context.Background(), shipping.WaybillRequest{
        OrderID:    "ORD001",
        SenderCity: "440300",
        // ...
    })
    assert.NoError(t, err)
    assert.Equal(t, "SF123456", result.TrackingNo)
}

func TestCarrierAdapter_4xx_NoRetry(t *testing.T) {
    // Verify that 4xx errors are NOT retried (fix: retry only on 5xx)
    callCount := 0
    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        callCount++
        w.WriteHeader(400)
    }))
    defer srv.Close()
    // ... assert callCount == 1 (no retry on 4xx)
}
```

#### Fee Calculation Test

```go
func TestFeeCalculation_ByWeight(t *testing.T) {
    // Test: 首重 1000g, 首费 1200分, 续重 500g, 续费 300分
    // Case 1: 500g <= firstWeight → fee = 1200
    // Case 2: 1500g → excess=500 → ceilDiv(500,500)=1 → fee = 1200+300=1500
    // Case 3: 2100g → excess=1100 → ceilDiv(1100,500)=3 → fee = 1200+900=2100
}

func TestFeeCalculation_ByVolume(t *testing.T) {
    // Test volume-based calculation with same ceiling division logic
}

func TestFeeCalculation_ByPiece(t *testing.T) {
    // Test piece-based: first 2 pieces = 800分, each extra 1 piece = 200分
    // Case: 5 pieces → excess=3 → ceilDiv(3,1)=3 → fee = 800+600=1400
}

func TestFeeCalculation_Fixed(t *testing.T) {
    // Fixed fee: always returns FirstFee regardless of weight/volume
}

func TestFeeCalculation_RegionNoDelivery(t *testing.T) {
    // RegionFee.NoDelivery=true → returns ErrNoDelivery
}

func TestFeeCalculation_RegionSurcharge(t *testing.T) {
    // RegionFee.ExtraFee > 0 → base fee + extra fee
}
```

### L2 集成测试

```go
func TestShippingService_CreateShipment(t *testing.T) {
    // Use sqlite in-memory DB, register a mock carrier, test full flow
}

func TestShippingService_QueryTracking_BatchInsert(t *testing.T) {
    // Verify batch insert with ON CONFLICT DO NOTHING deduplicates correctly
}

func TestTrackingSyncer_SyncPending(t *testing.T) {
    // Seed shipments with different last_sync_at, verify ORDER BY ASC
}
```

### 覆盖率目标

| 层级 | 目标覆盖率 | 说明 |
|------|-----------|------|
| L1 Unit | >= 80% | 运费计算、地址解析、物流商签名 |
| L2 Integration | >= 70% | DB 运单 CRUD + Redis 轨迹缓存 + 多租户隔离 |
| L3 E2E | 核心流程 | 创建运单 → 物流商 API → 轨迹同步 |

#### 多租户隔离测试

```go
func TestShipping_MultiTenantIsolation(t *testing.T) {
	// 验证 tenantA 的运单对 tenantB 不可见
	// 模板和运费按 tenant_id 过滤
	// 轨迹记录包含 tenantID：tracking:{tenantID}:{trackingNo}
}
```

#### 并发 API 重试测试

```go
func TestCarrier_SyncTracking_Concurrent(t *testing.T) {
	// 使用 backoff 测试物流商 API 重试行为
	// 验证网络超时后自动重试且不超过最大重试次数
}
```
