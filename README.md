<p align="center">
  <img src="logo.png" width="120" alt="bangumi2memos">
</p>

# bangumi2memos

![Python](https://img.shields.io/badge/Python-%E2%89%A53.11-3776AB)
[![Memos](https://img.shields.io/badge/Memos-%E2%89%A50.26.0-1E6D51)](https://github.com/usememos/memos)

把 Bangumi 用户「看过 / 玩过 / 读过 / 听过」且**带短评**的收藏导入 [Memos](https://github.com/usememos/memos)，
memo 为纯文字，正文含条目名、完成态文案、短评和 Bangumi 链接。

## 版本要求

| 依赖 | 版本 |
| --- | --- |
| Python | ≥ 3.11 |
| Memos | API 模式 ≥ 0.26.0；直写库 ≥ 0.22 |

## 原理

- 调 `GET /v0/users/{username}/collections?type={type}` 拉取完成态收藏
- 仅导入短评（`comment`）非空的条目，无文字则跳过
- memo 正文：`看过《阿基拉》：东京，燃烧；……` + 条目链接（默认 `https://bgm.tv/subject/5118`；
  链接域名可用 `--link-base` 换成其它镜像）
- 每条 memo 以 `uid = bgm-{subject_id}` 幂等，重复运行不产生重复 memo
- 时间用收藏的 `updated_at`（+08:00）写入 `createTime`/`created_ts`，保留原始时间
- 标签 `--tag` 默认以 `#tag` 追加到正文并同时显式传入（API: `tags`，直写库: `payload.tags`，双写确保标签生效）；
  可用 `--no-tag-in-content` 关闭正文追加，此时仅显式传入标签，正文不含 `#tag`，编辑 memo 后标签会丢失
- 列表按 `updated_at` 降序返回，状态文件记录最新 `updated_at`，下次运行提前停止处理更旧条目；
  `--full` 强制全量

## 使用方式

### 方式一：API 模式

memos >= 0.30（登录换取短期 token）：

```sh
python3 bangumi2memos.py --bangumi-username sai \
    --api http://localhost:5230 --user admin --password '你的密码'
```

 0.26.0 ≤ memos < 0.30（使用账号里的 Access Token）：

```sh
python3 bangumi2memos.py --bangumi-username sai \
    --api http://localhost:5230 --token 'AccessToken'
```

### 方式二：直写数据库

```sh
python3 bangumi2memos.py --bangumi-username sai --db ~/.memos/memos.db --user admin
```

## 同步

### cron

```sh
# 每 30 分钟同步一次
*/30 * * * * cd /path/to/bangumi2memos && python3 bangumi2memos.py --config config.toml >> sync.log 2>&1
```

### GitHub Actions

前提：memos 实例可从公网访问。使用 Cloudflare 时可出现 1010 错误，可选择配置 Security Rules 豁免

```
(http.host eq "你的公网域名" and starts_with(http.request.uri.path, "/api/v1/memos"))
```

的 Browser Integrity Check 等方法使 Action 可访问。

fork 本仓库，参考 [sync.yml](.github/workflows/sync.yml) 每 6 小时在 GitHub runner 上自动跑一次 API 模式同步。

配置仓库 Secrets / Variables（Settings → Secrets and variables → Actions）：

| 名称 | 类型 | 说明 |
| --- | --- | --- |
| `BANGUMI_USERNAME` | Secret | Bangumi 用户名（必填） |
| `MEMOS_API` | Secret | memos 地址，如 `https://memos.example.com`（必填） |
| `MEMOS_PASSWORD` | Secret | memos 密码（memos ≥ 0.30） |
| `MEMOS_USER` | Secret | memos 登录用户名（配合密码） |
| `MEMOS_TOKEN` | Secret | 或 memos < 0.30 的 Access Token（替代密码） |
| `MEMOS_VISIBILITY` | Secret / Variable | memo 可见性：`private` / `protected` / `public`（可选，默认 `private`；可用 Variables，更语义化） |
| `MEMOS_TAG` | Secret / Variable | 附加标签（可选，空 = 不加；如 `bangumi` 则默认正文追加 `#bangumi`，`--no-tag-in-content` 可关闭） |

`MEMOS_VISIBILITY` / `MEMOS_TAG` 同时对定时任务（`schedule`）与手动触发（`workflow_dispatch`）生效（Secrets 优先于 Variables，未配置则默认 `private` / 不加标签）。

可在首次本地全量导入后，用
**workflow_dispatch** 手动触发一次，在 `watermark` 输入框填本地 `state.json` 的
`last_updated_ts`（epoch 秒），避免重复全量初始化。

## 卸载

删除 uid 以 `bgm-` 开头（即本工具导入）的 memo，并重置增量状态文件，增加 `--delete` 参数即可，例：

```sh
python3 bangumi2memos.py --delete --api http://localhost:5230 --user admin --password '你的密码'
```

## 配置文件

默认读取当前目录 `config.toml`，也可用 `--config` 指定其它路径；

命令行参数会覆盖配置文件。参考 `config.example.toml`。

## 说明与限制

- 短评需为**公开收藏**（API 无鉴权时读不到私有收藏）
- Bangumi 存在 bug：修改评分/短评可能不更新 `updated_at`，此类「旧条目补短评」增量会漏，
  可定期用 `--full` 补扫
- 标签默认同时写入正文与显式标签字段（双写确保标签生效），Memos 前端编辑时会按正文重新提取标签；若用 `--no-tag-in-content` 关闭正文追加，仅显式传入标签（API: `tags`，直写库: `payload.tags`），再次编辑后会丢失
- 需设置规范的 User-Agent（默认值见 `config.example.toml`，可覆盖）

## 许可证

[GNU General Public License v3.0 or later](LICENSE)（GPL-3.0-or-later）
