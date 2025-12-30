# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

MoveCar 是一个基于 Cloudflare Workers 的 Serverless 挪车通知系统。核心特点：
- 单文件架构（movecar.js）包含所有后端逻辑和前端 HTML
- 使用 Bark 推送服务进行通知
- 通过 Cloudflare KV 存储临时状态数据
- 内置 IP 地理位置限制和访问频率限制

## 架构设计

### 单文件结构
整个应用在 `movecar.js` 中实现，包含：
1. **路由处理** (`handleRequest`): 处理所有 HTTP 请求
2. **API 端点**: `/api/notify`, `/api/get-location`, `/api/owner-confirm`, `/api/check-status`
3. **页面渲染**: `renderMainPage()` 和 `renderOwnerPage()` 返回完整的 HTML
4. **坐标转换**: WGS-84 转 GCJ-02（中国国测局坐标系）
5. **安全机制**: IP 地理位置检查和访问频率限制

### 数据流
```
请求者扫码 → 获取位置 → POST /api/notify → Bark 推送 → 车主收到通知
                                    ↓
                            KV 存储状态和位置
                                    ↓
车主点击 → /owner-confirm → 查看位置 → POST /api/owner-confirm → 更新状态
                                    ↓
请求者轮询 → GET /api/check-status → 获取车主确认和位置
```

### KV 存储键值
- `notify_status`: 通知状态 ('waiting' | 'confirmed')
- `requester_location`: 请求者位置信息（JSON）
- `owner_location`: 车主位置信息（JSON）
- `rate_limit:{ip}`: IP 访问时间戳（用于频率限制）

### 安全机制
1. **IP 地理位置限制**: 使用 `request.cf.country` 检查，仅允许中国 IP 访问
2. **访问频率限制**: 每个 IP 60 秒内只能发送一次通知请求
3. **位置验证**: 无位置信息时延迟 30 秒发送，降低恶意骚扰

## 部署和配置

### 本地开发
```bash
# 安装 Wrangler CLI
npm install -g wrangler

# 登录 Cloudflare
wrangler login

# 本地开发
wrangler dev

# 部署到 Cloudflare Workers
wrangler deploy
```

### 环境变量配置
在 Cloudflare Workers Dashboard 或 `wrangler.toml` 中配置：
- `BARK_URL`: Bark 推送服务地址（必需）
- `PHONE_NUMBER`: 备用联系电话（可选）

### KV 命名空间
必须创建并绑定 KV 命名空间：
- 变量名: `MOVE_CAR_STATUS`
- 用途: 存储通知状态、位置信息和访问频率限制数据

## 修改代码时的注意事项

### 修改 HTML 界面
- HTML 代码内嵌在 `renderMainPage()` 和 `renderOwnerPage()` 函数中
- 使用模板字符串，注意转义特殊字符
- CSS 样式内联在 `<style>` 标签中
- JavaScript 代码内联在 `<script>` 标签中

### 修改 API 逻辑
- 所有 API 处理函数都是 async 函数
- 使用 `MOVE_CAR_STATUS.get()` 和 `MOVE_CAR_STATUS.put()` 操作 KV
- 返回 JSON 响应时设置正确的 Content-Type 头

### 修改安全策略
- IP 检查在 `isChineseIP()` 函数中实现
- 频率限制在 `checkRateLimit()` 函数中实现
- 修改 `CONFIG.RATE_LIMIT_SECONDS` 调整限制时间

### 坐标系转换
- 浏览器 Geolocation API 返回 WGS-84 坐标
- 高德地图和 Apple 地图使用 GCJ-02 坐标
- 使用 `wgs84ToGcj02()` 进行转换，不要修改转换算法

## 常见问题

### 推送不工作
检查 `BARK_URL` 环境变量是否正确配置，格式应为 `https://api.day.app/your-key`

### KV 数据未保存
确认 KV 命名空间已正确绑定，变量名必须是 `MOVE_CAR_STATUS`

### IP 限制不生效
Cloudflare Workers 的 `request.cf.country` 仅在生产环境有效，本地开发时无法测试

### 位置信息不准确
这是浏览器 Geolocation API 的限制，需要用户授权并等待 GPS 定位
