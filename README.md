# 程序设计集训一体化训练平台 · CJU ACM

重庆交通大学 ACM 集训队的一体化训练平台。围绕**训练监控 → 算法薄弱点分析 → 每日一题 → 赛程备赛**闭环，
并配套**题面器**、**本地大模型驱动的出题器**、**可视化题解工坊**（题解 → 可继续修改的 PPTX/PDF）
与 **ICPC 滚榜**（成绩单 xlsx → 大屏逐个揭晓）。

## 两种客户端形态（同时提供，互不替代）

| 形态 | 入口 | 技术 | 适用场景 |
| --- | --- | --- | --- |
| **B/S**（浏览器） | `启动平台.bat` → <http://127.0.0.1:5190> | React + Vite 前端 + Python HTTP 后端 | 多人共享、跨设备、无需安装；**总览 + 训练/创作/赛事三子系统，创作中心页内嵌题面器 / 出题台原生界面** |
| **C/S**（桌面端） | `启动客户端.bat` | **Electron 壳**：原生窗口里跑同一份 Web 平台 → **题面器 / 出题台原生界面直接内嵌**（`Ctrl+7` / `Ctrl+8`） | 想要「桌面 App」体验、不想开浏览器 |
| 经典 C/S（备用） | `start_client_classic.bat` | tkinter 桌面客户端（零第三方依赖）· Steam 风格 + DPI 感知 + 命令行式创作面板 | 无 Electron 运行时的极简环境、离线排障 |

三者**共用同一套业务核心**（`platform/server/trainplatform/`）：

```
trainplatform/engine.py      核心门面（不依赖 Web 框架，两边都调它）
      ├── run.py             B/S / Electron：包成 HTTP 服务（/api/*）
      └── client/app.py      经典 C/S：桌面端直接 import（可离线）
```

> Electron 壳不重写任何界面：它拉起本地 B/S 服务端，开一个原生窗口加载平台页面，
> 所以桌面端和网页端**永远是同一份功能**（题面器、出题台的 iframe 内嵌也在其中）。
> 经典 tkinter 客户端默认走 `Engine` 直连本地数据，**后端没启动、甚至没网络也能查看
> 训练数据**；详见 `docs/方案与实测结论.md`。

## 界面结构：总览 + 三个子系统

平台不再把所有功能平铺在一个侧栏里，而是按**子系统**组织（默认进入 `#/home` 工作台）：

| 入口 | 包含 | 一句话 |
| --- | --- | --- |
| **总览（工作台）** | 3 个子系统入口、最近使用、同步状态 | 按子系统进入：训练、创作、赛事 |
| **训练中心** | 训练总览 / **队员管理** / 算法画像 / 每日一题 / 训练任务 / 成长趋势 / 队员档案 | 看训练数据、维护名单、找薄弱点、布置训练 |
| **创作中心** | 题面器 / 出题台 / 题解工坊 | 出题、写题面、做题解 |
| **赛事中心** | 赛程日历 / ICPC 滚榜 | 看赛程、办颁奖滚榜 |

- 每个子系统有独立入口页，用卡片说明每个工具「干什么 / 什么时候用」；
- 侧栏只显示当前子系统的菜单，面包屑可随时回总览/子系统；
- 页面顶部的标题与描述换成了一句人话，不再出现大段技术术语；
- 旧 hash 路由（`#/team`、`#/solution` 等）全部保留，深链接与 Electron 快捷键不变；
- 设计与验收见 [`docs/界面优化与子系统化.md`](docs/界面优化与子系统化.md)；
  静态预览（直接双击打开，可切换主题）：[`ui-home-preview.html`](docs/examples/ui-home-preview.html)（总览）、
  [`ui-training-hub-preview.html`](docs/examples/ui-training-hub-preview.html)（训练中心入口）。

### 修改训练队员名单（免重启）

平台读取的名单文件是 **`platform/data/members.xlsx`**（Excel/WPS 直接改；
如果删掉 Excel，会回退到旧的 `members.json`）。Excel 列：
`学号 / 姓名 / 年级 / 学院 / 洛谷ID / 监控 / 洛谷UID / 练习赛得分 / 备注`。

1. 直接编辑保存 `platform/data/members.xlsx`（增删人员、改洛谷 ID、改「监控」等）；
2. 回到平台，点右上角 **「↻ 重新读取名单」**；或者等约 20 秒，顶部出现
   **「人员文件已在磁盘上更新」** 提示后点「立即重新读取名单」；
3. 平台会重新读取该 Excel 并**刷新整页**（训练总览 / 算法画像 / 任务 / 队员档案全部用新名单），
   **不需要重启服务**。

> 旧版本后台没有这个按钮/接口时，重启一次平台即可升级；
> 也支持旧的 `members.json`（数组或 `{"members":[...]}`），但 Excel 优先。
> 文件里「监控」列填 是/否；洛谷ID 可留空（该队员只保留名单、不同步）。
> 详细字段、常见问题见 [`docs/人员名单维护.md`](docs/人员名单维护.md)。

### 队员信息管理系统（可视化维护名单）

训练中心 → **队员管理**（`#/roster`）：不打开 Excel 也能维护名单。

- 列表：搜索（姓名/学号/洛谷ID/学院）、按年级/学院/监控筛选、只看无洛谷ID；
- 增删改：新增/编辑队员（学号/姓名必填、监控开关、uid/练习赛得分/备注），删除；
- 监控开关：行内直接切换；
- Excel：导入名单（替换全部 / 按学号合并）、下载当前名单 Excel、重新读取磁盘文件；
- 统计与校验：总人数 / 监控中 / 无洛谷ID / 年级分布 / 学院分布，缺洛谷ID、重复学号/洛谷ID 警告；
- 保存即写回 `platform/data/members.xlsx`（并同步 JSON 镜像），每次写前自动备份到
  `platform/data/members.backups/`（保留最近 30 份）；页面加载时的 sha1 会随保存请求带回，
  若名单已被外部修改会返回 409 并提示重新读取，避免覆盖 Excel 里的新改动。
- 后端：`trainplatform/members_admin.py` + `members_file.write_members_xlsx`（纯标准库写 xlsx）；
  接口：`/api/members/save`、`/import`、`/export`、`/<id>/delete`。
- 设计与 API 契约：[`docs/队员信息管理系统.md`](docs/队员信息管理系统.md)；
  静态预览（真实 33 人）：[`ui-roster-manager-preview.html`](docs/examples/ui-roster-manager-preview.html)。

## 打包成 exe（免装 Python 分发）

桌面端可打包成 Windows 可执行文件，**目标机器无需安装 Python**。

```powershell
python packaging\build_exe.py --bootstrap   # 首次：装构建依赖（PyInstaller）
python packaging\build_exe.py               # 构建 5 个 exe 变体 + 组装 Electron 客户端
python packaging\build_electron.py          # 只重新组装 Electron 客户端（壳改了用这个）
```

产物（`release/` 目录）：

| 产物 | 体积 | 用途 |
| --- | --- | --- |
| `CJU-ACM-Client-Electron-1.0.0/` | ~340 MB | **推荐**。Electron 桌面客户端：双击即用、题面器/出题台原生界面内嵌 |
| `CJU-ACM-客户端-1.0.0/`（目录版） | 25 MB | 经典 tkinter 客户端（轻量备用）：免解压、启动快 |
| `CJU-ACM-客户端-1.0.0.exe`（单文件） | 10 MB | 经典客户端单文件版；受限环境可能无法启动 |
| `CJU-ACM-客户端-诊断-1.0.0/` | 25 MB | 用户报障时运行，能看到输出 |
| `CJU-ACM-服务端-1.0.0/` | 18 MB | **B/S 形态**：双击起服务，自动开浏览器（API + 前端页面一体） |

