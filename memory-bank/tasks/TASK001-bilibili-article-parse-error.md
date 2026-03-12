# [TASK001] - B站专栏链接解析失败

**Status:** Pending
**Added:** 2026-03-12
**Updated:** 2026-03-12

## Original Request
B站专栏链接 `bilibili.com/read/cvXXX` 触发运行时错误 `KeyError: 'readInfo'`。

## 错误信息
```
File "bilibili_api/article.py", line 603, in get_all
    cache_pool.article2dynamic[self.__cvid] = self.__get_all_data["readInfo"][
KeyError: 'readInfo'
```

## Thought Process
- 错误发生在 `bilibili_api` 库内部 `Article.turn_to_opus()` → `Article.get_all()` 时
- API 返回的 JSON 数据中缺少 `readInfo` 字段
- 可能原因:
  1. 某些文章类型（如笔记型专栏）API 响应格式不同
  2. bilibili_api 库版本 (17.4.1) 未适配新的 API 变更
  3. 特定文章已被删除或限制访问
- 项目中已有自定义的 `ArticleInfo` 数据结构 (`parsers/bilibili/article.py`)，但当前未被使用

## Implementation Plan
- [ ] 确认触发问题的具体专栏链接
- [ ] 检查 bilibili_api 是否有更新版本修复此问题
- [ ] 方案 A: 在 `_parse_read` 中添加 try-except，降级处理
- [ ] 方案 B: 绕过 `turn_to_opus()`，直接使用 B站 API 获取专栏数据，用自定义 `ArticleInfo` 解析

## Progress Tracking

**Overall Status:** Not Started - 0%

### Subtasks
| ID | Description | Status | Updated | Notes |
|----|-------------|--------|---------|-------|
| 1.1 | 复现问题并确认触发条件 | Not Started | 2026-03-12 | |
| 1.2 | 检查 bilibili_api 上游是否有修复 | Not Started | 2026-03-12 | |
| 1.3 | 实现修复方案 | Not Started | 2026-03-12 | |
| 1.4 | 测试验证 | Not Started | 2026-03-12 | |

## Progress Log
### 2026-03-12
- 记录问题，分析错误栈
- 确认错误位于 bilibili_api 库内部，非插件代码直接问题
- 发现项目中已有未使用的 `ArticleInfo` 自定义结构体可作为备选方案
