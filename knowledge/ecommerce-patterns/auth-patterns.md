# 认证授权模式 (Auth Patterns)

## 1. JWT RS256 + Refresh Token + Token 撤销

---

### 触发条件

- 需要用户认证/授权的 API 服务
- 微信/支付宝 OAuth 第三方登录
- 管理后台与用户端角色分离
- 需要支持 Token 主动撤销（登出、改密、设备管理）

---

### 决策树

```
需要认证？
  ├─ 内部服务间调用 → go-zero RPC（mTLS + gRPC metadata）
  └─ 客户端请求
      ├─ 需要长会话 / 多设备管理 → JWT RS256 + Refresh Token + 撤销列表
      ├─ 第三方 OAuth（微信/支付宝）→ OAuth 2.0 Authorization Code + state 防 CSRF
      └─ 小程序登录 → wx.login code → 服务端 code2Session → 签发 JWT
```

---

### 推荐第三方库

| 用途 | 推荐库 | 导入路径 |
|------|--------|----------|
| JWT 签发与验证 | golang-jwt/jwt | `github.com/golang-jwt/jwt/v5` |
| OAuth 2.0（微信/支付宝） | golang.org/x/oauth2 | `golang.org/x/oauth2` |
| Redis 操作 | go-redis | `github.com/redis/go-redis/v9` |
| 密码哈希 | golang.org/x/crypto | `golang.org/x/crypto/bcrypt` |

**为什么用 golang.org/x/oauth2：** 微信/支付宝 OAuth 本质是 OAuth 2.0 Authorization Code 流程，手写 http.Get + 字符串拼接容易遗漏 state 校验、token 刷新等安全细节。`golang.org/x/oauth2` 标准库处理所有边界情况。

---

### 代码模板

#### 1.1 JWT RS256 签发与验证中间件

```go
// pkg/jwt/jwt.go
package jwt

import (
	"context"
	"crypto/rsa"
	"crypto/x509"
	"encoding/pem"
	"errors"
	"fmt"
	"os"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

// RS256KeyPair holds RSA key pair for signing and verification
type RS256KeyPair struct {
	PrivateKey *rsa.PrivateKey
	PublicKey  *rsa.PublicKey
}

// LoadKeyPair loads RSA keys from PEM files.
// Accepts ctx for potential remote KMS/HSM integration in the future.
func LoadKeyPair(ctx context.Context, privPath, pubPath string) (*RS256KeyPair, error) {
	privData, err := os.ReadFile(privPath)
	if err != nil {
		return nil, fmt.Errorf("read private key: %w", err)
	}
	privBlock, _ := pem.Decode(privData)
	if privBlock == nil {
		return nil, errors.New("failed to decode private key PEM")
	}
	privKey, err := x509.ParsePKCS1PrivateKey(privBlock.Bytes)
	if err != nil {
		return nil, fmt.Errorf("parse private key: %w", err)
	}

	pubData, err := os.ReadFile(pubPath)
	if err != nil {
		return nil, fmt.Errorf("read public key: %w", err)
	}
	pubBlock, _ := pem.Decode(pubData)
	if pubBlock == nil {
		return nil, errors.New("failed to decode public key PEM")
	}
	pubAny, err := x509.ParsePKIXPublicKey(pubBlock.Bytes)
	if err != nil {
		return nil, fmt.Errorf("parse public key: %w", err)
	}
	pubKey, ok := pubAny.(*rsa.PublicKey)
	if !ok {
		return nil, errors.New("public key is not RSA")
	}

	return &RS256KeyPair{PrivateKey: privKey, PublicKey: pubKey}, nil
}

// Claims 自定义 JWT Claims
type Claims struct {
	UserID   int64  `json:"user_id"`
	TenantID string `json:"tenant_id"`
	Role     string `json:"role"`        // "admin" | "user"
	TokenVer int    `json:"token_ver"`   // 版本计数器，用于撤销
	JTI      string `json:"jti"`         // JWT ID，单设备撤销
	jwt.RegisteredClaims
}

const (
	AccessExpire    = 2 * time.Hour
	RefreshExpire   = 30 * 24 * time.Hour
	TokenVersionTTL = RefreshExpire + 24 * time.Hour // refresh 过期后留 1 天缓冲
)

// GenerateToken 签发 access token
func GenerateToken(kp *RS256KeyPair, userID int64, tenantID, role string, tokenVer int, jti string) (string, error) {
	now := time.Now()
	claims := Claims{
		UserID:   userID,
		TenantID: tenantID,
		Role:     role,
		TokenVer: tokenVer,
		JTI:      jti,
		RegisteredClaims: jwt.RegisteredClaims{
			ExpiresAt: jwt.NewNumericDate(now.Add(AccessExpire)),
			IssuedAt:  jwt.NewNumericDate(now),
			NotBefore: jwt.NewNumericDate(now),
			Issuer:    "storeforge-auth",
			Subject:   fmt.Sprintf("user:%d", userID),
			ID:        jti,
		},
	}
	token := jwt.NewWithClaims(jwt.SigningMethodRS256, claims)
	return token.SignedString(kp.PrivateKey)
}

// GenerateRefreshToken 签发 refresh token
func GenerateRefreshToken(kp *RS256KeyPair, userID int64, tenantID string, tokenVer int, jti string) (string, error) {
	now := time.Now()
	claims := Claims{
		UserID:   userID,
		TenantID: tenantID,
		TokenVer: tokenVer,
		JTI:      jti,
		RegisteredClaims: jwt.RegisteredClaims{
			ExpiresAt: jwt.NewNumericDate(now.Add(RefreshExpire)),
			IssuedAt:  jwt.NewNumericDate(now),
			Issuer:    "storeforge-auth",
			Subject:   fmt.Sprintf("refresh:%d", userID),
			ID:        jti,
		},
	}
	token := jwt.NewWithClaims(jwt.SigningMethodRS256, claims)
	return token.SignedString(kp.PrivateKey)
}

// ParseAndValidate 解析并验证 JWT（使用公钥）
func ParseAndValidate(pubKey *rsa.PublicKey, tokenString string) (*Claims, error) {
	token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(t *jwt.Token) (interface{}, error) {
		if _, ok := t.Method.(*jwt.SigningMethodRSA); !ok {
			return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
		}
		return pubKey, nil
	})
	if err != nil {
		return nil, fmt.Errorf("parse token: %w", err)
	}
	claims, ok := token.Claims.(*Claims)
	if !ok || !token.Valid {
		return nil, errors.New("invalid token claims")
	}
	return claims, nil
}
```

