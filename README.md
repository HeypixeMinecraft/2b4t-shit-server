# 2B4T Shit Server 分析报告

仓库：<https://github.com/HeypixeMinecraft/2b4t-shit-server>  
审计版本：`1c3b80b4929efa36919fd4a113e1f487c1d8bd8e`  

## 总结

这个仓库是一个完整 Paper 服务端快照，而不是干净的插件源码仓库。它包含 `paper.jar`、`libraries/`、`plugins/`、`logs/`、玩家数据、认证数据、OP 列表、控制台历史和大量运行时产物。

结论比较直接：

- 服务端确实有明显“AI 生成插件堆叠”的特征。
- 真实安全风险比代码质量问题更严重。
- 当前仓库不应该公开保存，也不应该直接作为生产服继续使用。
- 需要先做密钥轮换和玩家密码作废，再谈插件重构。

## 服务端概况

从 `logs/latest.log` 可见：

- Java：OpenJDK 21
- 系统：Windows 11
- 服务端：Paper `1.21.11-99`
- Minecraft：`1.21.11`
- 插件数：20 个
- 日志警告：服务端以管理员/root 权限运行

主要插件：

- 自制/疑似自制：`2B4T-Server-Plugin`、`2B4T-Server-Extension`、`2B4T-DeepSeek`、`2B4T-AntiEvasion`、`2B4T-account`、`2B4T-org`、`2B4TOP-limit`、`BotCommandExecutor`、`Daemon`、`zoob`、`output`
- 第三方：`GrimAC`、`WorldEdit`、`ViaVersion`、`ViaBackwards`、`ViaRewind`、`voicechat`、`TPA`、`LoginTo`、`SModeration`

## 最高优先级风险

### 1. DeepSeek API Key 已泄露

发现位置：

- `plugins/2B4T-DeepSeek/config.yml:3`
- `plugins/BotCommandExecutor/config.yml:6`

影响：

- 任何拿到仓库的人都可以调用这个 API Key。
- BotCommandExecutor 还把 AI 和服务端命令执行绑定在一起，风险更高。

处理：

- 立即在 DeepSeek 后台吊销该 Key。
- 重新生成 Key，不要提交到 Git。
- 改成环境变量或独立的未跟踪配置文件。
- Git 历史中已经出现过的 Key 也视为永久泄露。

### 2. 玩家密码明文泄露

发现位置：

- `plugins/LoginTo/data.json`
- `_1.console_history`

问题：

- `LoginTo/data.json` 中存在大量明文 `password` 字段。
- `_1.console_history` 中出现过 `changepassword` 命令及明文密码参数。

影响：

- 玩家账户已经不能视为安全。
- 如果玩家复用密码，可能影响站外账号。
- 公开仓库中保存这种数据是非常严重的隐私事故。

处理：

- 立即删除仓库中的 `plugins/LoginTo/data.json` 和控制台历史。
- 强制所有玩家重置密码。
- 换用支持强哈希的登录插件，至少要有 bcrypt/argon2/PBKDF2。
- 不要把运行时玩家数据提交到 Git。

### 3. 离线模式 + 多个 level 4 OP

发现位置：

- `server.properties:46`：`online-mode=false`
- `server.properties:47`：`op-permission-level=4`
- `ops.json`：多个 `level: 4` OP

影响：

- 离线模式下 UUID 与正版身份不绑定。
- 如果登录认证链、IP/代理链、反假名逻辑有漏洞，OP 冒用风险会非常高。
- `op-permission-level=4` 意味着 OP 拥有最高命令权限。

处理：

- 能开正版验证就开 `online-mode=true`。
- 如果必须离线模式，必须接入可信代理并正确配置 forwarding。
- 最小化 OP 数量。
- 把管理权限迁移到 LuckPerms 等权限系统，不要长期依赖原生 OP。

### 4. AI 执行服务端命令插件非常危险

相关插件：

- `BotCommandExecutor`
- `2B4T-DeepSeek`

证据：

- `BotCommandExecutor` 描述为允许管理员通过 DeepSeek API 执行命令。
- `ChatListener` 监听 `@bot`。
- `CommandManager` 使用 DeepSeek API 结果提取命令，然后通过控制台执行命令。
- 允许命令列表包含 `give`、`tp`、`gamemode`、`effect`、`clear`、`kick`、`ban` 等。

风险：

- prompt injection 后果直接变成控制台命令执行。
- 只要权限配置、OP 账号、登录插件任一处出问题，就可能扩大为全服控制。
- AI 输出不可预测，不适合直接接入命令执行。