> Electron 客户端会复用本机已有的 Electron 运行时（默认找
> `D:\2026学习\xcpc-statement-generator\node_modules\electron\dist`，也可用
> `ELECTRON_DIST` 或 `--electron-dist` 指定），不会重复下载；
> 服务端 exe 直接复用 `release/CJU-ACM-服务端-1.0.0/`。

### Windows 拦截未签名 exe 时（SmartScreen / 智能应用控制 / Defender）

Electron 与 PyInstaller 产物**未做代码签名**，部分 Windows 11 会拦。
三种处理方式（按推荐顺序）：

| 方式 | 做法 | 说明 |
| --- | --- | --- |
| **浏览器窗口版**（免未签名 exe） | 双击 `启动客户端.bat`（自动回退）或 `启动客户端-浏览器窗口.bat` | 只用系统 Python（PSF 签名）+ 微软 Edge（签名）开一个**无地址栏的应用窗口**，界面与 Electron 版相同；题面器/出题台照常内嵌。日志 `platform/data/out/app-window.log` |
| 解除拦截 | 右键 exe → 属性 → 勾「解除锁定」；或 SmartScreen 点「更多信息→仍要运行」；或 Defender 排除项加 `release\CJU-ACM-Client-Electron-1.0.0` | 适合只是 SmartScreen/Defender 提示的情况 |
| 关闭智能应用控制 | Windows 安全中心 → 应用和浏览器控制 → 智能应用控制 | **不可逆**（关闭后只能重装系统才能再开），建议优先用浏览器窗口版 |

> `启动客户端.bat` 现在的行为：先启动 Electron 客户端；**8 秒内没检测到进程**
> （通常就是被 Windows 拦住）就自动切到浏览器窗口版，所以你只需双击它一次。

> 若系统开着「智能应用控制」，未签名程序会被直接禁止运行，前两种都无效，
> 走「浏览器窗口版」或 B/S 网页端即可。

打包后需要知道的四件事：

1. **数据随 exe 放在哪**：首次运行在 exe 同级生成 `platform/data/`；
   若该目录不可写（如装在 `Program Files`），自动改用
   `%LOCALAPPDATA%\CJU-ACM-TrainPlatform\data\`。
2. **无控制台版会写日志**：正式版双击不弹黑框，所有输出写进
   `platform/data/client.log`（`--log-file` 可覆盖）。
3. **推荐目录版而非单文件版**：单文件版启动时要把 DLL 解压到 `%TEMP%`，
   少数受限环境（锁定 TEMP、部分杀软）会报
   `Failed to extract VCRUNTIME140.dll`；目录版不做解压，不受影响。
4. **打包后要重新同步数据再分发**：用户拿到的是你打包时带上的快照，
   否则界面是空的。

排障三件套（打包产物里可直接运行）：

```powershell
CJU-ACM-客户端.exe --diagnose    # 打印路径解析与数据可见性（排查「数据为空」）
CJU-ACM-客户端.exe --selftest    # 43 项自检
CJU-ACM-客户端.exe --smoke       # 真实建窗 + 三主题切换的 GUI 冒烟
```

完整机制（路径解析、`TRAINPLATFORM_DATA_DIR` 时序坑、hiddenimports 清单、
分发与签名注意事项）见 [`docs/打包为exe.md`](docs/打包为exe.md)。

## 自建化扩展（P0–P5：题库 / OJ / 反馈 / 比赛）

在原有训练分析之外，平台新增四个自建域，全部纯标准库、经 `engine.py`/`run.py` 暴露：

| 域 | 代码 | API / 数据 | 文档 |
| --- | --- | --- | --- |
| P0 题库 | `trainplatform/problems/` | `/api/problems*`；`platform/data/problems/<id>/`（状态机 draft/published/archived、版本不可静默删除、目录/zip 导入导出**只校验不执行**） | [自建题库设计与数据模型](docs/自建题库设计与数据模型.md) |
| P1 OJ | `trainplatform/judge/`、`submissions.py` | `/api/judge/status`、`/api/submissions*`；C++17/20/Python3、token/line/float checker、源码策略、逐点结果/重判 | [评测机设计与数据模型](docs/评测机设计与数据模型.md) |
| P2 可选后端 | `trainplatform/judge/backends/`、`scheduler.py` | 本地（Job Object 训练级）与 external 适配骨架；默认 external 关闭，**不声称强隔离** | 同上 |
| P3 双源 | `engine.py`、`tasks.py` | `tasks.problems[]` 支持 `{source: local|luogu, id}`；旧 `problemIds` 兼容；`?` 不判没做 | [训练闭环双源与数据口径](docs/训练闭环双源与数据口径.md) |
| P4 反馈 | `trainplatform/feedback/` | `/api/feedback/member/<key>`、`/team`、`/annotate`；260 维特征、基线对外、纯 Python 1D/2D CNN shadow、门禁回退、解释 | [模型反馈设计与门禁](docs/模型反馈设计与门禁.md) |
| P5 比赛 | `trainplatform/contest/` | `/api/contests*`；建赛/绑本地题/报名/封榜/榜单/rollout 兼容 xlsx 导出 | [比赛与滚榜联动](docs/比赛与滚榜联动.md) |

**诚实边界：** P1/P2 沙箱是训练级 best-effort，不是强隔离；external 后端默认关闭且未验证；
P4 CNN 默认 `trained=False`，未达门禁自动回落基线；模型效果与吞吐等新指标均标注“待实测”。

需求与可行性（自行整理，原始附件缺失）：[需求分析](docs/需求分析-自建化集训平台.md)、
[可行性分析](docs/可行性分析-自建化集训平台.md)。



> Electron 客户端的界面就是 Web 平台本身（见上文）；
> 这一节讲的是**经典 tkinter 客户端**（`start_client_classic.bat`），按
> **Steam 客户端** 的视觉语言在 2026-10 重做过：

- **布局**：顶栏（品牌 / 数据源徽章 / 主题切换 / 刷新 / ☰ 菜单）+ 左侧导航栏
  （6 个模块 · `Ctrl+1~6` + B/S 入口 / 帮助 / 关于 + 数据源状态）+ 内容区
  （渐变页头 + 圆角卡片 + 状态徽章 + 底部状态栏）。
- **去掉了两处「复杂」来源**：Windows 原生菜单栏（改成顶栏 ☰ 弹出）与
  Notebook 标签页（改成左侧导航），页面内也不再重复放「刷新」按钮。
- **暗色基线**：`#171a21` 外框 / `#1b2838` 内容底 / `#16202d` 卡片 /
  `#1a9fff` 强调蓝；浅色主题是同一套语义的浅色版本。窗口**标题栏**也会
  随主题染深（Win10 1809+，`DwmSetWindowAttribute`），不再被系统白条割裂。
- **图标是矢量自绘**：`widgets.Icon` 用 Canvas 画 20 种图标，不依赖 emoji /
  图标字体，DPI 变化下不会出现方块或模糊。

### 「像素低」的根因与修复（改动前必读）

旧版界面不是分辨率不够，而是**进程没有声明 DPI 感知**：150% / 200% 缩放下
Windows 会把 96 DPI 的窗口整体位图拉伸，文字与线条全部发虚。现在：

1. `theme` 模块**导入时**就调用 `enable_dpi_awareness()`（per-monitor v2 →
   per-monitor → system 逐级降级，必须在 `Tk()` 之前）；
2. 建窗后用 `theme.sync_from_root(root)` 以 Tk 实际看到的 DPI 校准，并显式设置
   `tk scaling`；
3. **字体用点字号**（`theme.SIZE_*`，Tk 自动随 DPI 放大），
   **间距/宽高用逻辑像素**（`theme.sp()`，96 DPI 基准 × 当前缩放）。
   新代码不要写裸数字，否则会出现「字大间距小」的错位。

可用环境变量 `CJU_UI_SCALE=1.0 / 1.25 / 2.0` 强制覆盖缩放，用于在没有
对应显示器的机器上回归不同布局。

