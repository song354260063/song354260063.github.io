---
layout:     post                    # 使用的布局（不需要改）
title:      Jenkins+Maven+Linux远程部署       # 标题
subtitle:   持续集成   #副标题
date:       2019-03-03              # 时间
author:     songjunhao                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - DevOps
    - 持续集成
    - Jenkins
---

### 下载Jenkins

首先进入官方网站

https://jenkins.io/download/

下拉页面，选择 .war 文件进行下载。如下图所示。

![](https://i.loli.net/2019/03/03/5c7b7dce44fbb.jpg)

### Jenkins搭建

进入 jenkins.war 所在目录，进入cmd执行
```
java -jar jenkins.war
```

首次启动后，在控制台中如下图所示

![](https://i.loli.net/2019/03/03/5c7b7ee632e53.jpg)

此时说明启动成功（默认启动端口:8080），打开浏览器
进入

http://localhost:8080

进入后待初始化结束后，如下图所示。请将上图红色框内字符串输入，输入后，点击【Continue】。

![](https://i.loli.net/2019/03/03/5c7b7fafbdaab.jpg)

进入下图所示插件初始化界面，选择建议的插件即可。

>如果未进入下图所示界面，而进入了Offline提示页面，可以进入 http://localhost:8080/pluginManager/advanced
这里面最底下有个【升级站点】，把其中的链接改成http，点击保存。然后重启Jenkins，再按上述步骤操作即可。

![](https://i.loli.net/2019/03/03/5c7b821698cb1.jpg)

点击后进入插件下载界面，待全部下载完成后进入管理员账号配置页面，如下图所示。

>如果失败点击【Retry】重新下载插件即可。
>如果一直失败，可以尝试进入 http://localhost:8080/pluginManager/advanced 将最下方的升级站点，修改为：
http://mirror.esuni.jp/jenkins/updates/update-center.json
然后重启Jenkins，继续操作。

![](https://i.loli.net/2019/03/03/5c7b8d8f997cd.jpg)

录入管理员信息后，点击蓝色按钮【Send And Continue】。
之后的步骤可以按默认直接点击确认，也可按需自行设置即可进入Jenkins控制台页面。

![](https://i.loli.net/2019/03/03/5c7b8e3da9ef9.jpg)

至此 Jenkisn 已经搭建成功，下面将对Jenkins进行打包及远程部署配置。

### Maven插件安装及配置

首先我们需要先安装 **Maven** 所需要的插件。
进入【系统管理】-> 【插件管理】，进入可安装的插件页签。
安装如下所示插件：

Maven Integration

![](https://i.loli.net/2019/03/03/5c7b92e97339e.jpg)

安装成功后，进入【系统管理】 -> 【全局工具配置】。在这里对Maven及JDK进行配置。如下图所示：

![](https://i.loli.net/2019/03/03/5c7b94db41032.jpg)

![](https://i.loli.net/2019/03/03/5c7b9582244a0.jpg)

配置成功后点击最下方【Save】按钮保存即可。

### Linux 远程部署插件及配置

首先我们需要先安装 **远程部署** 所需要的插件。
进入【系统管理】-> 【插件管理】，进入可安装的插件页签。
安装如下所示插件：

**Publish Over SSH**

安装成功后，进入【系统管理】 -> 【系统设置】

找到 Publish over SSH 栏，进行配置。

![](https://i.loli.net/2019/03/03/5c7b9cdc3c7f2.jpg)

首先点击 【Add】，展开如下图所示界面

![](https://i.loli.net/2019/03/03/5c7b9d5d15f61.jpg)

录入Name(服务器名称)(任意)，Hostname(服务器IP地址)，Usrename（用户名）之后，再点击【Advanced】按钮。展开如下图所示界面。勾选截图中内容。

![](https://i.loli.net/2019/03/03/5c7b9f1d0c632.jpg)

勾选后，显示密码输入栏。

![](https://i.loli.net/2019/03/03/5c7b9ff61ca8d.jpg)

IP，端口，用户名，密码等输入完毕，可以点击下方【Test Configuration】 测试连接。如果配置无误，则显示 Success。

![](https://i.loli.net/2019/03/03/5c7ba04d6e453.jpg)

配置成功后点击最下方【Save】按钮保存即可。

### Jenkins持续集成任务配置

进入Jenkins首页，点击【新建任务】。

![](https://i.loli.net/2019/03/04/5c7c81f9d4680.jpg)


输入名称，选择构建一个Maven项目，之后点击OK。

![](https://i.loli.net/2019/03/04/5c7c826fb7bd5.jpg)

#### 代码仓库配置

进入配置页面。在页面中下拉找到，**Source Code Management**，配置SVN地址及账号，用于构建时直接从SVN拉取代码。建议在SVN项目路径最后加上 @HEAD 这样可以每次都拉取最新版本，防止SVN仓库所在服务器与Jenkins服务器系统时间差异导致更新的代码非最新版本。

>Git 与 SVN类似。

![](https://i.loli.net/2019/03/04/5c7c84b993fce.jpg)

#### 介质远程部署配置

配置好SVN仓库后，继续在页面中下拉找到 **Build Environment**，此菜单用于设置，在构建完毕后，将Maven打好的包部署到远程部署机上，并执行脚本进行重启。如下图所示。

![](https://i.loli.net/2019/03/04/5c7c870ce5b06.jpg)

#### Maven 打包命令配置

在页面下拉找到 **Build** ，此菜单可以配置 maven 打包执行的命令，例如一个项目中只需打包某个模块，或者指定本地仓库配置，跳过测试等。

![](https://i.loli.net/2019/03/04/5c7c88534cd91.jpg)


以上均配置完毕后，点击【Save】按钮进行保存。

## 总结

现在，已经配置好了Jenkins的持续构建。现在可以点击【立即构建】进行构建啦。

>从SVN拉取代码 -> Maven打包编译 -> 远程部署Linux服务器 -> 重启服务。
