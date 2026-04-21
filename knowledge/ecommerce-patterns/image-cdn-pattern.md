# Image CDN Patterns（图片 CDN 策略）

## 推荐第三方库

| 用途                             | 推荐库 | 导入路径                              |
|----------------------------------|--------|---------------------------------------|
| 高级图片处理（WebP/AVIF/多格式） | govips | `github.com/davidbyttow/govips`       |
| SSRF 防护                        | 标准库 | `net/url`, `net/http`, `net`          |

## 触发条件（When to use）

- 电商平台需要展示商品图片、用户头像、Banner 等图片资源
- 用户上传自定义图片（商品主图、详情页图片、评价图片）
- 需要支持多端适配（PC 大屏、手机小屏、小程序、Flutter App）
- 需要防止 SSRF 攻击（用户提交的图片 URL 可能被用来探测内网）

## 决策树（Decision tree）

```
图片来源是什么？
├── 用户上传 URL
│   └── 必须 SSRF 防护（白名单 + 非私有 IP 校验 + 重定向跟踪）
│       └── 校验通过 → 下载 → 转 WebP/AVIF → 上传 CDN → 返回 CDN URL
│
├── 管理后台上传文件
│   └── 直接上传 OSS/CDN → 转 WebP/AVIF → 返回 CDN URL
│
└── 第三方图片链接
    └── 是否信任源？
        ├── 是 → 直接使用 + CDN 缓存
        └── 否 → 走 SSRF 防护流程

图片格式如何选择？
├── 浏览器支持 AVIF？→ 使用 AVIF（体积最小）
├── 浏览器支持 WebP？→ 使用 WebP（兼容性好）
└── 都不支持？→ 使用 JPEG/PNG 原始格式（降级）

图片尺寸如何选择？
├── PC 列表页：600x600
├── PC 详情页：1200x1200
├── 手机端：400x400
├── 缩略图：100x100
└── 原图：保留（不公开直接访问）
```

## 代码模板（Code template）

### 1. SSRF 防护 — 图片 URL 验证器

