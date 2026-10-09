# AGENTS.md — 第五人格区域选择模拟器

> 本文件供 AI Agent 阅读和记忆，用于快速理解本工程的架构、数据模型与修改方式。
> 全文行号对应版本：`第五人格区域选择模拟器.html`（共 2582 行，约 93 KB，已含宝箱视图与地窖视图）。
> 行号是**该版本的快照**——本工程改动频繁，行号会漂移；若与当前文件不符，请按函数名/关键字 grep 定位。

## 1. 项目概述

- **是什么**：《第五人格》（Identity V）赛事**地图区域选择 / 密码机分布模拟器**，单文件纯静态网页应用（无构建、无依赖、无后端）。
- **为谁而做**：北京大学第五人格赛事 PIL（Peking University Identity V League），作者"莲莲有鱼鱼"；页脚两行：`baseline@北京大学第五人格赛事@莲莲有鱼鱼@2026.7` 与 `修改版本@qh4869-20261008v0`。
- **核心用途**：模拟比赛中选择地图密码机组后，求生者/监管者选择区域；监管者选定区域后自动计算并高亮**封禁密码机**（离该区域中心最远的那台），右侧"密码机分析面板"展示每台机的名称、刷机特点、像素坐标、距离与战术提示。
- **三种并列视图**（top-bar 右侧按钮切换，互斥，默认密码机视图）：
  - **密码机视图**（默认）：密码机组图 + 区域框 + 密码机圈 + 封禁机圈 + 完整分析面板；
  - **宝箱视图**（"宝箱位置"按钮，琥珀色）：整图替换为 wiki 宝箱点位俯视图（每图一张**原图**，自带标记，**不做坐标标注、不列点位**）；
  - **地窖视图**（"地窖位置"按钮，翠绿色）：整图替换为 wiki 地窖刷点俯视图（每图一张 **1200px 缩略图**，同样不标注不列表）。
  - 后两种视图下密码机导航隐藏、右侧面板显示占位文案；**区域选择与区域框保留**，密码机/封禁机标注隐藏。
- **使用场景**：赛前 ban/pick 演练、密码机分布规律推导（如军工厂中场 DE 组）、规划破译优先级、查看宝箱/地窖刷点。

## 2. 仓库结构

```
D:\workspace\dwrg_sim\
├── 第五人格区域选择模拟器.html   # 唯一源码文件：CSS + HTML + JS 全部内联
├── README.md                      # 面向使用者的说明（功能/用法/致谢）
└── AGENTS.md                      # 本文件（面向 AI Agent 的记忆）
```

- **有 git 仓库**：`origin` = `git@github.com:qh4869/dwrg-sim.git`，主分支 `master`。
- **没有** `package.json`、构建脚本、测试框架、其他资源文件。
- 地图图片（含宝箱图、地窖图）、角色图标、字体均**运行时从公网 CDN 加载**。
- 修改本工程 = 编辑这一个 HTML 文件（README/AGENTS 为文档）。

## 3. 运行与验证

1. **直接运行**：浏览器（推荐 Chrome/Edge）打开该 HTML 即可，无需本地服务器。
2. **网络依赖**：必须联网。图片/字体缺失时走降级逻辑（图标降级为纯色圆点，图片显示"加载失败"并每 5 秒重试）。
3. **验证改动**：改完刷新页面，逐项检查：
   - 9 张地图卡片切换正常；每图 5~6 组密码机（‹ › 与圆点）切换正常；
   - 求生者点选 ≤4 区、监管者点选 1 区（单选，再点取消）；双方互斥提示正确；
   - 监管者选区域后出现**琥珀色虚线圆圈**（封禁机）且面板数字正确；切换密码机组后重算；
   - 右侧面板：最远机高亮（金黄）、锁定机（琥珀）、点击行后图中出现绿色虚线圆；
   - **宝箱视图**：点"宝箱位置"→ wiki 宝箱原图（不变形、永眠镇竖图正常）→ 密码机导航隐藏、面板"宝箱说明待补充"；
   - **地窖视图**：点"地窖位置"→ 1200px 地窖缩略图（永眠镇竖图）→ 面板"地窖说明待补充"；
   - **三视图互切**：宝箱↔地窖不串图；再点当前高亮按钮回密码机视图且区域选择不丢；任一视图下切地图都加载对应视图的新图；
   - 窗口缩放（含 ~700-800px 中等宽度，top-bar 会换行）画布重绘正常。
