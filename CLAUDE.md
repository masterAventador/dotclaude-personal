# 全局代码规范

## 规则管理说明

当用户说"把XXX添加到你的全局规则里面去"或类似的表述时，应该将相关规则添加到本文件 (`~/.claude/CLAUDE.md`) 中。这个文件是全局指令文件，用于存储所有跨项目的通用规则和规范。

## 新建项目前置加载规则

**核心规则:** 用户要求"新建 / 创建 / 搭建 <技术栈> 项目"时，**动手前**：

1. 先 `ls ~/.claude/rules/` 列目录
2. 找出文件名包含该技术栈关键词的所有分片（如 Flutter → `flutter*.md` 三分片；Java 后端 → `java-backend*.md` 三分片）
3. **全部主动 Read 加载到 context**
4. 再开始搭建

**原因:** 分片规则靠 `paths` frontmatter 按 Read 触发。空项目尚无文件可 Read，若不主动预加载会按默认结构乱搭。

**适用范围:** 任何技术栈（Flutter / Java 后端 / Node / Python 等），只要 `~/.claude/rules/` 下存在对应分片就必须预加载。

## 代码删除规范

**核心规则:**
- 去掉功能时，**直接删除代码**，而不是注释掉
- 删除后如果函数为空，连函数一起删除
- **排查并删除该功能相关的所有代码**：属性、网络接口、辅助方法、模型类等
- 除非用户明确说"别删除"，否则默认就是删除

**删除流程:**
1. 删除主功能代码和调用
2. 使用 Grep 搜索相关属性/方法的使用情况
3. 确认没有其他地方使用后，删除这些属性和方法
4. 删除无用的网络接口方法（包括 .h 和 .m 文件）

**排查清单:** 方法定义和声明、调用处、专用属性、网络接口、辅助方法、常量枚举、通知名称、UI控件

## 重构后清理规范

**核心规则:**
- 重构完代码后，**必须排查并删除废弃的属性、变量、逻辑**
- 这是重构工作的一部分，不是可选步骤

**排查清单:** 旧属性和变量、旧方法和函数、旧常量和枚举、旧配置项（URL、host等）、旧辅助类、setter/getter中被替换的逻辑

**操作流程:**
1. 完成重构的主要改动
2. 使用 Grep 搜索被替换的旧属性/方法名
3. 确认没有其他使用后，删除所有废弃代码
4. 确保代码能正常编译

## Bug 修复最小改动规范

**核心规则:** 修复 bug 时，最终提交的代码必须是**最小必要改动**，不允许夹带修复过程中的无效尝试代码。

**强制流程:**
1. Bug 修复完成后，**立即排查所有改动**（`git diff`）
2. 逐行检查：每一行改动是否是修复此 bug 的**必要代码**
3. 发现无效改动（之前尝试失败的代码、调试代码、多余的样式覆盖等），**立即回滚**
4. 回滚完成后，**再次验证** bug 确实已修复，确保没有误删必要代码
5. 确认无误后才能提交

**绝对禁止:**
- ❌ 把多次尝试的中间代码全部保留
- ❌ "反正能用就行"不清理无效代码
- ❌ 回滚后不验证就提交

## 代码复用原则

**核心规则（逻辑复用）:** 多个地方需要执行相同逻辑时，必须抽取成独立的可复用函数（如 `handleXxxResult`），在所有需要的地方调用该函数。

**核心规则（值复用）:** 同一个字面量值（字符串、魔法数字、URL、配置常量、枚举值、正则等）在多处出现或在多处需要保持一致时，必须抽取成命名常量（`static final`、`const`、enum 等），所有用到的地方都引用这个常量，禁止硬编码字面量做比较或赋值。

**典型反例（必须重构）:**
- ❌ A 处 `return "错误兜底文案"`，B 处 `if ("错误兜底文案".equals(result))` —— 改 A 处文案 B 处不会跟着变，定时炸弹
- ❌ 多个文件里都写 `"https://api.example.com"`，改 URL 要 grep 全项目
- ❌ 业务代码里到处出现 `60 * 60 * 24` 这种魔法数字
- ❌ 状态判断 `if (status.equals("PENDING"))` 字面量散落在多处

**正确做法:** 抽常量 → 单点定义 → 多处引用。命名要表达含义而不是值本身（`MAX_RETRY_COUNT` 而不是 `THREE`）。

## 写代码前必须先查现有可复用资源（强制）

**适用范围:** 所有技术栈（Flutter / Java / Vue / React / 小程序 / Node / Python 等）。**业务逻辑本身**（某个特定动作 / 流程的实现）不在此规则范围（它天然是新建的），但实现业务逻辑过程中**用到的所有非业务零件**都受此规则约束。

**核心规则:** 写**任何非业务逻辑代码**之前（视图 / UI 元素 / 工具函数 / 公共方法 / 常量 / 配置 / 类型 / 异常类 / 网络封装 / 缓存封装 / 校验器 / 格式化器 / mixin / hook / decorator / 中间件 等等），**必须先在项目里搜一遍是否已有可复用的实现**。严格走下面三步：