```go
// imagecdn/validator.go
package imagecdn

import (
	"context"
	"errors"
	"fmt"
	"io"
	"net"
	"net/http"
	"net/url"
	"strings"
	"time"

	"storeforge/pkg/logx"
)

var (
	ErrInvalidURL        = errors.New("invalid image URL format")
	ErrSchemeNotAllowed  = errors.New("only HTTP/HTTPS protocol allowed")
	ErrDomainNotAllowed  = errors.New("image domain not in allowlist")
	ErrPrivateIP         = errors.New("image address points to private/internal IP")
	ErrLoopbackIP        = errors.New("image address points to loopback address")
	ErrRedirectBlocked   = errors.New("redirect target not in allowlist")
	ErrImageTooLarge     = errors.New("image size exceeds limit")
	ErrContentTypeBad    = errors.New("content type is not a supported image format")
	ErrDownloadFailed    = errors.New("image download failed")
)

// ImageURLValidator 图片 URL 安全验证器
type ImageURLValidator struct {
	allowedDomains map[string]bool
	maxSizeBytes   int64
	maxRedirects   int
	httpClient     *http.Client
	// 安全校验用的 DNS 解析器
	resolver *net.Resolver
}

// ValidatorOption 配置选项
type ValidatorOption func(*ImageURLValidator)

func WithMaxSizeBytes(size int64) ValidatorOption {
	return func(v *ImageURLValidator) { v.maxSizeBytes = size }
}

func WithMaxRedirects(n int) ValidatorOption {
	return func(v *ImageURLValidator) { v.maxRedirects = n }
}

// NewImageURLValidator 创建验证器
// allowedDomains: 允许的图片域名白名单（如 "images.example.com", "cdn.example.com"）
func NewImageURLValidator(allowedDomains []string, opts ...ValidatorOption) *ImageURLValidator {
	v := &ImageURLValidator{
		allowedDomains: make(map[string]bool),
		maxSizeBytes:   10 << 20, // 默认 10MB
		maxRedirects:   3,
		resolver:       net.DefaultResolver,
	}

	for _, d := range allowedDomains {
		v.allowedDomains[strings.ToLower(d)] = true
	}

	for _, opt := range opts {
		opt(v)
	}

	// 配置 HTTP 客户端：自定义 DialContext 原子化 DNS 解析 + IP 校验 + 连接
	// 防止 TOCTOU (Time-Of-Check-Time-Of-Use) DNS Rebinding 攻击
	transport := &http.Transport{
		DialContext: func(ctx context.Context, network, addr string) (net.Conn, error) {
			host, port, err := net.SplitHostPort(addr)
			if err != nil {
				return nil, err
			}
			ips, err := v.resolver.LookupIPAddr(ctx, host)
			if err != nil {
				return nil, fmt.Errorf("DNS resolution failed for %s: %w", host, err)
			}
			for _, ipAddr := range ips {
				ip := ipAddr.IP
				if ip.IsLoopback() || ip.IsPrivate() || ip.IsLinkLocalUnicast() || isCGNAT(ip) {
					return nil, fmt.Errorf("blocked address %s: points to restricted IP", ip)
				}
			}
			// 使用校验通过的第一个 IP 直连
			dialer := &net.Dialer{}
			targetIP := ips[0].IP.String()
			return dialer.DialContext(ctx, network, net.JoinHostPort(targetIP, port))
		},
	}

	v.httpClient = &http.Client{
		Timeout:   10 * time.Second,
		Transport: transport,
		CheckRedirect: func(req *http.Request, via []*http.Request) error {
			if len(via) >= v.maxRedirects {
				return http.ErrUseLastResponse
			}
			return nil
		},
	}

	return v
}

// Validate 验证图片 URL 安全性并下载
// 返回图片二进制数据 + Content-Type
func (v *ImageURLValidator) Validate(ctx context.Context, rawURL string) ([]byte, string, error) {
	// 1. 解析 URL
	parsed, err := url.Parse(rawURL)
	if err != nil {
		return nil, "", fmt.Errorf("%w: %v", ErrInvalidURL, err)
	}

	// 2. 协议检查
	if parsed.Scheme != "http" && parsed.Scheme != "https" {
		return nil, "", ErrSchemeNotAllowed
	}

	// 3. 域名白名单检查
	domain := strings.ToLower(parsed.Hostname())
	if !v.allowedDomains[domain] {
		return nil, "", fmt.Errorf("%w: %s", ErrDomainNotAllowed, domain)
	}

	// 4. DNS 解析 + IP 白名单/黑名单检查
	if err := v.validateIP(ctx, parsed.Hostname()); err != nil {
		return nil, "", err
	}

	// 5. 下载图片（手动处理重定向，每次重定向都校验 IP）
	data, contentType, err := v.downloadWithRedirectCheck(ctx, rawURL, 0)
	if err != nil {
		return nil, "", err
	}

	// 6. 检查文件大小
	if int64(len(data)) > v.maxSizeBytes {
		return nil, "", fmt.Errorf("%w: %d > %d", ErrImageTooLarge, len(data), v.maxSizeBytes)
	}

	// 7. 检查 Content-Type
	if !isAllowedContentType(contentType) {
		return nil, "", fmt.Errorf("%w: %s", ErrContentTypeBad, contentType)
	}

	return data, contentType, nil
}

// validateIP 检查解析出的 IP 是否为私有/回环地址
func (v *ImageURLValidator) validateIP(ctx context.Context, hostname string) error {
	ips, err := v.resolver.LookupIPAddr(ctx, hostname)
	if err != nil {
		return fmt.Errorf("DNS resolution failed for %s: %w", hostname, err)
	}

	for _, ipAddr := range ips {
		ip := ipAddr.IP
		if ip.IsLoopback() {
			return ErrLoopbackIP
		}
		if ip.IsPrivate() {
			return ErrPrivateIP
		}
		// Link-local unicast (169.254.0.0/16)
		if ip.IsLinkLocalUnicast() {
			return ErrPrivateIP
		}
		// Carrier-grade NAT (100.64.0.0/10)
		if isCGNAT(ip) {
			return ErrPrivateIP
		}
	}

	return nil
}

// isCGNAT 检查是否为 Carrier-grade NAT 地址 (100.64.0.0/10)
func isCGNAT(ip net.IP) bool {
	cgnat := &net.IPNet{
		IP:   net.ParseIP("100.64.0.0"),
		Mask: net.CIDRMask(10, 32),
	}
	return cgnat.Contains(ip)
}

// downloadWithRedirectCheck 手动处理重定向，每次重定向都校验目标 IP
func (v *ImageURLValidator) downloadWithRedirectCheck(ctx context.Context, rawURL string, depth int) ([]byte, string, error) {
	if depth > v.maxRedirects {
		logx.WithContext(ctx).Errorf("redirect chain too deep: %s (depth=%d)", rawURL, depth)
		return nil, "", ErrRedirectBlocked
	}

	req, err := http.NewRequestWithContext(ctx, http.MethodGet, rawURL, nil)
	if err != nil {
		logx.WithContext(ctx).Errorf("failed to build request for %s: %v", rawURL, err)
		return nil, "", fmt.Errorf("%w: %v", ErrInvalidURL, err)
	}
	req.Header.Set("User-Agent", "StoreForge-ImageFetcher/1.0")

	resp, err := v.httpClient.Do(req)
	if err != nil {
		logx.WithContext(ctx).Errorf("request failed for %s: %v", rawURL, err)
		return nil, "", fmt.Errorf("%w: %v", ErrDownloadFailed, err)
	}

	// 处理重定向
	if resp.StatusCode == http.StatusMovedPermanently ||
		resp.StatusCode == http.StatusFound ||
		resp.StatusCode == http.StatusSeeOther ||
		resp.StatusCode == http.StatusTemporaryRedirect ||
		resp.StatusCode == http.StatusPermanentRedirect {

		location := resp.Header.Get("Location")
		if location == "" {
			io.Copy(io.Discard, resp.Body)
			resp.Body.Close()
			logx.WithContext(ctx).Errorf("redirect with no Location header: %s", rawURL)
			return nil, "", ErrRedirectBlocked
		}

		// 解析重定向目标
		redirectURL, err := url.Parse(location)
		if err != nil {
			io.Copy(io.Discard, resp.Body)
			resp.Body.Close()
			logx.WithContext(ctx).Errorf("failed to parse redirect location: %v", err)
			return nil, "", fmt.Errorf("%w: %v", ErrInvalidURL, err)
		}

		// 如果是相对路径，拼接
		if !redirectURL.IsAbs() {
			redirectURL = resp.Request.URL.ResolveReference(redirectURL)
		}

		// 重定向目标必须也在白名单内
		redirectDomain := strings.ToLower(redirectURL.Hostname())
		if !v.allowedDomains[redirectDomain] {
			io.Copy(io.Discard, resp.Body)
			resp.Body.Close()
			logx.WithContext(ctx).Errorf("redirect target %s not in allowlist", redirectDomain)
			return nil, "", fmt.Errorf("%w: redirect to %s", ErrRedirectBlocked, redirectDomain)
		}

		// 重定向目标 IP 校验
		if err := v.validateIP(ctx, redirectURL.Hostname()); err != nil {
			io.Copy(io.Discard, resp.Body)
			resp.Body.Close()
			return nil, "", err
		}

		// 排空 body 并关闭，然后递归下载
		io.Copy(io.Discard, resp.Body)
		resp.Body.Close()
		return v.downloadWithRedirectCheck(ctx, redirectURL.String(), depth+1)
	}

	if resp.StatusCode != http.StatusOK {
		io.Copy(io.Discard, resp.Body)
		resp.Body.Close()
		logx.WithContext(ctx).Errorf("unexpected status code %d for %s", resp.StatusCode, rawURL)
		return nil, "", fmt.Errorf("%w: HTTP %d", ErrDownloadFailed, resp.StatusCode)
	}

	// 读取 body（限制大小，检测截断）
	reader := io.LimitReader(resp.Body, v.maxSizeBytes+1)
	data, err := io.ReadAll(reader)
	if err != nil {
		resp.Body.Close()
		logx.WithContext(ctx).Errorf("failed to read response body: %v", err)
		return nil, "", fmt.Errorf("%w: %v", ErrDownloadFailed, err)
	}
	resp.Body.Close()

	// 检测是否被截断（超出限制的部分被丢弃）
	if int64(len(data)) > v.maxSizeBytes {
		return nil, "", ErrImageTooLarge
	}

	// 魔术数字校验（防止 Content-Type 伪造）
	if err := validateMagicNumber(data, resp.Header.Get("Content-Type")); err != nil {
		return nil, "", err
	}

	return data, resp.Header.Get("Content-Type"), nil
}

// validateMagicNumber 校验图片文件的魔术数字
func validateMagicNumber(data []byte, contentType string) error {
	contentType = strings.ToLower(strings.TrimSpace(contentType))
	switch {
	case strings.HasPrefix(contentType, "image/jpeg"):
		if len(data) < 2 || data[0] != 0xFF || data[1] != 0xD8 {
			return ErrContentTypeBad
		}
	case strings.HasPrefix(contentType, "image/png"):
		if len(data) < 4 || data[0] != 0x89 || data[1] != 0x50 || data[2] != 0x4E || data[3] != 0x47 {
			return ErrContentTypeBad
		}
	case strings.HasPrefix(contentType, "image/webp"):
		if len(data) < 4 || string(data[:4]) != "RIFF" {
			return ErrContentTypeBad
		}
	case strings.HasPrefix(contentType, "image/gif"):
		if len(data) < 3 || string(data[:3]) != "GIF" {
			return ErrContentTypeBad
		}
	}
	// SVG/AVIF 跳过魔术数字校验（格式复杂，不强制）
	return nil
}

// isAllowedContentType 检查是否为支持的图片格式
func isAllowedContentType(ct string) bool {
	ct = strings.ToLower(strings.TrimSpace(ct))
	allowed := []string{
		"image/jpeg",
		"image/png",
		"image/gif",
		"image/webp",
		"image/avif",
		"image/svg+xml",
	}
	for _, a := range allowed {
		if ct == a || strings.HasPrefix(ct, a+";") {
			return true
		}
	}
	return false
}
```

