# 发票系统 (Invoice System)

## 触发条件

- 需要电子发票开具（个人/企业/增值税专用）
- 发票状态追踪（已申请/已开具/已发送/已红冲/已作废）
- 红冲/作废流程
- 与支付系统对账（金额、税额、开票时间一致性校验）

## 决策树

```
订单完成 (order.status = PAID/DONE)
  └─→ 用户申请发票
       ├─→ 选择发票类型
       │    ├─ 个人发票 (personal)
       │    ├─ 企业普通发票 (enterprise)
       │    └─ 增值税专用发票 (vat)
       ├─→ 填写抬头 / 税号（企业/增值税必填）
       ├─→ 系统开具
       │    ├─→ 计算税额
       │    ├─→ 生成发票号码
       │    └─→ 生成 PDF
       ├─→ 发送
       │    ├─ 邮件发送 (email)
       │    └─ 短信通知 (sms)
       └─→ 后续操作
            ├─ 红冲 (red stamp) — 开具红字发票冲销
            └─ 作废 (void) — 当月未跨月可直接作废
```

## 推荐第三方库

| 用途 | 推荐库 | 导入路径 |
|------|--------|----------|
| PDF 发票生成 | gofpdf | `github.com/jung-kurt/gofpdf` |
| Excel 批量导出 | excelize | `github.com/xuri/excelize/v2` |
| 税务计算 | decimal | `github.com/shopspring/decimal` |
| 邮件发送 | gomail | `gopkg.in/gomail.v2` |

## 代码模板

### 1. 发票模型 (GORM)

```go
package model

import (
	"time"

	"gorm.io/gorm"
)

// InvoiceType 发票类型
type InvoiceType string

const (
	InvoiceTypePersonal   InvoiceType = "personal"   // 个人发票
	InvoiceTypeEnterprise InvoiceType = "enterprise" // 企业普通发票
	InvoiceTypeVAT        InvoiceType = "vat"        // 增值税专用发票
)

// InvoiceStatus 发票状态
type InvoiceStatus string

const (
	InvoiceStatusApplied    InvoiceStatus = "applied"    // 已申请
	InvoiceStatusIssued     InvoiceStatus = "issued"     // 已开具
	InvoiceStatusSent       InvoiceStatus = "sent"       // 已发送
	InvoiceStatusRedDashed  InvoiceStatus = "red_dashed" // 已红冲（红字发票冲销，税务合法操作）
	InvoiceStatusVoided     InvoiceStatus = "voided"     // 已作废（当月未跨月可直接作废）
)

// Invoice 发票主表
type Invoice struct {
	ID          int64         `gorm:"primaryKey;autoIncrement" json:"id"`
	TenantID    string        `gorm:"size:64;not null;index:idx_tenant" json:"tenant_id"` // 多租户隔离
	InvoiceNo   string        `gorm:"uniqueIndex:idx_invoice_no_tenant;size:64;not null" json:"invoice_no"` // 发票号码（按租户唯一）
	OrderID     int64         `gorm:"uniqueIndex:idx_order_tenant;not null" json:"order_id"`                  // 关联订单
	Type        InvoiceType   `gorm:"size:32;not null" json:"type"`                                           // 发票类型
	Title       string        `gorm:"size:256;not null" json:"title"`                                         // 发票抬头
	TaxNumber   string        `gorm:"size:64" json:"tax_number"`                                              // 税号（企业/增值税必填）
	Amount      int64         `gorm:"not null" json:"amount"`                                                 // 金额（分）
	TaxAmount   int64         `gorm:"not null" json:"tax_amount"`                                             // 税额（分）
	Status      InvoiceStatus `gorm:"size:32;not null;default:applied" json:"status"`                         // 状态
	PDFURL      string        `gorm:"size:512" json:"pdf_url"`                                                // PDF 下载地址
	VoidReason  string        `gorm:"size:512" json:"void_reason"`                                            // 红冲/作废原因
	VoidRefNo   string        `gorm:"size:64" json:"void_ref_no"`                                             // 红冲关联的原发票号码
	Email       string        `gorm:"size:256" json:"email"`                                                  // 接收邮箱（生产环境需 AES-256-GCM 加密）
	Phone       string        `gorm:"size:32" json:"phone"`                                                   // 接收手机（生产环境需 AES-256-GCM 加密，展示时脱敏）
	CreatedAt   time.Time     `json:"created_at"`
	UpdatedAt   time.Time     `json:"updated_at"`
	DeletedAt   gorm.DeletedAt `gorm:"index" json:"-"`
}

func (Invoice) TableName() string {
	return "invoices"
}

// 注意：SoftDelete 与唯一索引冲突（gorm.DeletedAt 会导致已删除记录阻塞新插入）。
// 生产环境应使用复合唯一索引 + is_deleted 字段替代 gorm.DeletedAt：
//   InvoiceNo + TenantID + is_deleted = 0 → 唯一约束
// 或使用 GORM 的 DeletedAt 时，创建部分唯一索引：
//   UNIQUE (invoice_no, tenant_id) WHERE deleted_at IS NULL
```

