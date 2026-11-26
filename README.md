# siyuan patch [思源笔记](https://github.com/siyuan-note/siyuan)本地VIP

> 当前补丁基线：**v3.8.12**（2026-11-26 由 **WorkBuddy**（模型：qwen3.8-flash）适配，详见 [补丁维护指南](./memory.md)）

## 修改了哪些内容

1. 本地VIP(即官方的 [`功能特性`](https://b3log.org/siyuan/pricing.html)), 可以使用
    - `第三方 S3 数据同步和备份`
    - `第三方 WebDAV 数据同步和备份（不支持坚果云）`
    - `搜索资源文件内容`
    - `导出 PDF/图片 添加水印`
2. **禁用** `自动下载更新安装包`
3. 默认关闭 `Google Analytics`
4. 默认关闭 `自动上传报错信息和诊断数据`
5. 默认关闭 `顶栏显示头衔` `顶栏显示 VIP`
6. 默认*开启* `点击关闭按钮后缩小到托盘中`

## 补丁如何工作

本仓库不 fork 思源源码，只保存两个 `git diff` 补丁，构建时（GitHub Actions）克隆官方对应 tag 后应用：

- `patches/siyuan.patch` → 应用到 [siyuan-note/siyuan](https://github.com/siyuan-note/siyuan)（Electron 桌面端 / Docker / iOS 共用）
- `patches/siyuan-android.patch` → 应用到 [siyuan-note/siyuan-android](https://github.com/siyuan-note/siyuan-android)（Android 壳工程）

补丁路径带有 `forkSrcPrefix/` / `forkDstPrefix/` 前缀（历史遗留格式），用 `git apply` 应用即可，前缀不影响匹配。

核心改动位置（升级时需重点核对）：

| 文件 | 改动 |
| --- | --- |
| `kernel/api/setting.go` | `getCloudUser` 返回内存 mock 的 VIP 用户（不落盘） |
| `kernel/model/cloud_service.go` | `refreshUser()` 直接 return，避免伪造 token 触发报错 |
| `kernel/model/updater.go` | `skipNewVerInstallPkg()` 直接 return true，禁用自动下载安装包 |
| `kernel/api/system.go` / `kernel/model/mount.go` | 注释掉 `CheckUpdate` 调用 |
| `kernel/conf/appearance.go` | 默认 `zh_CN`、关闭按钮最小化到托盘 |
| `kernel/conf/sync.go` / `kernel/conf/system.go` | 同步默认 S3、`DownloadInstallPkg: false` |
| `app/src/config/entryVisibility/catalog.ts` | 顶栏 VIP / 头衔默认隐藏 |
| `app/build.gradle`(android) | 注释 `signings.gradle`，内置 debug 签名配置 |

## 本次 v3.8.12 适配说明（2026-11-26）

**问题**：官方发布 v3.8.12 后 CI 构建失败，`git apply --3way` 报 `patch does not apply`。

**原因**：

1. `kernel/conf/appearance.go` 的 `NewAppearanceForContainer` 新增了 `ShowFloatingDockBar`、`Notifications` 等字段，gofmt 对齐宽度变化，旧补丁上下文匹配失败；浅克隆导致 blob 缺失，`--3way` 也无法回退三方合并。
2. siyuan-android 仓库重构了 `app/build.gradle`：`signingConfigs` 移到了 `signings.gradle`（仓库中只剩 `signings.templates.gradle`），且语法改为 `key = value` 赋值风格，旧补丁 hunk 上下文完全失效。
3. 其余文件仅行号漂移，`git apply` 可自动偏移，无需改动。

**修复方法**（完整流程见 [memory.md](./memory.md)）：克隆干净的 v3.8.12 源码 → 应用旧补丁（能自动应用的部分）→ 手工修复失效的 hunk → 用 `git diff --src-prefix=forkSrcPrefix/ --dst-prefix=forkDstPrefix/` 重新生成补丁 → `git apply --check` 与 `go build` 双重验证。

**验证结果**：新补丁对 v3.8.12 tag 与最新 master 分支均可干净应用；打补丁后 kernel `go build ./api/... ./model/... ./conf/...` 编译通过；6 项功能语义全部保留。

## 关于维护

本项目由 **WorkBuddy**（AI 助手，当前模型：qwen3.8-flash）维护：每次思源发新版后，由 WorkBuddy 按 [memory.md](./memory.md) 中记录的流程重新适配补丁并验证提交。若你使用 AI 助手维护本仓库，请直接照做该指南即可。

## 如何下载 客户端(Windows / Mac / Linux / Android)

1. 进入 [Release页面 https://github.com/demoshang/siyuan-patch/releases](https://github.com/demoshang/siyuan-patch/releases)

2. 选择对应平台下载  

![image](https://github.com/demoshang/siyuan-patch/assets/26966709/d81f9e8f-027c-4ae6-ba67-51bca5b62bd5)

## Docker 镜像

<https://hub.docker.com/r/demoshang/siyuan/tags>

## 关于 ios 应用

- 如果手机支持 [巨魔TrollStore](https://github.com/opa334/TrollStore), 可以在 [Release](https://github.com/demoshang/siyuan-patch/releases) 下载 `ipa` 文件(未签名)安装

- 否则, 不支持IOS, 因为签名证书需要花钱, 不花钱的只能使用7天, 所以就不提供了

## 没有最新版吗?

追求最新请按照 `如何自己构建` 教程 自行构建  
  
默认每周二和每周五20点尝试获取最新版本构建

## 如何自己构建

1. fork 本项目到你自己项目
2. 如果需要构建 Electron 客户端(Windows/Mac/Linux), 无需配置环境变量
3. 如果需要构建 Android 客户端, 请执行以下步骤
    1. 按照文章[生成上传密钥和密钥库](https://developer.android.com/studio/publish/app-signing?hl=zh-cn#generate-key)
        > 注意秘钥(Key)的 `Alias` 设置成 `debug`  

        ![generate-key](https://user-images.githubusercontent.com/26966709/275674510-3fe33b8f-5aa0-4eb0-bbb6-bfdd22c1fab2.png)  

    2. 秘钥转成base64编码

        ```bash
        openssl base64 < your_signing_keystore.jks | tr -d '\n' | tee your_signing_keystore_base64_encoded.txt
        ```

    3. 进入该项目的 `settings`-`Secrets and variables`-`Actions`, 点击 `New repository secret` 添加环境变量
    4. `KEYSTORE` 上面`your_signing_keystore_base64_encoded.txt` 里面的内容
    5. `KEYSTORE_PASSWORD` 生成上传密钥时填写的密码

4. 如果需要构建 Docker 镜像
    1. 进入 `https://hub.docker.com/settings/security`, 点击 `New Access Token` 添加 token, 记住并保存该token
    2. 进入该项目的 `settings`-`Secrets and variables`-`Actions`, 点击 `New repository secret` 添加环境变量
    3. `DOCKER_HUB_USER` 你的Docker账户名
    4. `DOCKER_HUB_PWD` 刚才保存的token

5. 按如下操作点击, 等待10分钟左右进入 `Release` 页面查看  
*(如果已经存在最新版本了, 可以删除 `Release` 页面的版本后重新再点击)*  

    ![release-cron](https://github.com/demoshang/siyuan-patch/assets/26966709/d139ff11-b4a8-46ff-a532-394fddf27c54)

## Mac 出现 `“SiYuan.app”已损坏，无法打开。`  

![image](https://github.com/demoshang/siyuan-patch/assets/26966709/b876218f-8184-4b2b-877f-a7a3fa92f2d3)

下载[修复『已损坏，无法打开』.zip](https://github.com/demoshang/siyuan-patch/files/14783846/default.zip), 解压后双击运行, 然后重新打开 `SiYuan.app` 即可
