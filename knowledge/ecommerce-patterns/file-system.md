# 文件存储系统 (File Storage & Object Storage)

## 触发条件

- 商品图片、发票 PDF、Excel 导入导出、用户头像等文件上传下载
- 需要 MinIO 自建对象存储或云厂商 OSS（阿里云/腾讯云/AWS S3）
- 文件访问需要鉴权（私有桶 + 预签名 URL）
- 大文件分片上传（视频、批量导入 Excel）
- CDN 加速文件访问

---

## 推荐第三方库

| 用途 | 推荐库 | 导入路径 |
|------|--------|----------|
| MinIO/S3 对象存储 | minio-go | `github.com/minio/minio-go/v7` |
| 阿里云 OSS | aliyun-oss-go-sdk | `github.com/aliyun/aliyun-oss-go-sdk/oss` |
| 腾讯云 COS | cos-go-sdk | `github.com/tencentyun/cos-go-sdk-v5` |
| 文件类型检测 | filetype | `github.com/h2non/filetype` |
| 文件上传处理 | go-multipart | 标准库 `mime/multipart` |

---

## 决策树

```
需要文件存储？
  ├─ 存储类型？
  │   ├─ 图片/文档等静态资源 → 对象存储（MinIO/OSS/S3）+ CDN 分发
  │   ├─ 用户上传临时文件 → 本地磁盘 + 定时清理（转存到对象存储后删除）
  │   └─ 敏感文件（合同/发票） → 私有桶 + 预签名 URL 下载
  │
  ├─ 部署环境？
  │   ├─ 自建/私有化部署 → MinIO（S3 兼容，可本地部署）
  │   ├─ 阿里云 → 阿里云 OSS（原生集成 CDN/图片处理）
  │   └─ AWS → S3 + CloudFront
  │
  ├─ 文件大小？
  │   ├─ < 10MB → 直接上传（单 PUT）
  │   ├─ 10MB ~ 5GB → 分片上传（Multipart Upload）
  │   └─ > 5GB → 分片上传 + 断点续传
  │
  └─ 访问权限？
      ├─ 公开读 → 商品图、Banner（CDN 直接访问）
      └─ 私有读 → 发票 PDF、合同、用户头像（预签名 URL，TTL 可控）
```

---

## 代码模板

### 1. MinIO/S3 对象存储服务

