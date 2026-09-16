# Release 构建说明（macOS）

本仓库（fork of Hammerspoon）支持：**推送 `v*` 标签 → GitHub Actions 自动构建 macOS 版 → 发布 GitHub Release**（带 `.app` zip / dSYM / docs）。工作流定义见 [`.github/workflows/release-build.yml`](../.github/workflows/release-build.yml)。

## 1. 触发方式

```bash
git tag -a v1.0.0 -m "v1.0.0"
git push origin v1.0.0
```

推送带 `v` 前缀的标签后，`Release Build (macOS)` 工作流会自动运行并在构建完成后把产物发布到该 tag 对应的 GitHub Release。仓库 `Actions` 页面可以看到进度。

> 首次使用前请确认 fork 仓库的 Actions 已启用（Settings → Actions → General → Allow all actions）。

## 2. 工作流做了什么

1. 在 `macos-15` runner 上安装 Xcode 16.1.0；
2. `./scripts/github-ci-pre.sh`：安装 brew / Python 依赖（coreutils、cocoapods、xcbeautify 等）；
3. 生成占位的 Sentry token 文件（Release 配置构建的断言只要求文件存在；没有 `-u` 参数时不会真正使用）；
4. **构建 Release**：`./scripts/build.sh build -s Release -c Release`，并设置环境变量 `CODE_SIGN_IDENTITY=-`（ad-hoc 签名，不需要 Apple 开发者证书）；
5. 打应用 zip、生成 API 文档、`./scripts/build.sh archive` 汇总产物；
6. `softprops/action-gh-release` 把以下产物挂到 tag 的 Release 上：

   - `Hammerspoon.app-<版本>.zip` —— 可安装的 macOS 应用
   - `Hammerspoon-dSYM-<版本>.zip` —— 调试符号
   - `docs/<版本>-docs.zip` —— API 文档
   - Release 正文由 GitHub 自动生成（`generate_release_notes`）

## 3. 与上游 `new_tag.yml` 的关系

官方仓库的 `new_tag.yml` 同样会在推 tag 时运行（它匹配所有 tag）：先生成 changelog 并创建（仅文字的）Release。两个工作流不冲突：

- `new_tag.yml` 先创建 Release 并写入 changelog 正文；
- `release-build.yml` 构建完成后把二进制产物挂到同一个 Release 上。

如果希望 `v*` 标签完全由我们自己的工作流接管（不带 milestone/changelog 逻辑），可以把 `new_tag.yml` 的触发改成：

```yaml
on:
  push:
    tags:
      - '*'
      - '!v*'
```

## 4. 限制（无 Apple 开发者证书）

- **ad-hoc 签名**：没有 Developer ID 证书，用 `CODE_SIGN_IDENTITY=-` 构建。产物未被 Gatekeeper 信任，**首次打开需右键 → 打开**（或 `xattr -dr com.apple.quarantine Hammerspoon.app`）。
- **未公证**：跳过 notarization（需要 Apple ID + 专用密码）。
- **不更新 Sparkle appcast**：`update_appcast` 等发布后动作需要官网仓库和签名材料，已跳过；如需做软件自动更新，需要另行配置。
- 以后若拿到 p12 证书 + 公证凭据，可参照官方 `ci_nightly.yml` 的 `keychain-prep` / `notarize` 流程补齐签名与公证。

## 5. 本地手动构建（与 CI 等价）

```bash
cd vendored/hammerspoon  # 仓库根目录
export CODE_SIGN_IDENTITY=-
./scripts/build.sh installdeps   # 首次：装依赖
./scripts/build.sh build  -s Release -c Release
./scripts/build.sh docs
./scripts/build.sh archive
```

产物在 `../archive/<版本>/` 下。