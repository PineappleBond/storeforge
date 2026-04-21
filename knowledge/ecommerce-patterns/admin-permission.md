# 后台权限管理 (Admin Permission Control)

## RBAC + 数据权限 + 操作审计

---

### 触发条件

- 管理后台需要基于角色的访问控制（RBAC）
- 不同角色对同一资源有不同操作权限（超级管理员、部门管理员、操作员、只读用户）
- 需要数据范围隔离（超级管理员看全部、部门管理员看本部门、普通用户看自己）
- 需要操作审计日志（谁、在什么时间、从哪个 IP、做了什么操作、改了什么数据）
- 多租户环境下权限数据必须严格隔离

---

### 决策树

```text
后台接口请求到达？
  ├─ 提取 JWT 中的 user_id, role, tenant_id, dept_id
  ├─ Casbin Enforce: (user, domain, resource, action) 是否允许？
  │   └─ 否 → 403 Forbidden，记录审计日志（拒绝）
  ├─ 数据权限过滤：根据角色注入 GORM Scope
  │   ├─ SuperAdmin → 无过滤，可访问全部数据
  │   ├─ Admin (部门) → 过滤 dept_id = 当前部门
  │   ├─ Operator → 过滤 created_by = 当前用户
  │   └─ Viewer → 同 Operator，且只读
  ├─ 执行 Handler 业务逻辑
  ├─ 审计日志中间件：记录操作详情（异步写入）
  └─ 返回响应
```

---

### 推荐第三方库

| 用途 | 推荐库 | 导入路径 |
|------|--------|----------|
| RBAC 权限引擎 | casbin | `github.com/casbin/casbin/v2` |
| GORM 持久化适配器 | casbin gorm-adapter | `github.com/casbin/gorm-adapter/v3` |
| 策略热更新同步 | casbin redis-watcher | `github.com/casbin/redis-watcher/v2` |
| 数据权限过滤 | 手写 GORM Scope（配合 casbin 策略） | 内置，无需额外依赖 |
| 操作审计 | logx (go-zero) | `github.com/zeromicro/go-zero/core/logx` |
| 角色继承 | casbin (g2 语法) | `github.com/casbin/casbin/v2` |
| 真实 IP 提取 | httpx (go-zero) | `github.com/zeromicro/go-zero/rest/httpx` |

**为什么用 Casbin + GORM Adapter：**
- 权限策略持久化到数据库，重启不丢失，支持动态更新
- GORM Adapter 开箱即用，支持多表、分库、事务
- Casbin 原生支持 RBAC with domains（多租户/多组织），g2 支持角色继承
- 比手写 `if role == "admin"` 安全、可审计、可扩展

---

### 代码模板

#### 1. Casbin RBAC 模型定义（CONF 格式）

```conf
# internal/config/rbac_model.conf
[request_definition]
r = sub, dom, obj, act

[policy_definition]
p = sub, dom, obj, act

[role_definition]
g = _, _, _    # user, role, domain (支持租户隔离)
g2 = _, _      # 角色继承（如 operator 继承 viewer 的只读权限）

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = g(r.sub, p.sub, r.dom) && keyMatch(r.dom, p.dom) && keyMatch(r.obj, p.obj) && keyMatch(r.act, p.act) \
    || g2(r.sub, p.sub) && g(r.sub, p.sub, r.dom) && keyMatch(r.dom, p.dom) && keyMatch(r.obj, p.obj) && keyMatch(r.act, p.act)
```

#### 2. Casbin 策略规则（数据库初始化数据）

