---
paths:
  - "**/business_packages/**/*.java"
---

# Java 后端业务层（`business_packages/`）架构规则

本文件规定 `business_packages/` 下每个业务领域模块的**内部结构**和**约束**。整体技术规范见 `java-backend.md`。

---

## 1. 业务模块组织原则

- 按 **DDD Bounded Context** 划分业务领域（**不对齐** app 端的 tab —— 组织维度不同）
- 每个业务领域 = **2 个 Maven 子模块**：`<proj>-<domain>-api` + `<proj>-<domain>-core`
- 业务模块之间**禁止直接 import 彼此的 `-core`**（Maven Enforcer + ArchUnit 守护）
- 业务模块可依赖**其他业务的 `-api`** 和**任意基础模块**

---

## 2. `-api` 模块（对内契约）

**作用**：模块对**后端内部**其他模块暴露的契约。**只含接口 + DTO + 事件定义**，不含任何实现。

### 2.1 目录结构

```
<proj>-<domain>-api/
└── src/main/java/com/<proj>/<domain>/
    ├── <Domain>Api.java              # 唯一主接口，放顶层
    ├── dto/
    │   ├── <Domain>Dto.java
    │   └── <Domain>SummaryDto.java
    └── event/
        ├── <Domain><Action1>Event.java     # 领域事件（record）
        └── <Domain><Action2>Event.java
```

### 2.2 核心约束（强制）

- **一个 `-api` 模块只有一个主 `<Domain>Api` 接口**。接口职责太多说明领域划分过粗，应拆领域
- 接口放**顶层包**（不进 `dto/` 或其他子包）
- DTO 放 `dto/` 子包，命名 `<Domain>Dto` / `<Domain><描述>Dto`（业务描述名）
- 领域事件放 `event/` 子包，**必须是 Java 21 `record`**，命名 `<Domain><动作过去式>Event`
- **禁止**在 `-api` 写任何实现代码或业务逻辑
- `-api` **不依赖**任何其他业务/基础模块（纯契约，只依赖 JDK）

### 2.3 示例

```java
// <proj>-user-api/src/main/java/com/<proj>/user/UserApi.java
package com.<proj>.user;

import com.<proj>.user.dto.UserDto;

public interface UserApi {
    UserDto findById(Long userId);
    boolean existsByPhone(String phone);
}

// <proj>-user-api/src/main/java/com/<proj>/user/dto/UserDto.java
public record UserDto(Long id, String phone, String nickname, String avatarUrl) {}

// <proj>-user-api/src/main/java/com/<proj>/user/event/UserRegisteredEvent.java
public record UserRegisteredEvent(Long userId, String phone, Instant registeredAt) {}
```

---

## 3. `-core` 模块（实现）

**作用**：业务逻辑实现。对外提供**两个入口**：
- **HTTP 入口**（给 app/web/第三方）：`web/` 子包下的 Controller
- **Java 接口入口**（给后端内部模块）：`api/` 子包下的 `XxxApiImpl`

### 3.1 目录结构（强制）

```
<proj>-<domain>-core/
└── src/main/java/com/<proj>/<domain>/
    ├── web/                          # HTTP 对外层
    │   ├── <Domain>Controller.java
    │   └── *Req.java                 # HTTP 请求/响应 DTO（同文件）
    ├── internal/                     # 对内实现层（业务逻辑核心）
    │   ├── <Domain><Action>Service.java   # 按动作拆 Service
    │   ├── <Domain>Repository.java
    │   ├── <Domain>Entity.java            # 带 Entity 后缀
    │   └── <Domain>ErrCode.java
    ├── api/                          # 对内 Java 接口实现
    │   └── <Domain>ApiImpl.java
    └── event/                        # 事件监听器
        └── <EventType>Listener.java
```

### 3.2 `web/` 子包

放**外部流量入口**：Controller + HTTP 请求/响应 DTO。

- **Controller 命名**：`<Domain>Controller`
- **Controller 只做三件事**：接请求 → 调 Service → 返回响应。**禁止**在 Controller 写业务逻辑
- **HTTP 请求 DTO**：`<Action>Req`
- **HTTP 响应 DTO**（分两类）：
  - **单接口专用响应**：`<Action>Data`，**必须与 `<Action>Req` 放在同一个文件**
  - **跨接口复用的领域模型**：业务描述名（如 `UserProfile`），不加 `Data` 后缀

### 3.3 `internal/` 子包

放**对内实现细节**。

**禁止**其他模块直接 import `internal/` 下的任何类。

- **Service 按动作拆**：命名 `<Domain><Action>Service`（名词开头）
  - 示例：`UserRegisterService` / `UserLoginService` / `UserProfileUpdateService`
  - **一个 Service 方法 = 一个 `@Transactional` 边界**
  - 跨 Service 的共享查询逻辑**下沉到 Repository**（`findByXxx` / `existsByXxx` 方法），**不要**再抽 `UserFinderService` 这种查询专用 Service
- **Repository**：`<Domain>Repository`，继承 `JpaRepository<Entity, Id>`
- **Entity**：`<Domain>Entity`（**带 `Entity` 后缀**，和同领域的 `<Domain>Dto` / HTTP 响应模型区分）
- **领域逻辑**：简单规则直接在 Service 方法里 `if` + 抛 `BusinessException`；复杂规则抽独立类（如 `XxxPolicy`），仍放 `internal/`
- **错误码**：`<Domain>ErrCode` enum，放 `internal/`

### 3.4 `api/` 子包

放**对内 Java 接口实现**：`<Domain>ApiImpl implements <Domain>Api`。