```powershell
# 逐页 + 逐主题截图（同时验证 DPI 输出、控件数、三主题切换）
python platform\client\app.py --smoke --offline --no-live --screenshot-dir .tmp-run\shots
$env:CJU_UI_SCALE='1.0'; python platform\client\app.py --smoke --offline --no-live   # 100% 布局回归
```

## 创作与赛事工具（子系统：创作中心 / 赛事中心）

两个原本只有命令行的工具，现在都以**原生 Web 界面 iframe 内嵌**的方式接进了平台：
**总览 → 创作中心 → 题面器 / 出题台**；**总览 → 赛事中心 → ICPC 滚榜**。
界面已按「总览 + 训练中心 / 创作中心 / 赛事中心」三个子系统重组，侧栏只显示当前子系统菜单。

- **B/S 网页端**（`启动平台.bat`）里能直接看到；
- **Electron 桌面客户端**（`启动客户端.bat`）窗口里也是同一份页面，因此同样原生内嵌；
- 经典 tkinter 客户端没有浏览器内核，无法真正内嵌，保留原有命令行式面板。

> 这两个工具本身就是 Web 应用（xcpc-statement-generator 是 React + Typst WASM；
> 出题台是 ACM Agent 的 `toolkit/webui.py`）。用 iframe 显示它们，功能 100% 保留
> （Polygon 导入、实时预览、导出 PDF、AI 出题…），不需要重写一遍。

| 界面 | 内嵌的是谁 | 后端怎么接 |
| --- | --- | --- |
| 题面器 | `xcpc-statement-generator/dist`（外部项目的前端产物） | `tools_bridge` 用独立端口托管静态资源（带 COOP/COEP、wasm MIME、SPA 回退），`/api/tools/status` 把 iframe 地址给前端 |
| 出题台 | ACM Agent 的 `toolkit/webui.py` | `tools_bridge` 按需起子进程（`--no-browser`），`/api/tools/setter/start` 启动并返回 URL，`/api/tools/setter/stop` 停止；启动时注入 `DSH_BIN` 并追加 `--dsh`，指向本机模型执行器（见下） |

**自动探测**：题面器默认找 `D:\2026学习\xcpc-statement-generator\dist`、仓库同级
`xcpc-statement-generator/dist`、`~/xcpc-statement-generator/dist`；出题台默认沿
`platform/data/problemgen.json` 的 `toolkit.root` 找，再兜底常见路径。
找不到时页面上会列出探测过的路径。

**手动指定**（可选，`platform/data/tools.json`）：

```json
{
  "statement": { "dist": "D:/path/to/xcpc-statement-generator/dist" },
  "setter":    { "webui": "D:/path/to/ACM Agent/toolkit/webui.py" }
}
```

环境变量优先级更高：`TRAINPLATFORM_STATEMENT_DIST`、`XCPC_STATEMENT_ROOT`、
`TRAINPLATFORM_WEBUI`、`TRAINPLATFORM_PYTHON`。

**出题台的「AI 写题面与代码」接的是本机模型（2026-10-28 起）**：`tools_bridge`
启动出题台子进程时额外注入环境变量 `DSH_BIN`，指向新增的 dsh 兼容执行器
`platform/tools/setter_ai.py`（实现在 `platform/problemgen/setter_ai.py`），
并给 `webui.py` 追加 `--dsh` 把同一命令写进 ACM Agent 根目录的 `.webui.json`
（只写该外部项目的运行时配置，不碰源码）。出题台上游的 `detect_dsh()` 本来就
支持读 `DSH_BIN`，于是这一步变成：读题目目录的 `spec.json` → 用
`platform/problemgen/` 的提示词与 agent 循环 → 调
`platform/data/problemgen.json` 里配置的本地模型（本机 = LM Studio
`http://127.0.0.1:1234/v1` 的 `qwen/qwen3.5-9b`）。**不需要装 dsh，也没有改动
ACM Agent 项目的任何文件**——只用了它公开的 `DSH_BIN` / `--dsh` 配置点；
**在本地引擎可用时（默认），每次从平台启动出题台，AI 引擎都会被覆盖成本地
执行器**（残留的旧 dsh 命令也盖掉）。`/api/tools/status` 的 `setter.ai` 返回
`engine / baseUrl / model / maxRounds / available / hint`，出题台页面工具栏会显示
「AI 引擎：LM Studio · <模型>」（本机当前 `qwen/qwen3.5-9b`）。

> 出题台里「建骨架 / 造数据 / 对拍 / 校验」本机直接可跑；「AI 写题面与代码」
> 由本机 LM Studio 模型执行，**不再要求 dsh**。模型没开 / 连不上时该步失败
> （执行器退出码 2），流水线停在「等待提示词」，题目目录仍会生成 `REQUEST.md`
> 可复制到任意 AI 会话手工执行；成功后自动继续造数据 / 对拍 / 校验。
> AI 只读了参考文件、没真正写出 `spec.json` / `problem.md` / `std.cpp` /
> `gen.cpp` 时也会判失败（退出码 1，日志「AI 没有真正写文件」，不会把建骨架
> 留下的占位文件当成完成）。
> **接入层 vs 模型层（2026-10-28 结论）**：本次接入解决的是原问题——本机没有 dsh
> 时「AI 写题面与代码」直接不可用；现在会真正调用本机 LM Studio、能写文件、失败时
> 给出正确退出码与日志。**模型产出可靠性是另一层限制**：本机 9B 不稳定，完整一条龙
> 仍未跑通（4 次整条 e2e 均失败，失败形态与日志见说明书 §7.4）。
> 出题台日志在 `platform/data/out/webui.log`，AI 执行器自己的对话日志在
> `<题目目录>/agent-log.jsonl`。手工单独启动出题台（不经平台）时需自己指定
> `DSH_BIN` / `--dsh`，见 [`docs/出题器使用说明.md`](docs/出题器使用说明.md) §3.6。

### 经典客户端（tkinter）内置的命令行式面板

经典 tkinter 客户端（`start_client_classic.bat`）左侧同样有**「创作工具」**分组
（`Ctrl+7` 题面器 / `Ctrl+8` 出题器），提供的是「目录选择 + 参数 + 实时日志 +
产物概览」的本地面板（不需要浏览器）：

#### 题面器（`platform/statement/`）

- **做什么**：题目目录（`problem.md`）→ **单个 HTML 文件**；图片 base64 内联，
  离线可打开，浏览器 `Ctrl+P` 可打印 / 另存为 PDF。支持 `:::figure` 自由定位
  图片、校徽 / 院徽（`platform/data/branding.json`）。
- **页面**：单题构建 / **题目册**（多题合成一个 HTML）/ 样例一致性校验；
  执行日志实时显示，完成后可「打开产物」。
- **可视化**：生成后自动刷新「产物概览」——题目数 / 章节数 / 测试点数 /
  插图数 / 产物体积 / 校徽院徽状态 6 个指标，右侧按 A4 版式画出
  **题面结构预览**（标题、章节、校徽占位、页脚），不依赖浏览器也能一眼看出结构。
- **等价命令行**：`python platform\tools\statement.py build <题目目录> --out x.html`

#### 出题器（`platform/problemgen/`）

- **做什么**：调用**本地大模型**（OpenAI 兼容 `/v1` 接口）跑 agent 循环，
  按 IOI / ICPC 型规格写题面、标程、生成器，并调 ACM Agent 工具链自检。
- **页面**：环境自检 / 探测模型 / 保存配置 / 新建题目骨架 /
  运行 agent（实时日志 + **停止**）/ 打开题目目录。
  没有模型时可勾选「只生成提示词」走 prompt-only 流程。
- **可视化**：运行时有**轮次进度条 + 轮次/工具调用/写入文件/耗时四个计数 +
  agent 流水线图**（准备提示词 → 调用模型 → 工具执行 → 自检收尾 → 完成，
  按日志实时点亮）；运行结束自动刷新「题目产物」文件清单
  （文件名 / 大小 / 说明、data 测试点数量、spec.json 摘要）。