### 2. WebP/AVIF 格式转换服务

```go
// imagecdn/converter.go
package imagecdn

import (
	"bytes"
	"fmt"
	"image"
	_ "image/jpeg"
	_ "image/png"

	"github.com/davidbyttow/govips/v2/vips"
)

// ImageFormat 图片格式
type ImageFormat string

const (
	FormatJPEG ImageFormat = "jpeg"
	FormatPNG  ImageFormat = "png"
	FormatWebP ImageFormat = "webp"
	FormatAVIF ImageFormat = "avif"
	FormatSVG  ImageFormat = "svg"
)

// ImageConverter 图片格式转换器
// 使用 govips（libvips 的 Go binding），纯 Go 调用，无外部命令行依赖
type ImageConverter struct{}

type ConverterOption func(*ImageConverter)

func init() {
	// govips must be initialized exactly once per process lifetime.
	vips.Startup(nil)
}

func NewImageConverter(opts ...ConverterOption) *ImageConverter {
	return &ImageConverter{}
}

// ConvertResult 转换结果
type ConvertResult struct {
	Data           []byte
	Format         ImageFormat
	Width          int
	Height         int
	OrigSize       int
	CompressedSize int
}

// ConvertToWebP 转换为 WebP 格式
func (c *ImageConverter) ConvertToWebP(input []byte, quality int) (*ConvertResult, error) {
	if quality <= 0 || quality > 100 {
		quality = 80
	}

	img, err := vips.NewImageFromBuffer(input)
	if err != nil {
		return nil, fmt.Errorf("govips: failed to load image: %w", err)
	}
	defer img.Close()

	exportParams := &vips.WebPExportParams{
		Quality: quality,
	}
	data, _, err := img.ExportWebP(exportParams)
	if err != nil {
		return nil, fmt.Errorf("govips: webp export failed: %w", err)
	}

	return &ConvertResult{
		Data:           data,
		Format:         FormatWebP,
		Width:          img.Width(),
		Height:         img.Height(),
		OrigSize:       len(input),
		CompressedSize: len(data),
	}, nil
}

// ConvertToAVIF 转换为 AVIF 格式
func (c *ImageConverter) ConvertToAVIF(input []byte, quality int) (*ConvertResult, error) {
	if quality <= 0 || quality > 100 {
		quality = 60
	}

	img, err := vips.NewImageFromBuffer(input)
	if err != nil {
		return nil, fmt.Errorf("govips: failed to load image: %w", err)
	}
	defer img.Close()

	exportParams := &vips.AvifExportParams{
		Quality: quality,
		Speed:   6, // 6=较快，质量损失极小
	}
	data, _, err := img.ExportAvif(exportParams)
	if err != nil {
		return nil, fmt.Errorf("govips: avif export failed: %w", err)
	}

	return &ConvertResult{
		Data:           data,
		Format:         FormatAVIF,
		Width:          img.Width(),
		Height:         img.Height(),
		OrigSize:       len(input),
		CompressedSize: len(data),
	}, nil
}

// Resize 图片缩放
func (c *ImageConverter) Resize(input []byte, width, height int, format ImageFormat, quality int) (*ConvertResult, error) {
	img, err := vips.NewImageFromBuffer(input)
	if err != nil {
		return nil, fmt.Errorf("govips: failed to load image: %w", err)
	}
	defer img.Close()

	// 缩放到目标尺寸，保持宽高比
	err = img.Thumbnail(width, height, vips.InterestingAttention)
	if err != nil {
		return nil, fmt.Errorf("govips: resize failed: %w", err)
	}

	var data []byte
	switch format {
	case FormatWebP:
		data, _, err = img.ExportWebP(&vips.WebPExportParams{Quality: quality})
	case FormatAVIF:
		data, _, err = img.ExportAvif(&vips.AvifExportParams{Quality: quality, Speed: 6})
	case FormatPNG:
		data, _, err = img.ExportPng(vips.NewDefaultPNGExportParams())
	default:
		data, _, err = img.ExportJpeg(&vips.JpegExportParams{Quality: quality})
	}
	if err != nil {
		return nil, fmt.Errorf("govips: %s export failed: %w", format, err)
	}

	return &ConvertResult{
		Data:           data,
		Format:         format,
		Width:          img.Width(),
		Height:         img.Height(),
		OrigSize:       len(input),
		CompressedSize: len(data),
	}, nil
}

// DetectFormat 自动检测图片格式
func DetectFormat(data []byte) (ImageFormat, error) {
	// SVG 优先检测（标准库 image.DecodeConfig 不支持 SVG）
	trimmed := bytes.TrimSpace(data)
	if bytes.HasPrefix(trimmed, []byte("<svg")) || bytes.HasPrefix(trimmed, []byte("<?xml")) {
		return FormatSVG, nil
	}
	_, format, err := image.DecodeConfig(bytes.NewReader(data))
	if err != nil {
		return "", fmt.Errorf("detect format failed: %w", err)
	}
	return ImageFormat(format), nil
}

// Shutdown 关闭 govips（应用退出时调用）
func Shutdown() {
	vips.Shutdown()
}
```

