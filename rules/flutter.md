---
paths:
  - "**/*.dart"
  - "**/pubspec.yaml"
---

# Flutter + GetX 通用技术规范

本文件规定所有 Flutter 项目的**整体结构**、**技术选型**和**编码规范**。创建新项目、新模块、新文件必须严格遵循本文件约束，不得临时发挥。

遇到本规范覆盖不到的新场景时，**先停下来与用户讨论迭代规则本身**，再落地到具体项目，禁止在项目里临时发挥。

---

## 1. 项目整体结构（强制）

所有 Flutter 项目统一使用两层结构：**业务层 `business_packages/` + 基础层 `foundation_packages/`**。

### 完整目录树

```
<project_root>/                              # Flutter 项目根目录
├── lib/
│   ├── main.dart                            # 启动入口（首行 await GetStorage.init()）
│   └── home_page.dart                       # 集成各业务模块暴露的 tab 首页
│
├── business_packages/                       # ── 业务层（按 app tab 划分模块）
│   ├── <proj>_home/                         # tab 1 业务模块（示例）
│   │   ├── assets/
│   │   │   └── images/                      # 后续可扩展其他资源类型
│   │   ├── lib/
│   │   │   ├── <proj>_home.dart             # 门面 barrel：只 export HomePage + <Proj>HomeRoutes
│   │   │   ├── <proj>_home_routes.dart      # 独立文件，唯一静态方法 routes() 返回 List<GetPage>
│   │   │   └── src/
│   │   │       ├── api/
│   │   │       │   ├── <proj>_home_api.dart # 聚合入口，所有网络请求走这里
│   │   │       │   ├── req/
│   │   │       │   │   └── *_req.dart       # 每个文件含 XxxReq + XxxData
│   │   │       │   └── models/              # 多接口共用的领域模型
│   │   │       ├── feature_a/               # 页面 A（page + controller 必须配对）
│   │   │       │   ├── feature_a_page.dart
│   │   │       │   ├── feature_a_controller.dart
│   │   │       │   └── widgets/             # 本页面私有小组件
│   │   │       ├── feature_b/
│   │   │       │   ├── feature_b_page.dart
│   │   │       │   ├── feature_b_controller.dart
│   │   │       │   └── widgets/
│   │   │       └── widgets/                 # 本模块多页面共用组件
│   │   └── test/
│   │       ├── <proj>_home_suite.dart       # 测试聚合入口（供 E2E 集成用）
│   │       ├── feature_a_controller_test.dart
│   │       └── <proj>_home_api_test.dart
│   ├── <proj>_message/                      # tab 2 业务模块（同上结构）
│   ├── <proj>_profile/                      # tab 3 业务模块（同上结构）
│   └── <proj>_auth/                         # 非 tab 业务模块（登录/注册等），走同结构
│
└── foundation_packages/                     # ── 基础层（6 个固定包）
    ├── <proj>_uikit/                        # 公共 UI 组件（Button/Input/Theme/Color）
    │   └── lib/src/                         # 初始扁平，同类文件超 2~3 个再拆子目录
    │
    ├── <proj>_network/                      # 网络库
    │   └── lib/src/
    │       ├── http_client.dart             # get/post/put/delete 封装
    │       ├── base_req.dart                # 业务层请求基类
    │       └── base_resp.dart               # 统一应答，data 走泛型
    │
    ├── <proj>_routes/                       # 路由名称集中表
    │   └── lib/<proj>_routes.dart           # 全是 static const String
    │
    ├── <proj>_user/                         # 用户中心（UserCenter）
    │   └── lib/src/
    │       ├── api/
    │       │   ├── <proj>_user_api.dart
    │       │   ├── req/
    │       │   └── models/
    │       ├── models/                      # User / Profile / Setting
    │       ├── local/
    │       │   └── user_cache.dart          # 封装 GetStorage
    │       └── user_center.dart             # 内存态 + 本地缓存 + 冷启动同步读
    │
    ├── <proj>_bizkit/                       # 公共业务组件（结构对齐业务模块但无 routes）
    │   ├── assets/images/
    │   └── lib/
    │       ├── <proj>_bizkit.dart           # barrel：直接 export 所有 page 类供业务模块调用
    │       └── src/
    │           ├── api/                     # 如有跨业务通用接口
    │           ├── feature_a/               # 每个公共业务页面
    │           │   ├── feature_a_page.dart
    │           │   ├── feature_a_controller.dart
    │           │   └── widgets/
    │           └── widgets/                 # 本包内跨页面共用
    │
    └── <proj>_util/                         # 工具类
        └── lib/src/                         # 初始扁平
            ├── date_util.dart
            ├── string_util.dart
            ├── crypto_util.dart
            └── ...
```