- 通常**薄**，委托给 `internal/` 下的 Service
- 同一个 Service **既被 Controller 调用**（HTTP 入口）**也被 ApiImpl 调用**（后端内部入口）—— 业务逻辑只写一次

### 3.5 `event/` 子包

放**事件监听器（Listener）**。Listener 可以订阅：

- **本模块自己发的事件**（如注册后本模块订阅做后续响应）
- **其他模块发的事件**（如订阅 `user` 模块的事件做扩散响应）

**事件定义**一律放在**发布方**的 `-api/event/` 里；**Listener** 放在**订阅方**的 `-core/event/` 下。

- 命名：`<EventType>Listener`（例：`UserRegisteredListener`）
- 同一事件多个 Listener（按关注点拆分）可加后缀：`UserRegisteredWelcomeMailListener`
- **默认**：`@Async @TransactionalEventListener(phase = AFTER_COMMIT)`（事务提交后异步触发，避免状态不一致）
- 同步订阅例外用普通 `@EventListener`，**代码注释说明理由**

### 3.6 完整示例（-core）

```java
// internal/UserEntity.java
@Entity
@Table(name = "user")
@Data
public class UserEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String phone;
    private String nickname;
    private String avatarUrl;
    private String passwordHash;
    private Instant createdAt;
}

// internal/UserRepository.java
public interface UserRepository extends JpaRepository<UserEntity, Long> {
    boolean existsByPhone(String phone);
}

// internal/UserRegisterService.java
@Service
@RequiredArgsConstructor
public class UserRegisterService {
    private final UserRepository repo;
    private final BCryptPasswordEncoder passwordEncoder;
    private final ApplicationEventPublisher events;

    @Transactional
    public Long register(String phone, String password, String nickname) {
        if (repo.existsByPhone(phone)) {
            throw new BusinessException(UserErrCode.PHONE_ALREADY_REGISTERED);
        }
        UserEntity user = new UserEntity();
        user.setPhone(phone);
        user.setNickname(nickname);
        user.setPasswordHash(passwordEncoder.encode(password));
        user.setCreatedAt(Instant.now());
        repo.save(user);
        events.publishEvent(new UserRegisteredEvent(user.getId(), phone, user.getCreatedAt()));
        return user.getId();
    }
}

// api/UserApiImpl.java
@Service
@RequiredArgsConstructor
public class UserApiImpl implements UserApi {
    private final UserRepository repo;

    @Override
    public UserDto findById(Long userId) {
        return repo.findById(userId)
                .map(e -> new UserDto(e.getId(), e.getPhone(), e.getNickname(), e.getAvatarUrl()))
                .orElse(null);
    }

    @Override
    public boolean existsByPhone(String phone) {
        return repo.existsByPhone(phone);
    }
}
```

---

## 4. 跨模块调用

### 4.1 同步查询：`@Autowired <Domain>Api`

```java
@Service
@RequiredArgsConstructor
public class MessageSendService {
    private final RelationApi relationApi;   // 来自 <proj>-relation-api
    private final UserApi userApi;           // 来自 <proj>-user-api

    public void send(Long fromId, Long toId, String content) {
        if (!relationApi.isFriend(fromId, toId)) {
            throw new BusinessException(MessageErrCode.NOT_FRIEND);
        }
        // ...
    }
}
```

依赖声明：`message-core/pom.xml` 里声明 `<dependency>relation-api</dependency>` + `<dependency>user-api</dependency>`，**不依赖** `relation-core` / `user-core`。

### 4.2 通知扩散：领域事件

发布方（如 `user-core`）的 Service 发事件；订阅方（如 `notification-core`）的 Listener 接收。订阅方只需依赖发布方的 `-api` 模块即可看到事件类。

---

## 5. 测试组织

### 5.1 目录结构

```
<proj>-<domain>-core/
└── src/test/java/com/<proj>/<domain>/        # 镜像 src/main
    ├── web/
    │   └── <Domain>ControllerIT.java         # @WebMvcTest
    ├── internal/
    │   ├── <Domain><Action>ServiceTest.java  # Mockito 单测
    │   └── <Domain>RepositoryIT.java         # @DataJpaTest（如有自定义查询）
    ├── api/
    │   └── <Domain>ApiImplTest.java
    ├── event/
    │   └── <EventType>ListenerTest.java
    └── fixtures/
        └── <Domain>TestFixtures.java         # Object Mother（强制）
```

### 5.2 Object Mother 强制（强制）

**Entity 和领域对象在测试中使用时，必须通过对应的 `<Domain>TestFixtures` 类构造**。简单 DTO 不强制。

**最低要求**：每个 Fixture 类至少提供一个"默认有效对象"方法：

```java
public class UserTestFixtures {
    public static UserEntity validUser() {
        UserEntity u = new UserEntity();
        u.setPhone("13800138000");
        u.setNickname("测试用户");
        u.setCreatedAt(Instant.now());
        return u;
    }

    public static UserEntity userWithPhone(String phone) {
        UserEntity u = validUser();
        u.setPhone(phone);
        return u;
    }
}
```

### 5.3 覆盖要求

- Service 业务逻辑 → Mockito 单测**必须有**（遵守 TDD 铁律）
- Controller → `@WebMvcTest` **必须有**
- Repository 自定义查询 → `@DataJpaTest` **必须有**
- 领域事件 Listener → Mockito 单测**必须有**

### 5.4 不需要测试聚合入口

Maven surefire/failsafe 自动发现 `*Test` / `*IT`，**不像** Flutter 需要手动写 `_suite.dart`。