### 3. CDN URL 生成器 + 响应式图片 HTML

```go
// imagecdn/cdn.go
package imagecdn

import (
	"fmt"
	"net/url"
	"strings"
)

// CDNConfig CDN 配置
type CDNConfig struct {
	BaseURL    string // CDN 基础 URL，如 "https://cdn.example.com"
	BucketName string // OSS bucket 名称
	Secret     string // CDN 鉴权密钥（可选）
}

// CDNImageURL 生成处理后的图片 CDN URL
// 使用 CDN 图片处理服务（如阿里云 OSS / 腾讯云 CI / Cloudflare Images）
// 强制多租户隔离：所有 objectKey 自动带 tenantID 前缀
type CDNImageURL struct {
	cfg      CDNConfig
	tenantID string
}

func NewCDNImageURL(cfg CDNConfig, tenantID string) *CDNImageURL {
	return &CDNImageURL{cfg: cfg, tenantID: tenantID}
}

// ImageOption 图片处理选项
type ImageOption func(url.Values)

// WithWidth 指定宽度（高度按比例缩放）
func WithWidth(w int) ImageOption {
	return func(v url.Values) {
		v.Set("w", fmt.Sprintf("%d", w))
	}
}

// WithHeight 指定高度
func WithHeight(h int) ImageOption {
	return func(v url.Values) {
		v.Set("h", fmt.Sprintf("%d", h))
	}
}

// WithFormat 指定输出格式
func WithFormat(f ImageFormat) ImageOption {
	return func(v url.Values) {
		v.Set("fmt", string(f))
	}
}

// WithQuality 指定质量
func WithQuality(q int) ImageOption {
	return func(v url.Values) {
		v.Set("q", fmt.Sprintf("%d", q))
	}
}

// WithAutoFormat 自动选择最优格式（根据浏览器 Accept 头）
func WithAutoFormat() ImageOption {
	return func(v url.Values) {
		v.Set("fmt", "auto")
	}
}

// Generate 生成 CDN 图片 URL
// objectKey: OSS 中的对象 key，如 "products/2024/01/sku_12345.jpg"
// tenantID 已自动前置到 key 中（多租户隔离）
func (c *CDNImageURL) Generate(objectKey string, opts ...ImageOption) string {
	key := fmt.Sprintf("%s/%s", c.tenantID, objectKey)
	params := url.Values{}
	for _, opt := range opts {
		opt(params)
	}

	query := params.Encode()
	if query != "" {
		return fmt.Sprintf("%s/%s?%s", strings.TrimRight(c.cfg.BaseURL, "/"), key, query)
	}
	return fmt.Sprintf("%s/%s", strings.TrimRight(c.cfg.BaseURL, "/"), key)
}

// GenerateResponsiveSet 生成响应式图片 URL 集合
func (c *CDNImageURL) GenerateResponsiveSet(objectKey string, baseFormat ImageFormat) map[string]string {
	sizes := map[string][]ImageOption{
		"100w":  {WithWidth(100), WithFormat(baseFormat), WithQuality(70)},
		"400w":  {WithWidth(400), WithFormat(baseFormat), WithQuality(75)},
		"600w":  {WithWidth(600), WithFormat(baseFormat), WithQuality(80)},
		"800w":  {WithWidth(800), WithFormat(baseFormat), WithQuality(80)},
		"1200w": {WithWidth(1200), WithFormat(baseFormat), WithQuality(85)},
		"original": {WithFormat(baseFormat), WithQuality(90)},
	}

	result := make(map[string]string)
	for name, sizeOpts := range sizes {
		result[name] = c.Generate(objectKey, sizeOpts...)
	}
	return result
}
```

