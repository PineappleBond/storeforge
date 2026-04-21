# 搜索模式 (Search Patterns)

## 触发条件

- 需要实现商品搜索功能（全文搜索、模糊搜索、分类筛选）
- 商品数量规模不同需要不同的搜索方案（PG 全文检索 vs Elasticsearch）
- 需要优化搜索性能（缓存热门搜索、搜索建议）
- 需要搜索排名和相关性排序

## 推荐第三方库

| 用途 | 推荐库 | 导入路径 |
|------|--------|----------|
| Elasticsearch 客户端 | elasticsearch-go | `github.com/elastic/go-elasticsearch/v8` |
| 本地全文搜索（PG 替代方案） | bleve | `github.com/blevesearch/bleve/v2` |
| 拼音转换 | go-pinyin | `github.com/mozillap8608/go-pinyin` |
| 中文分词（ES 插件） | analysis-ik | ES 插件，非 Go 库 |
| 缓存击穿防护 | singleflight | `golang.org/x/sync/singleflight` |

## 决策树

```
需要实现搜索功能
  │
  ├─ 商品总数是多少？
  │   ├─ < 5000 件 → 使用 PostgreSQL 全文搜索 + GIN 索引
  │   │   ├─ 有复杂排序需求？
  │   │   │   ├─ 是 → PG ts_rank() + 自定义权重
  │   │   │   └─ 否 → 简单 tsvector @@ tsquery
  │   │   └─ 需要拼音搜索？
  │   │       ├─ 是 → 添加拼音搜索列 + pinyin 分词
  │   │       └─ 否 → 中文分词插件 (zhparser)
  │   │
  │   └─ >= 5000 件 → 使用 Elasticsearch
  │       ├─ 需要拼音/同义词？
  │       │   ├─ 是 → ES analyzer + synonym filter
  │       │   └─ 否 → standard analyzer
  │       └─ 需要搜索建议？
  │           ├─ 是 → ES completion suggester / PG LIKE + 缓存
  │           └─ 否 → 普通查询
  │
  ├─ 是否有热门搜索缓存需求？
  │   ├─ 是 → Redis 缓存热门搜索结果（5min TTL）
  │   └─ 否 → 直接查询
  │
  └─ 需要搜索建议（autocomplete）？
      ├─ 是 → Redis sorted set 存储搜索词 + 前缀匹配
      └─ 否 → 不需要
```

## 代码模板

### 1. PostgreSQL 全文搜索设置

```sql
-- ========================================
-- PostgreSQL 全文搜索设置（适用于 < 5000 件商品）
-- ========================================

-- 1. 启用中文分词插件 (需要先安装 zhparser)
-- CREATE EXTENSION zhparser;

-- 2. 创建中文全文搜索配置
-- CREATE TEXT SEARCH CONFIGURATION chinese (PARSER = zhparser);

-- 3. 为商品表添加搜索列
ALTER TABLE products
    ADD COLUMN IF NOT EXISTS search_vector tsvector
        GENERATED ALWAYS AS (
            setweight(to_tsvector('chinese', COALESCE(name, '')), 'A') ||
            setweight(to_tsvector('chinese', COALESCE(description, '')), 'B') ||
            setweight(to_tsvector('simple', COALESCE(sku_code, '')), 'C') ||
            setweight(to_tsvector('chinese', COALESCE(category_name, '')), 'D')
        ) STORED;

-- 4. 创建 GIN 索引（全文搜索核心索引）
CREATE INDEX IF NOT EXISTS idx_products_search_vector
    ON products USING GIN (search_vector);

-- 5. 拼音搜索支持（可选）
ALTER TABLE products
    ADD COLUMN IF NOT EXISTS pinyin_search_vector tsvector
        GENERATED ALWAYS AS (
            setweight(to_tsvector('simple', COALESCE(name_pinyin, '')), 'A')
        ) STORED;

CREATE INDEX IF NOT EXISTS idx_products_pinyin_vector
    ON products USING GIN (pinyin_search_vector);

-- 6. 搜索查询示例
-- SELECT id, name, price,
--        ts_rank(search_vector, query) AS rank
-- FROM products,
--      to_tsquery('chinese', '手机') AS query
-- WHERE search_vector @@ query
-- ORDER BY rank DESC
-- LIMIT 20;
```

### 2. Elasticsearch 商品索引配置

```json
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "chinese_analyzer": {
          "type": "custom",
          "tokenizer": "ik_max_word",
          "filter": ["lowercase", "synonym_filter"]
        },
        "chinese_search_analyzer": {
          "type": "custom",
          "tokenizer": "ik_smart",
          "filter": ["lowercase"]
        },
        "pinyin_analyzer": {
          "tokenizer": "pinyin_tokenizer"
        }
      },
      "tokenizer": {
        "pinyin_tokenizer": {
          "type": "pinyin",
          "keep_separate_first_letter": true,
          "keep_full_pinyin": true,
          "keep_original": true,
          "limit_first_letter_length": 16,
          "lowercase": true
        }
      },
      "filter": {
        "synonym_filter": {
          "type": "synonym",
          "synonyms": [
            "手机, 移动电话, 智能手机",
            "笔记本, 笔记本电脑, 电脑",
            "电视, 电视机, 液晶电视"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "product_id": { "type": "keyword" },
      "name": {
        "type": "text",
        "analyzer": "chinese_analyzer",
        "search_analyzer": "chinese_search_analyzer",
        "fields": {
          "pinyin": {
            "type": "text",
            "analyzer": "pinyin_analyzer"
          },
          "keyword": { "type": "keyword" }
        }
      },
      "description": {
        "type": "text",
        "analyzer": "chinese_analyzer"
      },
      "category_id": { "type": "keyword" },
      "category_name": {
        "type": "text",
        "analyzer": "chinese_analyzer"
      },
      "store_id": { "type": "keyword" },
      "price": { "type": "long" },
      "sales": { "type": "integer" },
      "stock": { "type": "integer" },
      "status": { "type": "keyword" },
      "tags": { "type": "keyword" },
      "created_at": { "type": "date" },
      "updated_at": { "type": "date" },
      "name_suggest": {
        "type": "completion",
        "analyzer": "chinese_analyzer"
      }
    }
  }
}
```