- **本机配置（2026-10-28 起）**：LM Studio `http://127.0.0.1:1234/v1`，
  小参数模型 `qwen/qwen3.5-9b`（配置在 `platform/data/problemgen.json`）；
  通用配置（`llm` / `agent` 段）为短上下文电脑收紧为 `max_tokens=4096`、
  `max_rounds=16`、`tool_output_limit=1600`、`max_context_chars=20000`、
  `max_empty_retries=2`（空回复重试），LM Studio 建议加载上下文 16384。
  **同一份配置也是内嵌出题台「AI 写题面与代码」用的配置**（出题台经
  `platform/tools/setter_ai.py` 复用本出题器的提示词与 agent 循环）；
  出题台路径另有 `setter` 段单独调参
  （`max_tokens=7680`、`max_rounds=16`、`temperature=0.2`、
  `tool_output_limit=800`、`max_context_chars=12500`、`max_empty_retries=3`，
  只影响出题台；按 16K 上下文精算：输入约 7.6K tokens + 7680 输出 ≈ 15.3K）：
  本机 9B 是 **thinking 模型**，简单问题也要烧约 2000 reasoning tokens，且
  LM Studio 关不掉思考，`max_tokens=4096` 会被思考吃满导致空回复；预算段还要求
  「所有文件写在当前工作目录（题目目录）根下，不要建 `A/` 这类题号子目录」。
  换模型 / 换服务地址改 `problemgen.json` 即可，`PROBLEMGEN_BASE_URL` /
  `PROBLEMGEN_MODEL` 等环境变量优先级更高。
  出题台侧**已实测 4 次整条 e2e，均未跑通**（修复前 1 次 + 修复后 1 次 + 调参后
  2 次；失败形态都是空回复 / 只读不写：第三次 202s / 6 轮 / 7 次工具调用、空回复
  2 次，第四次 280s / 8 轮 / 12 次工具调用、空回复 3 次，均退出码 1；详见
  [`docs/出题器使用说明.md`](docs/出题器使用说明.md) §7.4）。**AI 接入链路本身
  已验证可用**（直接跑执行器时模型写得出 `spec.json` / `problem.md`），失败在
  本机 9B 模型层。本机 8GB 显存（RTX 4060 Laptop，已用 5.3GB）只够跑 9B，更强
  模型暂无法加载；短期建议把出题台 AI 一步当「半自动」用（失败后把 `REQUEST.md`
  贴到更强模型 / 在线会话，工具链自检照旧）。
  9B 全流程生成**已实测尝试 4 次，均未跑通**（失败形态与日志见
  [`docs/出题器使用说明.md`](docs/出题器使用说明.md) §7.4）。
- **历史实测（2026-10-17，30B）**：模型 `qwen/qwen3-coder-30b` 已真实跑通 2 轮 agent
  （48.9s / 4 次工具调用 / `ok=True`）；该记录保留为历史事实。
  详细说明见 [`docs/出题器使用说明.md`](docs/出题器使用说明.md)。
- **等价命令行**：`python platform\tools\problemgen.py run --dir <题目目录> --mode icpc`

#### 题解工坊（`platform/server/trainplatform/solution/`）

- **做什么**：把题解做成 **PPT 风格幻灯片**（复刻 `cqcpc2025_solution_v2.pdf` 的
  Metropolis/Beamer 视觉：深青黑标题栏 `#23373B` + 橙色强调 `#EB811B` + 浅灰底 `#FAFAFA`），
  在浏览器里**可视化编辑**，一键导出**可继续修改的 PPTX**、PDF 与单文件 HTML。
- **页面**：作品列表（新建 / 导入 / 复制 / 删除）→ 三栏编辑器（幻灯片缩略图 / 实时画布 /
  检查器）；幻灯片类型：封面、题目页、章节页、正文页、公式页；元素：段落/列表、标题区块、
  代码（行号）、表格、图片、**数学公式**、双栏、提示条、分隔线。
- **数学公式**：输入 LaTeX（分式、根号、求和/积分上下限、矩阵、分段、二项式、重音、
  `\mathbb/\mathcal/\text`、希腊字母、嵌套上下标…），编辑器提供模板/符号面板，
  服务端用 **Windows GDI + 纯标准库**排版成高清公式图，画布实时预览；
  Markdown 导入支持 `$$...$$` / `\[...\]` / equation 环境。29 条示例公式实测全部渲染成功。
  公式在导出物中是高清 PNG（PPTX 可整体缩放/替换，改公式回工坊改 LaTeX 重导）。
- **导入**：粘贴或上传 `题解.md`（`## A - 题名` 分题、`### 题目大意` 分块、代码块/表格/图片/
  `$$` 公式），或填题目目录（自动读 `题解.md` / `spec.json` / `std.cpp`）。
- **导出**：PPTX（零依赖手写 OOXML，PowerPoint/WPS 里逐字可改）；PDF（优先调本机
  PowerPoint COM 导出，其次 Edge/Chrome 无头打印，最后浏览器「打印预览」）；HTML（离线单文件，
  `Ctrl+P` 可打印为与页面同尺寸的 PDF）。**这就是「可继续修改的 PPT 形式题解」**。
- **主题**：Metropolis / Light / Night 三套预设，可改颜色、中文字体、拉丁/代码字体、
  4:3 / 16:9、页码与标题栏开关；版式引擎预换行，浏览器预览与 PPTX 排版一致。
- **内置示例**：首次打开列表里就有 `CQCPC 2025 题解示例`（22 页、6 题、代码/表格/插图/公式）
  与 `题解工坊数学公式展示`（8 页、20 条公式），可直接「打开示例」体验；导出样例见
  [`docs/examples/cqcpc2025-solution-demo.pptx`](docs/examples/cqcpc2025-solution-demo.pptx)
  （[PDF](docs/examples/cqcpc2025-solution-demo.pdf) /
  [HTML](docs/examples/cqcpc2025-solution-demo.html)）、
  [`docs/examples/cqcpc2025-math-showcase.pdf`](docs/examples/cqcpc2025-math-showcase.pdf)
  （[PPTX](docs/examples/cqcpc2025-math-showcase.pptx)），预览图
  [`demo`](docs/examples/cqcpc2025-solution-demo-preview.png) /
  [`公式`](docs/examples/cqcpc2025-math-showcase-preview.png)。
- **等价命令行**：
  `python platform\tools\solution.py build 题解.md --out 题解.pptx --pdf --html`
- 设计/数据模型/接口契约：[`docs/题解器设计与数据模型.md`](docs/题解器设计与数据模型.md)，
  使用说明：[`docs/题解器使用说明.md`](docs/题解器使用说明.md)，
  数学公式模块：[`docs/题解器数学公式模块.md`](docs/题解器数学公式模块.md)，
  样张风格复核：[`docs/题解PDF风格规格.md`](docs/题解PDF风格规格.md)。

> 这些工具都是**纯标准库**实现，不新增任何 pip 依赖；打包进 exe 后同样可用
> （`--selftest` 里有一节 `[3b/5] 创作工具` 做真实的最小题目渲染 + 新建骨架校验，
> 题解工坊还会做 layout/HTML/PPTX/公式渲染的本地冒烟，ICPC 滚榜会做合成成绩单解析冒烟）。
> 唯一例外：出题台「AI 写题面与代码」用的本地模型执行器需要源码方式启动
> （见上文「创作与赛事工具」）。
> B/S 的 iframe 内嵌层同样零第三方依赖：静态服务用 `http.server`，
> 出题台子进程用 `subprocess` + 端口探活，桥接代码在
> `platform/server/trainplatform/tools_bridge.py`（测试见
> `platform/server/tests/test_tools_bridge.py`）。

## 赛事中心：ICPC 滚榜（`platform/server/trainplatform/rollout/`）

