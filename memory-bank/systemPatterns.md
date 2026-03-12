# System Patterns

## 架构概览

```
Message with URL
       ↓
on_keyword_regex matcher (全局关键词+正则匹配)
       ↓
parser_handler()
  ├─ 缓存查找 [_RESULT_CACHE]
  ├─ KEYWORD_PARSER_MAP 获取对应 Parser
  ├─ Parser.parse(keyword, match) → ParseResult
  └─ Renderer.render_messages(ParseResult) → UniMessage[]
       ↓
发送消息到群聊
```

## 核心设计模式

### 1. 自动注册模式 (Auto-Registration)
- `BaseParser.__init_subclass__()` 自动收集所有子类
- `@handle(keyword, pattern)` 装饰器注册 URL 模式和处理方法
- 启动时根据配置过滤已启用的解析器，构建全局关键词→解析器映射

### 2. 工厂/策略模式 (Renderers)
- 根据 `parser_render_type` 配置选择渲染器
- 渲染器类型: `default`(纯文本), `common`(PIL渲染), `htmlrender`(Playwright), `htmlkit`
- 每个平台可有自定义渲染器（如微博）

### 3. 异步任务模式 (Download)
- 下载操作返回 `Task[Path]`，延迟执行
- `@auto_task` 装饰器包装为 asyncio task
- `MediaContent.get_path()` 在需要时 await 下载结果

### 4. 统一消息抽象
- `UniMessage` + `UniHelper` 抽象底层适配器差异
- 支持 OneBot v11、QQ 官方等多种协议

## 关键模块关系

```
parsers/
├── base.py          # BaseParser 基类, @handle 装饰器
├── data.py          # ParseResult, MediaContent 数据结构
├── cookie.py        # Cookie 管理
├── bilibili/        # B 站解析 (视频、动态、专栏、直播、收藏夹)
├── douyin/          # 抖音解析 (视频、图集)
├── weibo/           # 微博解析
├── xiaohongshu/     # 小红书解析
├── kuaishou/        # 快手解析
├── twitter.py       # 推特解析
├── tiktok.py        # TikTok 解析 (需 yt-dlp)
└── nga.py           # NGA 解析

renders/
├── base.py          # BaseRenderer, ImageRenderer 基类
├── common.py        # CommonRenderer (PIL 渲染)
├── default.py       # DefaultRenderer (纯文本)
├── htmlrender.py    # HtmlRenderer (Playwright)
├── weibo.py         # 微博专用渲染
├── resources/       # 静态资源
└── templates/       # HTML 模板

download/
├── __init__.py      # StreamDownloader
├── task.py          # 下载任务管理
└── ytdlp.py         # yt-dlp 下载器

matchers/
├── __init__.py      # Matcher 注册、命令处理
├── filter.py        # 消息过滤
└── rule.py          # 匹配规则
```

## 缓存策略
- `_RESULT_CACHE`: LimitedSizeDict(max_size=50)，解析结果缓存
- 下载文件缓存在 `cache_dir`，每天凌晨1点清理
- 渲染图片通过 `cache_or_render_image()` 缓存
