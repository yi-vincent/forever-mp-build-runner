# forever-mp-build-runner

公开的 GitHub Actions runner：在私有仓库 `yi-vincent/forever-talents-mp` push 之后，由本仓库执行小程序构建、上传微信后台，并推企业微信通知。

本仓库**只含** workflow 与本 README（外加 `.keepalive`），不放私有仓库的任何源码、数据或明文产物。构建日志重定向到 runner 临时目录；上传到 Actions 的 artifact 一律加密。

参照同结构的 [`forever-sync-runner`](https://github.com/yi-vincent/forever-sync-runner)。

## 需要的配置（本仓库 Settings → Secrets and variables → Actions）

### Variables

| 名字 | 值 | 说明 |
| --- | --- | --- |
| `CONTENT_REPO` | `yi-vincent/forever-talents-mp` | 私有内容仓库 |

### Secrets

| 名字 | 最小权限 / 来源 | 说明 |
| --- | --- | --- |
| `CONTENT_REPO_TOKEN` | Fine-grained PAT：Repository access 只选 `forever-talents-mp`，**Contents: Read**（Metadata: Read 自动带上） | 检出私有仓库指定 sha |
| `MP_UPLOAD_KEY` | 微信小程序后台 → 开发管理 → 开发设置 → 小程序代码上传 → 生成的 `private.*.key` 全文 | 只注入「上传」那一步；GitHub 出口 IP 不固定，后台需关掉上传密钥 IP 白名单 |
| `ARTIFACT_PASSPHRASE` | 自定长随机口令 | 加密 artifact；**没配就不上传 artifact**（公开仓库不放明文） |
| `NOTIFY_WEBHOOK` | 自建 Message Pusher 的 webhook URL | 企业微信卡片通知；没配则跳过通知 |

### 私有仓库侧（`forever-talents-mp`，不配在本仓库）

| 名字 | 最小权限 | 说明 |
| --- | --- | --- |
| `BUILD_RUNNER_TOKEN` | Fine-grained PAT：Repository access 只选 `forever-mp-build-runner`，权限只开 **Contents: Read and write**（Metadata: Read 会自动带上） | 调 `POST /repos/yi-vincent/forever-mp-build-runner/dispatches` 需要对目标仓库 Contents 读写；只读不行 |

## 触发方式

1. **自动**：私有仓库任意分支 push（路径命中 `src/**`、`package.json`、`pnpm-lock.yaml`、`scripts/**`）→ `dispatch-build.yml` 对本仓库发 `repository_dispatch`（`event_type=mp-build`，payload 为 `{ref, sha, group}`；`group` = ref 的 sha256 前 16 位，供 concurrency 分组，避免明文分支名出现在取消提示里）。
2. **手动**：本仓库 Actions 页 → `MP build` → Run workflow。可填 `ref` / `mode` / `robot` / `version` / `desc`。`version`、`desc` 会进 GitHub 的输入展示，尽量留空（留空则版本取 `package.json`、描述取提交信息，且只进加密日志）。手动触发的 concurrency 组名为 `mp-build-manual`。

机器人默认：`main` → 1，其它分支 → 2（可被手动输入覆盖）。

## 解密 artifact

```bash
export PASS='（与 ARTIFACT_PASSPHRASE 相同）'
openssl enc -d -aes-256-cbc -md sha256 -pbkdf2 -iter 200000 -pass env:PASS \
  -in run-N.tar.gz.enc | tar xzf -
```

解压后大致结构：`dist/mp-weixin/`（发行包）、`logs/`（私有步骤日志）、可选 `mp-preview.jpg`。

## 恢复 / 停用

- **停用自动构建**：在私有仓库禁用 `dispatch-build.yml`，或删掉 / 清空 `BUILD_RUNNER_TOKEN`；本仓库可 Disable workflow `MP build`。
- **临时切回私有仓库上传**：私有仓库的 `mp-upload.yml` 仍保留 `workflow_dispatch` 作为应急手动通道（已去掉 push 触发，避免与公开 runner 重复上传）。
- **恢复**：重新启用上述 workflow，并确保 secrets / vars 齐全；缺配置时 Preflight 只会在日志里列出缺失的**名字**然后失败。