**第 1 步：搜索现有实现**
- 先看**基础层 / 公共包**：项目通常有"基础设施层"（叫法各不同：`foundation_packages/` / `common/` / `shared/` / `utils/` / `lib/`）—— 优先在这里找
- 再看**当前模块的公共区**：本模块的 `widgets/` / `helpers/` / `utils/` / `internal/`
- 用 grep 关键词：按"功能名"搜（如要写"日期格式化" → 搜 `format`、`date`、`Util`），按"类型名"搜（如想找按钮组件 → 搜 `Button`），按"动作动词"搜（如要发请求 → 搜 `request`、`fetch`、`http`）
- 跨语言通用：搜函数名 / 类名 / 文件名 / 注释关键词

**第 2 步：判断如何复用**
- ✅ **直接用**：现有实现完全满足需求，import / require / 注入 后直接调用
- ✅ **改造后用**：现有实现接近但参数 / 返回值 / 行为略有差异，**优先改造现有的**（加可选参数、加新方法、把硬编码部分参数化、提取公共部分让两种用法共享）—— **不要新建一个名字差不多的与之并存**
- ❌ **新建**：以上都不行才新建。新建时要在 commit message 或代码注释里讲清楚"为什么不能复用现有的 X"

**第 3 步：新建时放对位置**
- 跨模块复用 → 沉到基础层 / 公共包
- 单模块多处复用 → 本模块的公共区
- 单一调用点私有 → 就近放，不要提前抽公共
- **核心原则**：等真出现"第二个调用方"再下沉，不要预测性抽公共

**绝对禁止:**
- ❌ "我先快速写一个，后面再统一" —— 后面永远不会统一，反而留下两份重复代码
- ❌ "这个和现有的 X 差不多但有点区别，所以我新建一个 Y" —— 应该改造 X 让它能覆盖两种场景
- ❌ "我没看见有现成的"（**没有先搜过就不算"没看见"**）—— 必须 grep / 列目录 / 读基础层包结构后才能下结论
- ❌ 新建一个名字与现有几乎相同的（如 `formatDate2` / `Toast2` / `HttpClient2` / `useUser2`）—— 这是规避复用的典型信号

**典型反例（多技术栈通用）:**
- 同一个常量值散落在 N 个文件各自定义 → 应该集中到一个 Constants
- 两个组件 / 两个类有大段相同逻辑 → 应该抽 base class / mixin / hook / 工具函数
- 项目里已经有一个 `Toast` / `Notification` / `Snackbar` 封装，业务代码却又自己直接调底层 API 包了一层 → 应该用项目封装
- 项目里已经有 HTTP 客户端封装（带统一拦截器 / 错误处理），业务里又冒出一处 `fetch` / `axios` / `OkHttp` 直调 → 应该走项目封装
- 工具函数同名不同实现（如两份 `formatPhone`）→ 合并到一处

**违规自检（写完代码后必须问自己）:**
1. 我刚写的这个零件，项目里有没有类似的？
2. 我有没有跳过搜索步骤直接写？
3. 如果有人 6 个月后想找这种功能，他能找到几份实现？（答案应是 1 份）

## static 优先原则（强制）

**适用范围:** 所有面向对象语言（Dart / Java / Kotlin / TypeScript / C# / Python class 等）。

**核心规则（成员层面 — 跨语言通用）:** 写**每个**方法 / 字段前必须先问一个问题——

> **"这个成员需要绑定到具体实例吗？"**

| 情况 | 必须 |
|---|---|
| 方法访问 `this.xxx`（实例字段或调用其他实例方法） | 实例方法 |
| 方法只用入参 + 外部资源（其他 static / 全局对象 / 注入参数） | **必须 `static`** |
| 字段值"每个实例独立" | 实例字段 |
| 字段值"全局唯一不变" | **必须 `static`** |

**绝对禁止:**
- ❌ 看到类里有方法直接写成实例方法 —— 必须先想清楚需不需要 `this`
- ❌ 单例模式包装一堆**不用 this** 的方法（伪实例方法）—— 这些方法本来就该是 `static`
- ❌ `instance.xxx()` 调用一个完全不依赖实例状态的方法 —— 直接 `Xxx.xxx()` 就行
- ❌ 写完类不复查 —— 必须自检每个方法/字段是否真用了 this

**类层面的归纳（语言相关）:** 按上面规则判定后，所有成员都是 `static` 的、整个类没有任何实例成员 —— 这种类**必须不可实例化**。各语言对应写法：

| 语言 | 不可实例化的写法 | 注意点 |
|---|---|---|
| **Dart** | `abstract class Xxx { static ... }` | **不需要**私有构造（abstract 已阻止 `new`）。冗余的私有构造是错误信号 |
| **Java / Kotlin** | `public final class Xxx { private Xxx() {} static ... }` | private constructor **必须**（否则能被反射 / 子类化）；`final` 防继承 |
| **TypeScript** | `abstract class Xxx { private constructor() {} static ... }` 或 `namespace` | 同 Java 思路 |
| **C#** | `public static class Xxx { static ... }` | 语言层面有 `static class` 关键字 |

