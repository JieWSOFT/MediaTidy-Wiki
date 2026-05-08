# 快速开始

## Docker 部署（推荐）

MediaTidy 推荐使用 Docker 部署，一行命令即可启动。

### 1. 创建 docker-compose.yml

```yaml
services:
  mediatidy:
    image: murongyun574/mediatidy:latest
    container_name: mediatidy
    restart: unless-stopped
    ports:
      - "2019:2019"             # 管理面板
    volumes:
      - ./data:/app/data        # 数据持久化
      - /CloudNAS:/CloudNAS:rslave  # CD2 FUSE 挂载点（需 rslave 传播挂载事件）
      - ./strm:/app/strm        # STRM 输出目录
    environment:
      - TZ=Asia/Shanghai
      - LICENSE_CODE=<your-license-code>
```

### 2. 启动服务

```bash
docker compose up -d
```

### 3. 访问管理面板

启动后打开浏览器访问：

```
http://<your-ip>:2019
```

首次访问需要注册管理员账号。

## 初始配置流程

部署完成后，建议按以下顺序完成配置：

1. **注册账号** — 首次访问时创建管理员用户
2. **网盘配置** — 添加 CloudDrive2 连接和 115 账号
3. **系统设置** — 配置 TMDB API Key（自动识别必需）
4. **创建整理方案** — 设置源目录、目标目录、命名规则
5. **创建 STRM 方案** — 设置源目录和输出目录
6. **Emby 代理**（可选）— 配置反向代理和 302 直链播放

::: tip 提示
网盘配置和 TMDB API Key 是使用核心功能（自动整理、STRM 生成）的前提条件，请优先完成。
:::

## 目录说明

| 容器路径 | 说明 |
|----------|------|
| `/app/data` | 数据持久化目录（数据库、日志、配置） |
| `/app/strm` | STRM 文件默认输出目录（可在方案中修改） |
| `/CloudNAS` | CD2 FUSE 挂载点（需与 CD2 的挂载路径一致） |

## 查看日志

```bash
# 查看实时日志
docker logs -f mediatidy

# 查看最近 100 行
docker logs --tail 100 mediatidy
```

日志文件也保存在 `./data/logs/` 目录下。
