# memory.md — 补丁升级维护手册

> 本文件是给**下一次维护者（人或 AI 助手）** 的操作手册。
> 记录当前补丁状态、适配方法论、踩过的坑和验证清单。
> 上次更新：2026-11-26，WorkBuddy（AI 助手，模型：qwen3.8-flash），基线 v3.8.7 → **v3.8.12**。

## 当前状态

- 补丁基线：`siyuan` **v3.8.12**、`siyuan-android` **v3.8.12**
- 补丁文件：
  - `patches/siyuan.patch`（9 个文件，桌面/Docker/iOS 共用）
  - `patches/siyuan-android.patch`（仅 `app/build.gradle`）
- 已验证：对 v3.8.12 tag 与 master 分支 `git apply --check` 均通过；kernel 编译通过
- CI 通过 `git apply --3way` 应用补丁（见 `.github/workflows/release-*.yml`），克隆官方仓库对应 tag

## 补丁的 6 项功能语义（升级时必须全部保留）

1. **本地 VIP mock**：`kernel/api/setting.go` 的 `getCloudUser` handler 直接构造 `&conf.User{...}`（关键字段 `UserSiYuanSubscriptionType: 1` 终生订阅、`UserSiYuanOneTimePayStatus: 1` 功能特性已付费），`model.Conf.SetUser(user)` 只写内存不落盘，返回 `apicontract.Success(settingUserPayload(user))`
2. **禁用云端用户刷新**：`kernel/model/cloud_service.go` 的 `refreshUser()` 函数体开头 `return`（否则伪造 token 会报「获取社区用户账号失败」）
3. **禁用自动下载更新包**：
   - `kernel/model/updater.go` `skipNewVerInstallPkg()` 开头 `return true`
   - `kernel/conf/system.go` `NewSystem()` 中 `DownloadInstallPkg: false`
4. **禁用检查更新**：`kernel/api/system.go` `checkUpdate` handler 注释掉 `model.CheckUpdate`；`kernel/model/mount.go` `mountBox` 里 goroutine 中的 `CheckUpdate(true)` 注释掉
5. **默认值改动**：`kernel/conf/appearance.go` `Lang: "zh_CN"`、`CloseButtonBehavior: 1`；`kernel/conf/sync.go` `NewSync()` `Provider: ProviderS3`、`GenerateConflictDoc: true`
6. **顶栏隐藏 VIP/头衔**：`app/src/config/entryVisibility/catalog.ts` 中 `toolbarVIP` / `toolbarTitle` 两个 `node()` 加第三个参数 `true` 和 options 对象 `{ defaultVisible: () => false, customDefaultVisible: false }`

Android 补丁：注释 `apply from: rootProject.file('signings.gradle')`，在 `defaultConfig` 后插入 `signingConfigs { siyuanConfig {...} debug {...} }`（keystore 指向 `../keystore/debug.jks`，debug 密码取环境变量 `KEYSTORE_PASSWORD`），debug buildType 加 `signingConfig signingConfigs.debug` 并把 `app_name` 从 `SiYuan-Debug` 改为 `SiYuan`。

## 标准升级流程（照做即可）

```bash
# 0. 准备干净的临时工作区（不要用带本地改动的仓库！索引不一致会导致
#    "does not match index" 报错）
mkdir -p /tmp/siyuan-patch-work && cd /tmp/siyuan-patch-work
rm -rf siyuan siyuan-android

# 1. 浅克隆最新 release tag（先 git tag 或 GitHub 查最新版本号）
git clone --depth 1 --branch v3.8.12 https://github.com/siyuan-note/siyuan.git siyuan
git clone --depth 1 --branch v3.8.12 https://github.com/siyuan-note/siyuan-android.git siyuan-android

# 2. 先直接试旧补丁，看哪些 hunk 失效
cd siyuan
git apply --reject ../../Users/feng/Documents/SHANG/siyuan-note/siyuan-patch/patches/siyuan.patch
#    - 输出里 "Hunk #1 succeeded at NNN (offset ...)" = 自动应用成功
#    - 生成 .rej 的文件 = 需要手工修复
find . -name "*.rej"

# 3. 手工修复 .rej 的 hunk（对照「功能语义」一节，直接改源文件）
#    例：appearance.go 结构体字段增删导致 gofmt 对齐变化时，
#    用 perl/sed 按当前缩进改 Lang 和 CloseButtonBehavior 两行即可

# 4. 验证 Go 侧语义依赖未变（关键字段/函数被重命名会导致编译失败）
grep -n "UserSiYuanOneTimePayStatus" kernel/conf/user.go
grep -n "func settingUserPayload" kernel/api/setting_contract.go
cd kernel && go build ./api/... ./model/... ./conf/...

# 5. 从工作树重新生成补丁（关键：保留 forkSrcPrefix/forkDstPrefix 前缀格式）
cd /tmp/siyuan-patch-work/siyuan
git diff --src-prefix=forkSrcPrefix/ --dst-prefix=forkDstPrefix/ > /tmp/siyuan-patch-work/siyuan-new.patch
cd ../siyuan-android   # 同样先 apply --reject 试一遍，手工修复后：
git diff --src-prefix=forkSrcPrefix/ --dst-prefix=forkDstPrefix/ > /tmp/siyuan-patch-work/siyuan-android-new.patch

# 6. 还原工作树，验证新补丁可干净应用（--check 干跑）
cd ../siyuan && git checkout -f -- . && git apply --check --3way ../siyuan-new.patch
cd ../siyuan-android && git checkout -f -- . && git apply --check --3way ../siyuan-android-new.patch
# 顺手对 master 也测一次（CI 偶尔会构建 dev/master）：
#   git clone --depth 1 --branch master ... && git apply --check ...

# 7. 语义 diff 对比：确认新旧补丁的“新增行内容”一致（只有上下文/行号/缩进变化）
diff <(git -C /path/to/siyuan-patch show HEAD:patches/siyuan.patch | grep '^+' \
      | grep -v '^+++' | sed 's/[[:space:]]\+/ /g' | sort) \
     <(grep '^+' /tmp/siyuan-patch-work/siyuan-new.patch \
      | grep -v '^+++' | sed 's/[[:space:]]\+/ /g' | sort)

# 8. 拷回仓库并提交（commit message 沿用惯例 "fix: v3.8.12"）
cp /tmp/siyuan-patch-work/siyuan-new.patch        /path/to/siyuan-patch/patches/siyuan.patch
cp /tmp/siyuan-patch-work/siyuan-android-new.patch /path/to/siyuan-patch/patches/siyuan-android.patch
cd /path/to/siyuan-patch && git add patches && git commit -m "fix: v<X.Y.Z>"
```

