---
name: storeforge-domain-flutter
description: Flutter 多端硬约束和最佳实践
---

# Flutter 硬约束 — 多端（Android/iOS/微信/支付宝/抖音小程序）

> 本文档定义 StoreForge Flutter 多端开发的不可变硬约束。违反反模式清单中任何一条 = BLOCK（阻断），必须修复后方可继续。
> 约束引擎由 `storeforge-harness` 驱动，违反规则时立即停止当前工作。

## 定位

当任务涉及 **Flutter 移动 App / 微信小程序 / 支付宝小程序 / 抖音小程序** 的开发时，必须调用 `storeforge-domain-flutter`。本 skill 定义了 Flutter 多端的项目结构、技术栈、12 条反模式清单和模式推荐，确保 Agent 不走错误路径。

与 `storeforge-domain-backend`、`storeforge-domain-admin`、`storeforge-domain-website` 并列，分别覆盖后端、管理后台、官网、Flutter 四个技术域。

---

## 项目结构（固定，不可变）

```
lib/
├── main.dart                  # 入口，仅做初始化 + runApp
├── app.dart                   # MaterialApp/GoRouter 顶层配置
├── src/
│   ├── features/              # Feature-first 架构，每个功能模块独立
│   │   ├── auth/              # 登录/注册/OAuth/Token 管理
│   │   │   ├── presentation/  # 页面、Widget、状态 Provider
│   │   │   ├── data/          # Repository、API client、DTO
│   │   │   └── domain/        # Entity、UseCase、接口定义
│   │   ├── product/           # 商品浏览/详情/SKU/搜索
│   │   ├── cart/              # 购物车增删改/失效/同步
│   │   ├── order/             # 订单创建/列表/状态/取消
│   │   ├── payment/           # 支付发起/回调/退款
│   │   └── profile/           # 用户信息/地址/设置
│   ├── core/                  # 跨功能共享核心设施
│   │   ├── network/           # Dio 实例、interceptor、错误处理
│   │   ├── theme/             # ThemeData、色彩/字体/间距常量
│   │   ├── routing/           # go_router 路由表、守卫、导航工具
│   │   ├── i18n/              # 多语言资源、本地化工具
│   │   ├── analytics/         # AnalyticsProvider 统一埋点
│   │   └── constants/         # 全局常量（金额单位、超时时间等）
│   ├── data/                  # 共享数据层
│   │   ├── models/            # freezed + json_serializable 数据模型
│   │   ├── repositories/      # 跨 feature 共享 Repository
│   │   └── api_clients/       # 通用 API 客户端封装
│   └── presentation/          # 跨功能共享 UI 组件
│       ├── widgets/           # 通用 Widget（空状态、加载指示器、错误页）
│       ├── layouts/           # 布局模板（Scaffold 封装、底部导航栏）
│       └── styles/            # 共享样式 mixin/扩展
```

### 结构规则

1. **feature-first**：每个业务功能在 `lib/src/features/` 下拥有独立目录，包含 `presentation/`、`data/`、`domain/` 三层
2. **跨功能共享**：只有被两个以上 feature 使用的代码才放在 `core/`、`data/`、`presentation/`
3. **单一职责**：每个文件只做一件事，单个文件不超过 300 行
4. **禁止循环依赖**：domain 层不依赖 data 层，presentation 层不反向依赖 domain 层的具体实现
5. **`main.dart` 仅做初始化**：不写业务逻辑，只做 provider 容器创建、Dio 初始化、GoRouter 配置、`runApp`

---

## 技术栈约束

### 必须使用

| 类别 | 包 | 版本要求 | 用途 |
|------|-----|---------|------|
| 状态管理 | `riverpod` + `riverpod_annotation` | 最新稳定版 | 全局状态、缓存、异步数据 |
| 代码生成 | `freezed` + `freezed_annotation` | 最新稳定版 | 不可变数据模型、联合类型 |
| JSON 序列化 | `json_serializable` + `json_annotation` | 最新稳定版 | 模型 <-> JSON 双向转换 |
| 网络请求 | `dio` + `dio_interceptors` 生态 | 最新稳定版 | HTTP 请求、拦截器、重试 |
| 路由 | `go_router` | 最新稳定版 | 声明式路由、深链接、导航守卫 |
| 图片缓存 | `cached_network_image` | 最新稳定版 | 网络图片缓存、占位图、错误图 |
| 代码生成工具 | `build_runner` | 最新稳定版 | 运行 freezed/json_serializable |

