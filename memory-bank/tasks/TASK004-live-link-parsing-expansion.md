# [TASK004] - 直播链接解析扩展（B站优先，多平台调研）

**Status:** Pending
**Added:** 2026-03-12
**Updated:** 2026-03-12

## Original Request
想捕获直播链接并解析。优先实现 B 站，同时评估其它支持平台是否也要加入直播解析。

## Current State
- 项目当前仅 B 站有直播解析入口：`live.bilibili.com/<room_id>`
- 入口位于 `parsers/bilibili/__init__.py::_parse_live()` 和 `parse_live()`
- 数据来源是 `bilibili_api.live.LiveRoom.get_room_info()`
- 数据结构位于 `parsers/bilibili/live.py`
- 已有网络集成测试 `tests/parsers/test_bilibili.py::test_live`
- `ParseResult` 当前没有“内容分类/结果类型”字段，只有 `platform`、`contents`、`extra` 等通用字段
- 渲染分发当前只按平台选择：`get_renderer(result.platform.name)`
- 通用卡片渲染器只按封面/图片/图文/正文等通用元素绘制，没有直播专用布局或视觉层级
- 因此 B 站直播虽然有解析入口，但仍被当成普通平台卡片处理，设计上没有被识别为独立内容类别
- 当前 B 站直播会把封面和关键帧塞进 `contents`，图片渲染完成后还会继续发送这些媒体，导致“卡片 + 附件图”重复表达
- 其余平台（抖音、快手、AcFun、YouTube、TikTok）暂无直播专用 handler

## Research Summary
- **Bilibili**
  - 风险最低，仓库里已有基础能力
  - `bilibili_api.live.LiveRoom` 还提供 `get_room_play_info()`、`get_room_play_url()`、`get_room_play_info_v2()`，后续可补充真实房间号、开播状态、清晰度和直播流信息
  - 当前实现只调用 `get_room_info()`，输出信息较轻，返回链接是 `blackboard/live/live-activity-player`，不是 canonical `live.bilibili.com/<room_id>`
- **Douyin**
  - 公开直播域名可见为 `live.douyin.com/<room_id>`
  - 现有短链 `v.douyin.com` / `jx.douyin.com` 已会先重定向，理论上可以复用短链入口，再为直播页新增专用 handler
  - 主要风险是网页数据结构变化和风控，接入风险高于 B 站
- **Kuaishou**
  - 公开直播域名可见为 `live.kuaishou.com/u/<user_or_room>`
  - 当前快手 parser 依赖 `window.INIT_STATE` 解析短视频/图集，直播页大概率不是同一套数据结构
  - 需要新增更具体的 live handler，避免被现有 `kuaishou.com/...` 泛匹配吞掉
- **AcFun**
  - 公开直播域名为 `live.acfun.cn/live/<id>`
  - 当前仅支持 `www.acfun.cn/v/ac...`
  - 直播页单独域名，适合新增独立 handler，复杂度中等
- **YouTube**
  - 公开 live 入口至少包括 `youtube.com/live/<video_id>` 和 `youtube.com/@<handle>/live`
  - 当前 regex 仅匹配 `watch|shorts`，直播链接不会命中
  - 现有 `YtdlpDownloader.VideoInfo` 强依赖 `duration: int` 和 `timestamp: int`；对 ongoing live 需要额外处理 `is_live/live_status`，不应直接复用当前下载逻辑
- **TikTok**
  - 本地 `yt-dlp` 已确认带 `TikTokLiveIE`，支持 `tiktok.com/@user/live` 与 `m.tiktok.com/share/live/<id>`
  - 可复用现有 `YTDLP_DOWNLOADER` 路径，但优先级仍低于 B 站、抖音、快手、AcFun、YouTube

## Thought Process
- 这不只是“链接捕获 + 元数据解析 + 媒体策略”的组合需求，还涉及解析结果分类和渲染架构
- 对直播场景，最重要的是先明确是否只做“房间预览解析”，还是还要支持直播流下载/转发
- 现有下载器和媒体发送链路更适合静态视频；对直播源直接下载会带来无限时长、文件体积、转码时间和平台风控问题
- 因此第一阶段应以“识别直播链接并返回标题、主播、封面、状态、分区等元信息”为目标，默认不自动下载直播流
- 当前真正的架构缺口不是“B 站没有直播 handler”，而是“直播没有被建模成独立分类”
- 如果不新增分类，后续其它平台直播仍会继续挤进通用视频/动态卡片模型，渲染和字段表达都会越来越别扭
- 更合理的第一步是给 `ParseResult` 增加独立的结果分类字段（例如 `category` / `result_type`），先引入 `live` 分类，再让渲染层按分类走专用逻辑
- 新增 live handler 时，要考虑 `BaseParser.search_url()` 的关键词优先匹配机制；应使用更具体的关键词（如 `live.douyin`、`live.kuaishou`），避免被现有泛匹配规则抢先命中
- 渲染分发也需要从“仅按平台”调整为“平台 + 分类”或“统一渲染器内按分类分支”，否则无法为直播新增专用渲染方法

