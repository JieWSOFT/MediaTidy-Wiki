# 升级指南

## Docker 升级

### 标准升级流程

```bash
# 拉取最新镜像
docker pull murongyun574/mediatidy:latest

# 停止并删除旧容器
docker compose down

# 重新启动
docker compose up -d
```

### 指定版本升级

如需升级到特定版本，修改 `docker-compose.yml` 中的镜像标签：

```yaml
services:
  mediatidy:
    image: murongyun574/mediatidy:v0.0.21  # 指定版本号
```

然后重新启动：

```bash
docker compose up -d
```

## 数据迁移

### 数据库自动迁移

MediaTidy 内置数据库迁移机制，每次启动时会自动检测并执行必要的数据库结构变更。**无需手动操作**。

### 备份建议

升级前建议备份 `data` 目录：

```bash
# 停止服务
docker compose down

# 备份数据
cp -r ./data ./data-backup-$(date +%Y%m%d)

# 升级并启动
docker pull murongyun574/mediatidy:latest
docker compose up -d
```

## 版本查看

当前运行版本可在以下位置查看：

- **管理面板**：左下角或「关于」页面
- **启动日志**：`MediaTidy starting version=x.x.x`
- **API**：`GET /api/version`

## 注意事项

::: warning 重要
- 升级前务必备份 `data` 目录（包含数据库和配置）
- 不建议跨多个大版本直接升级，建议逐步升级
- 如遇问题，可将备份的 `data` 目录还原后回退到旧版本镜像
:::