#### 1.2 公共类型（pkg/auth/common.go）

> `APIResponse`、错误码常量、`ctxKey` 类型定义在 `pkg/auth` 共享包中，供 middleware、logic 等所有 auth 相关模块引用。

```go
// pkg/auth/common.go
package auth

import "net/http"

// APIResponse 统一响应信封，与 design doc BaseResponse<T> 对齐
type APIResponse struct {
	Code      string `json:"code"`      // MODULE_SEQ 命名空间，如 AUTH_001
	Message   string `json:"msg"`
	Data      any    `json:"data,omitempty"`
	RequestID string `json:"requestId"`
}

// Auth error codes — MODULE_SEQ namespace (design doc §4.5)
const (
	ErrAuthMissingToken     = "AUTH_001" // 缺少认证令牌
	ErrAuthInvalidToken     = "AUTH_002" // 令牌无效或已过期
	ErrAuthTokenRevoked     = "AUTH_003" // 令牌已撤销
	ErrAuthMissingRefresh   = "AUTH_004" // 缺少 refresh token
	ErrAuthRefreshRevoked   = "AUTH_005" // refresh token 已被撤销
	ErrAuthTooManyRequests  = "AUTH_006" // 刷新请求过于频繁
	ErrAuthInvalidRole      = "AUTH_007" // 角色不匹配
	ErrServiceUnavailable   = "SYS_001"   // 服务暂时不可用
	ErrInternalError        = "SYS_002"   // 服务配置错误
)

// ctxKey is a typed context key to avoid string collisions
type CtxKey string

const (
	CtxUserID   CtxKey = "user_id"
	CtxUserRole CtxKey = "user_role"
	CtxTenantID CtxKey = "tenant_id"
	// Dependency keys — used by middleware to inject, handlers to retrieve
	CtxPubKey  CtxKey = "auth.pubKey"
	CtxRedis   CtxKey = "auth.redis"
	CtxKeyPair CtxKey = "auth.keyPair"
)

// APIError writes a structured JSON error response
func APIError(w http.ResponseWriter, httpStatus int, code string, msg string, r *http.Request) {
	requestID := r.Header.Get("X-Request-ID")
	if requestID == "" {
		requestID = "unknown"
	}
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(httpStatus)
	json.NewEncoder(w).Encode(APIResponse{
		Code:      code,
		Message:   msg,
		RequestID: requestID,
	})
}

// APIOK writes a structured JSON success response
func APIOK[T any](w http.ResponseWriter, data T, r *http.Request) {
	requestID := r.Header.Get("X-Request-ID")
	if requestID == "" {
		requestID = "unknown"
	}
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(APIResponse{
		Code:      "0",
		Message:   "success",
		Data:      data,
		RequestID: requestID,
	})
}
```

#### 1.3 go-zero JWT 中间件