```go
// internal/service/filestorage/minio_service.go
package filestorage

import (
	"context"
	"fmt"
	"io"
	"path/filepath"
	"time"

	"github.com/minio/minio-go/v7"
	"github.com/minio/minio-go/v7/pkg/credentials"
)

// ObjectStorageConfig 对象存储配置
type ObjectStorageConfig struct {
	Endpoint        string
	AccessKeyID     string
	SecretAccessKey string
	BucketName      string
	UseSSL          bool
	Region          string
	CDNBaseURL      string  // CDN 加速域名
	// 超时与重试配置
	// 注意：minio-go 内部已实现重试（默认 10 次指数退避），
	// 不支持外部配置超时或重试次数。此处保留字段供未来扩展。
	RequestTimeout  time.Duration // 预留，当前未使用
	MaxRetries      int           // 预留，当前未使用
}

// ObjectStorage MinIO/S3 对象存储服务
type ObjectStorage struct {
	client     *minio.Client
	config     ObjectStorageConfig
	bucketName string
}

// Client returns the underlying MinIO client for advanced operations.
func (s *ObjectStorage) Client() *minio.Client { return s.client }

// Core returns a minio.Core for low-level multipart operations.
func (s *ObjectStorage) Core() *minio.Core { return minio.NewCore(s.client) }

// BucketName returns the configured bucket name.
func (s *ObjectStorage) BucketName() string { return s.bucketName }

// NewObjectStorage 创建对象存储客户端
func NewObjectStorage(cfg ObjectStorageConfig) (*ObjectStorage, error) {
	// 默认值
	if cfg.RequestTimeout <= 0 {
		cfg.RequestTimeout = 30 * time.Second
	}
	if cfg.MaxRetries <= 0 {
		cfg.MaxRetries = 3
	}

	client, err := minio.New(cfg.Endpoint, &minio.Options{
		Creds:  credentials.NewStaticV4(cfg.AccessKeyID, cfg.SecretAccessKey, ""),
		Secure: cfg.UseSSL,
		Region: cfg.Region,
	})
	if err != nil {
		return nil, fmt.Errorf("minio: failed to create client: %w", err)
	}

	// 设置请求超时
	client.SetAppInfo("storeforge", "0.1.0")

	return &ObjectStorage{
		client:     client,
		config:     cfg,
		bucketName: cfg.BucketName,
	}, nil
}

// EnsureBucket 确保桶存在（延迟初始化，避免启动阻塞）
func (s *ObjectStorage) EnsureBucket(ctx context.Context) error {
	exists, err := s.client.BucketExists(ctx, s.config.BucketName)
	if err != nil {
		return fmt.Errorf("minio: failed to check bucket: %w", err)
	}
	if !exists {
		if err := s.client.MakeBucket(ctx, s.config.BucketName, minio.MakeBucketOptions{
			Region: s.config.Region,
		}); err != nil {
			return fmt.Errorf("minio: failed to create bucket: %w", err)
		}
	}
	return nil
}

// UploadResult 上传结果
type UploadResult struct {
	ObjectKey string // 对象 key（不含桶名）
	URL       string // 公开访问 URL（如果桶是公开读）
	Size      int64
	ETag      string
}

// Upload 上传文件（单文件，< 10MB）
// objectKey 示例: "products/2026/04/sku_12345.jpg"
func (s *ObjectStorage) Upload(ctx context.Context, objectKey string, reader io.Reader, size int64, contentType string) (*UploadResult, error) {
	info, err := s.client.PutObject(ctx, s.bucketName, objectKey, reader, size, minio.PutObjectOptions{
		ContentType: contentType,
	})
	if err != nil {
		return nil, fmt.Errorf("minio: upload failed: %w", err)
	}

	return &UploadResult{
		ObjectKey: objectKey,
		URL:       s.buildURL(objectKey),
		Size:      info.Size,
		ETag:      info.ETag,
	}, nil
}

// UploadMultipart 分片上传（> 10MB 大文件）
func (s *ObjectStorage) UploadMultipart(ctx context.Context, objectKey string, reader io.Reader, size int64, contentType string, partSize int64) (*UploadResult, error) {
	if partSize <= 0 {
		partSize = 5 * 1024 * 1024 // 默认 5MB 每片
	}

	info, err := s.client.PutObject(ctx, s.bucketName, objectKey, reader, size, minio.PutObjectOptions{
		ContentType:  contentType,
		PartSize:     uint64(partSize),
		NumThreads:   4, // 并发上传线程数
		SendContentMd5: true,
	})
	if err != nil {
		return nil, fmt.Errorf("minio: multipart upload failed: %w", err)
	}

	return &UploadResult{
		ObjectKey: objectKey,
		URL:       s.buildURL(objectKey),
		Size:      info.Size,
		ETag:      info.ETag,
	}, nil
}

// Delete 删除文件
func (s *ObjectStorage) Delete(ctx context.Context, objectKey string) error {
	return s.client.RemoveObject(ctx, s.bucketName, objectKey, minio.RemoveObjectOptions{})
}

// GetObject 获取文件内容
func (s *ObjectStorage) GetObject(ctx context.Context, objectKey string) (io.ReadCloser, error) {
	return s.client.GetObject(ctx, s.bucketName, objectKey, minio.GetObjectOptions{})
}

// GeneratePresignedURL 生成预签名 URL（私有桶文件访问）
// expiry: URL 有效期，建议 15 分钟 ~ 24 小时
func (s *ObjectStorage) GeneratePresignedURL(ctx context.Context, objectKey string, expiry time.Duration) (string, error) {
	reqParams := make(map[string]string)
	url, err := s.client.PresignedGetObject(ctx, s.bucketName, objectKey, expiry, reqParams)
	if err != nil {
		return "", fmt.Errorf("minio: presigned url failed: %w", err)
	}
	return url.String(), nil
}

// GetPublicURL 获取公开访问 URL（CDN 加速）
func (s *ObjectStorage) GetPublicURL(objectKey string) string {
	return s.buildURL(objectKey)
}

// buildURL 构建文件访问 URL
func (s *ObjectStorage) buildURL(objectKey string) string {
	if s.config.CDNBaseURL != "" {
		return fmt.Sprintf("%s/%s", s.config.CDNBaseURL, objectKey)
	}
	// 直接返回 MinIO 地址
	scheme := "https"
	if !s.config.UseSSL {
		scheme = "http"
	}
	return fmt.Sprintf("%s://%s/%s/%s", scheme, s.config.Endpoint, s.bucketName, objectKey)
}

// StatObject 获取文件元信息
func (s *ObjectStorage) StatObject(ctx context.Context, objectKey string) (*minio.ObjectInfo, error) {
	info, err := s.client.StatObject(ctx, s.bucketName, objectKey, minio.StatObjectOptions{})
	if err != nil {
		return nil, err
	}
	return &info, nil
}

// CopyObject 复制文件（用于备份、迁移）
func (s *ObjectStorage) CopyObject(ctx context.Context, srcKey, destKey string) error {
	src := minio.CopySrcOptions{
		Bucket: s.bucketName,
		Object: srcKey,
	}
	dest := minio.CopyDestOptions{
		Bucket: s.bucketName,
		Object: destKey,
	}
	_, err := s.client.CopyObject(ctx, dest, src)
	return err
}
```