### 2. 发票状态机

```go
package invoice

import (
	"errors"
	"fmt"
)

// InvoiceStatus 状态定义（同 model.InvoiceStatus 类型，此处为展示完整代码块而列出）
type InvoiceStatus string

const (
	StatusApplied    InvoiceStatus = "applied"
	StatusIssued     InvoiceStatus = "issued"
	StatusSent       InvoiceStatus = "sent"
	StatusRedDashed  InvoiceStatus = "red_dashed" // 红冲：红字发票冲销（中国税务合法操作）
	StatusVoided     InvoiceStatus = "voided"     // 作废：当月未跨月直接作废
)

// allowedTransitions 定义合法的状态转换
// 使用 map[InvoiceStatus][]InvoiceStatus 避免单值 map 的 key 冲突问题
var allowedTransitions = map[InvoiceStatus][]InvoiceStatus{
	StatusApplied: {StatusIssued},
	StatusIssued:  {StatusSent, StatusRedDashed, StatusVoided},
	StatusSent:    {StatusRedDashed, StatusVoided},
}

// Transition 执行状态转换
func (s InvoiceStatus) Transition(to InvoiceStatus) (InvoiceStatus, error) {
	for _, t := range allowedTransitions[s] {
		if t == to {
			return to, nil
		}
	}
	return s, fmt.Errorf("invalid transition: %s -> %s", s, to)
}

// CanVoid 判断是否可以作废
func (s InvoiceStatus) CanVoid() bool {
	return s == StatusIssued || s == StatusSent
}

// CanRedDash 判断是否可以红冲
func (s InvoiceStatus) CanRedDash() bool {
	return s == StatusIssued || s == StatusSent
}
```

### 3. 发票号码生成器（按年月递增，带并发控制）

```go
package invoice

import (
	"fmt"
	"sync"
	"time"
)

// InvoiceNoGenerator 发票号码生成器
// 格式: INV + YYYYMM + 6位序号 (例: INV202604000001)
type InvoiceNoGenerator struct {
	mu      sync.Mutex
	prefix  string   // 当前年月前缀，如 "INV202604"
	sequence int64   // 当前序号
}

// NewInvoiceNoGenerator 创建生成器（应从 DB/Redis 加载当前序号）
func NewInvoiceNoGenerator(year, month int, lastSeq int64) *InvoiceNoGenerator {
	return &InvoiceNoGenerator{
		prefix:   fmt.Sprintf("INV%04d%02d", year, month),
		sequence: lastSeq,
	}
}

// Next 生成下一个发票号码
func (g *InvoiceNoGenerator) Next() string {
	g.mu.Lock()
	defer g.mu.Unlock()

	g.sequence++
	return fmt.Sprintf("%s%06d", g.prefix, g.sequence)
}

// Rotate 跨月时重置序号（应配合定时任务或懒加载检查）
func (g *InvoiceNoGenerator) Rotate(year, month int, resetSeq int64) {
	g.mu.Lock()
	defer g.mu.Unlock()

	newPrefix := fmt.Sprintf("INV%04d%02d", year, month)
	if newPrefix != g.prefix {
		g.prefix = newPrefix
		g.sequence = resetSeq
	}
}
```

