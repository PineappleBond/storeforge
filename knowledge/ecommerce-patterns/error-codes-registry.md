# Error Codes Registry（错误码注册表）

## 触发条件（When to use）

- 任何 API 需要返回业务错误信息
- 前后端需要统一的错误码体系
- 多语言项目（Go 后端 + Vue 前端 + Next.js 官网 + Flutter App）需要错误码对齐
- 运维需要通过错误码快速定位问题（告警、日志分析）

## 决策树（Decision tree）

```text
错误类型是什么？
├── 客户端参数错误（参数校验失败、格式错误）
│   └── HTTP 400 + Code 101xxx/201xxx/301xxx/… → ecode.ErrUserParamInvalid / ecode.ErrOrderParamInvalid
│
├── 业务逻辑错误（库存不足、订单已取消、优惠券已用完、资源不存在）
│   └── HTTP 200 + Code 102xxx/202xxx/302xxx/… → ecode.ErrUserNotFound / ecode.ErrOrderStatusInvalid
│       注: 资源不存在（ErrOrderNotFound、ErrProductNotFound）归类为业务错误，HTTP 返回 200，通过 code 字段标识
│
├── 认证/授权错误（Token 过期、权限不足、未登录）
│   └── HTTP 401 + Code 111xxx → ecode.ErrAuthTokenExpired / ecode.ErrAuthTokenInvalid
│
├── 服务端内部错误（DB 连接失败、第三方 API 超时）
│   └── HTTP 500 + Code {module}5xxx → ecode.ErrSysDBConnection / ecode.ErrPayGatewayTimeout
│
└── 系统级错误（限流、熔断、降级）
    └── HTTP 200 + Code 902001/902003 → ecode.ErrSysRateLimited / ecode.ErrSysDegraded
        注: 限流/降级为业务错误，HTTP 返回 200；如需 HTTP 429/503 语义，在网关层根据 code 转换
```

## 与 go-zero 集成

本文件定义电商领域错误码注册表，**不重复实现 go-zero 已有的错误处理基础设施**。

go-zero 自带能力：

- `core/errorx.CodeError` — 基础错误类型，包含 `Code` 和 `Msg`
- `httpx.SetErrorHandler` — 注册全局 HTTP 错误处理器
- `httpx.OkJsonCtx` / `httpx.ErrorCtx` — 统一响应输出
- `rest.WithNotFoundHandler` — 404 自定义

**集成方式**：在 `svc.ServiceContext` 中通过 `httpx.SetErrorHandler` 注册 `ecode` 错误码到 HTTP 响应的映射。业务层使用 `ecode.New(code, msg)` 构造错误，go-zero handler 层由 `httpx.SetErrorHandler` 统一转换为 `{code, msg, data, requestId}` 格式。

## 代码模板（Code template）

### 1. 错误码分类规则

```text
错误码结构: 6 位数字 — {模块编号2位}{分类1位}{序号3位}

模块编号段:
  USER(10)    AUTH(11)    ORDER(20)     INVENTORY(30)
  PAY(40)     PROMO(50)   PRODUCT(60)   SYS(90)

分类:
  1xxx — 客户端错误（HTTP 4xx）
    1001  参数校验失败
    1002  请求格式错误
    1003  参数越界

  2xxx — 业务错误（HTTP 200 + code != 0）
    2001  资源不存在
    2002  状态不合法
    2003  条件不满足
    2004  操作被拒绝

  5xxx — 服务端错误（HTTP 5xx）
    5001  数据库错误
    5002  第三方服务超时
    5003  系统内部错误
    5004  缓存异常

示例:
  101001 → USER 模块 客户端错误 001 号
  202001 → ORDER 模块 业务错误 001 号
  405003 → PAY 模块 服务端错误 003 号

HTTP 状态码映射:
  200  — 成功（code=0）/ 业务错误（code=2xxx）
  400  — 客户端参数错误
  401  — 认证失败
  403  — 权限不足
  404  — 资源不存在
  429  — 请求过于频繁
  500  — 服务端内部错误
  502  — 网关错误
  503  — 服务降级
  504  — 网关超时
```