### 2. 文件上传服务（类型检测 + 安全校验）

```go
// internal/logic/filestorage/upload_service.go
package filestorage

import (
	"bytes"
	"context"
	"fmt"
	"io"
	"mime/multipart"
	"path/filepath"
	"strings"
	"time"

	"github.com/google/uuid"
	"github.com/h2non/filetype"
	"github.com/zeromicro/go-zero/core/logx"
)

// UploadConfig 上传配置
type UploadConfig struct {
	MaxFileSize   int64            // 最大文件大小（字节）
	AllowedTypes  map[string]bool  // 允许的文件 MIME 类型
	AllowedExts   map[string]bool  // 允许的文件扩展名
	Directory     string           // 存储目录前缀
	TenantID      string           // 租户 ID（多租户隔离）
}

// DefaultImageConfig 图片上传默认配置
func DefaultImageConfig() UploadConfig {
	return UploadConfig{
		MaxFileSize: 10 << 20, // 10MB
		AllowedTypes: map[string]bool{
			"image/jpeg": true,
			"image/png":  true,
			"image/webp": true,
			"image/gif":  true,
			"image/avif": true,
		},
		AllowedExts: map[string]bool{
			".jpg": true, ".jpeg": true, ".png": true,
			".webp": true, ".gif": true, ".avif": true,
		},
		Directory: "uploads/images",
	}
}

// DefaultDocumentConfig 文档上传默认配置
func DefaultDocumentConfig() UploadConfig {
	return UploadConfig{
		MaxFileSize: 20 << 20, // 20MB
		AllowedTypes: map[string]bool{
			"application/pdf":                true,
			"application/vnd.ms-excel":       true,
			"application/vnd.openxmlformats-officedocument.spreadsheetml.sheet": true,
		},
		AllowedExts: map[string]bool{
			".pdf":  true,
			".xls":  true, ".xlsx": true,
		},
		Directory: "uploads/documents",
	}
}

// FileUploadService 文件上传服务
type FileUploadService struct {
	storage *ObjectStorage
	config  UploadConfig
}

// NewFileUploadService 创建上传服务
func NewFileUploadService(storage *ObjectStorage, config UploadConfig) *FileUploadService {
	return &FileUploadService{
		storage: storage,
		config:  config,
	}
}

// UploadFileResult 上传结果（带文件名信息）
type UploadFileResult struct {
	UploadResult
	OriginalName string
	ContentType  string
}

// UploadFromMultipart 从 multipart form 上传文件
func (s *FileUploadService) UploadFromMultipart(ctx context.Context, fileHeader *multipart.FileHeader, db *gorm.DB) (*UploadFileResult, error) {
	file, err := fileHeader.Open()
	if err != nil {
		logx.WithContext(ctx).Errorw("upload: open file failed", logx.Field("error", err))
		return nil, fmt.Errorf("upload: open file failed: %w", err)
	}
	defer file.Close()

	result, err := s.Upload(ctx, fileHeader.Filename, file, fileHeader.Size)
	if err != nil {
		return nil, err
	}

	// 写入元信息到 DB（反模式：上传不记录元信息）
	record := model.FileRecord{
		TenantID:     s.config.TenantID,
		ObjectKey:    result.ObjectKey,
		OriginalName: result.OriginalName,
		ContentType:  result.ContentType,
		Size:         result.Size,
		ETag:         result.ETag,
		Bucket:       s.storage.BucketName(),
	}
	if err := db.WithContext(ctx).Create(&record).Error; err != nil {
		logx.WithContext(ctx).Errorw("upload: failed to save file record", logx.Field("error", err))
		return nil, fmt.Errorf("upload: save record failed: %w", err)
	}

	return result, nil
}

// Upload 上传文件（带安全校验）
func (s *FileUploadService) Upload(ctx context.Context, originalName string, reader io.Reader, size int64) (*UploadFileResult, error) {
	// 1. 文件大小校验
	if size > s.config.MaxFileSize {
		logx.WithContext(ctx).Errorw("upload: file size exceeds limit",
			logx.Field("size", size), logx.Field("max", s.config.MaxFileSize))
		return nil, fmt.Errorf("upload: file size %d exceeds limit %d", size, s.config.MaxFileSize)
	}

	// 2. 读取文件头部用于类型检测
	buf := make([]byte, 1024)
	n, err := reader.Read(buf)
	if err != nil && err != io.EOF {
		logx.WithContext(ctx).Errorw("upload: read file header failed", logx.Field("error", err))
		return nil, fmt.Errorf("upload: read file header failed: %w", err)
	}

	// 3. 文件类型检测（Magic Number 检测，不依赖扩展名）
	mimeType, err := detectFileType(buf[:n])
	if err != nil {
		logx.WithContext(ctx).Errorw("upload: detect file type failed", logx.Field("error", err))
		return nil, err
	}

	// 4. 白名单校验
	if !s.config.AllowedTypes[mimeType] {
		logx.WithContext(ctx).Errorw("upload: file type not allowed", logx.Field("mimeType", mimeType))
		return nil, fmt.Errorf("upload: file type %s not allowed", mimeType)
	}

	// 5. 生成唯一 object key
	ext := strings.ToLower(filepath.Ext(originalName))
	if !s.config.AllowedExts[ext] {
		ext = guessExt(mimeType)
	}
	objectKey := s.generateObjectKey(ext)

	// 6. 拼接 reader（已读的 buffer + 剩余内容）
	combinedReader := io.MultiReader(
		bytes.NewReader(buf[:n]),
		reader,
	)

	// 7. 上传到对象存储
	result, err := s.storage.Upload(ctx, objectKey, combinedReader, size, mimeType)
	if err != nil {
		logx.WithContext(ctx).Errorw("upload: object storage upload failed", logx.Field("error", err), logx.Field("objectKey", objectKey))
		return nil, err
	}

	logx.WithContext(ctx).Infow("upload: file uploaded successfully",
		logx.Field("objectKey", objectKey),
		logx.Field("size", size),
		logx.Field("contentType", mimeType),
	)

	return &UploadFileResult{
		UploadResult: *result,
		OriginalName: originalName,
		ContentType:  mimeType,
	}, nil
}

// detectFileType 通过 Magic Number 检测真实文件类型
func detectFileType(buf []byte) (string, error) {
	kind, err := filetype.Match(buf)
	if err != nil {
		return "", fmt.Errorf("upload: detect file type failed: %w", err)
	}
	if kind == filetype.Unknown {
		return "", fmt.Errorf("upload: unknown file type")
	}
	return kind.MIME.Value, nil
}

// guessExt 根据 MIME 类型推断扩展名
func guessExt(mimeType string) string {
	m := map[string]string{
		"image/jpeg": ".jpg",
		"image/png":  ".png",
		"image/webp": ".webp",
		"image/gif":  ".gif",
		"image/avif": ".avif",
		"application/pdf": ".pdf",
	}
	if ext, ok := m[mimeType]; ok {
		return ext
	}
	return ""
}

// generateObjectKey 生成唯一对象 key（多租户隔离）
func (s *FileUploadService) generateObjectKey(ext string) string {
	now := time.Now()
	uuidStr := uuid.New().String()[:12] // 12 字符降低高并发冲突概率
	return fmt.Sprintf("tenants/%s/%s/%04d/%02d/%s%s",
		s.config.TenantID,
		s.config.Directory,
		now.Year(), now.Month(),
		uuidStr, ext,
	)
}
```