处理：

- 生产服立即禁用 `BotCommandExecutor`。
- 如果保留，只允许只读查询类命令，不能执行控制台命令。
- 所有 AI 生成命令必须进入人工审批队列。
- 给每个命令做结构化 schema、参数白名单和审计日志。

## AI 生成痕迹

这些插件很像由 AI 或模板快速生成：

### 1. `author: YourName`

出现于：

- `2B4T-AntiEvasion`
- `2B4T-DeepSeek`
- `2B4T-Server-Extension`
- `2B4T-Server-Plugin`

这是典型的生成模板没有清理干净。

### 2. 包名保留 `com.example`

出现于：

- `2B4T-org`：`com.example.twobfourtorganizations`
- `BotCommandExecutor`：`com.example.botcommandexecutor`
- `Daemon`：`com.example.daemon`

`com.example` 是教学/模板包名，不应该出现在正式服务端插件里。

### 3. 命名和职责碎片化

自制插件数量过多，但职责边界混乱：

- `2B4T-Server-Plugin`：公告、更新倒计时、聊天敏感词、用户名限制。
- `2B4T-Server-Extension`：死亡统计。
- `2B4T-AntiEvasion`：聊天/命令规避检测。
- `Daemon`：命令限制、内存监控、TPS 保护、连接安全。
- `2B4TOP-limit`：OP 权限管理。
- `zoob`：OP 行为限制和特殊武器。
- `BotCommandExecutor`：AI 命令执行。

这些更像“想到一个功能就让 AI 写一个插件”，而不是统一设计的服务器治理系统。

### 4. 过度 shade 依赖

`2B4T-DeepSeek` 打包了 OkHttp、Gson、JetBrains annotations 等大量类。  
`2B4TOP-limit` 打包了 Kotlin 标准库，实际业务类只有少数几个。

这会带来：

- jar 体积膨胀。
- 重复依赖。
- 类冲突风险。
- 安全更新困难。

### 5. 版本和 API 不一致

例子：

- 服务端是 Paper/Minecraft `1.21.11`。
- `2B4T-account` 的 POM 使用 `1.21.10-R0.1-SNAPSHOT`。
- `zoob` 的 POM 使用 `1.21.1-R0.1-SNAPSHOT`。
- 多数自制插件仍是 `1.0` 或 `1.0-SNAPSHOT`。

这说明没有统一构建基线。

## 插件级分析

### `2B4T-DeepSeek`

用途：玩家通过 `@ai` 或 `/deepseek` 使用 DeepSeek 问答。

主要问题：

- API Key 明文写在配置中。
- 使用玩家输入直接发给外部 AI API。
- 没有看到足够的敏感信息过滤、速率全局限流、内容审计。
- 系统提示中写死服务器信息，维护性差。

建议：

- 可保留，但必须移除硬编码 Key。
- 加全局 QPS 限制、玩家每日额度、消息长度限制。
- 日志不要记录完整玩家提问。
- 明确告知玩家聊天会发送到第三方 API。

### `BotCommandExecutor`

用途：通过 AI 解析玩家 `@bot` 消息并执行命令。

主要问题：

- 风险极高，不建议在生产服使用。
- AI 输出和控制台命令执行之间缺少人工审批。
- 默认 API Key 明文写入字节码 fallback 和配置。
- 允许命令列表仍包含高影响命令。

建议：

- 删除或彻底重写。
- 如果要保留，改成“AI 生成建议命令 -> 管理员点击确认 -> 执行”。
- 禁止 `give`、`gamemode`、`effect`、`ban`、`kick`、`tp` 这类直接改变玩家或世界状态的命令。

### `2B4T-account`

用途：账户安全、认证状态、官方认证。

主要问题：

- 与 `LoginTo` 功能重叠。
- 配置里有登录/注册提示，但真实认证似乎仍依赖 LoginTo。
- 插件只记录运行时状态，是否能安全处理断线、重进、崩服恢复需要重新验证。

建议：

- 不要同时维护两个登录/账户系统。
- 选择一个成熟登录插件作为唯一认证源。
- 自制账户插件只做“认证后附加功能”，不要自己处理密码。

### `LoginTo`

用途：登录注册。

主要问题：

- `data.json` 存在明文密码。
- 运行时数据被提交到仓库。

建议：

- 立即替换或升级到安全存储方案。
- 清空公开仓库中的玩家认证数据。
- 强制玩家改密。

### `2B4T-org`

用途：组织、标签、公告、领地/签证。