## Proposed Architecture
### 1. 结果模型
- 在 `src/nonebot_plugin_parser/parsers/data.py` 中新增 `ResultCategory` 枚举，第一阶段至少包含：
  - `post`：现有默认内容
  - `live`：直播房间/直播回放/预约直播
- 在 `ParseResult` 上新增字段：
  - `category: ResultCategory = ResultCategory.POST`
  - `live: LiveMeta | None = None`
- 在 `ParseResultKwargs` 和 `BaseParser.result()` 的传参链路中补上 `category` 与 `live`
- 新增 `LiveStatus` 枚举，统一不同平台状态：
  - `live`：直播中
  - `upcoming`：预约中 / 即将开播
  - `offline`：未开播 / 已下播
  - `replay`：直播回放 / 轮播
  - `unknown`：无法确定
- 新增 `LiveMeta` 数据结构，不再把直播素材和状态继续塞进 `extra`：
  - `room_id: str`
  - `canonical_url: str`
  - `status: LiveStatus`
  - `hero_path_task: Path | Task[Path] | None`
  - `keyframe_path_task: Path | Task[Path] | None`
  - `owner_id: str | None`
  - `viewer_count: int | None`
  - `parent_area: str | None`
  - `area: str | None`
  - `tags: list[str]`
  - `notice: str | None`
  - `extra_lines: list[str]`
- 直播分类下，`contents` 默认应为空或仅保留真正需要额外发送的媒体，封面/关键帧转入 `LiveMeta` 供渲染使用，避免渲染后再次当附件发送

### 2. 渲染分发
- 将 `get_renderer()` 从 `get_renderer(platform: str)` 调整为 `get_renderer(result: ParseResult)`，把“平台 + 分类”的分发逻辑封装进去
- `matchers/__init__.py::parser_handler()` 改为传入完整 `result`
- 统一约定：先按 `category` 决定渲染路径，再按 `platform` 做样式差异
- 第一阶段的渲染策略：
  - `DefaultRenderer`：新增 `render_live_message()`，输出简洁文本版直播摘要
  - `CommonRenderer`：新增 `_create_live_card_image()` / `_render_live_card()`
  - `HtmlRenderer`：新增 `_render_live_image()`，使用单独模板 `live.html.jinja` 或在 `card.html.jinja` 中引入 `live` 专用 block
- `common` 和 `htmlrender` 的 live 卡片应具备统一信息层级：
  - 顶部：平台、主播、开播状态、时间
  - 主视觉：直播封面或关键帧
  - 核心信息：标题、分区、标签、房间号/频道标识
  - 辅助信息：人气、公告、预约/回放状态

### 3. 渲染素材约束
- live 分类默认不自动下载直播流
- live 分类默认不把封面/关键帧作为普通 `ImageContent` 附加发送
- 渲染器优先使用：
  - `hero_path_task` 作为主图
  - `keyframe_path_task` 作为可选补充图
- 如果平台没有稳定主图，允许渲染无主图的纯信息卡片

## Platform-specific Implementation
### Bilibili
- 目标文件：
  - `src/nonebot_plugin_parser/parsers/bilibili/__init__.py`
  - `src/nonebot_plugin_parser/parsers/bilibili/live.py`
- 接入方式：
  - 保留现有 `@handle("live.bili", r"live\.bilibili\.com/(?P<room_id>\d+)")`
  - 兼容带 query 的链接
  - `b23.tv` / `bili2233.cn` 短链继续通过 `parse_with_redirect()` 兜底
- 数据来源：
  - `LiveRoom.get_room_info()`：标题、封面、关键帧、分区、标签、主播基础信息
  - `LiveRoom.get_room_play_info()`：真实房间号、开播状态、uid 等补充字段
  - 暂不以 `get_room_play_url()` / `get_room_play_info_v2()` 作为首版依赖，只在需要补充状态或可用清晰度时再引入