- **做什么**：导入 **ICPC 成绩单 `.xlsx`**（如「华为杯」第 51 届 ICPC 成都站正式赛文件），
  自动识别赛事标题 / 多榜单 sheet（正式队伍 / 所有队伍 / 打星队伍）/ 队伍 / 学校 / 队员 /
  奖牌 / A~M 题目列 / 一血，界面参考 **XCPCIO Board** 风格（浅色白底排行榜），
  解封行为参考 **ICPC Tools Resolver**，支持**封榜前 → 标准 Resolver pending 揭晓 → 最终榜**的演示。
- **封榜/解封（标准 Resolver，AC + 推算 WA）**：成绩单只有最终结果，系统按每题的通过时间自动回放封榜——
  `pending` = 封榜后 AC + 按总尝试次数推算的 WA（默认每道有失败的 AC 题先揭 1 个 WA，再揭 AC）；
  每一步按当前 live 榜，从 **pending 非空且名次最低**的队伍中**只揭晓 1 个 pending**；
  WA 时黄格变红色 `-N`+「推」、榜不移动；AC 后才更新题数/罚时并重排，剩余 pending 保留；
  没有 walk、没有轮次，本场 **223 步（123 AC + 100 WA）**。
- **推算 WA 的诚实边界**：失败次数来自成绩单总尝试次数 `attempts−1`，**可能含封榜前失败**，
  不是封榜后失败提交的真实时间戳；仅用于演示“黄→红→尝试次数”的交互，设置里可关闭回到 123 步 AC-only。
  完整标准 Resolver 的逐次 WA/TLE pending 需要提交级数据，本平台不伪造。
  本成都站实测 111 队/123 AC、305/319 队名次变化，茉莉雨封榜 #4 → 最终冠军。
  解封时有**动画**：AC 题目格先绿闪，随后该队行平滑滑到新名次（约 1.2 秒），镜头仍停在下一队；
  WA 只做红闪，不移动行。封榜黄格带闪烁方框；焦点/悬停高亮只染非题目列，不覆盖 AC/WA 颜色。
- **文件选择**：支持拖拽/点击上传任意同类 xlsx、服务器路径导入；已导入赛事列表可随时切换、
  下载原文件、删除；同一文件重复导入自动复用。
- **滚榜操作**：开始滚榜/暂停、上一队/下一队、直接结束、重置、速度、Resolver 配置、全屏；
  快捷键 `空格/F` 下一队、`R/B` 上一队、`0` 重开、`P` 暂停、`+/-` 调速、`H`/`?`/`Esc`。
- **兼容性**：纯标准库自研 xlsx 读取（zipfile + ElementTree，无 openpyxl/pandas 依赖）；
  兼容 `t="str"`/sharedStrings/inlineStr/数字、多 sheet、列顺序不同、缺奖牌列等；
  缺解题数/罚时自动计算；非同类文件给出明确错误提示。
- **等价命令行**：`python platform\tools\rollout.py import <成绩单.xlsx>` / `list` / `export <id> --out contest.json`
- 设计/数据契约：[`docs/滚榜系统设计与数据模型.md`](docs/滚榜系统设计与数据模型.md)、
  [`docs/滚榜AC-only标准Resolver.md`](docs/滚榜AC-only标准Resolver.md)、
  [`docs/滚榜Resolver模式.md`](docs/滚榜Resolver模式.md)，
  使用说明：[`docs/滚榜系统使用说明.md`](docs/滚榜系统使用说明.md)。
- 界面静态预览（直接双击打开）：[封榜前](docs/examples/ui-rollout-frozen-preview.html) ·
  [解封滚榜](docs/examples/ui-rollout-rolling-preview.html) ·
  [最终榜](docs/examples/ui-rollout-final-preview.html)。

## 界面主题：浅色 / 暗色 / 跟随系统

三种客户端都支持三套主题，切换方式与实现完全对应：

| 形态 | 切换入口 | 持久化 |
| --- | --- | --- |
| B/S 网页端 / Electron 客户端 | 顶栏右侧分段选择器（☀ 浅色 / ☾ 暗色 / ◑ 跟随系统） | `localStorage['tp-theme']` |
| 经典 C/S（tkinter） | 顶栏图标式切换器 + 顶栏 ☰「视图 → 界面主题」 | `platform/data/client-prefs.json` |

> **语义**一致（同样的三档、同样的语义色），但**视觉基线不同**：
> Web 平台（B/S 与 Electron 共用）沿用原来的蓝灰控制台配色；
> 经典 tkinter 客户端换成 Steam 风格色板
> （见 `client/theme.py` 的 `DARK_PALETTE` / `LIGHT_PALETTE`）。
> 不要再假设两边色值逐字相同。

**「跟随系统」是真实现，不是占位**：

- B/S / Electron 用 CSS `@media (prefers-color-scheme: dark)`，浏览器系统配色一变即跟随
- 经典 C/S 读 Windows 注册表
  `HKCU\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize\AppsUseLightTheme`
  （零第三方依赖，用标准库 `winreg`）

实现要点（改动前必读）：

1. **所有颜色走变量**，组件里不写死色值。B/S 集中在 `global.css` 的
   `:root`（浅色）/ `[data-theme='dark']`（暗色）/ `@media` 三块；
   C/S 集中在 `client/theme.py` 的两个 `*_PALETTE` 字典，派生色
   （`TONE_BG` / `NOTE_TONES` / `ROW_TAGS`）由 `refresh_derived()` 统一重算。
2. **B/S 的 `data-theme` 契约**：`light` / `dark` 显式设置属性；
   `system` **移除属性**交给媒体查询。暗色覆盖用 `:root:not([data-theme])` 限定，
   保证「显式选浅色」不会被系统暗色覆盖 —— 这是最容易写错的地方。
3. **C/S 必须重建控件树**：tkinter 控件颜色在**创建时固化**，改模块级常量不会影响
   已存在的控件。所以 `set_theme()` 的流程是「改调色板 → 销毁全部控件 → 重建 →
   重新拉数据」（实测重建后控件数与初次渲染一致）。
4. **防首屏闪烁**：`index.html` 里有一段内联脚本，在样式与 React 加载前就设好
   `data-theme`；`build.mjs` 生成的产物 `index.html` 也包含同一段脚本。

验证命令：

```powershell
python platform\tools\check_theme.py          # CSS 三主题变量与选择器完整性
python platform\tools\check_theme_build.py    # 构建产物里三主题是否真的可用
python platform\tools\check_client_theme.py   # 客户端主题偏好持久化
python platform\client\app.py --smoke --screenshot-dir .tmp-run\shots
                                              # GUI 冒烟：三主题真实切换 + 换肤后重渲染 + 截图
```

## 模块与状态

