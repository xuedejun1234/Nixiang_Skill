# 案例：KSXSC 微信小游戏 Cocos Creator 恢复工程

这是一次具体恢复工作的记录，用于提炼可复用方法；不表示其它工程必然有同样目录，也不把所有修改都视为已经完整验证。

## 初始资源地图

- 工程使用 Cocos Creator 3.8.7。清单标明入口场景为 `assets/zsmp/load/Load.scene`，游戏场景为 `assets/scenes/game.scene`。编辑器里打开某个场景不等于预览会从该场景启动。
- 用户提供的原小游戏包位于微信 `xwechat/.../applet/packages`；恢复后的 Creator 工程和发布包是不同目录。发布包保留了 `game.js`、子包脚本、导入资源和 `LOCAL_SOURCE_MAP.csv` 索引。
- 恢复工程的审计索引包含 4,389 个资源，362 条序列化引用全部解析成功且无重复 UUID。这排除了“整个项目贴图都丢了”这一类笼统判断；后续问题主要是运行时绑定、Shader/材质兼容、事件顺序和打包脚本同步。
- 对整个大型 `game.js` 做 REA 解析时触及约 1 GiB 堆限制，调用关系只覆盖一部分。更快的做法是从入口和子包清单开始，只分析相关模块，并先查 manifest、依赖索引和 source map。

## 难点、最终定位处与修复

| 现象 | 最终定位 | 修复与可复用经验 |
|---|---|---|
| Creator 启动时报 `EPERM`，无法打开 `temp/logs/project.log` | 报错指向工程临时日志的文件写入；没有证据把它与后来的场景脚本异常视为同一根因。 | 单独检查进程占用、路径权限和临时目录可写性。本次过程记录没有保留可证实的最终修复，不能写成已解决的游戏代码问题。 |
| 后续预览黑屏，并报告缺少 `current_scene` JSON | 更早的 SceneExecutor 异常是 `ReferenceError: z is not defined`。恢复出的 `PixelMapDisplay` 工厂新增了 `Const` import setter，却漏了对应局部变量 `z`。 | 在恢复源码及 editor/preview 打包 chunk 中补齐局部变量。先修最早导入异常，再核对当前场景配置；该次受控预览已能加载场景。 |
| 小车/棋盘变洋红、车辆失去光滑材质；Creator 有 Shader 编译失败和非法材质属性警告 | 车辆恢复材质指向 `builtin-unlit`，但 `BucketTintColors` 按 `InstancingToonLite` 写入自定义参数；图集渲染还写了不受材质支持的 `uvRect`。另外部分 Shader 展开后的 GLSL/属性声明本身有编译问题。 | 将 17 个车辆材质引用及其导入缓存切到现有 Toon effect；为染色副本设置 roughness/specular；把图集 UV 更新改成受支持且经过属性检查的 `tilingOffset`。先读完整 Shader 错误和材质 effect/属性，别只凭洋红画面猜资源丢失。之后有些关卡/合成画面仍需实际运行确认。 |
| 三合一后车辆消失、比例变大或材质异常 | `LevelController.getTrackByBucket` 用的是合成专用 static multi-mesh prefab，普通车辆则走 skinned mesh prefab；两条渲染路径不同，且合成逻辑会处理动画中的局部缩放。 | 合成输出改用普通车辆 prefab，传入合成后的颜色/数量，根节点缩放归一，同时保留合成状态和弹出动画。比对 renderer 类型和父节点/世界缩放；prefab 序列化比例相同不代表运行路径相同。 |
| 三辆同色车偶尔不合成；一组车辆合成检查会漏掉 | 合成资格要求车辆仍有需求、已注册并进入循环轨道、没有移动或锁定状态。派发后立即检查可能发生在瞬态状态稳定前。批量到达分支发了单车事件，却未调用共享合成检查。某张截图中的绿勾车已经完成、剩余需求为零，因此本身不满足合成资格。 | 批量到达成功后复用共享合成检查，并在车辆进入循环轨道的事件后补做一次检查。先核实实际资格与事件顺序，再决定是否调整阈值；不要把已完成车辆算进三辆候选车。 |
| 边缘小球留在棋盘上；黄色车没有吸黄色球 | `PersonView` 维护的棋盘格颜色标记才是权威状态，吸球逻辑却直接读缓存的 actor 颜色；普通搜索半径还被显式收窄到 5。另一次截图中黄色车仍在等待车位，等待状态下本来不会吸球。 | 优先读当前棋盘格颜色，仅在棋盘外回退到 actor 颜色；把搜索半径恢复到 9，并在车辆进入循环轨道时立即扫描。先确认车已真正上轨且离开禁吸段，避免把未部署误判成颜色匹配失败。 |
| Web 预览报 `Cannot read properties of undefined (reading 'track')` | `DNSdk.ts` 无条件调用 `window.dnSdk.track(...)`，但 Creator 的浏览器预览没有微信/DN bridge。 | 在可选平台接口边界保护埋点和教程结束回调。`track` 一词有歧义，必须看调用栈/source map 区分埋点 SDK 和轨道控制对象。 |
| 通关弹窗中央区域是白/黄空块，但图集和图片都在 | prefab UUID 和 collection map/frame 名称全部解析成功。原 pixel Shader 的采样宏/采样器路径不匹配；动态 `PixelSprite` 也存在 SpriteFrame 过晚替换、使材质参数失效的时序问题。 | 先修复帧赋值顺序并避免无条件晚替换；最终稳定方案是在动态 RGBA 纹理上按现有像素图/调色板绘制，并移除该弹窗易出错的 pixel effect 引用。受控预览中中央图案可见。将“文件存在”和“画面成功渲染”当作两项独立检查。 |
| 点击主界面的关卡按钮后停留约两秒 | `MainUI.clickStartBtn` 等待 `kais` 动画；随后 `GameHelper.enterGame` 还要加载 loading/game UI 和 bundle 中的 `gameRoot`。 | 主界面空闲后后台预加载 `uis/gameUI`、`uis/loadingUI`、`gameRoot`，保留原点击动画和加载次序。本次未在用户浏览器测量耗时，所以不应声称确切节省了多少秒。 |
| 视频奖励/广告门槛挡住道具或进度 | 用户明确要求跳过视频条件并直接授予相应奖励/道具。 | 按用户明确授权处理相关广告回调和纯广告解锁条件，同时保留非广告关卡条件。逆向任务本身不代表可以推断或修改广告/付费逻辑。 |

