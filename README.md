# sub-converter-stack

自建订阅转换与短链接服务一键部署方案。基于开源项目二次定制，去除与个人使用无关的内容，专注于核心功能。

---

## 项目组成

| 组件 | 镜像 / 来源 | 说明 |
|---|---|---|
| 前端网页 | 本仓库自行构建 | 基于 [youshandefeiyang/sub-web-modify](https://github.com/youshandefeiyang/sub-web-modify) 定制 |
| 订阅转换后端 | `aethersailor/subconverter-extended` | 支持 Vless/Hysteria2/TUIC 等协议，[项目地址](https://github.com/aethersailor/subconverter-extended) |
| 短链接服务 | `stilleshan/myurls` | 自建短网址生成与重定向，[项目地址](https://github.com/stilleshan/myurls) |
| 短链缓存数据库 | `redis:alpine` | 存储短链映射 |

---

## 定制说明

基于原版做了以下个人化改动：

- **界面极简化**：精简页面结构，保留核心功能，去除与个人使用无关的导航入口
- **视觉优化**：纯白背景，自定义渐变 SVG favicon，按钮样式深色/浅色主题适配修复
- **智能路由**：局域网访问时自动使用本地后端，公网访问时切换为公网后端
- **短链集成**：新增「普通短链」标签页，可在前端直接为任意链接生成短网址
- **跨域代理**：通过前端 Nginx 反代 `/short` 接口，规避浏览器 CORS 限制

---

## 部署方法

### 前置条件

- 一台装有 Docker 和 Docker Compose 的服务器（Linux 推荐）
- 一个指向该服务器的域名，并已申请 SSL 证书（推荐使用 [Nginx Proxy Manager](https://nginxproxymanager.com/)）

### 第一步：克隆仓库

```bash
git clone https://github.com/White-Lie-88896/sub-converter-stack.git
cd sub-converter-stack/sub-services-backup
```

### 第二步：配置环境变量

```bash
cd sub-web-modify
cp .env.example .env
nano .env   # 将三处 your.domain.xyz 替换为您自己的域名
```

`.env` 中需要修改的三项：

```env
VUE_APP_SUBCONVERTER_DEFAULT_BACKEND = "https://sub.你的域名"
VUE_APP_MYURLS_DEFAULT_BACKEND       = "https://sub.你的域名"
VUE_APP_CONFIG_UPLOAD_BACKEND        = "https://sub.你的域名"
```

同时修改 `docker-compose.yml` 中的短链域名：

```yaml
environment:
  - MYURLS_DOMAIN=sub.你的域名   # ← 改这里
  - MYURLS_PROTO=https
```

### 第三步：构建前端镜像

```bash
# 在 sub-services-backup/sub-web-modify 目录下执行
docker build -t local/sub-web-modify:latest .
```

> 首次构建需要下载 Node.js 依赖，约需 1~3 分钟。

### 第四步：启动所有服务

```bash
# 回到 sub-services-backup 目录
cd ..
docker compose up -d
```

验证所有容器已启动：

```bash
docker ps
```

应看到以下四个容器均处于 `Up` 状态：

| 容器名 | 端口 | 作用 |
|---|---|---|
| `sub-web-modify` | `8090:80` | 前端网页 |
| `subconverter-extended` | `25500:25500` | 订阅转换 API |
| `myurls` | `8002:8080` | 短链接服务 |
| `myurls-redis` | 无公开 | 短链缓存数据库 |

### 第五步：配置反向代理

使用 Nginx Proxy Manager（或其他反代工具），将您的域名代理至前端容器，并在**高级配置（Advanced）**中添加以下 location 规则，确保 API 路由优先级正确：

```nginx
# 订阅转换 API（^~ 确保优先于短链正则）
location ^~ /sub {
    proxy_pass http://127.0.0.1:25500/sub;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}

location ^~ /version {
    proxy_pass http://127.0.0.1:25500/version;
    proxy_set_header Host $host;
}

# 短链生成 API
location ^~ /short {
    proxy_pass http://127.0.0.1:8090/short;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}

# 短链跳转（匹配 3-10 位字母数字）
location ~ "^/[a-zA-Z0-9]{3,10}$" {
    proxy_pass http://127.0.0.1:8002;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

> **注意**：`^~` 修饰符必须加，否则短链正则会抢匹配 `/sub`、`/version` 等路径，导致 API 404。

### 第六步：验证

```bash
# 后端版本
curl https://sub.你的域名/version

# 生成一条短链（测试）
curl -X POST https://sub.你的域名/short \
  -d "longUrl=$(echo -n 'https://google.com' | base64)"
```

---

## 后续更新前端

修改了 `sub-web-modify/src/views/Subconverter.vue` 后，重新构建并重启即可：

```bash
docker build -t local/sub-web-modify:latest ./sub-web-modify
docker compose restart sub-web
```

---

## 许可证

本项目为个人使用目的的二次定制，遵循各上游项目的开源协议，请勿用于商业用途。