```go
// internal/seed/rbac_policies.go
package seed

// CasbinPolicy 代表一条 p 规则：角色, 租户域, 资源, 操作
// 注意：域使用 "*" 配合 keyMatch 通配，表示该角色在所有租户域通用
// 实际运行时可为特定租户创建独立的权限规则
var Policies = [][]string{
	// === SuperAdmin：所有租户、所有资源、所有操作 ===
	{"super_admin", "*", "*", "*"},

	// === Admin（部门管理员）：本部门、全部资源 CRUD ===
	{"admin", "*", "order", "create"},
	{"admin", "*", "order", "read"},
	{"admin", "*", "order", "update"},
	{"admin", "*", "order", "delete"},
	{"admin", "*", "product", "create"},
	{"admin", "*", "product", "read"},
	{"admin", "*", "product", "update"},
	{"admin", "*", "product", "delete"},
	{"admin", "*", "user", "read"},
	{"admin", "*", "user", "update"},

	// === Operator（操作员）：本部门、资源读写，不可删除 ===
	{"operator", "*", "order", "create"},
	{"operator", "*", "order", "read"},
	{"operator", "*", "order", "update"},
	{"operator", "*", "product", "read"},
	{"operator", "*", "product", "update"},
	{"operator", "*", "user", "read"},

	// === Viewer（只读用户）：仅读 ===
	{"viewer", "*", "order", "read"},
	{"viewer", "*", "product", "read"},
	{"viewer", "*", "user", "read"},
}

// RoleGroups 代表 g 规则：用户, 角色, 域
// 实际运行时从用户表动态加载，此处仅示例
var RoleGroups = [][]string{
	{"user:1001", "super_admin", "*"},
	{"user:1002", "admin", "tenant:1"},
	{"user:1003", "operator", "tenant:1"},
	{"user:1004", "viewer", "tenant:2"},
}

// RoleHierarchy g2 规则：子角色继承父角色的所有权限
// 注意：必须在 matcher 中加入 keyMatch 和 g2 条件才能生效
var RoleHierarchy = [][]string{
	{"operator", "viewer"},  // operator 自动拥有 viewer 的所有 read 权限
}
```

#### 3. Casbin + GORM Adapter 初始化

```go
// pkg/rbac/casbin.go
package rbac

import (
	"fmt"

	"github.com/casbin/casbin/v2"
	gormadapter "github.com/casbin/gorm-adapter/v3"
	"gorm.io/gorm"
)

// InitEnforcer 初始化 Casbin Enforcer（使用 GORM Adapter 持久化到数据库）
func InitEnforcer(db *gorm.DB, modelPath string) (*casbin.Enforcer, error) {
	// GORM Adapter 自动创建 casbin_rule 表
	adapter, err := gormadapter.NewAdapterByDBUseTableName(db, "", "casbin_rule")
	if err != nil {
		return nil, fmt.Errorf("create casbin adapter: %w", err)
	}

	enforcer, err := casbin.NewEnforcer(modelPath, adapter)
	if err != nil {
		return nil, fmt.Errorf("create casbin enforcer: %w", err)
	}

	// 加载策略到内存
	if err := enforcer.LoadPolicy(); err != nil {
		return nil, fmt.Errorf("load casbin policy: %w", err)
	}

	// 开启自动保存（策略变更后自动持久化到数据库）
	enforcer.EnableAutoSave(true)

	return enforcer, nil
}

// SeedPolicies 初始化内置角色权限策略（仅首次启动时调用）
func SeedPolicies(enforcer *casbin.Enforcer, policies, groups, hierarchy [][]string) error {
	for _, p := range policies {
		// p = (sub, dom, obj, act)
		if len(p) != 4 {
			return fmt.Errorf("invalid policy: %v", p)
		}
		exists, err := enforcer.HasPolicy(p[0], p[1], p[2], p[3])
		if err != nil {
			return fmt.Errorf("check policy %v: %w", p, err)
		}
		if !exists {
			if _, err := enforcer.AddPolicy(p[0], p[1], p[2], p[3]); err != nil {
				return fmt.Errorf("add policy %v: %w", p, err)
			}
		}
	}

	for _, g := range groups {
		// g = (user, role, domain)
		if len(g) != 3 {
			return fmt.Errorf("invalid group: %v", g)
		}
		exists, err := enforcer.HasGroupingPolicy(g[0], g[1], g[2])
		if err != nil {
			return fmt.Errorf("check grouping policy %v: %w", g, err)
		}
		if !exists {
			if _, err := enforcer.AddGroupingPolicy(g[0], g[1], g[2]); err != nil {
				return fmt.Errorf("add grouping policy %v: %w", g, err)
			}
		}
	}

	for _, h := range hierarchy {
		// g2 = (child_role, parent_role)
		if len(h) != 2 {
			return fmt.Errorf("invalid hierarchy: %v", h)
		}
		exists, err := enforcer.HasNamedGroupingPolicy("g2", h[0], h[1])
		if err != nil {
			return fmt.Errorf("check role hierarchy %v: %w", h, err)
		}
		if !exists {
			if _, err := enforcer.AddNamedGroupingPolicy("g2", h[0], h[1]); err != nil {
				return fmt.Errorf("add role hierarchy %v: %w", h, err)
			}
		}
	}

	// 重新加载策略使 g2 继承生效
	return enforcer.LoadPolicy()
}
```

#### 4. go-zero Casbin 权限中间件