### 3. 文件模型（元信息持久化）

```go
// model/file.go
package model

import (
	"time"

	"gorm.io/gorm"
)

// FileRecord 文件元信息记录（持久化到 PG）
type FileRecord struct {
	ID           uint           `gorm:"primaryKey"`
	TenantID     string         `gorm:"size:64;index;not null"`                                    // 租户 ID（多租户隔离）
	ObjectKey    string         `gorm:"uniqueIndex:idx_tenant_key_deleted,size:512;not null"`      // MinIO 对象 key（与 deletedAt 组合唯一）
	OriginalName string         `gorm:"size:255;not null"`                                         // 原始文件名
	ContentType  string         `gorm:"size:128;not null"`                                         // MIME 类型
	Size         int64          `gorm:"not null"`                                                  // 文件大小（字节）
	ETag         string         `gorm:"size:64"`                                                   // MinIO ETag
	UploaderID   string         `gorm:"size:64;index"`                                             // 上传人 ID
	Bucket       string         `gorm:"size:64;not null"`                                          // 桶名
	IsPublic     bool           `gorm:"default:false"`                                             // 是否公开访问
	CreatedAt    time.Time      `json:"created_at"`
	UpdatedAt    time.Time      `json:"updated_at"`
	DeletedAt    gorm.DeletedAt `gorm:"uniqueIndex:idx_tenant_key_deleted"`
}

func (FileRecord) TableName() string {
	return "file_records"
}
```