### 禁止使用

| 禁止项 | 替代方案 | 原因 |
|--------|---------|------|
| `provider` 包 | `riverpod` | provider 编译期不安全、测试困难、无自动依赖追踪 |
| `flutter_bloc` / `bloc` | `riverpod` | bloc 样板代码过多、事件流复杂、与 feature-first 不兼容 |
| `get` / `getx` | `riverpod` | getx 隐式全局状态、破坏可测试性、与 Flutter 响应式理念冲突 |
| `Navigator.push` 手动跳转 | `go_router` | 手动路由无法统一管理深链接、导航守卫、路由参数校验 |
| `dart:convert` 手动 JSON 解析 | `freezed` + `json_serializable` | 手写 JSON 易漏字段、无反射安全、重构时不同步 |
| `http` 包直接请求 | `dio` + interceptor | http 包无 interceptor、无内置重试、无 token 刷新机制 |

---

## 反模式清单（违反即 BLOCK 阻断）

以下 12 条反模式，每条均为 BLOCK 级别。违反任何一条 = **立即阻断当前工作**，必须修复后才能继续。

### BLOCK-1：禁止 Provider/Bloc/GetX，必须 Riverpod

**违规示例**：
```dart
// 错误：使用 Provider
ChangeNotifierProvider(create: (_) => CartModel())

// 错误：使用 Bloc
BlocProvider(create: (_) => ProductBloc())

// 错误：使用 GetX
Get.put(CartController())
```

**正确写法**：
```dart
// 正确：使用 Riverpod @riverpod 注解
@riverpod
class CartNotifier extends _$CartNotifier {
  @override
  CartState build() => CartState.initial();
}
```

**原因**：Riverpod 提供编译期安全、自动依赖追踪、无需 BuildContext 即可访问、天然支持测试隔离。其他状态管理方案在电商场景下无法满足多端同步、离线缓存、状态持久化的需求。违反将导致状态不一致和难以追踪的 bug。

---

### BLOCK-2：禁止 Navigator.push 手动跳转，必须 go_router

**违规示例**：
```dart
// 错误：手动 push
Navigator.push(context, MaterialPageRoute(builder: (_) => ProductDetailPage(id: productId)))
```

**正确写法**：
```dart
// 正确：go_router 声明式跳转
context.go('/product/$productId')
// 或
context.push('/product/$productId')
```

**路由配置规则**：
```dart
final router = GoRouter(
  routes: [
    GoRoute(path: '/', builder: (_, __) => const HomePage()),
    GoRoute(path: '/product/:id', builder: (_, state) => ProductDetailPage(id: state.pathParameters['id']!)),
    GoRoute(path: '/cart', builder: (_, __) => const CartPage()),
    GoRoute(path: '/checkout', builder: (_, __) => const CheckoutPage()),
    GoRoute(path: '/order/:id', builder: (_, state) => OrderDetailPage(id: state.pathParameters['id']!)),
  ],
  errorBuilder: (_, __) => const NotFoundPage(),
  redirect: _authGuard, // 登录守卫
);
```

**原因**：手动 Navigator.push 无法统一管理局由表、深链接（Deep Link）、小程序路由映射、导航守卫（如未登录拦截）。在 5 端场景下，go_router 的声明式路由是唯一可维护方案。违反将导致路由混乱和小程序端无法正常导航。

---

### BLOCK-3：禁止手动写 JSON 解析，必须 freezed + json_serializable

**违规示例**：
```dart
// 错误：手写 JSON 解析
class Product {
  final String id;
  final int priceCents;
  Product.fromJson(Map<String, dynamic> json)
      : id = json['id'] as String,
        priceCents = json['price_cents'] as int;
}
```

