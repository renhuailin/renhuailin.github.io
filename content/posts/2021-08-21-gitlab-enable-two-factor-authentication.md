---
layout: article
title: Gitlab启用双因子认证
date: 2021-08-21 12:38:00
author: 任怀林
tags:
- linux
- gitlab
thumb: linux.png
comment: true
isCJKLanguage: true
---

## 1.  下载Microsoft Authenticator

这是一个手机app，iPhone请在App Store里搜索`Microsoft Authenticator`确认开发者为微软公司，然后安装。

![iPhone Authenticator](/assets/img/blog/2021/08/22/WechatIMG240.png)

安卓手机请在自带的商店里搜索`Authenticator`，然后找开发者为微软公司的app安装。
![loading-ag-3166](/assets/img/blog/2021/08/22/WechatIMG4.jpeg)

## 2. 启用双因子认证

登录到源Gitlab，点击右上角的头像图标，在下拉菜单中选择"Preferences"。

![preferences](/assets/img/blog/2021/08/22/WX20210731-163932@2x.png)

在打开页面的侧边栏里点击“Account“，打开“Account“页面。

点击”Two-Factor Authentication“下面的`Enable two-factor authentication`按钮。

![enable-two-factor-authentication](/assets/img/blog/2021/08/22/WX20210814-140952@2x.png)

系统会显示一个二维码，同时在二维码的右侧显示了Account和Key的信息，请一定要把这些信息复制下来并保存起来，以便在手机丢失或误删除`Microsoft Authenticator`账号时恢复账号。

**注意：如果为root帐户启用双因子认证，请一定要保存这些信息**。

![](/assets/img/blog/2021/08/22/WX20210814-145237@2x.png)

打开手机上的`Microsoft Authenticator` app，点击添加账户，选择`其他(Google、Facebook等)`，然后选择“扫描QR码”，然后用手机扫描页面上的二维码。添加成功后，点击这个帐户，app上会显示一组6位的数字，把它输入到页面上，然后点击`Register with two-factor app`，系统会显示` Two-factor Authentication Recovery codes`。

![recovery-codes](/assets/img/blog/2021/08/22/WX20210816-141222@2x.png)

请复制这些代码并保存起来。

**注意：如果为root帐户启用双因子认证，请一定要保存Recovery codes**。

## 3. 登录

启用双因子认证后，用户在登录时，在输入完用户名和密码后，Gitlab会跳转到一个页面，要求用户输入`Microsoft Authenticator` app上的6位密码才能成功登录。

![input-dynamic-password](/assets/img/blog/2021/08/22/WX20210816-135441@2x.png)

开启双因子认证后，无法再使用Gitlab的用户名和密码来clone项目，否则会报如下错误：

```
$ git clone http://10.94.0.48/oms/oms-gateway.git
Cloning into 'oms-gateway'...
Username for 'http://10.94.0.48': renhuailin
Password for 'http://renhuailin@10.94.0.48':
remote: HTTP Basic: Access denied
remote: You must use a personal access token with 'read_repository' or 'write_repository' scope for Git over HTTP.
remote: You can generate one at http://10.94.0.48/-/profile/personal_access_tokens
fatal: Authentication failed for 'http://10.94.0.48/oms/oms-gateway.git/'
```

开启双因子认证后，要用personal access token来代替密码来clone项目。

## 5. 生成Personal  Access Token

登录到源Gitlab，点击右上角的头像图标，在下拉菜单中选择"Preferences",

在打开页面的侧边栏里点击“Access Tokens”打开“Personal Access Tokens”页面。

输入一个名字，在Scopes下页面为新token选择权限。

请选择'read_repository' 和 'write_repository'这两个scope。

然后点击“Create personal access token”按钮，生成token。

![create-personal-access-token](/assets/img/blog/2021/08/22/WX20210814-161351@2x.png)

**注意：生成的token只显示一次请复制下来，保存好。**

以管理员身份登录到目标Gitlab，按同样的步骤生成一个token，保存下来以便在迁移时使用。

用这个token代替原来密码，就可以用git client来pull Gitlab上的代码了。

## 6. 丢失Authenticator帐户

如果安装`Microsoft Authenticator` app手机丢失，或是误删除了`Microsoft Authenticator` app上的帐户，该如何登录Gitlab呢？

### 6.1 使用Recovery codes

如果在启用双因子认证时保存了Recovery codes，那么可以用Recovery codes来代替`Microsoft Authenticator`上的动态密码来登录Gitlab。

**注意：每行Recovery code只能使用一次。**

### 6.2 使用Key重建Authenticator帐户

如果在启用双因子认证时保存了二维码右侧的Key，则可以用这个Key来重建`Microsoft Authenticator`账号。

打开手机上的`Microsoft Authenticator` app，点击添加帐户，选择`其他(Google、Facebook等)`，然后点击扫描框下面的`或手动输入代码`。

![](/assets/img/blog/2021/08/22/WechatIMG243.png)

把保存的Key填在第二个文本框里，点击`完成`就可以重建帐户了。

![](/assets/img/blog/2021/08/22/WechatIMG242.jpeg)

重建帐户后，就可以用帐户上显示的动态密码登录了。

### 6.3 以管理员身份双因子认证

如果丢失了手机，或是误删除了`Microsoft Authenticator` app上的帐户，而且也没有保存Key或`Recovery codes`。还可以让系统管理员把帐户的双因子认证关闭，然后用户登录后再次启用。

下面介绍一下管理员用户如何禁用其他用户的双因子认证。

以管理员身份登录到Gitlab，然后点击Gitlab图标旁边的`Menu`菜单，然后选择`Admin`,进入Gitlab管理界面，然后点击左侧菜单里的`Users`,进入用户管理界面。

![](/assets/img/blog/2021/08/22/WX20210816-153419@2x.png)

找到要操作的用户，然后点击用户名进入用户的详情页面。

![](/assets/img/blog/2021/08/22/WX20210816-153918@2x.png)

点击`Impersonate`按钮，模仿这个用户。

![](/assets/img/blog/2021/08/22/WX20210816-153956@2x.png)

通过上面截图，可以看到`You are now impoersonating renhuailin`,说明已经进入了模仿模式。

点击右上角的头像图标，在下拉菜单中选择"Preferences"。

在打开页面的侧边栏里点击“Account“，打开“Account“页面。

![](/assets/img/blog/2021/08/22/WX20210816-154032@2x.png)

点击”Two-Factor Authentication“下面的`Manage two-factor authentication`按钮。

![](/assets/img/blog/2021/08/22/WX20210816-154059@2x.png)

点击`Disable two-factor authentication`,禁用双因子认证。这样用户就可以只使用用户名和密码登录系统了。

### 6.4 以管理员身份生成Recovery codes

如果丢失了手机，或是误删除了`Microsoft Authenticator` app上的帐户，而且也没有保存Key或`Recovery codes`。可以让系统管理员重新生成`Recovery codes`，然后让用户使用`Recovery codes` 登录系统并重新关联`Microsoft Authenticator` 帐户。

具体操作方法，请参见`6.3 以管理员身份双因子认证`,在最后一步的界面上，点击`Regenerate recovery codes`并保存下来。
