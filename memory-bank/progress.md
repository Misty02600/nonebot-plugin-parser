# Progress

## 项目状态
v2.4.3 已发布，日常维护中

## 已完成的功能
- 10 个平台的链接解析（B站、抖音、微博、小红书、快手、acfun、YouTube、TikTok、Twitter、NGA）
- PIL 通用媒体卡片渲染 (CommonRenderer)
- HTMLRender 渲染（实验性）
- 群聊开关管理
- B 站/YouTube 音频下载命令
- B 站 QR 码登录
- 自定义字体支持
- Emoji 渲染（网络 CDN / 本地 SVG）
- 自动缓存清理

## 待修复的问题
- [ ] B 站专栏 `cv` 链接解析失败 (KeyError: 'readInfo') → TASK001
- [ ] 抖音精选页链接无法解析 (`douyin.com/jingxuan?modal_id=XXX`) → TASK002
- [ ] 解析后无法让用户按需选择下载视频/图片资源 → TASK003

## 已知限制
- htmlrender 模板效果不佳，不建议使用
- htmlkit 暂不可用
- 部分快手链接依赖有状态的浏览器模拟
