# mir2x Go 服务端复刻工作文档

- 文档版本：v0.4（服务端-only 评审定稿）
- 编写日期：2026-11-24
- 参考源码：E:\work\game\mir2x
- 参考提交：58264934b5562cd283db7dc7daddec496b2de86e
- Go 实现目录：E:\work\Godot-Mir2\server-go
- 当前状态：仅方案与工作拆解，未开始功能代码
- 交付范围：只做服务端；客户端不在交付范围
- 测试策略：tools/mockclient + tools/fixturegen，不依赖完整 mir2x_assets

---

## 0. v0.4 评审补充决议

本版根据评审意见调整如下：

1. 不依赖完整 mir2x_assets，不把贴图、音效、字体、表情等客户端资源纳入工程。
2. 不依赖真实客户端；服务端测试使用 tools/mockclient 按协议驱动并断言结果。
3. 测试用地图、数据库和脚本数据由 tools/fixturegen 生成，放在 tools/fixtures 或 testdata。
4. 经核对 tools/installres/src/install_res.py，服务端安装实际只需要 map/mapbin.zsdb，不含客户端图像和音频资源。
5. 真实地图包属于运营数据，可后续通过配置加载；它不是 Go 服务端代码完成的阻塞项。
6. 完整服务端指代码、协议、玩法、持久化和运维能力完整；真实世界内容是否可玩取决于后续提供的合法地图数据。
7. 管理界面确定为 Web 管理台，不保留 FLTK 管理界面，也不把 CLI 作为管理界面交付。
8. Lua 确定为官方 Lua 5.4 + cgo 桥接，不翻译成 Go；MSYS2 UCRT64/MinGW 仅作为 Windows 构建工具链。

---

## 1. 文档目的

本文档用于在正式复刻前确认以下事项：

1. 复刻的目标、边界和验收口径；
2. 与现有 C++ 服务端、客户端、数据库、资源和 Lua 脚本的兼容策略；
3. Go 服务端的总体架构和模块划分；
4. 完整工作拆分、里程碑、人员配置和排期；
5. 技术风险、前置条件和需要业务方拍板的决策。

本文档通过评审后，再开始 Phase 0 的工程落地。

---

## 2. 结论摘要

### 2.1 可行性

可行。建议采用「兼容替换」而不是「逐行翻译」：

- 对外保持客户端协议、SQLite 数据、地图与资源格式、Lua 脚本接口兼容；
- 对内用 Go 重新实现 Actor 调度、网络、地图、战斗、物品、社交和任务系统；
- 以 C++ 服务端作为行为基准（oracle），通过抓包回放和差分测试保证行为一致；
- 先完成单进程多地图版本，再在相同领域接口上扩展 master/slave 分布式。

### 2.2 工作量判断

源码规模与复杂度：

- 服务端 C++ 源码约 3.1 万行，165 个 .cpp/.hpp 文件；
- 服务端 Lua 脚本 690 个，约 4.4 万行；
- 公共协议、数据结构和通用逻辑约 1.87 万行 C++，另有大量生成的记录表；
- 约 89 个客户端消息枚举项、117 个服务端消息枚举项；
- 160 个不同的 Actor 消息类型；
- SQLite 22 张表；
- 原服务端还包含约 1.1 万行 FLTK 管理界面代码。

建议按 4 名工程师 + 1 名自动化测试工程师、24 周左右规划。总体约 22-26 工程师人月。若只有 2-3 人，周期需要明显拉长。

### 2.3 当前环境缺口

在开始复刻前，当前开发机还缺以下环境：

- Go 工具链未安装或未进入 PATH；
- MSYS2/C++ 编译器、Ninja、Lua、SQLite 命令行工具未就绪；
- C++ 参考服务端尚未构建；
- 测试地图和 fixture 数据库尚未生成。

以下内容不再是阻塞项：

- 不需要完整 mir2x_assets；
- 不需要真实客户端参与开发；
- 不需要等待地图包的商业授权才能开始编码。

测试数据全部由 tools/fixturegen 生成，mock 客户端由 tools/mockclient 提供。真实地图包只在运营部署时可选加载。

### 2.4 最终交付形态

v1.0 最终交付一个可独立部署的 Go 服务端包，目标形态如下：

```text
server/
  mir2d                  # Go 服务端主程序，可提供 server 兼容启动名
  config.yaml            # 端口、数据库、地图、脚本、日志、管理端口
  script/                # 兼容上游的 NPC/地图/任务 Lua 脚本
  map/mapbin.zsdb        # 测试 fixture 或运营方提供的真实地图包
  mir2x.db3              # SQLite 数据库，可新建或接管旧库
  web/                   # 可选，内嵌到二进制也可
```

同时交付：

- tools/mockclient：协议级测试客户端；
- tools/fixturegen：测试地图、数据库和脚本 fixture 生成器；
- 数据库迁移/备份/恢复工具；
- Web 管理台；
- Dockerfile、systemd unit 或启动脚本；
- 协议、数据库、部署和运维文档；
- 差分测试、压测和 soak 测试报告。

### 2.5 能否直接替换原版服务端

分层面定义：

| 层面 | 能否直接替换 | 条件 |
|---|---|---|
| 客户端通信协议 | 可以 | 固定 mir2x@5826493 协议基线，mock 和真实客户端均不改协议 |
| SQLite 数据 | 可以 | 保持 22 张表及字段语义；迁移前备份；允许新增版本元数据表 |
| 地图和 Lua 脚本 | 可以 | 使用同一 map/mapbin.zsdb 和 script 目录 |
| 启动端口和常见启动参数 | 基本可以 | 提供端口、地图、脚本、数据库等兼容参数和配置映射 |
| 管理界面 | 不可以完全替换 | FLTK 改为 Web 管理台，功能覆盖但不做像素级复刻 |
| master/slave 多进程 | v1.0 不可以 | v1.0 单进程；v1.1 补分布式 |
| 真实世界内容 | 取决于数据 | 代码可替换；无合法 mapbin 时只能运行 fixture 测试世界 |

