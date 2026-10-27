---
name: mini-game-reverse
description: 对微信、抖音等小游戏发布包进行资源索引、脚本与行为逆向、工程重建和针对性修复。需要从原始包快速定位关键资源、理解压缩脚本或验证恢复工程时使用；已有完整源码的普通代码分析无需启动逆向工具。
---

# 通用小游戏逆向模板

从一个具体问题出发，建立「原始发布包 → 资源/模块索引 → 静态与运行证据 → 恢复工程 → 验证结果」的可追溯链。首次使用时复制 [调查记录模板](assets/investigation-template.md) 到当前项目并填写，不必每次照搬全部阶段。

## 先确定目标和证据边界

- 记录原包位置、散列、平台、引擎/版本；将原包、解包工作目录、恢复工程、预览/构建输出分开。以原包作为行为与资源身份的参照。
- 列出用户想回答的具体问题，并为每个问题选证据：包/清单证明“存在和归属”，代码证明“可能执行的路径”，受控运行证明“实际发生”。截图仅证明画面上的状态；静态分析不等于运行观察。
- 先检查已有报告、索引、REA Evidence 和日志；复用相同目标与版本的结果。不要重新全量分析已解析的大包。

## 最短定位路径

1. **识别容器和入口。** 查看小游戏主清单、入口脚本、分包/远程包声明、引擎标记及启动场景。容器格式支持时用 REA 或相应格式工具取目录；解包后用 `rg --files` 建文件清单。不要根据文件名猜入口。
2. **按 manifest 建资源图。** 优先读 bundle 配置中的 path、UUID/资源 ID、pack、版本/哈希与依赖映射，再按「画面或错误中的路径/UUID → 所属 bundle → import/native 文件 → prefab/场景/图集/模型」追查。对 Cocos 读取 `config.*.json`、`.meta`、`import/`、`native/`；其它引擎使用其真实的 manifest/资源 ID 结构，不套用 Cocos UUID 规则。
3. **只分析相关脚本。** 先用 `rg -l`、source map、模块依赖索引定位入口/事件处理器。需要原始发布代码的独立静态证据时，使用 REA `analyze_javascript_application`，优先精确选择源文件；得到 Evidence ID 后用 `inspect_analysis_view` 翻到目标模块，再针对未解问题调用 `trace_application_feature`。大型脚本的 `detail: "summary"` 只缩短返回内容，不缩短分析工作；真正控规模要选 `source_paths` 或缩小目标。
4. **复现精确状态。** 能访问目标运行时且任务需要时，观察第一条异常、调用栈、请求、事件顺序与画面。REA 可先 `list_browser_targets` 再做被动观察；需要代理点击/导航时用 `capture_browser_scenario` 声明动作。也可用 Creator/浏览器控制台及网络面板。区分“用户操作的运行”与“代理执行的运行”。
5. **连到根因。** 追踪输入/事件 → 处理函数 → 状态资格 → 资源或渲染 → 完成/失败条件。分开检查资源缺失、引用正确但绑定失效、Shader 编译、材质属性不匹配、平台桥缺失、坐标/缩放、异步时序和事件漏检。先修最早的阻塞异常。
6. **最小修复并同步运行层。** 修改恢复工程的源文件与必要的资源元数据；若预览实际运行 Creator 生成的旧 chunk，还要定位并同步 active editor/preview chunk。只有审计证实引用错误时才改 UUID/导入缓存。保留原包证据及修改前后的映射。
7. **验证到目标功能。** 用用户报告的关卡和交互复测，查看控制台及画面；记录覆盖范围、未覆盖状态和剩余问题。启动成功不等于目标玩法已修好；未跑通的修复标为待验证。性能问题计时后再报告改善幅度。

## 工具选择

| 任务 | 优先工具 |
|---|---|
| 路径、脚本、UUID、错误文本定位 | `rg --files`、`rg -l`，随后短片段读取；JSON/CSV 用 PowerShell、Python 或 Node 解析并生成索引 |
| 发布 JS 的独立静态证据 | REA `analyze_javascript_application` → `inspect_analysis_view` → 有明确种子时 `trace_application_feature`；保留 Evidence ID、散列和覆盖限制 |
| 原包/资源容器清单 | 格式适配的解包器；REA `open_binary`/`inspect_artifact` 仅用于其支持的归档格式，不假设它能直接解每种小游戏专有包 |
| 运行行为、请求、截图 | 浏览器/小游戏开发工具控制台；REA `list_browser_targets`、`observe_web_session`、`capture_browser_scenario`、`capture_web_screenshot`，按可用工具和授权选用 |
| 恢复工程构建与修复 | 对应引擎编辑器/构建日志、预览与 source map；源码、序列化资源和实际执行 chunk 分别核对 |

REA 的具体用法、资源图建立方式和常见误判见 [工具与定位手册](references/tool-playbook.md)。本次 Cocos 工程的困难问题、最终定位处和修复证据见 [案例记录](references/case-study.md)。