**Java 重要例外 — Spring Bean 不在本规则约束范围内:**

Spring 框架管理的 Bean（`@Service` / `@Repository` / `@Controller` / `@Component` / `@Configuration`）是**有状态的实例**——它们持有注入的依赖（`@Autowired` / 构造器注入）作为**实例字段**。即使某个 Service 方法看起来"不用 this"，它实际上通过注入的字段访问其他 Bean，本质上**有实例状态**。

✅ **Spring Bean 用实例方法 + 依赖注入是 Java 后端核心模式**，不要尝试把它们改 static：

```java
@Service
@RequiredArgsConstructor
public class UserRegisterService {     // ✅ Spring 管理的实例
    private final UserRepository repo;     // 注入的依赖（实例字段）
    private final TokenGeneratorService tokenGenerator;  // 同上

    public RegisterResult register(...) {  // ✅ 实例方法（用了 this.repo / this.tokenGenerator）
        ...
    }
}
```

❌ Spring Bean 的方法**不要**改成 static + 把字段做成 static 全局——这破坏了依赖注入、测试隔离、配置灵活性。

**utility 类 / 常量类（非 Spring 管理）仍受本规则约束:**

✅ 不被注入的纯工具类，按本规则要求：

```java
// ✅ Java utility 类
public final class PhoneUtil {
    private PhoneUtil() {}    // private constructor 必须

    public static final String CN_MOBILE_REGEXP = "^1\\d{10}$";

    public static boolean isValid(String phone) { ... }
}
```

```java
// ❌ 反例：把无状态 utility 包成 Spring Bean 没意义
@Component
public class PhoneValidator {
    public boolean isValid(String phone) {
        return phone.matches("^1\\d{10}$");   // 没用任何注入
    }
}
// ↑ 应该改 final class + private constructor + static 方法
```

**典型反例（Dart 项目里出现过的）:**

```dart
// ❌ XinqiAuthApi.loginByCode 不用 this，却被包装成单例 + 实例方法
class XinqiAuthApi {
  XinqiAuthApi._();
  static final XinqiAuthApi instance = XinqiAuthApi._();
  Future<BaseResp<LoginData>> loginByCode({...}) {     // 没用 this
    return HttpClient.instance.send(...);
  }
}

// ✅ 正解
abstract class XinqiAuthApi {
  static Future<BaseResp<LoginData>> loginByCode({...}) {
    return HttpClient.send(...);
  }
}

// ❌ XinqiRoutes 只有静态常量，写成普通 class + 私有构造
class XinqiRoutes {
  XinqiRoutes._();
  static const String chat = '/chat';
}

// ✅ 正解
abstract class XinqiRoutes {
  static const String chat = '/chat';
}
```

**违规自检（写完类后必须问自己）:**

```
1. 每个非 static 方法都至少访问了一处 this.xxx 吗？
   - 否 → 立刻改 static
   - Java 例外：Spring Bean 内的方法即使看起来不用 this，
     如果它通过 @Autowired 字段调用其他 Bean，那是合理的实例方法，跳过此检查

2. 每个非 static 字段都"每个实例不一样"吗？
   - 否 → 立刻改 static
   - Java 例外：Spring Bean 注入的依赖字段（@Autowired / final + RequiredArgsConstructor）属于实例状态

3. 改完后类还有任何实例成员吗？
   - 否：
     - Dart → 改 abstract class（删私有构造）
     - Java/Kotlin → final class + private constructor
     - TS → abstract class + private constructor 或 namespace
     - C# → static class
   - 是 → 普通 class

4. Dart 项目：abstract class 还留着私有构造吗？
   - 是 → 删掉，冗余
```

**为什么这条规则很重要:**
- 单例 / 伪 Spring Bean 包装无状态方法是**典型的过度设计**——增加了 `instance.` 或 `@Autowired` 调用噪音、占用一份内存、没带来任何收益
- 这种"假实例"在测试时还得专门 mock 单例 / Bean（其实直接 mock 底层依赖就够）
- 写代码时上来就用单例 / Spring Bean 是 AI 的常见错误模式 —— 必须靠这条规则强制每次写方法都问一遍"用不用 this / 注入"，从源头避免

## 文件删除安全检查

**核心规则:** 删除任何文件前，必须用 Grep 搜索文件名确认没有被引用，检查配置文件（pubspec.yaml、AndroidManifest.xml、Info.plist 等）中是否有声明，确认无引用后才能删除。

## 命令行搜索工具规范（强制）

**核心规则:** 在 Bash 里搜代码 / 文件内容时，**默认使用 `rg`（ripgrep），不要用 `grep`**。

**理由:**
- 用户本机已装 `rg`（`/opt/homebrew/bin/rg`，15.x，带 `+pcre2`），可用性不是问题
- `rg` 自动忽略 `.gitignore`，不会扫 `node_modules` / `build/` / `.next/` 等垃圾目录；`grep -r` 会全扫，又慢又脏
- 大仓库下 `rg` 比 `grep -r` 快 5-10 倍（多线程 + SIMD + Rust 正则引擎）
- 默认带颜色、行号、文件名分组，输出更易读

