# [TASK002] - 抖音精选页链接无法解析

**Status:** Pending
**Added:** 2026-03-12
**Updated:** 2026-03-12

## Original Request
链接 `https://www.douyin.com/jingxuan?modal_id=7601590620959952180` 无法解析。

## Thought Process
- 当前抖音解析器的 `@handle` 注册模式:
  - `v.douyin.com/xxx` → 短链
  - `jx.douyin.com/xxx` → 短链
  - `douyin.com/video/xxx` → 视频
  - `douyin.com/note/xxx` → 图文
  - `iesdouyin.com/share/xxx` → 分享链接
  - `m.douyin.com/share/xxx` → 移动端分享
  - `jingxuan.douyin.com/m/video/xxx` → 精选页视频
- 用户提供的链接格式为 `www.douyin.com/jingxuan?modal_id=XXX`，不匹配任何已有模式
- `modal_id` 参数中的 ID 是视频 ID，可以提取后用现有解析逻辑处理
- 需要新增一个 `@handle` 规则来匹配这种格式

## Implementation Plan
- [ ] 分析 `douyin.com/jingxuan?modal_id=XXX` 的实际内容类型（视频/图集）
- [ ] 新增 `@handle` 规则匹配 `douyin.com/jingxuan` + `modal_id` 参数
- [ ] 提取 `modal_id` 作为视频 ID，复用现有解析逻辑
- [ ] 测试验证

## Progress Tracking

**Overall Status:** Not Started - 0%

### Subtasks
| ID | Description | Status | Updated | Notes |
|----|-------------|--------|---------|-------|
| 2.1 | 分析精选页链接格式和内容类型 | Not Started | 2026-03-12 | |
| 2.2 | 新增 @handle 规则 | Not Started | 2026-03-12 | |
| 2.3 | 测试验证 | Not Started | 2026-03-12 | |

## Progress Log
### 2026-03-12
- 记录问题
- 分析现有抖音解析器的 URL 匹配模式
- 确认 `www.douyin.com/jingxuan?modal_id=XXX` 格式不在已有匹配规则中
- `modal_id` 值为数字 ID (7601590620959952180)，可作为视频 ID 提取
