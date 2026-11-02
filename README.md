# 小雅资源捕获助手

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](./LICENSE)
![Manifest V3](https://img.shields.io/badge/Chrome%20Extension-Manifest%20V3-4285F4)
![Browser](https://img.shields.io/badge/Edge%20%2F%20Chromium-111%2B-0078D7)

一个面向小雅课程平台（`*.ai-augmented.com`）的浏览器扩展：把课程页面里**已经加载到你浏览器**的课件
（PPT / PDF / Word / Excel / 课程视频）捕获下来供你保存，并让视频页自动播放、支持断点续播。

**当前版本 v1.16.0 要点**
1. **新增「完成后自动提交」**：文档读到、视频看到结尾后，自动点页面上的「完成」按钮。
2. **修复重复捕获**：同一个课件经"直链 + 预览转码字节"两条通道入库、或其中一条没有文件名时，会出现 `数学.pptx` + 空白名两条。现以 `资源ID → 真名` 映射为桥梁合并，并保证任何条目都有名字（拿不到真名时用「资源ID + 类型后缀」占位，后续可反查真名）。
3. **修复漏捕**：平台 `/resource/download/<资源ID>` 这类无后缀下载接口此前被"严格模式"误判为无线索二进制流而丢弃，现识别为有效课件下发。
3.5 **修复「页面被当成文件」**：现已收紧为「必须含 download/file/material/attach 等下载动词段 + 资源ID，且排除页面路由」，并在启动时自动清理历史误收条目、补全空白名。
4. **修复三处底层缺陷**：
5. **界面重构**：弹窗改为「品牌行 + 三视图分页签（文件 / 调试 / 日志）+ 各视图独立工具栏」，卡片式列表；新增批量下载进度条、文件数量徽标、骨架屏与动画。
6. **其他修复**：切换视图时底部偶现的水平滚动条；"日志服务未就绪"（后台无响应时直读本机存储）。
7. **运行日志**：捕获、过滤、下载、HLS 等关键事件统一落盘，可查看、按级别过滤、导出 `.txt`。

> **合规边界**：插件只读取当前页面已经加载的资源地址，不破解、不绕过任何登录、权限或观看验证，
> 全部判定与保存在本机浏览器内完成，不上传任何数据。


---

## 功能特性

### 📄 课件捕获与下载
- 打开课件预览即自动截获**完整文件内容**（本地字节，最可靠），或记录可直连的下载地址（带登录态）。
- **批量下载**：勾选多条直接保存到下载目录，不逐个弹"另存为"；带进度条与逐条失败汇总。
- **自动匹配真实文件名**：从课程资源接口收割 `资源ID → 文件名` 映射，自动替换平台生成的数字名；列表内双击可手动改名。
  - 按字段**优先级**取真名：`resourceName`/`fileName`/`originalName` > `name`/`title` > 其它 `*name`；`groupName`/`teacherName`/`courseName`/`nodeName` 等一律排除（避免把章节名、老师名当文件名）。
  - **资源标题优先**：视频通过 `vod/duration/<资源ID>` 与资源配对后，一旦资源列表给出该 ID 的标题，会**覆盖**视频文件名（并保留 `.mp4` 后缀）——因为视频文件名常是老师上传时的名字。
- **下载自校验**：对响应做魔数校验（`%PDF` / `PK` / `ftyp` / gzip），拒绝把错误页当文件保存。

### 🧩 重复条目自动合并
平台把 PPT/Word 转成 PDF 供预览，于是同一课件会同时以两种形态进入列表。去重规则（`background.js` → `dedupeAll`）：

- 以「资源ID → 真实文件名」映射（`nameMap`）为桥梁推导规范名，再按**基名**分组；
- 扩展名相同、或 `pdf` 与 `office` 互为转码形态时，视为同一文件合并为一条；
- 合并时保留字节条目（可直接保存）、保留直链 URL（利于下载）、保留更早的捕获时间；

### 🎬 视频 HLS 下载
- `m3u8` → 解析播放列表（自动选最高码率）→ 6 路并发抓分片 → 拼接保存 `.ts`。
- 分片失败**自动重试**（指数退避，每片最多 2 次）；重试后仍有失败则**明确报错并拒绝保存**，绝不静默交付损坏文件。

### 🔍 严格的文件识别（单一规则源）
- 识别规则集中在 `scope.js`，后台 Service Worker、隔离世界、主世界**共用同一份判定**，避免多处正则漂移。
- 捕获必须**同时**满足：① 请求发生在课程学习页面上下文（平台域，或平台内嵌的预览框架，跨域框架靠祖先链识别）；② 资源类型明确属于课件（文件名 / 地址后缀 / 精确 MIME）。
- 明确拒绝：JS/CSS/HTML/JSON/字体/图片、`txt`、无关 `zip`/`rar`、HLS 分片（`.ts`/`.m4s`）、音频、可执行文件，以及返回 JSON 的平台元数据接口。
- `application/octet-stream` 这类含义不明的二进制流，严格模式下必须另有课件文件名线索才放行。
- 被过滤的资源连同**拒绝原因**写入调试面板，可观测、可排查。
- 设置项：捕获范围开关、严格模式开关、**额外允许站点**白名单（支持 `*.example.edu.cn` 通配）。

### ▶️ 视频页自动播放并静音
- 进入含视频的课程页后自动静音并播放（静音是浏览器允许自动播放的前提），含页面内嵌播放器框架与 open shadow DOM。
- 用户**手动暂停后不再抢播**；范围默认仅课程平台，不影响其他网站。

### ⏯ 断点续播
- 平台把视频重置回开头、或页面重新加载时，自动跳回你上次看到的位置。
- 从别的窗口切回标签页时，自动从暂停处继续。
- **边界（有意设计，并有测试固化）**：标签页隐藏期间视频保持暂停、进度不增长——不维持后台播放、不拦截平台暂停、不伪造任何观看记录；你自己点过暂停的视频不会被续播。
- 位置只存本机（最多 200 条，45 天过期），可在设置中关闭。

### ✅ 完成后自动提交
平台要求手动点「完成」才记录进度，忘了点就得重读一遍。此功能在你**确实完成、且满足平台时长门槛**后替你点：

| 类型 | 完成判定 |
| --- | --- |
| 视频 | **播放到结尾即可提交**（`ended`，或距结尾 1 秒内）。没播完时平台只给一条提示、不弹窗，所以视频页**不做试探点击**，静默等到播完再点 |
| 文档（可滚动） | 停留 ≥30 秒 **且** 主阅读区滚到 ≥95% |
| 文档（单页不可滚动） | 停留 ≥60 秒 |


### 🧰 其他
- **运行日志（排障用）**：后台为唯一写入点的环形日志（600 条，仅存本机），记录捕获/过滤/下载/HLS 事件及失败原因；弹窗"📋 日志"面板支持"全部 / 仅警告·错误"过滤、一键导出 `.txt`、清空。报问题时把导出的 txt 附上即可定位。
- **调试面板**：查看页面请求与每条被过滤资源的原因。
- **设置页**：独立标签页布局（780px，分组卡片），改动即存、一键恢复默认。

---

## 安装

1. 打开 Edge，地址栏输入 `edge://extensions/`（Chrome 为 `chrome://extensions/`）。
2. 打开 **开发人员模式**。
3. 点击 **加载解压缩的扩展**，选择本仓库的 `course-downloader/` 目录。
4. 建议将扩展固定到工具栏。若之前装过旧版，请点"重新加载"。

要求 **Chromium 111+**（Edge 111+ / Chrome 111+，2023 年起的版本均满足）：主世界内容脚本依赖
Manifest V3 的 `content_scripts.world: "MAIN"` 能力。

---

## 使用速览

**下载课件**：打开课程资源页 → 点开要下载的课件预览（让页面把它加载出来）→ 点扩展图标 → 单个"下载"或勾选后"下载选中"。
列表中"已捕获数据"条目表示已拿到完整字节，可直接保存；直链条目下载失败时，重新打开一次该课件预览再试。

**视频**：进入视频页自动静音播放；切走窗口再回来会自动续播；需要声音就在播放器上取消静音。

---

## 技术栈

| 层 | 选型 | 说明 |
| --- | --- | --- |
| 扩展框架 | Chrome Extension **Manifest V3** | Service Worker 后台 + 双世界内容脚本 |
| 语言 | 原生 JavaScript（ES2020+） | **零依赖、无构建步骤**，无打包器/框架 |
| 网络观察 | `chrome.webRequest`（观察模式） | 读响应头做捕获判定，不阻塞、不篡改请求 |
| 页面注入 | `content_scripts.world: "MAIN"` | 主世界钩住页面 `fetch`/`XMLHttpRequest`，记录认证头并截获文件字节 |
| 跨世界通信 | `window.postMessage` + `CustomEvent` | 隔离世界 ↔ 主世界桥接（`CD_*` 协议） |
| DOM | `MutationObserver`、Shadow DOM 遍历 | 播放器/预览组件多为动态挂载 |
| 存储 | `chrome.storage.local` | 捕获列表、命名映射、设置、断点位置（配合 `unlimitedStorage`） |
| 媒体 | HTMLMediaElement、`blob:` URL、MSE 指纹 | 自动播放策略规避（muted）、断点键稳定化 |
| 测试 | Node.js 内置模块 | 自定义断言，**无测试框架依赖**；真实浏览器验证走 CDP（Node ≥ 22 内置 WebSocket） |
| 打包 | 任意 zip | `Compress-Archive` / `zip` 均可 |

---

## 架构与数据流

```
 网络层  chrome.webRequest(响应头) ─┐
 页面层  main.js(MAIN 世界钩子) ────┼──► scope.js 统一裁决 ──► background.js 汇总
 DOM 层  content.js(DOM 扫描) ──────┘        (唯一裁决点)        │ 去重 / 命名映射 / 视频配对
                                                                ▼
                                                    popup 列表 ── 下载链
 各上下文  logEvent(lv,tag,msg) -------------> CDLog（后台唯一写入点，环形缓冲）
                                                                v
                                                     popup 日志面板（过滤 / 导出 .txt）
```

关键设计决策：

- **`scope.js` 单一事实来源**：同一段 UMD 代码被后台（`importScripts`）、隔离世界、主世界与 Node 单测四个环境共用，
  从根上消除"各处正则不一致"这类 bug。
- **后台是唯一裁决点**：内容脚本/主世界只上报，是否入库一律由后台按规则判定，被拒原因写入调试面板（可观测性）。
- **预筛 + 精判两级**：明显静态资源不进入上下文判定（避免每个请求都查标签页）；可疑资源再带页面上下文精判。
- **断点续播的红线**：`shouldAutoResume()` 在"标签页仍隐藏"时恒为 false，该行为由单测断言固化，
  而非依赖约定。
- **跨域祖先链**：实测确认 `location.ancestorOrigins` 在多级跨域 iframe 中返回完整祖先链，
  据此识别"平台内嵌的永中预览框"（学习上下文）与"独立打开的预览页"（范围外）。
- **日志后台单写点**：`logger.js` 只做纯逻辑（规范化/截断/过滤/导出），落盘统一由后台节流写入，
  消除多上下文并发写 storage 的竞态。

---

## 项目结构

```
repo-root/
├─ course-downloader/          # 扩展源码（加载解压缩的扩展时选择本目录）
│  ├─ manifest.json            # MV3 清单
│  ├─ background.js            # Service Worker：webRequest 捕获、下载、命名映射、视频配对
│  ├─ scope.js                 # ★ 统一识别规则（单一事实来源）
│  ├─ content.js               # 隔离世界：DOM 扫描、消息桥、自动确认预览
│  ├─ main.js                  # 主世界：认证头记录、字节截获、HLS 下载、JSON 映射收割
│  ├─ video-autoplay.js        # 视频自动播放并静音
│  ├─ video-resume.js          # 断点续播
│  ├─ auto-complete.js         # 完成后自动提交（条件严格：确实读完/看完）
│  ├─ logger.js                # ★ 运行日志纯逻辑（环形缓冲/级别过滤/导出）
│  ├─ popup.html / popup.js    # 弹窗：三视图（文件/调试/日志）、批量下载进度、卡片布局
│  ├─ options.html / options.js# 设置页（独立标签页布局）
│  ├─ LICENSE
│  └─ README.md
└─ tests/                      # 纯 Node 单测 + 真实浏览器验证脚本
   ├─ scope.test.js            # 识别规则（94 项）
   ├─ video-resume.test.js     # 断点续播（38 项）
   ├─ video-autoplay.test.js   # 自动播放（27 项）
   ├─ options.test.js          # 设置逻辑 + 渲染度量（32 项）
   ├─ ancestor-probe.js        # 跨域 iframe 祖先链实测（6 项）
   ├─ capture-e2e.js           # 模拟课程页端到端过滤（15 项）
   ├─ ui-audit.js              # 布局审计：溢出/裁切/控件尺寸（8 项）
   └─ shots/                   # 渲染截图产物
```

---

## 权限说明

| 权限 | 用途 |
| --- | --- |
| `webRequest` | 观察响应头（content-type / content-disposition / content-length），识别课件资源 |
| `storage` / `unlimitedStorage` | 本机保存捕获列表、命名映射、设置与断点位置 |
| `downloads` | 调用浏览器下载（直链 / dataURL 保存） |
| `tabs` / `activeTab` | 弹出面板向当前课程页各框架派发下载/控制指令 |
| `webNavigation` | 枚举页面框架，向预览 iframe 派发抓取任务 |
| `scripting` | **当前代码未使用**（历史预留），发布前可移除 |
| `host_permissions: <all_urls>` | 课件可能由平台外的 CDN/预览服务商下发，宿主权限需覆盖这些域；**实际捕获范围仍由 `scope.js` 规则限制**在课程学习页面上下文 |

---

## 开发与测试

```bash
# 语法检查（零依赖，无需安装任何包）
node --check course-downloader/scope.js

# 运行纯逻辑单测（Node ≥ 18 即可）
node tests/scope.test.js
node tests/dedupe.test.js         # 重复条目合并规则
node tests/logger.test.js
node tests/video-resume.test.js
node tests/video-autoplay.test.js
node tests/auto-complete.test.js  # 完成后自动提交的判定与红线
node tests/options.test.js

# 后台集成测试（VM 内加载真实 background.js，stub chrome API）
# 覆盖：直链上报 + 转码字节保存的合并、范围外拒绝、重复上报/重复字节去重、类型识别
node tests/background-integration.test.js
# 启动清理：历史误收的页面条目、空白名条目
node tests/background-startup.test.js

# 真实浏览器验证（需本机安装 Chrome/Edge，Node ≥ 22；脚本内的浏览器与项目路径需按需调整）
node tests/ancestor-probe.js    # 跨域 iframe 祖先链实测
node tests/capture-e2e.js       # 模拟课程页：课件与噪声混合投放，核对过滤结果
node tests/ui-audit.js          # 设置页/弹窗布局审计并截图


## 隐私

- 所有判定、存储、下载均在**本机浏览器内**完成；扩展**没有任何网络上报、遥测或统计**。
- 本机持久化的数据：捕获列表（≤200 条）、命名映射、设置项、断点位置（≤200 条，45 天过期），均可在扩展面板一键清空。

---

## 已知限制

- 「完成后自动提交」依赖文案解析时长门槛。若要求只以图片/Canvas 呈现，插件只能走「点击后校验+重试」兜底，可能多等一会儿才成功（日志会显示每次的停留/要求时长）。

- MSE 类播放器的媒体地址是每次随机的 `blob:`，断点键退化为"框架路径 + 时长"指纹；同一页面存在多个**同时长**视频时可能撞键。
- 播放器若挂在**闭合** shadow root 内，内容脚本无法到达（open shadow root 已覆盖）。
- 课件若由平台转码为逐页图片预览（无原始文件直链），则无法还原出原始文档。
- Chrome/Edge 137+ 移除了 `--load-extension`，自动化联调需采用仓库内的等价验证脚本。
- 仓库内 `操作界面.png` / `设置界面.png` 为旧版界面截图，待更新。

---

## 版本历史

| 版本 | 要点 |
| --- | --- |
| 1.16.0 | 运行日志（面板/级别过滤/导出 txt，含存储兜底）；弹窗界面重构（三视图分页签、卡片布局、批量进度条、骨架屏、动画）；重复条目自动合并；新增完成后自动提交；修复重复捕获、漏捕、页面误收（下载成 .htm）、空白名条目（含启动自动清理）、切换视图滚动条闪现、头部按钮重复 |
| 1.15.0 | 断点续播；HLS 分片重试与"拒绝损坏文件"；设置页新增断点续播开关 |
| 1.14.0 | 设置页改为独立标签页（780px）并重设计；文件识别收紧（页面上下文 + 严格模式 + 站点白名单）；新增 `scope.js` 统一规则源 |
| 1.13.0 | 视频页自动播放并静音 |
| 1.12.2 | 基线版本：课件捕获/批量下载/真实文件名/HLS 下载/调试面板 |

---

## 许可证

本项目基于 [MIT License](./LICENSE) 开源。

```
MIT License

Copyright (c) 2026 [Icyyu7266]

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```