**正确写法**：
```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'product.freezed.dart';
part 'product.g.dart';

@freezed
class Product with _$Product {
  const factory Product({
    required String id,
    @JsonKey(name: 'price_cents') required int priceCents,
    required String name,
    @JsonKey(name: 'image_urls') required List<String> imageUrls,
    required List<SkuOption> skus,
  }) = _Product;

  factory Product.fromJson(Map<String, dynamic> json) => _$ProductFromJson(json);
}

@freezed
class SkuOption with _$SkuOption {
  const factory SkuOption({
    required String skuId,
    required String spec,
    @JsonKey(name: 'price_cents') required int priceCents,
    @JsonKey(name: 'stock_count') required int stockCount,
  }) = _SkuOption;

  factory SkuOption.fromJson(Map<String, dynamic> json) => _$SkuOptionFromJson(json);
}
```

**原因**：手写 JSON 解析容易遗漏字段、类型不匹配时崩溃、重构时不同步。freezed + json_serializable 提供编译期检查、自动生成不可变模型、支持联合类型（如支付结果的不同状态），大幅降低运行时错误。违反将导致金额字段精度丢失等严重问题。

---

### BLOCK-4：禁止 http 包直接请求，必须 Dio + interceptor（token 刷新、重试、日志）

**违规示例**：
```dart
// 错误：使用 http 包直接请求
final response = await http.get(Uri.parse('/api/v1/products'));
```

**正确写法**：
```dart
// lib/src/core/network/dio_client.dart
final dioProvider = Provider<Dio>((ref) {
  final dio = Dio(BaseOptions(
    baseUrl: const String.fromEnvironment('API_BASE_URL'),
    connectTimeout: const Duration(seconds: 10),
    receiveTimeout: const Duration(seconds: 15),
    headers: {'Content-Type': 'application/json'},
  ));

  // interceptor 顺序：token 刷新 -> 日志 -> 重试
  dio.interceptors.addAll([
    TokenRefreshInterceptor(ref),    // 先：401 时自动刷新 token
    LoggingInterceptor(),            // 中：请求/响应日志（仅 debug）
    RetryInterceptor(                // 后：5xx/超时自动重试
      retries: 2,
      retryDelay: const Duration(milliseconds: 500),
    ),
  ]);

  return dio;
});

// Token 刷新 interceptor 核心逻辑
class TokenRefreshInterceptor extends Interceptor {
  final Ref ref;
  bool _isRefreshing = false;
  final _queue = <(RequestOptions, Completer<Response>)>[];

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    if (err.response?.statusCode == 401 && !_isRefreshing) {
      _isRefreshing = true;
      try {
        await ref.read(authRepositoryProvider).refreshToken();
        // 重试队列中的请求
        for (final entry in _queue) {
          final newOptions = entry.$1.copyWith(
            headers: {'Authorization': 'Bearer ${ref.read(authTokenProvider)}'},
          );
          final response = await ref.read(dioProvider).fetch(newOptions);
          entry.$2.complete(response);
        }
        _queue.clear();
      } finally {
        _isRefreshing = false;
      }
      // 重试原始请求
      final opts = err.requestOptions.copyWith(
        headers: {'Authorization': 'Bearer ${ref.read(authTokenProvider)}'},
      );
      final response = await ref.read(dioProvider).fetch(opts);
      handler.resolve(response);
    } else if (err.response?.statusCode == 401 && _isRefreshing) {
      // token 正在刷新中，加入队列
      final completer = Completer<Response>();
      _queue.add((err.requestOptions, completer));
      completer.future.then((r) => handler.resolve(r));
    } else {
      handler.next(err);
    }
  }
}
```

**interceptor 顺序至关重要**：
1. **TokenRefreshInterceptor**（第一位）：处理 401，刷新 token 后重试
2. **LoggingInterceptor**（第二位）：记录请求/响应，仅 debug 模式启用
3. **RetryInterceptor**（第三位）：5xx/网络超时自动重试，最多 2 次

**原因**：电商场景下 token 过期是常态，无自动刷新机制将导致用户频繁掉线。无重试机制将导致网络抖动时大量请求失败。无日志拦截器将无法排查线上问题。违反将直接影响用户体验和线上故障排查能力。