4. **无 lint/test 命令**。静态校验手段：提取 `<script>` 用 `node --check` 验语法；node eval `MAPS_DATA`/`CIPHER_DATA` 验数据完整性（长度、chest/cellar 字段、下标范围、zoneNames 对齐）；图片 URL 用 web_fetch 验可达（返回 `image/png` 即成功）。

## 4. 整体架构

```
┌─ <head> <style>（约 1–1150 行）──────────────────────────┐
│ :root 设计令牌；背景层 .bg-noise/.particles/body::before   │
│ 组件：header、map-selector/map-card、top-bar、cipher-nav、  │
│       side-toggle、chest-btn/.cellar-btn（视图开关）、      │
│       map-viewer/map-canvas、action-bar/legend、           │
│       distance-panel、view-placeholder、toast、footer      │
│ 响应式断点：938 / 804 / 686 / 429px（≤804 top-bar 换行，   │
│             ≤686 整体单列堆叠）                             │
└──────────────────────────────────────────────────────────┘
┌─ <body> HTML 骨架（约 1152–1299 行）──────────────────────┐
│ header → main-container[ map-selector → content-grid[       │
│   main-left[ top-bar(地图名/密码机导航/阵营/宝箱/地窖) +    │
│     map-viewer(loading-overlay + img+canvas + action-bar) ] │
│   distance-panel(密码机列表 + 监管者位置卡 + 提示卡 +        │
│     view-placeholder) ] ] → footer → toast-message          │
└──────────────────────────────────────────────────────────┘
┌─ <script>（约 1301–2582 行）──────────────────────────────┐
│ 数据层：initParticles(1301)、MAPS_DATA(1319–1611)、         │
│         CIPHER_DATA(1613–1742)                             │
│ 逻辑层(1744–2582)：派生数据 → 全局状态 → 画布渲染 →         │
│                    交互回调 → 初始化入口                    │
└──────────────────────────────────────────────────────────┘
```

**数据流**：常量 `MAPS_DATA` + `CIPHER_DATA` → 初始化算**相对坐标**（0~1，与显示尺寸解耦）→ 交互只改 `survivorState/hunterState/viewMode` → `drawCanvas()`（重绘）与 `updateDistancePanel()`（面板重算）负责全部 UI 刷新。

## 5. 数据模型（最关键）

### 5.1 `MAPS_DATA`（约 1319–1611 行）— 9 张地图

数组下标即地图 id（0–8），图标写死在 `initMapSelector()`（`['🏭','🏥','⛪','🏖️','🎡','🏠','🏮','🏯','🌲']`）：

| id | 名称 | size (px) | 宝箱图 chestSize | 地窖图 cellarSize | 密码机组数 | 区域数 |
|----|------|-----------|------------------|-------------------|-----------|--------|
| 0 | 军工厂 | 699×600 | 1481×1247 | 2401×2062 | 5 | 9 |
| 1 | 圣心医院 | 644×604 | 1352×1281 | 2213×2074 | 5 | 9 |
| 2 | 红教堂 | 596×599 | 1278×1272 | 2047×2062 | 5 | 9（错位网格，见 §6） |
| 3 | 湖景村 | 750×750 | 1320×1287 | 2062×2073 | 5 | 12（4×3） |
| 4 | 月亮河公园 | 750×523 | 1849×1261 | 2335×1620 | 5 | 12（4×3） |
| 5 | 里奥的回忆 | 750×632 | 1503×1263 | 2232×1896 | 5 | 9 |
| 6 | 永眠镇 | 885×750 | 1231×1426 | 1794×2075 | 5 | 10（3×3+额外第10区，见 §6） |
| 7 | 唐人街 | 750×649 | 1483×1289 | 2198×1893 | **6** | 9 |
| 8 | 不归林 | 750×773 | 898×925 | 2060×2090 | 5 | 9 |