因此，v1.0 的目标是：**在单进程、相同协议、相同数据库结构和相同地图/脚本数据的条件下，可以替换原版服务端。** 这不是当前的口头保证，而是 Phase 8 差分测试、官方协议回放、旧库迁移和 72 小时 soak 通过后的验收结论。

如果未来真实客户端接入，仍建议预留 1-2 周兼容修正窗口，用于覆盖 mock 客户端未模拟到的隐含行为。

---

## 3. 参考基线与事实盘点

### 3.1 源码基线

| 项目 | 当前值 |
|---|---|
| 参考仓库 | E:\work\game\mir2x |
| 当前分支 | main |
| 参考提交 | 58264934b5562cd283db7dc7daddec496b2de86e |
| 工作区状态 | 已清理干净 |
| 上游远端 | https://github.com/etorth/mir2x.git |
| 客户端资源仓库 | 不需要；服务端不依赖完整 mir2x_assets |
| 服务端地图数据 | 仅 map/mapbin.zsdb；测试由 fixturegen 生成 |

### 3.2 服务端模块概览

主要目录和职责：

| C++ 区域 | 主要职责 |
|---|---|
| server/src/main.cpp | 进程入口、全局对象初始化、主循环 |
| server/src/server.* | 服务生命周期、地图加载、账号/角色入口、日志接口 |
| server/src/netdriver.*、channel.* | 客户端 TCP 接入、连接槽、发送队列 |
| server/src/receiver.*、dispatcher.* | 收包、解包、消息分发 |
| server/src/actor*.* | Actor Pod、Actor Pool、消息队列、协程调度 |
| server/src/servicecore*.* | 登录后的全局服务核心、地图路由、会话管理 |
| server/src/servermap*.* | 地图、格子、AOI、观察者、触发器、安全区、寻路 |
| server/src/serverobject*.* | 所有服务端 Actor 对象基类 |
| server/src/charobject*.* | 角色对象公共状态、位置、视野、移动 |
| server/src/player*.* | 玩家属性、背包、装备、魔法、任务、交易等 |
| server/src/monster*.* | 怪物状态、AI、攻击、掉落 |
| server/src/npchar*.* | NPC 对话、商店、脚本事件 |
| server/src/battleobject*.* | 战斗对象基类、生命、伤害、死亡 |
| server/src/buff*.* | Buff 定义、效果、触发和修正器 |
| server/src/quest*.* | 任务 FSM、任务状态、任务脚本接口 |
| server/src/*db.* | 角色、物品、聊天、拍卖、摊位、公会等持久化 |
| common/src/clientmsg.hpp | 客户端请求消息定义 |
| common/src/servermsg.hpp | 服务端响应消息定义 |
| server/src/actormsg.hpp | 服务间 Actor 消息定义 |
| server/script/ | 地图、NPC、任务、运行脚本 |
| server/src/server.sql | SQLite 数据库结构 |

### 3.3 协议与测试资产

现有测试可复用为行为基线：

- test_gridtrigger.cpp：地图触发器；
- test_guilddb.cpp：公会数据库；
- test_itemgrant.cpp：物品发放；
- test_luarunner.cpp：Lua 协程运行；
- test_playerquest.cpp、test_questdb.cpp、test_queststate.cpp：任务系统；
- test_questscripts.lua、test_magicquests.lua、test_taoistquests.lua、test_worldquests.lua：脚本级测试。

Go 复刻应将这些测试逐项迁移或建立等价测试，不应只依赖人工试玩。

---

## 4. 复刻目标与范围

### 4.1 v1.0 目标

v1.0 完成时应满足：

1. 保持现有 C++ 客户端协议字节兼容，mock 客户端可完整驱动服务端；
2. 登录、建角、进图、移动、战斗、物品、NPC、交易、聊天、组队、公会、任务等主流程可用；
3. SQLite 表结构和现有角色数据可迁移；
4. 地图二进制格式、记录表和 Lua 脚本接口兼容；
5. 行为以当前 mir2x 为基准，核心游戏结果与参考服务端一致；
6. 单进程可稳定承载约定的在线规模；
7. 有完整的日志、指标、管理命令、备份和故障恢复能力。

### 4.2 范围内

- 客户端 TCP 协议与消息处理；
- 账号、角色、会话、断线处理；
- 地图加载、AOI、寻路、移动、地图切换；
- 玩家、怪物、NPC、守卫、宠物、召唤物；
- 战斗、魔法、Buff、伤害、死亡、复活、经验、掉落；
- 背包、装备、腰带、仓库、邮件、拍卖、摊位、直接交易；
- 好友、聊天、聊天组、组队、公会、公会战、排行；
- 任务 FSM、任务数据库、任务脚本；
- Lua 运行时及服务端脚本 API；
- SQLite 数据访问、迁移、备份；
- Web 管理台、GM 命令、监控；
- tools/mockclient 协议驱动与场景测试；
- tools/fixturegen 测试地图、数据库和脚本 fixture；
- 兼容性测试、差分测试、压测和 soak 测试。

### 4.3 范围外

- 客户端复刻；
- 完整 mir2x_assets、贴图、音效、字体和表情资源；
- 地图编辑器、动画编辑器、资源查看器；
- 原 FLTK 管理界面的像素级复刻；
- 原版 1.45 旧服务端兼容；
- 上游 TODO 中尚未完成的历史功能；
- 商业化运营系统，如支付、渠道、封号平台；
- 反外挂客户端模块。

### 4.4 完整的验收定义

建议将「完整」定义为一组可测试的兼容性口径，而不是主观判断：

- 协议完整：所有客户端消息都有接收、校验、处理和响应路径；
- 功能完整：mock 客户端可完成全部已定义的玩家流程；
- 脚本完整：690 个 Lua 脚本可加载，代表性脚本行为一致；
- 数据完整：22 张表可读写，旧库可迁移；
- 行为一致：在固定输入和随机种子条件下，关键事件序列和数据库副作用一致；
- 运行完整：满足性能、稳定性、恢复和管理要求。

不建议承诺所有 C++ 实现的 bug、崩溃路径和 SQL 注入行为也完全一致。安全漏洞、崩溃、越权、可复制物品等缺陷应有记录地修正，并通过兼容开关控制需要保留的旧行为。

---

## 5. 八个待确认问题的处理建议

### 5.1 兼容基线

建议：以 mir2x@5826493 为 v1.0 基线，同时冻结协议头文件和关卡数据格式；不依赖客户端分支和完整资源仓库。

具体做法：

- 建立 compat/manifest.json，记录上游仓库、提交、协议快照、数据库结构、Lua 版本、SQLite 版本、Go 版本；
- 建立 compat/schema/，保存协议枚举、数据库结构、消息布局快照和校验和；
- tools/mockclient 与 C++ 协议头文件作为契约验证来源；
- tools/fixturegen 生成测试地图、数据库和脚本 fixture；
- 上游 main 更新只作为显式升级任务处理，不允许自动跟随；
- 每次升级兼容基线必须跑完整差分测试。

建议清单字段：

```json
{
  "upstreamRepo": "https://github.com/etorth/mir2x.git",
  "upstreamCommit": "58264934b5562cd283db7dc7daddec496b2de86e",
  "protocolSnapshot": "compat/protocol",
  "serverFixture": "tools/fixtures",
  "luaVersion": "5.4.x",
  "sqliteVersion": "TBD",
  "goVersion": "TBD",
  "protocolSha256": "TBD",
  "dbSchemaSha256": "TBD"
}
```

### 5.2 完整的范围

建议：v1.0 只复刻当前 mir2x 已有行为，不把上游 TODO 和旧版 1.45 未实现功能纳入 v1.0。

理由：

- 当前 TODO 中的很多功能尚未在上游实现，没有可对比的参考行为；
- 把未完成功能混入复刻，会造成范围不可控；
- 先达到当前版本功能对等，再以独立版本迭代新增功能更稳妥。

需要产出一份功能兼容矩阵，按以下状态跟踪：

- 未开始；
- 协议已支持；
- 服务端已实现；
- 已通过参考服务端差分测试；
- 已通过 mock 客户端端到端测试；
- 已验收。

上游 TODO 中的旧版功能作为 v1.1+ 候选需求，单独估算，不占用 v1.0 关键路径。

### 5.3 master/slave 多进程架构

建议：v1.0 先实现单进程多地图；内部接口和 UID/路由设计保持分布式可扩展，v1.1 再做 master/slave。

理由：

- v1.0 的主要风险是行为兼容，而不是横向扩容；
- 单进程版本更容易调试、差分测试和定位问题；
- 先冻结 Actor 边界、地图归属、全局服务接口，后续加 peer 不需要重写业务系统；
- 客户端协议不依赖内部 master/slave 协议，后续升级对客户端透明。

如果业务要求 v1.0 必须支持 master/slave，需要增加约 4-6 周，并增加分布式故障、重连、地图迁移和跨进程消息顺序测试。

### 5.4 管理界面

建议：用 Go 内置 Web 管理台替代 FLTK，功能覆盖优先，不做视觉复刻。

建议覆盖：

- 服务启动/停止/重启/维护模式；
- 在线玩家、地图、Actor 数量和资源监控；
- 账号查询、角色查询、GM 命令；
- 日志查询；
- 脚本控制台；
- 配置查看与受控修改；
- 数据库备份/恢复；
- 指标和 pprof 入口。

Web 端建议使用 Go embed + html/template 或轻量前端，不引入独立 Node 构建链。管理端口必须默认只监听本机或受保护内网，必须有认证和审计日志。

如果要求 FLTK 界面完全一致，需额外 4-6 周，且不建议纳入 v1.0 关键路径。

### 5.5 Lua 运行时处理策略

结论：v1.0 采用官方 Lua 5.4 桥接，不把 690 个 Lua 脚本翻译成 Go。

处理原则是「Go 实现核心玩法，Lua 继续承载内容脚本」：

- Go 负责网络、Actor、地图、AOI、战斗、物品、交易、持久化；
- Lua 负责 NPC 对话、地图事件、任务流程、刷怪和内容编排；
- Go 通过受控 API 暴露能力给 Lua；
- Script Engine 做成可替换接口，但不为 v1.0 增加第二套实现。

现有脚本不是简单的配置表：

- 约 690 个脚本、4.4 万行；
- 约 254 处 uidRemoteCall，分布在约 54 个文件中；
- 大量脚本使用 coroutine；
- 存在运行时 load() 动态执行代码；
- NPC、任务、地图刷怪都有联动的业务状态。

桥接架构：

```text
Go Actor / Game Systems
        |
        v
Script Engine Interface
        |
        v
Official Lua 5.4 VM (cgo)
        |
        v
server/script/*.lua
```

设计要求：

- 每个 Actor 一个独立 Lua state，禁止跨 goroutine 并发调用；
- Lua 调用固定在 Actor 所属 goroutine 内执行；
- Go 函数只通过白名单 API 暴露；
- 每个脚本调用都有超时、调用深度和内存限制；
- Lua 协程的 yield/resume 接入 Actor 定时器和消息调度；
- uidRemoteCall 按 Actor 请求/响应实现，支持超时和取消；
- 脚本错误转为可观测告警，不允许单个脚本崩溃整个进程；
- Lua 不开放 os、io、package 加载任意文件等能力。

为什么不翻译成 Go：

- uidRemoteCall 会把 Lua 代码字符串交给目标 Actor 动态执行，不能靠静态翻译解决；
- Lua 协程的挂起、恢复、超时和取消需要逐段重建状态机；
- 大量动态表、可变参数和运行时类型会降低 Go 代码的可读性；
- NPC 和任务文案需要长期维护，翻译后每次改内容都要重新编译和测试；
- 上游脚本更新无法继续复用；
- 690 个脚本的人工翻译和回归成本远高于桥接。

桥接与翻译对比：

| 维度 | Lua 桥接 | 翻译成 Go |
|---|---|---|
| 行为兼容 | 高，直接运行原脚本 | 低，容易漏状态和边界 |
| 初始工作量 | 较小 | 大，至少增加 3-6 工程师月 |
| 后续内容维护 | 高，可继续改 Lua | 低，每次都要改 Go |
| 性能 | 足够，脚本通常不在热路径 | 更高 |
| 类型安全 | 较弱，需要检查工具 | 强 |
| 构建部署 | 需要 cgo 和 Lua 库 | 纯 Go，较简单 |
| 上游同步 | 高 | 低 |

Lua POC 验收门槛：

1. Go 程序能启动 Lua 5.4 并加载现有脚本；
2. Go 能绑定函数供 Lua 调用；
3. Lua 能发起 Actor remote call 并取得结果；
4. Lua 协程能挂起、定时恢复和取消；
5. 脚本超时、异常和内存问题不会拖垮服务端；
6. 用代表性 NPC、地图和任务脚本跑通端到端场景。

如果项目以后硬性要求完全无 cgo，需要单独做纯 Go Lua 5.4 引擎 POC，预计增加 2-4 周，并保留 20%-30% 的进度风险。v1.0 不采用这条路线。
### 5.6 服务端数据策略

结论：不需要完整 mir2x_assets，也不需要客户端资源。

已核对 tools/installres/src/install_res.py：服务端安装只打包 map/mapbin.zsdb，不包含贴图、音效、字体、表情等客户端资源。

服务端完整运行需要区分两类数据：

1. 代码内置或仓库内数据：
   - 物品、怪物、魔法、Buff、地图记录表；
   - server/script 下的 NPC、地图、任务 Lua 脚本；
   - SQLite 表结构。
2. 可选的外部运行时数据：
   - map/mapbin.zsdb 地图二进制包；
   - 运营配置、账号数据库和备份。

测试策略：

- tools/fixturegen 生成小型 mapbin.zsdb，覆盖墙、可走/不可走格、安全区、传送点、刷怪点；
- tools/fixturegen 生成 SQLite 测试库和最小脚本 fixture；
- tools/mockclient 按真实协议驱动登录、移动、战斗、物品、交易等场景；
- CI 全部使用 fixture，不下载完整 assets；
- 需要真实地图的回归测试作为可选集成任务，不阻塞主开发。

生产部署：

- 如果运营方有合法地图包，通过配置加载；
- 没有真实地图包时，服务端仍可完整构建和运行测试世界；
- 地图包的版权和授权由运营侧负责，不影响代码复刻。

### 5.7 性能与部署目标

建议先按以下默认基线设计，最终由业务方确认：

| 指标 | 建议默认值 |
|---|---|
| 主部署系统 | Linux x86_64 |
| 开发系统 | Windows x86_64 |
| 单节点规格 | 8 vCPU / 16 GB / NVMe |
| 在线玩家 | 1,000 CCU |
| 同时加载地图 | 50-100 |
| 活跃怪物 | 10,000 |
| 地图逻辑 tick | p95 <= 50 ms |
| 登录完成 | p95 <= 2 s |
| 持续运行 | 72 小时 soak 无崩溃、无持续内存增长 |
| 发布方式 | 单进程，容器或 systemd |
| 热更新 | v1.0 不要求 |
| 不停服维护 | v1.0 不要求，支持优雅停机和快速恢复 |

如果要求 3,000-5,000 CCU、跨机部署、不停服热更新，需要在架构和排期中单独增加数据库扩展、分布式路由和版本兼容工作。

### 5.8 测试客户端方案

结论：交付只做服务端，不要求真实客户端参与开发。

建议在 tools/mockclient 实现一个 Go 编写的协议测试客户端：

- 按 C++ 头文件实现 CM/SM 消息编解码；
- 支持登录、建角、进图、移动、战斗、使用物品、交易、组队、公会、任务等场景；
- 支持多连接并发，模拟多个玩家；
- 支持录制、回放和断言服务端响应；
- 可生成协议 golden vector 和异常包；
- 可用于 CI、压测和 soak。

真实客户端联调不是 v1.0 的阻塞项。如果以后客户端分支完成，可作为可选集成测试接入，用于发现 mock 客户端未覆盖的协议细节。

风险说明：mock 客户端只能证明我们理解的协议正确，不能完全证明真实客户端的所有隐含假设。缓解方式是尽量从 C++ 协议代码自动生成结构测试，并保留可选的抓包回放接口。

---

## 6. 兼容契约设计

### 6.1 外部客户端协议

协议兼容是最高优先级契约，包含：

- 消息类型 ID；
- 消息头字节；
- 响应 ID 编码；
- 长度编码；
- 消息体 packed 布局；
- 位域顺序；
- 字节序；
- XOR64 压缩；
- 心跳和超时；
- 连接绑定、频道 ID 和玩家 UID 映射。

实施方式：

1. 建立 C++ 结构布局导出工具，输出结构体大小、字段偏移、位域信息和枚举值；
2. 生成 Go 的 wire 结构和编解码代码；
3. 对每条 CM_* / SM_* 生成 golden vector；
4. 使用 fuzz 测试拒绝畸形长度、截断包和非法响应 ID；
5. 使用 tools/mockclient 做协议回放，真实客户端抓包作为可选项。

### 6.2 数据库契约

- 保持 22 张表的字段名、类型、主键、外键、索引和触发器语义；
- 保持角色、物品、魔法、任务、公会等 ID 的含义；
- 所有资产变更必须使用事务；
- 建立数据库迁移版本表；
- 上线前必须用现有数据库副本做迁移和回滚测试；
- 禁止依赖数据库文件可被并发多进程随意写的假设，SQLite 采用单写者模型。

### 6.3 资源和地图契约

- 复用现有 Mir2xMapData 二进制格式；
- 复用 ZSDB/资源包读取逻辑；
- 保持地图 ID、格子尺寸、阻挡、安全区、传送点语义；
- 地图脚本继续使用现有 Lua 脚本；
- 所有资源读取必须有版本和校验和检查。

### 6.4 Lua 脚本契约

- Lua 版本与上游一致；
- 现有脚本原则上不改；
- 对确实存在平台相关或 C++ 特有的脚本，提供兼容层，而不是批量改写脚本；
- 脚本 API 兼容性按对象分类测试：
  - 全局 API；
  - 地图 API；
  - 玩家 API；
  - NPC API；
  - 任务 API；
  - 物品/背包 API；
  - 协程和远程调用 API。

### 6.5 ID 稳定性

物品、怪物、魔法、地图等记录 ID 是数据库和历史角色数据的一部分。必须做到：

- 生成记录表的顺序固定；
- 新记录只能追加，不能重排；
- 生成器输出有哈希校验；
- 每次升级对比 ID 表；
- 旧数据库在任何版本都可映射到正确记录。

---

## 7. 总体架构

### 7.1 设计原则

1. 先兼容，后重构：先通过协议和差分测试保证行为一致，再优化内部实现。
2. Actor 隔离：状态归属明确，避免全局锁和隐式共享。
3. 单进程优先：不为了分布式提前引入不必要的复杂度。
4. 边界清晰：网络、协议、Actor、世界、玩法、存储、脚本分层。
5. 可观测：每个关键路径有日志、指标和 profile。
6. 可回放：所有外部输入都能录制和重放。
7. 安全默认：参数化 SQL、输入校验、限流、管理接口鉴权。

### 7.2 进程结构

```text
                         +----------------------+
                         |        Admin Web     |
                         +----------+-----------+
                                    |
+-------------+   TCP    +----------v-----------+     +----------------+
| C++ Client  |--------->|   Network/Channels   |---->|  Service Core  |
+-------------+          +----------+-----------+     +--------+-------+
                                    |                          |
                          +---------v---------+       +--------v--------+
                          |   Actor Runtime   |<----->| Global Systems  |
                          +---------+---------+       | chat/guild/etc. |
                                    |                 +-----------------+
                         +----------v-----------+
                         | Map Shards / Actors  |
                         | AOI, movement, mobs  |
                         +----------+-----------+
                                    |
                         +----------v-----------+
                         | SQLite / Lua / Data  |
                         +----------------------+
```

### 7.3 Actor 运行时

建议模型：

- Mailbox：有界消息队列；
- Actor：单线程状态机，顺序处理消息；
- Request/Reply：带超时和取消的请求/响应；
- Timer：逻辑定时器和一次性延迟事件；
- Supervisor：panic 恢复、关闭传播和背压；
- Registry：UID 到 Actor 的路由；
- Scheduler：按地图/对象分片调度，避免所有 Actor 争抢一个全局队列。

业务代码禁止直接跨 Actor 修改状态，只能发消息或调用带超时的 request。

### 7.4 网络层

- net.Listener + 每连接读/写 goroutine；
- 单连接写队列，保证消息顺序；
- 连接槽和频道 ID 生命周期明确；
- 心跳、空闲超时、单包大小限制、每连接速率限制；
- 广播按地图和 AOI 批量聚合，避免每个玩家单独序列化；
- 断线后保留角色状态一小段时间，支持重连。

### 7.5 存储层

- SQLite WAL 模式；
- 一个写事务队列或单写连接；
- 读操作使用只读连接池；
- 对物品转移、金币、拍卖、公会创建等关键操作使用事务和幂等键；
- 启动时做数据库版本和完整性检查；
- 定期备份和 WAL checkpoint。

### 7.6 Lua 层

- 每个 Actor 一个独立 Lua state；
- Lua 调用在 Actor 所属 goroutine 内执行；
- Go 函数只通过受控绑定暴露；
- 脚本错误转为 Actor 级 warning/fatal，不让单个脚本崩溃整个进程；
- 脚本远程调用和协程挂起接入 Actor 调度器；
- 脚本执行时间、内存和调用深度有限制。

### 7.7 世界与地图

- 地图拥有格子、占用、观察者、地面物品、火墙和触发器；
- 玩家、怪物、NPC 作为 Actor 或地图拥有的实体；
- AOI 使用格子索引，避免全图扫描；
- 地图切换走统一的离图/入图事务；
- 路径查找、跳跃、传送和碰撞使用同一套规则；
- 所有地图事件带地图 UID、格子坐标和发生时间。

### 7.8 管理与运维

- 统一配置加载：文件 + 环境变量 + 命令行覆盖；
- 结构化 JSON 日志；
- Prometheus 指标；
- pprof；
- 管理 API 与 Web 管理台；
- 审计日志；
- 维护模式：禁止新登录、保留已有会话、可控停机；
- 备份和恢复脚本。

---

## 8. C++ 模块到 Go 包的映射

| C++ 模块 | Go 包建议 | 优先级 |
|---|---|---|
| main.cpp、server.cpp | cmd/mir2d、internal/runtime | P0 |
| netdriver.cpp、channel.cpp | internal/net | P0 |
| receiver.cpp、dispatcher.cpp | internal/net、internal/protocol | P0 |
| actormsg.hpp、actorpool.*、actorpod.* | internal/actor | P0 |
| clientmsg.hpp、servermsg.hpp | internal/protocol | P0 |
| server.sql、*db.cpp | internal/storage/sqlite | P0/P1 |
| servicecore.* | internal/game/world | P1 |
| servermap.* | internal/game/map | P1 |
| serverobject.* | internal/game/entity | P1 |
| charobject.* | internal/game/entity | P1 |
| player.*、playernet.*、playerop.* | internal/game/player | P1/P2 |
| monster.*、monsterop.* | internal/game/monster | P2 |
| npchar.*、npcharop.* | internal/game/npc | P2 |
| battleobject.*、damagenode.* | internal/game/combat | P2 |
| buff*.*、magicrecord.* | internal/game/combat | P2 |
| inventorydb.*、inventory* | internal/game/item | P2 |
| auctiondb.*、stalldb.*、deliverydb.* | internal/game/economy | P3 |
| chatdb.*、guilddb.*、friendtype.hpp | internal/game/social | P3 |
| quest*.* | internal/game/quest | P3 |
| serverluamodule.*、serverluacoroutinerunner.* | internal/script | P1/P3 |
| *window.fl、FLTK 监控 | internal/admin | P4 |
| tools/zsdbmaker 等生成器 | tools/ + 兼容数据读取 | P1 |

模块实施顺序以「可登录、可进图、可移动、可战斗、可持久化」为主轴，避免同时铺开全部系统。

---

## 9. 数据设计

### 9.1 数据库分组

22 张表按业务分组：

- 账号与角色：tbl_account、tbl_char、tbl_charvarlist；
- 物品与装备：tbl_belt、tbl_wear、tbl_inventory、tbl_secureditemlist；
- 魔法与配置：tbl_learnedmagiclist、tbl_playerconfig；
- 聊天与好友：tbl_chatgroup、tbl_chatgroupmember、tbl_friend、tbl_blacklist、tbl_chatmessage；
- 邮件与交易：tbl_delivery、tbl_auctionitem、tbl_stall、tbl_stallitem；
- 公会：tbl_guild、tbl_guildmember、tbl_guildinvite、tbl_guildwar。

### 9.2 数据一致性要求

- 角色登录时一次性加载核心状态，进入地图后再懒加载非关键列表；
- 背包、装备、金币、仓库、拍卖、摊位、邮件操作必须原子；
- 所有写操作带冲突检测和幂等键；
- 物品转移必须有唯一交易/操作 ID；
- 断线、宕机、重复请求后不能产生复制或丢失；
- 定期做 SQLite integrity_check。

### 9.3 数据迁移

- 使用版本化迁移脚本；
- 只允许向前迁移，禁止在玩家在线时自动改表；
- 迁移前自动备份；
- 迁移后校验行数、ID 范围、外键和关键聚合值；
- 支持迁移失败回滚。

### 9.4 资源与记录表

- 物品、怪物、魔法、地图记录由数据工具生成；
- Go 侧加载不可变记录表；
- 记录表版本和资源版本绑定；
- 启动时校验 ID 与名称映射；
- 提供兼容 API：按 ID 查记录、按名称查 ID；
- 测试地图由 tools/fixturegen 生成，生产地图包通过配置可选加载。

---

## 10. 测试与验收

### 10.1 测试层次

| 层级 | 内容 |
|---|---|
| 单元测试 | 协议编解码、SQLite 仓储、寻路、战斗公式、Lua 绑定 |
| 契约测试 | 所有 CM/SM 消息 golden vector、结构大小和字段偏移 |
| 集成测试 | mock 客户端驱动的登录、建角、进图、移动、战斗、交易、任务流程 |
| 差分测试 | 同一 mock 输入分别跑 C++ 服务端和 Go 服务端，比较事件和数据库副作用 |
| 模糊测试 | 截断包、超长包、非法字段、畸形 Lua/脚本输入 |
| 压力测试 | 并发连接、地图广播、战斗和物品操作 |
| Soak 测试 | 72 小时持续运行、内存、goroutine、文件句柄 |
| 迁移测试 | 真实数据库副本迁移、回滚、校验 |
| 安全测试 | 越权、重放、刷物品、SQL 注入、管理接口 |

### 10.2 差分测试方法

1. 固定 C++ 服务端、数据库快照、资源版本和随机种子；
2. 录制 tools/mockclient 输入序列；
3. 分别向 C++ 服务端和 Go 服务端回放；
4. 归一化时间戳、发送顺序中允许的非确定性字段；
5. 对比服务端发出的消息序列、玩家状态快照和数据库副作用；
6. 差异必须记录为兼容性 bug 或明确的安全修正。

### 10.3 验收标准

v1.0 发布前必须满足：

- 服务端协议消息覆盖率 100%；
- 登录到进入地图的完整链路通过；
- mock 客户端至少覆盖 50 个代表性游戏场景；
- 690 个 Lua 脚本全部加载成功；
- 22 张表全部有读写和迁移测试；
- 关键战斗、物品、经济、公会、任务流程通过差分测试；
- 无已知可复制物品、刷金币、越权路径；
- 通过 72 小时 soak；
- 通过目标在线规模的压测；
- 数据库备份和恢复演练通过；
- 管理接口有认证和审计；
- 发布包可重复构建，版本和资源哈希可追踪。

---

## 11. 工作拆分与排期

以下按 4 名 Go/服务端工程师 + 1 名自动化测试工程师估算，约 24 周。阶段允许重叠，但每个里程碑必须通过门禁才能进入下一阶段。

| 阶段 | 周期 | 主要工作 | 交付门槛 |
|---|---:|---|---|
| P0 基线与契约冻结 | 第 1-2 周 | 固定提交；搭建 mockclient/fixturegen；整理协议、数据库、Lua API 清单；技术 POC | 兼容性基线文档、fixture 测试环境 |
| P1 基础设施 | 第 2-5 周 | Go 工程骨架、CI、协议 codegen/codec、TCP、Actor 运行时、SQLite 层、配置和日志 | 协议 golden test、Actor 压测通过 |
| P2 登录与角色 | 第 4-8 周 | 账号、角色、会话、频道绑定、角色数据加载保存、基础地图进入 | mock 客户端可登录、建角、进图、退出后数据持久化 |
| P3 地图与移动 | 第 6-11 周 | 地图二进制、AOI、格子占用、寻路、动作广播、地图切换、触发器、安全区、基础刷怪和 NPC | 多个 mock 客户端互相可见，移动、换图、观察范围正确 |
| P4 战斗与魔法 | 第 9-14 周 | 攻击、伤害、死亡复活、经验、Buff、魔法、怪物 AI、宠物/召唤、掉落拾取 | 战斗场景与 C++ 服务端差分通过 |
| P5 物品与经济 | 第 12-17 周 | 背包、装备、耐久、NPC 买卖、仓库、邮件、拍卖、摆摊、直接交易 | 经济流程端到端通过，崩溃恢复后资产一致 |
| P6 社交与任务 | 第 15-20 周 | 好友、聊天、组队、公会、公会战、排行、任务 FSM、全部 Lua 脚本兼容 | 690 个脚本可加载，代表性任务链通过 |
| P7 分布式与管理 | 第 18-22 周 | master/slave/peer 路由、服务发现、重连、Web 管理台、指标、备份、安全加固 | 多进程部署、故障恢复、管理操作可用 |
| P8 兼容验收与发布 | 第 20-24 周 | 差分测试、mock 场景回归、负载测试、72 小时 soak、数据库迁移、文档和发布 | v1.0 验收通过 |

关键里程碑：

- M1 第 5 周：Go 服务端骨架 + 协议 codec + Actor 运行时；
- M2 第 8 周：登录、角色、持久化闭环；
- M3 第 11 周：地图、AOI、移动、换图；
- M4 第 15 周：战斗、魔法、怪物、掉落；
- M5 第 18 周：物品、经济、NPC；
- M6 第 21 周：社交、组队、公会、任务脚本；
- M7 第 23 周：管理、运维和必要的分布式能力；
- M8 第 24 周：兼容性验收和发布。

总体工作量约 22-26 工程师人月。粗略换算：

| 团队规模 | 预计周期 |
|---|---:|
| 1 名工程师 | 18-24 个月 |
| 2 名工程师 | 11-14 个月 |
| 3 名工程师 | 8-10 个月 |
| 4 名工程师 + 1 QA | 6-7 个月 |

如果要求 FLTK 管理界面也完全一致，增加约 4-6 周；如果 master/slave 分布式不是 v1.0 必需，可缩短约 3-4 周。

---

## 12. 技术风险与应对

| 风险 | 等级 | 应对 |
|---|---|---|
| Lua 协程与 Actor 调度语义复杂 | 高 | 第 2 周完成 remote call、yield/resume、任务 FSM POC；官方 Lua 5.4 + cgo |
| 客户端协议位域和压缩细节 | 高 | C++ 布局导出、golden vector、mockclient 回放、fuzz |
| C++ 隐式行为没有文档 | 高 | C++ 服务端作为 oracle，录制回放和差分测试 |
| fixture 地图过度简化 | 中高 | fixturegen 覆盖多种地形、阻挡、安全区和传送；真实地图回归可选 |
| ID 漂移导致旧角色物品错乱 | 高 | ID 表冻结、哈希校验、追加式生成 |
| SQLite 并发与复制物品 | 高 | 单写者事务、幂等键、崩溃恢复测试 |
| mock 客户端覆盖不足 | 中高 | 协议从 C++ 头文件生成；保留真实客户端抓包接口作为补充 |
| Actor 死锁和背压 | 中高 | 有界 mailbox、超时、取消、死锁检测和压力测试 |
| cgo 部署复杂 | 中 | 构建镜像统一工具链，Lua 封装隔离 |
| 范围蔓延到 TODO/旧版功能 | 中高 | 功能矩阵和变更控制，v1.0 冻结 |
| 性能不达标 | 中 | 从 P1 开始基准测试，关键路径 pprof |
| 分布式一致性 | 中 | 推迟到 v1.1，先冻结领域接口和 UID 路由 |

---

## 13. 仓库与工程规范

### 13.1 server-go 目录

```text
server-go/
  cmd/mir2d/
  internal/protocol/
  internal/actor/
  internal/net/
  internal/game/world/
  internal/game/map/
  internal/game/entity/
  internal/game/player/
  internal/game/monster/
  internal/game/npc/
  internal/game/combat/
  internal/game/item/
  internal/game/economy/
  internal/game/social/
  internal/game/quest/
  internal/storage/sqlite/
  internal/script/
  internal/data/
  internal/admin/
  compat/
  configs/
  test/
  tools/
  docs/
  deploy/
```

### 13.2 工程约定

- 所有业务方法接收 context.Context；
- 禁止裸 goroutine，统一由 Actor/任务组管理；
- 禁止在业务代码拼接 SQL；
- 公共错误必须可分类、可观测；
- 协议结构使用生成代码，不手写重复结构；
- 核心模块必须有单元测试；
- 每两周做一次依赖和协议基线检查；
- 所有架构决策记录在 docs/adr/。

### 13.3 开发环境前置条件

需要区分「构建环境」和「运行环境」。

Go 纯代码本身只需要 Go 工具链，不需要 MSYS2 或 MinGW。以下场景才需要 C 工具链：

- Go 通过 cgo 编译官方 Lua 5.4；
- 构建 C++ 参考服务端用于差分测试；
- 使用需要 cgo 的 SQLite 或其他 C 库。

Windows 上 Go 官方工具链只自带 Go 编译器。启用 cgo 时，Go 会调用外部 C 编译器和链接器，所以需要 MSYS2 UCRT64 中的 MinGW-w64 GCC。MSYS2 UCRT64 提供：

- gcc / g++；
- Windows UCRT 运行库头文件；
- Lua、SQLite 等 C 库的编译和链接环境；
- pkg-config、make、ninja 等构建工具；
- 构建 C++ 参考服务端所需的工具链。

推荐在 MSYS2 UCRT64 终端中安装：

```sh
pacman -S --needed \
  mingw-w64-ucrt-x86_64-toolchain \
  mingw-w64-ucrt-x86_64-cmake \
  mingw-w64-ucrt-x86_64-ninja \
  mingw-w64-ucrt-x86_64-pkgconf
```

安装后检查：

```sh
gcc --version
g++ --version
go env CC CXX CGO_ENABLED
```

Lua 5.4 建议静态链接进最终二进制。这样生产环境运行 `mir2d.exe` 时不需要安装 MSYS2；MSYS2 只是开发机和 CI 的构建工具。如果采用动态链接，则发布包必须同时带 Lua DLL 和对应运行库。

Linux 构建机使用发行版的 GCC 和 Lua 5.4 开发包即可，最终可构建静态或自包含二进制。

Phase 0 必须先完成：

- 安装并固定 Go 版本；
- 安装 Git、CMake、Ninja、Python；
- 安装 MSYS2 UCRT64/MinGW，或准备 Linux 构建机；
- 安装 Lua 5.4 开发库、SQLite 开发库；
- 验证 CGO_ENABLED=1 且能编译一个 cgo 程序；
- 成功构建 C++ 参考服务端，用于差分测试；
- 实现 tools/fixturegen，生成测试地图和数据库；
- 实现 tools/mockclient，打通登录和基础场景；
- 不需要完整 mir2x_assets 和真实客户端。

---

## 14. 需要业务方拍板的决策

| 编号 | 决策 | 推荐选项 | 不采纳的影响 |
|---|---|---|---|
| D1 | 兼容基线 | 固定 mir2x@5826493 + 协议快照 | 不可控漂移 |
| D2 | 完整范围 | 当前 mir2x 行为，不含 TODO/旧版 1.45 | 范围不可控，无法估算 |
| D3 | master/slave | v1.0 单进程，v1.1 分布式 | 增加 4-6 周 |
| D4 | 管理界面 | Web 管理台功能替代 | 增加 4-6 周 |
| D5 | Lua | 官方 Lua 5.4 + cgo 桥接，不翻译成 Go | 增加 POC 2-4 周和兼容风险 |
| D6 | 资源 | 测试用 fixturegen；生产地图包可选外部挂载 | 无真实地图时只能运行测试世界 |
| D7 | 性能目标 | 1,000 CCU / 8C16G 起步 | 无法定义验收 |
| D8 | 客户端基线 | tools/mockclient，不依赖真实客户端 | 协议覆盖不足风险增加 |

建议默认全部采纳推荐选项；如任一项要调整，在 Phase 0 评审时更新本文档和排期。

---

## 15. 下一步计划

在本文档通过后，Phase 0 的执行顺序：

1. 固定 C++ 服务端提交和协议头文件；
2. 搭建 C++ 参考环境和 Go 开发环境；
3. 建立兼容性清单、协议快照和数据库快照；
4. 完成四个技术 POC：
   - Actor mailbox + request/reply + timeout；
   - 服务端协议编解码 + XOR64 + golden vector；
   - 地图二进制和 fixturegen；
   - Lua 5.4 + 协程 + remote call；
5. 实现 tools/mockclient 登录冒烟测试；
6. 输出 Phase 1 的详细任务和可执行排期。

Phase 0 预计 2 周。Phase 0 结束前不进入大规模业务开发。

---

## 附录 A：功能兼容矩阵初始分类

- 账号与登录：登录、建角、删角、改密、查询角色；
- 会话：在线、离线、断线、重连；
- 地图：加载、换图、传送、触发器、安全区；
- 移动：走、跑、跳、路径、阻挡；
- 战斗：普攻、魔法、伤害、死亡、复活、经验；
- 怪物：AI、仇恨、召唤、宠物、守卫、掉落；
- 物品：拾取、丢弃、使用、装备、修理、耐久、品质；
- NPC：对话、商店、仓库、修理、事件；
- 经济：金币、拍卖、摆摊、直接交易、邮件；
- 社交：好友、黑名单、聊天、聊天组；
- 团队：组队、队长、招募、跟随；
- 公会：创建、邀请、成员、职位、公告、公会战；
- 任务：任务状态、脚本事件、奖励；
- 排行：等级、财富等排行；
- 管理：GM、日志、配置、备份、监控。

## 附录 B：Phase 0 交付物

- 基线清单和版本清单；
- 协议字段/枚举/布局快照；
- 数据库结构快照；
- Lua API 清单；
- C++ 参考服务端运行说明；
- tools/fixturegen 第一版；
- tools/mockclient 第一版；
- 四个 POC 报告；
- 兼容性测试框架第一版；
- 更新后的详细排期。

## 附录 C：文档评审结论

待填写：

- 评审日期：
- 评审人：
- D1-D8 决策结果：
- fixture 和 mockclient 范围确认：
- 性能目标确认：
- 是否批准进入 Phase 0：
