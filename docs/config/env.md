# 环境变量

MediaTidy 支持通过环境变量配置运行时参数。

## 完整列表

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `SERVER_PORT` | `8080` | 后端 HTTP 服务端口 |
| `DB_PATH` | `/app/data/mediatidy.db` | SQLite 数据库文件路径 |
| `LOG_DIR` | `/app/data/logs` | 日志文件输出目录 |
| `LOG_LEVEL` | `info` | 日志级别：`debug` / `info` / `warn` / `error` |
| `LICENSE_CODE` | — | **必填**，授权码 |
| `TZ` | — | 时区（如 `Asia/Shanghai`） |
| `GOMEMLIMIT` | `512MiB` | Go 运行时软内存限制 |
| `STATIC_DIR` | — | 前端静态文件目录（Docker 中自动设置） |
| `JWT_SECRET` | 内置默认值 | JWT 签名密钥 |

## Docker Compose 配置示例

```yaml
services:
  mediatidy:
    image: murongyun574/mediatidy:latest
    container_name: mediatidy
    restart: unless-stopped
    ports:
      - "2019:2019"
    volumes:
      - ./data:/app/data
      - /CloudNAS:/CloudNAS:rslave
      - ./strm:/app/strm
    environment:
      - TZ=Asia/Shanghai
      - LICENSE_CODE=your-license-code
      # - LOG_LEVEL=debug        # 调试时开启
      # - GOMEMLIMIT=1GiB        # 大规模库可适当调大
```

## 说明

### LICENSE_CODE

授权码是运行 MediaTidy 的必要条件。未配置或授权失效时，服务会停止但进程保持存活（避免 Docker 无限重启）。

### GOMEMLIMIT

控制 Go 运行时的内存使用上限。默认 512MiB 适合大多数场景。如果媒体库规模较大（数万文件），可适当增加到 1GiB。

### LOG_LEVEL

- `debug`：输出所有调试信息（日志量大，仅调试时使用）
- `info`：常规运行信息（推荐）
- `warn`：仅警告和错误
- `error`：仅错误信息

日志级别也可以在管理面板的「系统设置」中动态修改，无需重启。