每张地图对象字段：

| 字段 | 含义 |
|------|------|
| `id` / `name` / `nameEn` | 地图 id（=数组下标）、中/英文名 |
| `size` | **密码机组原图像素尺寸 [宽,高]** —— 密码机坐标、区域网格的基准 |
| `chestSize` | **宝箱俯视图尺寸**（wiki **原图**尺寸）—— 宝箱视图容器高度按它算 |
| `cellarSize` | **地窖刷点图尺寸**（wiki **原图**尺寸）—— 地窖视图容器高度按它算。注意 `cellarUrl` 实际加载 **1200px 缩略图**（原图 3.6~7.3 MB 太重），宽高比与原图一致，故只需比例正确 |
| `zonesInfo` | `[x_init, y_init, w, h, num_h, num_v]`，**密码机原图像素**值：网格起点、单格宽高、横向/纵向格数 |
| `ciphersCoords` | 5~6 个"密码机组"（下标=组号）。每组 **7 个整数下标**，索引 `CIPHER_DATA[id]`。**是索引不是坐标** |
| `urls` | 与 `ciphersCoords` 等长的密码机地图图 URL（每组一张） |
| `chestUrl` | 宝箱图 URL（每图一张，**原图**）。多为 `<图名> 俯视图 箱子点位.png`；**不归林特殊**：`不归林-俯视图-箱子.png` |
| `cellarUrl` | 地窖图 URL（每图一张，**1200px 缩略图**）：`.../thumb/<a>/<ab>/<hash>.png/1200px-<图名>_俯视图_地窖刷点.png`（中文已 percent-encode）。该 wiki **1500px 缩略图 404**，1200px 可用（9 张已实测） |
| `fixedHint` | 地图总体密码机规律（提示卡上半，恒定） |
| `hints` | 与 `urls` 等长，逐组战术提示（空串则隐藏） |
| `zoneNames` | 区域名，顺序同 `calculateMapsZonesRelative()` 生成顺序（i 横向外层、j 纵向内层，index=i*num_v+j；特判追加区在最后） |

### 5.2 `CIPHER_DATA`（约 1613–1742 行）— 每图候选密码机点

每元素 `{ point: [x, y], name: '沙包', feature: 'A组 必刷机' }`：`point` 是**密码机原图像素坐标**（人工标注，改错会直接算错封禁机）；`feature` 直接展示在"刷机特点"列；每图 11~15 个，`ciphersCoords` 下标都落在范围内。

**宝箱与地窖都没有坐标数据**——两种视图只整图替换，不标注、不列表（设计决策，YAGNI）。

### 5.3 派生数据（启动时算一次，全局只读）

| 函数（行号） | 产物 | 说明 |
|---|---|---|
| `calculateMapsZonesRelative()` (1744) | `mapsZonesRelative[mapId][] = {x,y,width,height}`（0~1） | `zonesInfo ÷ size`；含 id=2、6 特判（§6） |
| `calculateCiphersCoordsRelative()` (1789) | `ciphersCoordsRelative[mapId][group][] = [x,y]`（0~1） | `CIPHER_DATA[mapId][idx].point ÷ size` |
| `calculateZoneCenters(zones)` (1811) | `[{x,y}]` 区域中心 | 纯函数 |
| `calculateLockCipher(hunterPos, ciphers, mapSize)` (1815) | 最远密码机下标 | **封禁规则**：`dx=(hunter.x-c.x)*imgW`、`dy=(hunter.y-c.y)*imgH`，欧氏距离最大者（按像素宽高分别缩放，校正宽高比） |

## 6. 全局状态与交互规则

**状态变量（约 1833–1865 行）**：

- `currentMapId`、`currentSide`（`1`=求生者，`-1`=监管者）、`currentCipherIndex`（当前密码机组）；
- **`viewMode`（字符串三态：`'cipher'` 默认 / `'chest'` / `'cellar'`）** —— 已取代早期的 `chestMode` 布尔；
- `survivorState` / `hunterState`：长度=区域数的数组，`survivorState[i]>0` 为求生者选中，`hunterState[i]<0` 为监管者选中（互斥，`changeStates()` 内互相拦截并 toast）；
- `flagLock`、`lockCipherIndex`（当前封禁机）、`selectedCipherIndex`（面板点选的绿色高亮机）；
- 画布：`canvasCtx/canvasWidth/canvasHeight`、`survivorImg/hunterImg` 及 loaded 标志、重试定时器。

