---
title: 逃离UnityCN
date: 2025-11-11 17:47:34
tags: [Unity, Unity CN]
---

Unity中国实在是罪大恶极，自从Unity中国变成独立公司之后，国内就没办法以正常途径下载国际版Unity了，即使使用的是国际版Hub

这里介绍一种逃离办法，需要有VPN

- 首先UnityHub默认启动时，不会读取系统代理，也就是说，即使你开了VPN，挂到了国外节点，Hub也不会走代理访问Unity服务器
- 如何判断当前UnityHub是否走了代理？
  - 退出登陆，然后点Sign In，如果弹出的登陆页面里有cn，不能用Google登陆，那就是代理没生效
- 我们要通过命令行的方式启动Unity Hub ，并且让它走代理

  - MacOS下，打开终端，执行以下命令，生成一个launchUnityHub.command文件
    ```shell
    echo '#!/bin/bash
    export HTTP_PROXY=http://127.0.0.1:7890
    export HTTPS_PROXY=http://127.0.0.1:7890
    nohup "/Applications/Unity Hub.app/Contents/MacOS/Unity Hub" &>/dev/null &' > launchUnityHub.command
    chmod +x launchUnityHub.command
    ```

  - Windows下，打开记事本，输入以下内容，保存为launchUnityHub.bat文件
    ```shell
    @echo off
    set HTTP_PROXY=http://127.0.0.1:1080
    set HTTPS_PROXY=http://127.0.0.1:1080
    start "" "C:\Program Files\Unity Hub\Unity Hub.exe"
    ```
  - 将1080设置为自己的代理端口，Clash默认是7890，Shadowsocks默认是1080
  - 双击launchUnityHub.command或launchUnityHub.bat启动Unity Hub
- 这时候你再点Sign In，就会发现可以用Google账号登陆了
- 登陆之后，你就可以正常下载国际版Unity了