### 2. Protobuf 错误码枚举定义（唯一来源）

> 注意：错误码定义为 go-zero `errorx.CodeError` 的业务码扩展。
> 业务层通过 `ecode.New(code, msg)` 构造错误，go-zero handler 层通过 `httpx.SetErrorHandler` 统一输出。

```protobuf
// event/error_codes.proto
syntax = "proto3";

package storeforge.error;

option go_package = "github.com/storeforge/event;eventpb";

// ============================================
// 错误码编号规则: 6 位数字 — {模块2位}{分类1位}{序号3位}
// 模块: USER=10, AUTH=11, ORDER=20, INVENTORY=30, PAY=40, PROMO=50, PRODUCT=60, SYS=90
// 分类: 1=客户端, 2=业务, 5=服务端
// ============================================

// ============================================
// 用户模块 (USER = 10)
// ============================================
enum UserErrorCode {
  USER_UNSPECIFIED = 0;

  // 客户端错误 101xxx
  USER_PARAM_INVALID     = 101001; // 用户参数校验失败
  USER_PHONE_FORMAT      = 101002; // 手机号格式错误
  USER_EMAIL_FORMAT      = 101003; // 邮箱格式错误
  USER_PASSWORD_WEAK     = 101004; // 密码强度不足

  // 业务错误 102xxx
  USER_NOT_FOUND         = 102001; // 用户不存在
  USER_ALREADY_EXISTS    = 102002; // 用户已存在
  USER_DISABLED          = 102003; // 账号已禁用
  USER_PHONE_DUPLICATE   = 102004; // 手机号已被使用

  // 服务端错误 105xxx
  USER_DB_ERROR          = 105001; // 用户数据库异常
  USER_SMS_SEND_FAIL     = 105002; // 短信发送失败
}

// ============================================
// 认证模块 (AUTH = 11)
// ============================================
enum AuthErrorCode {
  AUTH_UNSPECIFIED = 0;

  // 客户端错误 111xxx
  AUTH_TOKEN_MISSING   = 111001; // Token 未提供
  AUTH_TOKEN_EXPIRED   = 111002; // Token 已过期
  AUTH_TOKEN_INVALID   = 111003; // Token 无效
  AUTH_TOKEN_REVOKED   = 111004; // Token 已撤销

  // 业务错误 112xxx
  AUTH_LOGIN_FAILED    = 112001; // 账号或密码错误
  AUTH_LOGIN_TOO_MANY  = 112002; // 登录失败次数过多
  AUTH_SMS_CODE_WRONG  = 112003; // 验证码错误
  AUTH_SMS_CODE_EXPIRED = 112004; // 验证码已过期
  AUTH_REFRESH_INVALID = 112005; // Refresh Token 无效

  // 服务端错误 115xxx
  AUTH_INTERNAL_ERROR  = 115001; // 认证服务内部错误
  AUTH_REDIS_ERROR     = 115002; // Redis 异常
}

// ============================================
// 订单模块 (ORDER = 20)
// ============================================
enum OrderErrorCode {
  ORDER_UNSPECIFIED = 0;

  // 客户端错误 201xxx
  ORDER_PARAM_INVALID    = 201001; // 订单参数错误
  ORDER_ADDR_MISSING     = 201002; // 收货地址缺失
  ORDER_PAYMENT_METHOD   = 201003; // 支付方式无效

  // 业务错误 202xxx
  ORDER_NOT_FOUND        = 202001; // 订单不存在
  ORDER_STATUS_INVALID   = 202002; // 订单状态不合法
  ORDER_ALREADY_PAID     = 202003; // 订单已支付
  ORDER_ALREADY_CANCELLED = 202004; // 订单已取消
  ORDER_TIMEOUT          = 202005; // 订单已超时自动取消
  ORDER_CART_EMPTY       = 202006; // 购物车为空
  ORDER_MIN_AMOUNT       = 202007; // 未达到最低下单金额

  // 服务端错误 205xxx
  ORDER_DB_ERROR         = 205001; // 订单数据库异常
  ORDER_CREATE_FAILED    = 205002; // 订单创建失败
}

// ============================================
// 库存模块 (INVENTORY = 30)
// ============================================
enum InventoryErrorCode {
  INVENTORY_UNSPECIFIED = 0;

  // 客户端错误 301xxx
  INVENTORY_PARAM_INVALID = 301001; // 库存参数错误

  // 业务错误 302xxx
  INVENTORY_OUT_OF_STOCK = 302001; // 库存不足
  INVENTORY_SKU_NOT_FOUND = 302002; // SKU 不存在
  INVENTORY_LOCKED        = 302003; // 库存已锁定

  // 服务端错误 305xxx
  INVENTORY_DB_ERROR     = 305001; // 库存数据库异常
  INVENTORY_REDIS_ERROR  = 305002; // Redis 库存异常
}

// ============================================
// 支付模块 (PAY = 40)
// ============================================
enum PayErrorCode {
  PAY_UNSPECIFIED = 0;

  // 客户端错误 401xxx
  PAY_PARAM_INVALID    = 401001; // 支付参数错误
  PAY_AMOUNT_INVALID   = 401002; // 支付金额异常
  PAY_CURRENCY_INVALID = 401003; // 币种不支持

  // 业务错误 402xxx
  PAY_ORDER_NOT_FOUND  = 402001; // 支付订单不存在
  PAY_ALREADY_PAID     = 402002; // 订单已支付
  PAY_CANCELLED        = 402003; // 用户取消支付
  PAY_BALANCE_INSUFF   = 402004; // 账户余额不足
  PAY_TIMEOUT          = 402005; // 支付已超时

  // 服务端错误 405xxx
  PAY_GATEWAY_TIMEOUT  = 405001; // 支付网关超时
  PAY_GATEWAY_ERROR    = 405002; // 支付网关返回错误
  PAY_SIGNATURE_FAIL   = 405003; // 支付回调验签失败
  PAY_REFUND_FAIL      = 405004; // 退款失败
  PAY_CERT_ERROR       = 405005; // 支付证书异常
}

// ============================================
// 促销模块 (PROMO = 50)
// ============================================
enum PromoErrorCode {
  PROMO_UNSPECIFIED = 0;

  // 客户端错误 501xxx
  PROMO_PARAM_INVALID   = 501001; // 促销参数错误

  // 业务错误 502xxx
  PROMO_NOT_FOUND       = 502001; // 优惠券/活动不存在
  PROMO_EXPIRED         = 502002; // 优惠券已过期
  PROMO_USED_UP         = 502003; // 优惠券已用完
  PROMO_NOT_APPLICABLE  = 502004; // 不满足使用条件
  PROMO_CONFLICT        = 502005; // 优惠券互斥
  PROMO_LIMIT_REACHED   = 502006; // 已达领取上限
  PROMO_ALREADY_CLAIMED = 502007; // 已领取过

  // 服务端错误 505xxx
  PROMO_DB_ERROR        = 505001; // 促销数据库异常
  PROMO_CALC_ERROR      = 505002; // 优惠计算异常
}

// ============================================
// 商品模块 (PRODUCT = 60)
// ============================================
enum ProductErrorCode {
  PRODUCT_UNSPECIFIED = 0;

  // 客户端错误 601xxx
  PRODUCT_PARAM_INVALID = 601001; // 商品参数错误
  PRODUCT_PAGE_INVALID  = 601002; // 分页参数错误

  // 业务错误 602xxx
  PRODUCT_NOT_FOUND     = 602001; // 商品不存在
  PRODUCT_OFF_SHELF     = 602002; // 商品已下架
  PRODUCT_SKU_NOT_FOUND = 602003; // SKU 不存在
  PRODUCT_SKU_OFF_SHELF = 602004; // SKU 已下架

  // 服务端错误 605xxx
  PRODUCT_DB_ERROR      = 605001; // 商品数据库异常
  PRODUCT_SEARCH_ERROR  = 605002; // 搜索服务异常
}

// ============================================
// 系统模块 (SYS = 90)
// ============================================
enum SysErrorCode {
  SYS_UNSPECIFIED = 0;

  // 客户端错误 901xxx
  SYS_PARAM_INVALID     = 901001; // 通用参数错误
  SYS_REQUEST_TOO_LARGE = 901002; // 请求体过大

  // 业务错误 902xxx
  SYS_RATE_LIMITED     = 902001; // 请求频率超限
  SYS_MAINTENANCE      = 902002; // 系统维护中
  SYS_DEGRADED         = 902003; // 服务降级

  // 服务端错误 905xxx
  SYS_INTERNAL_ERROR   = 905001; // 系统内部错误
  SYS_DB_CONNECTION    = 905002; // 数据库连接失败
  SYS_REDIS_CONNECTION = 905003; // Redis 连接失败
  SYS_MQ_ERROR         = 905004; // 消息队列异常
  SYS_UPSTREAM_TIMEOUT = 905005; // 上游服务超时
  SYS_UPSTREAM_ERROR   = 905006; // 上游服务错误
}
```