**关键交互逻辑**：

- `changeStates(index)` (2362)：求生者最多 4 个（达上限静默不选）；监管者**单选**（选新自动换旧，点已选则取消并清空封禁）。
- 监管者选区域 → 立即算封禁机 → `flagLock=true`；切换密码机组时 `prevCipher/nextCipher` → `checkLock()` (2301) 重算。
- `toggleViewMode(mode)` (2251)：`viewMode = viewMode === mode ? 'cipher' : mode`（点当前高亮按钮回密码机视图）；同步 `chestBtn`/`cellarBtn` 的 `active` 类与 `cipherNav` 显隐，再 `updateDistancePanel()` + `loadMapImage()` + `drawCanvas()`。**切换地图时视图保持**（`selectMap` 不改 `viewMode`）。
- `updateDistancePanel()` (2419)：面板总重算。**非密码机视图分支在最前**（2429）：按 `viewMode` 设置 `#viewPlaceholderTitle`/`#viewPlaceholderText`（📦 宝箱说明 / 🕳️ 地窖说明），隐藏 `#distanceGrid`、显示 `#viewPlaceholder` 后 return。密码机视图：地图尺寸、监管者区域名/中心像素坐标、封禁机名称/像素距离（`toFixed(1)+' px'`）、每台机的行（编号/名称/刷机特点/像素坐标），打 `farthest`/`locked`/`selected-cipher` 高亮类，刷新固定/动态提示；行点击切换 `selectedCipherIndex`。
- `handleMapClick` (2323)：点击坐标转相对坐标，**按数组顺序命中第一个**包含它的区域 → `changeStates` + 区域中心水波纹（约 536 ms 后移除）。宝箱/地窖视图下同样生效。
- `resetSelection()` (2541)：清空全部选择与封禁/选中态。`toggleSide()` (2238)：切换阵营。
- 窗口 `resize` → `setMapContainerHeight()` + `initCanvas()` 按 `devicePixelRatio` 重建画布（视图感知）。

**特殊地图/视图特判（改代码时极易踩坑）**：

| 位置 | 对象 | 行为 |
|---|---|---|
| `calculateMapsZonesRelative()` (1753) | **id 2 红教堂** | 网格非矩形：`y = y_init - i*0.125*h + j*h`（每右移一列整体上移 12.5% 格高，错位/阶梯） |
| `calculateMapsZonesRelative()` (1775) | **id 6 永眠镇** | 3×3 之外**追加第 10 区**：`x = x_init+3w, y = y_init, w = 1.19w, h = 1.7h`（右侧"墓园"竖条），故 `zoneNames` 有 10 项 |
| `setMapContainerHeight()` (1877) | **宝箱/地窖视图** | 尺寸源三选一：`chest`→`chestSize`、`cellar`→`cellarSize`、否则 `size`（`MAPS_DATA[currentMapId]` 只取一次存 `mapInfo`） |
| `setMapContainerHeight()` (1883) | **id 6 永眠镇** | **仅密码机视图**（`viewMode === 'cipher' && currentMapId === 6`）按 `宽*(imgW/imgH)` 铺满横容器；宝箱/地窖视图一律普通 `高/宽` 比例（wiki 图原生方向即最终方向：宝箱 1231×1426、地窖 1794×2075 都是竖图） |
| `drawCanvas()` (2086) | **id 6 永眠镇** | **仅密码机视图** `rotate(-90°)` 画图；其余视图走普通 `drawImage` 全幅分支 |
| `drawCanvas()` (2177 / 2206) | **宝箱/地窖视图** | 跳过封禁机圈（琥珀虚线）与选中密码机圈（绿色虚线）；区域边框、求生者/监管者图标照常 |
| `getImageUrl()` (2553) | **三态取图** | `chest`→`chestUrl`；`cellar`→`cellarUrl`；否则 `urls[cipherIndex]` |
| `getImageDrawRect()` (1893) | — | **死代码**：已定义但全文件无调用 |
| `chooseState` (1831) | — | **冗余变量**：只写不读 |

