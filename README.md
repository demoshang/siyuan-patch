# siyuan patch

## 如何下载 桌面端(Windows / Mac / Linux) 和 Android 应用

1. 进入 [Actions页面 https://github.com/demoshang/siyuan-patch/actions/workflows/build.yml](https://github.com/demoshang/siyuan-patch/actions/workflows/build.yml)

2. 选择最近一个构建成功的 workflow 任务
  <img width="1666" alt="image" src="https://github.com/demoshang/siyuan-patch/assets/26966709/67e7dfe0-94d9-4b37-8115-88d63118bb0a">

3. 滚动到页面最底部
4. 选择对应平台下载 (需要登录 github 账户)
  <img width="1553" alt="image" src="https://github.com/demoshang/siyuan-patch/assets/26966709/0c780d32-bd4a-4790-9d25-ebbac9713b62">

## Docker 镜像

<https://hub.docker.com/r/demoshang/siyuan/tags>

## 没有 ios 应用吗?

**没有**,  因为需要花钱,  不花钱的只能使用7天, 所以就不提供了

可以使用上面 Docker 镜像的 网页版

## 没有最新版吗?

追求最新请按照 `如何自己构建` 教程 自行构建

## 如何自己构建

1. fork 本项目到你自己项目
2. 如果需要构建 Electron 客户端(Windows/Mac/Linux), 无需配置环境变量
3. 如果需要构建 Android 客户端, 请执行以下步骤
    - 按照文章[生成上传密钥和密钥库](https://developer.android.com/studio/publish/app-signing?hl=zh-cn#generate-key)
      > 注意秘钥(Key)的 `Alias` 设置成 `debug`
      ![](https://user-images.githubusercontent.com/26966709/275674510-3fe33b8f-5aa0-4eb0-bbb6-bfdd22c1fab2.png)
    - 秘钥转成base64编码

        ```bash
        openssl base64 < your_signing_keystore.jks | tr -d '\n' | tee your_signing_keystore_base64_encoded.txt
        ```

    - 进入该项目的 `settings`-`Secrets and variables`-`Actions`, 点击 `New repository secret` 添加环境变量
    - `KEYSTORE` 上面`your_signing_keystore_base64_encoded.txt` 里面的内容
    - `KEYSTORE_PASSWORD` 生成上传密钥时填写的密码

4. 如果需要构建 Docker 镜像
    - 进入 `https://hub.docker.com/settings/security`, 点击 `New Access Token` 添加 token, 记住并保存该token
    - 进入该项目的 `settings`-`Secrets and variables`-`Actions`, 点击 `New repository secret` 添加环境变量
    - `DOCKER_HUB_USER` 你的Docker账户名
    - `DOCKER_HUB_PWD` 刚才保存的token
5. 进入 Actions 页面, 选择对应的 workflow, 点击运行

    - 构建 Android 和  桌面端(Windows / Mac / Linux)
      <img width="1658" alt="image" src="https://github.com/demoshang/siyuan-patch/assets/26966709/72aa5f0d-b6e3-4a0f-a98d-6c7916d9124c">
    - 构建 Docker 镜像
      <img width="1666" alt="image" src="https://github.com/demoshang/siyuan-patch/assets/26966709/675b47c6-94ed-41b8-97fb-936452cfb4cd">