**常用对应:**

| 老 grep 写法 | 改成 rg |
|---|---|
| `grep -rn "foo" .` | `rg foo` |
| `grep -rn "foo" --include="*.ts"` | `rg foo -t ts` |
| `grep -rn "foo" --include="*.{ts,tsx}"` | `rg foo -t ts -t tsx` |
| `grep -irn "foo"` | `rg -i foo` |
| `grep -A 3 -B 3 "foo"` | `rg -C 3 foo` |
| `grep -l "foo" -r .` | `rg -l foo` |
| `grep -c "foo" file` | `rg -c foo file` |

**例外（这些场景仍用 grep）:**
- **处理 stdin 流**：`some_cmd | grep xxx` —— rg 也能读 stdin 但管道里 grep 更顺手
- **服务器 / 容器内**：远程环境（如 `ssh new ...`）不一定装了 rg，先用 grep 保险，或者先 `which rg` 检查再决定
- **要求 POSIX 严格行为**的脚本场景

**绝对禁止:**
- ❌ 在本机项目里写 `grep -rn` 递归搜代码（本机有 rg 没理由不用）
- ❌ 边夸 rg 好用边自己用 grep —— 言行一致

## 语言交互规范

**核心规则:** 无论任何情况（包括上下文压缩后），自己说的话都必须使用**中文**。所有输出内容（操作说明、改动总结、文件变更描述等）都用中文表达。代码中的变量名、方法名等可以直接引用原名，但描述性的语句必须是中文。代码本身和代码注释可以用英文。

**英语纠正规则:** 如果用户用英文跟我对话，每次回答完正事后，检查用户说的英文，只要有任何不对的地方或者不符合母语者日常表达习惯的地方，都要在末尾纠正，告诉用户应该怎么说。**用户用中文对话时，回答末尾不要出现任何英语纠正相关的内容，连"English correction: ... 跳过"这种占位提示也不要写，直接结束回答。**

## Git 提交规范

**分支命名:** 直接使用分支名，不要用 `feature/`、`bugfix/` 等前缀。

**提交时机:** 不主动提交代码，等待用户明确指令。

**提交时自动推送:** 用户说"提交代码"时，默认同时执行 commit 和 push。只有用户明确说"只提交不推送"时才只 commit。

**提交范围:** 只提交 AI 自己的修改。用户说"一起提交"时才包含用户的改动。

**提交信息格式:** 只写改动内容，不添加 Claude/AI 署名（不要 Co-Authored-By 等）。

**提交信息语言:** 所有 commit message **必须用中文**。包括 title 和 body。conventional commit 前缀（feat/fix/refactor/test/docs 等）可保留英文，冒号后的描述用中文。例如：`feat(workbench): 退回修改卡接入真实数据`；禁止写 `feat: add xxx endpoint`。

**跨项目分支确认:** 当原生项目和 Flutter 项目都需要改代码时，如果两个项目分支名称不一致，必须先询问用户。

**子模块分支确认:** 如果主模块和子模块分支名字不一样，必须先询问用户是否要在子模块创建与主模块同名的分支，还是直接提交到子模块当前分支。

**分支合并规范:** 合并分支到 master（或其他目标分支）时，必须使用 `git merge --no-ff`，确保产生明确的 merge commit。禁止 fast-forward 合并，否则在 Git 历史中看不出分支合并记录，不利于后续排查问题。

**Flutter 子模块提交流程:**
1. 先进入子模块目录单独提交和推送
2. 回到主工程提交（包括子模块引用）

## 总结内容规范

**核心规则:** 总结修改内容时，使用普通人能看懂的语言描述，不包含代码片段。面向非技术人员，用业务语言描述功能变化和用户体验改变。

## 富文本文档读取规范

**核心规则:** 读取 Word(.docx)、PPT(.pptx) 等富文本格式文档时，**必须尝试提取并查看其中的图片**。

**操作方式:**
1. 使用 `unzip` 命令解压文档到临时目录（如 `/tmp/xxx_extract/`）
2. 图片通常在 `word/media/`（docx）或 `ppt/media/`（pptx）目录下
3. 使用 Read 工具逐张查看提取的图片
4. 如果解压或读取失败，**必须明确告知用户**图片无法读取，不能默默跳过

**重要:** 不要假设文档只有文字内容，设计图/截图/示意图往往是最重要的信息来源。

## Claude 配置文件同步规范

**核心规则:** `~/.claude/` 目录下的配置、规则等内容发生变更时（包括 `CLAUDE.md`、`settings.json`、`.gitignore` 等），**必须及时提交并推送到 GitHub**（`masterAventador/dotclaude-personal` 仓库，private）。

**白名单（只提交这些）:** `CLAUDE.md`、`rules/` 目录下所有分片规则、`.gitignore` 本身。其他全部由 `.gitignore` 排除（`settings.json` / `plugins/` / `projects/` / `history.jsonl` / `backups/` 等含本地状态、对话记录、敏感 token，禁止提交）。