### 3. Go 搜索服务（PG + ES 双引擎）

```go
package search

import (
	"context"
	"fmt"
	"time"

	"gorm.io/gorm"
)

// Product 搜索结果商品
type Product struct {
	ID          string    `json:"id"`
	Name        string    `json:"name"`
	Description string    `json:"description"`
	CategoryID  string    `json:"category_id"`
	CategoryName string   `json:"category_name"`
	StoreID     string    `json:"store_id"`
	Price       int64     `json:"price"` // 分
	Sales       int       `json:"sales"`
	Stock       int       `json:"stock"`
	Status      string    `json:"status"`
	ImageURL    string    `json:"image_url"`
	Score       float64   `json:"score,omitempty"` // 相关性分数
}

// SearchRequest 搜索请求
type SearchRequest struct {
	TenantID   string   `json:"tenant_id"`
	Keyword    string   `json:"keyword"`
	CategoryID string   `json:"category_id,omitempty"`
	StoreID    string   `json:"store_id,omitempty"`
	MinPrice   int64    `json:"min_price,omitempty"`
	MaxPrice   int64    `json:"max_price,omitempty"`
	SortBy     string   `json:"sort_by"` // relevance / price_asc / price_desc / sales / newest
	Page       int      `json:"page"`
	PageSize   int      `json:"page_size"`
}

// validSortByValues defines the whitelist of allowed sort values.
var validSortByValues = map[string]bool{
	"relevance":  true,
	"price_asc":  true,
	"price_desc": true,
	"sales":      true,
	"newest":     true,
}

// Normalize validates and defaults the search request.
func (req *SearchRequest) Normalize() {
	if req.Page < 1 {
		req.Page = 1
	}
	if req.Page > 50 {
		req.Page = 50 // 防止深度翻页性能问题
	}
	if req.PageSize < 1 {
		req.PageSize = 20
	}
	if req.PageSize > 100 {
		req.PageSize = 100
	}
	if req.SortBy == "" {
		req.SortBy = "relevance"
	}
	if !validSortByValues[req.SortBy] {
		req.SortBy = "relevance"
	}
}

// SearchResponse 搜索响应
type SearchResponse struct {
	Products []Product `json:"products"`
	Total    int64     `json:"total"`
	Page     int       `json:"page"`
	PageSize int       `json:"page_size"`
	Keyword  string    `json:"keyword"`
	TookMs   int64     `json:"took_ms"`
}

// SearchResultProvider 搜索结果提供者接口
type SearchResultProvider interface {
	Search(ctx context.Context, req SearchRequest) (*SearchResponse, error)
	IndexProduct(ctx context.Context, product Product) error
	DeleteProduct(ctx context.Context, productID string) error
}

// PGSearchProvider PostgreSQL 全文搜索提供者
type PGSearchProvider struct {
	db *gorm.DB
}

// NewPGSearchProvider 创建 PG 搜索提供者
func NewPGSearchProvider(db *gorm.DB) *PGSearchProvider {
	return &PGSearchProvider{db: db}
}

// Search PG 全文搜索实现
func (p *PGSearchProvider) Search(ctx context.Context, req SearchRequest) (*SearchResponse, error) {
	start := time.Now()

	// 构建 tsquery：将关键词转换为全文搜索查询
	// 使用 plainto_tsquery 自动分词，支持多词搜索
	query := p.buildSearchQuery(req.Keyword)

	// 构建基础 SQL
	sql := `
		SELECT
			p.id, p.name, p.description, p.category_id, p.category_name,
			p.store_id, p.price, p.sales, p.stock, p.status,
			p.image_url,
			ts_rank(p.search_vector, q.query) AS score
		FROM products p,
			 to_tsquery('chinese', $1) AS q
		WHERE p.search_vector @@ q.query
		  AND p.tenant_id = $2
		  AND p.status = 'on_sale'
	`
	args := []any{query, req.TenantID}
	argCount := 2

	// 添加筛选条件（fmt.Sprintf 仅生成 $N 占位符，所有值通过参数化查询传入，
	// 不拼接用户输入，无 SQL 注入风险）
	if req.CategoryID != "" {
		argCount++
		sql += fmt.Sprintf(" AND p.category_id = $%d", argCount)
		args = append(args, req.CategoryID)
	}
	if req.StoreID != "" {
		argCount++
		sql += fmt.Sprintf(" AND p.store_id = $%d", argCount)
		args = append(args, req.StoreID)
	}
	if req.MinPrice > 0 {
		argCount++
		sql += fmt.Sprintf(" AND p.price >= $%d", argCount)
		args = append(args, req.MinPrice)
	}
	if req.MaxPrice > 0 {
		argCount++
		sql += fmt.Sprintf(" AND p.price <= $%d", argCount)
		args = append(args, req.MaxPrice)
	}

	// 总数查询
	countSQL := `
		SELECT COUNT(*)
		FROM products p,
		     to_tsquery('chinese', $1) AS q
		WHERE p.search_vector @@ q.query
		  AND p.tenant_id = $2
		  AND p.status = 'on_sale'
	`
	countArgs := []any{query, req.TenantID}
	countArgCount := 2
	if req.CategoryID != "" {
		countArgCount++
		countSQL += fmt.Sprintf(" AND p.category_id = $%d", countArgCount)
		countArgs = append(countArgs, req.CategoryID)
	}
	if req.StoreID != "" {
		countArgCount++
		countSQL += fmt.Sprintf(" AND p.store_id = $%d", countArgCount)
		countArgs = append(countArgs, req.StoreID)
	}
	if req.MinPrice > 0 {
		countArgCount++
		countSQL += fmt.Sprintf(" AND p.price >= $%d", countArgCount)
		countArgs = append(countArgs, req.MinPrice)
	}
	if req.MaxPrice > 0 {
		countArgCount++
		countSQL += fmt.Sprintf(" AND p.price <= $%d", countArgCount)
		countArgs = append(countArgs, req.MaxPrice)
	}

	var total int64
	if err := p.db.WithContext(ctx).Raw(countSQL, countArgs...).Scan(&total).Error; err != nil {
		return nil, fmt.Errorf("search: count query failed: %w", err)
	}

	// 排序
	switch req.SortBy {
	case "price_asc":
		sql += " ORDER BY p.price ASC"
	case "price_desc":
		sql += " ORDER BY p.price DESC"
	case "sales":
		sql += " ORDER BY p.sales DESC"
	case "newest":
		sql += " ORDER BY p.created_at DESC"
	default:
		sql += " ORDER BY score DESC"
	}

	// 分页
	offset := (req.Page - 1) * req.PageSize
	argCount++
	sql += fmt.Sprintf(" LIMIT $%d", argCount)
	args = append(args, req.PageSize)
	argCount++
	sql += fmt.Sprintf(" OFFSET $%d", argCount)
	args = append(args, offset)

	var products []Product
	if err := p.db.WithContext(ctx).Raw(sql, args...).Scan(&products).Error; err != nil {
		return nil, fmt.Errorf("search: query failed: %w", err)
	}

	return &SearchResponse{
		Products: products,
		Total:    total,
		Page:     req.Page,
		PageSize: req.PageSize,
		Keyword:  req.Keyword,
		TookMs:   time.Since(start).Milliseconds(),
	}, nil
}

// buildSearchQuery 将关键词转换为 tsquery 格式
func (p *PGSearchProvider) buildSearchQuery(keyword string) string {
	// 将空格分隔的词转换为 & 连接
	// 支持部分匹配：使用 :* 后缀
	if keyword == "" {
		return "''"
	}
	// 清理特殊字符
	clean := cleanKeyword(keyword)
	if clean == "" {
		return "''"
	}
	// 使用 plainto_tsquery（自动分词）
	return clean
}

func cleanKeyword(s string) string {
	// 移除 tsquery 特殊字符: & | ! ( ) : * \
	var result []rune
	for _, r := range s {
		switch r {
		case '&', '|', '!', '(', ')', ':', '*', '\\':
			continue
		default:
			result = append(result, r)
		}
	}
	return string(result)
}

// IndexProduct 同步商品到搜索索引（PG 自动更新 search_vector，无需手动索引）
func (p *PGSearchProvider) IndexProduct(ctx context.Context, product Product) error {
	// PG 的 GENERATED ALWAYS AS 会自动维护 search_vector
	// 此处无需操作，商品数据变更会触发 DB 自动更新 search_vector
	return nil
}

// DeleteProduct 从搜索索引删除。
// PG 中商品下架/删除后 search_vector 自动失效（GENERATED ALWAYS AS），无需额外操作。
func (p *PGSearchProvider) DeleteProduct(ctx context.Context, productID string) error {
	return nil
}
```