| 模块 | 目录 | 状态 |
| --- | --- | --- |
| M1 训练监控（洛谷公开数据） | `platform/server/trainplatform/luogu/` | ✅ 可用 |
| M2 **算法知识点级薄弱分析** | `trainplatform/algo_tags.py` + `algorithm_analysis.py` | ✅ 可用 |
| M1b 难度维度分析（能力上界） | `trainplatform/analytics.py` | ✅ 可用 |
| M4 每日一题推荐 | `trainplatform/recommend.py` | ✅ 可用 |
| M3 赛程日历 | `trainplatform/calendar/` | ✅ 可用 |
| Electron 桌面客户端（推荐） | `platform/electron/` | ✅ 可用（原生窗口 + 题面器/出题台原生界面内嵌；~340 MB 发布包） |
| 经典 C/S 桌面客户端（tkinter） | `platform/client/` | ✅ 可用（Steam 风格界面 + 高分屏 DPI 感知） |
| 题面器（图片定位 + 校徽院徽） | `platform/statement/` | ✅ 可用（**B/S / Electron 客户端原生界面内嵌** + 经典客户端命令行面板 + 命令行） |
| 出题器（本地大模型 + IOI/ICPC） | `platform/problemgen/` | ✅ 可用（**B/S / Electron 客户端原生出题台内嵌** + 经典客户端命令行面板 + 命令行）；同一套提示词与 agent 循环经 `setter_ai.py` 供出题台的「AI 写题面与代码」使用（**接入链路可用；本机 9B 整条 e2e 未跑通**，见 [`docs/出题器使用说明.md`](docs/出题器使用说明.md) §7.4） |
| **题解工坊（可视化题解 + PPTX/PDF）** | `trainplatform/solution/` + `platform/web/src/views/SolutionStudio.tsx` | ✅ 可用（**B/S / Electron 原生页面**：编辑幻灯片 → 导出可改 PPTX / PDF / HTML；命令行同名工具） |
| **ICPC 滚榜（成绩单 → 大屏揭晓）** | `trainplatform/rollout/` + `platform/web/src/views/Rollout.tsx` | ✅ 可用（**B/S / Electron 原生页面**：导入任意同类 xlsx → 播放/暂停/上下队/速度/全屏；纯标准库解析；命令行同名工具） |
| 训练任务 / 题单闭环（T+1） | `trainplatform/tasks.py` | ✅ 可用（按薄弱算法生成题单、设截止时间、同步后自动比对 AC/卡住/没做） |
| 成长趋势（咕值 / Elo / 通过题数） | `trainplatform/trends.py` | ✅ 可用（Elo 用洛谷完整比赛历史；咕值/通过题数随每日快照累积） |
| 创作工具桥接层 | `trainplatform/tools_bridge.py` | ✅ 可用（题面器静态托管 + 出题台子进程托管；启动出题台时注入 `DSH_BIN` 并追加 `--dsh` 覆盖为本机模型执行器，启动后回查 `/api/state` 确认接上（没接上则 `setter.ai.available=false` + hint），`/api/tools/status` 返回 `setter.ai`；测试见 `platform/server/tests/test_tools_bridge.py`） |

## 薄弱点分析：算法维度 vs 难度维度

用户明确要求「**不是对难度进行薄弱分析，而是对算法薄弱点进行分析**」，因此平台提供**两个互补维度**：

| 维度 | 回答的问题 | 数据来源 | 接口 |
| --- | --- | --- | --- |
| **算法知识点**（主） | 他**哪个算法**不行？ | 洛谷题目标签（318 个算法标签） | `/api/algorithm/team` |
| 难度档（辅） | 他的**能力上界**在哪一档？ | 题目 difficulty | `/api/analytics/team` |

算法维度还会把两种情况**分开**，因为结论完全不同：

- `category: "weak"` —— **练了但过不去**（有未过题）→ 专项突破
- `category: "blind"` —— **根本没接触过**（通过 0、未过 0）→ 安排学习

### 每个队员都是同一套完整指标（2026-10-17 起）

用户要求「每个同学都要有**完整**指标，而且指标都要**一样**」。实现方式：

- 后端 `algorithm_analysis.canonical_metric_axes()` 从标签库生成一份**与成员无关**的
  全量算法指标清单（当前 **260 个算法维度**），排序口径固定：**核心考点优先 →
  题库题量降序 → 算法名**；
- `/api/algorithm/team` 与 `/api/algorithm/member/<key>` 都返回 `metricAxes`（完全相同）；
- 前端「算法画像 → 单个队员」：
  - 雷达图固定取**前 12 个统一指标**（每个同学轴完全一样，可横向比较）；
  - 下方「完整指标」列表展示**全部 260 个维度**（顺序全班一致、可搜索、带
    掌握度 / 过题数 / 未过题数 / 分类徽章）；
  - 元分类（`Ad-hoc` / `其它技巧` / `分类讨论`）保留在完整指标里但不计入薄弱排序。

> 早期版本是「每人各自最相关的 10 个算法」，不同队员轴不同、列表也不全，无法比较；
> 现在数据与展示口径已统一（后端 61 项算法分析测试覆盖，含「多人指标必须完全相同」）。

### 训练任务 / 题单闭环（T+1 自动校验）

「每日一题」只是推荐；**训练任务**才是可追踪的布置。B/S / Electron 客户端左侧
**「训练任务」** 页：

1. **按薄弱算法生成题单**：选算法维度（如「动态规划 DP」）+ 题量（如 20），
   点「按算法生成题单」——后端只挑**还没被这些队员做过**的题（已通过人数少的优先，
   其次按难度），题号自动填好，也可手工粘贴洛谷题号；
2. **设截止时间 & 圈定队员**：默认全体在训队员，也可只选部分人；
3. **T+1 自动验收**：每天同步一次后，用快照里的 `practice.passed / submitted` 比对，
   每人每题显示 **✓ 已 AC / ✗ 卡住（提交过没过）/ · 还没做 / ? 数据不可见**；
4. **一键复制催交名单**：包含未完成队员、卡住的题号，直接发群里。

口径与平台一致：洛谷数据不可见（未实名 / 隐私 403）的队员标 `?` 并计入
「数据不可见」，**绝不说成「没做」**。T+1 是洛谷公开数据的固有延迟。

数据文件：`platform/data/tasks.json`（人工可读、可手工编辑）。
接口：`GET/POST /api/tasks`、`GET /api/tasks/meta`、`GET /api/tasks/suggest`、
`GET/POST /api/tasks/<id>`、`POST /api/tasks/<id>/delete`。
后端 `trainplatform/tasks.py` 12 项单测覆盖四种状态与 CRUD。

### 成长趋势：咕值 / Elo / 通过题数曲线

左侧「成长趋势」页，按队员看三条线（以及全队速览表）：

- **Elo 曲线**：来自洛谷 `profile.elo` 的**完整比赛历史**（每场 rating / 时间 / 比赛名），
  立刻可画；界面显示当前 Elo、较首场变化、比赛场次。**只有打过 rated 比赛的队员才有**
  （本机实测 5/33 人有记录，其余显示「暂无记录」，不编造）。
- **咕值曲线**：洛谷单次快照只给**当前值**（+ 五项的分数明细），没有历史接口；
  所以这条线靠**每天同步的快照逐日累积**，界面如实显示「已累积 N 个快照」
  （当前 2 天 → 2 个点，随每日同步自动变长）。
- **通过题数曲线**：同样来自快照的 `derived.passedCount`，逐日累积。
- 全队速览表按咕值排序，含 Elo、比赛场次、通过题数与同步状态。

接口：`GET /api/trends/team`、`GET /api/trends/member/<key>`；
后端 `trainplatform/trends.py` 10 项单测。

### 算法标签库（关键基础设施）

洛谷练习数据只给 `{pid, name, difficulty}`，**不含算法标签**。平台通过
`platform/tools/fetch_tags.py` 构建标签库：

- 来源：`/problem/list` 每页直接给出 50 道题的 `tags` 数组，
  同一页 HTML 里还有 `<a href="/problem/list?tag=N">名称</a>` → **一次请求同时拿到两个映射**
- 规模：**386 个标签 / 17762 道题**（全题库），耗时约 10 分钟，长期缓存
- 实测覆盖率：`B` 前缀 100%、`P` 前缀 95.0%、整体 94.3%
  （缺口是 `AT`/`CF`/`UVA`/`SP` 镜像题，洛谷题库列表不收录）

```powershell
python platform\tools\fetch_tags.py --stats          # 看统计
python platform\tools\fetch_tags.py                  # 构建（首次约 10 分钟）
python platform\tools\fetch_tags.py --probe P1001 P2152   # 查某题的算法标签
```

## 快速开始

### B/S 形态（浏览器）

双击 **`启动平台.bat`**，或在终端里：

```bat
:: 一键启动（后端 + 前端 + 自动开浏览器）
启动平台.bat

:: 启动前先同步一次洛谷数据
启动平台.bat -Sync

:: 只起后端
启动平台.bat -NoWeb

:: 跳过 Vite，直接「预构建 + 静态服务」（本环境推荐，见下）
启动平台.bat -Static
```