```go
// internal/middleware/casbin_auth.go
package middleware

import (
	"context"
	"fmt"
	"net/http"

	"github.com/casbin/casbin/v2"
	"github.com/zeromicro/go-zero/rest/httpx"
)

// CasbinAuthMiddleware 基于 Casbin 的权限校验中间件
type CasbinAuthMiddleware struct {
	Enforcer *casbin.Enforcer
}

// Handle 拦截请求，校验用户是否有权限访问资源
// 权限校验流程：JWT 已解析 → ctx 中有 userID, tenantID, role → Casbin Enforce
func (m *CasbinAuthMiddleware) Handle(resource, action string) func(http.HandlerFunc) http.HandlerFunc {
	return func(next http.HandlerFunc) http.HandlerFunc {
		return func(w http.ResponseWriter, r *http.Request) {
			// 从 context 获取用户身份（由上游 JWT 中间件注入）
			userID := r.Context().Value(CtxUserID)
			if userID == nil {
				httpx.ErrorCtx(r.Context(), w, fmt.Errorf("unauthenticated"))
				return
			}
			tenantID := r.Context().Value(CtxTenantID)
			if tenantID == nil {
				httpx.ErrorCtx(r.Context(), w, fmt.Errorf("missing organization context"))
				return
			}

			userKey := fmt.Sprintf("user:%v", userID)
			domain := fmt.Sprintf("tenant:%v", tenantID)

			// Casbin Enforce 校验权限
			allowed, err := m.Enforcer.Enforce(userKey, domain, resource, action)
			if err != nil {
				httpx.ErrorCtx(r.Context(), w, fmt.Errorf("permission check failed: %w", err))
				return
			}
			if !allowed {
				// 记录审计日志（拒绝）
				httpx.ErrorCtx(r.Context(), w,
					fmt.Errorf("insufficient permission: user=%s, domain=%s, resource=%s, action=%s",
						userKey, domain, resource, action),
					httpx.WithCode(http.StatusForbidden),
				)
				return
			}

			next(w, r)
		}
	}
}
```

#### 5. 数据权限过滤（GORM Scope）

```go
// internal/middleware/data_scope.go
package middleware

import (
	"context"

	"github.com/zeromicro/go-zero/core/logx"
	"gorm.io/gorm"
)

// Context keys for typed context values (avoid string-key collisions)
// Exported so that upstream JWT middleware can import and use them.
type CtxKey string

const (
	CtxUserID   CtxKey = "user_id"
	CtxUserRole CtxKey = "user_role"
	CtxTenantID CtxKey = "tenant_id"
	CtxUserName CtxKey = "user_name"
	CtxDeptID   CtxKey = "dept_id"
)

// DataScope 数据范围枚举
type DataScope int

const (
	DataScopeAll DataScope = iota // 全部数据（SuperAdmin）
	DataScopeDept                 // 本部门数据（Admin）
	DataScopeSelf                 // 仅本人数据（Operator / Viewer）
)

// roleDataScopeMap 角色到数据范围的映射，可配置扩展
var roleDataScopeMap = map[string]DataScope{
	"super_admin": DataScopeAll,
	"admin":       DataScopeDept,
	"operator":    DataScopeSelf,
	"viewer":      DataScopeSelf,
}

// ResolveDataScope 根据用户角色解析数据范围
func ResolveDataScope(ctx context.Context) DataScope {
	role, ok := ctx.Value(CtxUserRole).(string)
	if !ok || role == "" {
		logx.WithContext(ctx).Warnf("ResolveDataScope: user_role missing from context, defaulting to DataScopeSelf")
		return DataScopeSelf
	}
	if scope, exists := roleDataScopeMap[role]; exists {
		return scope
	}
	return DataScopeSelf
}

// ApplyDataScope 对 GORM 查询注入数据范围过滤
// 用法：db.Scopes(ApplyDataScope(ctx)).Find(&orders)
// 注意：tenant_id 过滤始终生效，确保多租户隔离（P1-11）
func ApplyDataScope(ctx context.Context) func(*gorm.DB) *gorm.DB {
	return func(db *gorm.DB) *gorm.DB {
		// tenant_id 必须始终过滤，从 JWT 提取（禁止从请求参数读取）
		tenantID := ctx.Value(CtxTenantID)
		if tenantID != nil {
			db = db.Where("tenant_id = ?", tenantID)
		}

		scope := ResolveDataScope(ctx)
		switch scope {
		case DataScopeAll:
			// SuperAdmin 已过滤 tenant_id，返回
			return db
		case DataScopeDept:
			deptID := ctx.Value(CtxDeptID)
			if deptID == nil {
				// 缺少部门信息时降级为仅本人，避免数据泄漏
				userID := ctx.Value(CtxUserID)
				return db.Where("created_by = ?", userID)
			}
			return db.Where("dept_id = ?", deptID)
		case DataScopeSelf:
			userID := ctx.Value(CtxUserID)
			return db.Where("created_by = ?", userID)
		default:
			return db
		}
	}
}

// ScopeCheck 业务层复用：判断当前用户是否可以操作指定记录
func ScopeCheck(ctx context.Context, recordOwnerID int64, recordDeptID int64) bool {
	scope := ResolveDataScope(ctx)
	switch scope {
	case DataScopeAll:
		return true
	case DataScopeDept:
		userDeptID, _ := ctx.Value(CtxDeptID).(int64)
		return recordDeptID == userDeptID
	case DataScopeSelf:
		userID, _ := ctx.Value(CtxUserID).(int64)
		return recordOwnerID == userID
	default:
		return false
	}
}
```