- 结果构造：
  - `ParseResult.category = live`
  - `ParseResult.live = LiveMeta(...)`
  - `ParseResult.url` 统一改为 `https://live.bilibili.com/<room_id>`
  - `ParseResult.contents = []`
  - `title` 保留直播标题，`text` 只保留简短摘要，不再混入过多渲染字段
- 状态映射：
  - 按 B 站直播状态码映射到 `LiveStatus`
  - 首版至少区分 `live / offline / replay / unknown`
- 渲染重点：
  - 强调“直播中/未开播/轮播”状态
  - 展示分区、标签、房间号、主播
  - 主图优先关键帧，其次封面

### Douyin
- 目标文件：
  - `src/nonebot_plugin_parser/parsers/douyin/__init__.py`
  - `src/nonebot_plugin_parser/parsers/douyin/live.py`
- URL 入口：
  - 新增 `@handle("live.douyin", r"live\.douyin\.com/(?P<room_id>\d+)")`
  - 现有 `v.douyin.com` / `jx.douyin.com` 短链保持先重定向，再由 live handler 接管
- 实现思路：
  - 先请求直播页 HTML
  - 优先解析页面内嵌 JSON 状态树，提取标题、主播、封面、状态、房间号
  - 如果内嵌 JSON 提取失败，则 fallback 到 `og:title` / `og:image` / 页面标题做弱化解析
  - 为适配频繁变更的页面结构，解析逻辑单独放在 `parsers/douyin/live.py`
- 结果构造：
  - 返回 `category = live`
  - `contents = []`
  - `live.extra_lines` 保留回退解析得到的辅助信息
- 风险控制：
  - 抖音网页结构变更频繁，首版要接受“只拿到基础卡片信息”的降级结果
  - 若房间强依赖登录态，则将状态标记为 `unknown`，不因缺少结构化字段直接失败

### Kuaishou
- 目标文件：
  - `src/nonebot_plugin_parser/parsers/kuaishou/__init__.py`
  - `src/nonebot_plugin_parser/parsers/kuaishou/live.py`
- URL 入口：
  - 新增 `@handle("live.kuaishou", r"live\.kuaishou\.com/(?:u/)?(?P<room_token>[A-Za-z0-9_\\-]+)")`
  - 与现有 `www.kuaishou.com/...` 视频 handler 分离，避免直播页走错解析器
- 实现思路：
  - 请求直播页 HTML
  - 优先抽取初始化状态 JSON
  - 次级 fallback 为 `meta` 标签和标题信息
  - `room_token` 可能不是纯数字，内部需要先解析真实房间标识
- 结果构造：
  - 返回 `category = live`
  - 统一填充主播、标题、封面、状态、房间标识
- 风险控制：
  - 快手直播页和短视频页不是同一结构，解析逻辑必须拆分，不要复用现有 `states.decoder`

### AcFun
- 目标文件：
  - `src/nonebot_plugin_parser/parsers/acfun/__init__.py`
  - `src/nonebot_plugin_parser/parsers/acfun/live.py`
- URL 入口：
  - 新增 `@handle("live.acfun", r"live\.acfun\.cn/live/(?P<room_id>\d+)")`
- 实现思路：
  - 使用直播域名页面或其页面内嵌数据
  - 若存在稳定 JSON 入口，优先走 JSON；否则走 HTML 提取
  - 现有 `AcfunParser` 已有 `referer` 习惯，可按直播域单独设置请求头
- 结果构造：
  - 与其它平台保持统一 `LiveMeta`
- 风险控制：
  - 首版不做直播流下载，也不做弹幕/礼物等高频信息抓取

### YouTube
- 目标文件：
  - `src/nonebot_plugin_parser/parsers/youtube/__init__.py`
  - `src/nonebot_plugin_parser/download/ytdlp.py`
- URL 入口：
  - 扩展现有 regex，支持：
    - `youtube.com/live/<video_id>`
    - `youtube.com/@<handle>/live`
    - `youtube.com/channel/<id>/live`
    - `youtube.com/user/<name>/live`
    - `youtube.com/c/<name>/live`