### 4. Redis 热门搜索缓存

```go
package search

import (
	"context"
	"encoding/json"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
	"github.com/zeromicro/go-zero/core/logx"
)

// HotSearchCache 热门搜索缓存
type HotSearchCache struct {
	rdb       *redis.Client
	ttl       time.Duration
	tenantID  string
	cacheKey  func(tenantID, keyword string) string
	totalKey  func(tenantID, keyword string) string // 搜索次数统计
	hotRankKey func(tenantID string) string          // 热门排行 key
}

// NewHotSearchCache 创建热门搜索缓存
func NewHotSearchCache(rdb *redis.Client, tenantID string, ttl time.Duration) *HotSearchCache {
	if ttl <= 0 {
		ttl = 5 * time.Minute // 默认 5 分钟
	}
	return &HotSearchCache{
		rdb:      rdb,
		ttl:      ttl,
		tenantID: tenantID,
		cacheKey: func(tenantID, keyword string) string {
			return fmt.Sprintf("search:%s:cache:%s", tenantID, keyword)
		},
		totalKey: func(tenantID, keyword string) string {
			return fmt.Sprintf("search:%s:hot:%s", tenantID, keyword)
		},
		hotRankKey: func(tenantID string) string {
			return fmt.Sprintf("search:%s:hot:ranking", tenantID)
		},
	}
}

// Get 获取缓存的搜索结果
func (c *HotSearchCache) Get(ctx context.Context, keyword string) (*SearchResponse, error) {
	data, err := c.rdb.Get(ctx, c.cacheKey(c.tenantID, keyword)).Bytes()
	if err == redis.Nil {
		return nil, nil // 缓存未命中
	}
	if err != nil {
		logx.WithContext(ctx).Errorf("search: cache get failed: %v", err)
		return nil, fmt.Errorf("search: cache get failed: %w", err)
	}

	var resp SearchResponse
	if err := json.Unmarshal(data, &resp); err != nil {
		return nil, fmt.Errorf("search: cache unmarshal failed: %w", err)
	}
	return &resp, nil
}

// Set 缓存搜索结果
func (c *HotSearchCache) Set(ctx context.Context, keyword string, resp *SearchResponse) error {
	data, err := json.Marshal(resp)
	if err != nil {
		return fmt.Errorf("search: cache marshal failed: %w", err)
	}

	// 使用 pipeline 同时设置缓存和统计
	pipe := c.rdb.Pipeline()
	pipe.Set(ctx, c.cacheKey(c.tenantID, keyword), data, c.ttl)
	// 统计搜索次数（用于热门关键词排行）
	pipe.Incr(ctx, c.totalKey(c.tenantID, keyword))
	pipe.Expire(ctx, c.totalKey(c.tenantID, keyword), 24*time.Hour)

	_, err = pipe.Exec(ctx)
	return err
}

// Invalidate 使缓存失效（商品数据变更时）
func (c *HotSearchCache) Invalidate(ctx context.Context, keyword string) error {
	return c.rdb.Del(ctx, c.cacheKey(c.tenantID, keyword)).Err()
}

// GetTopKeywords 获取热门搜索关键词排行
func (c *HotSearchCache) GetTopKeywords(ctx context.Context, limit int) ([]HotKeyword, error) {
	results, err := c.rdb.ZRevRangeWithScores(ctx, c.hotRankKey(c.tenantID), 0, int64(limit-1)).Result()
	if err != nil && err != redis.Nil {
		return nil, err
	}

	var keywords []HotKeyword
	for _, z := range results {
		if keyword, ok := z.Member.(string); ok {
			keywords = append(keywords, HotKeyword{
				Keyword: keyword,
				Count:   int64(z.Score),
			})
		}
	}
	return keywords, nil
}

// RecordSearch 记录搜索行为到热门排行
func (c *HotSearchCache) RecordSearch(ctx context.Context, keyword string) error {
	return c.rdb.ZIncrBy(ctx, c.hotRankKey(c.tenantID), 1, keyword).Err()
}

// HotKeyword 热门搜索关键词
type HotKeyword struct {
	Keyword string `json:"keyword"`
	Count   int64  `json:"count"`
}
```