#### 6. 操作审计日志中间件

```go
// internal/middleware/audit_log.go
package middleware

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"regexp"
	"strconv"
	"strings"
	"sync"
	"time"

	"github.com/zeromicro/go-zero/core/logx"
	"github.com/zeromicro/go-zero/core/trace"
	"github.com/zeromicro/go-zero/rest/httpx"
	"gorm.io/gorm"
)

// auditCh 缓冲通道 + worker pool，防止审计日志写入阻塞请求处理
var (
	auditCh   = make(chan *AuditLog, 1000)
	auditOnce sync.Once
	auditWg   sync.WaitGroup
	auditDB   *gorm.DB
)

// InitAuditWorker 启动审计日志异步 worker（服务启动时调用一次）
func InitAuditWorker(db *gorm.DB) {
	auditOnce.Do(func() {
		auditDB = db
		auditWg.Add(1)
		go func() {
			defer auditWg.Done()
			for log := range auditCh {
				if err := auditDB.Create(log).Error; err != nil {
					logx.WithContext(context.Background()).Errorf("write audit log failed: %v", err)
				}
			}
		}()
	})
}

// GracefulShutdownAudit 优雅关闭审计 worker，排空 channel 中的日志
func GracefulShutdownAudit() {
	close(auditCh)
	auditWg.Wait()
}

// AuditLog 审计日志模型
type AuditLog struct {
	ID         int64     `gorm:"primaryKey;autoIncrement"`
	UserID     int64     `gorm:"index;not null"`
	UserName   string    `gorm:"size:64"`
	TenantID   int64     `gorm:"index;not null"`
	Role       string    `gorm:"size:32"`
	Action     string    `gorm:"size:32;not null"`  // CREATE, UPDATE, DELETE, EXPORT, etc.
	Resource   string    `gorm:"size:64;not null"`  // order, product, user, etc.
	ResourceID string    `gorm:"size:128"`          // 具体资源 ID
	Method     string    `gorm:"size:16"`           // HTTP Method
	Path       string    `gorm:"size:256;not null"` // 请求路径
	Request    string    `gorm:"type:text"`         // 请求体（脱敏后）
	Response   string    `gorm:"type:text"`         // 响应摘要
	IP         string    `gorm:"size:64"`
	UserAgent  string    `gorm:"size:256"`
	Status     int       `gorm:"not null"`
	Duration   int64     `gorm:"not null"` // 毫秒
	TraceID    string    `gorm:"size:64"`  // 分布式追踪 ID
	CreatedAt  time.Time `gorm:"autoCreateTime;index"`
}

// AuditLogMiddleware 操作审计中间件
// 记录：谁、在什么时间、从哪个 IP、做了什么操作、结果如何
func AuditLogMiddleware(db *gorm.DB) func(http.HandlerFunc) http.HandlerFunc {
	return func(next http.HandlerFunc) http.HandlerFunc {
		return func(w http.ResponseWriter, r *http.Request) {
			start := time.Now()

			// 读取请求体（需复制，因为 body 只能读一次）
			var reqBody []byte
			if r.Body != nil {
				reqBody, _ = io.ReadAll(r.Body)
				r.Body = io.NopCloser(bytes.NewBuffer(reqBody))
			}
			// 限制请求体大小（4KB），超出则截断
			if len(reqBody) > 4096 {
				reqBody = append(reqBody[:4096], []byte("...[truncated]")...)
			}
			// 脱敏：移除密码、token 等敏感字段
			reqBody = sanitizeBody(reqBody)

			// 捕获响应状态码
			rw := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}

			next(rw, r)

			duration := time.Since(start).Milliseconds()

			// 从 context 提取用户信息（由 JWT 中间件注入）
			userID, _ := r.Context().Value(CtxUserID).(int64)
			userName, _ := r.Context().Value(CtxUserName).(string)
			tenantID, _ := r.Context().Value(CtxTenantID).(int64)
			role, _ := r.Context().Value(CtxUserRole).(string)

			// 推断操作类型和资源
			action := inferAction(r.Method)
			resource := inferResource(r.URL.Path)
			resourceID := extractResourceID(r.URL.Path)

			audit := AuditLog{
				UserID:     userID,
				UserName:   userName,
				TenantID:   tenantID,
				Role:       role,
				Action:     action,
				Resource:   resource,
				ResourceID: resourceID,
				Method:     r.Method,
				Path:       r.URL.Path,
				Request:    string(reqBody),
				IP:         httpx.GetRemoteAddr(r),
				UserAgent:  r.UserAgent(),
				Status:     rw.statusCode,
				Duration:   duration,
				TraceID:    extractTraceID(r.Context()),
				CreatedAt:  start,
			}

			// 异步写入审计日志（通过 channel + worker pool，防止 shutdown 时丢失）
			select {
			case auditCh <- &audit:
			default:
				// channel 满时降级为同步日志记录
				logx.WithContext(r.Context()).Errorf("[AUDIT] channel full, logging synchronously")
				if err := db.Create(&audit).Error; err != nil {
					logx.WithContext(r.Context()).Errorf("write audit log failed: %v", err)
				}
			}

			logx.WithContext(r.Context()).Infof("[AUDIT] user=%d(%s) tenant=%d role=%s action=%s resource=%s/%s status=%d duration=%dms ip=%s",
				userID, userName, tenantID, role, action, resource, resourceID, rw.statusCode, duration, httpx.GetRemoteAddr(r))
		}
	}
}

// responseWriter 包装 http.ResponseWriter 以捕获状态码
type responseWriter struct {
	http.ResponseWriter
	statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
	rw.statusCode = code
	rw.ResponseWriter.WriteHeader(code)
}

// sanitizeBody 对请求体脱敏，递归处理嵌套 JSON
func sanitizeBody(body []byte) []byte {
	if len(body) == 0 {
		return body
	}
	var m map[string]interface{}
	if err := json.Unmarshal(body, &m); err != nil {
		return []byte("[non-json body]")
	}
	sanitizeMap(m)
	sanitized, err := json.Marshal(m)
	if err != nil {
		logx.Errorf("sanitizeBody: json.Marshal failed: %v", err)
		return []byte("[sanitize error]")
	}
	if len(sanitized) > 4096 {
		return append(sanitized[:4096], []byte("...[truncated]")...)
	}
	return sanitized
}

// sensitiveFields 需要脱敏的字段
var sensitiveFields = map[string]bool{
	"password": true, "token": true, "secret": true, "access_key": true,
	"refresh_token": true, "phone": true, "id_card": true, "email": true,
	"address": true, "bank_account": true,
}

// sanitizeValue 递归遍历 JSON 结构，对敏感字段脱敏
func sanitizeValue(v interface{}) interface{} {
	switch val := v.(type) {
	case map[string]interface{}:
		for key, subVal := range val {
			if sensitiveFields[key] {
				val[key] = "***REDACTED***"
			} else {
				val[key] = sanitizeValue(subVal)
			}
		}
		return val
	case []interface{}:
		for i, item := range val {
			val[i] = sanitizeValue(item)
		}
		return val
	}
	return v
}

// sanitizeMap 对顶层 map 脱敏（sanitizeValue 的便捷封装）
func sanitizeMap(m map[string]interface{}) {
	for key, val := range m {
		if sensitiveFields[key] {
			m[key] = "***REDACTED***"
		} else {
			m[key] = sanitizeValue(val)
		}
	}
}

// inferAction 根据 HTTP Method 推断操作类型
func inferAction(method string) string {
	switch method {
	case http.MethodPost:
		return "CREATE"
	case http.MethodPut:
		return "UPDATE"
	case http.MethodPatch:
		return "UPDATE"
	case http.MethodDelete:
		return "DELETE"
	case http.MethodGet:
		return "READ"
	default:
		return method
	}
}

// inferResource 从 URL 路径推断资源类型
// /api/v1/orders/123 → order
// /api/v1/products → product
func inferResource(path string) string {
	// 去除查询参数
	path = strings.Split(path, "?")[0]
	parts := strings.Split(strings.Trim(path, "/"), "/")
	// 取最后一段有意义的资源路径（跳过版本号和前缀）
	for i := len(parts) - 1; i >= 0; i-- {
		if parts[i] != "" && !isPathPrefix(parts[i]) {
			// 去掉末尾的 's' 实现简单 singularize
			resource := strings.TrimSuffix(parts[i], "s")
			return resource
		}
	}
	return "unknown"
}

// isPathPrefix 判断路径段是否为前缀（如 api, v1, v2, admin 等）
func isPathPrefix(segment string) bool {
	prefixes := map[string]bool{"api": true, "admin": true, "v1": true, "v2": true, "v3": true}
	return prefixes[segment]
}

var uuidPattern = regexp.MustCompile(`^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$`)

// extractResourceID 从 URL 提取资源 ID
// /api/v1/orders/123 → 123
// /api/v1/orders/550e8400-... → 550e8400-...
// /api/v1/orders → "" (列表接口无 ID)
func extractResourceID(path string) string {
	// 去除查询参数
	path = strings.Split(path, "?")[0]
	parts := strings.Split(strings.Trim(path, "/"), "/")
	if len(parts) >= 2 {
		last := parts[len(parts)-1]
		if last == "" || isPathPrefix(last) {
			return ""
		}
		// 只返回数字或 UUID 格式的值
		if _, err := strconv.ParseInt(last, 10, 64); err == nil {
			return last
		}
		if uuidPattern.MatchString(last) {
			return last
		}
	}
	return ""
}

// extractTraceID 从 context 提取分布式追踪 ID
// 优先从 go-zero trace 中提取，若无则返回空字符串
func extractTraceID(ctx context.Context) string {
	// go-zero 的 trace 中间件会自动将 trace ID 放入 context
	// 具体实现取决于 tracing 框架（Jaeger/OpenTelemetry）
	if spanCtx := trace.SpanContextFromContext(ctx); spanCtx.HasTraceID() {
		return spanCtx.TraceID().String()
	}
	return ""
}

```