主要问题：

- 包名是 `com.example`。
- `/sudo` 命令会伪造玩家聊天输出，虽然权限默认 OP，但审计和滥用风险很高。
- 组织、签证、标签、公告、领地保护耦合在一个插件内。

建议：

- 拆掉 `/sudo`，或改名为 `/fakechat` 并强制记录审计日志。
- 把组织数据模型和聊天展示分离。
- 使用 UUID 作为主键是对的，但配置数据需要备份和迁移工具。

### `Daemon`

用途：管理员身份、禁用命令、内存/TPS 监控、连接安全。

主要问题：

- 使用玩家名 `bu_xu_yao` 作为特殊管理员身份，容易被离线模式绕过。
- 命令拦截依赖 `PlayerCommandPreprocessEvent`，不是可靠权限系统。
- 内存“自动优化”通常效果有限，可能只是触发 GC 或清理逻辑，容易制造假安全感。

建议：

- 不要用玩家名做超级管理员判断。
- 使用 LuckPerms 权限节点。
- 监控只做报警，不要自动做高风险动作。

### `2B4TOP-limit`

用途：OP 权限管理。

主要问题：

- 主类名被混淆为 `h.a.ATRgFhNYZpKS`。
- 对一个管理 OP 权限的插件来说，混淆会严重降低可审计性。
- 打包 Kotlin 运行库，业务类很少，体积和复杂度不成比例。

建议：

- 不建议在生产服运行不可审计的 OP 管理插件。
- 需要源码、构建脚本和明确策略后再考虑保留。

### `2B4T-Server-Plugin`

用途：公告、倒计时、聊天过滤、用户名检查。

主要问题：

- 聊天过滤逻辑简单，使用包含匹配和正则替换，容易误伤和绕过。
- 违规词触发后会通过控制台执行 mute 类命令，依赖外部命令存在且格式正确。
- 敏感词、公告、更新倒计时混在一个插件内。

建议：

- 聊天治理交给成熟 moderation 插件。
- 服务器公告/倒计时可以保留，但应独立成小插件。

### `2B4T-AntiEvasion`

用途：反规避检测。

主要问题：

- 依赖 `2B4T-Server-Plugin`。
- 从类名看只监听聊天和命令预处理。
- 很可能只是基于字符串规则的规避检测，容易误伤/绕过。

建议：

- 和聊天过滤合并成一个 moderation 模块。
- 规则需要可测试，不要只靠硬编码。

### `zoob`

用途：限制 OP 玩家行为、发特殊武器。

主要问题：

- 权限节点都是 OP 默认。
- 功能描述比较危险：特殊武器、限制 OP 行为。
- 需要完整源码审计后才能判断是否安全。

建议：

- 暂时禁用。
- 把特殊物品发放放到权限系统和审计日志后面。

### `output.jar`

用途不明。

主要问题：

- 主类 `Wanxin1337Obfuscation.l`，明显混淆。
- 名称 `output.jar` 不像正式插件。
- 不可维护，不可审计。

建议：

- 直接删除，除非能提供源码和明确用途。

## 配置问题

### `server.properties`

关键项：

- `allow-flight=true`
- `online-mode=false`
- `op-permission-level=4`
- `max-players=2026`
- `player-idle-timeout=1`
- `sync-chunk-writes=true`
- `view-distance=6`
- `simulation-distance=4`
- `management-server-secret` 已提交

建议：

- `management-server-secret` 立即轮换。
- 如果不是刻意允许外挂飞行，重新评估 `allow-flight=true`。
- `player-idle-timeout=1` 太激进，会导致玩家体验差。
- `max-players=2026` 不现实，容易误导容量规划。
- `sync-chunk-writes=true` 更安全但可能牺牲性能，结合磁盘和崩服恢复策略评估。

### Git 仓库内容

不应提交：

- `paper.jar`
- `libraries/`
- `versions/`
- `cache/`
- `logs/`
- `plugins/*/data`
- 玩家数据
- 控制台历史
- 任何 API Key、密码、token、secret
- 运行时生成的数据库，如 `violations.sqlite`

建议仓库只保留：

- 插件源码
- 构建脚本
- 脱敏配置模板
- `README`
- 部署脚本模板
- `.gitignore`
- 文档

## 插件处理清单

