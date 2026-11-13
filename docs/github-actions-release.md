# GitHub Actions 桌面端发布（fork）

本 fork（`lliangworld/ZCode`）通过 GitHub Actions 编译桌面端安装包。上游的 GitLab CI（见
[desktop-ci-cd.md](./desktop-ci-cd.md)）依赖的 CI 脚本不在本仓库，本流水线直接复用仓库自带的
`pnpm bundle:desktop` 自包含打包入口，不依赖任何缺失的内网脚本。

## Remote 说明

| remote     | 地址                                   | 用途                               |
| ---------- | -------------------------------------- | ---------------------------------- |
| `origin`   | `git@github.com:lliangworld/ZCode`     | 自己的 fork，推 dev 分支与发版 tag |
| `upstream` | `https://github.com/zai-org/ZCode.git` | 同步上游源码，只 fetch 不 push     |

## 触发方式

工作流文件：[.github/workflows/release-desktop.yml](../.github/workflows/release-desktop.yml)

- **手动触发（测试构建）**：Actions 页选择 "Release Desktop" → "Run workflow" → 选 dev 分支。
  只构建并上传 Actions Artifacts（`zcode-desktop-{mac-arm64,win-x64,linux-x64}`），不发布 Release。
  产物版本不加日期后缀（如 `ZCode-3.17.1-win-x64.exe`）。
- **tag 触发（正式发版）**：推送 `v*` 格式 tag 自动构建三平台，并发布 GitHub Release（附全部安装包）。

## 版本约定（与上游区分）

fork 发版 tag 必须采用 **`v<版本>-<YYYYMMDD>`** 格式，例如 `v3.17.1-20261031`，
避免与上游的纯 `v<版本>` tag（如 `v3.17.1`）混淆。

发版步骤：

1. 把 root `package.json` 的 `version` 更新为基础版本号（可用仓库自带的 `pnpm release`
   （release-it）或手动 `npm version <version> --no-git-tag-version`）。
2. 确认 tag 打在**包含本工作流文件的提交**上（Actions 按 tag 指向的提交加载 workflow）。
3. 推送 tag：`git push origin v3.17.1-20261031`。

完整命令清单见下文「命令清单（手动执行）」一节。

CI 会在构建前校验 tag 格式（版本部分必须等于 package.json 当前版本，日期为 8 位数字），
并把 root `package.json` 的 version 改写为 `<版本>-<日期>`。该改写只发生在 CI 运行区，不提交回仓库。
产物版本经由 `packages/desktop/scripts/build-metadata.mjs`（`appVersion`）→
`electron-builder` `extraMetadata.version` 注入，安装包文件名随之带上日期：

```
ZCode-3.17.1-20261031-win-x64.exe
ZCode-3.17.1-20261031-mac-arm64.dmg / .zip
ZCode-3.17.1-20261031-linux-x64.AppImage / .deb / .rpm / .pkg.tar.zst
```

## 命令清单

从同步上游到打 tag 发版的完整命令，按顺序执行：

```bash
# 1. 基线检查（只读）：确认工作区干净、与 origin/dev 同步
node scripts/check-workspace-freshness.mjs
git status

# 2. 拉取上游更新
git fetch upstream

# 3. 查看上游新提交（无输出即已是最新，可到此为止）
git log --oneline upstream/main ^dev

# 4. main 分支快进同步（main 是上游镜像，应可 ff；失败说明已分叉，停下报告）
git checkout main
git merge --ff-only upstream/main

# 5. 合并上游到 dev（有冲突则 git merge --abort 后人工处理，不自动解决）
git checkout dev
git merge upstream/main --no-edit

# 6. 推送分支（--no-verify 跳过 pre-push 钩子；需要完整验证时删掉该参数）
git push --no-verify origin dev main

# 7. 确认版本号与 dev HEAD（tag 版本必须等于 package.json 的 version）
grep -m1 '"version"' package.json
git rev-parse HEAD

# 8. 打附注 tag：格式 v<版本>-<YYYYMMDD>，日期为当天，消息沿用惯例
git tag -a v3.18.0-20261112 -m "ZCode fork desktop release v3.18.0 (2026-11-12)"

# 9. 推送 tag 触发 CI（三平台构建 + 自动发布 GitHub Release）
git push --no-verify origin v3.18.0-20261112

# 10. 查看 CI 运行状态（gh 需显式指定 fork 仓库，否则默认解析到上游）
gh run list -R lliangworld/ZCode --workflow=release-desktop.yml --limit 3
```

- 第 8 步示例为 `v3.18.0-20261112`；实际执行时版本取 root `package.json` 的 `version`，日期取当天，否则 CI 校验失败。
- tag 必须打在包含 `.github/workflows/release-desktop.yml` 的提交上（dev 分支自 `e35f950e` 起均满足）。
- 合并冲突时 `git merge --abort` 停止并人工处理；fork 特有文件（`.github/workflows/release-desktop.yml`、`docs/github-actions-release.md`）最可能冲突。
- 第 3 步无输出表示上游没有新提交，不做任何合并和推送。