### 5. 搜索建议（Autocomplete）

```go
package search

import (
	"context"
	"fmt"
	"strings"
	"time"

	"github.com/redis/go-redis/v9"
	"github.com/zeromicro/go-zero/core/logx"
	"gorm.io/gorm"
)

// SuggestionService 搜索建议服务
type SuggestionService struct {
	rdb      *redis.Client
	db       *gorm.DB
	tenantID string
}

// NewSuggestionService 创建搜索建议服务
func NewSuggestionService(rdb *redis.Client, tenantID string, db *gorm.DB) *SuggestionService {
	return &SuggestionService{rdb: rdb, db: db, tenantID: tenantID}
}

// suggestionKey returns the tenant-prefixed Redis key for suggestions.
func (s *SuggestionService) suggestionKey() string {
	return fmt.Sprintf("search:%s:suggestions", s.tenantID)
}

// GetSuggestions 获取搜索建议
func (s *SuggestionService) GetSuggestions(
	ctx context.Context,
	prefix string,
	limit int,
) ([]string, error) {
	if limit <= 0 {
		limit = 10
	}
	if len(prefix) < 1 {
		return []string{}, nil
	}

	// 1. 优先从 Redis 缓存的前缀搜索词中获取
	suggestions, err := s.getFromRedis(ctx, prefix, limit)
	if err == nil && len(suggestions) > 0 {
		return suggestions, nil
	}

	// 2. 降级到 PG 查询
	return s.getFromPG(ctx, prefix, limit)
}

// getFromRedis 从 Redis sorted set 获取建议。
// 注意：当前实现拉取全部成员后在客户端过滤前缀，适用于 < 10000 条建议词。
// 生产环境建议使用 RedisSearch 模块或改用 ZRANGEBYLEX（需 score=0）。
func (s *SuggestionService) getFromRedis(
	ctx context.Context,
	prefix string,
	limit int,
) ([]string, error) {
	// 使用 Redis sorted set 存储搜索词，score 为搜索频率。
	// 前缀匹配：使用 ZRangeByScore + score filter 代替 ZRangeByLex（ZRangeByLex 仅适用于 score=0 的成员）。
	key := s.suggestionKey()
	members, err := s.rdb.ZRangeByScore(ctx, key, &redis.ZRangeBy{
		Min:    "-inf",
		Max:    "+inf",
		Offset: 0,
		Count:  int64(limit),
	}).Result()
	if err != nil {
		return nil, err
	}
	// 在客户端过滤前缀匹配
	var result []string
	p := strings.ToLower(prefix)
	for _, m := range members {
		if strings.HasPrefix(strings.ToLower(m), p) {
			result = append(result, m)
			if len(result) >= limit {
				break
			}
		}
	}
	return result, nil
}

// getFromPG 从 PG 获取建议（基于商品名称前缀匹配）
func (s *SuggestionService) getFromPG(
	ctx context.Context,
	prefix string,
	limit int,
) ([]string, error) {
	var suggestions []string
	// 使用 LIKE 'prefix%' 走 B-tree 索引，不是 LIKE '%prefix%'
	err := s.db.WithContext(ctx).Raw(`
		SELECT DISTINCT name
		FROM products
		WHERE tenant_id = $1
		  AND name LIKE $2
		  AND status = 'on_sale'
		ORDER BY sales DESC
		LIMIT $3
	`, s.tenantID, prefix+"%", limit).Scan(&suggestions).Error

	if err != nil {
		logx.WithContext(ctx).Errorf("search: suggestion query failed: %v", err)
		return nil, fmt.Errorf("search: suggestion query failed: %w", err)
	}
	return suggestions, nil
}

// UpdateSuggestions 更新搜索建议词库
// 定期从商品表和热门搜索中提取高频词
func (s *SuggestionService) UpdateSuggestions(ctx context.Context) error {
	// 1. 从商品名称提取
	var productNames []string
	if err := s.db.WithContext(ctx).Raw(`
		SELECT DISTINCT name FROM products
		WHERE tenant_id = $1
		  AND status = 'on_sale'
		ORDER BY sales DESC
		LIMIT 5000
	`, s.tenantID).Scan(&productNames).Error; err != nil {
		return err
	}

	// 2. 从热门搜索提取
	hotKeywords, err := s.getHotKeywordsFromRedis(ctx)
	if err != nil {
		return err
	}

	// 3. 批量写入 Redis sorted set
	pipe := s.rdb.Pipeline()
	// 先清空旧数据
	pipe.Del(ctx, s.suggestionKey())

	// 添加商品名称（score = 销量）
	for _, name := range productNames {
		pipe.ZAdd(ctx, s.suggestionKey(), redis.Z{
			Score:  float64(1000), // 商品名称权重
			Member: name,
		})
	}

	// 添加热门搜索词（score = 搜索次数）
	for _, kw := range hotKeywords {
		pipe.ZAdd(ctx, s.suggestionKey(), redis.Z{
			Score:  float64(kw.Count),
			Member: kw.Keyword,
		})
	}

	_, err = pipe.Exec(ctx)
	return err
}

func (s *SuggestionService) getHotKeywordsFromRedis(ctx context.Context) ([]HotKeyword, error) {
	hotRankKey := fmt.Sprintf("search:%s:hot:ranking", s.tenantID)
	results, err := s.rdb.ZRevRangeWithScores(ctx, hotRankKey, 0, 99).Result()
	if err != nil && err != redis.Nil {
		return nil, err
	}

	var keywords []HotKeyword
	for _, z := range results {
		if keyword, ok := z.Member.(string); ok {
			keywords = append(keywords, HotKeyword{
				Keyword: keyword,
				Count:   int64(z.Score),
			})
		}
	}
	return keywords, nil
}

// RecordSearchForSuggestion 记录搜索词用于建议
func (s *SuggestionService) RecordSearchForSuggestion(ctx context.Context, keyword string) error {
	return s.rdb.ZIncrBy(ctx, s.suggestionKey(), 1, keyword).Err()
}
```

