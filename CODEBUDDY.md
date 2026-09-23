# CODEBUDDY.md This file provides guidance to CodeBuddy when working with code in this repository.

## 仓库定位

这是一个 **Scoop 软件源（bucket）**，托管在 Gitee（`https://gitee.com/pine1212/scoop-cn.git`），本地已注册为 bucket 名称 `scoop-cn`。仓库没有编译产物或业务代码：核心资产是 `bucket/*.json` 应用清单（manifest），其余是运维脚本与非清单资源。

## 常用命令

### 本地挂载本 bucket
`scoop bucket add scoop-cn https://gitee.com/pine1212/scoop-cn.git` —— 安装/调试清单前先挂载。`scoop install scoop-cn/<app>` 安装某个应用，`scoop update <app>` 更新。

### 运行测试（CI 等价）
`.\bin\test.ps1` —— 运行全量 Pester 测试，需已安装 Scoop 且 Pester ≤ 4.99；脚本自动把 `SCOOP_HOME` 解析为 `scoop prefix scoop`，最后以 `FailedCount` 作为退出码。CI 中改为手动设置 `$env:SCOOP_HOME` 指向 checkout 的 `scoop_core`。

### 运行单个测试文件
`Invoke-Pester -Path .\Scoop-Bucket.Tests.ps1 -PassThru` —— 只跑仓库根测试入口。该文件在加载时 dot-source Scoop 核心的 `Import-Bucket-Tests.ps1`，由它为每个清单动态生成 Describe 块，因此不要直接 `pwsh .\Scoop-Bucket.Tests.ps1`。

### 版本检查与自动更新
`.\bin\checkver.ps1` 扫描全部清单；`.\bin\checkver.ps1 <app> -u` 只检查并就地更新单个应用的 version/url/hash。`.\bin\missing-checkver.ps1` 列出缺少 `checkver` 的清单。

### 链接与格式检查
`.\bin\checkurls.ps1` 校验所有下载 URL 可达性；`.\bin\formatjson.ps1` 按 Scoop 规范重排/缩进 `bucket/` 下 JSON（4 空格、UTF-8），提交前建议先跑一次。

### 自动提 PR
`.\bin\auto-pr.ps1 -upstream <owner/repo:branch>` —— 把本地更新批量推送为上游 PR，默认 upstream 为 `ScoopInstaller/Main:master`。

### 搜索应用是否已存在于公开源
`.\bin\scoopsearch.ps1 -appName <name>` —— 调用 `https://scoop.zhoujin7.com/search` 查询应用是否已在其他 bucket 收录，新增清单前用它避免重复。

## 架构与结构

### 总体分层

仓库由三块互不依赖的内容组成：

1. **Scoop 源本体**：`bucket/` + `bin/` + `Scoop-Bucket.Tests.ps1` + `.github/`。这是上游 ScoopInstaller 模板桶的完整副本，决定了仓库的运行方式。
2. **`bin/` 是薄封装层，不含实际逻辑**。除 `scoopsearch.ps1` 外，每个脚本都做同一件事：把 `SCOOP_HOME` 解析为 `Resolve-Path (scoop prefix scoop)`，然后用 `Invoke-Expression` 调用 Scoop 安装目录里的同名脚本（`$env:SCOOP_HOME/bin/xxx.ps1`），并固定传入 `-dir "<repo>/bucket"`。因此这些工具的行为完全由已安装的 Scoop 版本决定，**不要在此处实现功能**，参数直接透传给上游脚本。要理解某个命令支持哪些 flag，需查看本地 Scoop 的 `bin/` 而非本仓库。

3. **非清单资源（与 Scoop 无关，纯附带收藏）**：`lx_music/`、`myconf.json`、`channels.m3u`、`myiptv.txt`。这些文件不被任何清单引用，也不参与测试，属于第三方影音配置/列表的存放区，修改它们不会影响 CI。

### 清单约定（`bucket/*.json`）

每个文件就是一个应用的安装定义，命名即应用名。仓库内常见写法：

