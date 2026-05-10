# AGENTS.md

## 项目概览

Cloudflare Pages + Workers 应用 — 一个基于 R2 的云盘系统，支持自定义认证、多用户权限管理、媒体预览和批量操作。

- **前端**: Vue 3 SPA，通过 `vue3-sfc-loader` 在**运行时**加载（无构建/打包步骤）。入口：`index.html` → `assets/App.vue`。
- **后端**: Cloudflare Pages Functions，位于 `functions/` 目录（基于文件系统的路由，onRequest 处理函数）。共享工具函数在 `utils/`。
- **部署目标**: Cloudflare Pages，需要绑定名为 `BUCKET` 的 R2 存储桶，以及可选的名为 `LOGIN_ATTEMPTS` 的 KV 绑定。

## 开发命令

```bash
npm run dev          # 启动 wrangler pages dev --r2 BUCKET（需要本地 R2 环境）
```

没有定义 lint、typecheck 或 test 命令。包管理器为 `yarn`（`yarn@1.22`）。

## 架构说明

### 前端（无构建步骤）

- `assets/` 中的 Vue 3 SFC 文件在**运行时**由 `vue3-sfc-loader` 加载 — 它们**不会被预先编译**。编辑 `.vue` 文件后刷新页面即可生效。
- Vue、axios 和 vue3-sfc-loader 在 `index.html` 中从 CDN 加载。
- `assets/main.mjs` 导出共享工具函数（`generateThumbnail`、`blobDigest`、`multipartUpload`、`SIZE_LIMIT`）。
- 认证凭据存储在 `localStorage`（`authCredentials`、`currentUser`）中，并以 `Authorization: Basic ...` 头注入到 fetch 请求中。

### 后端（Cloudflare Pages Functions）

- `functions/` 下的基于文件系统的路由：
  - `functions/api/children/[[path]].ts` — 列出文件/文件夹，带权限过滤
  - `functions/api/write/items/[[path]].ts` — 上传、复制、移动、删除（PUT/DELETE + 分片上传）
  - `functions/api/auth/login.ts` — 登录，带基于 KV 的暴力破解防护
  - `functions/api/auth/check-write-permission.ts` — 检查用户是否可对某路径写入
  - `functions/api/config.ts` — 向前端暴露环境变量
  - `functions/raw/[[path]].ts` — 通过公共 R2 URL（`PUBURL` 环境变量）代理文件
- `utils/auth.ts` — 多个 API 处理函数共用的认证状态检查；解析 `username:password` 和 `username:password:r`（只读）格式的环境变量
- `utils/bucket.ts` — `parseBucketPath` 从子域名解析 R2 绑定（`env[driveid]`）或回退到 `env.BUCKET`
- `utils/s3.ts` — 自定义 `S3Client`，实现 AWS Signature V4，用于跨桶 S3 API 调用
- TS 路径别名 `@/` 映射到仓库根目录（用于 `@/utils/auth` 这类导入）

### 认证与权限

- 用户通过 Cloudflare Pages 环境变量定义：
  - `admin:password=*` — 管理员（完全访问权限）
  - `user:password=dir1/,dir2/` — 普通用户（对指定目录有读写权限）
  - `user:password:r=dir1/` — 只读用户（目录必须以 `/` 结尾）
  - `GUEST=public/` 或 `guest=public/` — 未登录游客可访问的目录
- 权限基于路径前缀。特殊目录 `_$flaredrive$/` 用于内部系统文件（CNAME 标记、缩略图）。
- 登录限制需要绑定 KV 命名空间 `LOGIN_ATTEMPTS` — 最多 5 次尝试，封禁 30 分钟，1 小时后自动重置。

### 目录标记

R2 中的文件夹使用 `_$folder$` 后缀的 key 表示（例如 `docs/_$folder$`）。前端和 API 会将这些标记从普通文件列表中过滤掉。

## 文件修改指南

- **前端修改**：编辑 `assets/` 中的 `.vue` 文件，或编辑 `index.html` 来更改 CDN 依赖或全局配置。
- **后端修改**：编辑 `functions/` 或 `utils/` 中的 `.ts` 文件。
- **静态资源**：`favicon.ico`、`robots.txt`、`404.html`、`assets/bg-light.webp`、`assets/homescreen.png`。
- `404.html` 不是标准 404 页面——它通过 meta refresh 自动重定向到 `/`，用于 SPA 路由。
- `docs/` 目录包含中文的开发日志/变更日志——仅供信息参考，不属于应用程序代码。