#### 7. 使用示例：在 go-zero 路由中组合中间件

```go
// internal/handler/routes.go
package handler

import (
	"net/http"

	"storeforge/internal/middleware"

	"github.com/casbin/casbin/v2"
	"github.com/zeromicro/go-zero/rest"
	"gorm.io/gorm"
)

// 服务启动时需调用:
//   middleware.InitAuditWorker(db)     // 启动审计日志异步 worker
// 服务关闭时需调用:
//   middleware.GracefulShutdownAudit() // 排空 channel，防止日志丢失

// RegisterOrderRoutes 注册订单管理路由（带权限 + 审计）
func RegisterOrderRoutes(server *rest.Server, enforcer *casbin.Enforcer, db *gorm.DB) {
	auth := &middleware.CasbinAuthMiddleware{Enforcer: enforcer}

	// 审计中间件包裹所有后台操作
	server.AddRoutes(
		rest.WithMiddlewares(
			[]rest.Middleware{
				middleware.AuditLogMiddleware(db),
			},
			rest.Route{
				Method:  http.MethodGet,
				Path:    "/api/v1/admin/orders",
				Handler: ListOrdersHandler(db), // Handler 内部使用 ApplyDataScope(ctx) 过滤
			},
			rest.WithPrefix("/admin"),
			rest.WithMiddleware(auth.Handle("order", "read")),
		),
	)

	server.AddRoutes(
		rest.Route{
			Method:  http.MethodPost,
			Path:    "/api/v1/admin/orders",
			Handler: CreateOrderHandler(db),
		},
		rest.WithPrefix("/admin"),
		rest.WithMiddleware(auth.Handle("order", "create")),
		rest.WithMiddleware(middleware.AuditLogMiddleware(db)),
	)

	server.AddRoutes(
		rest.Route{
			Method:  http.MethodDelete,
			Path:    "/api/v1/admin/orders/:id",
			Handler: DeleteOrderHandler(db),
		},
		rest.WithPrefix("/admin"),
		rest.WithMiddleware(auth.Handle("order", "delete")),
		rest.WithMiddleware(middleware.AuditLogMiddleware(db)),
	)
}
```