## 踩坑记录（重要，别重复踩）

1. **不要在已有本地改动的 siyuan 工作副本上测补丁**。`git checkout -f <tag>` 后 `git apply --3way` 仍会报 `does not match index`（暂存区残留）。永远用全新浅克隆。
2. **CI 是浅克隆，`--3way` 经常没有 blob**，会打印 `repository lacks the necessary blob` 然后退回直接应用——所以上下文行必须精确匹配，不能指望三方合并兜底。生成补丁时基于干净工作树，`index abc1234..def5678` 短哈希是真实 blob 哈希，CI 上如果 blob 缺失同样退回直接应用，无碍。
3. **`--src-prefix` / `--dst-prefix` 必须显式指定**为 `forkSrcPrefix/` 和 `forkDstPrefix/`，否则生成的是 `a/` `b/` 前缀，与仓库既有格式不一致（`git apply` 其实两者都吃，但保持一致便于 diff 审阅）。
4. **appearance.go 是重灾区**：官方频繁往 `NewAppearanceForContainer` 返回的结构体字面量里加字段（v3.8.12 加了 `ShowFloatingStatusBar`、`Notifications`、`EntryVisibility` 等），字段名长度变化会让 gofmt 把**所有**键的冒号对齐列挪动，旧上下文全部失配。遇到此文件直接手工改两行最省事，不要硬调补丁。
5. **siyuan-android 的 build.gradle 已重构**（v3.8.9+）：官方把签名配置拆到 `signings.gradle`（仓库里只有 `signings.templates.gradle` 模板，需自行复制），且 Groovy 语法改成 `key = value`。旧补丁上下文（`versionName siyuanVersionName` 无 `=`）在新版完全对不上。注意 `release` buildType 引用了 `signingConfigs.siyuanConfig`，所以补丁必须保留我们自定义的 `siyuanConfig` 块，否则 Gradle 配置阶段直接报错。
6. **conf.User 字段类型是 float64**，mock 里写 `-1` / `1` 整数字面量没问题（Go 自动转换），但如果官方新增必填字段（非指针、零值不合法），`go build` 不会报错、运行才会出问题——升级时人工过一遍 `kernel/conf/user.go` 的字段列表和 mock 是否对齐。
7. **iOS 工作流也用 `siyuan.patch`**（`release-ios.yml` 同样 `git apply --3way`），改 kernel 补丁时不需要单独处理 iOS，但验证时如果 iOS 有额外源码改动需留意。
8. 行号偏移（`offset N lines`）不是错误，`git apply` 会自动处理，无需重新生成——只有 `patch does not apply` 才要动手。

## 验证清单（提交前逐项确认）

- [ ] `siyuan.patch` 对新 tag `git apply --check --3way` 通过
- [ ] `siyuan.patch` 对 `master`（或 `dev`）`git apply --check` 通过
- [ ] `siyuan-android.patch` 对新 tag 通过
- [ ] 打补丁后 `cd kernel && go build ./api/... ./model/... ./conf/...` 成功
- [ ] 新旧补丁「新增行内容」语义 diff 一致（第 7 步），或差异有明确理由
- [ ] `conf.User` mock 字段与 `kernel/conf/user.go` 当前定义一一对应
- [ ] catalog.ts 的 `node()` 签名（参数个数/顺序）与补丁假设一致
- [ ] commit message 格式 `fix: v<X.Y.Z>`
- [ ] 更新本文件「当前状态」与 README 顶部的基线版本号

## 快速参考

- 官方仓库：`siyuan-note/siyuan`（主分支叫 **master**，不是 main）、`siyuan-note/siyuan-android`
- 查最新版本：`git ls-remote --tags https://github.com/siyuan-note/siyuan.git | grep -v alpha | grep -v beta | tail`
- CI 应用命令：`git apply --3way patches/siyuan.patch`（工作目录 = 克隆的 siyuan 根）
- 本次修复的完整记录：git log `fix: v3.8.12`（2026-11-26）
