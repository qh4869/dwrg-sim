# AGENTS.md — 第五人格区域选择模拟器

> 本文件供 AI Agent 阅读和记忆，用于快速理解本工程的架构、数据模型与修改方式。
> 全文所述行号对应当前版本：`第五人格区域选择模拟器_20260809.html`（共 2527 行，约 90 KB，已含宝箱模式）。

## 1. 项目概述

- **是什么**：《第五人格》（Identity V）赛事**地图区域选择 / 密码机分布模拟器**，单文件纯静态网页应用（无构建、无依赖、无后端）。
- **为谁而做**：北京大学第五人格赛事 PIL（Peking University Identity V League），作者"莲莲有鱼鱼"；页脚为两行：`baseline@北京大学第五人格赛事@莲莲有鱼鱼@2026.7`（基准署名）与 `修改版本@qh4869-20261008v0`（修改版本）；文件名后缀 `20260809` 为构建日期。
- **核心用途**：模拟比赛中选择地图密码机组后，求生者/监管者选择区域的过程；当监管者选定区域时，自动计算并高亮**封禁密码机**（离监管者区域中心最远的那台），并在右侧"密码机分析面板"展示每台机的名称、刷机特点、像素坐标、距离与战术提示。
- **宝箱模式**（新增功能）：top-bar 右侧"宝箱位置"按钮切换到宝箱视图——地图区整图替换为 bilibili wiki 的宝箱点位俯视图（每图一张，原图自带标记，**不做坐标标注、不列点位**），右侧面板切换为占位文案"宝箱说明待补充"（后续扩充）；区域选择在该模式下保留，密码机/封禁机标注隐藏。
- **使用场景**：赛前 ban/pick 演练、判断密码机分布规律（如军工厂中场 DE 组推导全图）、规划破译优先级、查看宝箱刷点分布。

## 2. 仓库结构

工程根目录只有两个文件：

```
D:\workspace\dwrg_sim\
├── 第五人格区域选择模拟器_20260809.html   # 唯一源码文件：CSS + HTML + JS 全部内联
└── AGENTS.md                              # 本文件
```

- **没有** `package.json`、构建脚本、测试、git 仓库、其他资源文件。
- 所有地图图片（含宝箱图）、角色标记图标、字体均**运行时从公网 CDN 加载**（见 §6）。
- 修改本工程 = 编辑这一个 HTML 文件。

## 3. 运行与验证

1. **直接运行**：用浏览器（推荐 Chrome/Edge）打开该 HTML 文件即可，无需本地服务器。
2. **网络依赖**：必须联网。Google Fonts（Noto Sans SC）、biligame wiki 地图图片、bilibili CDN 图标缺失时会走降级逻辑（图标降级为纯色圆点，图片显示"加载失败"并每 5 秒重试）。
3. **验证改动**：改完后刷新页面，逐项检查：
   - 顶部 9 张地图卡片切换是否正常；每张图 5~6 组密码机（‹ › 按钮与圆点）切换是否正常；
   - 求生者模式点选 ≤4 个区域、监管者模式点选 1 个区域（单选，再次点击取消）；
   - 监管者选区域后出现**琥珀色虚线圆圈**（封禁机）且面板数字正确；切换密码机组后封禁机重新计算；
   - 右侧面板：最远机高亮（金黄）、锁定机高亮（琥珀）、点击行选中后图中出现绿色虚线圆；
   - **宝箱模式**：点"宝箱位置"→ 换成 wiki 宝箱原图（不变形，永眠镇为竖图正常显示）→ 密码机导航隐藏、面板显示"宝箱说明待补充"；区域点击/图标仍可用；再点一次切回密码机图且选择不丢；宝箱模式下切换地图加载新图宝箱图；
   - 窗口缩放时画布重绘正常。
4. **无 lint/test 命令**。可做静态校验：提取 `<script>` 块用 `node --check` 验语法；用 node eval `MAPS_DATA`/`CIPHER_DATA` 验数据完整性（长度、下标范围、zoneNames 对齐）。

## 4. 整体架构

单文件内部分四层（行号为约数）：