**触发时机:** 每次修改完 Claude 相关配置后，立即 `git add` → `git commit` → `git push`，不要等用户提醒。

## Monitor 工具使用规范

**核心规则:** 用 Monitor 盯长时间任务（部署、编译等）时，**任务完成后必须主动调 `TaskStop` 杀掉 monitor**，不要让它超时自然退出。

**具体:**
- Monitor 超时自然退出会给用户推一条"Monitor timed out"通知，造成噪音
- 用户明确知道任务已完成时（比如部署流程收到 "deploy done"），应立刻 `TaskStop(monitor_task_id)`
- 监听脚本的 grep pattern 要**同时覆盖成功路径和失败路径**（如 `deploy done|BUILD FAIL|ERROR`），保证不管正常/异常结束都能及时推送
- Monitor 的 `timeout_ms` 只是兜底，不应作为退出机制

**典型流程:**
1. Bash(run_in_background=true) 启动部署脚本
2. Monitor 盯 log，推送中间进度 + 结束标记
3. 收到 "deploy done" 或 "FAIL" 通知 → 立刻 TaskStop
4. 继续下一步（下一轮部署 / 汇报结果）

## 部署规范

**核心规则:** 开发前后端的时候，**永远不要自动去部署**，除非用户明确说"部署"。

**具体:**
- 改完代码、合并到 dev 之后，**停在这一步**，等用户下指令
- 不要主动问"要不要部署"之后就跑，即使用户上一次同意过部署，也要每次都等用户明确下指令
- `./deploy-test-*.sh`、`npm run deploy:dev` 等部署命令只在用户说"部署"/"上测试环境"/"发布"等明确指令时才执行

## 模拟器截图等待时间规范

**核心规则:** 执行点击/滚动脚本后截图，默认等待 **300ms**（`sleep 0.3`），不要用 1.5s 或 3s。只有需要等待网络加载等特殊场景才临时加长等待时间。

## UI 开发临时文件清理规范

**核心规则:** UI 开发过程中产生的截图、对比图等临时文件（通常在 `/tmp/` 目录），**用完后必须及时删除**，不要积累到最后。

**清理时机:**
- 每次截图对比完成、确认无问题后，立即删除该轮产生的截图
- 一个功能点开发完成时，清理该功能相关的所有临时截图
- 不要等到整个任务结束才一次性清理

## Playwright 测试临时文件清理规范

**核心规则:** 使用 Playwright 浏览器测试/截图后，**每跑完一个用例就立即删除**该用例产生的截图和临时文件，不要等全部跑完再清理。

**清理范围:**
- 项目目录下的截图文件（`.png`、`.jpeg`）
- `.playwright-mcp/` 目录（Playwright MCP 的日志/缓存）
- `/tmp/` 下的 Playwright 相关临时文件

**清理时机:**
- 截图查看/对比完成后，立即删除该截图
- 一轮页面浏览测试完成后，清理所有截图和 `.playwright-mcp/` 目录
- 不要等到整个会话结束才清理

## E2E 全量测试浏览器共享规范

**核心规则:** 执行 E2E 全量测试时，**必须使用共享的浏览器和页面**，不要每个测试用例都新开一个浏览器实例。

**实现方式:**
- 在 Playwright 配置或测试 setup 中，使用共享的 browser/context/page，多个测试用例复用同一个浏览器实例
- 避免每个 `test()` 或 `describe()` 块都独立启动和关闭浏览器
- 全量测试开始时启动一次浏览器，所有用例跑完后再关闭

**原因:** 每个用例单独开浏览器会导致执行速度极慢、系统资源浪费严重，尤其是全量测试场景下。

## E2E 全量测试后服务清理规范

**核心规则:** 跑完 E2E 全量测试且最终确认没问题后，**必须主动停掉本地为测试启动的前后端服务**（如前端 dev server、后端 Spring Boot/Node 服务等），避免下次启动时端口占用报错。

**操作流程:**
1. 全量 E2E 测试通过，确认无问题
2. 立即停掉本地启动的前端服务（如 `npm run dev`、`vite` 等）
3. 立即停掉本地启动的后端服务（如 Java 进程、Node 进程等）
4. 确认端口已释放后再继续后续工作

## 本地开发服务管理规范（强制）

**适用范围:** 本机开发环境的所有长期运行服务——数据库（PostgreSQL / MySQL / MongoDB 等）、缓存（Redis / Memcached）、消息队列（Kafka / RabbitMQ）、对象存储（minio）、后端服务（Spring Boot / Node / Python 等）、前端 dev server（Vite / webpack-dev-server / Next dev 等）、Docker compose 起的服务栈等等。

**核心规则:** **按需启动，用完就关。禁止常驻、禁止开机自启。**

**绝对禁止:**
- ❌ `brew services start <name>`（这会设置开机自启 + 后台常驻）
- ❌ `systemctl enable <name>`（Linux 同理）
- ❌ Docker `docker compose up -d` 之后忘记 `down`
- ❌ 后端 / 前端服务一启动就放着不管，下班 / 关电脑前不停
- ❌ "反正它在后台不影响"——它影响内存、电池、安全攻击面、环境干净度