### Tenant-scoped caching and CDN object keys

> **Enforced in code**: `NewCDNImageURL(cfg, tenantID)` requires `tenantID`. `Generate()` auto-prepends it to all object keys.
>
> - **Redis cache keys**: Must be prefixed with `tenantID:`, e.g. `tenant_42:image:meta:products/123/main.webp`. This prevents cross-tenant cache pollution.
> - **CDN object keys**: Auto-prefixed by `Generate()` method: `objectKey = fmt.Sprintf("%s/%s", c.tenantID, key)`.
> - **Storage isolation**: Different tenants' images are isolated at the storage layer automatically.

### 4. 响应式图片 HTML 模板

```html
<!-- templates/product-image.html -->
<!-- 响应式图片 + WebP/AVIF 自动降级 + 懒加载 -->

<picture>
  <!-- AVIF 格式（最高效，Chrome 91+, Firefox 93+, Safari 16+） -->
  <source
    type="image/avif"
    srcset="{{.CDNURL "products/123/main.avif" "100w"}},
            {{.CDNURL "products/123/main.avif" "400w"}} 400w,
            {{.CDNURL "products/123/main.avif" "600w"}} 600w,
            {{.CDNURL "products/123/main.avif" "800w"}} 800w,
            {{.CDNURL "products/123/main.avif" "1200w"}} 1200w"
    sizes="(max-width: 640px) 100vw,
           (max-width: 1024px) 50vw,
           33vw"
  >

  <!-- WebP 格式（广泛支持，Chrome 32+, Firefox 65+, Safari 14+） -->
  <source
    type="image/webp"
    srcset="{{.CDNURL "products/123/main.webp" "100w"}},
            {{.CDNURL "products/123/main.webp" "400w"}} 400w,
            {{.CDNURL "products/123/main.webp" "600w"}} 600w,
            {{.CDNURL "products/123/main.webp" "800w"}} 800w,
            {{.CDNURL "products/123/main.webp" "1200w"}} 1200w"
    sizes="(max-width: 640px) 100vw,
           (max-width: 1024px) 50vw,
           33vw"
  >

  <!-- JPEG 降级格式（兜底，所有浏览器） -->
  <img
    src="{{.CDNURL "products/123/main.jpg" "600w"}}"
    alt="{{.ProductName}}"
    loading="lazy"
    decoding="async"
    width="800"
    height="800"
    style="aspect-ratio: 1 / 1; object-fit: cover; background-color: #f5f5f5;"
    srcset="{{.CDNURL "products/123/main.jpg" "400w"}} 400w,
            {{.CDNURL "products/123/main.jpg" "600w"}} 600w,
            {{.CDNURL "products/123/main.jpg" "800w"}} 800w,
            {{.CDNURL "products/123/main.jpg" "1200w"}} 1200w"
    sizes="(max-width: 640px) 100vw,
           (max-width: 1024px) 50vw,
           33vw"
  >
</picture>

<!-- 首屏商品图（above-the-fold）不使用 lazy loading -->
<picture>
  <source type="image/avif" srcset="{{.CDNURL "hero/banner.avif" "1200w"}}" media="(min-width: 1024px)">
  <source type="image/webp" srcset="{{.CDNURL "hero/banner.webp" "1200w"}}" media="(min-width: 1024px)">
  <img
    src="{{.CDNURL "hero/banner.jpg" "800w"}}"
    alt="促销活动"
    loading="eager"
    fetchpriority="high"
    width="1200"
    height="400"
    style="width: 100%; height: auto;"
  >
</picture>

<!-- 商品列表缩略图（高度懒加载） -->
<img
  src="{{.CDNURL .ImageKey "100w"}}"
  alt="{{.ProductName}}"
  loading="lazy"
  decoding="async"
  width="100"
  height="100"
  style="aspect-ratio: 1 / 1; object-fit: cover;"
>
```