> **高并发场景建议**: 使用 Redis `INCR` 或数据库序列表替代内存 `sync.Mutex`。

```go
// Redis 版本（推荐用于分布式部署）
func GenerateInvoiceNoWithRedis(ctx context.Context, rdb *redis.Client, tenantID string, now time.Time) (string, error) {
	key := fmt.Sprintf("invoice:%s:seq:%s", tenantID, now.Format("200601"))
	seq, err := rdb.Incr(ctx, key).Result()
	if err != nil {
		return "", err
	}
	return fmt.Sprintf("INV%s%06d", now.Format("200601"), seq), nil
}
```

### 4. PDF 发票生成（使用 gofpdf）

> **合规警告（IMPORTANT）**: gofpdf 生成的 PDF 仅用于内部预览/测试，**不是合法的电子发票**。
> 中国法律规定的电子发票必须通过税局系统开具（百望/航天信息/国家税务总局 API），
> gofpdf 输出的文件无税务效力，不可作为正式发票交付给客户。
> 生产环境应使用第三方税控服务返回的 OFD/PDF，此代码仅作辅助用途。

```go
package invoice

import (
	"fmt"

	"github.com/jung-kurt/gofpdf"
)

// InvoiceItem 发票明细行
type InvoiceItem struct {
	Name     string // 商品/服务名称
	Spec     string // 规格型号
	Quantity string // 数量
	UnitPrice string // 单价
	Amount   string // 金额
	TaxRate  string // 税率
}

// GenerateInvoicePDF 生成 PDF 发票
func GenerateInvoicePDF(ctx context.Context, inv *Invoice, items []InvoiceItem) ([]byte, error) {
	pdf := gofpdf.New("P", "mm", "A4", "")
	pdf.AddPage()

	// ---- 发票头 ----
	pdf.SetFont("Helvetica", "B", 20)
	pdf.CellFormat(190, 15, "电 子 发 票", "", 1, "C", false, 0, "")

	pdf.SetFont("Helvetica", "", 10)
	pdf.CellFormat(95, 8, fmt.Sprintf("发票号码: %s", inv.InvoiceNo), "", 0, "L", false, 0, "")
	pdf.CellFormat(95, 8, fmt.Sprintf("开具日期: %s", inv.CreatedAt.Format("2006-01-02")), "", 1, "R", false, 0, "")

	pdf.Ln(4)

	// ---- 购买方信息 ----
	pdf.SetFont("Helvetica", "B", 12)
	pdf.CellFormat(190, 8, "购买方信息", "1", 1, "L", true, 0, "")
	pdf.SetFont("Helvetica", "", 10)
	pdf.CellFormat(40, 7, "发票抬头:", "1", 0, "R", false, 0, "")
	pdf.CellFormat(150, 7, inv.Title, "1", 1, "L", false, 0, "")
	if inv.TaxNumber != "" {
		pdf.CellFormat(40, 7, "纳税人识别号:", "1", 0, "R", false, 0, "")
		pdf.CellFormat(150, 7, inv.TaxNumber, "1", 1, "L", false, 0, "")
	}

	pdf.Ln(4)

	// ---- 明细表头 ----
	pdf.SetFont("Helvetica", "B", 9)
	headers := []string{"商品名称", "规格型号", "数量", "单价", "金额", "税率"}
	widths := []float64{50, 30, 25, 25, 30, 30}
	for i, h := range headers {
		pdf.CellFormat(widths[i], 7, h, "1", 0, "C", true, 0, "")
	}
	pdf.CellFormat(0, 7, "", "1", 1, "C", true, 0, "")

	// ---- 明细行 ----
	pdf.SetFont("Helvetica", "", 9)
	for _, item := range items {
		cells := []string{item.Name, item.Spec, item.Quantity, item.UnitPrice, item.Amount, item.TaxRate}
		for i, c := range cells {
			pdf.CellFormat(widths[i], 7, c, "1", 0, "L", false, 0, "")
		}
		pdf.CellFormat(0, 7, "", "1", 1, "L", false, 0, "")
	}

	pdf.Ln(4)

	// ---- 金额合计 ----
	pdf.SetFont("Helvetica", "B", 12)
	pdf.CellFormat(95, 8, "合计金额 (不含税):", "1", 0, "R", true, 0, "")
	pdf.CellFormat(95, 8, fmt.Sprintf("¥%.2f", float64(inv.Amount)/100), "1", 1, "L", true, 0, "")
	pdf.CellFormat(95, 8, "合计税额:", "1", 0, "R", true, 0, "")
	pdf.CellFormat(95, 8, fmt.Sprintf("¥%.2f", float64(inv.TaxAmount)/100), "1", 1, "L", true, 0, "")
	total := inv.Amount + inv.TaxAmount
	pdf.CellFormat(95, 8, "价税合计:", "1", 0, "R", true, 0, "")
	pdf.CellFormat(95, 8, fmt.Sprintf("¥%.2f", float64(total)/100), "1", 1, "L", true, 0, "")

	pdf.Ln(10)

	// ---- 签章区域 ----
	pdf.SetFont("Helvetica", "", 10)
	pdf.CellFormat(95, 7, "收款人: ____________", "", 0, "L", false, 0, "")
	pdf.CellFormat(95, 7, "开票人: ____________", "", 1, "L", false, 0, "")
	pdf.CellFormat(95, 7, "复核: ____________", "", 0, "L", false, 0, "")
	pdf.CellFormat(95, 7, "销售方 (签章):", "", 1, "L", false, 0, "")

	// 输出 PDF 到字节
	var buf []byte
	err := pdf.Output(&buf)
	if err != nil {
		return nil, fmt.Errorf("generate pdf: %w", err)
	}
	return buf, nil
}
```