---

### BLOCK-5：禁止商品详情页不支持 SKU 选择/多图轮播/分享

**违规内容**：商品详情页缺少以下任一功能：
- SKU 规格选择器（颜色、尺码、配置等）
- 多图轮播（支持手势滑动、缩略图导航）
- 分享功能（生成分享卡片/链接）

**原因**：SKU 选择是电商核心路径（影响价格、库存、下单），多图轮播是用户决策关键，分享是增长核心渠道。缺失任一将导致转化率下降。这是电商业务红线，不是技术选择。

---

### BLOCK-6：购物车不支持离线缓存/跨端同步/失效标记

**违规内容**：购物车功能缺少以下任一能力：
- **离线缓存**：无网络时可查看本地购物车内容（Hive/SharedPreferences 持久化）
- **跨端同步**：登录后自动合并本地与远程购物车数据
- **失效标记**：商品下架/库存不足/价格变更时明确标记

**正确模式**（详见 `knowledge/ecommerce-patterns/cart-patterns.md`）：
```dart
@riverpod
Future<List<CartItem>> cartItems(CartItemsRef ref) async {
  final localCart = await ref.watch(localCartProvider.future);
  final remoteCart = ref.watch(remoteCartProvider);

  // 合并策略：登录后以服务端为准，未登录使用本地缓存
  return remoteCart.when(
    data: (remote) => _mergeCarts(localCart, remote),
    loading: () => localCart,
    error: (_, __) => localCart,
  );
}
```

**原因**：用户在小程序/APP 间切换是常态，不同步购物车数据将导致重复加购或丢失商品。离线缓存是基础体验，失效标记避免下单失败。违反将导致用户流失和订单异常。

---

### BLOCK-7：禁止支付不接入条件编译（微信/支付宝/抖音）

**违规内容**：支付功能未使用条件编译区分 5 端，导致：
- 微信小程序调用支付宝 SDK
- 抖音小程序调用微信支付
- 非小程序端调用小程序特定 API

**正确写法**：
```dart
// lib/src/features/payment/data/payment_gateway.dart
Future<PaymentResult> processPayment({
  required PaymentMethod method,
  required String orderId,
  required int amountCents,
}) async {
  // 条件编译：微信小程序
  if (const String.fromEnvironment('TARGET_PLATFORM') == 'wechat_mini_program') {
    return _wechatPay(orderId, amountCents);
  }

  // 条件编译：支付宝小程序
  if (const String.fromEnvironment('TARGET_PLATFORM') == 'alipay_mini_program') {
    return _alipayPay(orderId, amountCents);
  }

  // 条件编译：抖音小程序
  if (const String.fromEnvironment('TARGET_PLATFORM') == 'douyin_mini_program') {
    return _douyinPay(orderId, amountCents);
  }

  // Android/iOS 原生支付
  return _nativePay(method, orderId, amountCents);
}
```

**构建时指定平台**：
```bash
flutter build apk --dart-define=TARGET_PLATFORM=android
flutter build ios --dart-define=TARGET_PLATFORM=ios
flutter build web --dart-define=TARGET_PLATFORM=wechat_mini_program  # 小程序编译
```

**支付模式参考**：详见 `knowledge/ecommerce-patterns/payment-patterns.md`（小程序支付通过服务端 exchange code 流程）

**原因**：5 端支付 SDK 完全不同，API 签名、回调处理、验签方式各异。不区分平台将导致编译错误或运行时崩溃。支付是电商核心链路，错误直接影响收入。

---

### BLOCK-8：禁止 Image.network，必须 CachedNetworkImage（含 maxSizeBytes 限制）

**违规示例**：
```dart
// 错误：直接使用 Image.network
Image.network(product.imageUrl)
```