### 3. Go 错误码定义（`ecode` 包）

> 包名 `ecode`，参考 bilibili/bilibili-discovery/ecode 实践。
> 不遮蔽标准库 `errors`，可使用 `errors.Is` / `errors.As`。
> 与 go-zero 集成：业务层构造 ecode 错误，handler 层通过 `httpx.SetErrorHandler` 统一输出。

```go
// ecode/ecode.go
package ecode

import (
	"context"
	"errors"
	"fmt"
	"net/http"

	"github.com/zeromicro/go-zero/rest/httpx"
)

// Code 错误码（6 位数字: {模块2位}{分类1位}{序号3位}）
type Code int

func (c Code) Int() int        { return int(c) }
func (c Code) String() string  { return fmt.Sprintf("%06d", c) }

// Category 返回错误分类: 1=客户端, 2=业务, 5=服务端
func (c Code) Category() int { return int(c) / 1000 % 10 }

// HTTPStatus 返回对应的 HTTP 状态码（基于模块+分类细化映射）
func (c Code) HTTPStatus() int {
	module := int(c) / 10000        // 前2位: 模块
	category := int(c) / 1000 % 10  // 第3位: 分类

	switch {
	case module == 11 && category == 1: // AUTH 客户端错误 → 401
		return http.StatusUnauthorized
	case category == 1: // 其他模块客户端错误 → 400
		return http.StatusBadRequest
	case category == 2: // 业务错误 → 200 (通过 code 字段标识)
		return http.StatusOK
	case category == 5: // 服务端错误 → 500
		return http.StatusInternalServerError
	default:
		return http.StatusInternalServerError
	}
}

// Error 封装 go-zero errorx.CodeError 的电商扩展
type Error struct {
	Code Code
	Msg  string
}

func (e *Error) Error() string {
	return fmt.Sprintf("[%06d] %s", e.Code.Int(), e.Msg)
}

// New 构造业务错误
func New(code Code, msg string) *Error {
	return &Error{Code: code, Msg: msg}
}

// Newf 带格式化的业务错误
func Newf(code Code, format string, args ...interface{}) *Error {
	return &Error{Code: code, Msg: fmt.Sprintf(format, args...)}
}

// 错误分类常量
const (
	CategoryClient  = 1
	CategoryBusiness = 2
	CategoryServer  = 5
)

// 用户模块错误码 (USER = 10)
const (
	ErrUserParamInvalid   Code = 101001
	ErrUserPhoneFormat    Code = 101002
	ErrUserEmailFormat    Code = 101003
	ErrUserPasswordWeak   Code = 101004
	ErrUserNotFound       Code = 102001
	ErrUserAlreadyExists  Code = 102002
	ErrUserDisabled       Code = 102003
	ErrUserPhoneDuplicate Code = 102004
	ErrUserDBError        Code = 105001
	ErrUserSMSSendFail    Code = 105002
)

// 认证模块错误码 (AUTH = 11)
const (
	ErrAuthTokenMissing   Code = 111001
	ErrAuthTokenExpired   Code = 111002
	ErrAuthTokenInvalid   Code = 111003
	ErrAuthTokenRevoked   Code = 111004
	ErrAuthLoginFailed    Code = 112001
	ErrAuthLoginTooMany   Code = 112002
	ErrAuthSMSCodeWrong   Code = 112003
	ErrAuthSMSCodeExpired Code = 112004
	ErrAuthRefreshInvalid Code = 112005
	ErrAuthInternalError  Code = 115001
	ErrAuthRedisError     Code = 115002
)

// 订单模块错误码 (ORDER = 20)
const (
	ErrOrderParamInvalid    Code = 201001
	ErrOrderAddrMissing     Code = 201002
	ErrOrderPaymentMethod   Code = 201003
	ErrOrderNotFound        Code = 202001
	ErrOrderStatusInvalid   Code = 202002
	ErrOrderAlreadyPaid     Code = 202003
	ErrOrderAlreadyCancelled Code = 202004
	ErrOrderTimeout         Code = 202005
	ErrOrderCartEmpty       Code = 202006
	ErrOrderMinAmount       Code = 202007
	ErrOrderDBError         Code = 205001
	ErrOrderCreateFailed    Code = 205002
)

// 库存模块错误码 (INVENTORY = 30)
const (
	ErrInventoryParamInvalid Code = 301001
	ErrInventoryOutOfStock   Code = 302001
	ErrInventorySKUNotFound  Code = 302002
	ErrInventoryLocked       Code = 302003
	ErrInventoryDBError      Code = 305001
	ErrInventoryRedisError   Code = 305002
)

// 支付模块错误码 (PAY = 40)
const (
	ErrPayParamInvalid     Code = 401001
	ErrPayAmountInvalid    Code = 401002
	ErrPayCurrencyInvalid  Code = 401003
	ErrPayOrderNotFound    Code = 402001
	ErrPayAlreadyPaid      Code = 402002
	ErrPayCancelled        Code = 402003
	ErrPayBalanceInsuff    Code = 402004
	ErrPayTimeout          Code = 402005
	ErrPayGatewayTimeout   Code = 405001
	ErrPayGatewayError     Code = 405002
	ErrPaySignatureFail    Code = 405003
	ErrPayRefundFail       Code = 405004
	ErrPayCertError        Code = 405005
)

// 促销模块错误码 (PROMO = 50)
const (
	ErrPromoParamInvalid   Code = 501001
	ErrPromoNotFound       Code = 502001
	ErrPromoExpired        Code = 502002
	ErrPromoUsedUp         Code = 502003
	ErrPromoNotApplicable  Code = 502004
	ErrPromoConflict       Code = 502005
	ErrPromoLimitReached   Code = 502006
	ErrPromoAlreadyClaimed Code = 502007
	ErrPromoDBError        Code = 505001
	ErrPromoCalcError      Code = 505002
)

// 商品模块错误码 (PRODUCT = 60)
const (
	ErrProductParamInvalid Code = 601001
	ErrProductPageInvalid  Code = 601002
	ErrProductNotFound     Code = 602001
	ErrProductOffShelf     Code = 602002
	ErrProductSKUNotFound  Code = 602003
	ErrProductSKUOffShelf  Code = 602004
	ErrProductDBError      Code = 605001
	ErrProductSearchError  Code = 605002
)

// 系统模块错误码 (SYS = 90)
const (
	ErrSysParamInvalid     Code = 901001
	ErrSysRequestTooLarge  Code = 901002
	ErrSysRateLimited      Code = 902001
	ErrSysMaintenance      Code = 902002
	ErrSysDegraded         Code = 902003
	ErrSysInternalError    Code = 905001
	ErrSysDBConnection     Code = 905002
	ErrSysRedisConnection  Code = 905003
	ErrSysMQError          Code = 905004
	ErrSysUpstreamTimeout  Code = 905005
	ErrSysUpstreamError    Code = 905006
)

// go-zero handler 层注册 — 在 svc.ServiceContext 或 main 中调用
func RegisterHandler(mode string) {
	httpx.SetErrorHandlerCtx(func(ctx context.Context, err error) (int, any) {
		requestID, _ := ctx.Value("RequestId").(string)

		// 默认消息：生产环境隐藏内部细节
		msg := "系统异常，请稍后重试"
		if mode == "dev" || mode == "test" {
			msg = err.Error()
		}

		var e *Error
		if errors.As(err, &e) {
			// ecode 错误：生产环境只返回错误码，开发环境返回详细消息
			if mode == "dev" || mode == "test" {
				msg = e.Msg
			}
			return e.Code.HTTPStatus(), BaseResponse[any]{
				Code:      e.Code.Int(),
				Message:   msg,
				RequestID: requestID,
			}
		}
		// 非 ecode 错误 → 内部错误
		return http.StatusInternalServerError, BaseResponse[any]{
			Code:      ErrSysInternalError.Int(),
			Message:   msg,
			RequestID: requestID,
		}
	})
}
```