**正确做法:**

| 服务类型 | 启动方式 | 停止方式 |
|---|---|---|
| brew services（pg / redis 等） | `brew services run <name>`（不自启）或临时 `brew services start <name>` | `brew services stop <name>` |
| 后端服务（Spring Boot / Node） | IDE 里 Run/Debug；或命令行 `mvn spring-boot:run` / `npm start` | 关 IDE 进程 / Ctrl+C |
| 前端 dev server | `npm run dev` / `fvm flutter run` 等 | Ctrl+C |
| Docker compose | `docker compose up`（前台跑） | Ctrl+C / `docker compose down` |

**理由:**
1. **资源占用**：pg + redis + 后端 + 前端 dev server 加起来轻松 2-4GB 内存，Mac 电池也吃
2. **开机速度**：自启服务越多，开机越慢
3. **安全面**：暴露的端口越多攻击面越大（即使本机也别留默认配置）
4. **环境一致性**：每次开发前过一遍"我现在需要哪些服务"，避免上次的脏状态影响这次
5. **强迫脑子记得它在跑**：避免出现"上次开发完忘了停，三天后才发现 redis 一直在跑"

**生产环境例外:** 本规则只适用于**本机开发**。线上服务器当然要常驻 + 开机自启 + 监控守护——那是生产责任，跟本规则无关。

**macOS 自检命令:**
```bash
brew services list           # 看是否有 started 的服务
launchctl list | grep -v 0x  # 看 launchd 自启列表
ps aux | grep -E "java|node|python" | grep -v grep  # 看是否有遗留进程
```

不需要的全部 stop / kill。开发要用时再 `brew services start` + 用完 `brew services stop`。

**Claude 协助开发时的自觉:** 当 Claude 协助跑测试 / 部署 / 调试 / dogfood 时启动了任何后端 / 前端 / DB / 缓存等服务，**任务结束后必须主动停掉**——按本规则。不要假设"用户后面还会用所以留着"。用户需要时会重新启。

## Feature-Dev 流程规范

**核心规则:** 执行 feature-dev skill 时，**不允许跳过任何步骤**。必须严格按照 Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 5 → Phase 6 → Phase 7 的顺序执行，每个 Phase 都要完整执行，不能因为觉得"简单"或"明显"就跳过。

## Superpowers 可视化伴侣使用规范

**核心规则:** 在 superpowers 流程（尤其是 brainstorming）中，**如果用户同意了使用可视化伴侣**，当需要展示数据结构、架构图、流程图、模块关系图、数据模型、API 设计等结构化/图形化内容时，**必须使用可视化伴侣（Visual Companion）在浏览器中呈现**，而不是仅用文字输出。如果用户拒绝了可视化伴侣，则正常用文字输出即可。

**适用场景:** 数据模型/表结构、架构图/模块关系图、流程图/时序图、API 设计概览、方案对比、任何用图形比文字更直观的内容

**不适用场景:** 纯文字讨论（需求澄清问题、概念选择等）

**内容留存规范:**
- 用户选择完方案后，最终确认的设计内容（架构图、流程图、数据模型、页面结构等）**必须保存下来**，方便以后直接打开网页查看整体项目设计
- **只保留最终选择的方案**，不保留选择过程中的备选项（如 A/B/C 方案选择页面）
- 留存文件统一放在项目的 `.superpowers/brainstorm/` 目录下，使用语义化文件名（如 `architecture.html`、`data-model.html`、`page-structure.html`）
- brainstorming 结束时，清理掉选择过程中的临时文件，只保留最终确认的设计图

## ⛔ TDD 铁律 — 最高优先级，不可违反

**这是所有规则中优先级最高的一条。任何理由、任何借口、任何"效率考量"都不能绕过它。违反此规则等同于产出不合格代码。**

**铁律：没有失败的测试，就不允许写任何生产代码。一行都不行。**

**强制流程（Red-Green-Refactor）：**
1. **RED — 先写测试，运行，必须看到失败**。不是"脑补它会失败"，而是实际执行测试命令，亲眼看到红色失败输出。
2. **GREEN — 写最小实现代码，让测试通过**。只写刚好让测试通过的代码，不多写一行。
3. **REFACTOR — 重构**。测试通过后才能优化代码结构。

**绝对禁止的行为：**
- ❌ 先写实现代码再补测试 — **这不是 TDD，这是事后找补，严禁**
- ❌ 同时写测试和实现 — **必须分开，先测试后实现，中间必须有一次失败的测试运行**
- ❌ 发现 bug 后直接修复 — **必须先写一个能暴露该 bug 的失败测试，再修复**
- ❌ "这个太简单了不需要 TDD" — **没有这个例外。简单的代码 TDD 更快，没有理由跳过**
- ❌ "用户说了不要停下来问我" — **这不是跳过 TDD 的理由。TDD 是内部流程，不需要问用户**
- ❌ "时间紧" — **TDD 不会更慢，跳过 TDD 写出的代码才会因为返工浪费更多时间**