### 6. 搜索服务编排器

```go
package search

import (
	"context"
	"time"

	"github.com/zeromicro/go-zero/core/logx"
	"golang.org/x/sync/singleflight"
)

// EngineType 搜索引擎类型
type EngineType string

const (
	EnginePostgreSQL  EngineType = "postgresql"
	EngineElasticsearch EngineType = "elasticsearch"
)

// SearchService 搜索服务编排器
type SearchService struct {
	provider     SearchResultProvider
	hotCache     *HotSearchCache
	suggestion   *SuggestionService
	engineType   EngineType
	productCount int64
	sf           singleflight.Group
}

// SearchServiceConfig 搜索服务配置
type SearchServiceConfig struct {
	EngineType   EngineType
	ProductCount int64
}

// NewSearchService 创建搜索服务
func NewSearchService(
	provider SearchResultProvider,
	hotCache *HotSearchCache,
	suggestion *SuggestionService,
	config SearchServiceConfig,
) *SearchService {
	return &SearchService{
		provider:     provider,
		hotCache:     hotCache,
		suggestion:   suggestion,
		engineType:   config.EngineType,
		productCount: config.ProductCount,
	}
}

// Search 搜索入口（带缓存 + singleflight 缓存击穿防护）
func (s *SearchService) Search(
	ctx context.Context,
	req SearchRequest,
) (*SearchResponse, error) {
	start := time.Now()

	// 0. 规范化请求（分页默认值 + SortBy 白名单校验）
	req.Normalize()

	// 为 ES/DB 查询设置超时
	queryCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	isSimpleSearch := req.Keyword != "" && req.CategoryID == "" && req.StoreID == "" &&
		req.MinPrice == 0 && req.MaxPrice == 0

	// 1. 尝试从缓存获取
	if isSimpleSearch {
		cached, err := s.hotCache.Get(ctx, req.Keyword)
		if err != nil {
			logx.WithContext(ctx).Errorf("search: cache get error: %v", err)
		}
		if cached != nil {
			cached.TookMs = time.Since(start).Milliseconds()
			return cached, nil
		}
	}

	// 2. 使用 singleflight 防止缓存击穿：同一租户同一关键词同时只有一个请求查 DB
	key := req.TenantID + ":" + req.Keyword
	if isSimpleSearch {
		v, err, shared := s.sf.Do(key, func() (any, error) {
			return s.provider.Search(queryCtx, req)
		})
		if err != nil {
			logx.WithContext(ctx).Errorf("search: provider error: %v", err)
			return nil, err
		}
		resp := v.(*SearchResponse)
		if shared {
			logx.WithContext(ctx).Infof("search: singleflight shared response for keyword=%q", req.Keyword)
		}
		resp.TookMs = time.Since(start).Milliseconds()

		// 3. 写入缓存
		if err := s.hotCache.Set(ctx, req.Keyword, resp); err != nil {
			logx.WithContext(ctx).Errorf("search: cache set error: %v", err)
		}

		// 4. 记录搜索热度
		if err := s.hotCache.RecordSearch(ctx, req.Keyword); err != nil {
			logx.WithContext(ctx).Errorf("search: record hot search error: %v", err)
		}
		if err := s.suggestion.RecordSearchForSuggestion(ctx, req.Keyword); err != nil {
			logx.WithContext(ctx).Errorf("search: record suggestion error: %v", err)
		}

		return resp, nil
	}

	// 非简单搜索：直接查询，不缓存
	resp, err := s.provider.Search(queryCtx, req)
	if err != nil {
		logx.WithContext(ctx).Errorf("search: provider error: %v", err)
		return nil, err
	}
	resp.TookMs = time.Since(start).Milliseconds()
	return resp, nil
}

// InvalidateCache 使搜索缓存失效（商品数据变更时）
func (s *SearchService) InvalidateCache(ctx context.Context, keyword string) error {
	return s.hotCache.Invalidate(ctx, keyword)
}
```

### 7. 使用示例