### 5. Next.js SSRF 防护中间件

```typescript
// middleware/imageProxy.ts
// Next.js route handler — 代理用户提交的图片 URL，带 SSRF 防护

import { NextRequest, NextResponse } from 'next/server'

// 允许的域名白名单
const ALLOWED_DOMAINS = new Set([
  'images.example.com',
  'cdn.example.com',
  'img.example.com',
  'upload.example.com',
])

// 禁止的 IP 范围（私有/回环/保留）
function isPrivateIP(ip: string): boolean {
  // 简单判断，生产环境使用更完整的 CIDR 匹配
  const parts = ip.split('.').map(Number)
  if (parts.length !== 4) return true

  // 127.0.0.0/8
  if (parts[0] === 127) return true
  // 10.0.0.0/8
  if (parts[0] === 10) return true
  // 172.16.0.0/12
  if (parts[0] === 172 && parts[1] >= 16 && parts[1] <= 31) return true
  // 192.168.0.0/16
  if (parts[0] === 192 && parts[1] === 168) return true
  // 169.254.0.0/16
  if (parts[0] === 169 && parts[1] === 254) return true
  // 0.0.0.0
  if (parts[0] === 0) return true
  // ::1 / localhost
  if (ip === '::1' || ip === 'localhost') return true

  return false
}

export async function GET(request: NextRequest) {
  const imageUrl = request.nextUrl.searchParams.get('url')

  if (!imageUrl) {
    return NextResponse.json(
      { error: 'Missing url parameter' },
      { status: 400 }
    )
  }

  // 1. 解析 URL
  let parsed: URL
  try {
    parsed = new URL(imageUrl)
  } catch {
    return NextResponse.json(
      { error: 'Invalid URL format' },
      { status: 400 }
    )
  }

  // 2. 协议检查
  if (parsed.protocol !== 'http:' && parsed.protocol !== 'https:') {
    return NextResponse.json(
      { error: 'Only HTTP/HTTPS protocol allowed' },
      { status: 403 }
    )
  }

  // 3. 域名白名单检查
  const domain = parsed.hostname.toLowerCase()
  if (!ALLOWED_DOMAINS.has(domain)) {
    return NextResponse.json(
      { error: `Domain ${domain} not in allowlist` },
      { status: 403 }
    )
  }

  // 4. DNS 解析 + IP 检查（Node.js 环境）
  const dns = await import('node:dns').then(m => m.promises)
  try {
    const addresses = await dns.lookup(domain)
    const ip = typeof addresses === 'string' ? addresses : addresses.address
    if (isPrivateIP(ip)) {
      return NextResponse.json(
        { error: 'Image address points to private/reserved IP, access denied' },
        { status: 403 }
      )
    }
  } catch {
    return NextResponse.json(
      { error: 'DNS resolution failed' },
      { status: 400 }
    )
  }

  // 5. 使用自定义 Agent 代理图片（TOCTOU-safe）
  // 通过自定义 lookup 函数在 DNS 解析 + 连接建立之间原子化校验 IP
  const https = await import('node:https')
  const http = await import('node:http')
  const net = await import('node:net')

  const safeLookup = (
    hostname: string,
    options: net.LookupAddressOptions,
    callback: net.LookupCallback
  ) => {
    net.lookup(hostname, options, (err, addresses, family) => {
      if (err) { callback(err, [], undefined); return }
      const addrs = Array.isArray(addresses) ? addresses : [{ address: addresses as string, family }]
      for (const addr of addrs) {
        if (isPrivateIP(addr.address)) {
          callback(new Error('Blocked: private/reserved IP'), [], undefined)
          return
        }
      }
      callback(null, addrs, family)
    })
  }

  const agentOptions = { lookup: safeLookup }
  const agent = parsed.protocol === 'https:'
    ? new https.Agent(agentOptions)
    : new http.Agent(agentOptions)

  // 用 Promise 包装 node:http 回调式请求，同时捕获 headers
  type ProxyResult = { body: Buffer; contentType: string }
  const result = await new Promise<ProxyResult>((resolve, reject) => {
    const client = parsed.protocol === 'https:' ? https : http
    const req = client.get(imageUrl, { agent }, (res) => {
      const chunks: Buffer[] = []
      res.on('data', (chunk: Buffer) => { chunks.push(chunk) })
      res.on('end', () => resolve({
        body: Buffer.concat(chunks),
        contentType: res.headers['content-type'] || 'application/octet-stream',
      }))
      res.on('error', reject)
    })
    req.on('error', reject)
  })

  if (result.body.byteLength > 10 * 1024 * 1024) {
    return NextResponse.json({ error: 'Image exceeds size limit' }, { status: 413 })
  }

  return new NextResponse(result.body, {
    headers: {
      'Content-Type': result.contentType,
      'Cache-Control': 'public, max-age=86400',
    },
  })
}
```