**外部依赖与降级（约 1922–1942、2056–2093 行）**：

- 角色图标：`patchwiki.biligame.com` 两张 PNG，`crossOrigin='anonymous'`；`onerror` → `loaded=false` 且 **3350 ms** 后重试；未加载时画布降级为纯色圆点（求生者 `#00f0ff`、监管者 `#ff2a6d`）。
- 地图图片：密码机图多为 `patchwiki.biligame.com`，两张在 `i0.hdslb.com`（湖景村第 3 组、月亮河公园第 1 组）；宝箱图为 wiki 原图；地窖图为 wiki 1200px 缩略图。预加载失败显示 `.image-error` 并 **5000 ms** 重试（`loadMapImage` 三视图通用）。
- 字体：CSS 顶部 `@import` Google Fonts `Noto Sans SC`，失败回退系统字体栈。
- "魔法毫秒数"（3350/5000/1340/536）是原作者风格，保持即可。

## 7. 常见修改任务指南

| 任务 | 要做的事 |
|---|---|
| **新增一张地图** | `MAPS_DATA` 尾部追加（id 递增）：（1）`CIPHER_DATA` 追加全部点位；（2）`ciphersCoords` 每组 7 个下标且都在新 `CIPHER_DATA` 范围内；（3）`urls` 与 `ciphersCoords` 等长；（4）`hints` 与 `urls` 等长（可空串）；（5）`zoneNames` 顺序匹配网格遍历（错位/追加区要在 `calculateMapsZonesRelative` 加特判并补名）；（6）`initMapSelector()` 的 `icons` 加 emoji；（7）补 `chestSize`/`chestUrl` 与 `cellarSize`/`cellarUrl`（wiki API 搜 `箱子点位` / `地窖刷点`） |
| **新增/调整密码机组** | 该图 `ciphersCoords` 加一组 7 下标 + `urls` 加一张同尺寸图 + `hints` 加一条 |
| **换宝箱/地窖图 URL** | 改 `chestUrl` / `cellarUrl`；**尺寸同步改 `chestSize` / `cellarSize`**（容器高度按它算，写错会变形）。地窖换尺寸档位前先实测 wiki 是否支持（1200px 可用、1500px 404） |
| **新增第四种视图** | 沿用三态模式：`viewMode` 加取值 + `MAPS_DATA` 加 `xxxUrl/xxxSize` + top-bar 加按钮（复用 `.chest-btn, .cellar-btn` 基类）+ `getImageUrl`/`setMapContainerHeight`/`drawCanvas` 分支 + `updateDistancePanel` 占位文案 + 事件绑定 |
| **扩充宝箱/地窖说明** | 改 `updateDistancePanel` 内 `#viewPlaceholderTitle`/`#viewPlaceholderText` 的按视图文案；若要升级为点位列表+画布标注，需新建坐标数据（以对应视图图尺寸为基准）并在 `drawCanvas` 对应分支绘制——三种视图坐标系互不通用 |
| **改提示文案** | 改 `fixedHint`（全组恒定）或 `hints[i]`（第 i 组），纯文本 |
| **调封禁判定** | 只动 `calculateLockCipher()`；面板展示在 `updateDistancePanel()` |
| **换密码机图片** | 可换 URL，但新图**必须与原图同尺寸**（`size` 与像素坐标依赖原图），否则封禁机与区域框全错 |
| **改样式** | 全在 `<style>`：改 `:root` 变量即可全局换色；断点 938/804/686/429px（804 起 top-bar 换成 3 列 + 按钮换第二行）；`.chest-btn` 激活琥珀、`.cellar-btn` 激活翠绿（共用基类 `.chest-btn, .cellar-btn`）；`.cipher-row` 是 grid 四列模板 |
| **改画布观感** | `drawCanvas()` (2074)：地图→区域边框→求生者→监管者→(仅密码机视图)封禁机(琥珀虚线)→(仅密码机视图)选中机(绿色虚线) |

