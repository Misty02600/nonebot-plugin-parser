# Project Brief

## 项目名称
nonebot-plugin-parser

## 项目概述
NoneBot2 链接分享自动解析插件（Alconna 版），当用户在群聊中发送支持平台的链接时，自动解析并返回结构化的内容卡片。

## 核心需求
1. **多平台链接解析** - 支持 B站、抖音、快手、微博、小红书、YouTube、TikTok、Twitter、AcFun、NGA
2. **媒体内容处理** - 自动下载和转发视频、图集、音频
3. **渲染输出** - 将解析结果渲染为美观的图片卡片
4. **群聊管理** - 支持按群开启/关闭解析功能
5. **可扩展性** - 第三方可通过继承 BaseParser 添加新平台

## 技术栈
- **框架**: NoneBot2 + nonebot-plugin-alconna
- **语言**: Python 3.10+
- **包管理**: uv
- **代码风格**: ruff

## 项目版本
v2.4.3

## 项目仓库
https://github.com/fllesser/nonebot-plugin-parser
