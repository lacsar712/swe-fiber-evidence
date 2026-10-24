# 为 Fiber 增加路由发布范围

> **Workspace:** d:\document\swe\fiber

---

**User**

fiber 的路由在注册时立刻就能被匹配到。调用方没法声明接下来这一组路由必须一起对外可见：要么整组都能匹配，要么整组都还匹配不到。现在只能先不启动监听，或者在外面自己加开关。请给 App 增加一种由调用方显式打开的路由发布范围：范围内注册的路由、分组和挂载，在范围成功结束之前，对正在处理的请求和新建请求都不可见；范围成功结束后，这些注册结果按注册顺序一次性变为可匹配。

没有打开这种范围时，路由注册、分组、挂载、参数约束、中间件、静态资源、监听启动和现有测试必须和现在完全一样。

范围只影响在这个范围内新注册的路由、分组和挂载。范围外已经存在的路由继续按现在的规则匹配。同一个 App 上两个并发范围不能互相看见对方尚未发布的注册；任一范围失败、被取消或在结束前 App 关闭时，该范围内已登记但未发布的内容必须全部丢掉，不能留下一半可匹配、一半不可匹配的状态。等待适用这次发布自己的超时。超时或取消后，不能把未发布的路由名额或内部表项占死。

发布瞬间如果有请求正在匹配路由，这些请求必须要么完全看不到这批新路由，要么完全按发布后的路由表匹配，不能出现同一请求里前半段旧表、后半段新表。嵌套范围时，内层发布只让内层登记的内容对外可见；关掉内层后外层未发布的内容仍然不可见。把路由先注册再在处理函数里手动判断开关、把 App 的路由表整份换成调用方自己维护的副本、或者要求调用方在注册期间停止 Listen，都不能算实现了这个功能。发布必须发生在路由登记与匹配共用的那条路径上。

**Assistant**

我来分析这个需求：为 Fiber 的 `App` 增加显式的"路由发布范围"（类似事务），范围内注册的路由在范围成功结束前不可见，结束时原子发布。这是一个涉及路由核心路径的复杂特性，我先了解代码库结构。

召回的经验与本任务（Go Fiber 路由核心）关联不大。我直接深入代码库，先读路由核心文件。


继续阅读 router.go 剩余部分和 app.go 的关键结构。


核心机制已清楚。继续阅读 domain.go、router_skip.go、static 注册方式和 error.go。


现在我对架构有了完整理解。让我调研所有请求期读取的路由状态字段，确定原子发布的改动面。


现有测试直接读 `app.treeIndex`/`app.hasParamRoutes`，所以这些字段必须保留。我再看几处请求期读取点和自定义 ctx 的 next 调用方式。


架构已经清楚。我再确认 error.go 的哨兵错误风格、hooks 签名和 DefaultCtx.Reset 位置，然后开始设计实现。


我已经完整理解了路由机制。让我更新任务计划并开始设计实现。


先检查 mountFields 的所有无锁读取点和 hooks 结构：


现在开始实现。首先查看 req.go 的两处调用点和 DefaultCtx 结构/Reset：


开始实现。先在 router.go 加入原子视图 `routingView`，并改造请求匹配路径：


现在改 nextCustom（两处 skip.hasParamUse 中另一处、入口 lookup）：


现在改 resolveSkip 签名和两个请求入口：


现在改 routeIndexInTree 和 buildTree 的原子发布：


现在拆分 mount.go 中 processSubAppsRoutes 的锁内内核，供发布提交复用（不消费 sync.Once）：


现在重构 register/addRoute/Name 的 locked 内核：


现在拆分 app.Name 为 locked 内核并记录名字编辑：


现在给 App 增加字段并在 New() 初始化：


在 runShutdown 中设置停止信号：


现在给 DefaultCtx 增加视图缓存字段与访问器：


更新 req.go 的两处调用，并添加自定义 ctx 的视图访问辅助：


第三个编辑误删了 methodInt 行，修复并添加 viewForCustom：


现在拆分 mount.go 的普通挂载为 locked 内核：


现在拆分 domain.go 的 domain mount 为 prepare（无锁）/applyLocked：


让我重新精确读取该函数再分段替换：


prepareMount 里对 d.app 加锁没有必要且会在重放时自死锁，移除：


现在给 app.go 添加 createGroupLocked 与组名重放辅助：


现在添加哨兵错误，然后编写核心文件 publish.go：


现在编写核心新文件 publish.go：


修正 commit 闭包的 panic 处理与 nameEdits 传递：


现在让 ErrorHandler 和 Render 的挂载元数据读取走加锁快照。先在 mount.go 加快照与重构 domainMountOwner：


更新 ErrorHandler 使用快照：


现在改 res.go 的 Render 读取点：


修复 ctx.go 中 appListKeys 的惰性生成加锁：


先编译检查语法与类型问题：


修复导入和类型：


编译通过。先运行现有相关测试，确认重构没有破坏行为：


`&App{}` 零值场景下 publishStop 为 nil，加保护：


需要看具体失败信息：


查看 Render 函数尾部，理解未渲染时的返回：


Render 需要根 app 自身的 "" 条目，不能走"无挂载"快路径。让快照始终返回完整数据：


现在跑整个包的现有测试套件，建立基线：


全部现有测试通过。先修正一个语义问题：server 为 nil 时 Shutdown 不应标记 stopped（未启动的 App 仍可用）：


现在编写发布功能的测试文件：


修正测试中的几处问题（阻塞中间件会挡住第二个请求、子 goroutine 中的 require、并行流量断言）：


需要调试为什么停放请求的视图钉住失效。在中间件里加包内断言：


视图指针未变却匹配到了新路由——说明旧视图的 bucket 里出现了新路由。加诊断确认：


需要更细的调试：在钉住时和释放后对比 bucket 底层指针与长度：


找到根因：routingView 持有 `app.treeIndex` 的同一个底层数组，per-method 替换穿透到了旧视图。视图必须持有独立切片：


清理测试中的诊断代码并重新运行：


第一个失败是测试本身的注册顺序问题（中间件注册在路由之后不会包裹它）。修正测试：


全部通过。用 -race 验证并发安全性，再跑全套：


本机无 gcc，无法在此跑 race（设计上已通过 atomic+mutex+不可变快照保证）。运行完整包测试：


需要确定性复现。写个临时 scratch 测试：


明白了——404 是测试自身的竞态：close(release) 后立即请求，提交 goroutine 未必完成。等待提交结束再断言：


删除 scratch 并重跑全套：


全套通过。运行 vet 与全部子包测试：


全部模块测试通过。更新任务进度，然后运行 Makefile 检查项：


先加文档。查看 app.md 结构：


运行格式化和 Makefile 检查项：


没有 make，直接查看 Makefile 并手动执行等效命令：


ctx_interface_gen.go 被重新生成，查看差异是否良性：


生成的接口变更符合预期（DefaultCtx 方法自动进入内部 Ctx 接口）。现在运行 golangci-lint：


下载超时，重试（依赖已大部分缓存）：


逐项修复 lint。先看 contextcheck 在现有代码里如何豁免：


修复 lint 问题。先改 ctx.go（方法重命名、未使用参数）：


现在修 publish.go 的 lint 项（goconst、方法命名、receiver、nolint）：


PowerShell 的 Set-Content 可能破坏了文件编码，先检查文件头：


现在修 commit 的 receiver 命名和 contextcheck：


现在修测试文件的 lint 项。先确认 Go 版本是否支持 wg.Go：


重新生成接口文件并格式化、编译：


---