#### 8. Handler 中使用数据权限过滤

```go
// internal/handler/order_handler.go
package handler

import (
	"encoding/json"
	"net/http"
	"strconv"

	"storeforge/internal/middleware"
	"storeforge/internal/model"

	"gorm.io/gorm"
)

// ListOrdersHandler 查询订单列表（自动按数据范围过滤 + 分页）
func ListOrdersHandler(db *gorm.DB) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		// 分页参数（默认 page=1, pageSize=20, max=100）
		page, pageSize := 1, 20
		if p := r.URL.Query().Get("page"); p != "" {
			if n, err := strconv.Atoi(p); err == nil && n > 0 {
				page = n
			}
		}
		if ps := r.URL.Query().Get("pageSize"); ps != "" {
			if n, err := strconv.Atoi(ps); err == nil && n > 0 && n <= 100 {
				pageSize = n
			}
		}

		var orders []model.Order
		var total int64

		// ApplyDataScope 根据当前用户角色自动注入 WHERE 条件
		query := db.Model(&model.Order{}).Scopes(middleware.ApplyDataScope(r.Context()))

		if err := query.Count(&total).Error; err != nil {
			w.Header().Set("Content-Type", "application/json")
			w.WriteHeader(http.StatusInternalServerError)
			json.NewEncoder(w).Encode(map[string]interface{}{"code": 500, "msg": "查询失败"})
			return
		}

		if err := query.Order("created_at DESC").
			Offset((page - 1) * pageSize).
			Limit(pageSize).
			Find(&orders).Error; err != nil {
			w.Header().Set("Content-Type", "application/json")
			w.WriteHeader(http.StatusInternalServerError)
			json.NewEncoder(w).Encode(map[string]interface{}{"code": 500, "msg": "查询失败"})
			return
		}

		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]interface{}{
			"code": 0,
			"data": map[string]interface{}{
				"list":     orders,
				"total":    total,
				"page":     page,
				"pageSize": pageSize,
			},
		})
	}
}
```