```
┌─ <head> <style>（约 1–1122 行）──────────────────────────┐
│ 设计令牌 CSS 变量 :root（--primary 等主题色）              │
│ 背景层：.bg-noise / .particles / body::before 浮动动画     │
│ 组件：header、map-selector/map-card、top-bar、cipher-nav、  │
│       side-toggle、chest-btn（宝箱开关）、map-viewer/       │
│       map-canvas、action-bar/legend、distance-panel、       │
│       chest-placeholder（宝箱占位）、toast、footer          │
│ 响应式断点：938px / 804px / 686px / 429px（≤686px 上下堆叠）│
└──────────────────────────────────────────────────────────┘
┌─ <body> HTML 骨架（约 1124–1271 行）──────────────────────┐
│ header（标题）→ main-container[ map-selector 卡片列表        │
│  → content-grid[ main-left[ top-bar(地图名/密码机导航/阵营切换│
│    /宝箱按钮) + map-viewer(loading-overlay + img+canvas     │
│      + action-bar(重置/图例/计数)) ]                          │
│    distance-panel(密码机列表 + 监管者位置卡 + 提示卡          │
│      + chest-placeholder) ] ]                                │
│ → footer（署名+版本）→ toast-message                          │
└──────────────────────────────────────────────────────────┘
┌─ <script> 数据层（约 1273–1646 行）─────────────────────────┐
│ MAPS_DATA：9 张地图的元数据（含 chestSize/chestUrl，见 §5）  │
│ CIPHER_DATA：每张图的全部候选密码机点（point+name+feature）    │
└──────────────────────────────────────────────────────────┘
┌─ <script> 逻辑层（约 1648–2525 行）─────────────────────────┐
│ 派生数据 → 全局状态 → 画布渲染 → 交互回调 → 初始化入口        │
└──────────────────────────────────────────────────────────┘
```

**数据流**：静态常量 `MAPS_DATA` + `CIPHER_DATA` → 初始化时计算**相对坐标**（0~1，与显示尺寸解耦）→ 交互只改 `survivorState/hunterState/chestMode` 等状态 → `drawCanvas()`（地图重绘）与 `updateDistancePanel()`（面板重算）两个渲染函数负责全部 UI 刷新；`loadMapImage()` 按 `chestMode` 决定加载密码机组图还是宝箱图。

## 5. 数据模型（最关键）

### 5.1 `MAPS_DATA`（约 1273–1546 行）— 9 张地图

数组下标即地图 id（0–8），图标固定写死在 `initMapSelector()`（`['🏭','🏥','⛪','🏖️','🎡','🏠','🏮','🏯','🌲']`）：

| id | 名称 | 英文名 | size (px) | 宝箱图 chestSize (px) | 密码机组数 | 区域数 |
|----|------|--------|-----------|------|-----------|--------|
| 0 | 军工厂 | Arms Factory | 699×600 | 1481×1247 | 5 | 9 |
| 1 | 圣心医院 | Sacred Heart Hospital | 644×604 | 1352×1281 | 5 | 9 |
| 2 | 红教堂 | The Red Church | 596×599 | 1278×1272 | 5 | 9（错位网格，见 §6） |
| 3 | 湖景村 | Lakeside Village | 750×750 | 1320×1287 | 5 | 12（4×3） |
| 4 | 月亮河公园 | Moonlit River Park | 750×523 | 1849×1261 | 5 | 12（4×3） |
| 5 | 里奥的回忆 | Leo's Memory | 750×632 | 1503×1263 | 5 | 9 |
| 6 | 永眠镇 | Eversleeping Town | 885×750 | 1231×1426 | 5 | 10（3×3+额外第10区，见 §6） |
| 7 | 唐人街 | China Town | 750×649 | 1483×1289 | **6** | 9 |
| 8 | 不归林 | Darkwoods | 750×773 | 898×925 | 5 | 9 |

每张地图对象字段：

| 字段 | 含义 |
|------|------|
| `id` | 地图 id（与数组下标一致） |
| `name` / `nameEn` | 中/英文名 |
| `size` | **地图原图（密码机组图）像素尺寸 [宽, 高]** —— 密码机像素坐标、区域网格都以它为基准 |
| `chestSize` | **宝箱俯视图像素尺寸 [宽, 高]**（wiki 原图尺寸）—— 宝箱模式下容器高度按它计算；**与 `size` 不同图、不同裁切，宽高比有 0~2% 差异**（永眠镇差异大：竖图） |
| `zonesInfo` | `[x_init, y_init, w, h, num_h, num_v]`，均为**密码机原图像素**值：区域网格起点、单个区域宽高、横向/纵向格数 |
| `ciphersCoords` | 5~6 个"密码机组"（数组下标=组号）。每组是 **7 个整数下标**，索引 `CIPHER_DATA[id]`。**它不是坐标，是索引** |
| `urls` | 与 `ciphersCoords` 等长的密码机地图图 URL 列表（每组一张图） |
| `chestUrl` | 宝箱俯视图 URL（每图一张，wiki 来源；永眠镇文件名特殊：`不归林-俯视图-箱子.png`，其余为`<图名> 俯视图 箱子点位.png`） |
| `fixedHint` | 该地图的总体密码机规律说明（提示卡上半部分，恒定） |
| `hints` | 与 `urls` 等长，逐组的战术提示（提示卡下半部分，空字符串则隐藏） |
| `zoneNames` | 区域名称，顺序必须与 `calculateMapsZonesRelative()` 生成顺序一致（横向 i 外层、纵向 j 内层，index = i*num_v + j；特判追加的区在最后） |