```go
// internal/middleware/auth.go
package middleware

import (
	"context"
	"crypto/rsa"
	"net/http"
	"strings"

	"github.com/redis/go-redis/v9"
	"github.com/zeromicro/go-zero/core/logx"
	"storeforge/pkg/auth"
	"storeforge/pkg/jwt"
	tokenstorepkg "storeforge/pkg/redis"
)

type AuthMiddleware struct {
	PubKey    *rsa.PublicKey
	Rdb       *redis.Client // Redis 地址用于查 token 版本
}

func (m *AuthMiddleware) Handle(next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		authHeader := r.Header.Get("Authorization")
		if authHeader == "" {
			auth.APIError(w, http.StatusUnauthorized, auth.ErrAuthMissingToken, "缺少认证令牌", r)
			return
		}

		parts := strings.SplitN(authHeader, " ", 2)
		if len(parts) != 2 || parts[0] != "Bearer" {
			auth.APIError(w, http.StatusUnauthorized, auth.ErrAuthInvalidToken, "认证令牌格式错误", r)
			return
		}

		claims, err := jwt.ParseAndValidate(m.PubKey, parts[1])
		if err != nil {
			auth.APIError(w, http.StatusUnauthorized, auth.ErrAuthInvalidToken, "令牌无效或已过期", r)
			return
		}

		// 检查 token 版本是否被撤销
		storedVer, err := tokenstorepkg.GetTokenVersion(r.Context(), m.Rdb, claims.TenantID, claims.UserID)
		if err != nil {
			// Redis 不可用时 fail-closed，拒绝请求以保障安全
			logx.WithContext(r.Context()).Errorf("redis unavailable, rejecting token version check: %v", err)
			auth.APIError(w, http.StatusServiceUnavailable, auth.ErrServiceUnavailable, "认证服务暂时不可用", r)
			return
		}
		if claims.TokenVer < storedVer {
			auth.APIError(w, http.StatusUnauthorized, auth.ErrAuthTokenRevoked, "令牌已撤销，请重新登录", r)
			return
		}

		// 注入用户信息到 context
		ctx := context.WithValue(r.Context(), auth.CtxUserID, claims.UserID)
		ctx = context.WithValue(ctx, auth.CtxUserRole, claims.Role)
		ctx = context.WithValue(ctx, auth.CtxTenantID, claims.TenantID)
		next(w, r.WithContext(ctx))
	}
}
```

#### 1.3 Token 撤销列表（Redis 版本计数器）

```go
// pkg/redis/token.go
package tokenstore

import (
	"context"
	"fmt"
	"strconv"
	"time"

	"github.com/redis/go-redis/v9"
	"github.com/zeromicro/go-zero/core/logx"
)

// TokenVersionKey returns the Redis key for a user's token version counter
func TokenVersionKey(tenantID string, userID int64) string {
	return fmt.Sprintf("auth:%s:token_ver:%d", tenantID, userID)
}

// BlacklistKey returns the Redis key for a token blacklist entry
func BlacklistKey(tenantID, tokenJTI string) string {
	return fmt.Sprintf("auth:%s:blacklist:%s", tenantID, tokenJTI)
}

// RevokeTokens 撤销用户所有 token（版本号+1）
// 登录、改密、登出时调用
func RevokeTokens(ctx context.Context, rdb *redis.Client, tenantID string, userID int64) error {
	key := TokenVersionKey(tenantID, userID)
	// INCR 原子操作，返回新版本号
	_, err := rdb.Incr(ctx, key).Result()
	if err != nil {
		return fmt.Errorf("incr token version: %w", err)
	}
	// 设置过期时间（与 refresh token 过期时间对齐），避免永不过期 key 堆积
	rdb.Expire(ctx, key, TokenVersionTTL)
	return nil
}

// GetTokenVersion 获取当前有效 token 版本
func GetTokenVersion(ctx context.Context, rdb *redis.Client, tenantID string, userID int64) (int, error) {
	key := TokenVersionKey(tenantID, userID)
	val, err := rdb.Get(ctx, key).Result()
	if err == redis.Nil {
		return 0, nil // 从未撤销过，版本 0
	}
	if err != nil {
		return 0, fmt.Errorf("get token version: %w", err)
	}
	ver, err := strconv.Atoi(val)
	if err != nil {
		return 0, fmt.Errorf("parse token version: %w", err)
	}
	return ver, nil
}

// RevokeDevice 撤销指定设备（将设备 token 加入黑名单 2h）
func RevokeDevice(ctx context.Context, rdb *redis.Client, tenantID, tokenJTI string) error {
	key := BlacklistKey(tenantID, tokenJTI)
	return rdb.Set(ctx, key, "1", 2*time.Hour).Err()
}

// IsTokenBlacklisted 检查 token 是否在黑名单中
func IsTokenBlacklisted(ctx context.Context, rdb *redis.Client, tenantID, tokenJTI string) (bool, error) {
	key := BlacklistKey(tenantID, tokenJTI)
	_, err := rdb.Get(ctx, key).Result()
	if err == redis.Nil {
		return false, nil
	}
	if err != nil {
		logx.WithContext(ctx).Errorf("redis blacklist check failed: %v", err)
		return false, err
	}
	return true, nil
}
```