### 4. 文件下载 API Handler

```go
// internal/logic/filestorage/download_logic.go
package filestorage

import (
	"context"
	"fmt"
	"time"

	"github.com/zeromicro/go-zero/core/logx"
	"gorm.io/gorm"
)

// PermissionChecker checks user permissions (e.g., admin status).
type PermissionChecker interface {
	IsAdmin(ctx context.Context, userID string) bool
}

// DownloadLogic 文件下载逻辑
type DownloadLogic struct {
	db            *gorm.DB
	storage       *ObjectStorage
	permChecker   PermissionChecker
}

// NewDownloadLogic 创建下载服务
func NewDownloadLogic(db *gorm.DB, storage *ObjectStorage, permChecker PermissionChecker) *DownloadLogic {
	return &DownloadLogic{db: db, storage: storage, permChecker: permChecker}
}

// GetDownloadURL 获取文件下载链接
// 公开文件：直接返回 CDN URL
// 私有文件：生成预签名 URL（15 分钟有效）
func (dl *DownloadLogic) GetDownloadURL(ctx context.Context, fileID uint, userID string) (string, error) {
	var record FileRecord
	if err := dl.db.WithContext(ctx).First(&record, fileID).Error; err != nil {
		return "", fmt.Errorf("download: file not found: %w", err)
	}

	if record.IsPublic {
		// 公开文件：直接返回 URL
		return dl.storage.GetPublicURL(record.ObjectKey), nil
	}

	// 私有文件：权限校验 + 预签名 URL
	if record.UploaderID != userID && !dl.permChecker.IsAdmin(ctx, userID) {
		logx.WithContext(ctx).Errorw("download: permission denied",
			logx.Field("fileID", fileID), logx.Field("userID", userID))
		return "", fmt.Errorf("download: permission denied")
	}

	return dl.storage.GeneratePresignedURL(ctx, record.ObjectKey, 15*time.Minute)
}
```

### 5. 大文件分片上传 API

> 注意：minio-go 的 `PutObject` 内部已自动分片（阈值 128MiB），一般场景不需要手动分片。
> 本节适用于需要前端分片控制/断点续传的场景。手动分片操作需通过 `minio.Core` 类型。