**适用范围：** 所有代码变更，包括但不限于：功能开发、bug 修复、重构、E2E 测试对应的页面代码修复、配置文件中涉及业务逻辑的变更。唯一例外是 TDD skill 自身定义的三种例外（Throwaway prototypes、Generated code、Configuration files），且不允许自行扩展例外范围。

**违规自检：** 如果发现自己已经写了生产代码但没有先写失败测试，必须**立即停下来**，删掉刚写的生产代码，回到 RED 步骤重新开始。不允许"下次注意"。

## 子代理 prompt 注入规则

**核心规则:** 派遣子代理时，prompt 中必须显式注入**子代理无法靠自行读取获得**的信息：

1. **主对话中已达成的决策和上下文**（子代理看不到主对话历史）
   - 架构选择（如"本任务已选 A1 方案：api+core 双模块"）
   - 临时约束、例外、用户明确偏好

2. **相关 spec / plan 文件路径**（让子代理主动 Read 加载）
   - 已写好的设计文档 / 实现计划路径
   - 验证标准和完成定义

**不需要 prompt 显式注入（子代理会自行加载）：**
- 全局 `~/.claude/CLAUDE.md` 和 `~/.claude/rules/` 下无 `paths` 的规则（启动时自动加载）
- 带 `paths` 的规则分片（子代理 Read 匹配文件时自动触发）

**适用范围:** 所有派遣子代理的场景。下面 "Superpowers 流程规范" 里的 TDD 专项条款是本规则的一个具体实例。

## Superpowers 流程规范

**核心规则:** 使用 superpowers 的任何 skill 时，**必须 100% 严格按照 skill 定义的流程执行**。这是**最高优先级规则**，任何理由都不能违反：
- **不允许跳过任何步骤**，即使觉得"太简单"、"没必要"、"显而易见"
- **不允许合并步骤**，即使觉得"效率更高"
- **不允许简化流程**，即使觉得"本质一样"
- **不允许自作主张省略**，必须把 skill 文档当作法律条文逐条执行

**如果 skill 引用了其他 skill（如"Subagents should use: superpowers:test-driven-development"），被引用的 skill 也必须完整执行，不能只"参考精神"。**

**禁止自行添加 TDD 例外：**
- TDD skill 已经定义了例外情况（Throwaway prototypes、Generated code、Configuration files），除此之外不允许自行扩展例外范围
- **前端组件不是"配置文件"**——有交互逻辑的组件（事件处理、状态变更、API 调用、表单提交）是生产代码，必须先有失败的测试
- **"UI 代码"、"薄层封装"不是跳过 TDD 的理由**——如果代码有逻辑分支或副作用，就必须 TDD
- 子代理 prompt 中**绝对不能**写"TDD exception applies"之类的豁免语句，除非该代码确实属于 TDD skill 定义的三种例外
- **每个 Task 必须验证测试通过，但是否跑全量见下文「Superpowers Task 测试粒度规范」**——按 task 波及范围分级，不是每个 task 无脑全量

**TDD 任务级 prompt 注入要求（本章节作为"子代理 prompt 注入规则"的具体实例）：**
- 虽然子代理会自动加载 `CLAUDE.md` 里的 TDD 铁律，但**任务级的 TDD 细节**必须通过 prompt 显式注入
- **每次派遣实现子代理时，必须在 prompt 中包含完整的 TDD 流程要求**（Iron Law、Red-Green-Refactor、验证步骤）
- 不能假设子代理"应该知道"要 TDD，必须显式写明
- 子代理的 prompt 中必须包含：需要创建的测试文件列表、具体的测试用例描述、`mvn test` 或 `npm test` 的验证命令
- 子代理返回后，我必须通过 spec 审查确认测试确实存在且覆盖了业务逻辑

**具体要求:**
- **test-driven-development**: 所有有业务逻辑的代码必须 TDD。先写测试 → 运行确认失败 → 写最小实现 → 运行确认通过。子代理必须在 prompt 中接收完整的 TDD skill 流程并严格执行。没有失败的测试就没有生产代码。
- **subagent-driven-development**: 每个 Task 必须单独派遣一个子代理，完成后必须**依次**执行 spec 审查和代码质量审查，**两轮审查必须是独立的两次子代理调用**，绝对不允许合并为一次。spec 审查用 general-purpose 子代理，代码质量审查用 superpowers:code-reviewer 子代理。两轮审查都通过后才能进入下一个 Task。不允许合并多个 Task 给同一个子代理。即使觉得"Task 简单"、"逻辑不复杂"也不能合并审查，没有任何例外。
- **子代理类型必须匹配 skill 模板定义**: 实现子代理用 general-purpose，spec 审查用 general-purpose，代码质量审查用 superpowers:code-reviewer。**绝对不能**使用 feature-dev:code-reviewer 或其他非 superpowers 体系的 agent 来执行 superpowers 流程中的步骤。
- **brainstorming**: 必须完整走完 checklist 的每一步，包括 spec 审查循环和用户审查。
- **writing-plans**: 每个 chunk 写完后必须派遣审查子代理，修复所有 issues 后才能继续。
- **executing-plans**: 必须按计划顺序执行，每个 checkpoint 都要停下来审查。
- **verification-before-completion**: 宣称完成之前必须运行验证命令并确认输出。
- 其他 skill 同理，skill 文档里定义的步骤就是必须执行的步骤。