**正确写法**：
```dart
CachedNetworkImage(
  imageUrl: product.imageUrl,
  placeholder: (_, __) => const CircularProgressIndicator(),
  errorWidget: (_, __, ___) => const Icon(Icons.error, color: Colors.grey),
  memCacheWidth: 800,              // 内存缓存宽度限制
  memCacheHeight: 800,             // 内存缓存高度限制
  maxWidthDiskCache: 1200,         // 磁盘缓存宽度
  maxHeightDiskCache: 1200,        // 磁盘缓存高度
  imageRenderMethodParameters: {   // maxSizeBytes 限制
    'maxSizeBytes': 5 * 1024 * 1024, // 单图最大 5MB
  },
)
```

**原因**：`Image.network` 无缓存，每次滚动重建都会重新请求图片，导致流量浪费、页面卡顿、OOM 崩溃。电商商品图列表尤其严重。CachedNetworkImage 提供内存+磁盘双层缓存，`maxSizeBytes` 防止大图撑爆内存。违反将导致低端设备 OOM 和列表严重卡顿。

---

### BLOCK-9：禁止埋点分散，必须统一 AnalyticsProvider

**违规示例**：
```dart
// 错误：埋点分散在各个页面
class ProductDetailPage extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    FirebaseAnalytics.instance.logEvent(name: 'view_product', parameters: {...});
    return ...;
  }
}
```

**正确写法**：
```dart
// lib/src/core/analytics/analytics_provider.dart
@riverpod
AnalyticsService analyticsService(AnalyticsServiceRef ref) {
  return AnalyticsService(ref);
}

class AnalyticsService {
  final Ref _ref;
  AnalyticsService(this._ref);

  // 统一埋点入口
  Future<void> trackProductView({required String productId, required String productName}) async {
    _track('product_view', {
      'product_id': productId,
      'product_name': productName,
      'timestamp': DateTime.now().toIso8601String(),
    });
  }

  Future<void> trackCartAdd({required String productId, required int quantity}) async {
    _track('cart_add', {
      'product_id': productId,
      'quantity': quantity,
    });
  }

  Future<void> trackPaymentStart({required String orderId, required String method}) async {
    _track('payment_start', {
      'order_id': orderId,
      'payment_method': method,
    });
  }

  void _track(String event, Map<String, String> params) {
    // 统一上报逻辑：可切换 Firebase/GA/自定义埋点平台
    // 平台条件编译：小程序端使用平台特定埋点 API
    if (const String.fromEnvironment('TARGET_PLATFORM') == 'wechat_mini_program') {
      // 微信小程序埋点
    } else {
      // APP 端统一埋点
    }
  }
}

// 页面中使用
ref.read(analyticsServiceProvider).trackProductView(productId: product.id, productName: product.name);
```

**原因**：埋点分散导致难以维护、遗漏关键事件、数据口径不一致。电商核心漏斗（浏览->加购->下单->支付）必须完整追踪。统一 AnalyticsProvider 保证埋点一致性，方便切换埋点平台和 A/B 测试。

---

### BLOCK-10：禁止平台特定代码不用条件编译（5端差异处理）

**违规内容**：在非对应平台使用平台特定 API，包括：
- 微信 SDK 相关调用未在微信小程序条件编译下
- 支付宝 SDK 相关调用未在支付宝小程序条件编译下
- 抖音 SDK 相关调用未在抖音小程序条件编译下
- 平台原生 API（相机、定位、支付、分享）未做平台判断

**正确写法**：
```dart
// 条件编译统一使用 const String.fromEnvironment('TARGET_PLATFORM')
String get platformName =>
    const String.fromEnvironment('TARGET_PLATFORM');

Future<ShareResult> share({required String title, required String url}) async {
  final platform = platformName;

  return switch (platform) {
    'wechat_mini_program' => _wechatShare(title, url),
    'alipay_mini_program' => _alipayShare(title, url),
    'douyin_mini_program' => _douyinShare(title, url),
    'android' => _nativeShare(title, url),
    'ios' => _nativeShare(title, url),
    _ => throw UnsupportedError('Unsupported platform: $platform'),
  };
}
```

**5 端差异速查表**：