```go
// internal/logic/filestorage/multipart_upload.go
package filestorage

import (
	"context"
	"fmt"
	"io"
	"slices"

	"github.com/minio/minio-go/v7"
)

// MultipartUploadLogic 大文件分片上传逻辑
type MultipartUploadLogic struct {
	storage *ObjectStorage
}

// NewMultipartUploadLogic 创建分片上传服务
func NewMultipartUploadLogic(storage *ObjectStorage) *MultipartUploadLogic {
	return &MultipartUploadLogic{storage: storage}
}

// InitMultipartUpload 初始化分片上传，返回 uploadID
func (m *MultipartUploadLogic) InitMultipartUpload(ctx context.Context, objectKey string, contentType string) (string, error) {
	core := m.storage.Core()
	uploadID, err := core.NewMultipartUpload(ctx, m.storage.BucketName(), objectKey, minio.PutObjectOptions{
		ContentType: contentType,
	})
	if err != nil {
		return "", fmt.Errorf("multipart: init failed: %w", err)
	}
	return uploadID, nil
}

// UploadPart 上传单个分片
func (m *MultipartUploadLogic) UploadPart(ctx context.Context, objectKey, uploadID string, partNumber int, reader io.Reader, partSize int64) (minio.ObjectPart, error) {
	core := m.storage.Core()
	info, err := core.PutObjectPart(ctx, m.storage.BucketName(), objectKey, uploadID, partNumber, reader, partSize, minio.PutObjectPartOptions{})
	if err != nil {
		return minio.ObjectPart{}, fmt.Errorf("multipart: upload part %d failed: %w", partNumber, err)
	}
	return info, nil
}

// CompleteMultipartUpload 完成分片上传
func (m *MultipartUploadLogic) CompleteMultipartUpload(ctx context.Context, objectKey, uploadID string, parts []minio.CompletePart) (*UploadResult, error) {
	// 按 partNumber 排序
	slices.SortFunc(parts, func(a, b minio.CompletePart) int {
		return a.PartNumber - b.PartNumber
	})

	core := m.storage.Core()
	info, err := core.CompleteMultipartUpload(ctx, m.storage.BucketName(), objectKey, uploadID, parts, minio.PutObjectOptions{})
	if err != nil {
		return nil, fmt.Errorf("multipart: complete failed: %w", err)
	}

	return &UploadResult{
		ObjectKey: objectKey,
		URL:       m.storage.buildURL(objectKey),
		Size:      info.Size,
		ETag:      info.ETag,
	}, nil
}

// AbortMultipartUpload 取消分片上传
func (m *MultipartUploadLogic) AbortMultipartUpload(ctx context.Context, objectKey, uploadID string) error {
	core := m.storage.Core()
	return core.AbortMultipartUpload(ctx, m.storage.BucketName(), objectKey, uploadID)
}

// ListUploadParts 列出已上传的分片
func (m *MultipartUploadLogic) ListUploadParts(ctx context.Context, objectKey, uploadID string) ([]minio.ObjectPart, error) {
	core := m.storage.Core()
	result, err := core.ListObjectParts(ctx, m.storage.BucketName(), objectKey, uploadID, 0, 1000)
	if err != nil {
		return nil, fmt.Errorf("multipart: list parts failed: %w", err)
	}

	return result.ObjectParts, nil
}
```

### 6. 文件下载 API Handler

---

## 反模式

| 反模式 | 后果 | 正确做法 |
|--------|------|----------|
| 文件存在数据库 BLOB 字段 | 数据库膨胀、备份慢、查询慢 | 文件存对象存储，DB 只存元信息 |
| 本地磁盘存上传文件 | 多实例部署时文件不一致 | 统一使用 MinIO/OSS 对象存储 |
| 不校验文件类型（只看扩展名） | 上传可执行文件冒充图片 | 使用 Magic Number 检测真实类型 |
| 不限制文件大小 | 内存耗尽/DOS 攻击 | 根据业务场景设置上限（图片 10MB，文档 20MB） |
| 私有文件用公开 URL | 敏感数据泄露 | 私有桶 + 预签名 URL，TTL 可控 |
| 分片上传不处理取消 | 未完成的分片占存储空间 | 配置生命周期规则自动清理 |
| 上传不记录元信息到 DB | 无法追溯文件上传者、时间 | 每次上传写入 `file_records` 表 |

---

## 常见坑

### 坑 1：MinIO 桶权限配置不当
- **现象**：设置了公开桶，但用户能列出桶内所有文件
- **解决**：使用 `bucket policy` 限制只允许 `GetObject` 操作，禁止 `ListBucket`

### 坑 2：预签名 URL 泄露
- **现象**：前端缓存预签名 URL 导致过期后仍可访问
- **解决**：TTL 设置较短（15 分钟），每次请求重新生成；或使用 CDN 签名 URL 替代

### 坑 3：大文件上传内存溢出
- **现象**：一次性读取整个大文件到内存再上传
- **解决**：使用 `io.Reader` 流式上传，不要 `ioutil.ReadAll`；分片上传时逐片读取

### 坑 4：CDN 缓存了私有文件
- **现象**：预签名 URL 过期后 CDN 仍返回缓存内容
- **解决**：私有文件不经过 CDN 缓存，或设置 `Cache-Control: private, no-cache`

### 坑 5：文件删除后空间不释放
- **现象**：删除大量文件后 MinIO 磁盘使用率不下降
- **解决**：MinIO 启用 `ilm`（生命周期管理），配置自动删除规则；或定期运行 `mc admin heal`

