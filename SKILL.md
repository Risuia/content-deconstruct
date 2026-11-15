---
name: content-deconstruct
description: 纯 AI 拆解 B站、抖音、小红书的创作内容。AI 自己找内容、自己读弹幕评论分章、自己下载封面读图、自己对齐时间轴做归因，直接输出可复用的创作配方与传播基因，不写爬虫不接授权。触发词：拆解视频、拆爆款、拆解B站、拆解抖音、拆解小红书、这条视频为什么火、爆款拆解、拆片、选题分析、竞品内容拆解、爆款复盘、内容拆解。
---

# 纯 AI 内容拆解工作流

## 核心原则

**AI 自己当分析师**，不是搭数据管线。不写爬虫、不接开放平台授权、不存向量库、不建 Schema 表。

拆解的本质 = 把一条内容还原成**带时间戳的证据包**（转写 / OCR / 弹幕 / 评论 / 元数据 / 图像）→ 在统一时间轴上做归因 → 输出结构化结论。

**没有时间信号 = 不是拆解，只是读后感。** 弹幕是唯一免费且带时间戳的情绪标注。

## 三平台能力边界（2026-10-29 实测）

| B站 | 抖音 | 小红书 |
|---|---|---|
| 实时榜单 ✅ popular + ranking + 热搜 | ✅ App 端热点榜（免签名） | ❌ 需 Key |
| 单条详情 ✅ 免登录 | ❌ 需 a_bogus 签名 | ❌ 需 Key |
| 弹幕（带时间戳）✅ 免登录 | ❌ 需签名 | ❌ 无此数据 |
| 评论 ✅ 免登录 | ❌ 需签名 | ❌ 需 Key |
| 字幕 ⚠️ 需登录态 | ❌ 需签名 | ⚠️ 正文可得（需 Key） |
| 官方分章 ✅ 免登录 | ❌ 无 | ❌ 无 |
| 热搜 / 话题词 ✅ search/square 30 条 | ✅ 热点榜 49 条 | ❌ 需 Key |
| 配图看图 ✅ 封面 + 首帧 + 分章图 | ✅ 热点配图 | ⚠️ 需 Key |
| 单博主完整视频记录 ✅ 零配置 | ❌ 需签名 + Cookie | ❌ 需登录态 |
| 单博主合集 / 系列 ✅ 零配置 | ❌ | ❌ |
| 单博主专栏 △ 列表可，正文需登录 | ❌ | ❌ |
| 单博主动态图文 ❌ 需登录态 | ❌ | ❌ |
| 单条深度拆解 ✅ 成立 | △ 需工具 | △ 需工具 |
| 选题预判 ✅ 成立 | ✅ 成立（更强） | ⚠️ 需 Key |

**分工原则**：B站 = 拆解引擎（答「为什么火」）＋作品全景库（答「他怎么做到」），抖音 = 选题雷达（答「什么值得扒」），小红书 = 需 Key 的第三极。**三者是互补而非对等关系。**

## 单博主作品全景（B站，零登录零 Key）

```bash
# 一键采集：资料 + 7 类形态计数 + 全量投稿视频 + 合集系列 + 专栏容器
python creator_profile.py {mid}
```

| 数据源 | 接口 | 产出 |
|---|---|---|
| 形态计数 | `x/space/navnum?mid=` | video / article / audio / album / pugv / opus / season_num 各自总量 |
| 基础资料 | `x/web-interface/card?mid=&photo=false` | 昵称 / 粉丝 / 等级 |
| 全部投稿视频 | `x/space/arc/list?mid=&pn=&ps=30` | `page.count` = 真实总数，含实时 stat 与 rights |
| 合集与系列 | `x/polymer/web-space/seasons_series_list?mid=&page_num=&page_size=20` | 每个合集 / 系列含成员视频明细 |
| 专栏容器 | `x/article/up/lists?mid=&pn=&ps=50&sort=publish_time` | 专栏名 + articles_count + read |

**关键点**：

- `x/space/arc/list` 是**旧版接口，免 WBI 签名免登录**，比 `x/space/wbi/arc/search` 好用（后者实测 HTTP 412）
- `seasons_series_list` 的 **page_size 上限 20**，传 50 报 -400
- `arc/list` 的 stat 是**实时值**（不同于热门榜快照）
- **差额本身是信息**：navnum 计数 − 实际取到数 = 充电专属 + 已删除 + 未公开
- 分析时用**效率指标**（点赞率 / 收藏率）而非绝对播放量——高效率作品常常不在播放量 Top

## 执行流程

### 第 0 步 · 请求头（必做，否则会被风控）

**B站接口必须带完整浏览器头**，否则返回 `{"code":-352}` 而不是数据：