#### 1.4 Token 刷新流程

```go
// internal/logic/auth/refresh_logic.go
package authlogic

import (
	"context"
	"crypto/rand"
	"crypto/rsa"
	"encoding/hex"
	"net/http"
	"sync"

	"github.com/redis/go-redis/v9"
	"github.com/zeromicro/go-zero/core/logx"
	"storeforge/pkg/auth"
	"storeforge/pkg/jwt"
	tokenstorepkg "storeforge/pkg/redis"
)

// RefreshDeps holds external dependencies injected at startup
type RefreshDeps struct {
	PubKey  *rsa.PublicKey
	KeyPair *jwt.RS256KeyPair
	Rdb     *redis.Client
	Secure  bool // true 时 Cookie 仅 HTTPS；dev 环境设为 false
}

var refreshDeps RefreshDeps

// RefreshResponse 响应结构（与 BaseResponse<T> 对齐，Data 为 RefreshResponse）
type RefreshResponse struct {
	AccessToken string `json:"accessToken"`
	ExpiresIn   int64  `json:"expiresIn"`
}

// refreshLocks prevents concurrent refresh for the same token JTI
var refreshLocks sync.Map

// generateJTI 生成唯一的 JWT ID
func generateJTI() string {
	b := make([]byte, 16)
	_, _ = rand.Read(b)
	return hex.EncodeToString(b)
}

// RefreshHandler Token 刷新端点（带 Refresh Token Rotation）
// 每次刷新：版本号+1，签发新 access token + 新 refresh token
// 旧 refresh token 立即加入黑名单，防止重放攻击
func RefreshHandler(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()

	// 从 HttpOnly Cookie 读取 refresh token
	cookie, err := r.Cookie("refresh_token")
	if err != nil {
		auth.APIError(w, http.StatusUnauthorized, auth.ErrAuthMissingRefresh, "缺少 refresh token", r)
		return
	}

	// 安全类型断言，防止 panic
	pubKey, ok := ctx.Value(auth.CtxPubKey).(*rsa.PublicKey)
	if !ok {
		logx.WithContext(ctx).Errorf("missing pubKey in context")
		auth.APIError(w, http.StatusInternalServerError, auth.ErrInternalError, "服务配置错误", r)
		return
	}

	claims, err := jwt.ParseAndValidate(pubKey, cookie.Value)
	if err != nil {
		auth.APIError(w, http.StatusUnauthorized, auth.ErrAuthInvalidToken, "refresh token 无效", r)
		return
	}

	rdb, ok := ctx.Value(auth.CtxRedis).(*redis.Client)
	if !ok {
		logx.WithContext(ctx).Errorf("missing redis in context")
		auth.APIError(w, http.StatusInternalServerError, auth.ErrInternalError, "服务配置错误", r)
		return
	}
	tenantID := claims.TenantID

	// 并发刷新锁：防止同一 token 被同时刷新多次
	lockKey := tokenstorepkg.BlacklistKey(tenantID, claims.JTI)
	if _, loaded := refreshLocks.LoadOrStore(lockKey, struct{}{}); loaded {
		auth.APIError(w, http.StatusTooManyRequests, auth.ErrAuthTooManyRequests, "刷新请求过于频繁", r)
		return
	}
	defer refreshLocks.Delete(lockKey)

	// 检查旧 token 是否在黑名单（防重放）
	blacklisted, err := tokenstorepkg.IsTokenBlacklisted(ctx, rdb, tenantID, claims.JTI)
	if err != nil {
		logx.WithContext(ctx).Errorf("blacklist check failed during refresh: %v", err)
	}
	if blacklisted {
		auth.APIError(w, http.StatusUnauthorized, auth.ErrAuthRefreshRevoked, "令牌已被撤销", r)
		return
	}

	// 检查版本是否被撤销 — 从 Redis 读取当前版本，防止竞态
	storedVer, err := tokenstorepkg.GetTokenVersion(ctx, rdb, tenantID, claims.UserID)
	if err != nil {
		logx.WithContext(ctx).Errorf("token version check failed during refresh: %v", err)
	}
	if claims.TokenVer < storedVer {
		auth.APIError(w, http.StatusUnauthorized, auth.ErrAuthTokenRevoked, "令牌已撤销", r)
		return
	}

	keyPair, ok := ctx.Value(auth.CtxKeyPair).(*jwt.RS256KeyPair)
	if !ok {
		logx.WithContext(ctx).Errorf("missing keyPair in context")
		auth.APIError(w, http.StatusInternalServerError, auth.ErrInternalError, "服务配置错误", r)
		return
	}

	// Token Rotation：版本号+1，旧 token 加入黑名单
	if err := tokenstorepkg.RevokeTokens(ctx, rdb, tenantID, claims.UserID); err != nil {
		logx.WithContext(ctx).Errorf("revoke tokens failed during refresh: %v", err)
		auth.APIError(w, http.StatusInternalServerError, auth.ErrInternalError, "刷新令牌失败", r)
		return
	}
	newVer := storedVer + 1

	// 将旧 refresh token 加入黑名单
	if err := tokenstorepkg.RevokeDevice(ctx, rdb, tenantID, claims.JTI); err != nil {
		logx.WithContext(ctx).Errorf("revoke device failed during refresh: %v", err)
	}

	// 签发新 refresh token（带新版本号和 JTI）
	newJTI := generateJTI()
	newRefreshToken, err := jwt.GenerateRefreshToken(keyPair, claims.UserID, tenantID, newVer, newJTI)
	if err != nil {
		logx.WithContext(ctx).Errorf("generate refresh token failed: %v", err)
		auth.APIError(w, http.StatusInternalServerError, auth.ErrInternalError, "签发令牌失败", r)
		return
	}

	// 签发新 access token（保留原始角色，防止 admin 降权）
	newAccessToken, err := jwt.GenerateToken(keyPair, claims.UserID, tenantID, claims.Role, newVer, newJTI)
	if err != nil {
		logx.WithContext(ctx).Errorf("generate access token failed: %v", err)
		auth.APIError(w, http.StatusInternalServerError, auth.ErrInternalError, "签发令牌失败", r)
		return
	}

	// 设置新 refresh token 到 HttpOnly Cookie
	http.SetCookie(w, &http.Cookie{
		Name:     "refresh_token",
		Value:    newRefreshToken,
		Path:     "/auth/refresh",
		HttpOnly: true,
		Secure:   refreshDeps.Secure, // 生产环境必须为 true
		SameSite: http.SameSiteStrictMode,
		MaxAge:   int(jwt.RefreshExpire.Seconds()),
	})

	auth.APIOK(w, RefreshResponse{
		AccessToken: newAccessToken,
		ExpiresIn:   int64(jwt.AccessExpire.Seconds()),
	}, r)
}
```