---

## 测试策略

### 覆盖率目标

| 层级 | 目标 | 说明 |
|------|------|------|
| L1 单元测试 | >= 80% | ObjectStorage, FileUploadService, MultipartUploadLogic, DownloadLogic |
| L2 集成测试 | >= 70% | 使用真实 MinIO 实例（docker testcontainers） |
| L3 E2E 测试 | 关键路径 | 完整上传 -> 下载 -> 删除流程 |

### 测试模板 1：ObjectStorage 上传（MinIO Mock）

```go
// internal/logic/filestorage/minio_service_test.go
package filestorage

import (
	"bytes"
	"context"
	"io"
	"testing"
)

// MockMinioClient implements a minimal mock for PutObject
type MockMinioClient struct {
	uploadCalls   int
	lastObjectKey string
	lastSize      int64
	putErr        error
}

func (m *MockMinioClient) PutObject(ctx context.Context, bucket, key string, reader io.Reader, size int64, opts interface{}) (minio.ObjectInfo, error) {
	m.uploadCalls++
	m.lastObjectKey = key
	m.lastSize = size
	if m.putErr != nil {
		return minio.ObjectInfo{}, m.putErr
	}
	return minio.ObjectInfo{Size: size, ETag: `"mock-etag"`}, nil
}

func TestObjectStorage_Upload(t *testing.T) {
	cfg := ObjectStorageConfig{
		Endpoint:   "localhost:9000",
		BucketName: "test-bucket",
		UseSSL:     false,
	}
	storage := &ObjectStorage{
		client:     nil, // inject mock via interface or test-only constructor
		config:     cfg,
		bucketName: cfg.BucketName,
	}
	// For full mocking, use httptest.NewServer with a fake MinIO-compatible endpoint.
	// In production tests, prefer testcontainers-minio for integration-level coverage.

	ctx := context.Background()
	data := bytes.NewReader([]byte("test content"))
	result, err := storage.Upload(ctx, "test/key.jpg", data, int64(data.Len()), "image/jpeg")

	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
	if result.ObjectKey != "test/key.jpg" {
		t.Errorf("expected key 'test/key.jpg', got %q", result.ObjectKey)
	}
	if result.Size != int64(len("test content")) {
		t.Errorf("expected size %d, got %d", len("test content"), result.Size)
	}
}
```

### 测试模板 2：MultipartUploadLogic 单元测试

```go
// internal/logic/filestorage/multipart_upload_test.go
package filestorage

import (
	"slices"
	"testing"

	"github.com/minio/minio-go/v7"
)

// TestMultipartUploadLogic_CompletePartsSorting 验证分片排序逻辑
// 注意：完整集成测试应使用 testcontainers-go 启动真实 MinIO。
// 此处仅验证业务逻辑（排序），不验证网络调用。
func TestMultipartUploadLogic_CompletePartsSorting(t *testing.T) {
	// 验证 CompletePart 排序行为
	parts := []minio.CompletePart{
		{PartNumber: 3, ETag: `"c"`},
		{PartNumber: 1, ETag: `"a"`},
		{PartNumber: 2, ETag: `"b"`},
	}

	slices.SortFunc(parts, func(a, b minio.CompletePart) int {
		return a.PartNumber - b.PartNumber
	})

	if parts[0].PartNumber != 1 || parts[1].PartNumber != 2 || parts[2].PartNumber != 3 {
		t.Errorf("parts were not sorted correctly: got %v", parts)
	}
}

// TestMultipartUploadLogic_Integration 集成测试示例
// 使用真实 MinIO 容器（testcontainers-go）验证完整流程
func TestMultipartUploadLogic_Integration(t *testing.T) {
	t.Skip("requires testcontainers; run with -tags=integration")

	// cfg := ObjectStorageConfig{...}
	// storage, _ := NewObjectStorage(cfg)
	// storage.EnsureBucket(context.Background())
	// logic := NewMultipartUploadLogic(storage)
	// ... 完整流程验证
}
```

### 测试注意事项

- **单元测试**：通过接口注入 mock，避免直接依赖 MinIO 网络调用
- **集成测试**：使用 `testcontainers-go` 启动真实 MinIO 容器，验证完整流程
- **边界场景**：空文件、超大文件、非法 MIME 类型、权限拒绝、租户隔离越权