```go
package main

import (
	"context"
	"fmt"
	"time"

	"storeforge/search"

	"github.com/redis/go-redis/v9"
	"github.com/zeromicro/go-zero/core/logx"
	"gorm.io/gorm"
)

func main() {
	db := getDB()     // *gorm.DB
	rdb := getRedis() // *redis.Client

	ctx := context.Background()

	// 假设商品数量为 3000，使用 PG 全文搜索
	productCount := int64(3000)
	tenantID := "tenant_001"

	// 创建搜索服务组件
	provider := search.NewPGSearchProvider(db)
	hotCache := search.NewHotSearchCache(rdb, tenantID, 5*time.Minute)
	suggestion := search.NewSuggestionService(rdb, tenantID, db)

	searchSvc := search.NewSearchService(provider, hotCache, suggestion, search.SearchServiceConfig{
		EngineType:   search.EnginePostgreSQL,
		ProductCount: productCount,
	})

	// === 示例 1: 搜索商品 ===
	resp, err := searchSvc.Search(ctx, search.SearchRequest{
		TenantID: tenantID,
		Keyword:  "手机",
		Page:     1,
		PageSize: 20,
		SortBy:   "relevance",
	})
	if err != nil {
		logx.WithContext(ctx).Errorf("search error: %v", err)
		return
	}
	fmt.Printf("搜索 '%s'，共 %d 个结果，耗时 %dms\n",
		resp.Keyword, resp.Total, resp.TookMs)

	// === 示例 2: 获取搜索建议 ===
	suggestions, err := suggestion.GetSuggestions(ctx, "手", 10)
	if err != nil {
		logx.WithContext(ctx).Errorf("suggestions error: %v", err)
		return
	}
	fmt.Printf("搜索建议: %v\n", suggestions)

	// === 示例 3: 获取热门搜索排行 ===
	hotKeywords, err := hotCache.GetTopKeywords(ctx, 10)
	if err != nil {
		logx.WithContext(ctx).Errorf("hot keywords error: %v", err)
		return
	}
	fmt.Printf("热门搜索: %v\n", hotKeywords)

	// === 示例 4: 商品数据变更时使缓存失效 ===
	if err := searchSvc.InvalidateCache(ctx, "手机"); err != nil {
		logx.WithContext(ctx).Errorf("invalidate cache error: %v", err)
	}

	// === 示例 5: 定时更新搜索建议词库 ===
	// 每 10 分钟执行一次
	// ticker := time.NewTicker(10 * time.Minute)
	// go func() {
	//     for range ticker.C {
	//         suggestion.UpdateSuggestions(ctx)
	//     }
	// }()
}
```

## 反模式

### 1. 在大表上使用 LIKE %query%

```sql
-- 错误：LIKE '%手机%' 全表扫描，不走索引
SELECT * FROM products WHERE name LIKE '%手机%';
```

**问题**：前导通配符 `%` 导致 B-tree 索引失效，商品量大时查询极慢。

**正确做法**：
- < 5000 件：使用 PostgreSQL 全文搜索（`tsvector @@ tsquery` + GIN 索引）
- >= 5000 件：使用 Elasticsearch
- 搜索建议用 `LIKE 'prefix%'`（前缀匹配，可走索引）

### 2. 热门搜索不缓存

```go
// 错误：每次搜索都查数据库，不缓存热门结果
func Search(keyword string) ([]Product, error) {
    return db.Where("name LIKE ?", "%"+keyword+"%").Find(&products).Error
}
// 用户反复搜同一个词（如"手机"），每次都走全文检索
```

**问题**：热门搜索（如"手机"、"iPhone"）占搜索量的 70%+，每次都执行全文搜索浪费数据库资源。

**正确做法**：Redis 缓存热门搜索结果，TTL = 5 分钟。

### 3. 搜索不分页或分页过大

```go
// 错误：无分页限制
func Search(keyword string) ([]Product, error) {
    return db.Where("...").Find(&products).Error // 可能返回 10000+ 条
}
```

**问题**：返回全部结果，内存和响应时间都不可控。

**正确做法**：所有搜索接口必须分页，`page/pageSize`，默认 `pageSize=20`，最大 `100`。

### 4. 搜索结果无相关性排序

```sql
-- 错误：按 ID 排序，搜索结果与关键词无关
SELECT * FROM products WHERE name LIKE '%手机%' ORDER BY id DESC;
```

**问题**：用户搜索"苹果手机"，结果按 ID 排序可能先出现"手机壳"。

**正确做法**：PG 使用 `ts_rank(search_vector, query)` 排序，ES 使用 `_score`。

### 5. 搜索词不做安全过滤

```go
// 错误：用户输入直接拼接进 SQL
query := fmt.Sprintf("SELECT * FROM products WHERE name LIKE '%%%s%%'", userInput)
```

**问题**：SQL 注入风险。

**正确做法**：使用参数化查询（GORM 的 `Where("? LIKE ?", "name", "%"+keyword+"%")`）。

## 常见坑

### 1. GIN 索引导致写入变慢

**坑**：`search_vector` 使用 `GENERATED ALWAYS AS` 自动维护，每次 `INSERT/UPDATE` 都要更新 GIN 索引，大量商品批量导入时写入变慢。

**解决**：
- 批量导入时先 `ALTER TABLE ... DISABLE TRIGGER ALL`，导完后再启用
- 或使用 `SET enable_seqscan = off` 临时关闭索引更新，批量完成后重建索引：`REINDEX INDEX idx_products_search_vector`
- 大批量数据建议使用 Elasticsearch，写入性能更好

### 2. 中文分词效果不佳

**坑**：`simple` 配置按字符切分，搜索"苹果手机"匹配不到"苹果 15 Pro"。

**解决**：
- 安装 `zhparser` 中文分词插件（基于 SCWS）
- 或者在 Elasticsearch 中使用 `ik_max_word` / `ik_smart` 分词器
- 添加同义词库：`手机 => 移动电话, 智能手机`
- 对于拼音搜索，额外维护 `name_pinyin` 字段

### 3. Redis 缓存与数据库不一致

**坑**：商品价格更新后，Redis 缓存的搜索结果仍是旧价格。

**解决**：
- 商品数据变更时调用 `InvalidateCache(keyword)`
- 但关键词可能有很多个（商品名中的每个词），不可能全部失效
- 折中方案：缓存 TTL 设为 5 分钟，可以接受短暂不一致
- 或者缓存中只缓存商品 ID 列表，价格等实时数据从 DB 查

### 4. 搜索建议词库膨胀

**坑**：用户输入的每个搜索词都加入 Redis sorted set，包含大量错别字和无意义词。

**解决**：
- 设置 sorted set 的最大大小（如 10000），超出时删除 score 最低的
- 过滤长度 < 2 的搜索词
- 定期清理：只保留近 7 天的搜索词
- 搜索建议词库应定期从商品表和热门搜索词中重建，而非无限制积累