### 5. 税额计算（VAT）

```go
package invoice

import (
	"github.com/shopspring/decimal"
)

// CalculateVAT 计算增值税
// 公式: 税额 = 含税金额 / (1 + 税率) * 税率
// 返回: 不含税金额, 税额
func CalculateVAT(amountFen int64, taxRatePercent int64) (noTaxAmountFen, taxAmountFen int64) {
	amount := decimal.NewFromInt(amountFen)
	rate := decimal.NewFromInt(taxRatePercent).Div(decimal.NewFromInt(100))

	// 不含税金额 = 含税金额 / (1 + 税率)
	noTaxAmount := amount.Div(decimal.NewFromInt(1).Add(rate))
	// 税额 = 含税金额 - 不含税金额
	taxAmount := amount.Sub(noTaxAmount)

	// 四舍五入到分
	return noTaxAmount.Round(0).IntPart(), taxAmount.Round(0).IntPart()
}

// CalculateVATWithRate 使用外部配置的税率计算
func CalculateVATWithRate(amountFen int64, rate decimal.Decimal) (noTaxAmountFen, taxAmountFen int64) {
	amount := decimal.NewFromInt(amountFen)
	noTaxAmount := amount.Div(decimal.NewFromInt(1).Add(rate))
	taxAmount := amount.Sub(noTaxAmount)
	return noTaxAmount.Round(0).IntPart(), taxAmount.Round(0).IntPart()
}
```

### 6. Excel 批量导出（供财务对账使用）

