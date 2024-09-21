# siyuan patch [思源笔记](https://github.com/siyuan-note/siyuan)本地VIP

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