- 实现思路：
  - 不直接抓网页，优先复用 `yt-dlp`
  - `yt-dlp` 本地 extractor 已覆盖 `/live` 视频页和频道 live tab
  - 扩展 `VideoInfo`，允许以下字段：
    - `duration: int | None`
    - `timestamp: int | None`
    - `live_status: str | None`
    - `is_live: bool | None`
    - `was_live: bool | None`
    - `concurrent_view_count: int | None`
  - 如果 `live_status` 属于 `is_live / is_upcoming / was_live / post_live`，或 URL 显式是 `/live` 入口，则构造 `category = live`
  - 仅当 `live_status == not_live` 且 URL 不是 live 入口时，继续走现有视频逻辑
- 结果构造：
  - 首版只返回直播卡片，不自动 `download_video()`
  - `@handle/live` 这类频道 live 页可直接交给 `yt-dlp` 解析，不需要手写二次跳转规则
- 风险控制：
  - ongoing live 可能没有固定 `duration`，必须先放宽 `VideoInfo` 类型定义，否则现有 `convert()` 会失败

### TikTok
- 目标文件：
  - `src/nonebot_plugin_parser/parsers/tiktok.py`
  - `src/nonebot_plugin_parser/download/ytdlp.py`
- URL 入口：
  - 新增：
    - `tiktok.com/@<user>/live`
    - `m.tiktok.com/share/live/<id>`
- 实现思路：
  - 优先复用 `yt-dlp`
  - 本地 `yt-dlp` 已带 `TikTokLiveIE`，支持上述 live URL 形态
  - 与 YouTube 类似，扩展 `VideoInfo` 或新增 `LiveInfo` 转换模型，读取 `live_status`、`concurrent_view_count`、封面和主播信息
- 接入策略：
  - 不列入第一批实现，只作为第二梯队候选
  - 前提是当前项目已经启用 `YTDLP_DOWNLOADER`

### 暂不首批接入的平台
- 微博 / 小红书 / Twitter(X) 当前虽然是已支持平台，但本轮不纳入直播首批范围
- 原因：
  - 当前仓库里没有直播入口积累
  - 公开链接形态与稳定解析路径还不清晰
  - 优先级低于已经具备明显直播 URL 入口的平台

## Renderer Implementation Detail
### CommonRenderer
- 新增 `render_live_image()` 或在 `render_image()` 内按 `result.category` 分派
- live 卡片布局建议：
  - 第一屏是主视觉图
  - 右上角是状态徽标
  - 标题单独一层，不混在 header
  - 主播、平台、时间作为次级信息
  - 分区 / 标签 / 房间号 / 人气使用信息块展示
- 直播卡片不沿用视频按钮覆盖层，避免误导用户认为消息内附带可下载视频

### HtmlRenderer
- `CardData` 新增：
  - `category`
  - `live`
  - `hero_path`
  - `status_label`
  - `status_class`
- 推荐新增独立模板 `live.html.jinja`，不要把大量 live 条件判断塞回通用 `card.html.jinja`
- 如果短期内不拆模板，则至少在 `card.html.jinja` 中新增 `data-category="live"` 和 live 专用 block

### DefaultRenderer
- 对 live 分类输出：
  - 平台 + 主播 + 状态 + 标题
  - 分区 / 标签 / 房间号
  - 链接
- 不自动拼接封面图，保证终端环境下结果简洁可读

## Test Plan
- 解析测试：
  - `tests/parsers/test_bilibili.py` 扩充 live 断言：
    - `result.category == ResultCategory.LIVE`
    - `result.live is not None`
    - `result.url` 为 canonical live URL
    - `result.contents` 不再包含仅用于渲染的封面/关键帧
- 渲染测试：
  - 新增 `tests/renders/test_live_render.py`
  - 使用构造出的 `ParseResult(category=live)` 做纯本地渲染测试，避免网络波动影响渲染验证
  - 同时覆盖 `CommonRenderer` 和 `HtmlRenderer`
- 集成渲染测试：
  - 在 `tests/renders/test_combined_render.py` 增加 B 站 live case
  - 后续每新增一个平台 live parser，再增补对应 case
- 解析器单元测试：
  - 对抖音 / 快手 / AcFun 这类易变 HTML 页面，优先补“冻结 HTML 片段 -> 解码器输出”的单元测试
  - 避免所有验证都依赖线上页面
- 测试数据：
  - `tests/others/test_urls.md` 新增 `live` 分组
  - 若需要冻结样本，建议新增 `tests/fixtures/live/`

## Delivery Phases
### Phase 1
- 新增 live 分类、状态模型和渲染分发
- 完成 Bilibili live 分类落地
- 完成 live 专用渲染器路径