```go
package invoice

import (
	"fmt"
	"time"

	"github.com/xuri/excelize/v2"
)

// InvoiceRecord 导出用发票记录
type InvoiceRecord struct {
	InvoiceNo string
	OrderID   int64
	TenantID  string // 多租户隔离
	Type      string
	Title     string
	TaxNumber string
	Amount    int64 // 分，导出时转换为元
	TaxAmount int64 // 分，导出时转换为元
	Status    string
	IssuedAt  time.Time
	PDFURL    string
}

// ExportInvoicesToExcel 批量导出发票到 Excel
func ExportInvoicesToExcel(ctx context.Context, records []InvoiceRecord) ([]byte, error) {
	f := excelize.NewFile()

	sheetName := "发票明细"
	index, err := f.NewSheet(sheetName)
	if err != nil {
		return nil, fmt.Errorf("create sheet: %w", err)
	}
	f.SetActiveSheet(index)

	// 表头
	headers := []string{
		"发票号码", "订单号", "发票类型", "发票抬头", "纳税人识别号",
		"金额(元)", "税额(元)", "状态", "开具日期", "PDF链接",
	}
	for i, h := range headers {
		cell := fmt.Sprintf("%s1", string(rune('A'+i)))
		f.SetCellValue(sheetName, cell, h)
	}

	// 样式：表头加粗
	style, _ := f.NewStyle(&excelize.Style{
		Font: &excelize.Font{Bold: true},
		Fill: excelize.Fill{Type: "pattern", Color: []string{"#D9E2F3"}, Pattern: 1},
	})
	f.SetCellStyle(sheetName, "A1", "J1", style)

	// 数据行
	for i, r := range records {
		row := i + 2
		f.SetCellValue(sheetName, fmt.Sprintf("A%d", row), r.InvoiceNo)
		f.SetCellValue(sheetName, fmt.Sprintf("B%d", row), r.OrderID)
		f.SetCellValue(sheetName, fmt.Sprintf("C%d", row), r.Type)
		f.SetCellValue(sheetName, fmt.Sprintf("D%d", row), r.Title)
		f.SetCellValue(sheetName, fmt.Sprintf("E%d", row), r.TaxNumber)
		f.SetCellValue(sheetName, fmt.Sprintf("F%d", row), float64(r.Amount)/100)   // 分→元
		f.SetCellValue(sheetName, fmt.Sprintf("G%d", row), float64(r.TaxAmount)/100) // 分→元
		f.SetCellValue(sheetName, fmt.Sprintf("H%d", row), r.Status)
		f.SetCellValue(sheetName, fmt.Sprintf("I%d", row), r.IssuedAt.Format("2006-01-02 15:04:05"))
		f.SetCellValue(sheetName, fmt.Sprintf("J%d", row), r.PDFURL)
	}

	// 调整列宽
	f.SetColWidth(sheetName, "A", "J", 18)
	f.SetColWidth(sheetName, "D", "D", 30)

	if err := f.Close(); err != nil {
		return nil, fmt.Errorf("close excel: %w", err)
	}

	buf, err := f.WriteToBuffer()
	if err != nil {
		return nil, fmt.Errorf("write excel: %w", err)
	}
	return buf.Bytes(), nil
}
```

### 7. 发票邮件发送

```go
package invoice

import (
	"fmt"
	"time"

	"gopkg.in/gomail.v2"
)

// MailerConfig SMTP 邮件配置
// 生产环境：凭据必须通过 vault/环境变量注入，禁止硬编码在代码中。
type MailerConfig struct {
	SMTPHost string
	SMTPPort int
	User     string
	Password string // 生产环境请通过 vault 或环境变量注入
	SSL      bool
}

// SendInvoiceEmail 发送发票邮件（含 PDF 附件）
// 注意：gomail 不支持 context，通过 Dialer.Timeout 控制超时。
// ctx 参数保留以保持调用链一致，方便接入 tracing 系统。
func SendInvoiceEmail(ctx context.Context, to, subject, body string, pdfData []byte, pdfFileName string, cfg MailerConfig) error {
	m := gomail.NewMessage()

	m.SetHeader("From", cfg.User)
	m.SetHeader("To", to)
	m.SetHeader("Subject", subject)
	m.SetBody("text/html", body)

	// 附加 PDF 发票
	m.Attach(pdfFileName, gomail.SetCopyFunc(func(w gomail.Writer) error {
		_, err := w.Write(pdfData)
		return err
	}))

	d := gomail.NewDialer(cfg.SMTPHost, cfg.SMTPPort)
	d.SSL = cfg.SSL
	d.Timeout = 30 * time.Second
	d.Username = cfg.User
	d.Password = cfg.Password

	if err := d.DialAndSend(m); err != nil {
		return fmt.Errorf("send invoice email to %s: %w", to, err)
	}
	return nil
}

// BuildInvoiceEmailBody 构建发票邮件 HTML 正文
func BuildInvoiceEmailBody(inv *Invoice) string {
	return fmt.Sprintf(`