```bash
UA="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"
curl -s -A "$UA" -H "Referer: https://www.bilibili.com/" "https://api.bilibili.com/..."
```

实测：同一接口在短时间内重复裸请求会触发 -352，**加 UA + Referer 后可正常返回**。抖音热点榜不需要头，但带上无妨。

### 第 1 步 · 定位候选

按**互动效率**排序，不按绝对播放量（绝对量会被大号碾压）。

```bash
# B站热门流（需 UA + Referer）
curl -s -A "$UA" -H "Referer: https://www.bilibili.com/" \
  "https://api.bilibili.com/x/web-interface/popular?ps=20&pn=1"

# B站全站排行榜 100 条（官方按质量排序，比热门流更适合找"强内容"；rid=0 全站，可选分区）
curl -s -A "$UA" -H "Referer: https://www.bilibili.com/" \
  "https://api.bilibili.com/x/web-interface/ranking/v2?rid=0&type=all"

# B站热搜词 30 条（独立信源，与抖音热点榜交叉价值高）
curl -s -A "$UA" -H "Referer: https://www.bilibili.com/" \
  "https://api.bilibili.com/x/web-interface/search/square?limit=30"

# B站搜索页（HTML，绕开 WBI 签名）
# https://search.bilibili.com/all?keyword={关键词}

# 抖音官方热点榜（免登录免签名，49 条实时词条）
curl -s "https://aweme.snssdk.com/aweme/v1/hot/search/list/?detail_list=1"

# 抖音指数（原巨量算数，选题预判）
# https://trendinsight.oceanengine.com
```

抖音热点条目字段：`word` / `position` / `hot_value` / `word_cover.url_list`（配图免鉴权可下载）/ `group_id` / `sentence_id` / `video_count` / `event_time` / `sentence_tag`；另有 `data.trending_list[]` 5 条与 `data.active_time` 更新时间。参数变体 `detail_list=2`、`board_type=2` 均返回正常数据（另一套榜单）。

排序指标：

- B站：投币率高 = 硬核认可，收藏率高 = 干货，评论率高 = 争议
- 抖音：`hot_value` 绝对值 + `video_count`（关联视频数 = 热点宽度）

**跨平台热点检测（已实测可自动化）**：三源交叉匹配——抖音热点榜 + B站热搜词 + B站排行榜标题，归一化后做包含匹配（去标点，保留中英文数字，长度 ≥ 4）。

实测命中：抖音热点「iPhone Duo 深度评测」↔ B站热搜「影视飓风 iPhone Duo 深度评测」↔ B站排行「折 1 万次，会坏吗？iPhone Duo 深度评测」（BV1xrnz6gEVD，194 万播放，30653 弹幕）——**同一条内容三源在榜**；另有「赛博朋克边缘行者 2 剧情评析」为话题级共振。

### 第 2 步 · 取证据（B站四路并行）

**oid 类型必须分清：弹幕用 cid，评论用 aid。填错静默返回空 / 404。**

```bash
# ① 元数据 + 标签 + 合集 + 荣誉（最全）
curl -s -A "$UA" -H "Referer: https://www.bilibili.com/" \
  "https://api.bilibili.com/x/web-interface/view/detail?bvid={BV}"

# ② 官方分章 view_points（from/to 秒级时间戳 + 每章封面图）
curl -s -A "$UA" -H "Referer: https://www.bilibili.com/" \
  "https://api.bilibili.com/x/player/wbi/v2?bvid={BV}&cid={cid}"

# ③ 弹幕（全量，带秒级时间戳）—— 拆解的命根子
curl -s -A "$UA" -H "Referer: https://www.bilibili.com/" \
  "https://api.bilibili.com/x/v1/dm/list.so?oid={cid}"

# ④ 评论（带点赞数、回复数；sort=2 按热度）
curl -s -A "$UA" -H "Referer: https://www.bilibili.com/" \
  "https://api.bilibili.com/x/v2/reply?type=1&oid={aid}&sort=2&pn=1&ps=20"
```

`ps` 上限 20，超了报 `-400 ps out of bounds`。

### 第 3 步 · 看画面

从 ① 取 `pic`（封面）与 `first_frame`（首帧），从 ② 取 `view_points[].imgUrl`（分章封面）：

```bash
curl -s -o cover.jpg "https://i0.hdslb.com/bfs/archive/{pic}.jpg"
curl -s -o chapter1.jpg "https://i0.hdslb.com/bfs/storyff/{imgUrl}.jpg"
```

然后用视觉模型读图。读出：构图（主体位置 / 留白）、花字（有无 / 大小 / 位置 / 颜色）、人物状态、色彩策略（主色调与情绪匹配度）、信息密度、封面与首帧的职能差异。