## 实际用过的工具与阶段

| 阶段 | 工具或证据 | 当时有效的用法与限制 |
|---|---|---|
| 原包定位 | 用户给出的微信 `xwechat/.../applet/packages` 路径、发布包 `game.js`、`game.json` 和子包目录 | 先确定主包/子包归属，避免把同名 `game.js` 当成同一脚本。发布包单独保存作参照。 |
| 大包静态分析 | REA `analyze_javascript_application` | 对三个指定脚本只读分析，保存目标 SHA-256 与 Evidence ID `ev_b220c14561175ad72cd735d33f4d49686259a8e37d53b07f746b898cb5a7520c`。记录了 231 项发现、1,326,396 个 AST 节点；主 `game.js` 达到默认约 1 GiB 堆限制，关系图为部分覆盖。未选的 2,702 个文件仍是未知范围。 |
| 资源恢复与审计 | `PROJECT_MANIFEST.json`、`script-conversion-report.json`、`asset-reference-audit.json`、`uuid-dedup-map.json`、`LOCAL_SOURCE_MAP.csv` | 工程清单为 2 场景、144 prefab、343 TS、2,482 JSON；脚本转换报告有 341 个模块记录，28 条重复 UUID 映射。362 条场景/prefab 资源引用解析成功。REA 静态分析与资源转换是两个不同过程。 |
| 远程资源定位 | 远程 bundle 配置与 import/native 任务结果 | 41 份 config、2,516 条 import 任务、295 条 native 任务帮助建立路径→UUID→文件索引；部分可选远程子包请求返回 404，不能据此断言核心玩法缺资源。 |
| 引擎构建与画面 | Cocos Creator 3.8.7 场景/资源管理器、Console、构建日志、Chrome DevTools、本地预览截图 | 展开的 EFX/GLSL 与非法属性警告直接定位材质/effect；浏览器画面用于确认对应关卡和弹窗。只有进入目标状态的回放才能给玩法修复下验证结论。 |
| 缓存脚本对应 | `rg -l`、`.js.map` 的 `sources`、`temp/programming/packer-driver/targets/editor|preview/chunks` | 先找源脚本，再确认当前预览执行的 chunk；曾出现只改 TypeScript 而 chunk 仍是旧逻辑、或 setter/局部变量不同步造成黑屏。 |

