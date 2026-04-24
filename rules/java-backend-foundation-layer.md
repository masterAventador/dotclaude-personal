---
paths:
  - "**/foundation_packages/**/*.java"
---

# Java 后端基础能力层（`foundation_packages/`）架构规则

本文件规定 `foundation_packages/` 下基础能力包的**构成**和**每个包的内部规则**。整体技术规范见 `java-backend.md`。

---

## 1. 基础能力层构成（8 个包）

| 包 | 级别 | 职责 |
|---|------|------|
| `<proj>-common-error` | **强制** | 业务异常 + 错误码基础设施 |
| `<proj>-common-web` | **强制** | HTTP 响应封装 + 全局处理器 |
| `<proj>-common-auth` | **强制** | 鉴权 + 当前用户上下文 |
| `<proj>-common-cache` | **强制** | Redis 封装 |
| `<proj>-common-storage` | **强制** | 对象存储封装 |
| `<proj>-common-util` | **强制** | 通用工具类 |
| `<proj>-common-test` | **强制** | 测试基础设施 |
| `<proj>-common-ws` | 可选 | WebSocket（有需求才建） |

**"强制"的语义**：当项目需要该能力时，**必须单拉一个基础 Maven 模块**，不允许把这类代码塞在业务模块或其他基础模块里。对绝大多数业务后端，前 7 个是默认全要。

---

## 2. 命名与依赖规则（强制）

### 命名

- Maven 子模块：`<proj>-common-<role>`
- Java 包根：`com.<proj>.common.<role>`

### 依赖方向

- 基础能力之间**可互相依赖**（典型：`common-auth` → `common-cache` + `common-error`；`common-web` → `common-error`）
- **避免循环依赖**
- **禁止**基础能力依赖任何业务模块

---

## 3. 各包详细职责

### 3.1 `<proj>-common-error`（强制）

**核心组件**：
- `BusinessException`：全项目唯一业务异常基类
- `ErrCode` 接口：所有错误码实现此接口
- `CommonErrCode` enum：通用错误码（400-499 区间）
- `ErrCodeConflictDetector`：启动时扫描所有 `ErrCode` enum，冲突即失败

详细定义见 `java-backend.md` §5.1-5.4。

### 3.2 `<proj>-common-web`（强制）

**核心组件**：
- `BaseResp<T>`：统一响应体，字段 `success / code / message / data`
- `GlobalExceptionHandler`：`@RestControllerAdvice` 把业务/参数校验/系统异常转成 HTTP 响应
- 全局响应包装器：Controller 返回业务数据时自动包成 `BaseResp<T>`
- 参数校验整合
- 通用拦截器接口（请求日志 / 链路追踪）

`BaseResp` 字段语义见 `java-backend.md` §5.5。

### 3.3 `<proj>-common-auth`（强制）

**核心组件**：
- JWT 生成 / 验证（基于 `jjwt`）
- 密码加密 `BCryptPasswordEncoder`（基于 `spring-security-crypto`）
- 鉴权拦截器：从 HTTP header 读 token → 验证 → 填充当前用户上下文
- `CurrentUser` 工具：从线程上下文（`ThreadLocal`）拿当前登录用户 ID
- `@LoginRequired` 注解：Controller 方法标注，拦截器检测并强制登录

典型使用：

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    @LoginRequired
    @GetMapping("/profile")
    public UserProfile myProfile() {
        Long myId = CurrentUser.id();   // 从上下文拿
        return profileService.load(myId);
    }
}
```

### 3.4 `<proj>-common-cache`（强制）

**核心组件**：
- Redis 封装（基于 Spring Data Redis）
- 常用模式便捷 API：`KvCache` / `HashCache` / `ListCache` / `SortedSetCache`
- 分布式锁 `RedisLock`

**约束**：
- 业务代码**不直接碰** `RedisTemplate`，统一走 common-cache 的封装
- 避免各业务模块各写各的 key 格式，common-cache 可以统一 key 前缀策略

### 3.5 `<proj>-common-storage`（强制）

**核心组件**：
- `ObjectStorage` 接口（抽象）：`upload` / `download` / `presignedUrl` / `delete`
- 具体实现（按项目选）：`CosObjectStorage`（腾讯云）/ `OssObjectStorage`（阿里云）/ `S3ObjectStorage`

**约束**：
- 业务代码只依赖 `ObjectStorage` 接口，不直接引入第三方 SDK
- 未来切换云服务商时只改配置和实现，业务代码零改动

### 3.6 `<proj>-common-util`（强制）

**核心组件**（扁平组织）：
- `DateUtil` / `StringUtil` / `JsonUtil` / `CollectionUtil`
- `CryptoUtil`（AES / RSA 加解密，**不是**密码哈希 —— 密码用 `common-auth` 的 BCrypt）
- 格式化 / 校验工具（手机号、邮箱、身份证等）

**组织**：
- 初始阶段**全部扁平**放在 `src/main/java/com/<proj>/common/util/`
- 同类文件**超过 10 个**时再按子领域拆子包（如 `util/date/` / `util/crypto/`）
- **不提前**建空文件夹

### 3.7 `<proj>-common-test`（强制）

**核心组件**：
- `BaseIntegrationTest`：封装 `@SpringBootTest` 常用配置 + H2 自动切换
- ArchUnit 基类测试：扫描所有模块的依赖图，守护业务模块间互不依赖
- MockMvc 辅助工具
- Fixture 通用父类（可选）

**依赖范围**：仅 `test` scope，业务模块的 test scope 依赖本模块。

### 3.8 `<proj>-common-ws`（可选，按需建）

**核心组件**（仅有 WebSocket 需求的项目建）：
- WebSocket 连接管理（session ↔ userId 映射，通常结合 `common-cache` 存 Redis）
- 在线状态查询
- 推送封装：`pushTo(userId, msg)` / `pushToConversation(convId, msg)`
- 心跳 / 断线重连服务端支持

---

## 4. 内部结构规则

### 扁平优先

```
<proj>-common-<role>/
└── src/main/java/com/<proj>/common/<role>/
    ├── XxxClass1.java
    ├── XxxClass2.java
    └── ...
```

- **初始阶段扁平**组织，文件直接放 `src/main/java/com/<proj>/common/<role>/`
- 同一基础能力下文件**超过 10 个**时再按子领域拆子包
- **不提前**建空目录

---

## 5. 边界约束（强制）

- 基础能力**只提供通用抽象与工具**，**禁止**包含任何业务领域知识
- 基础能力之间**可互相依赖**（但避免循环）
- **禁止**基础能力依赖任何业务模块（`business_packages/*`）
- 禁止在基础能力里引用业务模型（如 `UserEntity` / `OrderEntity`）

---

## 6. 不建议单独建的基础能力

| 能力 | 理由 |
|------|------|
| `common-event` | Spring 自带 `ApplicationEventPublisher` + `@TransactionalEventListener` 已够用 |
| `common-scheduler` | `@Scheduled` 内置够用，除非有复杂调度需求 |
| `common-mq` | 当前项目无需 MQ 时不提前引入；等真需要 Kafka/RabbitMQ 再加 |
