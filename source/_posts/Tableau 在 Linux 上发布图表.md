---
title: Tableau 在 Linux 上发布图表
date: 2017-12-20
tags:
- Tableau
- 数据可视化
- 持续集成
---

　　这是一段比较有趣的经历，大家都知道 Tableau 是 C\S 结构，发布图表的常规做法是在 Tableau Desktop 上进行手动发布，但作为一个注重效率的程序员，这显然不是我们的首选。Tableau 在 Windows 上提供了命令行工具：[tabcmd](https://onlinehelp.tableau.com/current/server/en-us/tabcmd.htm) ，可以解决我们命令行的问题，然而工作都使用 Mac 环境，我们的 CI 工具也部署在 Linux 环境，这条路似乎走不通了。

### Linux 版本

　　偶然间发现 Tableau Server 有 [Linux beta](https://www.tableau.com/about/blog/2017/7/add-tableau-your-linux-shop-and-download-beta-72929?signin=317faeb631f1a3c9ba2d074e56928b14) 版本，同时有 tabcmd 命令行：

![](/assets/images/tableau/linux_cmd.jpg)

这对于我们来说就足够了，RPM 在Mac 上可以通过 tar 命令解压就可以使用：

```shell
tar -zxvf ./tableau-tabcmd-10.5-beta5.noarch.rpm
```

Tabcmd 使用可以点击[这里](https://onlinehelp.tableau.com/current/server/en-us/tabcmd_cmd.htm#iddf805b62-18ff-4497-9245-adc6905b2084)，这里不过多赘述。

有了命令行我们就可以在开发阶段持续集成，一切都很美好~

### 发布陷入死局

　　然而在上线前夕，我们得到通知客户使用的 Tableau Server 是10.3 版本，并不是最新版 10.4，看起来这只是个小版本，却导致我们所有拖拽出来的报表都无法正常开的，tabcmd 也无法正常工作，并且在官网上无法找到低版本的Linux tabcmd，更由于客户的内部政治问题，本地开发无法连接生产环境，不能够在 Windows 上进行发布等等，我们再一次陷入死局。

### 破局

　　在万念俱灰之时又一次的偶然发现 tabcmd 命令，竟然是使用 Java 进行推送，竟然还是 `Spring-Boot`，不知道 Tableau 用了什么技术将 Jar 包打到了shell 命令中：

``` shell
#!/bin/bash

...
TABCMD_HOME=$HOME/.tableau/tabcmd
exec "${JAVA}" -Xmx64m -Xss2048k -Djsse.enableSNIExtension=false -Dpid=$$ -Dlog.file=$TABCMD_HOME/tabcmd.log -Dsession.file=$TABCMD_HOME/tabcmd-session.xml -Din.progress.dir=$TABCMD_HOME -Dconsole.codepage="${LANG}" -Dconsole.cols=$COLUMNS -jar "${BASH_SOURCE[0]}" "$@"
PK
    [�TK            	   META-INF/PK
    [�TK\��v  v     META-INF/MANIFEST.MFManifest-Version: 1.0
Gradle-Version: Gradle 4.0.2
Built-By: Tableau Software
Start-Class: com.tableausoftware.tabcmd.Tabcmd
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Spring-Boot-Version: 1.4.0.RELEASE
Created-By: 1.8.0_131 (Oracle Corporation 25.131-b11)
Build-Version: latest
Main-Class: org.springframework.boot.loader.JarLauncher

PK
    ���H               META-INF/maven/PK
...

```

　　虽然后面这一坨完全看不懂，但可以猜测 Windows 的 tabcmd 应该使用了同样技术，我们能够在官网中找到 10.3 的 Windows 版本：

![](/assets/images/tableau/linux_cmd2.jpg)

![](/assets/images/tableau/linux_cmd3.jpg)

安装好后，我们可以看到命令中带有一个 jar 文件，虽然 exe 中的内容是什么我们无法得知，但一定是使用这个jar 来实现登录和发布的：

![](./tableau/linux_cmd4.jpg)

我们将 Jar 包拷贝到本地的 Mac，再尝试使用 Linux tabcmd 的执行方式:

```shell
java -Xmx64m -Xss2048k -Djsse.enableSNIExtension=false -Dlog.file=./dist/tabcmd.log -Dsession.file=./tabcmd-session.xml -Din.progress.dir=./ -jar ./app-tabcmd-latest-jar.jar login -s $TABLEAU_HOST -u $TABLEAU_USERNAME -p $TABLEAU_PASSWORD
```

![](/assets/images/tableau/linux_cmd5.jpg)

当当当~竟然成功了，我们再尝试一下发布命令：

``` shell
java -Xmx64m -Xss2048k -Djsse.enableSNIExtension=false -Dlog.file=./dist/tabcmd.log -Dsession.file=./tabcmd-session.xml -Din.progress.dir=./ -jar ./app-tabcmd-latest-jar.jar publish $FILE -t $SITE -r $PROJECT --overwrite --db-username $DB_USERNAME --db-password $DB_PASSWORD --save-db-password -s $TABLEAU_HOST -u $TABLEAU_USERNAME -p $TABLEAU_PASSWORD
```

Oh Yeah! 完美搞定~

### 总结

　　整个折腾的过程极其有趣，当中遇到很多问题，有很多同事参与帮忙，甚至打 Tableau 的400 电话，但因为不是售后付费用户，不能提供技术支持，而中国的 Tableau 代理商也只是使用 Desktop 进行发布，从来没有使用过命令行进行发布。

　　最终还是要靠自己搞定，当然也侧面突出了 Java 是最好的语言。