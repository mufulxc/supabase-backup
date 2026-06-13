# supabase-backup

Supabase 数据库自动备份，通过 GitHub Actions 每日执行，备份数据存储在仓库 `backups/` 目录下。

## 备份内容

每次备份生成 4 个文件：

| 文件 | 内容 | 说明 |
|------|------|------|
| `roles.sql` | 数据库角色与权限 | 角色定义、登录权限等（不含角色密码） |
| `schema.sql` | 完整表结构 | 所有 schema 的表、索引、函数、触发器、RLS 策略等 |
| `data.sql` | 用户数据 | 仅 `public` schema 的 table data（详见下方说明） |
| `settings_schema.sql` | settings 表结构 | `public.settings` 表结构参考，不含数据 |

### data.sql 包含范围

- ✅ `public` schema 中所有用户创建的表数据
- ❌ `public.settings` 表数据（可能含密码/API key，仅备份结构）
- ❌ `auth` / `vault` / `storage` / `realtime` schema 数据

> Supabase 内部 schema（`auth`、`vault`、`storage`、`realtime`）不导出数据，因为这些 schema 由 Supabase 管理，数据中包含 JWT 密钥、service_role 密钥等敏感信息，会被 GitHub 推送保护拦截。如需迁移这些数据，请在 Supabase Dashboard 中单独导出。

## 目录结构

```
backups/
├── 2026-06-13/
│   ├── roles.sql
│   ├── schema.sql
│   ├── data.sql
│   └── settings_schema.sql
├── 2026-06-14/
│   └── ...
└── ...
```

- 每日子目录以 `YYYY-MM-DD` 命名
- **保留 90 天**，超过 90 天的备份自动清理

## 配置

在仓库 **Settings → Secrets and variables → Actions → Secrets** 中添加：

| Secret | 说明 |
|--------|------|
| `SUPABASE_DB_URL` | Supabase 数据库连接字符串 |

连接字符串格式：
```
postgresql://postgres:<密码>@db.<项目ID>.supabase.co:5432/postgres
```

获取方式：Supabase Dashboard → Settings → Database → Connection Info → 选择 **URI** 格式。

## 运行方式

| 方式 | 说明 |
|------|------|
| **自动** | 每天北京时间 10:00（UTC 2:00）执行 |
| **手动** | Actions → Supabase 数据库备份 → Run workflow |

## 恢复

在 Supabase SQL Editor 中**按顺序**执行：

```
roles.sql → schema.sql → data.sql
```

`settings_schema.sql` 为参考文件，`public.settings` 表的数据需要手动恢复。

## 技术细节

- 使用 PostgreSQL 17 客户端工具（`pg_dumpall` + `pg_dump`）
- 通过 SSL 加密连接 Supabase 数据库
- 备份前自动测试数据库连接，连接失败时明确报错
- 基于目录名日期比较清理过期备份（比 `-mtime` 更可靠）
- `schema.sql` 和 `roles.sql` 即使备份失败也会回退生成占位文件

## 故障排查

| 问题 | 可能原因 | 解决方案 |
|------|---------|---------|
| `无法连接到 Supabase 数据库` | `SUPABASE_DB_URL` 错误或密码过期 | 检查 Secret 是否正确，在 Supabase Dashboard 重新生成密码 |
| `schema 备份失败` / `data 备份失败` | 数据库权限不足 | 检查 Supabase 项目状态，确认数据库可访问 |
| GitHub 推送保护拦截 | Supabase 内部 schema（`auth`、`vault`）的数据中包含 JWT/service_role 密钥 | 确认工作流已配置 `--schema=public` 跳过内部 schema；如用户表中存了 API key，在 `--exclude-table-data` 中添加对应表名 |