**分章封面图 = 免费关键帧分镜表**，比随机抽帧质量更高（官方精挑的）。

### 第 3.5 步 · 深度方案：下载视频 + 抽帧 + 读图

补上「字幕需登录」和「看不到画面动态」两个缺口。**零配置零 Key**，只多一个 pip 包。

```bash
python -m pip install yt-dlp imageio-ffmpeg   # 一次性

# ① 下载音视频分离流（30080=1080P 视频，30280=纯音频）
python -m yt_dlp -f "30080+30280" --no-playlist -o "dl.%(ext)s" "https://www.bilibili.com/video/{BV}"

# ② 抽帧（每 N 秒一帧）
FF=$(python -c "import imageio_ffmpeg;print(imageio_ffmpeg.get_ffmpeg_exe())")
"$FF" -i dl.mp4 -vf "fps=1/8,scale=480:-1" -q:v 3 frames/f_%03d.jpg -y

# ③ 逐张读图
```

实测：97 秒视频 → 12 帧，能读出画面主体、UP 主水印、底部字幕文字、构图与色调。

`--print-json` 另给 51 个字段：`chapters`（与 view_points 同源）、`tags`、`formats[]` 直链、`filesize_approx`、`upload_date`、`description`、`duration_string`、`aspect_ratio`、`dynamic_range`。

**音频转写**：拿到 `.m4a` 后用 FunASR 或 whisper 转写 → 补上口播原文。

### 第 3.6 步 · 字幕（需登录态）

```bash
# 免登录只有 danmaku 轨，拿不到 ai-zh
python -m yt_dlp --list-subs "URL" | grep -E "ai-zh|danmaku"

# 带浏览器 cookie（目标浏览器必须未运行）
python -m yt_dlp --cookies-from-browser chrome --write-subs --sub-lang ai-zh \
  --skip-download --no-simulate -o "bili_output" "URL"
```

三个坑：

- `--no-simulate` **必需**，否则只打印不落盘
- `--cookies-from-browser` **仅在目标浏览器未运行时可用**；否则用 Playwright 的 `context.cookies()` 取 HttpOnly cookie
- 验证标准：`--list-subs` 输出出现 `ai-zh` 行才算拿到登录态，只出现 `danmaku` 说明没有

### 第 3.7 步 · 抖音 / 小红书：浏览器扩展路线

需要抖音单条内容或小红书任何内容时，用扩展复用已登录会话（无需破解签名）：

- **MediaClaw MCP（社媒虾）**：明确支持 WorkBuddy，浏览器插件 + STDIO MCP，40+ 工具覆盖小红书 / 抖音 / TikTok，自带视频逐字稿（带时间戳）、配图文字提取、12 套研究方法（含单篇内容拆解）。Agent 拿不到你的 Cookie
- **OpenBiliClaw**：Chrome 插件任务链路，抖音 search / hot / feed 从首页 DOM 操作触发
- **xhs-mcp**：Playwright + 扫码登录，`get_note` 需从 search 结果取 `xsecToken`
- **douyin-mcp**：本地 V8（py-mini-racer）生成 a_bogus，只需 cookies.txt，覆盖搜索 / 详情 / 评论 / 推荐流

小红书关键：`note_url` 必须原样保留（含 `xsec_token` query 参数），**不能用 note_id 重新拼接链接**。

### 第 4 步 · 对齐时间轴

把弹幕、分章、评论按时间 / 权重对齐，得出：钩子第几秒生效、哪个瞬间弹幕爆发（= 高光 / 争议点）、观众何时开始质疑、结尾引导是否奏效。

**这一步是拆解与复述的分界线。**

### 第 5 步 · 归因输出

严格按下方格式输出。每条结论必须能指回具体证据（哪个接口的哪个字段、哪个时间戳）。

## 输出格式

```
■ 钩子（前 3 秒）
  画面：/ 口播：/ 屏幕文字：/ 音频：/ 生效时间点：/ 留人机制：

■ 结构骨架（表格：时间区间 | 章节/内容 | 画面策略 | 弹幕密度 | 观众情绪）

■ 情绪曲线（时间点 | 情绪 | 强度 | 触发原因）

■ 争议点（评论区高赞质疑 + 对应弹幕时间）

■ 传播基因（靠什么传播：梗 / 选题 / 形式 / 情绪）

■ 可复用配方
  公式：/ 必须保留：/ 必须避开：/ 适用时长：

■ 置信度（高 / 中 / 低）+ 缺失的证据说明
```

## 置信度规则（硬约束）

