# Tech Context

## 技术栈
- **框架**: NoneBot2 >= 2.4.3
- **Python**: >= 3.10 (支持到 3.14)
- **包管理**: uv
- **代码风格**: ruff
- **CI**: pre-commit

## 核心依赖
| 依赖 | 用途 |
|------|------|
| nonebot2 | Bot 框架 |
| nonebot-plugin-alconna | 跨平台消息处理 |
| nonebot-plugin-localstore | 本地存储管理 |
| nonebot-plugin-uninfo | 统一用户信息 |
| nonebot-plugin-apscheduler | 定时任务 |
| bilibili-api-python >= 17.4.1 | B 站 API |
| httpx | HTTP 客户端 |
| curl_cffi | 模拟浏览器请求 |
| pillow | 图片渲染 |
| apilmoji | Emoji 渲染 |
| msgspec | 高性能序列化 |
| beautifulsoup4 | HTML 解析 |
| aiofiles | 异步文件操作 |

## 可选依赖
| 依赖 | 用途 |
|------|------|
| yt-dlp | YouTube/TikTok 视频下载 |
| emosvg | 本地 Emoji SVG 渲染 |
| nonebot-plugin-htmlrender | Playwright HTML 渲染 |

## 开发环境
- **运行**: `nb run` 或 `python bot.py`（需先 `uv sync`）
- **测试**: `pytest`
- **.env 配置**: LOCALSTORE_USE_CWD=true 时，config/data/cache 在项目根目录
- **目录结构**:
  - `config/nonebot_plugin_parser/` - Cookie 文件、字体文件
  - `data/nonebot_plugin_parser/` - 持久数据（禁用群列表）
  - `cache/nonebot_plugin_parser/` - 缓存文件（下载、渲染）

## 技术约束
- B 站 API 需要配置 Cookie (SESSDATA) 才能使用 AI 总结
- YouTube/TikTok 需要 yt-dlp + JavaScript Runtime (Deno)
- 视频最大时长默认 480 秒，文件最大 90MB
- 快手解析依赖有状态的浏览器模拟
