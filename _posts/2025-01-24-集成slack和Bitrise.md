---
layout:     post
title:      Android Integrate Bitrise and Slack
subtitle:   海外应用的CI/CD开发
date:       2025-01-24
author:     River
header-img: img/slack/post-bg-coffee.jpeg
catalog: true
tags:
    - CI/CD
    - Bitrise
    - Android
--- 

# 前言

两年多没写博客了，最近到了做了海外项目的公司；记录一下海外项目持续集成的技术；
因为日常工作沟通用的比较多的是Slack，国内是钉钉；
本文记录怎么通过，在Slack发送一个打包命令就能构建apk的过程。


![开发完成后，实际使用效果图](/img/slack/slack-build-succeed.jpg)


### 1、Bitrise构建apk时，通知到slack的打包群。

登录Bitrise后台，找老板申请管理员权限；选择一个workflow，然后添加一个message的step步骤。

![选择workflow](/img/slack/bitrise_1_1.jpg)
![添加一个message消息](/img/slack/bitrise_1_2.jpg)



#### 1.2、填写send a slack message的step
分别是Slack Webhook URL，和Target Slack channel

下图是Target Slack channel的填写
![填写Target Slack channel-1](/img/slack/bitrise_1_3_1.jpg)
![填写Target Slack channel-2](/img/slack/bitrise_1_3_2.jpg)

下图是Slack Webhook URL的填写, 需要找管理员开通Collaborate
![填写Target Slack channel-2](/img/slack/bitrise_1_4_1.jpg)
![填写Target Slack channel-2](/img/slack/bitrise_1_4_2.jpg)
![填写Target Slack channel-2](/img/slack/bitrise_1_4_3.jpg)

点击新增Webhook URL的copy按钮，去bitrise配置一个系统变量；key需要自己命名，值就是刚刚copy的Webhook URL；
![填写Target Slack channel-2](/img/slack/bitrise_1_4_4.jpg)
![填写Target Slack channel-2](/img/slack/bitrise_1_4_5.jpg)

### 2、Slack发送消息触发Bitrise的workflow

核心是创建Create New Command，填写command 和 Request URL两个参数。

#### 2.1、创建Create New Command

打开slack配置后台，左侧菜单打开Slack Commands，点击Create New Command。
![创建Create New Command-2-1](/img/slack/bitrise_2_1.jpg)


#### 2.2、填写command 和 Request URL两个参数
command也是自定义的，只需要达意就行没有固定写法，一般Android打包可以写成 /build-android
![创建Create New Command-2-2](/img/slack/bitrise_2_2.jpg)

Request URL还需要到Bitrise后台，在Setting中寻找
![创建Create New Command-2-3](/img/slack/bitrise_2_3.jpg)
![创建Create New Command-2-4](/img/slack/bitrise_2_4.jpg)
![创建Create New Command-2-5](/img/slack/bitrise_2_5.jpg)


### 3、构建时候，获取版本号

#### 3.1、在bitrise的steps中添加一个script的step，并编辑添加shell脚本。

![获取版本号-3-1](/img/slack/bitrise_3_1.jpg)

#### 3.1、shell脚本的编写

核心是使用envman命令加一个临时系统变量

```
#!/usr/bin/env bash
# fail if any commands fails
set -e
# make pipelines' return status equal the last command to exit with a non-zero status, or zero if all commands exit successfully
set -o pipefail
# debug log
set -x

# write your script here
echo "Hello World!"

current_dir=$(pwd)
echo "current_dir=${current_dir}"

# 列出当前目录下的文件和文件夹，并通过grep过滤出文件夹
folders=$(ls -l | grep '^d' | awk '{print $9}')

# 打印文件夹列表
echo "当前目录下的文件夹有:"
echo "$folders"

# or run a script from your repository, like:
# bash ./path/to/script.sh
# not just bash, e.g.:
# ruby ./path/to/script.rb

GLOBAL_APP_VERSION=$(grep -m 1 -w "var GLOBAL_APP_VERSION ="  Coins/coins/build.gradle | tr -cd "[0-9]")
echo "GLOBAL_APP_VERSION=${GLOBAL_APP_VERSION}"

MAJOR_FEATURE_VERSION=$(grep -m 1 -w "var MAJOR_FEATURE_VERSION =" Coins/coins/build.gradle | tr -cd "[0-9]")
echo "MAJOR_FEATURE_VERSION=${MAJOR_FEATURE_VERSION}"

MINOR_FEATURE_VERSION=$(grep -m 1 -w "var MINOR_FEATURE_VERSION =" Coins/coins/build.gradle | tr -cd "[0-9]")
echo "MINOR_FEATURE_VERSION=${MINOR_FEATURE_VERSION}"

BUGFIX_VERSION=$(grep -m 1 -w "var BUGFIX_VERSION =" Coins/coins/build.gradle | tr -cd "[0-9]")
echo "BUGFIX_VERSION=${BUGFIX_VERSION}"

appVersion="${GLOBAL_APP_VERSION}.${MAJOR_FEATURE_VERSION}.${MINOR_FEATURE_VERSION}.${BUGFIX_VERSION}"

envman add --key Coins_Android_App_Version --value "$appVersion"

echo "appVersion=${appVersion}"
```

### 4、收工测试，见标题截图

### 总结 
今年领悟了一句话，做对事情比勤奋重要的多，慢就是快；