### 5.2 `CIPHER_DATA`（约 1548–1676 行）— 每张图的候选密码机点

每元素：`{ point: [x, y], name: '沙包', feature: 'A组 必刷机' }`

- `point`：**密码机原图像素坐标**（人工标注，不要用尺子量着改，改错会直接算错封禁机）。
- `name`：密码机点位名；`feature`：刷机规律描述，直接展示在面板"刷机特点"列。
- 数量每图不同（11~15），`ciphersCoords` 里的下标都落在范围内。
- **注意：宝箱点位没有坐标数据**——宝箱模式只整图替换，不标注、不列表（设计决策，YAGNI）。

### 5.3 派生数据（启动时一次性计算，全局只读）

| 函数（行号约） | 产物 | 说明 |
|---|---|---|
| `calculateMapsZonesRelative()` (1679) | `mapsZonesRelative[mapId][] = {x,y,width,height}`（0~1 相对值） | 由 `zonesInfo` ÷ `size` 得到；含 id=2、6 特判（§6） |
| `calculateCiphersCoordsRelative()` (1724) | `ciphersCoordsRelative[mapId][group][] = [x,y]`（0~1） | 由 `CIPHER_DATA[mapId][idx].point` ÷ `size` 得到 |
| `calculateZoneCenters(zones)` (1746) | `[{x,y}]` 区域中心 | 纯函数 |
| `calculateLockCipher(hunterPos, ciphers, mapSize)` (1750) | 最远密码机下标 | **封禁规则**：`dx=(hunter.x-c.x)*imgW`、`dy=(hunter.y-c.y)*imgH`，欧氏距离最大者（按像素宽高分别缩放，校正图像宽高比） |

## 6. 全局状态与交互规则

**状态变量（约 1772–1800 行）**：

- `currentMapId`、`currentSide`（`1`=求生者，`-1`=监管者）、`currentCipherIndex`（当前密码机组）；
- `chestMode`（布尔，`false`=密码机视图，`true`=宝箱视图）；
- `survivorState` / `hunterState`：长度=区域数的数组，`survivorState[i]>0` 表示该区被求生者选中，`hunterState[i]<0` 表示被监管者选中（互斥，`changeStates()` 里互相拦截并弹 toast）；
- `flagLock`、`lockCipherIndex`（当前封禁机）、`selectedCipherIndex`（面板中点选的绿色高亮机）；
- 画布相关：`canvasCtx/canvasWidth/canvasHeight`、`survivorImg/hunterImg` 及其 loaded 标志、重试定时器。

**关键交互逻辑**：

- `changeStates(index)` (2312)：求生者最多选 4 个（达上限静默不选）；监管者**单选**（选新区域自动换旧，点到已选区域则取消并清空封禁）。
- 监管者选区域 → 立即算封禁机（最远机）→ `flagLock=true`；切换密码机组时 `prevCipher/nextCipher` → `checkLock()` (2251) 重算。
- `toggleChestMode()` (2202)：切换宝箱模式——按钮激活态、隐藏/恢复密码机导航（`cipherNav`）、`updateDistancePanel()`（面板切占位）、`loadMapImage()`（按模式取图）、`drawCanvas()`。**切换地图时模式保持**（`selectMap` 不改 `chestMode`）。
- `updateDistancePanel()` (2369)：面板总重算函数。**宝箱模式分支在最前**：隐藏 `#distanceGrid`、显示 `#chestPlaceholder`（占位文案"宝箱说明待补充"）后直接 return。密码机模式下：地图尺寸、监管者区域名/中心像素坐标、封禁机名称/像素距离（`toFixed(1)+' px'`）、每台机的行（编号/名称/刷机特点/像素坐标），打 `farthest`/`locked`/`selected-cipher` 高亮类，刷新固定/动态提示；行点击切换 `selectedCipherIndex`。
- `handleMapClick` (2273)：把点击坐标转相对坐标，**按数组顺序命中第一个**包含它的区域 → `changeStates` + 区域中心水波纹（约 536 ms 后移除）。宝箱模式下依然生效（区域选择保留）。
- `resetSelection()` (2478)：清空全部选择与封禁/选中态。
- `toggleSide()` (2190)：切换阵营。
- 窗口 `resize` → `setMapContainerHeight()` + `initCanvas()` 按 `devicePixelRatio` 重建画布（均模式感知）。