### 4. 统一响应信封

```go
// ecode/response.go
package ecode

import "github.com/zeromicro/go-zero/rest/httpx"

// BaseResponse 统一响应结构
type BaseResponse[T any] struct {
	Code      int    `json:"code"`
	Message   string `json:"msg"`
	Data      T      `json:"data"`
	RequestID string `json:"requestId"`
}

// PaginatedData 分页数据
type PaginatedData[T any] struct {
	List     []T   `json:"list"`
	Total    int64 `json:"total"`
	Page     int   `json:"page"`
	PageSize int   `json:"pageSize"`
}
```

> 实际 HTTP 响应输出由 go-zero `httpx.OkJsonCtx` / `httpx.ErrorCtx` 处理，不需要手写 `Success`/`Error` 函数。

### 5. 错误码校验脚本（CI 用）

```go
// ecode/validate_test.go
package ecode

import (
	"testing"
)

func TestErrorCodeNoConflict(t *testing.T) {
	seen := make(map[int]string)

	allCodes := []Code{
		ErrUserParamInvalid, ErrUserPhoneFormat, ErrUserEmailFormat, ErrUserPasswordWeak,
		ErrUserNotFound, ErrUserAlreadyExists, ErrUserDisabled, ErrUserPhoneDuplicate,
		ErrUserDBError, ErrUserSMSSendFail,
		ErrAuthTokenMissing, ErrAuthTokenExpired, ErrAuthTokenInvalid, ErrAuthTokenRevoked,
		ErrAuthLoginFailed, ErrAuthLoginTooMany, ErrAuthSMSCodeWrong, ErrAuthSMSCodeExpired,
		ErrAuthRefreshInvalid, ErrAuthInternalError, ErrAuthRedisError,
		ErrOrderParamInvalid, ErrOrderAddrMissing, ErrOrderPaymentMethod,
		ErrOrderNotFound, ErrOrderStatusInvalid, ErrOrderAlreadyPaid, ErrOrderAlreadyCancelled,
		ErrOrderTimeout, ErrOrderCartEmpty, ErrOrderMinAmount, ErrOrderDBError, ErrOrderCreateFailed,
		ErrInventoryParamInvalid, ErrInventoryOutOfStock, ErrInventorySKUNotFound, ErrInventoryLocked,
		ErrInventoryDBError, ErrInventoryRedisError,
		ErrPayParamInvalid, ErrPayAmountInvalid, ErrPayCurrencyInvalid,
		ErrPayOrderNotFound, ErrPayAlreadyPaid, ErrPayCancelled, ErrPayBalanceInsuff,
		ErrPayTimeout, ErrPayGatewayTimeout, ErrPayGatewayError, ErrPaySignatureFail,
		ErrPayRefundFail, ErrPayCertError,
		ErrPromoParamInvalid, ErrPromoNotFound, ErrPromoExpired, ErrPromoUsedUp,
		ErrPromoNotApplicable, ErrPromoConflict, ErrPromoLimitReached, ErrPromoAlreadyClaimed,
		ErrPromoDBError, ErrPromoCalcError,
		ErrProductParamInvalid, ErrProductPageInvalid, ErrProductNotFound, ErrProductOffShelf,
		ErrProductSKUNotFound, ErrProductSKUOffShelf, ErrProductDBError, ErrProductSearchError,
		ErrSysParamInvalid, ErrSysRequestTooLarge, ErrSysRateLimited, ErrSysMaintenance,
		ErrSysDegraded, ErrSysInternalError, ErrSysDBConnection, ErrSysRedisConnection,
		ErrSysMQError, ErrSysUpstreamTimeout, ErrSysUpstreamError,
	}

	for _, code := range allCodes {
		if existing, dup := seen[int(code)]; dup {
			t.Errorf("错误码冲突: %06d 已被 %s 使用", code.Int(), existing)
		}
		seen[int(code)] = code.String()
	}
}

func TestErrorCodeCategory(t *testing.T) {
	// 1xx → 客户端
	clientCodes := []Code{ErrUserParamInvalid, ErrAuthTokenExpired, ErrOrderParamInvalid}
	for _, code := range clientCodes {
		if code.Category() != CategoryClient {
			t.Errorf("%06d 应为客户端错误，实际分类 %d", code.Int(), code.Category())
		}
	}

	// 2xx → 业务
	businessCodes := []Code{ErrUserNotFound, ErrAuthLoginFailed, ErrInventoryOutOfStock}
	for _, code := range businessCodes {
		if code.Category() != CategoryBusiness {
			t.Errorf("%06d 应为业务错误，实际分类 %d", code.Int(), code.Category())
		}
	}

	// 5xx → 服务端
	serverCodes := []Code{ErrUserDBError, ErrPayGatewayTimeout, ErrSysInternalError}
	for _, code := range serverCodes {
		if code.Category() != CategoryServer {
			t.Errorf("%06d 应为服务端错误，实际分类 %d", code.Int(), code.Category())
		}
	}
}
```