> **本机实测：引擎会自动降级。** Vite dev server 需要 esbuild 派生子进程，
> 而受限环境禁止这一点（`spawn EPERM`）。启动器会先试 Vite，失败后**自动**改用
> `esbuild CLI 预构建 + 标准库静态服务`（功能完整，只是没有热更新），
> 无需你手动干预。加 `-Static` 可跳过那次注定失败的尝试，启动更快。

> **创作/赛事工具在哪**：浏览器打开后默认进入**总览**，右侧有三个子系统卡片：
> **训练中心** / **创作中心** / **赛事中心**。创作中心里有 **题面器**（内嵌
> xcpc-statement-generator 原生界面）/ **出题台**（内嵌 ACM Agent 的出题台 Web 界面）/
> **题解工坊**（自研可视化题解编辑器，编辑后导出可继续修改的 PPTX / PDF）；
> 赛事中心里有 **赛程日历** 和 **ICPC 滚榜**（导入成绩单 → 大屏逐个揭晓）。
> 首次进入出题台会自动拉起它的本地服务（约 2 秒），页面右上角可「在新窗口打开」或「停止服务」。
> 出题台的「AI 写题面与代码」由本机 LM Studio 模型执行（平台启动时自动注入
> `DSH_BIN` 并追加 `--dsh`，见上文「创作与赛事工具」），不需要安装 dsh。
> 注意：**PyInstaller 打包 exe / Electron 发布包运行时暂不支持本地 AI 引擎**，
> 请用源码方式启动（`启动平台.bat`、`启动客户端-浏览器窗口.bat`，或
> `python platform\tools\app_window.py`）。打包入口加 `--setter-ai` 分发与
> hiddenimports 的支持**尚未实现**。

### C/S 形态（桌面客户端 · Electron 壳）

双击 **`启动客户端.bat`**：它优先启动 Electron 客户端（没构建时自动回退经典版）。

```bat
启动客户端.bat          :: Electron 客户端（推荐；自动拉起本地服务并开原生窗口）

:: Windows 拦截未签名 exe 时用这个（只用系统 Python + Edge，不会被拦）：
启动客户端-浏览器窗口.bat

:: 直接双击 exe 也可以（跳过 .bat）：
release\CJU-ACM-Client-Electron-1.0.0\CJU-ACM-Client.exe

:: 经典 tkinter 客户端（需要 Python；下面这些参数会自动路由到经典客户端）：
启动客户端.bat --selftest
启动客户端.bat --http
start_client_classic.bat
```

> **Electron 窗口里就是完整平台**：左侧是**总览 + 训练中心 / 创作中心 / 赛事中心**四个入口，
> 创作中心的题面器 / 出题台 / 题解工坊是原生界面直接内嵌（同一份 React 页面），
> 赛事中心的 ICPC 滚榜同样原生页面。
> 快捷键：`Ctrl+1~5` 训练模块、`Ctrl+7` 题面器、
> `Ctrl+8` 出题台、`Ctrl+9` 题解工坊、`Ctrl+0` ICPC 滚榜、`F5` / `Ctrl+R` 重新载入、`F11` 全屏、
> `Ctrl+Shift+I` 开发者工具。
> 关闭窗口即停止本地服务。
> 日志：`%APPDATA%\cju-acm-client\launcher.log` / `server.log`。
> 首次使用需先构建（见「打包成 exe」）：
> `python packaging\build_exe.py --only server` + `python packaging\build_electron.py`。

#### 经典 tkinter 客户端（备用）

