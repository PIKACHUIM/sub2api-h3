# sub2api-h3：本 fork 相对上游的差异

基线：`Wei-Shaw/sub2api` 的 `530c5e3`，叠加 PR [#8446](https://github.com/Wei-Shaw/sub2api/pull/8446)（MiniMax 媒体计费）的 cherry-pick。上游未合并该 PR，本 fork 自行维护。

## 1. 合入的 PR #8446

新增 MiniMax 原生媒体接口（语音、音色、图片、H3 视频）、按秒/按张计量、分组媒体价卡与独立倍率。协议范围与计费语义见 [MINIMAX_MEDIA.md](./MINIMAX_MEDIA.md)。

## 2. 自定义媒体 origin（本 fork 新增）

上游把媒体地址锁死在 `https://api.minimax.cn` / `https://api.minimax.io`，理由是「不允许任意中转站」。本 fork 增加按账号开关，默认行为不变：

| 账号 `extra` 字段 | 说明 |
| --- | --- |
| `minimax_media_enabled` | 必须为 `true`，否则账号不参与媒体调度（上游已有） |
| `minimax_media_base_url` | 媒体接口地址。官方两域名直接用；其他地址必须同时开启下一项 |
| `minimax_media_allow_custom_origin` | `true` 时才接受任意 `http(s)` 地址，可带路径前缀 |

后台「编辑账号 → MiniMax 媒体 → 媒体接口地区」选「自定义 / 中转站」即会自动写入后两个字段。

```json
{
  "minimax_media_enabled": true,
  "minimax_media_base_url": "https://relay.example.com",
  "minimax_media_allow_custom_origin": true
}
```

约束与行为：

- 请求地址 = `origin` + 固定路径（`/v2/video_generation`、`/v2/query/video_generation/{id}`、`/v1/t2a_v2`、`/v1/image_generation`、`/v1/get_voice` 等），所以 origin 可带路径前缀，例如 `https://relay.example.com/minimax`。
- 带用户名密码、查询串、片段的地址直接拒绝，不会被静默裁剪；末尾 `/` 与前后的空白会去掉；主机名统一小写。
- 仍走过 `validateUpstreamBaseURL`：`security.url_allowlist.enabled: true` 时必须把该域名加入 `upstream_hosts`；`allow_insecure_http` 决定是否允许 `http://`。
- 上游返回的 `content.url` 原样透传给调用方；若该地址需要鉴权，调用方自行带凭据（MiniMax 官方托管文件要求同一 API Key）。
- 模型名/分辨率/时长仍按原生目录严格校验：中转站必须接受 `MiniMax-H3`、`MiniMax-H3-Highspeed`、`MiniMax-H3-Lite`、`MiniMax-H3-Max` 这些原生名。

## 3. GHCR 镜像

`.github/workflows/docker-ghcr.yml`：

- `verify`：`go build ./...` + `./internal/pkg/minimax/...` 与 `./internal/service/` 的 MiniMax/Media 测试。
- `image`：buildx 构建并推送 `ghcr.io/pikachuim/sub2api-h3`，tag 为 `main`、`h3`、`latest`、`sha-<短 sha>`，打 `v*` 标签时额外生成 semver tag。
- 触发：push 到 `main`、`v*` tag、手动 dispatch（可指定 `platforms`，默认仅 `linux/amd64`）。

## 4. 另一条路线：把 H3 伪装成 Seedance

不升级本 fork 也能用 H3：`/v1/contents/generations/tasks` 是通用透传，账号只需 `type=apikey` + 自定义 `base_url` + `credentials.openai_capabilities` 含 `seedance`，body 原样转发、只改写 `model`。需要一个把 Ark 协议转译成 MiniMax v2 的中间层（零依赖 Node 版 `ark-h3-adapter`，未纳入本仓库），部署到任意 https 域名后把账号 `base_url` 指向它即可。

两条路线可以并存：MiniMax 原生分组走本 fork 的 `/v2/*`，Seedance 分组走适配层。

## 5. 与上游同步

```bash
git fetch upstream main
git rebase upstream/main    # 冲突热点：channel.go、api_key_auth_cache_impl.go、wire_gen.go、UsageTable.spec.ts
```

`apiKeyAuthSnapshotVersion` 取两边较大值；`wire_gen.go` 的 `provideCleanup` 实参顺序必须与 `wire.go` 形参一致。