| 插件 | 建议 | 原因 |
| --- | --- | --- |
| `BotCommandExecutor` | 删除/重写 | AI 直接执行命令，风险最高 |
| `output.jar` | 删除 | 混淆且用途不明 |
| `2B4TOP-limit` | 暂停使用 | OP 管理插件不可审计 |
| `LoginTo` | 替换/升级 | 明文密码数据已泄露 |
| `2B4T-DeepSeek` | 可重写保留 | 作为问答机器人可以，但不能硬编码 Key |
| `Daemon` | 重写 | 玩家名超级管理员和命令拦截不可靠 |
| `2B4T-account` | 合并/弱化 | 与 LoginTo 重叠 |
| `2B4T-org` | 重构 | 功能多且包名模板化 |
| `2B4T-Server-Plugin` | 拆分 | 公告、聊天、用户名检查混杂 |
| `2B4T-AntiEvasion` | 合并进 moderation | 字符串规则型插件单独存在价值低 |
| `zoob` | 暂停审计 | 特殊武器/OP 限制风险高 |
| `GrimAC` | 保留 | 成熟反作弊，继续调配置 |
| `ViaVersion/ViaBackwards/ViaRewind` | 保留 | 常规兼容插件 |
| `WorldEdit` | 仅管理员使用 | 必须配权限和限制 |
| `voicechat` | 按需保留 | 注意端口和隐私说明 |
| `SModeration` | 保留或替换为统一 moderation | 避免和自制聊天插件冲突 |

## 优先整改路线

### 第 0 天：止血

1. 立即吊销 DeepSeek API Key。
2. 删除公开仓库或改私有。
3. 清除 Git 历史中的密钥、密码、日志和玩家数据。
4. 强制所有玩家重置密码。
5. 禁用 `BotCommandExecutor`、`output.jar`、`2B4TOP-limit`。
6. 检查所有 OP，移除不必要 OP。

### 第 1 周：安全基线

1. 引入 LuckPerms 管理权限。
2. 统一登录认证方案。
3. 统一 moderation，不再让多个插件同时处理聊天/命令。
4. 所有配置改成模板加本地私有配置。
5. 服务端不要用管理员权限运行。
6. 建立备份和恢复流程。

### 第 2 到 4 周：工程化

1. 建一个 monorepo 保存所有自制插件源码。
2. 统一 Maven/Gradle、Java 版本、Paper API 版本。
3. 删除 `com.example`、`YourName`、`1.0-SNAPSHOT` 等模板残留。
4. 给关键插件加单元测试和集成测试。
5. 每个插件写清楚职责、命令、权限、配置和数据文件。
6. CI 自动构建 jar，不再手动提交 jar。

## 建议的新架构

把当前一堆小插件合并成几个边界清楚的模块：

| 模块 | 包含功能 |
| --- | --- |
| `core` | 公告、重启倒计时、公共工具、统一配置 |
| `auth-bridge` | 只对接成熟登录插件，不保存密码 |
| `moderation` | 聊天过滤、反规避、禁言、审计 |
| `orgs` | 组织、标签、签证、领地关系 |
| `admin-guard` | OP/权限保护，但基于 LuckPerms |
| `ai-chat` | 只做问答，不执行命令 |
| `telemetry` | TPS、内存、玩家数、告警 |

## AI 写代码的使用建议

AI 可以继续用，但不要让 AI 一次写完整插件后直接丢进生产服。

更安全的流程：

1. 先让 AI 写需求文档和权限表。
2. 再让 AI 写最小实现。
3. 人工检查命令权限、事件监听、线程模型、文件读写。
4. 本地测试服验证。
5. 压测或模拟玩家行为。
6. 再上线。

AI 写 Bukkit/Paper 插件时必须特别检查：

- 是否在主线程做网络请求或磁盘 IO。
- 是否在异步线程调用 Bukkit API。
- 是否把玩家输入拼接成命令。
- 是否使用玩家名作为身份判断。
- 是否有权限节点。
- 是否记录审计日志。
- 是否泄露 token、密码、IP。
- 是否处理重载、禁用、崩服恢复。

## 最终判断

这个服务端目前像是“跑起来了，但治理没有跟上”的状态。  
如果只是朋友服或实验服，可以继续折腾；如果是公开服，当前仓库暴露出的风险已经达到必须停服整改的级别。

最危险的不是“插件 AI 写的”，而是：

- AI 写的插件直接接入控制台命令。
- 密钥和玩家密码进入仓库。
- 离线模式下仍保留大量高权限 OP。
- 权限和安全逻辑分散在多个自制插件里。
- 混淆 jar 和不明 jar 缺少源码审计。

优先级一句话：

先删密钥和明文密码，再禁用 AI 命令执行，再收回 OP，最后重构插件。
