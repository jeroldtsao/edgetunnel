# GitHub Actions 自动部署配置指南

本指南帮助你配置 GitHub Actions 自动部署 edgetunnel 到 Cloudflare Pages，支持自动设置 ADMIN 密码、自动创建 KV 命名空间和自动绑定自定义域名。

## 📋 前置条件

1. Cloudflare 账号
2. Fork 的 edgetunnel 仓库
3. 已配置的 GitHub Secrets
4. **域名已托管在 Cloudflare**（如需自动绑定域名）

## 🔧 配置步骤

### 第一步：获取 Cloudflare API Token

1. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com/)
2. 点击右上角头像 → `我的资料` → `API 令牌`
3. 点击 `创建令牌`
4. 选择 `编辑 Cloudflare Workers` 模板，或自定义权限：
   - `Account` → `Account Settings` → `读取`
   - `Account` → `Cloudflare Pages` → `编辑`
   - `Account` → `Workers KV Storage` → `编辑`
   - `Account` → `Workers Scripts` → `编辑`
   - `Zone` → `DNS` → `编辑`（自动绑定域名需要）
   - `Zone` → `Zone` → `读取`
5. 点击 `继续以摘要显示` → `创建令牌`
6. **⚠️ 复制令牌并保存**（只显示一次）

### 第二步：获取 Account ID

1. 在 Cloudflare Dashboard 右侧边栏找到你的账号
2. 点击账号名称，在右侧信息栏可以看到 `账号 ID`
3. 复制该 ID

### 第三步：配置 GitHub Secrets 和 Variables

进入你 Fork 的仓库 → `Settings` → `Secrets and variables` → `Actions`

#### 必需的 Secrets（敏感信息）

| Secret 名称 | 说明 | 示例值 |
|------------|------|--------|
| `CLOUDFLARE_API_TOKEN` | Cloudflare API 令牌 | 第一步获取 |
| `CLOUDFLARE_ACCOUNT_ID` | Cloudflare 账号 ID | 第二步获取 |
| `ADMIN_PASSWORD` | 后台管理员密码 | `your_password` |

点击 `New repository secret` 添加上述 secrets。

#### 可选的 Secrets（扩展配置）

| Secret 名称 | 说明 | 示例值 |
|------------|------|--------|
| `UUID` | 固定 UUID（UUIDv4 格式） | `90cd4a77-141a-43c9-991b-08263cfe9c10` |
| `PROXYIP` | 自定义反代 IP | `proxyip.cmliussss.net:443` |
| `KEY` | 快速订阅密钥 | `CMLiussss` |

#### 可选的 Variables（非敏感配置）

| Variable 名称 | 默认值 | 说明 |
|--------------|-------|------|
| `CLOUDFLARE_PAGES_PROJECT_NAME` | `edgetunnel` | Cloudflare Pages 项目名称 |

点击 `Variables` 标签 → `New repository variable` 添加（可选）。

### 第四步：触发首次部署

#### 首次完整部署（推荐）

1. 进入 GitHub 仓库 → `Actions`
2. 选择 `Deploy to Cloudflare Pages`
3. 点击 `Run workflow`
4. 配置选项：
   - **部署环境**: `production`
   - **自动创建 KV**: `true` ✓
   - **KV 名称**: `EDT2`（可自定义）
   - **自定义域名**: `vless.example.com`（可选，需域名已托管在 CF）
5. 点击 `Run workflow`

部署过程中会自动：
- ✅ 创建 Pages 项目（如不存在）
- ✅ 创建 KV 命名空间（如不存在）
- ✅ 将 KV 绑定到 Pages 项目（变量名 `KV`）
- ✅ 设置 ADMIN 等环境变量
- ✅ 绑定自定义域名（如指定且域名已托管）
- ✅ 自动添加 CNAME 记录

#### 后续部署

推送代码到 `main` 分支会自动触发部署。

### 第五步：域名要求（自动绑定）

如需自动绑定域名，需满足以下条件：

1. **域名已托管在 Cloudflare**：主域名（如 `example.com`）必须已添加到 Cloudflare DNS
2. **次级域名**：填写完整的次级域名（如 `vless.example.com`）
3. **API Token 权限**：必须包含 `Zone` → `DNS` → `编辑` 权限

如果域名未托管在 Cloudflare，需要手动添加域名后才能自动绑定。

## 🌐 访问管理后台

部署完成后访问：
- 默认地址：`https://<project>.pages.dev/admin`
- 自定义域名：`https://<custom-domain>/admin`（如已绑定）
- 使用 `ADMIN_PASSWORD` 密码登录

## ⚠️ 注意事项

1. **API Token 权限**：需包含 Pages、KV、Workers、Zone DNS 编辑权限
2. **KV 绑定名称**：自动绑定为 `KV`（大写），符合项目要求
3. **ADMIN 密码**：通过 `ADMIN_PASSWORD` Secret 自动设置
4. **域名托管**：域名必须已在 Cloudflare 才能自动绑定
5. **首次部署**：建议勾选所有选项一次性完成配置

## 📚 参考链接

- [edgetunnel 部署教程](https://blog.cmliussss.com/p/edt2/)
- [Cloudflare Pages 文档](https://developers.cloudflare.com/pages/)
- [wrangler-action 文档](https://github.com/cloudflare/wrangler-action)

## 🆘 常见问题

### Q: 部署失败提示 "Authentication error"
检查 `CLOUDFLARE_API_TOKEN` 是否正确，令牌权限需包含 Pages、KV、Workers、Zone DNS 编辑权限。

### Q: KV 创建失败
确保 API Token 有 `Workers KV Storage` → `编辑` 权限。

### Q: ADMIN 密码无效
检查 `ADMIN_PASSWORD` Secret 是否正确设置。

### Q: 域名绑定失败，提示"域名未托管"
需要先在 Cloudflare Dashboard 添加主域名（如 `example.com`），然后再触发部署。

### Q: 如何修改密码
在 GitHub Secrets 中更新 `ADMIN_PASSWORD` 值，然后重新触发部署。

### Q: 如何更换域名
手动触发部署，填写新的自定义域名即可。