#### 1.5 微信 OAuth 登录（使用 golang.org/x/oauth2）

```go
// internal/logic/auth/wechat_login.go
package authlogic

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"fmt"
	"net/http"
	"time"

	"github.com/redis/go-redis/v9"
	"github.com/zeromicro/go-zero/core/logx"
	"golang.org/x/oauth2"
	"storeforge/pkg/auth"
)

// generateState generates a cryptographically secure random state string for OAuth CSRF protection
func generateState() string {
	b := make([]byte, 16)
	_, _ = rand.Read(b)
	return hex.EncodeToString(b)
}

// WechatOAuthConfig 微信 OAuth 2.0 端点配置
// 使用 golang.org/x/oauth2 处理 state 生成、token 交换、刷新等全部流程
var WechatOAuthEndpoint = oauth2.Endpoint{
	AuthURL:  "https://open.weixin.qq.com/connect/oauth2/authorize",
	TokenURL: "https://api.weixin.qq.com/sns/oauth2/access_token",
}

func NewWechatOAuthConfig(appID, appSecret string) *oauth2.Config {
	return &oauth2.Config{
		ClientID:     appID,
		ClientSecret: appSecret,
		Endpoint:     WechatOAuthEndpoint,
		RedirectURL:  "https://api.example.com/auth/wechat/callback",
		Scopes:       []string{"snsapi_userinfo"},
	}
}

// WechatAuthRedirect 重定向到微信授权页
// oauth2.Config.AuthCodeURL 自动生成 state 参数并拼接到授权 URL
func WechatAuthRedirect(cfg *oauth2.Config) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		rdb, ok := r.Context().Value(auth.CtxRedis).(*redis.Client)
		if !ok {
			http.Error(w, "服务配置错误", http.StatusInternalServerError)
			return
		}
		state := generateState()
		stateKey := fmt.Sprintf("oauth:wechat:state:%s", state)
		if err := rdb.Set(r.Context(), stateKey, "pending", 5*time.Minute).Err(); err != nil {
			logx.WithContext(r.Context()).Errorf("save wechat oauth state failed: %v", err)
		}

		authURL := cfg.AuthCodeURL(state)
		http.Redirect(w, r, authURL, http.StatusFound)
	}
}

// WechatAuthCallback 微信授权回调
func WechatAuthCallback(cfg *oauth2.Config) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		ctx := r.Context()
		rdb, ok := ctx.Value(auth.CtxRedis).(*redis.Client)
		if !ok {
			http.Error(w, "服务配置错误", http.StatusInternalServerError)
			return
		}

		state := r.URL.Query().Get("state")
		code := r.URL.Query().Get("code")

		if state == "" || code == "" {
			http.Error(w, "缺少 state 或 code 参数", http.StatusBadRequest)
			return
		}

		// 验证 state（防 CSRF）
		stateKey := fmt.Sprintf("oauth:wechat:state:%s", state)
		stateVal, err := rdb.Get(ctx, stateKey).Result()
		if err != nil && err != redis.Nil {
			logx.WithContext(ctx).Errorf("wechat oauth redis error: %v", err)
			auth.APIError(w, http.StatusServiceUnavailable, auth.ErrServiceUnavailable, "认证服务暂时不可用", r)
			return
		}
		if stateVal == "" {
			logx.WithContext(ctx).Errorf("wechat oauth state validation failed: state mismatch or expired")
			http.Error(w, "state 验证失败，请重新授权", http.StatusBadRequest)
			return
		}
		rdb.Del(ctx, stateKey)

		// oauth2.Config.Exchange 完成 token 交换（携带 client_id + client_secret + code）
		// 注意：oauth2 库不负责 state 校验，state 已在上一步通过 Redis 手动验证
		oauthCtx := context.WithValue(ctx, oauth2.HTTPClient, http.DefaultClient)
		token, err := cfg.Exchange(oauthCtx, code)
		if err != nil {
			logx.WithContext(ctx).Errorf("wechat oauth exchange failed: %v", err)
			http.Error(w, "微信授权失败", http.StatusBadRequest)
			return
		}

		// 使用 token.AccessToken 获取用户信息
		// client := cfg.Client(oauthCtx, token)
		// resp, err := client.Get("https://api.weixin.qq.com/sns/userinfo?...")
		// 后续：查找或创建用户，签发 JWT
		_ = token // 示例用
	}
}
```

