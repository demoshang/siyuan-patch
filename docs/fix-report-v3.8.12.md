# 补丁适配报告：v3.8.7 → v3.8.12

- **日期**：2026-11-26
- **执行人**：**WorkBuddy**（AI 助手，模型：qwen3.8-flash）
- **对应提交**：`fix: v3.8.12`
- **影响文件**：`patches/siyuan.patch`、`patches/siyuan-android.patch`

## 1. 背景

CI 工作流（`release-pc.yml` / `release-android.yml` / `release-docker.yml` / `release-ios.yml`）在构建新版思源时，
执行 `git apply --3way patches/*.patch` 失败，报 `patch does not apply`，导致 v3.8.12 无法出包。

## 2. 失效原因分析

### 2.1 `patches/siyuan.patch`（9 个文件中 1 个失效）

| 文件 | 状态 | 原因 |
| --- | --- | --- |
| `kernel/conf/appearance.go` | ❌ 失效 | 官方在 `NewAppearanceForContainer` 返回的结构体字面量中新增了 `ShowFloatingDockBar`、`ShowFloatingStatusBar`、`Notifications`、`EntryVisibility` 等字段。字段名变长后 gofmt 将全部键的冒号对齐列右移，旧补丁的上下文行（如 `Lang:                "en",`）与新源码（`Lang:                  "en",`）空格数不一致，无法匹配 |
| 其余 8 个文件 | ✅ 仅行号漂移 | `git apply` 按 offset 自动应用成功（updater.go +81、setting.go -236 等） |

另外，CI 使用浅克隆（`git clone --branch <tag>`），补丁中记录的 blob 哈希在对象库中不存在，
`--3way` 打印 `repository lacks the necessary blob` 后退化为直接应用——即上下文必须逐字符匹配，没有三方合并兜底。

### 2.2 `patches/siyuan-android.patch`（唯一文件失效）

siyuan-android 仓库在 v3.8.9 前后重构了 `app/build.gradle`：

1. 签名配置从 `build.gradle` 拆出到 `signings.gradle`（仓库中只保留模板 `signings.templates.gradle`，实际文件需构建者自行复制）；
2. Groovy 语法整体改为 `key = value` 赋值风格（`versionName = rootProject.siyuanVersionName`、`buildConfig = true` 等）；
3. `release` buildType 引用 `signingConfigs.siyuanConfig`，该配置现在由 `signings.gradle` 提供。

旧补丁的 hunk 上下文（无 `=` 的旧语法、原 `signingConfigs` 位置）完全对不上新文件。

## 3. 修复方法

1. 在 `/tmp/siyuan-patch-work` 全新浅克隆 `siyuan@v3.8.12` 与 `siyuan-android@v3.8.12`（干净工作树，避免索引残留导致 `does not match index`）；
2. 对干净树执行 `git apply --reject` 应用旧补丁：8 个文件自动成功，`appearance.go` 生成 `.rej`；
3. 手工修复 `appearance.go`：按新缩进将 `Lang` 改为 `"zh_CN"`、`CloseButtonBehavior` 改为 `1`；
4. Android 侧按旧补丁语义在新版 `build.gradle` 上重做三处修改（注释 `signings.gradle` 引入、插入 `signingConfigs` 块、debug buildType 加签名并改名 `SiYuan`），**新增内容与旧补丁逐行一致**；
5. 用 `git diff --src-prefix=forkSrcPrefix/ --dst-prefix=forkDstPrefix/` 从工作树重新生成两个补丁文件，保持仓库既有的前缀格式约定；
6. 还原工作树（`git checkout -f -- .`）后对生成的新补丁做 `git apply --check --3way` 干跑验证。

## 4. 验证结果

| 验证项 | 结果 |
| --- | --- |
| 新 `siyuan.patch` 应用于 v3.8.12 | ✅ 9 个文件全部干净应用 |
| 新 `siyuan.patch` 应用于最新 master | ✅ `git apply --check` 通过 |
| 新 `siyuan-android.patch` 应用于 v3.8.12 | ✅ 干净应用 |
| 打补丁后 kernel 编译（`go build ./api/... ./model/... ./conf/...`） | ✅ 通过（确认 `conf.User` 字段、`settingUserPayload`、`node()` 签名等依赖未变） |
| 新旧补丁「新增行」语义对比 | ✅ 一致（仅 appearance.go 两行因 gofmt 对齐列变化而空格数不同） |
| 6 项功能语义 | ✅ 全部保留 |

## 5. 后续建议

- 下一次思源发版时按 [memory.md](./memory.md) 的「标准升级流程」重跑一遍即可，其中「踩坑记录」收录了本次遇到的全部问题；
- `appearance.go` 与 siyuan-android `build.gradle` 是历次升级的固定失效点，可优先检查；
- 修复后需 push 到 origin/main，cron 工作流（每周二/周五 20 点）才会取到最新补丁。