| 差异点 | Android/iOS | 微信小程序 | 支付宝小程序 | 抖音小程序 |
|--------|------------|-----------|-------------|-----------|
| 支付 | 原生 SDK | 小程序支付 API | 小程序支付 API | 小程序支付 API |
| 登录 | OAuth/手机号 | code2Session | authCode | tt.login |
| 分享 | share_plus | onShareAppMessage | onShareAppMessage | tt.shareAppMessage |
| 埋点 | Firebase/GA | 自定义事件 | 自定义事件 | 自定义事件 |
| 图片选择 | image_picker | chooseImage | chooseImage | chooseImage |

**原因**：5 端运行时环境完全不同，调用不存在 API 直接导致崩溃。电商场景下支付、登录、分享都是核心路径，任何一端崩溃都意味着收入损失。

---

### BLOCK-11：禁止 ListView 全量渲染，必须 ListView.builder / SliverList 懒加载

**违规示例**：
```dart
// 错误：全量渲染所有子 Widget
ListView(
  children: productList.map((p) => ProductCard(product: p)).toList(),
)
```

**正确写法**：
```dart
// 正确：懒加载
ListView.builder(
  itemCount: productList.length,
  itemBuilder: (context, index) => ProductCard(product: productList[index]),
)

// 复杂滚动场景：SliverList + CustomScrollView
CustomScrollView(
  slivers: [
    SliverAppBar(expandedHeight: 200, flexibleSpace: ...),
    SliverList(
      delegate: SliverChildBuilderDelegate(
        (context, index) => ProductCard(product: productList[index]),
        childCount: productList.length,
      ),
    ),
  ],
)
```

**原因**：商品列表通常有几十到上百个 item，全量渲染会创建所有 Widget 导致严重内存占用和首屏卡顿。电商场景下列表页面是用户停留最长的页面，性能直接影响转化率。违反将导致低端设备崩溃和高端设备明显卡顿。

---

### BLOCK-12：禁止小程序端调用非条件编译的原生 API

**违规内容**：在小程序端直接调用原生平台 API（如 MethodChannel、platform channels），而非通过条件编译或小程序特定 API 替代。

**违规示例**：
```dart
// 错误：在小程序中直接使用 MethodChannel
const channel = MethodChannel('com.example/native_feature');
final result = await channel.invokeMethod('getBatteryLevel');
```

**正确写法**：
```dart
// 通过条件编译隔离平台特定代码
Future<String> getDeviceInfo() async {
  if (const String.fromEnvironment('TARGET_PLATFORM') == 'android' ||
      const String.fromEnvironment('TARGET_PLATFORM') == 'ios') {
    return _nativeDeviceInfo();
  }
  // 小程序端使用小程序 API 替代或返回降级信息
  return 'mini_program_unsupported';
}
```

**原因**：小程序运行在宿主 App 的沙箱中，没有完整的原生运行时环境。直接调用 MethodChannel 会导致小程序崩溃或白屏。电商场景下这可能导致支付、登录等关键路径完全失效。

---

## 模式推荐

### 状态管理：Riverpod (stateless) + freezed

```
遇到场景                    -> 推荐模式
─────────────────────────────────────────
异步数据加载（商品列表）      -> @riverpod FutureProvider + freezed AsyncValue
表单状态（登录/注册）        -> @riverpod StateNotifierProvider + freezed 状态联合类型
全局配置（主题/语言）        -> @riverpod StateProvider + SharedPreferences 持久化
缓存数据（用户信息/购物车）   -> @riverpod FutureProvider.autoDispose.family + keepAlive
复杂业务逻辑（订单流程）      -> @riverpod NotifierProvider + freezed 事件流
```

**Riverpod provider 生命周期管理**：
```dart
// 使用 @riverpod 注解 + keepAlive 控制缓存
@Riverpod(keepAlive: true)  // 持久化：应用生命周期内不销毁（如用户信息）
Future<UserProfile> userProfile(UserProfileRef ref) async { ... }

@Riverpod(keepAlive: false) // 自动销毁：离开页面后释放（如商品详情）
Future<ProductDetail> productDetail(ProductDetailRef ref, String id) async { ... }

// 手动控制失效
ref.invalidate(userProfileProvider);  // 用户信息更新后主动失效
```

