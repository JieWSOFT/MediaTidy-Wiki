# 前置条件

在使用 MediaTidy 之前，请确保以下服务或组件已准备就绪。

## 必需组件

### CloudDrive2（CD2）

MediaTidy 通过 CloudDrive2 的 gRPC 接口与云盘交互，这是核心依赖。

- **安装 CD2**：参考 [CloudDrive2 官方文档](https://www.clouddrive2.com/) 完成安装
- **挂载网盘**：在 CD2 中添加 115 网盘并完成挂载
- **记录连接信息**：CD2 的地址（如 `http://192.168.1.100:19798`）和登录凭据

::: warning 注意
MediaTidy 需要 CD2 保持运行状态，所有文件浏览、移动、复制等操作都通过 CD2 完成。
:::

### 115 网盘账号

- 至少一个 115 网盘账号
- 获取 115 Cookie（用于直连 API 模式）
- Cookie 可在浏览器登录 115 后从开发者工具中获取

### TMDB API Key

自动整理功能依赖 TMDB（The Movie Database）进行媒体识别。

1. 注册 [TMDB 账号](https://www.themoviedb.org/signup)
2. 进入 [API 设置页面](https://www.themoviedb.org/settings/api) 申请 API Key
3. 在 MediaTidy 的「系统设置」中填入 API Key

### 授权码

MediaTidy 为商业授权软件，需要有效的授权码才能运行。

- 通过 `LICENSE_CODE` 环境变量传入
- 授权码绑定设备，更换设备需重新授权

## 可选组件

### Emby Server

如需使用 Emby 反向代理（302 直链播放）功能：

- 安装并运行 Emby Server
- 获取 Emby 的 API Key（管理面板 → 高级 → API 密钥）
- 记录 Emby 服务器地址和端口

### FUSE 挂载

如需使用 FUSE 模式（本地文件系统访问），需要：

- CD2 开启 FUSE 挂载
- Docker 容器挂载 FUSE 目录（如 `/CloudNAS:/CloudNAS:rslave`）

### Telegram Bot

如需 Telegram 通知或 Bot 交互功能：

- 通过 [@BotFather](https://t.me/BotFather) 创建 Bot 并获取 Token
- 获取你的 Chat ID

## 网络要求

| 服务 | 地址 | 用途 |
|------|------|------|
| CloudDrive2 | 局域网 | 文件操作核心 |
| TMDB API | api.themoviedb.org | 媒体识别 |
| 115 API | 115.com | 直连模式 |
| Emby Server | 局域网 | 代理播放 |
| 授权服务器 | 公网 | 授权验证 |

::: tip 代理配置
如果网络环境无法直接访问 TMDB，可在「系统设置」中配置 HTTP 代理。
:::