### 6. TypeScript 错误码枚举生成（前端用）

> 以下代码由 `openapi-typescript-codegen` 从 OpenAPI spec 自动生成，禁止手写。

```typescript
// src/types/error-codes.ts — 自动生成，禁止手写

/** 错误码分类 */
export enum ErrorCodeCategory {
  Client = 1,    // 客户端错误
  Business = 2,  // 业务错误
  Server = 5,    // 服务端错误
}

/** 错误码常量（与后端 ecode 包对齐） */
export const ecode = {
  // USER = 10
  USER_PARAM_INVALID: 101001,
  USER_NOT_FOUND: 102001,
  // AUTH = 11
  AUTH_TOKEN_EXPIRED: 111002,
  AUTH_LOGIN_FAILED: 112001,
  // ORDER = 20
  ORDER_NOT_FOUND: 202001,
  ORDER_STATUS_INVALID: 202002,
  // INVENTORY = 30
  INVENTORY_OUT_OF_STOCK: 302001,
  // PAY = 40
  PAY_GATEWAY_TIMEOUT: 405001,
  // PROMO = 50
  PROMO_EXPIRED: 502002,
  PROMO_CONFLICT: 502005,
  // PRODUCT = 60
  PRODUCT_NOT_FOUND: 602001,
  PRODUCT_OFF_SHELF: 602002,
  // SYS = 90
  SYS_RATE_LIMITED: 902001,
  SYS_INTERNAL_ERROR: 905001,
} as const;

/** 错误码 → 前端展示消息映射 */
export const ErrorMessageMap: Record<number, string> = {
  [ecode.USER_NOT_FOUND]: '用户不存在',
  [ecode.AUTH_TOKEN_EXPIRED]: '登录已过期，请重新登录',
  [ecode.INVENTORY_OUT_OF_STOCK]: '库存不足',
  [ecode.PAY_GATEWAY_TIMEOUT]: '支付处理中，请稍后查看订单状态',
  [ecode.SYS_RATE_LIMITED]: '操作过于频繁，请稍后重试',
};

export function getErrorMessage(code: number): string {
  return ErrorMessageMap[code] || '系统异常，请稍后重试';
}
```