- `version` + `description` + `homepage` + `license`（字符串或 `{identifier, url}` 对象）。
- 单文件应用用顶层 `url`/`hash`；分架构用 `architecture.64bit|32bit|arm64`，各自可带 `url`、`hash`、`pre_install`。
- `checkver`：GitHub 项目写 `"github"`（或 `{ "github": "<repo url>" }`）；非 GitHub 用 `{ "re": "<正则，含 (?<version>...)>" }`，如 `chfs.json`。
- `autoupdate`：url 模板中用 `$version`；哈希可从发布元数据提取，见 `zyplayer.json` 的 `hash.url = "$baseurl/latest.yml"` + `hash.regex = "sha512:\s+$base64"`（`$baseurl`、`$base64` 是 Scoop 内置变量）。
- 安装副作用：`installer.script` / `uninstaller.script`（如 zyplayer 用 Junction 把 `%APPDATA%` 指到 `$persist_dir`）、`pre_install`（如 chfs 重命名 exe、zyplayer 用 `Expand-7zipArchive` 解 NSIS 内层 7z）、`persist`、`extract_dir`、`bin`（可含参数）、`shortcuts`（`[目标, 快捷方式名]` 数组）。
- hash 支持 `sha512:` 前缀（Scoop 0.5.x），旧清单仍用裸 sha256。

新增清单时应尽量带 `checkver` + `autoupdate`，否则 `missing-checkver.ps1` 与 Excavator 无法自动跟随上游（例如 `aida64.json` 目前就没有）。

### 测试是如何工作的

`Scoop-Bucket.Tests.ps1` 只有两行：设置 `SCOOP_HOME` 后 dot-source `$env:SCOOP_HOME\test\Import-Bucket-Tests.ps1`。真正的断言（JSON 合法性、必填字段、`checkver`/`autoupdate` 自洽、URL 与 hash 匹配等）全部来自 Scoop 核心，会遍历 `bucket/` 下每个清单动态生成用例。所以**本仓库自身几乎不含测试代码**，测试失败通常意味着清单写错，或本地 Scoop 版本与 CI（checkout `ScoopInstaller/Scoop` 主干 + `SCOOP_BRANCH: develop`）不一致。

### CI / 自动化（`.github/`）

- `ci.yml`：push/PR/手动触发，`windows-latest`，`powershell` 与 `pwsh` 双 shell 矩阵；checkout 本桶到 `my_bucket`、Scoop 核心到 `scoop_core`，缓存 `BuildHelpers`+`Pester`，然后 `$env:SCOOP_HOME=...; .\my_bucket\bin\test.ps1`。
- `update.yml`：**每日 01:00 UTC（北京时间 09:00）自动检查更新**。checkout 本桶与 Scoop 核心后执行 `.\bin\checkver.ps1 -u`（可 `-f` 强制重抓 hash）、`.\bin\formatjson.ps1`，把 `bucket/` 的改动推到 `auto-update/<时间戳>` 分支并开 PR（`mode: direct` 时直接推回默认分支）。附带把缺少 `checkver` 的清单写进 job summary。手动触发支持 `app`（只更新指定清单）、`force`、`mode` 三个输入。它用 `SCOOP_GH_TOKEN` 避免 GitHub API 匿名限流，并可选 `BOT_TOKEN`（PAT）；未配置 PAT 时，`GITHUB_TOKEN` 产生的推送/PR **不会**再触发其他工作流。
- `excavator.yml`：ScoopInstaller 官方 Excavator，目前只保留 `workflow_dispatch`，定时任务已交给 `update.yml`，避免两个更新器同时改写 `bucket/`。
- `pull_request.yml`：PR 打开或评论 `/verify` 时运行 ScoopInstaller 的 PR 校验与自动 hash 修复。
- `issues.yml`：issue 打开或打上 `verify` 标签时处理，可自动修复 hash 错误。
- `lint-pr.yml`：校验 PR 标题格式。
- `dependabot.yml`：每周检查 GitHub Actions 版本，commit 前缀 `(chore)`；`CODEOWNERS` 把 `.github/workflows/` 指派给 `ScoopInstaller/maintainers`。

### 提交与 PR 规范

PR 标题必须遵循 `<manifest-name[@version]|chore>: <摘要>`，否则 `lint-pr.yml` 会失败；`.github/pull_request_template.md` 中已给出该清单项。提交信息实践中多为简短的 `update` / `add <app>`。

修改 `.github/workflows/` 需 ScoopInstaller maintainers 评审；第三方 action 一律用 commit SHA 固定版本（现有文件已如此）。