### 路由：go_router 声明式配置

```
遇到场景                    -> 推荐模式
─────────────────────────────────────────
页面跳转                     -> context.go('/path') 替换当前栈
打开新页面保留返回           -> context.push('/path')
深链接/Universal Link        -> go_router 自动解析 URL -> 对应页面
未登录拦截                   -> GoRouter redirect 回调
带参数跳转                   -> path parameters + query parameters
底部导航 Tab                 -> StatefulShellRoute
```

### 网络：Dio interceptor 统一处理

```
遇到场景                    -> 推荐模式
─────────────────────────────────────────
统一 token 注入              -> Interceptor onRequest 添加 Authorization header
token 过期自动刷新           -> Interceptor onError 拦截 401 -> 刷新 -> 重试
请求失败自动重试             -> RetryInterceptor（5xx/超时，最多 2 次）
全局请求日志                 -> LoggingInterceptor（仅 debug 模式）
文件上传进度                -> Dio onSendProgress 回调
请求取消                    -> CancelToken（页面销毁时取消）
```

---

## 常见坑

### 1. Riverpod provider 生命周期问题

**坑**：使用 `autoDispose` 的 provider 在页面切换后自动销毁，返回页面时重新加载数据，导致闪烁。

**解法**：
```dart
// 需要跨页面持久化的数据，使用 keepAlive: true
@Riverpod(keepAlive: true)
class CartNotifier extends _$CartNotifier { ... }

// 不需要持久化的数据，使用 autoDispose（默认行为）
@riverpod
class ProductDetailNotifier extends _$ProductDetailNotifier { ... }

// 手动控制：离开页面时不销毁，但数据更新时主动失效
ref.listenSelf((previous, next) {
  if (next.hasError) {
    ref.invalidateSelf(); // 失败时自动重试
  }
});
```

### 2. Dio interceptor 执行顺序

**坑**：interceptor 顺序错误导致 token 刷新后重试请求仍然使用旧 token。

**正确顺序**：
1. **TokenRefreshInterceptor**（第一个）：拦截 401，刷新 token，重试
2. **LoggingInterceptor**（第二个）：日志记录
3. **RetryInterceptor**（第三个）：网络层重试（非 401 的 5xx/超时）

**关键点**：TokenRefreshInterceptor 必须放在 RetryInterceptor 之前，否则 401 会被当作普通网络错误重试 3 次，每次都失败。

### 3. 条件编译正确用法

**坑**：使用 `Platform.isAndroid` 等平台判断在小程序端无效，因为小程序不运行 Dart VM 原生代码。

**正确用法**：
```dart
// 正确：编译时常量，小程序编译时会剔除不匹配的分支
const platform = String.fromEnvironment('TARGET_PLATFORM');
if (platform == 'wechat_mini_program') { ... }

// 错误：运行时判断，小程序端可能报错
if (Platform.isAndroid) { ... }

// 推荐：用 switch 表达式处理所有平台
final result = switch (const String.fromEnvironment('TARGET_PLATFORM')) {
  'wechat_mini_program' => _wechatAction(),
  'alipay_mini_program' => _alipayAction(),
  'douyin_mini_program' => _douyinAction(),
  'android' || 'ios' => _nativeAction(),
  _ => throw UnsupportedError('...'),
};
```

**构建命令**：
```bash
# 每个平台独立编译
flutter build apk --dart-define=TARGET_PLATFORM=android
flutter build ios --dart-define=TARGET_PLATFORM=ios
# 小程序端通过 flutter 小程序框架编译
```

### 4. freezed 联合类型使用

**坑**：支付结果有多种状态（成功、失败、处理中、取消），用 bool/int 表示容易出错。

**正确用法**：
```dart
@freezed
class PaymentResult with _$PaymentResult {
  const factory PaymentResult.success({
    required String transactionId,
    required DateTime paidAt,
  }) = _PaymentSuccess;

  const factory PaymentResult.failed({
    required String errorCode,
    required String errorMessage,
  }) = _PaymentFailed;

  const factory PaymentResult.pending({
    required String orderId,
  }) = _PaymentPending;

  const factory PaymentResult.cancelled() = _PaymentCancelled;
}

// 使用：编译器强制处理所有情况
final message = switch (result) {
  PaymentSuccess() => '支付成功：${result.transactionId}',
  PaymentFailed() => '支付失败：${result.errorMessage}',
  PaymentPending() => '处理中...',
  PaymentCancelled() => '已取消',
};
```