**特殊地图/模式特判（改代码时极易踩坑）**：

| 位置 | 对象 | 行为 |
|---|---|---|
| `calculateMapsZonesRelative()` (1679) | **id 2 红教堂** | 网格不是矩形：`y = y_init - i*0.125*h + j*h`（每向右一列整体上移 12.5% 格高，形成错位/阶梯） |
| `calculateMapsZonesRelative()` (1699) | **id 6 永眠镇** | 3×3 之外**追加第 10 个区域**：`x = x_init+3w, y = y_init, w = 1.19w, h = 1.7h`（右侧"墓园"竖条），故 `zoneNames` 有 10 项 |
| `setMapContainerHeight()` (1834) | **id 6 永眠镇** | 仅密码机模式：原图竖图（885×750）按 `宽*(imgW/imgH)` 铺满横容器 |
| `setMapContainerHeight()` (1830) | **宝箱模式** | 一律用 `chestSize` 且**不做旋转特判**：wiki 宝箱图原生方向即最终方向（永眠镇宝箱图 1231×1426 竖图，直接按竖图比例） |
| `drawCanvas()` (2037) | **id 6 永眠镇** | 仅密码机模式：`rotate(-90°)` 画图；宝箱模式走普通 `drawImage` 全幅分支 |
| `drawCanvas()` (2128/2157) | **宝箱模式** | 跳过封禁机圈（琥珀虚线）与选中密码机圈（绿色虚线）；区域边框、求生者/监管者图标照常绘制 |
| `getImageUrl()` (2499) | **宝箱模式** | 返回 `MAPS_DATA[mapId].chestUrl`，否则返回密码机组 URL |
| `getImageDrawRect()` (1825) | — | **死代码**：已定义但全文件无调用，勿依赖 |
| `chooseState` (1783) | — | **冗余变量**：只写不读，别指望它驱动逻辑 |

**外部依赖与降级（约 1850–1888、1949–1982 行）**：

- 角色图标：`patchwiki.biligame.com` 两张 PNG（survivor/hunter），`crossOrigin='anonymous'`；`onerror` → 置 `loaded=false` 并 **3350 ms** 后重试；未加载成功时画布降级为纯色圆点（求生者 `#00f0ff`、监管者 `#ff2a6d`）。
- 地图图片：密码机图 `urls` 指向 `patchwiki.biligame.com`（多数）与 `i0.hdslb.com`（湖景村第 3 组、月亮河公园第 1 组）；宝箱图 `chestUrl` 全部来自 `patchwiki.biligame.com`。预加载失败显示 `.image-error` 并 **5000 ms** 重试（`loadMapImage` 对两种模式通用）。
- 字体：CSS 顶部 `@import` Google Fonts `Noto Sans SC`，失败时回退系统字体栈。
- "魔法毫秒数"（3350/5000/1340/536）遍布代码，是原作者风格，保持即可。

## 7. 常见修改任务指南

| 任务 | 要做的事 |
|---|---|
| **新增一张地图** | 在 `MAPS_DATA` 尾部追加对象（id 顺序递增）：（1）`CIPHER_DATA` 追加该图全部点位；（2）`ciphersCoords` 每组 7 个下标且都指向新 `CIPHER_DATA` 条目；（3）`urls` 与 `ciphersCoords` 等长；（4）`hints` 与 `urls` 等长（可为空串）；（5）`zoneNames` 顺序与 `zonesInfo` 网格遍历顺序一致（错位/追加区需在 `calculateMapsZonesRelative` 加特判并在 `zoneNames` 末尾补名）；（6）`initMapSelector()` 的 `icons` 加一个 emoji；（7）**补 `chestSize`/`chestUrl`**（wiki 搜"<图名> 俯视图 箱子点位"，不归林是"不归林-俯视图-箱子"） |
| **新增/调整密码机组** | 在该图 `ciphersCoords` 加一组 7 下标 + `urls` 加一张同尺寸地图图 + `hints` 加一条（空串则自动隐藏） |
| **换宝箱图 URL** | 只改 `chestUrl`；**尺寸必须同步改 `chestSize`**（容器高度按它算，写错会变形）；无需坐标数据（原图替换模式） |
| **扩充宝箱说明** | 编辑 `#chestPlaceholder` 内的文案（当前"宝箱说明待补充"）；若要升级为点位列表+画布标注，需新建 `CHEST_DATA`（`point` 以宝箱图 `chestSize` 为基准）并在 `drawCanvas` 宝箱分支里绘制——注意与密码机图坐标系不同 |
| **改提示文案** | 直接改 `fixedHint`（全组恒定）或 `hints[i]`（第 i 组专用），纯文本 |
| **调封禁判定** | 只动 `calculateLockCipher()`（欧氏像素距离，最大者封禁）；面板展示在 `updateDistancePanel()` |
| **换密码机图片** | 可以替换 URL，但新图**必须与原图同尺寸**（`size` 与像素坐标全部依赖原图坐标），否则封禁机、区域框全错 |
| **改样式** | 全部在 `<style>`：主题改 `:root` 变量即可全局换色；断点 938/804/686/429px；`.chest-btn` 激活态为琥珀金；注意 `.cipher-row` 是 grid 四列模板 |
| **改画布观感** | `drawCanvas()` (2027)：依次绘制 地图→区域边框→求生者→监管者→(仅密码机模式)封禁机(琥珀虚线)→(仅密码机模式)选中机(绿色虚线) |