<html>
<body style="font-family: sans-serif; max-width: 600px;">
  <h2>您的电子发票已开具</h2>
  <p>尊敬的 %s，</p>
  <p>您的发票已开具完成，详细信息如下：</p>
  <table border="1" cellpadding="8" cellspacing="0" style="border-collapse: collapse;">
    <tr><td>发票号码</td><td>%s</td></tr>
    <tr><td>发票类型</td><td>%s</td></tr>
    <tr><td>金额</td><td>¥%.2f</td></tr>
    <tr><td>税额</td><td>¥%.2f</td></tr>
    <tr><td>价税合计</td><td>¥%.2f</td></tr>
  </table>
  <p>PDF 发票已作为附件附在本邮件中，请妥善保管。</p>
  <p>如有疑问，请联系客服。</p>
</body>
</html>
`, inv.Title, inv.InvoiceNo, inv.Type,
		float64(inv.Amount)/100,
		float64(inv.TaxAmount)/100,
		float64(inv.Amount+inv.TaxAmount)/100)
}
```

## 反模式 (Anti-Patterns)

| 反模式 | 问题 | 正确做法 |
|--------|------|----------|
| 使用 `float64` 做税务计算 | 浮点精度丢失，导致金额差 1 分 | 使用 `shopspring/decimal`，所有金额以"分"存储（int64） |
| 发票号码递增无并发控制 | 多线程/多实例下发票号码冲突 | 使用 Redis `INCR` 或数据库序列表（带行锁） |
| PDF 生成阻塞 HTTP 响应 | 大发票文件生成耗时，请求超时 | 异步生成 + 消息队列 + 邮件/短信通知 |
| 无红冲发票关联追踪 | 红冲后无法追溯原发票 | 红冲时必须记录 `void_ref_no` 指向原发票号码 |
| 发票状态随意修改 | 状态机混乱，产生脏数据 | 使用状态机约束，只允许合法转换 |
| 重复申请不幂等 | 同一订单重复开票导致金额重复计算 | 基于 `OrderID + TenantID` 做唯一约束，重复请求返回已有发票 |
| PII 明文存储 | 邮箱、手机号泄露 | 手机号 AES-256-GCM 加密，展示时脱敏（138****1234） |

## 常见陷阱 (Common Pitfalls)

1. **税率硬编码**
   - 中国税率会随政策调整（如增值税 13% / 9% / 6% 等）
   - **正确做法**: 税率外部化到配置中心或数据库，不可硬编码

   ```go
   // 错误
   taxRate := decimal.NewFromFloat(0.13) // 硬编码

   // 正确
   taxRate := config.GetTaxRate(inv.Type, productCategory) // 从配置读取
   ```