| 可用证据 | 置信度 | 能得出 / 不能得出 |
|---|---|---|
| 弹幕 + 评论 + 分章 + 图 | 高 | 完整归因，结论可指回时间戳 |
| 弹幕 + 评论 + 图（无分章） | 中 | 结构靠弹幕时序反推，须标注「结构为推断」 |
| 仅评论 + 图 | 中 | 可做争议点 / 需求挖掘 / 视觉策略。禁止输出「结构分析」（无时间信号） |
| 仅元数据 + 标题 | 低 | 只能做选题定位。禁止称为「拆解」 |

**三条硬规则**：

- 无时间信号时禁止输出「结构分析」
- 每个结论标注证据来源
- 拿不到就说拿不到，**禁止用「可能 / 通常 / 一般来说」填充空白**

## 常见失败模式

- **B站必须带 UA + Referer**：否则 `{"code":-352}`。这解释了为什么 WebFetch 能通（自带完整浏览器头）而 curl 裸请求会失败
- **oid 用错类型**：弹幕 cid / 评论 aid，填反静默返回空，最常见失败原因
- **-352 风控**：UP 主投稿列表、wbi 搜索接口触发，改用 popular + ranking + 搜索页 HTML
- **字幕是登录态接口**：`need_login_subtitle:true` 时别反复请求，用「分章 + 弹幕 + 封面图 + 评论」替代
- **用绝对播放量选内容**：必须用效率指标
- **把复述当拆解**：输出全是「这条视频讲了什么」就是失败
- **抖音的「假成功」**：`www.douyin.com/aweme/v1/web/hot/search/list/` 返回 HTTP 200 但 body 0 字节；App 端搜索 `status_code:0` 但 aweme_list 为 null。**200 不等于有数据，必须查 body 长度与字段是否 null**。抖音热点榜要用 **snssdk 域**而非 www 域
- **小红书的假 noteId**：`__INITIAL_STATE__` 里 22 个 24 位 hex ID 来自 `disableLikeNotes` 灰度名单，不是真实笔记。**拿到 ID 必须先验证能否打开**
- **小红书免登录不存在**：详情页 302、搜索页 URL 带签名、`feeds:[]` + `needsRefreshAfterLogin:false`。做小红书必须引入 API Key（Rnote / Just One API，一个 Token 可覆盖三平台）
- **curl 与 WebFetch 行为不同**：小红书 explore 页 curl 会 302 跳登录但 WebFetch 能读到标题列表；详情页两者都读不到
- **`ps` 参数上限 20**：`x/v2/reply` 传 `ps=25` 会报 `-400 ps out of bounds`
- **`weekly/list` 已下线**：返回 404，改用 ranking/v2
- **判断成功的标准**：HTTP 200 + body 长度 > 0 + 目标字段非 null + 业务 code 为 0，四者同时满足
- **热门榜 stat 是入榜时刻的快照，不是实时值**：popular 与 view/detail 口径一致（实测 1.00x），但同一视频一周内可从 217 万涨到 1061 万。**做效率排序用热门榜快照值没问题；做绝对量对比或趋势分析必须用 view/detail 取实时值**
- **小红书 note_url 不能重建**：`xsec_token` 等 query 参数必须原样保留，用 note_id 拼接的链接读不到内容
- **yt-dlp 文件名规则**：`-o "dl.%(ext)s"` 得到 `dl.mp4`。ffmpeg 输入路径要跟实际文件名对上

## 需要 Key 的补充路径

| 方案 | 覆盖 | 说明 |
|---|---|---|
| Just One API | B站 + 抖音 + 小红书 | 一个 Token 覆盖三平台，B站接口直接给弹幕与字幕 |
| Rnote API | 小红书 | `/api/v2/crawler` 公开内容 + `/api/v2/pgy` 蒲公英投放数据 |
| SandBase 类 | 抖音 + 小红书 + 微博 | 统一 `POST /v1/api/{platform}/app-v2/{endpoint}` |
| 抖音开放平台 | 抖音（仅授权账号） | 需企业资质，纯采集过不了审 |
| 小红书开放平台 | 小红书（偏电商） | 笔记发布无公开 API；阅读量属蒲公英数据面 |

**小红书数据边界**：阅读量不在公开消费端，须走蒲公英，且部分帖子指标可能不可用，需做空值兜底。

## 合规

- 采集优先级：官方开放平台 > 平台官方工具 > 第三方合规 API > 浏览器插件（低频人工触发）> 自建爬虫（仅公开页、严格限速、只读）> 协议逆向 / RPA（高风险）
- 不绕过登录态或风控（小红书签名头 / 设备指纹属风控体系，逆向它属于规避技术保护措施）
- 不采集未授权个人数据
- 拆解产出的是**结构与方法**，不是内容本身；仿写须换题材换表达换素材
- 采集频率遵守官方节奏（小红书官方 API 默认 QPS=1）
