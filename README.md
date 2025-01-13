# AI 同传音频采集系统

这是一个基于Vue.js开发的AI同传音频采集系统，用于批量登录和音频采集管理。

## 功能特点

- 批量账号登录管理
- 实时音频采集
- WebSocket实时通信
- 音频能量检测
- 多语言切换支持

## 技术栈

- Vue.js 2.6
- Element UI 2.15
- WebSocket
- Web Audio API
- Axios

## 环境要求

- Node.js >= 12.x
- npm >= 6.x

## 安装

```bash
# 安装依赖
npm install
```

## 开发环境配置

项目使用`.env`文件管理环境配置，包括：

```bash
# API基础URL
VUE_APP_API_BASE_URL=https://dtranxgateway_uat.dtranx.com/aiaudio

# WebSocket服务URL
VUE_APP_WS_BASE_URL=wss://aitongchuan-uat.dtranx.com/service

# 登录服务URL
VUE_APP_LOGIN_SERVICE_URL=https://shuyiloginservice_uat.dtranx.com/login
```

## 运行

```bash
# 开发环境运行
npm run serve

# 构建生产环境
npm run build

# 代码检查
npm run lint
```

## 使用说明

1. **批量登录**
   - 在文本框中输入账号密码，格式为：`账号,密码`（每行一个）
   - 点击"批量登录"按钮开始登录

2. **音频采集**
   - 登录成功后可以进行音频采集
   - 点击"全部开始"按钮开始采集
   - 可以实时查看音频能量值
   - 支持单个停止或全部停止采集

3. **语言设置**
   - 可以选择源语言和目标语言
   - 支持多种语言组合

## 注意事项

- 确保麦克风权限已开启
- 建议使用Chrome浏览器以获得最佳体验
- 音频采集时请保持网络稳定