## 反模式（Anti-patterns）

| 反模式 | 说明 | 后果 |
|--------|------|------|
| 硬编码错误码 | `return 40001` 直接写数字 | 不同模块使用相同数字导致冲突，无法追溯 |
| 错误码不规范 | 有的用字符串 `"USER_NOT_FOUND"`，有的用数字 `2001` | 前后端对接混乱，无法自动对齐 |
| 没有唯一来源 | Go 定义一套、TypeScript 定义一套、Dart 定义一套 | 三端不一致，前端无法处理某些错误 |
| 错误消息硬编码在 handler | 每个 handler 写 `msg: "库存不足"` | 文案不统一，国际化无法做 |
| HTTP 状态码滥用 | 业务错误返回 HTTP 500 | 监控误报，无法区分"系统故障"和"业务拒绝" |
| 错误码没有分类 | 不区分 1xxx/2xxx/5xxx | 前端无法根据错误类型做不同的 UI 处理 |
| 新增错误码不校验冲突 | 手动分配号码，可能重复 | 两个不同错误共用一个错误码，排查困难 |
| 错误信息包含敏感数据 | 返回数据库异常详情给前端 | 信息泄露（SQL 语句、表结构、内网 IP） |

## 常见坑（Common pitfalls + solutions）

### 坑 1：不同模块错误码冲突

