# supabase-backup

Supabase 数据库自动备份，通过 GitHub Actions 每日执行。

## 备份内容

每次备份生成 3 个文件：

| 文件 | 内容 |
|------|------|
| `roles.sql` | 数据库角色与权限 |
| `schema.sql` | 表结构、索引、函数 |
| `data.sql` | 表数据 |

## 目录结构

```
backups/
├── 2026-06-01/
│   ├── roles.sql
│   ├── schema.sql
│   └── data.sql
├── 2026-06-02/
│   └── ...
└── ...
```

## 配置

在仓库 **Settings → Secrets and variables → Actions → Secrets** 中添加：

| Secret | 说明 |
|------|------|
| `SUPABASE_DB_URL` | Supabase 数据库连接字符串 |

连接字符串格式：
```
postgresql://postgres:<密码>@db.<项目ID>.supabase.co:5432/postgres
```

在 Supabase Dashboard → Settings → Database → Connection Info 中复制。

## 运行方式

- **自动**：每天北京时间 10:00
- **手动**：Actions → Supabase 数据库备份 → Run workflow

## 恢复

在 Supabase SQL Editor 中按顺序执行：

```
roles.sql → schema.sql → data.sql
```