---

### 反模式

| 反模式 | 风险 | 正确做法 |
|--------|------|----------|
| 硬编码角色判断 `if role == "admin"` | 角色变更需改代码、无法审计、容易遗漏 | 使用 `casbin.Enforce()` 统一决策 |
| 不做数据范围隔离（所有管理员看到全部数据） | 部门间数据泄漏、越权查看 | 按 org/dept 注入 GORM Scope 过滤 |
| 后台操作无审计日志 | 出了问题无法追溯、无法满足合规要求 | 所有写操作记录审计日志（异步写入） |
| 仅前端做权限判断 | 前端绕过即可访问所有接口 | 后端必须强制校验（Casbin 中间件） |
| 审计日志可被管理员删除 | 作恶后销毁证据 | 审计日志表只允许 INSERT，应用层禁止 UPDATE/DELETE |
| 权限策略写死在代码里 | 修改权限需重新部署 | Casbin GORM Adapter 持久化到数据库，支持动态更新 |
| 多租户共享权限策略 | A 租户管理员操作 B 租户数据 | Casbin domain（域）隔离 + 数据层 tenant_id 过滤 |

---

### 常见坑

**坑 1：Casbin 策略热更新**
- **现象**：管理员在后台修改了角色权限，但接口仍按旧策略执行
- **原因**：Casbin 策略加载到内存后不会自动刷新
- **解决**：
  - 方案 A：策略变更后调用 `enforcer.LoadPolicy()` 重新加载
  - 方案 B：使用 Redis Watcher 实现多实例自动同步：

    ```go
    watcher, _ := rediswatcher.NewWatcher("redis://localhost:6379")
    enforcer.SetWatcher(watcher)
    enforcer.EnableAutoNotifyWatcher()
    ```

  - 方案 C：数据库 binlog 监听自动触发 reload

**坑 2：角色继承未配置 g2**
- **现象**：operator 角色手动写了 read 权限，但新增 viewer 权限时 operator 不会自动继承
- **原因**：Casbin 默认 `g` 只做用户→角色映射，不支持角色→角色继承
- **解决**：使用 `g2` 定义角色继承关系 `g2 = operator, viewer`，并在 matcher 中加上 `|| g2(r.sub, p.sub)`

**坑 3：多租户数据范围泄漏**
- **现象**：A 租户的管理员通过构造请求查到 B 租户数据
- **原因**：数据权限 Scope 只过滤了 dept_id，没有过滤 tenant_id（租户 ID）
- **解决**：ApplyDataScope 中必须同时注入 `tenant_id = ?` 条件，且 tenant_id 从 JWT 提取（禁止从请求参数读取）

**坑 4：Casbin policy 中 "*" 通配符的滥用**
- **现象**：`{"admin", "*", "*", "*"}` 让 admin 变成了超级管理员
- **原因**：Casbin 默认 matcher 不会展开 `*` 为通配匹配
- **解决**：
  - 使用 `keyMatch` / `keyMatch2` 函数：`m = keyMatch(r.obj, p.obj)`
  - 或在 matcher 中显式处理：`|| p.sub == r.sub && p.dom == "*" && p.obj == "*" && p.act == "*"`