**现象**：各模块都从 `1001` 开始编号，Go 常量在同一 package 下冲突。

**解决**：
- 使用 6 位数字 `{模块2位}{分类1位}{序号3位}`，保证全局数值唯一
- 模块编号段: USER=10, AUTH=11, ORDER=20, INVENTORY=30, PAY=40, PROMO=50, PRODUCT=60, SYS=90
- CI 中运行 `TestErrorCodeNoConflict` 校验

### 坑 2：业务错误用 HTTP 500 返回

**现象**：`库存不足` 返回 500，告警系统误以为服务挂了。

**解决**：
- 业务错误统一返回 HTTP 200 + `code != 0`
- 前端根据 code 分类（Category=2）展示 toast 提示
- 只有真正的系统故障（DB 连接失败、OOM）才返回 HTTP 500

### 坑 3：错误消息未国际化

**现象**：错误消息都是中文，英文站无法使用。

**解决**：
- 后端只返回错误码（`code`），不返回中文消息（生产环境）
- 前端通过 `ErrorMessageMap` 根据当前语言展示对应文案
- 中文消息仅用于开发/调试环境（通过 go-zero `Mode` = `dev` 判断）

### 坑 4：Protobuf 枚举值和 Go 常量不一致

**现象**：Protobuf 中 `USER_NOT_FOUND = 102001`，Go 中手动写成 `102002`。