### 结构不变性声明

- 日常开发、新建模块、新建项目，**严格按**此目录树组织代码
- 遇到此结构覆盖不到的新场景时，**停下来与用户讨论迭代规则本身**
- 业务层细则：`~/.claude/rules/flutter-business-layer.md`
- 基础层细则：`~/.claude/rules/flutter-foundation-layer.md`

---

## 2. 包命名规范（强制）

- **所有包强制使用统一的项目短名前缀** `<proj>_xxx`
- 前缀具体值由项目决定，但同一项目内**所有包必须统一前缀**
- 业务包：`<proj>_<tab_name>`（如 `<proj>_home`、`<proj>_message`、`<proj>_profile`、`<proj>_auth`）
- 基础包：`<proj>_<role>`（`<proj>_uikit` / `<proj>_network` / `<proj>_routes` / `<proj>_user` / `<proj>_bizkit` / `<proj>_util` 共 6 个，**固定**）

---

## 3. Widget 类型选择

优先使用 **StatelessWidget**，用 GetX 管理状态。

**必须使用 StatefulWidget 的场景：**
- 需要 `TickerProvider`（如 `TabController`、`AnimationController`）
- 需要管理 `TextEditingController` / `ScrollController` 等且不适合放在 Controller 中

---

## 4. Page 与 Controller 职责分离（强制）

- **Page**：**只写页面布局代码**，绑定 UI 事件（通过回调调用 controller 方法）
- **Controller**：数据管理、网络请求、页面跳转、所有业务逻辑
- **Page 类绝对禁止 `setState`**，也禁止写任何业务逻辑（包括简单跳转）
- 即使页面没有任何逻辑，**也必须创建对应 Controller 类**（空类也要写）

---

## 5. Controller 初始化三场景模板（强制）

所有 page 的 controller 属性**固定命名为 `c`**，`Get.put` **必须**在属性声明处或构造函数初始化列表完成。

**绝对禁止：**
- ❌ `late final c = Get.put(...)`（惰性初始化，controller 的 `onInit` 不会在页面打开时立即触发）
- ❌ 在 `build()` 方法里 `Get.put`

### 场景 ① 无参 controller

```dart
class ConversationListPage extends StatelessWidget {
  ConversationListPage({super.key});
  final c = Get.put(ConversationListController());

  @override
  Widget build(BuildContext context) { ... }
}
```

### 场景 ② 有参 controller（业务模块页面）

用构造函数**初始化列表**初始化 `c`：

```dart
class ChatDetailPage extends StatelessWidget {
  final String conversationId;
  final ChatDetailController c;

  ChatDetailPage({super.key, required this.conversationId})
      : c = Get.put(ChatDetailController(conversationId: conversationId));

  @override
  Widget build(BuildContext context) { ... }
}
```

### 场景 ③ bizkit 页面（有参 + 强制 `tag`）

bizkit 页面可能被不同业务并发打开，强制带 `tag: UniqueKey().toString()` 避免 controller 实例互相替换：

```dart
class NameSearchPage extends StatelessWidget {
  final String initialKeyword;
  final NameSearchController c;

  NameSearchPage({super.key, required this.initialKeyword})
      : c = Get.put(
          NameSearchController(initialKeyword: initialKeyword),
          tag: UniqueKey().toString(),
        );

  @override
  Widget build(BuildContext context) { ... }
}
```

`UniqueKey().toString()` 一次性字符串不需保存 —— 路由 pop 时 GetX 自动回收 controller。

---

## 6. Widget 交互事件处理

Widget 中的交互事件（点击、长按等）**不要直接处理业务逻辑**，通过**回调函数向上透传**给 Controller 处理：

```
Widget 接收回调参数 → 事件触发时调用回调 → Controller 处理业务逻辑
```

---

## 7. 页面参数传递