### 5. 翻页深度过大（Deep Pagination）

**坑**：用户翻到第 100 页，`OFFSET 2000`，PG 需要扫描前 2000 条再跳过，性能急剧下降。

**解决**：
- 限制最大翻页数（如最多 50 页）
- 使用游标分页（`WHERE id < last_seen_id ORDER BY id DESC LIMIT 20`）
- ES 使用 `search_after` 替代 `from/size` 深度翻页

### 6. 搜索缓存击穿

**坑**：某个热门搜索词缓存过期瞬间，大量请求同时到达数据库。

**解决**：
- 使用 singleflight 模式（`golang.org/x/sync/singleflight`）确保同一时刻只有一个请求查 DB
- 或使用互斥锁：缓存过期时先加锁，查完 DB 再释放
- 或使用 probabilistic early expiration（提前 10% 概率刷新缓存）

### 7. ES 与 PG 数据同步延迟

**坑**：商品上架后，ES 索引延迟几秒才更新，用户搜不到新商品。

**解决**：
- 商品变更后通过消息队列（RabbitMQ/Kafka）异步同步到 ES
- 搜索接口可以加一个"仅查 PG"的开关，在数据同步完成前强制查 PG
- 或者使用 Elasticsearch 的 `refresh=wait_for` 参数，写入后等待刷新

### 8. 搜索性能监控缺失

**坑**：搜索慢查询没有监控，用户抱怨搜索慢但开发不知道。

**解决**：
- 搜索响应时间写入 `SearchResponse.TookMs`，超过阈值（如 500ms）记录告警
- PG 开启慢查询日志：`log_min_duration_statement = 500`
- ES 使用 `_slowlog` 记录慢查询
- 定期分析搜索日志，优化热门但慢的查询

## go-zero 集成示例

### 将 SearchService 接入 go-zero 的 svc.ServiceContext

```go
// svc/service_context.go

package svc

import (
	"time"

	"storeforge/model"
	"storeforge/search"

	goRedis "github.com/redis/go-redis/v9"
	"github.com/zeromicro/go-zero/core/stores/redis"
	"github.com/zeromicro/go-zero/rest"
	"gorm.io/gorm"
)

type ServiceContext struct {
	Config     rest.RestConf
	DB         *gorm.DB
	Redis      *redis.Client
	SearchSvc  *search.SearchService
	HotCache   *search.HotSearchCache
	Suggestion *search.SuggestionService
}

func NewServiceContext(c rest.RestConf) *ServiceContext {
	rdb := redis.New(c.Redis.Host)
	dbInstance := model.NewDB(c) // 自行实现 GORM 初始化

	// 搜索组件使用官方 go-redis/v9，与 go-zero 的 redis.Client 类型不同，需独立初始化。
	rdbSearch := goRedis.NewClient(&goRedis.Options{
		Addr: c.Redis.Host,
	})

	// 根据商品数量选择搜索引擎
	var provider search.SearchResultProvider
	productCount := getProductCountFromDB(dbInstance) // 自行实现

	if productCount >= 5000 {
		// provider = search.NewESSearchProvider(esClient) // TODO: 需要额外实现 ES provider
		provider = search.NewPGSearchProvider(dbInstance) // fallback
	} else {
		provider = search.NewPGSearchProvider(dbInstance)
	}

	// 多租户场景：tenantID 应从请求上下文（JWT claim / 子域名 / header）派生，
	// 此处从配置读取仅适用于单租户部署。
	tenantID := c.Name

	hotCache := search.NewHotSearchCache(rdbSearch, tenantID, 5*time.Minute)
	suggestion := search.NewSuggestionService(rdbSearch, tenantID, dbInstance)

	searchSvc := search.NewSearchService(
		provider,
		hotCache,
		suggestion,
		search.SearchServiceConfig{
			EngineType:   search.EnginePostgreSQL,
			ProductCount: productCount,
		},
	)

	return &ServiceContext{
		Config:     c,
		DB:         dbInstance,
		Redis:      rdb,
		SearchSvc:  searchSvc,
		HotCache:   hotCache,
		Suggestion: suggestion,
	}
}
```

在 handler 中使用：

```go
// handler/search_handler.go

package handler

import (
	"net/http"

	"storeforge/internal/svc"
	"storeforge/search"

	"github.com/zeromicro/go-zero/core/logx"
	"github.com/zeromicro/go-zero/rest/httpx"
)

type SearchRequest struct {
	Keyword    string `form:"keyword"`
	CategoryID string `form:"category_id,omitempty"`
	StoreID    string `form:"store_id,omitempty"`
	MinPrice   int64  `form:"min_price,omitempty"`
	MaxPrice   int64  `form:"max_price,omitempty"`
	SortBy     string `form:"sort_by,omitempty"`
	Page       int    `form:"page,omitempty"`
	PageSize   int    `form:"page_size,omitempty"`
}

func SearchHandler(svcCtx *svc.ServiceContext) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		var req SearchRequest
		if err := httpx.Parse(r, &req); err != nil {
			httpx.ErrorCtx(r.Context(), err)
			return
		}

		// 多租户场景：应从 JWT claim 或 X-Tenant-ID header 派生 tenantID。
		tenantID := r.Header.Get("X-Tenant-ID")
		if tenantID == "" {
			tenantID = svcCtx.Config.Name // fallback 单租户
		}
		searchReq := search.SearchRequest{
			TenantID:   tenantID,
			Keyword:    req.Keyword,
			CategoryID: req.CategoryID,
			StoreID:    req.StoreID,
			MinPrice:   req.MinPrice,
			MaxPrice:   req.MaxPrice,
			SortBy:     req.SortBy,
			Page:       req.Page,
			PageSize:   req.PageSize,
		}

		resp, err := svcCtx.SearchSvc.Search(r.Context(), searchReq)
		if err != nil {
			logx.WithContext(r.Context()).Errorf("search failed: %v", err)
			httpx.ErrorCtx(r.Context(), err)
			return
		}

		httpx.OkJsonCtx(r.Context(), w, resp)
	}
}
```