#### 1.6 支付宝 OAuth 登录

> **注意**：支付宝开放平台使用自定义签名协议（app_id + method + timestamp + RSA 签名），不兼容标准 `oauth2.Config.Exchange`。
> 生产环境应使用支付宝官方 SDK（如 `github.com/smartwalle/alipay/v3`）。以下示例展示 OAuth 2.0 标准流程框架，实际对接时替换为 SDK 调用。

```go
// internal/logic/auth/alipay_login.go
package auth

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"fmt"
	"net/http"
	"time"

	"github.com/redis/go-redis/v9"
	"github.com/zeromicro/go-zero/core/logx"
	"golang.org/x/oauth2"
	"storeforge/pkg/auth"
)

// generateState generates a cryptographically secure random state string for OAuth CSRF protection
func generateState() string {
	b := make([]byte, 16)
	_, _ = rand.Read(b)
	return hex.EncodeToString(b)
}

var AlipayOAuthEndpoint = oauth2.Endpoint{
	AuthURL:  "https://openauth.alipay.com/oauth2/publicAppAuthorize.htm",
	TokenURL: "https://openapi.alipay.com/gateway.do", // 支付宝使用统一网关
}

func NewAlipayOAuthConfig(appID, appSecret string) *oauth2.Config {
	return &oauth2.Config{
		ClientID:     appID,
		ClientSecret: appSecret,
		Endpoint:     AlipayOAuthEndpoint,
		RedirectURL:  "https://api.example.com/auth/alipay/callback",
		Scopes:       []string{"auth_user"},
	}
}

// AlipayAuthRedirect 重定向到支付宝授权页
func AlipayAuthRedirect(cfg *oauth2.Config) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		rdb, ok := r.Context().Value(auth.CtxRedis).(*redis.Client)
		if !ok {
			http.Error(w, "服务配置错误", http.StatusInternalServerError)
			return
		}
		state := generateState()
		stateKey := fmt.Sprintf("oauth:alipay:state:%s", state)
		if err := rdb.Set(r.Context(), stateKey, "pending", 5*time.Minute).Err(); err != nil {
			logx.WithContext(r.Context()).Errorf("save alipay oauth state failed: %v", err)
		}

		authURL := cfg.AuthCodeURL(state)
		http.Redirect(w, r, authURL, http.StatusFound)
	}
}

// AlipayAuthCallback 支付宝授权回调（镜像 Wechat 模式）
func AlipayAuthCallback(cfg *oauth2.Config) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		ctx := r.Context()
		rdb, ok := ctx.Value(auth.CtxRedis).(*redis.Client)
		if !ok {
			http.Error(w, "服务配置错误", http.StatusInternalServerError)
			return
		}

		state := r.URL.Query().Get("state")
		code := r.URL.Query().Get("code")

		if state == "" || code == "" {
			http.Error(w, "缺少 state 或 code 参数", http.StatusBadRequest)
			return
		}

		// 验证 state（防 CSRF）
		stateKey := fmt.Sprintf("oauth:alipay:state:%s", state)
		stateVal, err := rdb.Get(ctx, stateKey).Result()
		if err != nil && err != redis.Nil {
			logx.WithContext(ctx).Errorf("alipay oauth redis error: %v", err)
			http.Error(w, "认证服务暂时不可用", http.StatusServiceUnavailable)
			return
		}
		if stateVal == "" {
			logx.WithContext(ctx).Errorf("alipay oauth state validation failed: state mismatch or expired")
			http.Error(w, "state 验证失败，请重新授权", http.StatusBadRequest)
			return
		}
		rdb.Del(ctx, stateKey)

		// oauth2.Config.Exchange 自动完成 token 交换
		oauthCtx := context.WithValue(ctx, oauth2.HTTPClient, http.DefaultClient)
		token, err := cfg.Exchange(oauthCtx, code)
		if err != nil {
			logx.WithContext(ctx).Errorf("alipay oauth exchange failed: %v", err)
			http.Error(w, "支付宝授权失败", http.StatusBadRequest)
			return
		}

		// 使用 token.AccessToken 获取用户信息
		// client := cfg.Client(oauthCtx, token)
		// resp, err := client.Get("https://openapi.alipay.com/gateway.do?...")
		// 后续：查找或创建用户，签发 JWT
		_ = token // 示例用
	}
}
```

