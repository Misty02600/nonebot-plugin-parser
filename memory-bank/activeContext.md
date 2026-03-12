# Active Context

## 当前工作重点
- 日常使用中发现的 Bug 修复和记录
- 解析器兼容性问题排查

## 最近发现的问题
1. **B 站专栏链接解析失败** - `bilibili.com/read/cvXXX` 触发 `KeyError: 'readInfo'`，bilibili_api 的 `article.turn_to_opus()` 在某些文章返回数据缺少 `readInfo` 字段
2. **抖音精选链接无法解析** - `douyin.com/jingxuan?modal_id=XXX` 格式未被正则匹配，现有模式仅匹配 `jingxuan.douyin.com/m/video/XXX`

## 近期决策
- 暂不修改代码，先记录为任务跟踪

## 下一步
- 分析抖音精选链接的具体格式和解析方式
- 研究 bilibili_api 是否有替代的专栏解析方法
