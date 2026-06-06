# 🌐 cool-bose: 自建订阅转换与短链接服务集成包

本项目是为个人定制的订阅转换（Subscription Converter）与短链服务（Shortener）一键部署方案，包含完整的 Docker Compose 部署配置、自动化脚本、自定义反向代理配置以及深度优化后的去广告 Vue 前端代码。

---

## 🛠️ 项目结构与组件来源

整个工作区包含以下核心组件：

### 1. 🎨 订阅转换前端网页 (`sub-services-backup/sub-web-modify`)
* **原项目来源**：
  * 最早基于 CareyWang 的 [sub-web](https://github.com/CareyWang/sub-web)
  * 后期重构基于 肥羊 (youshandefeiyang) 的 [sub-web-modify](https://github.com/youshandefeiyang/sub-web-modify)
* **本次个人定制修改内容**：
  * **界面纯净化**：移除了原版的所有广告弹窗、SweetAlert2 机场推广弹窗，并清理了顶部导航栏的 GitHub、Telegram、YouTube、Bilibili 等社交图标和“视频教程”按钮。
  * **视觉系统优化**：强制设置页面为极简纯白背景，更新浏览器 Tab 的 favicon 图标为现代渐变矢量 `favicon.svg`。
  * **按钮可读性修复**：修复了在浅色/深色主题下因为全局样式强制覆盖导致的辅助按钮与复制按钮看不清、颜色填充不完全的问题。
  * **智能路由逻辑**：支持根据访问源自动切换后端。局域网访问默认使用本地局域网后端，公网访问时默认使用公网自建后端。
  * **短链接口本地代理**：优化短链接 API 的网络路由，配置反代规避了浏览器的跨域 CORS 错误。
  * **独立短链功能**：在前端额外引入了独立的「普通短链」生成标签页，允许在同一个界面中直接缩短任意长网址链接。

### 2. 🔌 订阅转换后端 (`subconverter-extended`)
* **镜像来源**：使用 aethersailor 的 [subconverter-extended](https://github.com/aethersailor/subconverter-extended) 增强版后端。
* **特性**：内置支持 Vless Reality, Encryption, Hysteria2, AnyTLS, TUIC, Mieru 等增强型节点的订阅转换。

### 3. 🔗 短网址重定向服务 (`myurls`)
* **镜像来源**：使用 stilleshan 的 [myurls](https://github.com/stilleshan/myurls) 自建短链接服务。
* **数据库**：配合轻量级 `redis:alpine` 容器存储短链映射键值。

---

## 🚀 部署与使用指南

项目的所有配置文件都经过适配，可以直接运行在您的公网 VPS `DC1` 上：

1. **部署目录**：`/opt/sub-services`
2. **启动命令**：
   ```bash
   docker compose -f docker-compose.yml up -d --build
   ```
3. **接口配置**：通过 Nginx Proxy Manager 反向代理前端网页及 `/sub` (转换接口) 与 `/short` (短链接口)，并开启 `https://sub.your.domain.xyz` 加密访问。
