# GitHub Actions 自动部署指南

本仓库已改为通过 GitHub Actions 部署到 Cloudflare Workers。相比 Pages 部署，Workers 的 KV 绑定和自定义域名都由 Wrangler 在同一次发布中处理，少了 Pages 项目配置、CNAME 验证和 API PATCH 漂移的问题。

## 一、Cloudflare API Token

在 Cloudflare Dashboard 创建 API Token，建议使用自定义令牌并授予：

| 范围 | 权限 | 用途 |
|------|------|------|
| Account | Workers Scripts: Edit | 发布 Worker |
| Account | Workers KV Storage: Edit | 创建/读取 KV namespace（仅 `bootstrap` 模式需要） |
| Account | Account Settings: Read | Wrangler 校验账号 |
| Zone | Workers Routes: Edit | 绑定自定义域名 |
| Zone | DNS: Edit | 自动创建自定义域名 DNS 记录 |
| Zone | Zone: Read | 查找域名所在 zone |

如果不使用 `CUSTOM_DOMAIN`，Zone 相关权限可以不加。

## 二、GitHub Secrets

进入仓库 `Settings` -> `Secrets and variables` -> `Actions`，添加以下 Secrets：

| Secret | 必填 | 说明 |
|--------|------|------|
| `CLOUDFLARE_API_TOKEN` | 是 | Cloudflare API Token |
| `CLOUDFLARE_ACCOUNT_ID` | 是 | Cloudflare Account ID |
| `ADMIN_PASSWORD` | 是 | 后台 `/admin` 登录密码，会部署为 Worker secret `ADMIN` |
| `UUID` | 否 | 固定 UUID，必须是 UUIDv4 |
| `PROXYIP` | 否 | 自定义反代地址 |
| `KEY` | 否 | 快速订阅密钥 |
| `HOST` | 否 | 自定义订阅里使用的 host |
| `URL` | 否 | 默认伪装首页地址 |
| `GO2SOCKS5` | 否 | SOCKS5 规则名单 |
| `DEBUG` | 否 | `1` 或 `true` 开启调试日志 |
| `OFF_LOG` | 否 | `1` 或 `true` 关闭 KV 日志 |
| `BEST_SUB` | 否 | `1` 或 `true` 开启优选订阅生成器 |

## 三、GitHub Variables

同一页面切到 `Variables`，按需添加：

| Variable | 默认值 | 说明 |
|----------|--------|------|
| `CLOUDFLARE_WORKER_NAME` | `edgetunnel` | Worker 名称，只能用小写字母、数字和连字符 |
| `CUSTOM_DOMAIN` | 空 | 自定义域名，例如 `vless.example.com`，不要带 `https://` 或路径 |
| `KV_ID` | 空 | 已创建 KV namespace 的 id，`deploy` 模式和 `push` 发布都会使用它 |
| `KV_NAME` | `EDT2` | `bootstrap` 模式用于创建或查找 KV namespace 的名称 |

兼容旧配置：如果没有设置 `CLOUDFLARE_WORKER_NAME`，workflow 会继续读取旧的 `CLOUDFLARE_PAGES_PROJECT_NAME`。

## 四、部署

首次接入时建议按下面顺序执行：

1. 在 `Actions` 页面手动运行 `Deploy to Cloudflare Workers`，选择 `mode=bootstrap`。
2. workflow 会创建或复用名为 `KV_NAME` 的 KV namespace，并在 Summary 输出 `KV_ID`。
3. 把这个 `KV_ID` 保存到仓库 `Variables`。
4. 再次手动运行同一个 workflow 并选择 `mode=deploy`，或者以后直接推送到 `main` 分支。

之后的日常更新只需要 `push` 到 `main`，workflow 会直接使用固定的 `KV_ID` 发布 Worker，不再在每次发版时探测或创建 KV。

workflow 分为两种模式：

1. `bootstrap`
   - 校验 Cloudflare 凭据和 `KV_NAME`。
   - 使用 Wrangler 校验 Cloudflare 登录。
   - 创建或复用名为 `KV_NAME` 的 KV namespace。
   - 在 Summary 输出 `KV_ID`，供你保存到仓库 Variables。
2. `deploy`
   - 校验必需 Secrets、`KV_ID`、Worker 名称和自定义域名格式。
   - 使用 Wrangler 校验 Cloudflare 登录。
   - 生成临时 `wrangler.generated.toml`，固定绑定名为 `KV`。
   - 用 `--secrets-file` 注入 `ADMIN` 等 secrets 并发布 Worker。
   - 如果设置了 `CUSTOM_DOMAIN`，通过 Workers Custom Domain 自动绑定域名。

部署成功后访问：

- 默认 Worker 地址：以 Actions Summary 中输出为准。
- 自定义域名：`https://<CUSTOM_DOMAIN>/admin`。

## 五、自定义域名注意事项

`CUSTOM_DOMAIN` 使用的是 Workers 自定义域名，不再是 Pages CNAME。请确认：

1. 主域名已经托管在同一个 Cloudflare 账号中。
2. 填写的是完整子域名，例如 `vless.example.com`。
3. 不要填写根域名、URL、路径或通配符。
4. API Token 有 `Zone:Workers Routes:Edit`、`Zone:DNS:Edit`、`Zone:Zone:Read`。

如果域名之前已经在 Pages 或其他 Worker 里绑定，先到 Cloudflare 控制台删除旧绑定，再重新运行 workflow。

## 六、常见问题

### KV 绑定失败

如果是 `bootstrap` 失败，先检查 Token 是否有 `Workers KV Storage: Edit`。如果是 `deploy` 失败并提示缺少 `KV_ID`，说明还没有先完成一次 `bootstrap`，或者仓库 Variables 里的 `KV_ID` 没填对。

### 后台提示 noKV

通常是你访问的不是本 workflow 发布的 Worker，或者 Cloudflare 上旧版本还未刷新。重新运行一次 workflow，并确认 Summary 里的 Worker 名和访问域名一致。

### 自定义域名不可用

确认域名已托管在 Cloudflare，并且没有残留的 Pages 自定义域名绑定或冲突 DNS 记录。Workers Custom Domain 正常情况下会自动创建 DNS 记录和证书。

### 如何改密码或变量

更新 GitHub Secrets 后重新运行 workflow。`ADMIN_PASSWORD` 会作为 Worker secret `ADMIN` 注入。

### 如何更换或重建 KV

修改 `KV_NAME` 后重新手动运行 `mode=bootstrap`，拿到新的 `KV_ID` 后覆盖仓库 Variables 里的 `KV_ID`，再执行一次 `deploy` 即可。

### 如何让部分域名走直连

在后台配置 JSON 中维护 `直连规则`，值可以是数组，也可以是逗号/换行分隔的字符串。保存后重新获取订阅即可，不需要重新部署。

```json
{
  "直连规则": ["m-team", "lbx"]
}
```

## 七、参考

- [Cloudflare Workers Custom Domains](https://developers.cloudflare.com/workers/configuration/routing/custom-domains/)
- [Cloudflare Wrangler deploy](https://developers.cloudflare.com/workers/wrangler/commands/workers/#deploy)
- [Cloudflare Workers Secrets](https://developers.cloudflare.com/workers/configuration/secrets/)
- [Cloudflare Wrangler KV commands](https://developers.cloudflare.com/workers/wrangler/commands/kv/)