**创建时赋值**，不要在 `onInit` 里从 `Get.arguments` / `Get.parameters` 获取。

- ✅ 正确：调用方创建 Page 时直接传参 → Page 构造函数接收 → 传给 Controller 构造函数
- ❌ 错误：`Get.to()` 传 arguments → Controller 在 `onInit` 里从 `Get.arguments` 读取
- **唯一例外**：路由声明里的 `GetPage`（没有创建入口可以直接赋值，只能从 `Get.arguments` / `Get.parameters` 取）

---

## 8. 页面加载状态

- 首次加载显示 Loading 指示器，**刷新不显示**
- 判断条件：`if (c.isLoading.value && c.dataList.isEmpty)` 才显示 Loading
- `isLoading` 初始化为 `true.obs`，首次加载完成后设为 `false`
- 下拉刷新**不**修改 `isLoading`

---

## 9. 刷新粒度分层（强制）

| 场景 | 方式 | 说明 |
|------|------|------|
| 单变量刷新 | `Obx` 包裹 `.obs` | **首选**，细粒度自动刷新 |
| 大块内容刷新 | `GetBuilder` + controller 调 `update()` | 手动触发整块刷新 |
| `setState` | 仅限**独立 widget 小组件 + 极简**场景 | **page 类绝对禁用** |

---

## 10. 本地存储（强制）

**统一使用 `GetStorage`，禁止 `SharedPreferences`。**

- `main()` 首行必须 `await GetStorage.init();` 才能 `runApp`
- 按模块拆 container 避免 key 撞车：`GetStorage('user')` / `GetStorage('settings')` / `GetStorage()`（默认）
- 同一模块内部的 key 用字符串常量集中管理，禁止硬编码

---

## 11. 网络请求

### 响应处理
- 使用 `.then()` 处理响应，**不要用 try-catch**
- `resp.success` 为 false 时调用 `Toast.show(resp.message)`

### 请求参数
- 所有请求类型（GET/POST/PUT/DELETE）统一用 `parameters` 方法返回参数
- **禁止**使用 `toJson`（`toJson` 是响应数据类用的）

---

## 12. 接口请求与应答命名

### 请求类命名
- `XxxReq`（**不使用** `XxxResp` 这种命名习惯）

### 响应数据类分两类处理

**单接口专用响应**（该数据结构只被这一个接口使用）：
- 命名 `XxxData`
- **必须与 `XxxReq` 放在同一个文件**

**跨接口/跨模块复用的领域实体**（如 `User` / `Message` / `Conversation`，会被多个接口返回或被 WebSocket 事件等非 HTTP 路径使用）：
- 放独立的 `models/` 目录
- 保留业务命名，**不强制加 `Data` 后缀**

### 示例
- 单接口专用：`check_phone_exist_req.dart` 包含 `CheckPhoneExistReq` 和 `CheckPhoneExistData`
- 领域实体：`api/models/message.dart` 定义 `Message`（被 `ListMessagesReq` 返回，也被 WebSocket 推送事件复用）

---

## 13. 资源文件管理

### 添加图片
- 只需要 2.0x 和 3.0x 版本（**不需要** 1x）
- 放到对应目录后**必须在 pubspec.yaml 中单独声明该图片**（如 `- assets/images/new_image.png`）
- 执行 `flutter pub get`

### 原生迁移规范
- 迁移原生已有功能到 Flutter 时，**必须使用与原生一致的图片资源**
- 从原生项目中找到对应图片，把 2.0x 和 3.0x 复制到 Flutter 项目
- 使用 `Image.asset` 加载，**不要用 `Icon` widget** 或 Material 图标替代

---

## 14. Flutter 测试栈

- **单元测试**：`package:test` + `flutter_test`（对工具函数、Controller 逻辑、API 数据清洗编写单元测试）
- **Widget 测试**：`flutter_test` 的 `testWidgets`
- **Mock**：`mocktail`（不推荐 `mockito`，注解生成太重）
- **E2E / 集成测试**：`integration_test`（官方包）
- **TDD 流程**：有业务逻辑的代码必须先写失败测试 → 写最小实现 → 测试通过
- **测试聚合入口**：每个业务包的 `test/<pkg_name>_suite.dart` 聚合所有测试，供顶层 E2E 调用
