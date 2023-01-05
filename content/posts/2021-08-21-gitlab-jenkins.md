---
layout: article
title: 在Jenkins中访问Gitlab
date: 2021-08-21 12:38:38
author: 任怀林
tags:
- linux
- gitlab
- jenkins
thumb: linux.png
comment: true
isCJKLanguage: true
---

## 1.1 Gitlab Credentials

### 生成访问Gitlab所需Token

首先要在Gitlab里创建一个名为`Jenkins`的账号，Jenkins系统使用此账号访问Gitlab。创建好账号后，把这个账号添加到Jenkins要访问的Gitlab项目中，并赋予只读的权限。

以`Jenkins`的身份登录到源Gitlab，点击右上角的头像图标，在下拉菜单中选择"Preferences",

在打开页面的侧边栏里点击“Access Tokens”打开“Personal Access Tokens”页面。

输入一个名字，在Scopes下页面为新token选择权限。然后点击“Create personal access token”按钮，生成token。

**注意：生成的token只显示一次请复制下来，保存好。**

### 创建Jenkins Credentials

在Jenkins左侧菜单栏里选择“系统管理”，然后点击“Manage Credentials”，打开凭据管理页面。

拉到页面底部找到“Stores scoped to Jenkins”部分。

![](/assets/img/blog/2021/08/22/WX20210810-111337@2x.png)

点击“全局”这个链接，进入“全局凭据”配置页面，点击左侧的“添加凭据”链接，打开“添加凭据”页面。

在这里可以创建两种类型的凭据，一种是"Gitlab API token"，这种凭据主要用在pipeline脚本里。

![](/assets/img/blog/2021/08/22/WX20210810-112007@2x.png)

类型选择“Gitlab API token”,范围选择“全局”，把在Gitlab里创建的personal access token填在“API token”文本框里。

ID要注意不要包含特殊字符，否则在Pipeline里使用时可能会有问题。

另一种凭据是用户名和密码凭据。

![](/assets/img/blog/2021/08/22/WX20210810-141827@2x.png)

## 1.2 通过用户名密码的凭据访问Gitlab

在项目的"源码管理"部分，选择Git，然后输入项目的git地址，在Credentials里选择上面创建的凭据。

![](/assets/img/blog/2021/08/22/WX20210810-144235@2x.png)

Jenkins会自动连接Gitlab，如果所选凭据无法连接Gitlab会报错。

![](/assets/img/blog/2021/08/22/WX20210810-144619@2x.png)

## 1.3 通过Pipeline连接Gitlab

通过Pipeline连接Gitlab适用于多分支流水线项目，在Jenkinsfile里连接Gitlab。

### 创建Gitlab连接

接下来创建一个Gitlab连接。

在Jenkins左侧菜单栏里选择“系统管理”，然后点击“系统配置”。

![](/assets/img/blog/2021/08/22/WX20210810-102956@2x.png)

 在“系统配置”页面向下接，找到Gitlab的配置部分，点击“新增”按钮，创建一个新的Gitlab连接。

![](/assets/img/blog/2021/08/22/WX20210810-112751@2x.png)

“Connection name”注意不要包含特殊字符。

“Credentials”选择上面创建的凭据。

填写好信息后，点击“Test Connection”测试是否能连接到Gitlab，如果测试成功会在左侧显示"Success"。

```groovy
properties([gitLabConnection('Gitlab_Cluster')])

node {
  checkout scm // Jenkins will clone the appropriate git branch, no env vars needed

  // Further build steps happen here

}
```

注意：上面的脚本只在多分支流水线的Pipeline里使用。