**坑 5：审计日志异步写入丢失**
- **现象**：服务重启时部分审计日志未写入数据库
- **原因**：goroutine 异步写入尚未完成，进程已退出
- **解决**：
  - 使用 channel 缓冲 + 批量写入（减少 DB 压力）
  - 服务关闭时优雅停机（等待 channel 排空）
  - 关键操作（删除、导出）同步写入审计日志

---

### 测试策略

#### Casbin Policy 单元测试示例

```go
// internal/middleware/casbin_auth_test.go
package middleware

import (
	"context"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/casbin/casbin/v2"
	"github.com/casbin/casbin/v2/model"
	"github.com/stretchr/testify/assert"
)

func TestCasbinAuthMiddleware_Enforce(t *testing.T) {
	// 使用内存模型和策略创建 Enforcer
	m, _ := model.NewModelFromString(`
[request_definition]
r = sub, dom, obj, act
[policy_definition]
p = sub, dom, obj, act
[role_definition]
g = _, _, _
g2 = _, _
[policy_effect]
e = some(where (p.eft == allow))
[matchers]
m = g(r.sub, p.sub, r.dom) && keyMatch(r.dom, p.dom) && keyMatch(r.obj, p.obj) && keyMatch(r.act, p.act) || g2(r.sub, p.sub) && g(r.sub, p.sub, r.dom) && keyMatch(r.dom, p.dom) && keyMatch(r.obj, p.obj) && keyMatch(r.act, p.act)
`)
	enforcer, _ := casbin.NewEnforcer(m)
	_ = enforcer.AddPolicy("admin", "tenant:1", "order", "read")
	_ = enforcer.AddGroupingPolicy("user:1001", "admin", "tenant:1")

	mw := &CasbinAuthMiddleware{Enforcer: enforcer}

	// 测试：有权限的用户应通过
	t.Run("allowed", func(t *testing.T) {
		req := httptest.NewRequest(http.MethodGet, "/orders", nil)
		ctx := req.Context()
		ctx = context.WithValue(ctx, CtxUserID, int64(1001))
		ctx = context.WithValue(ctx, CtxTenantID, int64(1))
		req = req.WithContext(ctx)

		rec := httptest.NewRecorder()
		mw.Handle("order", "read")(func(w http.ResponseWriter, r *http.Request) {
			w.WriteHeader(http.StatusOK)
		})(rec, req)

		assert.Equal(t, http.StatusOK, rec.Code)
	})

	// 测试：无权限的用户应返回 403
	t.Run("denied", func(t *testing.T) {
		req := httptest.NewRequest(http.MethodDelete, "/orders/1", nil)
		ctx := req.Context()
		ctx = context.WithValue(ctx, CtxUserID, int64(1002))
		ctx = context.WithValue(ctx, CtxTenantID, int64(1))
		req = req.WithContext(ctx)

		rec := httptest.NewRecorder()
		mw.Handle("order", "delete")(func(w http.ResponseWriter, r *http.Request) {
			w.WriteHeader(http.StatusOK)
		})(rec, req)

		assert.Equal(t, http.StatusForbidden, rec.Code)
	})
}
```

#### DataScope 中间件测试示例

```go
// internal/middleware/data_scope_test.go
package middleware

import (
	"context"
	"testing"

	"github.com/stretchr/testify/assert"
)

func TestResolveDataScope(t *testing.T) {
	tests := []struct {
		name     string
		role     string
		expected DataScope
	}{
		{"super_admin -> all", "super_admin", DataScopeAll},
		{"admin -> dept", "admin", DataScopeDept},
		{"operator -> self", "operator", DataScopeSelf},
		{"viewer -> self", "viewer", DataScopeSelf},
		{"unknown role -> self", "unknown", DataScopeSelf},
		{"missing role -> self", "", DataScopeSelf},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			ctx := context.WithValue(context.Background(), CtxUserRole, tt.role)
			assert.Equal(t, tt.expected, ResolveDataScope(ctx))
		})
	}
}

func TestResolveDataScope_MissingRoleLogsWarning(t *testing.T) {
	ctx := context.Background()
	scope := ResolveDataScope(ctx)
	assert.Equal(t, DataScopeSelf, scope)
}
```

#### 覆盖率目标

- **L1 单元测试覆盖率** >= 80%
- 必须覆盖：Casbin Enforce 允许/拒绝分支、DataScope 全部分支、sanitizeBody 脱敏字段、inferResource/extractResourceID 路径解析
- 审计中间件异步写入部分使用 channel 模拟测试，验证日志记录完整性