### 5. 金额字段处理

**坑**：JSON 中 `price_cents` 是大整数，`int` 在 Dart 中默认可以处理，但如果 JSON 返回字符串类型的金额会解析失败。

**正确用法**：
```dart
@freezed
class Product with _$Product {
  const factory Product({
    @JsonKey(name: 'price_cents') required int priceCents,
    @JsonKey(name: 'original_price_cents') int? originalPriceCents,
  }) = _Product;

  factory Product.fromJson(Map<String, dynamic> json) => _$ProductFromJson(json);
}

// 展示时统一转换（整数运算，避免浮点数精度问题）
extension PriceFormatting on int {
  String toYuan() {
    final yuan = this ~/ 100;
    final cents = this % 100;
    return '$yuan.${cents.toString().padLeft(2, '0')}';
  }
  String toYuanSymbol() => '¥${toYuan()}';
}

// 使用
Text(product.priceCents.toYuanSymbol())  // ¥99.00
```

---

## 与其他 Skill 的关联

| 引用 | 说明 |
|------|------|
| `storeforge-domain-backend` | 后端 API 契约，Flutter 端类型定义由 `.api` → `goctl api plugin` → OpenAPI spec → `openapi-generator` 生成 Dart 模型，禁止手写 API 类型 |
| `knowledge/ecommerce-patterns/payment-patterns.md` | 小程序支付完整流程（服务端 exchange code、RSA/SHA256 验签、nonce+timestamp 防重放） |
| `knowledge/ecommerce-patterns/cart-patterns.md` | 购物车跨端同步策略（Redis + PG 双写、离线缓存、失效标记、合并冲突解决） |
| `storeforge-harness` | Harness Engineering 约束引擎，自动加载本 skill 的 12 条反模式并执行 BLOCK 检查 |
| `storeforge-executing` | 执行引擎，每个子任务完成后自动触发 harness 约束检查 |
| `storeforge-verification` | 验证体系，harness 检查结果写入验证清单 |
| `storeforge-review` | 多代理 review，architect 和 security-auditor 会引用本 skill 的反模式 |
| `storeforge-testing` | 三层测试体系，Flutter 端要求 `flutter test` 覆盖率 >= 70% |
| `storeforge-domain-backend` | 后端 API 契约，Flutter 端必须遵循 BaseResponse<T> 统一响应格式 |

---

## 检查清单

在标记 Flutter 相关任务完成前，确认以下项：

- [ ] 项目结构符合 feature-first 规范（lib/src/features/、core/、data/、presentation/）
- [ ] 所有状态管理使用 Riverpod（无 Provider/Bloc/GetX）
- [ ] 所有路由跳转使用 go_router（无 Navigator.push）
- [ ] 所有数据模型使用 freezed + json_serializable（无手写 JSON 解析）
- [ ] 所有网络请求使用 Dio + interceptor（无 http 包）
- [ ] 所有网络图片使用 CachedNetworkImage（无 Image.network）
- [ ] 商品详情页包含 SKU 选择、多图轮播、分享
- [ ] 购物车支持离线缓存、跨端同步、失效标记
- [ ] 支付功能使用条件编译区分微信/支付宝/抖音/原生
- [ ] 埋点使用统一 AnalyticsProvider（不分散在页面中）
- [ ] 列表页面使用 ListView.builder / SliverList（无全量渲染）
- [ ] 小程序端无直接调用原生 API（全部条件编译隔离）
- [ ] `flutter test` 覆盖率 >= 70%
- [ ] 平台特定代码使用 `const String.fromEnvironment('TARGET_PLATFORM')`（非 Platform.isXxx）
- [ ] Dio interceptor 顺序正确：token 刷新 -> 日志 -> 重试