---

### 反模式

| 反模式 | 风险 | 正确做法 |
|--------|------|----------|
| 生产环境使用 HS256 | 密钥泄漏后可伪造任意 Token | 必须 RS256/ES256，私钥仅服务端持有 |
| Token 存储在 localStorage | XSS 攻击可直接窃取 Token | Access Token 存内存，Refresh Token 存 HttpOnly Cookie |
| OAuth 流程不带 state 参数 | CSRF 攻击，攻击者可冒充用户登录 | 必须生成随机 state，回调时验证 |
| Refresh Token 无过期时间 | 永久有效，泄漏后无法自动失效 | 设置 30 天过期，配合版本计数器撤销 |
| 不分 Admin/User 角色 | 权限越权 | JWT claim 中携带 role，中间件严格校验 |
| 撤销后不清理 Redis key | 内存泄漏 | 设置 TTL = TokenVersionTTL（RefreshExpire + 24h），见代码常量 |
| 公钥暴露给前端 | 虽然 RS256 公钥可公开，但不应由前端验证 | 前端只负责携带 token，验证全部在后端 |

---

### 常见坑

**坑 1：RS256 密钥格式问题**
- **现象**：`x509.ParsePKCS1PrivateKey` 报错 `asn1: structure error`
- **原因**：PKCS#8 格式与 PKCS#1 格式不匹配
- **解决**：使用 `openssl genrsa -out private.pem 2048` 生成 PKCS#1；如已有 PKCS#8 用 `openssl rsa -in pkcs8.pem -out pkcs1.pem` 转换

**坑 2：Token 版本计数器初始化**
- **现象**：老用户首次登录 token_ver=0，但 Redis 中没有 key，`GetTokenVersion` 返回 0，正常；但若先撤销再登录，INCR 从 1 开始，新 token 的 ver=0 被拒绝
- **解决**：签发 token 前先读取当前版本号（默认 0），用该版本号签发

**坑 3：Cookie SameSite 导致跨域刷新失败**
- **现象**：前后端分离部署在不同域名，HttpOnly Cookie 无法携带
- **解决**：使用 `SameSite=None; Secure` 且配置 CORS `Access-Control-Allow-Credentials: true`；或采用同主域子域名方案（`api.example.com` + `app.example.com`）

**坑 4：微信 OAuth code 只能使用一次**
- **现象**：重复刷新页面导致 code 已使用报错
- **解决**：code 有效期 5 分钟且单次有效，回调后应立即用掉；页面刷新时应提示用户重新授权