2. **PDF 渲染性能瓶颈**
   - 大量并发开具时，PDF 生成会占用大量 CPU 和内存
   - **正确做法**: 预渲染模板、使用连接池/Worker Pool 限制并发度

   ```go
   // Worker Pool 控制并发（配合 WaitGroup 确保所有 PDF 生成完成）
   var wg sync.WaitGroup
   sem := make(chan struct{}, 10) // 限制最大并发度
   for _, inv := range invoices {
       wg.Add(1)
       go func(inv *Invoice) {
           defer wg.Done()
           sem <- struct{}{}
           defer func() { <-sem }()
           // 异步任务使用衍生 context，确保父请求取消不影响 PDF 生成
           bgCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
           defer cancel()
           pdfData, err := GenerateInvoicePDF(bgCtx, inv, items)
           if err != nil {
               log.Printf("failed to generate PDF for invoice %d: %v", inv.ID, err)
               return
           }
           // 上传到对象存储或发送邮件
       }(inv)
   }
   wg.Wait() // 阻塞直到所有 PDF 生成完成
   ```

3. **企业发票需要税局对接**
   - 增值税专用发票需要与国家税务总局系统对接
   - **正确做法**: 使用官方税局 API 或第三方服务（如百望、航天信息），不自建开票引擎

   ```go
   // 通过第三方税控服务开具
   type TaxControlClient struct {
       BaseURL    string
       AppKey     string
       AppSecret  string
       httpClient *http.Client // 超时 10s，生产环境加 circuit breaker
   }

   // NewTaxControlClient 创建税控客户端（含超时配置）
   func NewTaxControlClient(baseURL, appKey, appSecret string) *TaxControlClient {
       return &TaxControlClient{
           BaseURL:   baseURL,
           AppKey:    appKey,
           AppSecret: appSecret,
           httpClient: &http.Client{
               Timeout: 10 * time.Second,
               Transport: &http.Transport{
                   MaxIdleConns:        10,
                   MaxIdleConnsPerHost: 5,
                   IdleConnTimeout:     30 * time.Second,
               },
           },
       }
   }

   func (c *TaxControlClient) IssueVATInvoice(ctx context.Context, req *VATInvoiceRequest) (*VATInvoiceResult, error) {
       // 调用百望/航天信息等第三方税控服务 API
       // 返回发票号码、PDF/OFD 文件
       // 注意：生产环境应配置 circuit breaker（如 github.com/sony/gobreaker）
   }
   ```

## go-zero 集成说明

> 本文档展示的是**业务逻辑层**代码模板。在 go-zero 项目中：
>
> 1. **API 定义**: 接口路由和请求/响应结构应写在 `.api` 文件中，通过 `goctl api go` 生成 handler 和 logic 骨架。
> 2. **Model 层**: `Invoice` 等内部模型仅在 logic/service 层使用，**绝不可直接作为 API 响应返回**。API 响应应使用独立的 DTO/Response 结构体，避免暴露内部字段（如 `DeletedAt`、`gorm.DeletedAt`）。
> 3. **Handler → Logic → Service**: Handler 接收请求后委托给 Logic 层，Logic 调用 Service（如状态机、PDF 生成、邮件发送），最后将结果映射为 API 响应结构体。
> 4. **goctl 生成**: 使用 `goctl api go -dir .` 生成 handler 和 logic，然后在 logic 层嵌入本文档中的业务逻辑。

## 测试策略

### 覆盖率目标

| 层级 | 覆盖率要求 | 说明 |
|------|-----------|------|
| L1 Unit | >= 80% | FSM 状态转换、PDF 模板渲染、发票号生成、税率计算 |
| L2 Integration | >= 70% | DB 事务 + 审计日志写入 + 多租户隔离 |
| L3 E2E | 核心流程 | 订单 → 开具发票 → PDF 生成 → 邮件发送完整流程 |

### FSM 状态转换测试

