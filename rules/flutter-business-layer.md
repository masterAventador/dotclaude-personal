---
paths:
  - "**/business_packages/**/*.dart"
---

# Flutter 业务层（`business_packages/`）架构规则

本文件规定 Flutter 项目**业务层**内每个业务模块的内部结构和约束。整体目录树和技术规范见 `~/.claude/rules/flutter.md`。

---

## 1. 业务模块组织原则

- 业务层按 app 的 **tab 划分模块**，1 个 tab = 1 个业务模块
- 非 tab 的业务（如登录/注册 `<proj>_auth`）也作为独立业务模块，走同样结构
- 每个模块是独立的 Flutter package，**通过 `pubspec.yaml` 声明依赖**

---

## 2. 模块目录结构（强制）

每个业务模块必须包含 3 个顶层目录：

```
<proj>_<module>/
├── assets/
│   └── images/                # 后续可扩展其他资源类型
├── lib/
│   ├── <proj>_<module>.dart   # 门面（barrel）
│   ├── <proj>_<module>_routes.dart  # 路由注册
│   └── src/                   # 所有实现代码
└── test/                      # 单元测试
```

---

## 3. 模块门面 `<proj>_<module>.dart`（强制）

**作用**：模块对外的唯一入口，外层壳工程和其他模块只通过它引用本模块。

**内容约束**：**只允许 export 两样东西**，不写任何实现代码：

```dart
// <proj>_home.dart
export 'src/home/home_page.dart' show HomePage;  // tab 首页（给外层 home_page.dart 集成）
export '<proj>_home_routes.dart';                 // 本模块路由类
```

`src/` 下的 page / controller / api 等**不得**被外层 import，严格隐藏。

---

## 4. `<proj>_<module>_routes.dart` 独立文件（强制）

**独立文件**，不和门面合并。**唯一内容**是一个类，类里只有一个**静态方法 `routes()`** 返回 `List<GetPage>`：

```dart
// <proj>_home_routes.dart
import 'package:get/get.dart';
import 'src/feature_a/feature_a_page.dart';
import 'src/feature_b/feature_b_page.dart';
import 'package:<proj>_routes/<proj>_routes.dart';

class <Proj>HomeRoutes {
  static List<GetPage> routes() => [
        GetPage(name: <Proj>Routes.featureA, page: () => FeatureAPage()),
        GetPage(name: <Proj>Routes.featureB, page: () => FeatureBPage()),
      ];
}
```

外层壳工程的 `GetMaterialApp` 聚合所有业务模块的 `routes()` 完成全局路由注册。

---

## 5. `lib/src/` 内部组织（强制）

```
src/
├── api/                       # 本模块网络层
│   ├── <proj>_<module>_api.dart
│   ├── req/
│   │   └── *_req.dart         # 每个文件：XxxReq + XxxData
│   └── models/                # 多接口共用的领域模型
├── feature_a/                 # 每个页面一个文件夹
│   ├── feature_a_page.dart
│   ├── feature_a_controller.dart
│   └── widgets/               # 本页面私有小组件
├── feature_b/
│   └── ...
└── widgets/                   # 本模块多页面共用组件
```

---

## 6. api 封装规则（强制）

### `<proj>_<module>_api.dart`（聚合入口）
- 模块内**所有网络请求**必须通过它间接调用 `<proj>_network`，**禁止**页面/controller 直接调用 `<proj>_network` 的 http 方法
- 负责**数据清洗**：把服务端返回的原始数据处理成业务需要的格式

### `api/req/` 目录
- 每个接口一个 `*_req.dart` 文件
- 文件内放两样东西：`XxxReq` 请求类 + `XxxData` 单接口专用响应数据类
- 命名规范见 `~/.claude/rules/flutter.md` 第 12 节

### `api/models/`
- 仅放**跨接口复用**的领域模型（如 `Message` 被多个接口/WebSocket 事件共用）
- 单接口专用数据类**不进** models/，留在 `_req.dart` 里

---

## 7. 页面必须有 Controller 配对（强制）

- 每个页面文件夹**必须**包含 `xxx_page.dart` + `xxx_controller.dart` 两个文件
- 即使页面没有任何逻辑，也**必须创建 controller 空类**，不得省略
- Controller 属性名、初始化模板见 `~/.claude/rules/flutter.md` 第 4~5 节

---

## 8. 业务模块间依赖（强制）

- `business_packages` 之间**绝对禁止**互相引用（pubspec 依赖 + import 都禁止）
- 业务模块间需要共享代码时，**下沉到 `foundation_packages`**
- 业务模块可以依赖 `foundation_packages` 下任意基础包

---

## 9. 跨模块跳转走路由（强制）

- 业务模块之间跳转**必须**用 `Get.toNamed(<Proj>Routes.xxx)`，路由名从 `<proj>_routes` 取
- 本模块内部跳转可以直接 `Get.to(() => XxxPage())`（不走路由名）
- bizkit 页面调用**不走路由**，通过 `Get.to(() => XxxPage(...))` 直接传参（bizkit 的 barrel export 页面类可直接 import，见基础层规则）

---

## 10. 测试聚合命名（强制）

- 每个业务包的 `test/` 目录下**必须有一个** `<proj>_<module>_suite.dart` 文件
- 该文件聚合本模块所有测试用例，供顶层 E2E 集成测试调用
- 其他单元测试文件命名 `*_test.dart`