### 6. CDN 缓存策略配置（Nginx 示例）

```nginx
# nginx/cdn-cache.conf

# 商品图片缓存策略
location ~* ^/products/.*\.(jpg|jpeg|png|webp|avif|gif|svg)$ {
    # 强缓存：图片不变则长期缓存
    expires 30d;
    add_header Cache-Control "public, immutable, max-age=2592000";

    # CDN 回源配置
    proxy_cache IMAGE_CACHE;
    proxy_cache_valid 200 30d;
    proxy_cache_valid 404 1m;
    proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;

    # 按格式缓存
    proxy_cache_key "$uri$args";

    # 安全头
    add_header X-Content-Type-Options "nosniff" always;
    add_header Content-Security-Policy "default-src 'none'" always;
}

# 用户头像（更新频率高）
location ~* ^/avatars/.*\.(jpg|jpeg|png|webp)$ {
    expires 1h;
    add_header Cache-Control "public, max-age=3600, stale-while-revalidate=86400";

    # ETag 支持
    etag on;

    proxy_cache AVATAR_CACHE;
    proxy_cache_valid 200 1h;
    proxy_cache_valid 404 1m;
}

# 缩略图/动态裁剪图片
location ~* ^/images/thumb/ {
    expires 7d;
    add_header Cache-Control "public, max-age=604800";

    proxy_cache THUMB_CACHE;
    proxy_cache_valid 200 7d;
    proxy_cache_valid 404 5m;
}
```

## 反模式（Anti-patterns）

| 反模式 | 说明 | 后果 |
|--------|------|------|
| 直接返回原始图片 | 不压缩、不转换格式，直接上传/返回原图 | 带宽浪费 5-10x，页面加载极慢 |
| 无 SSRF 防护 | 用户提交的 URL 直接作为请求目标 | 攻击者探测内网、读取 K8s metadata、SSRF 攻击 |
| 不验证图片 Content-Type | 接受任意 Content-Type | 用户上传可执行文件冒充图片 |
| 不限制图片大小 | 允许上传任意大小的图片 | 内存耗尽、DOS 攻击 |
| 无 CDN 缓存 | 每次都从源站回源 | 源站压力巨大，加载慢 |
| 不用响应式图片 | 所有设备加载同一张大图 | 手机端浪费 90% 流量 |
| 不用懒加载 | 页面底部图片也立即加载 | 首屏加载时间翻倍 |
| 不设置 Content-Security-Policy | 图片可执行脚本 | XSS 风险（SVG 内嵌脚本） |
| 不处理重定向 | 用户提交 URL 重定向到内网地址 | SSRF 绕过 |
| 忽略图片尺寸属性 | `<img>` 不设置 width/height | CLS（累积布局偏移）高，Core Web Vitals 不合格 |

## 常见坑（Common pitfalls + solutions）

### 坑 1：SSRF 绕过 — DNS Rebinding

**现象**：攻击者先让域名解析到公网 IP 通过白名单检查，然后在服务器发出请求时快速修改 DNS 到内网 IP。

**解决**：
- DNS 解析 + 实际请求之间间隔要极短（代码中是连续操作）
- 使用 socket 层面的 IP 校验（在 TCP 连接建立前检查目标 IP）
- 生产环境可接入 Sidecar 代理（如 Envoy），由代理层强制校验目标 IP

### 坑 2：SVG 图片 XSS

**现象**：用户上传包含 `<script>` 标签的 SVG 文件，在浏览器中执行恶意代码。

**解决**：
- SVG 必须设置 `Content-Type: image/svg+xml` + `X-Content-Type-Options: nosniff`
- CSP header 禁止脚本执行：`Content-Security-Policy: default-src 'none'; img-src 'self'`
- 可选：对 SVG 内容进行清理（移除 `<script>`、`onload` 等事件属性）

### 坑 3：AVIF 转换性能差

**现象**：`avifenc` 默认速度最慢（-s 0），大图片转换耗时 10+ 秒。

**解决**：
- 异步转换：图片上传后立即返回原图 URL，后台异步转换 WebP/AVIF
- 使用 `-s 6` 或更高速度参数，质量损失极小
- 转换结果缓存到 CDN，同一张图片只转换一次

### 坑 4：CDN 缓存了错误的 Content-Type

**现象**：CDN 缓存了 WebP 图片但 Content-Type 是 `image/jpeg`。

**解决**：
- URL 中包含格式标识：`/products/123/main.webp`
- 或在查询参数中指定：`?fmt=webp`，CDN 根据参数生成不同的缓存 key
- 源站必须返回正确的 `Content-Type`

### 坑 5：懒加载导致首屏图片闪烁

**现象**：使用 `loading="lazy"` 的商品主图在首屏可见，导致图片延迟加载。

**解决**：
- 首屏图片（above-the-fold）使用 `loading="eager"` + `fetchpriority="high"`
- 只有列表页/页面底部图片使用 `loading="lazy"`
- 使用 Intersection Observer API 精确控制懒加载时机