```go
func TestInvoiceFSM_Transitions(t *testing.T) {
	tests := []struct {
		name   string
		from   InvoiceStatus
		to     InvoiceStatus
		wantOK bool
	}{
		{"applied → issued", StatusApplied, StatusIssued, true},
		{"issued → sent", StatusIssued, StatusSent, true},
		{"issued → red_dashed", StatusIssued, StatusRedDashed, true},
		{"issued → voided", StatusIssued, StatusVoided, true},
		{"sent → red_dashed", StatusSent, StatusRedDashed, true},
		{"sent → voided", StatusSent, StatusVoided, true},
		{"applied → sent (illegal)", StatusApplied, StatusSent, false},
		{"red_dashed → issued (illegal)", StatusRedDashed, StatusIssued, false},
		{"voided → issued (illegal)", StatusVoided, StatusIssued, false},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got, err := tt.from.Transition(tt.to)
			if (err == nil) != tt.wantOK {
				t.Errorf("Transition(%q -> %q): err=%v, wantOK=%v", tt.from, tt.to, err, tt.wantOK)
			}
			if err == nil && got != tt.to {
				t.Errorf("Transition(%q -> %q): got=%q, want=%q", tt.from, tt.to, got, tt.to)
			}
		})
	}
}

func TestInvoiceFSM_CanVoidAndRedDash(t *testing.T) {
	// issued 和 sent 状态可以红冲和作废
	for _, s := range []InvoiceStatus{StatusIssued, StatusSent} {
		if !s.CanVoid() {
			t.Errorf("%s.CanVoid() = false, want true", s)
		}
		if !s.CanRedDash() {
			t.Errorf("%s.CanRedDash() = false, want true", s)
		}
	}
	// applied 状态不能红冲或作废
	for _, s := range []InvoiceStatus{StatusApplied, StatusRedDashed, StatusVoided} {
		if s.CanVoid() {
			t.Errorf("%s.CanVoid() = true, want false", s)
		}
		if s.CanRedDash() {
			t.Errorf("%s.CanRedDash() = true, want false", s)
		}
	}
}
```

### PDF 模板渲染测试

```go
func TestPDFGenerator_RenderTemplate(t *testing.T) {
	inv := &Invoice{
		InvoiceNo: "INV202604000001",
		Title:     "测试公司",
		Type:      InvoiceTypeEnterprise,
		Amount:    10000, // ¥100.00
		TaxAmount: 600,   // ¥6.00
	}
	items := []InvoiceItem{
		{Name: "商品 A", Spec: "规格1", Quantity: "2", UnitPrice: "50.00", Amount: "100.00", TaxRate: "6%"},
	}
	pdf, err := GenerateInvoicePDF(context.Background(), inv, items)
	if err != nil {
		t.Fatalf("Generate PDF: %v", err)
	}
	if len(pdf) == 0 {
		t.Fatal("PDF is empty")
	}
	// 验证 PDF 头部魔术字节 %PDF-
	if !bytes.HasPrefix(pdf, []byte("%PDF-")) {
		t.Errorf("PDF magic bytes: got %q, want %%PDF-", pdf[:5])
	}
}
```

### 发票号并发安全测试

```go
func TestInvoiceNoGenerator_ConcurrentRedis(t *testing.T) {
	// 使用 miniredis 测试 Redis INCR 发票号生成
	mr := miniredis.NewMiniRedis()
	tenantID := "TENANT-001"

	var wg sync.WaitGroup
	results := make([]string, 100)
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func(idx int) {
			defer wg.Done()
			now := time.Date(2026, 4, 1, 0, 0, 0, 0, time.UTC)
			no, err := GenerateInvoiceNoWithRedis(context.Background(), mr, tenantID, now)
			if err != nil {
				t.Errorf("goroutine %d: %v", idx, err)
				return
			}
			results[idx] = no
		}(i)
	}
	wg.Wait()

	// 验证无重复
	seen := make(map[string]bool)
	for _, no := range results {
		if no == "" {
			continue
		}
		if seen[no] {
			t.Errorf("duplicate invoice no: %s", no)
		}
		seen[no] = true
	}
}
```

### 多租户隔离测试

```go
func TestInvoiceService_MultiTenantIsolation(t *testing.T) {
	// 验证 tenantA 无法查询 tenantB 的发票
	// tenantA 开票后 tenantB 不可见
	// 发票号前缀按租户隔离
}
```