**解决**：

- 错误码的**唯一来源**是 `.proto` 文件中的枚举定义
- Go `ecode/` 包中的常量必须与 `.proto` 数值严格一致，修改时先改 `.proto`，再同步 Go 常量
- CI 中运行 `TestErrorCodeNoConflict` 校验 Go 常量无冲突、无遗漏
- TypeScript/Dart 代码由 `protoc` 编译自动生成，禁止手写

### 坑 5：错误码过多导致前端难以处理

**现象**：50+ 个错误码，前端需要为每个写 case。

**解决**：
- 前端按**分类**处理：客户端错误 → 表单校验、业务错误 → toast、服务端错误 → 全局提示
- 只有高频错误（如库存不足、登录失败）做精细处理
- 其余错误统一 fallback 到 `ErrorMessageMap[code] || "系统异常，请稍后重试"`

### 坑 6：错误码新增后前端未同步

**现象**：后端新增 `PAY_405006`，前端不知道如何处理。

**解决**：
- CI 流程：protobuf 变更 → 自动生成 Go/TypeScript/Dart 代码
- PR 中必须包含 `openapi-typescript-codegen` 生成的更新
- 错误码变更属于 API 契约变更，需要 review 前端适配

### 坑 7：错误详情暴露敏感信息

**现象**：错误响应中包含 SQL 错误信息。

**解决**：
- `Details` 仅在 go-zero `Mode` = `dev` 时返回
- 生产环境中 `Details` 为空
- 数据库异常统一映射为 `ecode.ErrSysInternalError`，不返回原始错误信息
- 内部错误日志通过 `requestId` 关联追踪，不暴露给前端