**违规自检:** 如果发现自己跳过了某个步骤，必须立即停下来补回去，不能"下次注意"。

## Superpowers Task 测试粒度规范

**核心规则:** 使用 superpowers 跑大型 plan（5+ task）时，**不是每个 task 都跑全量测试**——按 task 的"波及范围"决定测试粒度。30 个 task × 全量 = 大量重跑已经验证过的旧用例，浪费时间。

**每个 Task 必跑（不可跳过）:**
- 该 Task 新增/修改文件所属包的测试（如 `yes | fvm flutter test business_packages/ekw_topic`）
- `flutter analyze` 防止编译警告
- 该 Task 新增的单元测试必须先 RED 再 GREEN（TDD 铁律不变）

**触发跑全量测试的场景（必须）:**

| 场景 | 理由 |
| --- | --- |
| Task 改动涉及 foundation_packages（基础层下沉、公共组件 API 改动、共享 token 调整） | 改基础层会广泛影响所有调用方，必须全量验证 |
| Task 修改了被多包引用的现有公共类 | 同上 |
| Task 改动了多个业务模块间的跳转 / 共享 enum / 共享 model | 跨模块影响 |
| 每完成一个 Phase 做一次 checkpoint（每 3-5 个 task 一组） | 防止跨 task regression 累积超过 5 个 commit，找问题成本可控 |
| 整个 plan 完成、merge 前的 final gate | 必须 |

**只跑相关包的场景:**
- 业务模块内新增 page / controller / widget / req / 测试
- 业务模块内私有逻辑修改
- api 层（非跨模块共享）新增

**为什么这样合理:**
- 每个 task 的"自检测试"（新加的单测 + 相关包测试）已经覆盖你这次写的代码
- 全量的额外价值是 catch **跨模块 regression**——但连续多个 task 都在同一模块内时，连续跑 3 次全量等于在重跑同一批旧用例
- Phase checkpoint 把跨 task regression 的 bisect 距离控制在 5 个 commit 内，找问题代价可控
- 只在最后跑全量的风险：Task 5 引入隐性 regression，到 Task 30 才发现要 bisect 25 个 commit；Phase checkpoint 把这个范围压到 5 个内

**子代理 prompt 中要写明:**
- 子代理看不到 CLAUDE.md，每次派遣实现子代理时必须显式说明本 task 该跑什么测试
- 例：`只跑 business_packages/ekw_topic 包的测试 + flutter analyze 即可，不需要跑全量`
- 或：`本 task 涉及 foundation 层下沉，必须跑全量 yes | fvm flutter test`

## 自动化测试规范

**核心规则:** 所有项目的前后端代码都必须编写自动化测试。每个 Task 提交前必须运行**相关包**的测试并确认通过；是否跑全量按上文「Superpowers Task 测试粒度规范」分级判定（不是每个 task 都需要全量）。

### 后端测试（Java 项目）

- **单元测试**：Service、Repository、核心业务逻辑类必须有单元测试，使用 JUnit + Mockito
- **接口自动化测试**：每个 Controller 接口必须有集成测试，使用 `@SpringBootTest` + `MockMvc`
- **TDD 流程**：先写失败的测试 → 运行确认失败 → 写最小实现 → 运行确认通过
- **测试文件位置**：与源码同包路径，放在 `src/test/java/` 下

### 前端测试（Vue/React 项目）

- **单元测试**：使用 Vitest，对工具函数、Store、API 函数、组件逻辑编写单元测试
- **组件测试**：使用 Vitest + @vue/test-utils（Vue）或 Testing Library（React），测试组件交互逻辑
- **E2E 测试**：使用 Playwright，覆盖核心业务流程
- **TDD 流程**：有业务逻辑的代码必须先写测试再写实现

### 测试要求

- 每个 Task 完成后必须运行**相关包**的测试（具体粒度见上文「Superpowers Task 测试粒度规范」）
- 写实现计划时，每个 Task 必须按 TDD 顺序组织步骤（测试先行），不允许先写实现再补测试
- 整个 plan 完成、merge 前必须跑一次完整的全量测试做 final gate

## 服务器操作规范

**服务器信息:**

| 别名 | IP | 角色 |
|---|---|---|
| `new` | `49.233.213.109` | **默认服务器**（Debian 13），用户说"服务器"时优先使用 |
| `old` | `154.8.162.83` | 备用服务器，**仅当用户明确说明"老服务器"或"154"时使用** |

- 用户: `root`
- 登录方式: SSH 密钥认证（已在 `~/.ssh/config` 中配置好别名，本机公钥已推送）

**操作方式:** 当需要在服务器上执行任何操作时（部署、查看日志、管理服务等），默认使用 `ssh new` 登录执行命令。只有用户明确说明是老服务器时才用 `ssh old`。