## 测试策略

### 覆盖率目标

- **L1（单元测试） >= 80%**
- L2（集成测试）覆盖 Redis 和 PG/ES 实际查询
- L3（E2E）覆盖完整搜索流程

### Mock SearchResultProvider 示例

```go
package search_test

import (
	"context"
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/mock"
)

// MockSearchResultProvider 模拟搜索提供者
type MockSearchResultProvider struct {
	mock.Mock
}

func (m *MockSearchResultProvider) Search(
	ctx context.Context,
	req SearchRequest,
) (*SearchResponse, error) {
	args := m.Called(ctx, req)
	if args.Get(0) == nil {
		return nil, args.Error(1)
	}
	return args.Get(0).(*SearchResponse), args.Error(1)
}

func (m *MockSearchResultProvider) IndexProduct(ctx context.Context, product Product) error {
	args := m.Called(ctx, product)
	return args.Error(0)
}

func (m *MockSearchResultProvider) DeleteProduct(ctx context.Context, productID string) error {
	args := m.Called(ctx, productID)
	return args.Error(0)
}

func TestSearchService_Search_CacheHit(t *testing.T) {
	// 构建组件（实际测试中 Redis 需要 mock 或用 miniredis）
	mockProvider := new(MockSearchResultProvider)
	// hotCache 和 suggestion 需要真实 Redis 或 miniredis 实例
	// 此处仅演示测试结构

	ctx := context.Background()
	req := SearchRequest{
		TenantID: "test_tenant",
		Keyword:  "手机",
		Page:     1,
		PageSize: 20,
		SortBy:   "relevance",
	}

	expected := &SearchResponse{
		Products: []Product{{ID: "1", Name: "手机壳"}},
		Total:    1,
	}

	mockProvider.On("Search", ctx, req).Return(expected, nil)

	resp, err := mockProvider.Search(ctx, req)
	assert.NoError(t, err)
	assert.Equal(t, int64(1), resp.Total)
	mockProvider.AssertExpectations(t)
}
```

### Redis mock for HotSearchCache

使用 `github.com/alicebob/miniredis/v2` 提供 Redis 内存实例：

```go
package search_test

import (
	"context"
	"testing"
	"time"

	"github.com/alicebob/miniredis/v2"
	"github.com/redis/go-redis/v9"
	"github.com/stretchr/testify/assert"
)

func TestHotSearchCache_GetSet(t *testing.T) {
	s := miniredis.RunT(t)
	rdb := redis.NewClient(&redis.Options{
		Addr: s.Addr(),
	})
	defer rdb.Close()

	cache := NewHotSearchCache(rdb, "test_tenant", 5*time.Minute)
	ctx := context.Background()

	// Set 缓存
	expected := &SearchResponse{
		Products: []Product{{ID: "1", Name: "测试商品"}},
		Total:    1,
		Page:     1,
		PageSize: 20,
		Keyword:  "手机",
	}
	err := cache.Set(ctx, "手机", expected)
	assert.NoError(t, err)

	// Get 缓存
	got, err := cache.Get(ctx, "手机")
	assert.NoError(t, err)
	assert.Equal(t, int64(1), got.Total)

	// 缓存未命中
	got, err = cache.Get(ctx, "不存在的词")
	assert.NoError(t, err)
	assert.Nil(t, got)
}

func TestHotSearchCache_RecordSearch(t *testing.T) {
	s := miniredis.RunT(t)
	rdb := redis.NewClient(&redis.Options{
		Addr: s.Addr(),
	})
	defer rdb.Close()

	cache := NewHotSearchCache(rdb, "test_tenant", 5*time.Minute)
	ctx := context.Background()

	err := cache.RecordSearch(ctx, "手机")
	assert.NoError(t, err)

	// 验证热门排行 key 存在
	score, err := s.Zscore("search:test_tenant:hot:ranking", "手机")
	assert.NoError(t, err)
	assert.Equal(t, float64(1), score)
}

func TestSearchRequest_Normalize(t *testing.T) {
	tests := []struct {
		name     string
		input    SearchRequest
		expected SearchRequest
	}{
		{
			name:     "defaults page and page_size",
			input:    SearchRequest{SortBy: "relevance"},
			expected: SearchRequest{Page: 1, PageSize: 20, SortBy: "relevance"},
		},
		{
			name:     "caps page_size at 100",
			input:    SearchRequest{Page: 1, PageSize: 200, SortBy: "relevance"},
			expected: SearchRequest{Page: 1, PageSize: 100, SortBy: "relevance"},
		},
		{
			name:     "invalid sortBy defaults to relevance",
			input:    SearchRequest{Page: 1, PageSize: 10, SortBy: "invalid"},
			expected: SearchRequest{Page: 1, PageSize: 10, SortBy: "relevance"},
		},
		{
			name:     "empty sortBy defaults to relevance",
			input:    SearchRequest{Page: 1, PageSize: 10, SortBy: ""},
			expected: SearchRequest{Page: 1, PageSize: 10, SortBy: "relevance"},
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			tt.input.Normalize()
			assert.Equal(t, tt.expected.Page, tt.input.Page)
			assert.Equal(t, tt.expected.PageSize, tt.input.PageSize)
			assert.Equal(t, tt.expected.SortBy, tt.input.SortBy)
		})
	}
}
```

### 并发搜索与 singleflight 测试

```go
func TestSearchService_Search_ConcurrentSingleflight(t *testing.T) {
	// 50 个 goroutine 同时搜索同一关键词
	// 验证 singleflight 只调用一次 provider.Search
	// 其余 49 个请求复用缓存结果
	var callCount atomic.Int64
	provider := &countingProvider{count: &callCount}
	// ... 验证 callCount == 1
}
```

### 多租户隔离测试

```go
func TestSearch_MultiTenantIsolation(t *testing.T) {
	// 验证 tenantA 和 tenantB 的热门关键词各自独立
	// Redis key 前缀包含 tenantID：search:{tenantID}:hot:ranking
}
```