## 8. Agent 注意事项（务必记住）

1. **单文件工程**：源码改动都在 `第五人格区域选择模拟器.html`；不要新建构建体系、不要拆文件（除非用户明确要求）。仓库已有 git（origin 指向 GitHub）。
2. **三套坐标系互不通用**：`CIPHER_DATA[].point`/`zonesInfo`/`size` 是密码机图坐标系；`chestSize`、`cellarSize` 分别是宝箱图、地窖图坐标系。改任一个都要同步核对。
3. **`ciphersCoords` 是索引不是坐标**：值是 `CIPHER_DATA[mapId]` 的下标，换算在 `calculateCiphersCoordsRelative()`。
4. **id 特判三处**：红教堂错位（id 2）、永眠镇竖图旋转 + 第 10 区（id 6）；**旋转只在密码机视图生效**，宝箱/地窖视图绕过——通用逻辑要显式判断 `viewMode === 'cipher'`。
5. **三视图是设计决策**：只整图替换、不标注不列表、面板占位、区域选择保留。若用户要升级（编号标注/距离分析/联动），属新需求，重新走设计确认。
6. **地窖图用缩略图**：wiki 原图 3.6~7.3 MB，`cellarUrl` 一律 1200px 缩略图；`cellarSize` 仍写原图尺寸（只用于宽高比）。宝箱图则是原图。
7. **中文语境**：UI 文案、注释、数据（`name/feature/hint/zoneNames`）全部中文；保持中文输出与命名习惯。
8. **无测试**：验证 = 浏览器手点一遍（§3 清单）+ `node --check` + 数据完整性脚本 + 图片 URL 可达性抽查。
9. **封禁机=最远机**是产品核心规则，改动前先与用户确认。
10. 首屏即 `selectMap(0)`（军工厂第 1 组），调试入口在文件末尾 `initMapSelector(); selectMap(0)`。

## 9. 函数速查（`<script>` 逻辑层）

| 函数 | 行号 | 职责 |
|---|---|---|
| `initParticles` | 1301 | 生成 30 个背景粒子 div |
| `calculateMapsZonesRelative` | 1744 | zonesInfo→相对区域网格（含 id2 错位、id6 追加区） |
| `calculateCiphersCoordsRelative` | 1789 | CIPHER_DATA→相对密码机坐标 |
| `calculateZoneCenters` | 1811 | 区域中心点 |
| `calculateLockCipher` | 1815 | 求最远机=封禁机 |
| `setMapContainerHeight` | 1873 | 容器高度（按 `viewMode` 选 size；id6 旋转仅密码机视图） |
| `getImageDrawRect` | 1893 | 死代码，未调用 |
| `loadSurvivorImg` / `loadHunterImg` | 1922 / 1942 | 图标加载+重试 |
| `showToast` | 1965 | 顶部 toast（约 1.34s 消失） |
| `initMapSelector` / `selectMap` | 1973 / 1991 | 地图卡片与切换（视图保持） |
| `loadMapImage` | 2021 | 地图图预加载+失败重试（三视图通用） |
| `initCanvas` / `drawCanvas` | 2056 / 2074 | 画布初始化 / 全量重绘（非密码机视图跳过密码机相关层） |
| `toggleSide` | 2238 | 阵营切换 |
| `toggleViewMode` | 2251 | 三视图切换（点当前视图回密码机） |
| `prevCipher` / `nextCipher` | 2261 / 2272 | 密码机组切换 |
| `updateCipherText` / `updateCipherDots` | 2283 / 2290 | 密码机导航文案与圆点 |
| `checkLock` | 2301 | 封禁机重算 |
| `handleMapClick` / `changeStates` | 2323 / 2362 | 点选区域与互斥规则 |
| `updateSelectionInfo` | 2411 | 顶部计数 |
| `updateDistancePanel` | 2419 | 右侧面板总重算（非密码机视图显示占位文案） |
| `resetSelection` | 2541 | 重置 |
| `getImageUrl` | 2553 | 按 `viewMode` 取图（密码机组图 / chestUrl / cellarUrl） |