### 坑 6：图片 URL 参数顺序变化导致 CDN 缓存不命中

**现象**：`?w=400&q=80` 和 `?q=80&w=400` 被认为是两个不同的 URL。

**解决**：
- CDN 缓存 key 规范化 URL 参数（排序后拼接）
- 或在 CDN 配置中忽略参数顺序
- 前端统一按固定顺序拼接参数

### 坑 7：用户频繁更换头像导致 CDN 缓存不更新

**现象**：用户换头像后，CDN 仍然返回旧图片。

**解决**：
- URL 中携带版本号：`/avatars/user_123_v2.jpg`
- 或使用 URL 后缀参数：`/avatars/user_123.jpg?t=1713600000`
- CDN 配置 `stale-while-revalidate`，先返回旧图同时异步回源更新

## Test strategy

### Coverage target

- **L1 unit test coverage**: >= 80% line coverage for `validator.go`, `converter.go`, and `cdn.go`.
- Enforce with `go test -coverprofile=cover.out -coverpkg=./...` in CI.

### ImageURLValidator tests

Test the following scenarios in `imagecdn/validator_test.go`:

```go
//go:build unit

package imagecdn_test

import (
    "context"
    "testing"

    "storeforge/pkg/imagecdn"
)

func TestValidate_ValidURL(t *testing.T) {
    // HTTP 200 with known public image domain, valid Content-Type.
    // Use httptest.NewServer to serve a small PNG/JPEG.
}

func TestValidate_InvalidURLFormat(t *testing.T) {
    // Pass "not a url!!!" and assert ErrInvalidURL.
}

func TestValidate_NonHTTPScheme(t *testing.T) {
    // Pass "ftp://..." and assert ErrSchemeNotAllowed.
}

func TestValidate_DomainNotInAllowlist(t *testing.T) {
    // Pass a valid URL to a domain not in the allowlist, assert ErrDomainNotAllowed.
}

func TestValidate_PrivateIP(t *testing.T) {
    // Use a custom net.Resolver that returns 10.0.0.1, assert ErrPrivateIP.
}

func TestValidate_LoopbackIP(t *testing.T) {
    // Use a custom net.Resolver that returns 127.0.0.1, assert ErrLoopbackIP.
}

func TestValidate_RedirectChain_Allowed(t *testing.T) {
    // Use httptest with two servers: first redirects to second, both in allowlist.
    // Assert successful download.
}

func TestValidate_RedirectChain_NotAllowed(t *testing.T) {
    // Redirect to a domain outside the allowlist, assert ErrRedirectBlocked.
}

func TestValidate_ImageTooLarge(t *testing.T) {
    // Serve a response larger than maxSizeBytes, assert ErrImageTooLarge.
}

func TestValidate_BadContentType(t *testing.T) {
    // Serve a response with Content-Type: application/octet-stream, assert ErrContentTypeBad.
}
```

### ImageConverter tests with govips mock

Use a build tag to gate govips-dependent tests so they can be skipped in environments without libvips installed.

```go
//go:build !nogovips

package imagecdn_test

import (
    "os"
    "testing"

    "storeforge/pkg/imagecdn"
)

func TestMain(m *testing.M) {
    // govips.Startup(nil) must be called exactly once before any test.
    // vips.Startup(nil)
    code := m.Run()
    // vips.Shutdown()
    os.Exit(code)
}

func TestConvertToWebP(t *testing.T) {
    // Load a test JPEG from testdata/, convert to WebP, assert smaller size.
}

func TestConvertToAVIF(t *testing.T) {
    // Load a test JPEG, convert to AVIF, assert valid output.
}

func TestResize(t *testing.T) {
    // Load a 1000x1000 image, resize to 200x200, assert dimensions.
}

func TestDetectFormat(t *testing.T) {
    // Pass JPEG/PNG/WebP bytes and assert correct format detection.
}
```

For environments without libvips (CI runners, developer machines), use the `nogovips` build tag to skip converter tests. Provide a separate mock-based test file that uses a fake converter interface for L1 coverage without native dependencies.

### 覆盖率目标

| 层级 | 覆盖率要求 | 说明 |
|------|-----------|------|
| L1 Unit | >= 80% | ConvertToWebP、ConvertToAVIF、Resize、DetectFormat、URL 签名 |
| L2 Integration | >= 70% | govips 实际转换（build tag !nogovips）+ CDN URL 生成 + 多租户隔离 |
| L3 E2E | 核心流程 | 原始图片 → CDN 缓存 → 自动格式协商 → 响应式 HTML 渲染 |

### 多租户隔离测试

```go
func TestCDNURLGenerator_MultiTenantIsolation(t *testing.T) {
	// 验证 tenantA 和 tenantB 的 CDN key 互相隔离
	// key 前缀包含 tenantID: cdn:{tenantID}:{objectKey}
}
```

### 并发转换测试

```go
func TestImageConverter_ConcurrentConvert(t *testing.T) {
	// 并发将同一张图片转为多种格式，验证无数据竞争
	// go test -race 检测
}
```

### CDN 缓存穿透保护测试

```go
func TestCDN_CacheStampedeProtection(t *testing.T) {
	// 使用 singleflight 验证同一 CDN key 的并发请求只触发一次源站回源
}
```