### Phase 2
- 接入 Douyin / Kuaishou / AcFun
- 为这些平台补充冻结样本和最小网络测试

### Phase 3
- 接入 YouTube / TikTok（复用 `yt-dlp`）
- 评估是否支持直播回放和预约直播的细分展示

### Phase 4
- 评估是否需要用户显式触发“获取直播流地址/下载回放”
- 若要做，作为单独任务，不与 live 卡片首版耦合

## Implementation Plan
- [ ] 明确第一阶段直播解析的输出边界：仅元数据预览，暂不下载直播流
- [x] 设计并落地“直播是独立分类”的结果模型：
  - [x] 新增 `ParseResult` 的分类字段（如 `category` / `result_type`）
  - [x] 定义至少包含 `live` 的结果分类枚举
  - [x] 评估现有视频/图文/动态是否需要逐步迁移到同一分类体系
- [x] 改造渲染分发和渲染接口：
  - [x] 评估 `get_renderer()` 是否需要从按平台改为按“平台 + 分类”选择
  - [x] 或保留现有 renderer 选择方式，但在 renderer 内新增 `live` 专用渲染方法
  - [x] 明确 `default` / `common` / `htmlrender` 三种渲染路径对 live 的兼容策略
- [ ] 以 B 站直播作为首个 `live` 分类落地：
  - [ ] `parse_live()` 返回带 `live` 分类的结果
  - [ ] canonical URL 改为 `https://live.bilibili.com/<room_id>`
  - [ ] 补充开播状态、真实房间号、分区/标签、人气或关键帧等结构化字段
  - [ ] 新增直播专用渲染方法，而不是沿用普通动态/视频卡片
  - [ ] 补充更多直播链接形态测试（标准链接、短链重定向、带 query 的链接）
- [ ] 评估并按优先级接入其它平台：
  - [ ] `live.douyin.com/<room_id>`
  - [ ] `live.kuaishou.com/u/...`
  - [ ] `live.acfun.cn/live/<id>`
  - [ ] `youtube.com/live/<id>` / `youtube.com/@handle/live`
- [ ] 为各平台补充独立的 live handler，并统一归入 `live` 分类
- [ ] 增加测试样例和 `tests/others/test_urls.md` 的直播链接集合
- [ ] 评估是否需要第二阶段再支持直播流地址提取或按需下载

## Progress Tracking

**Overall Status:** Design Complete - 35%

### Subtasks
| ID | Description | Status | Updated | Notes |
|----|-------------|--------|---------|-------|
| 4.1 | 梳理现有 B 站直播解析实现和缺口 | Completed | 2026-03-12 | 已确认 parser、数据结构和测试入口 |
| 4.2 | 调研其它平台直播链接形态与可行性 | Completed | 2026-03-12 | 已完成第一轮候选平台筛选 |
| 4.3 | 设计直播独立分类与结果模型 | Completed | 2026-03-12 | 已补充 `ResultCategory` / `LiveMeta` 方案 |
| 4.4 | 设计渲染分发与 live 专用渲染路径 | Completed | 2026-03-12 | 已明确 Common/HTML/Default 三路实现 |
| 4.5 | 实现 B 站 live 分类落地与补充测试 | Not Started | 2026-03-12 | 先作为第一批落地平台 |
| 4.6 | 按优先级扩展其它平台直播解析 | Not Started | 2026-03-12 | Douyin/Kuaishou/AcFun 优先 |
| 4.7 | 接入 yt-dlp 路径的直播平台 | Not Started | 2026-03-12 | YouTube/TikTok 第二梯队 |

## Progress Log
### 2026-03-12
- 记录“直播链接解析扩展”需求，明确 B 站优先
- 确认项目当前已经支持 B 站直播解析，但仅覆盖基础元数据
- 识别到 B 站之外优先级较高的候选平台为抖音、快手、AcFun、YouTube
- 识别到 `yt-dlp` 路径对 ongoing live 需要额外字段处理，第一阶段不宜直接复用现有下载逻辑
- 补充确认当前架构缺口在于 `ParseResult` 没有独立分类字段，渲染层也只能按平台分发
- 明确后续目标不是“把直播继续塞进动态/视频卡片”，而是新增 `live` 分类并配套专用渲染方法
- 补充了可执行实现方案：结果模型、渲染分发、各平台入口与解析路径、测试和分阶段交付