## 构建矩阵

| 平台    | runner                  | 目标            | 产物                                                                |
| ------- | ----------------------- | --------------- | ------------------------------------------------------------------- |
| macOS   | `macos-latest`（arm64） | `--mac --arm64` | `.dmg` / `.zip` + `latest-mac.yml`                                  |
| Windows | `windows-latest`        | `--win --x64`   | `.exe` + `latest.yml`                                               |
| Linux   | `ubuntu-latest`         | `--linux --x64` | `.AppImage` / `.deb` / `.rpm` / `.pkg.tar.zst` + `latest-linux.yml` |

构建命令统一为 `ZCODE_ENV=production pnpm bundle:desktop -- --os <os> --arch <arch>`，
正式产物无 `_TEST` 文件名后缀，Built-in Provider 使用 `config/provider/zcode-builtin.json`。

## 注意事项

- **fork 首次启用**：推送 dev 分支后需在仓库 Actions 标签页手动启用 workflow
  （"I understand my workflows, go ahead and enable them"）。
- **推送基线**：本 fork 没有 `origin/staging` 分支，pre-push 的受影响工程测试基线需显式指定，
  否则新分支首次推送会失败：`ZCODE_AFFECTED_TEST_BASE=origin/main git push ...`。
- **win 构建依赖 VS 2022 Build Tools**：上游 v3.18.0 起桌面打包包含 Windows 原生组件
  `zcode-runtime-host`（`scripts/windows-runtime-host`，CMake 硬编码
  `Visual Studio 17 2022` 生成器，见 `scripts/build-windows-runtime-host.mjs`）。
  `windows-latest` 镜像已升级到 VS 2026（无 VS 2022），configure 会报
  "could not find any instance of Visual Studio" 导致 win 打包失败
  （v3.18.0-20261112 发版时 win 作业即此原因；mac/linux 不受影响：该步骤仅 win 路径执行）。
  主修复：win 作业固定 `runs-on: windows-2022`（预装 VS 2022 Enterprise 17.14，
  与 pnpm、ncnn 等社区做法一致）；镜像退役时改回 `windows-latest`。
  兜底：win 作业保留 `Install Visual Studio 2022 Build Tools` 步骤——镜像没有 v17.x
  实例时用 `vs/17/release` 频道的 `vs_buildtools.exe` 安装 MSVC 工具集与
  NativeDesktop 核心组件。注意引导程序父进程派生子安装进程后立即退出（退出码 0），
  不能以父进程退出或 `--wait` 为准，必须轮询 vswhere 确认实例注册（同 pnpm 的结论）。
  不要改用 `microsoft/setup-msbuild`：该 action 只把 MSBuild.exe 加入 PATH，
  不安装 VS 也不解决 CMake 生成器发现。上游若改为自适应 VS 版本或更换构建方式，可移除该步骤。
- **pnpm hoisted 链接竞态**：本仓库 `.npmrc` 为 `node-linker=hoisted` 且有 9 个
  `patchedDependencies`，命中 pnpm 已知竞态（pnpm/pnpm#13627、#12880）：install 末期
  worker 线程把 `_tmp_*` 暂存目录 rename 到 hoisted 目标路径时可能
  `ERR_PNPM_ENOENT` 失败。官方修复在 pnpm>=11.25（#13627→11.22，#12880→11.25），
  10.x 线至 10.34.9 无 backport，升级属大版本迁移需单独排期。
  临时缓解：`Install dependencies` 步骤内最多 4 次重试——首次尝试已把 store 写满，
  重试即热 store 场景（v3.18.2-20261113 起启用）。
  注意：不要用整 job 重跑替代——`actions/setup-node` 的 pnpm store 缓存只在 job
  成功时保存，失败的 job 不写缓存，Re-run failed jobs 会再次冷安装、再次踩雷。
- **未签名产物**：本 fork 无代码签名证书，产物为未签名安装包——Windows 会弹 SmartScreen
  提示（点"仍要运行"），macOS 首次打开需右键 → 打开，或 `xattr -cr /Applications/ZCode.app`。
- **签名扩展（可选）**：如需签名，准备 OV 代码签名证书（.pfx），在仓库 secrets 配置
  `CSC_LINK` 与 `CSC_KEY_PASSWORD` 即可——electron-builder 内置签名链路会自动生效，
  不需要上游内部脚本。不要设置 `ZCODE_ENABLE_WINDOWS_SIGN=1`：该链路依赖本 fork 中不存在的
  `scripts/sign-windows.ps1`（内部 vsigntool / DigiCert token 封装），开启必失败。
- **公开仓库免费**：Actions 分钟数与产物存储对公开仓库不计量收费（含 macOS runner）。