**坑 5：并发刷新导致 token 版本竞态**
- **现象**：两个请求同时刷新，第二个请求的 access token 因版本落后被拒绝
- **解决**：Refresh Token 旋转（rotation）——每次刷新前通过 `sync.Map` 对同一 JTI 加锁，防止并发刷新；旧 refresh token 立即加入黑名单（2h TTL）；版本号仅在全量撤销（登出/改密）时递增，正常刷新时不递增版本号（保持其他设备 token 有效）
- **注意**：`RevokeTokens` 递增版本号会使该用户所有设备的 token 失效（单会话模式）。如需多设备共存，正常刷新不应调用 `RevokeTokens`，改为仅将旧 JTI 加入黑名单即可

---

### 测试策略

#### OAuth Flow Test Example

```go
// TestWechatOAuthFlow 验证完整的 OAuth 授权码流程
func TestWechatOAuthFlow(t *testing.T) {
	// 1. 调用 WechatAuthRedirect，验证 302 重定向到微信授权页
	// 2. 提取重定向 URL 中的 state 参数，验证 Redis 中存在对应 key
	// 3. 携带 mock code + state 调用 WechatAuthCallback
	// 4. 验证 state key 被删除（防重放）
	// 5. 验证 token 被正确交换（mock oauth2.Exchange）
}
```

#### Token Rotation Test

```go
// TestTokenRotation 验证 Refresh Token Rotation 机制
func TestTokenRotation(t *testing.T) {
	// 1. 签发初始 token（ver=0）
	// 2. 第一次刷新：验证版本号递增到 1，返回新 refresh token
	// 3. 使用旧 refresh token 再次刷新：验证被拒绝（黑名单检测）
	// 4. 使用新 refresh token 刷新：验证成功，版本号递增到 2
	// 5. 验证并发刷新时第二个请求被拒绝（版本竞态保护）
}
```

#### 覆盖率目标

| 层级 | 目标 | 说明 |
|--------|------|------|
| L1 单元测试 | >= 80% | JWT 签发/解析、Redis 版本计数器、state 生成与验证 |
| L2 集成测试 | >= 70% | miniredis OAuth 回调端到端、Token 刷新完整流程 |
| L3 E2E 测试 | 关键场景 | 用户登录 -> 刷新 -> 撤销 -> 重新登录 |

#### 多租户隔离测试

```go
func TestTokenVersion_MultiTenantIsolation(t *testing.T) {
	mr := miniredis.RunT(t)
	rdb := redis.NewClient(&redis.Options{Addr: mr.Addr()})
	ctx := context.Background()

	// tenantA 和 tenantB 各自独立的 token version counter
	keyA := fmt.Sprintf("auth:%s:token_ver:1", "tenantA")
	keyB := fmt.Sprintf("auth:%s:token_ver:1", "tenantB")
	rdb.Incr(ctx, keyA)
	rdb.Incr(ctx, keyB)
	rdb.Incr(ctx, keyB) // tenantB 刷新两次

	verA, _ := rdb.Get(ctx, keyA).Int()
	verB, _ := rdb.Get(ctx, keyB).Int()
	if verA != 1 || verB != 2 {
		t.Errorf("tenantA ver=%d want 1, tenantB ver=%d want 2", verA, verB)
	}
}
```

#### 并发刷新测试

```go
func TestTokenRefresh_ConcurrentRace(t *testing.T) {
	// 使用 miniredis + redsync 测试并发 token 刷新
	// 两个 goroutine 同时使用同一 refresh token 刷新
	// 验证只有一个成功，另一个被拒绝（JTI 锁保护）
	// go test -race 检测数据竞争
}
```

#### Token 版本计数器 — 完整单元测试

```go
func TestTokenVersion_RevokeAndGet(t *testing.T) {
	mr := miniredis.RunT(t)
	rdb := redis.NewClient(&redis.Options{Addr: mr.Addr()})
	ctx := context.Background()

	// 初始版本为 0（从未撤销）
	ver, err := tokenstorepkg.GetTokenVersion(ctx, rdb, "t1", 1)
	if err != nil || ver != 0 {
		t.Fatalf("initial version: got %d err %v, want 0", ver, err)
	}

	// 撤销后版本递增
	if err := tokenstorepkg.RevokeTokens(ctx, rdb, "t1", 1); err != nil {
		t.Fatal(err)
	}
	ver, err = tokenstorepkg.GetTokenVersion(ctx, rdb, "t1", 1)
	if err != nil || ver != 1 {
		t.Fatalf("after revoke: got %d err %v, want 1", ver, err)
	}

	// TTL 已设置，验证 key 不会永不过期
	ttl := mr.TTL(fmt.Sprintf("auth:%s:token_ver:%d", "t1", 1))
	if ttl <= 0 {
		t.Fatal("token version key missing TTL")
	}
}
```