> 界面为 **Steam 风格**：左侧导航（`Ctrl+1~6` 训练模块、`Ctrl+7~8` 题面器 / 出题器）
> + 渐变页头 + 圆角卡片；已声明 DPI 感知，150% / 200% 缩放下文字清晰不发虚。
> 详见上文
> [「经典客户端界面：Steam 风格 + 高分屏清晰度」](#经典客户端界面steam-风格--高分屏清晰度)。

> 经典客户端**默认直连本地数据引擎**（`trainplatform.engine.Engine`），
> 因此**后端没启动、甚至没网络也能查看训练数据**；`--http` 可切换为连远程后端。
> 启动器会自动探测可用的 Python，且**不信任 `py -3`** ——
> 本机实测它会指向一个损坏的解释器（`Unable to create process ...`）。

经典客户端首次运行前：

```bat
python -m pip install requests
cd platform\web
pwsh -File install.ps1
```

手动启动：

```bat
python platform\server\run.py --port 8790
cd platform\web
node build.mjs --out dist
python ..\tools\serve.py --web dist --api http://127.0.0.1:8790 --port 5190
```

自检（不启动服务，检查文件/模块/名单/快照/测试）：

```bat
python platform\tools\selfcheck.py
```

> **首次使用没有数据是正常的。** 平台需要先同步一次洛谷数据；
> 在界面「训练总览」里点「同步」按钮，或用 `启动平台.bat -Sync`。
> 33 人全量同步约需 45 秒。

> ⚠️ **改脚本前必读** [`docs/Windows启动器与编码约定.md`](docs/Windows启动器与编码约定.md)。
> `.bat` 必须纯 ASCII + CRLF，`.ps1` 必须 UTF-8 with BOM，`.npmrc` 必须纯 ASCII ——
> 三者猜错都会表现为「脚本崩解 / 中文乱码 / 找不到文件」，而报错信息看起来毫不相关。

## 已验证的关键技术事实（2026-10 实测）

这些事实决定了整个平台的架构，改动前请先复验。完整论证见 [`docs/方案与实测结论.md`](docs/方案与实测结论.md)。

1. **洛谷新版前端数据不在 `window._feInjection`**，而在页面内
   `<script type="application/json">` 块里，形如
   `{"instance","template","status","locale","data"}`（其中一块带 `id="lentille-context"`）。
   **实测 `_feInjection` / `JSON.parse` / `decodeURIComponent` 在页面上出现 0 次** ——
   网上 2021–2023 年的洛谷爬虫代码（含 PyPI 的 `pyluog`，最新版停在 2021-06-29）
   大概率已失效，不要作为依赖引入。
2. **`GET /api/user/search?keyword=<用户名>`** 返回干净 JSON，可把洛谷用户名解析为 uid。
   `/user/<用户名>` 会 404 —— 用户名不能当路径用。
3. **`GET /user/<uid>/practice`** 无需 Cookie 即返回 `data.passed` / `data.submitted`，
   每项为 `{pid, type, name, difficulty}`，`difficulty` 取 1~7。
   **这是薄弱点分析的数据底座。**
4. **`GET /user/<uid>`** 无需 Cookie 返回 `data.{user, gu, elo, prizes, dailyCounts}`：
   咕值评分、比赛 rating 历史、获奖记录、每日做题数。
5. **`GET /contest/list?_contentOnly=1`** 无需 Cookie 可用，是赛程日历的洛谷赛数据源。
   ⚠️ `/api/contest/list` **404，该接口不存在**；`x-luogu-type` 请求头**已失效**，
   只有 `_contentOnly=1` 查询参数有效。
6. **未实名认证的洛谷账号整体返回 403**，`errorMessage = "该用户未通过实名认证"`。
   实测连续 6 次请求全部 403 → **不是限流，重试无用**。本队 33 人中有 7 人属于此类，
   必须单独归因并给出「联系本人实名认证」的可行动指引，**不能显示成「没做题」**。
7. **洛谷会先回 302 并下发 `C3VK` cookie**（反爬，`Location` 指向自身）。
   必须用带 CookieJar 的会话自动跟进。另：**带任何 `Cookie` 头反而触发 302**，
   所以不要伪造 Cookie。
8. **`GET /record/list?user=<uid>`** 返回 401 `needLogin` —— 提交记录**必须 Cookie**。
   本项目按「不存 Cookie」策略**不使用**该接口，因此分析只基于通过/未过题目集合，
   不涉及尝试次数、评测状态与罚时。
9. **ICPC / CCPC / 蓝桥杯 / 天梯赛 / CSP 五类赛事全部没有可抓接口** ——
   官网是 JS SPA，HTML 里 0 个日期串。⚠️ 天梯赛与 PATEST 的 `/api/*` **返回 200
   但响应体是 SPA 兜底 HTML**，看到 200 不能认为接口可用。这五类**只能人工维护**。
   唯一可靠的自动源是 Codeforces `/api/contest.list`（真 JSON）。
10. **本机 Python 3.8 没有 `zoneinfo` 模块**，`pytz` 未安装 →
    ICS 生成硬编码 `+0800`（Asia/Shanghai 自 1991 年起无夏令时，这是精确值）。
    全天事件的 `DTEND` 是**排他**的，必须是**次日**；折行必须**按字节**（中文 3 字节/字）。

沙箱内 `curl.exe` 的 schannel 不可用（`SEC_E_NO_CREDENTIALS`），
Python + `requests`（OpenSSL）可正常访问外网。

## 全队实测基线（2026-10-16 同步）

| 状态 | 人数 | 说明 |
| --- | --- | --- |
| `ok` | 25 | 主页 + 练习数据都拿到 |
| `unverified` | 7 | 洛谷未实名认证，公开数据整体不可见 |
| `skipped` | 1 | 名单中无洛谷 ID |

合计 **通过 1685 题、未过 113 题**，全队同步耗时约 48 秒。

难度分布：入门 479 / 普及- 763 / 普及·提高- 279 / **普及+·提高 111（瓶颈）** /
提高+·省选- 32 / 省选·NOI- 7 / NOI·CTSC 8。
**难度 4 是瓶颈档，难度 5 以上几乎无覆盖** —— 这正是每日一题应当发力的区间。

## 目录结构

```
程序设计集训一体化训练平台/
├── platform/
│   ├── server/
│   │   ├── trainplatform/          平台后端包（可直接 import 复用，不依赖 web 层）
│   │   │   ├── config.py           路径、难度口径、抓取节流参数
│   │   │   ├── store.py            JSON 数据仓（原子写 / 快照 / 缓存）
│   │   │   ├── roster.py           名单维护与校验
│   │   │   ├── luogu/              洛谷数据源：client / sync
│   │   │   ├── analytics.py        薄弱点分析引擎
│   │   │   ├── recommend.py        每日一题推荐
│   │   │   ├── calendar/           赛程日历 + ICS 导出
│   │   │   ├── tasks.py            训练任务/题单闭环（T+1 自动比对 AC/卡住/没做）
│   │   │   ├── trends.py           成长趋势（咕值 / Elo / 通过题数曲线）
│   │   │   └── tools_bridge.py     创作工具桥接（题面器静态托管 + 出题台子进程 + 注入本机 AI 执行器）
│   │   ├── run.py                  后端服务入口（含完整 API 合同注释）
│   │   └── tests/                  离线回归测试（387 项）
│   ├── web/                        React + Vite 前端
│   │   └── src/views/              含 StatementGenerator / SetterConsole 两个 iframe 内嵌视图
│   ├── client/                     经典 C/S 桌面客户端（tkinter · Steam 风格）
│   │   ├── app.py                  主窗口 + 8 个页面 + 自检 / 冒烟
│   │   ├── tool_runner.py          题面器 / 出题器的运行层（CLI 捕获 + 实时日志）
│   │   ├── theme.py                配色 / DPI 感知 / 缩放 / ttk 样式
│   │   └── widgets.py              圆角卡片 / 渐变页头 / 矢量图标 / 图表
│   ├── electron/                   Electron 桌面客户端（推荐）
│   │   ├── main.cjs                主进程：拉起服务端 → 原生窗口 → 快捷键 → 退出清理
│   │   ├── server-manager.cjs      服务端进程管理（ready-file + HTTP 探活 + taskkill）
│   │   └── icon.ico / icon.png     应用图标
│   ├── tools/
│   │   └── app_window.py           浏览器窗口壳（系统 Python + Edge，免未签名 exe）
│   ├── statement/                  题面器（题目目录 → HTML/PDF，纯标准库）
│   ├── problemgen/                 出题器（本地大模型 agent 出题；setter_ai.py 供出题台 AI 一步调用）
│   └── data/                       运行时数据（members / snapshots / events / 缓存）
│       └── tools.json              可选：创作工具路径覆盖（不写则自动探测）
├── packaging/
│   ├── build_exe.py                5 个 PyInstaller 变体 + 调用 Electron 组装
│   └── build_electron.py           Electron 客户端组装（复用本机 Electron 运行时）
├── docs/
│   ├── 方案与实测结论.md            完整方案、决策依据与本项目全部实测事实
│   ├── 出题器使用说明.md            M2 出题器（网页端内嵌 / 客户端 / 命令行 + 配置）
│   └── 题面器增强方案.md            M5 题面器增强的勘察报告与路线推荐
├── _probe/                         阶段0 可行性探针（可复跑的实测脚本）
├── _m5_probe/                      M5 勘察的 Typst 编译验证产物
├── 启动平台.bat                     一键启动 B/S
├── 启动客户端.bat                   Electron 桌面客户端启动入口
├── 启动客户端-浏览器窗口.bat        免未签名 exe 的桌面窗口启动入口
└── start_client_classic.bat         经典 tkinter 客户端启动入口
```

## 前端构建说明（重要）

标准构建是 `pnpm run build`（`tsc -b && vite build`）。**在受限终端里 `vite build`
会失败于 `spawn EPERM`**，原因见下，替代命令：

```powershell
cd platform\web
pnpm run typecheck           # 类型检查（0 错误）
node build.mjs               # 等效生产打包（受限环境可用）
```

`spawn EPERM` 是**环境边界，不是代码问题**：esbuild 0.21.5 的 JS API 在
`ensureServiceIsRunning()` 里无条件 spawn 子进程且使用管道 stdio，而沙箱禁止程序
打开命名管道。`node build.mjs` 直接用 esbuild 的平台二进制（独立进程，不经管道），
因此可用；它还会对产物做 11 组完整性抽查（各视图 / 共享组件 / 样式 / API 路由 /
中文文案 / 新增的创作工具 iframe 内嵌），避免「打包成功但内容缺失」的静默失败。

**依赖安装的两个坑**（详见 `platform\web\NPMRC-说明.md`）：

1. 必须用 `pnpm install --node-linker=hoisted`。本机无法创建真符号链接，pnpm 会退化
   成 NTFS junction，而 Node 的 ESM 解析器**不解析 junction**，导致
   `Cannot find package 'esbuild'`。
2. **`.npmrc` 不能写中文注释** —— pnpm 按系统 ANSI 代码页（GBK）读取它，中文注释会与
   配置行粘连导致配置失效。`.npmrc` 只允许 ASCII。
   一键修复：`pwsh -File platform\web\install.ps1`

## 数据流

```
洛谷公开页面 ──► luogu/client ──► luogu/sync ──► data/snapshots/<date>.json
                                                      │
                                                      ▼
                              analytics（薄弱点雷达）──┴──► recommend（每日一题）
                                                      │
                                       calendar（赛程）┴──► run.py（HTTP API）──► web/（前端）
```

## 运行

```bash
# 后端（Python 3.8+）
python platform/server/run.py --port 8790

# 前端（Node + pnpm）
cd platform/web && pnpm install && pnpm run dev
```

## 授权

`statement/` 目录 fork 自 [xcpc-statement-generator](https://github.com/lihaoze123/xcpc-statement-generator)，
遵循 **AGPL-3.0**；二次分发需同样开源。平台其余部分为原创。

## 隐私约定

平台**不保存任何洛谷账号凭证（Cookie）**。所有队员数据均来自洛谷公开页面，
仅用于本队训练分析；队员可在洛谷设置中关闭公开可见而随时退出监控。
名单与学号属于个人信息，`data/members.xlsx` / `data/members.json` 不纳入版本控制的公开分发。