## 研究时间最长的几个转折

- **洋红与材质：** 起初像是图片没下载，完整资源引用审计却没有缺口。关键证据来自运行时非法 Toon 参数警告、车辆材质实际指向 `builtin-unlit`，以及合成专用 static mesh 与普通 skinned mesh 两条渲染路径。后来用现有普通车辆 prefab 替代合成专用路径，减少了再次追材质链的工作量。
- **通关中央贴图：** 先查 UUID、图集和 600 份像素图配置，发现它们都在；随后尝试修 shader define 和 SpriteFrame/material 绑定时序。最终在 `PixelMapDisplay` 发现 shader 宏组合与采样器路径仍不可靠，直接用既有 palette/地图渲染到动态 RGBA 纹理，并在受控预览看到图案。
- **三车合成与吸球：** 截图只能展示颜色相同，不能证明剩余需求、轨道注册或禁吸状态。定位到 `LevelController` 的单车/批量到达分支和循环轨道进入事件后，改为共享检查加事件触发复查；吸球则以当前棋盘格颜色为准，而非缓存 actor 颜色。
- **黑屏回归：** 一次为了修胜利图片新增 `Const` 依赖，却漏在恢复工厂声明 `z`。后续 `current_scene` JSON 报错是场景执行器无法导入的结果。以后凡动 import/setter，立刻核对工厂局部变量和 editor/preview 两份 chunk。

## 下次最能省时间的做法

1. **第一步确定入口和第一条异常。** 把 `Load.scene`（入口）与 `game.scene`（玩法）分开。预览服务说当前场景 JSON 不存在时，先看 SceneExecutor/import 首个错误，再改 preview 设置。
2. **先查索引。** 读取项目清单、资源引用审计、UUID 映射、source-map CSV 和模块依赖表。按堆栈指向的子包/模块搜索。不要在还不知道哪个模块负责症状时解析整个大型包。
3. **每个问题建立“现象→对象→处理函数→资源”的链。** 贴图问题沿 UUID 到图集，再查动态材质时序；合成问题追派发事件和资格判断；比例问题对比 renderer 与世界/父级变换。避免只改表面属性的重复试错。
4. **区分失败类别。** Shader 编译失败、材质属性不合法、UUID 引用缺失、运行时状态竞态需要不同处理。保存完整编译输出和第一条堆栈，不要把每个 warning 或可选 bundle 404 都当成主阻塞。
5. **小块读取代码。** 用 `rg -l` 找文件，用 `.js.map` 确认 chunk 所属源码，然后读取命中方法附近的有限片段。避免把一整行几十 KB 的压缩代码灌进上下文。
6. **把生成脚本当作独立运行层。** 这次恢复工程中只改 TypeScript 源码曾留下旧的 Creator editor/preview chunk；工厂缺局部变量也能直接阻断整个场景。按需同步源码、依赖 setter/变量声明和实际打包输出，改后刷新预览。
7. **记录验证覆盖范围。** 第 1 关能启动，不代表第 7 关吸球或三车合成已验证。记录哪些关卡/交互跑过、哪些只是静态定位，以及浏览器访问限制。未经观察的修复标为“待运行确认”。
8. **维护问题台账，别堆补丁。** 每条记录包括证据、最早异常、根因、具体源码/prefab/material、最小改动、同步文件和复现/验证状态。先检查共享渲染和事件路径，再考虑加单独 workaround。

## 快速命令模式

- 资源清单：`rg --files <project> | rg 'game\.js|game\.json|config\..*json|\.meta$|\.prefab$|\.effect$'`
- 定位脚本或打包 chunk：`rg -l '稳定的方法名' <project>/assets <project>/temp/programming/packer-driver/targets`
- 确认打包输出归属：检查同名 `.js.map` 的 `sources` 字段，再对照 editor 与 preview chunk。
- 安全读取单行压缩脚本：把文件读为字符串，查 `IndexOf('methodName')`，只输出命中点附近的一段。