## 8. Agent 注意事项（务必记住）

1. **单文件工程**：所有改动都在 `第五人格区域选择模拟器_20260809.html`；不要新建构建体系、不要拆文件（除非用户明确要求）。
2. **像素坐标与原图绑定**：`CIPHER_DATA[].point`、`zonesInfo`、`MAPS_DATA[].size` 是密码机图坐标系；`chestSize` 是宝箱图坐标系。**两套坐标系不通用**，改任何一个都要同步核对。
3. **`ciphersCoords` 是索引不是坐标**：值是 `CIPHER_DATA[mapId]` 的下标；换算在 `calculateCiphersCoordsRelative()`。
4. **id 特判三处**：红教堂错位（id 2）、永眠镇竖图旋转 + 第 10 区（id 6）；宝箱模式**绕过永眠镇旋转特判**（wiki 宝箱图原生竖图）。写通用逻辑时避开或显式处理 `chestMode`。
5. **宝箱模式是设计决策**：只整图替换、不标注不列表、面板占位；区域选择保留。若用户要升级（编号标注/距离分析/联动），属于新需求，重新走设计确认。
6. **中文语境**：UI 文案、注释、数据（`name/feature/hint/zoneNames`）全部中文；保持中文输出与命名习惯。
7. **无测试**：验证=浏览器手动点一遍（§3 清单）+ `node --check` 语法 + 数据完整性脚本。
8. **封禁机=最远机**是产品核心规则，改动前先与用户确认。
9. 页面首次加载即 `selectMap(0)`（军工厂第 1 组密码机），调试入口在文件末尾两行 `initMapSelector(); selectMap(0)`。

## 9. 函数速查（`<script>` 逻辑层）

| 函数 | 行号(约) | 职责 |
|---|---|---|
| `initParticles` | 1274 | 生成 30 个背景粒子 div |
| `calculateMapsZonesRelative` | 1679 | zonesInfo→相对区域网格（含特判） |
| `calculateCiphersCoordsRelative` | 1724 | CIPHER_DATA→相对密码机坐标 |
| `calculateZoneCenters` | 1746 | 区域中心点 |
| `calculateLockCipher` | 1750 | 求最远机=封禁机 |
| `setMapContainerHeight` | 1829 | 容器高度（chestMode 用 chestSize；id 6 仅密码机模式特判） |
| `loadSurvivorImg` / `loadHunterImg` | 1875 / 1895 | 图标加载+重试 |
| `showToast` | 1918 | 顶部 toast（约 1.34s 消失） |
| `initMapSelector` / `selectMap` | 1926 / 1944 | 地图卡片与切换（模式保持） |
| `loadMapImage` | 1974 | 地图图预加载+失败重试（模式感知） |
| `initCanvas` / `drawCanvas` | 2009 / 2027 | 画布初始化 / 全量重绘（宝箱模式跳过密码机相关层） |
| `toggleSide` | 2190 | 阵营切换 |
| `toggleChestMode` | 2202 | 宝箱/密码机模式切换 |
| `prevCipher` / `nextCipher` / `checkLock` | 2211 / 2222 / 2251 | 密码机组切换与封禁重算 |
| `handleMapClick` / `changeStates` | 2273 / 2312 | 点选区域与互斥规则 |
| `updateSelectionInfo` | 2361 | 顶部计数 |
| `updateDistancePanel` | 2369 | 右侧分析面板总重算（宝箱模式显示占位） |
| `resetSelection` | 2478 | 重置 |
| `getImageUrl` | 2499 | 按模式取图（chestUrl 或密码机组 URL） |
