# Gradle发布构件(Jar)到Maven中央仓库

## OSSRH
> 在开始之前，先对`OSSRH`做下了解是很必要的，因为一开始，我并不知道这是个啥玩儿意。我想和我一样的人应该还是有很多的。

`OSSRH`：`Sonatype Open Source Software Repository Hosting Service`，为开源软件提供maven仓库托管服务。你可以在上面部署snapshot、release等，最后你可以申请把你的release同步到`Maven Central Repository`（`Maven中央仓库`）。

个人的理解，`OSSRH`是`Maven中央仓库`的前置审批仓库，只有你完全符合了发布要求，成功的将你的项目发布到了`OSSRH`，才有机会申请同步到`Maven中央仓库`。

这篇主要是记录这整个流程，方便以后自己查阅，同时可以帮助到想做同样事情的朋友。

## 1、注册Sonatype JIRA账号

> JIRA是Atlassian公司出品的项目与事务跟踪工具，被广泛应用于缺陷跟踪、客户服务、需求收集、流程审批、任务跟踪、项目跟踪和敏捷管理等工作领域。

网址：https://issues.sonatype.org/

无非就是填写下注册信息，没有什么特别的

## 2、创建一个Issue

### 填写资料
可以在头部看到一个`Create`的按钮

![sonatype-create-issue-button](https://user-gold-cdn.xitu.io/2019/1/25/168853a9a2042cf9?w=772&h=130&f=png&s=8386)

会弹出`Create Issue`表单

![sonatype-create-issue-form](https://user-gold-cdn.xitu.io/2019/1/25/168853a99b2c98c9?w=825&h=848&f=png&s=36185)

- `Project`：选择`Community Support - Open Source Project Repository Hosting (OSSRH)`

- `Issue Type`：选择`New Project`

- `Summary`：写个标题做个简单概述你要做啥。真不知道写什么，直接把项目名称写上就行，我就这么干了哈。

- `Group Id`：

    - 自己有域名  
        可以使用子域名作为`Group Id` 。
        例：我的项目叫`paladin2`，那么就用`org.zhangxiao.paladin2`作为`Group Id`    
        > 注意：不能瞎编一个，因为后面审核人员会来审核你是否是该域名的拥有者

    - 自己没域名
        可以借助github，例：我的用户名为`michaelzx`，那么就用`com.github.michaelzx.paladin2`作为作为`Group Id`    

- `Project URL`：要与`Group Id`一定关联性

    - 例1：  
        `Project URL`=`http://paladin2.zhangxiao.org`  
        `Group Id`=`org.zhangxiao.paladin2`  
    - 例2：  
        `Project URL`=`https://github.com/michaelzx/Paladin2`  
        `Group Id`=`com.github.michaelzx.paladin`  

- `SCM url`：版本仓库的拉取地址

### 等待回复

如果有问题，老外在评论中把问题给你指出来，可以在原有的issue把资料改正确
> 我之前是犯了个低级的错误把`Group Id`写成了域名
> 审核人员要处理的issue很多，你可能要耐心等待一会，不要急  
> 我之前急了，就重新提交了2个新的issue，最后管理员还是耐心的把重复的issue关闭

如果一切顺利，那么你会收到审核人员，这样的一个评论：

![sonatype-create-issue-success](https://user-gold-cdn.xitu.io/2019/1/25/168853a9a11bf10e?w=885&h=572&f=png&s=48023)

## 3、准备工作

### 文件要求
为了确保中央存储库中可用组件的质量水平，`OSSRH`对提交的文件有明确的要求。

一个基础的提交，应该包含一下文件：
```
example-application-1.4.7.pom
example-application-1.4.7.pom.asc
example-application-1.4.7.jar
example-application-1.4.7.jar.asc
example-application-1.4.7-sources.jar
example-application-1.4.7-sources.jar.asc
example-application-1.4.7-javadoc.jar
example-application-1.4.7-javadoc.jar.asc
```

- 除了jar包和pom文件，`Javadoc`和`Sources`是必须的，后面会说到用Gradle的一些插件来生成
- 每个文件都有一个对应的`asc`文件，这是`GPG`签名文件，可以用于校验文件

### GPG

#### 安装

说明：后续过程均在`OSX`环境下

`OSX`下可以通过brew来安装`gpg`命令行工具

```bash
$ brew update
$ brew install -v gpg
```
你会发现从`brew`的`gpg`命令行工具，做了国际化支持，连help都是中文，赞👍

> 另外推荐一个工具`GPG Suite`，[传送门](https://gpgtools.org/)  
> 在OSX提供了一个图形化界面，把GPG作为钥匙串来做管理，蛮有意思，不过感觉缺失点功能  

> Windows下  
> 可以安装`gpg4win`，网址：[传送门](https://www.gpg4win.org/download.html)    
页面最上面有个很大的下载按钮，点了以后，貌似会让你捐个款啥的……  
不过你可以往下看，有所有版本的列表地址，可以跳过捐款，在祭上一个[传送门](https://files.gpg4win.org/)  
> 人家只是隐藏的好了一些而已，还是免费的。装完了以后，在命令行中也可以用gpg了  


#### 公钥、私钥、签名
GPG的默认秘钥类型是RSA，这里涉及涉及几个概念`公钥`（public-key）、`私钥`（secret-key）、`签名`(sign/signature)

- `公钥`和`私钥`是成对
- `公钥`加密，`私钥`解密。
- `私钥`签名，`公钥`验证。

#### 新建一个密钥

生成了密钥以后，才能导出公钥、私钥

```
$ gpg --generate-key
```
创建的时候，会让你输入`密码`，别输了以后忘记了，后面gradle插件中会用到。

#### 查看已经生成的密钥
```
$ gpg -k
---------------------------------
pub   rsa2048 2019-01-25 [SC] [有效至：2021-01-24]
      72963F6B33D962380B1DC4BD8C446B86DF855F85
uid           [ 绝对 ] zhangxiao'paladin2 <mail@zhangxiao.org>
sub   rsa2048 2019-01-25 [E] [有效至：2021-01-24]
```
72963F6B33D962380B1DC4BD8C446B86DF855F85，这个叫做`密钥指纹`，用来做唯一识别  
后面8位`DF855F85`,叫做`标识`或`KEY ID`，后面会用到

#### 导出私钥文件

很多英文文档或文章中经常出现`KeyRingFile`这个词，这个到底是啥？
> https://users.ece.cmu.edu/~adrian/630-f04/PGP-intro.html
> Keys are stored in encrypted form. PGP stores the keys in two files on your hard disk; one for public keys and one for private keys. These files are called keyrings. As you use PGP, you will typically add the public keys of your recipients to your public keyring. Your private keys are stored on your private keyring. If you lose your private keyring, you will be unable to decrypt any information encrypted to keys on that ring.


```
$ gpg --export-secret-keys [密钥指纹] > secret.gpg
```
以上命令就可以生成一个`二进制`的私钥文件，后面需要配置到gradle中，让插件帮我们给文件批量签名
> 加上`-a`会生成一个用ASCII 字符封装的`文本`文件，方便复制,不过我们这里不需要

#### 上传公钥到公钥服务器

```
$ gpg --keyserver keyserver.ubuntu.com --send-keys [密钥指纹]
```
在sonatype的仓库提交后，会需要一个校验步骤  
会需要从`多个`公钥服务器上下载匹配的公钥，然后来校验你上传的文件的签名

![sonatype-gpg-signature-validation](https://user-gold-cdn.xitu.io/2019/1/25/168853a9a1205653?w=2858&h=1372&f=jpeg&s=898521)

简单的说，你用来签名的`私钥`和你上传的`公钥`，必须要一对，这样才能通过校验

以下是sonatype会去拉取的公钥服务器列表
```
keys.gnupg.net
pool.sks-keyservers.net
keyserver.ubuntu.com
```
> 为什么我要特意列出来？
> 因为有些文章或教程里面，都仅给出了一个服务器，如`pool.sks-keyservers.net`
> 但是，我在实际操作有时候因为网络原因，并不是总能成功上传。
> 所以，如果把公钥上传到`keyserver.ubuntu.com`也是OK的。


#### 总结

1. 密钥的`key id`
2. 密钥的`password`
3. 私钥的`KeyRingFile`
4. 公钥上传到了公钥服务器

准备好了以上几项，我们就可以开始撸Gradle了

> 有些文章说，最好是先生成一个吊销凭证备用，暂时不知道什么场景下会用到

## 4、配置Gradle插件

主要依赖于两个插件：
- `signing`：https://docs.gradle.org/4.10.2/userguide/signing_plugin.html
- `maven-publish`：https://docs.gradle.org/4.10.2/userguide/publishing_maven.html

### 常规配置

修改`build.gradle`
```groovy
// ... 在最下面新增以下代码

apply plugin: 'maven-publish'
apply plugin: 'signing'

task sourcesJar(type: Jar) {
    from sourceSets.main.allJava
    classifier = 'sources'
}

task javadocJar(type: Jar) {
    from javadoc
    classifier = 'javadoc'
}


publishing {
    // 定义发布什么
    publications {
        mavenJava(MavenPublication) {
            // groupId = project.group
            // artifactId = project.name
            // version = project.version
            // groupId,artifactId,version，如果不定义，则会按照以上默认值执行
            from components.java
            artifact sourcesJar
            artifact javadocJar
            pom {
                // 构件名称
                // 区别于artifactId，可以理解为artifactName
                name = 'Paladin2 Common'
                // 构件描述
                description = 'Paladin2 Common library'
                // 构件主页
                url = 'https://paladin2.zhangxiao.org'
                // 许可证名称和地址
                licenses {
                    license {
                        name = 'The Apache License, Version 2.0'
                        url = 'http://www.apache.org/licenses/LICENSE-2.0.txt'
                    }
                }
                // 开发者信息
                developers {
                    developer {
                        name = '听风.Michael'
                        email = 'mail@zhangxiao.org'
                    }
                }
                // 版本控制仓库地址
                scm {
                    url = 'https://github.com/michaelzx/Paladin2'
                    connection = 'scm:git:git://github.com/michaelzx/Paladin2.git'
                    developerConnection = 'scm:git:ssh://git@github.com:michaelzx/Paladin2.git'
                }
            }
        }
    }
    // 定义发布到哪里
    repositories {
        maven {
            url "https://oss.sonatype.org/service/local/staging/deploy/maven2"
            credentials {
                // 这里就是之前在issues.sonatype.org注册的账号
                username sonatypeUsername
                password sonatypePassword
            }
        }
    }
}

signing {
    sign publishing.publications.mavenJava
}


javadoc {
    // <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
    // 防止本地打开中文乱码
    options.addStringOption("charset", "UTF-8")
}
```

在工程根目录下新增`gradle.properties`

```
signing.keyId=密钥keyId
signing.password=密钥password
signing.secretKeyRingFile=私钥keyRingFile路径

sonatypeUsername=sonatype账号
sonatypePassword=sonatype密码
```

### 保护敏感信息

以上方式，基本是按照gradle文档的例子写的。  
但是假如你的工程是提交到非私有仓库，那么Sonatype的账号信息和GPG的信息，就这样暴露的话，似乎不是那么科学。所以，我们需要换个方式来加载这些properties。

删除刚才的`gradle.properties`，新建一个`publish.gradle`文件，放到一个不会被提交到仓库的位置（记得做好备份）。
```
ext.'signing.keyId' = '密钥keyId'
ext.'signing.password' = '密钥password'
ext.'signing.secretKeyRingFile' = '私钥keyRingFile路径'

ext.'sonatypeUsername'='sonatype账号'
ext.'sonatypePassword'='sonatype密码'
```
在`build.gradle`最上面加上一行：
```
apply from: 'path/publish.gradle'
```
> 为了方便，我是直接放在了工程根目录的`.gradle`目录下  
> `apply from: rootDir.canonicalPath + '/.gradle/publish.gradle'`

## 5、提交到Sonatype仓库

首先，用Gradle执行`publishToMavenLocal`任务，先在本地发布下，看看生成了哪些文件，或还有什么问题

然后，用Gradle执行`publish`任务，发布到指定的maven仓库

如果没有报错，那么恭喜你，已经成功提交到了sonatype的仓库中

但是提交成功，并不代表发布成功

## 6、到Sonatype OSS发布

用你之前注册的账号密码，登录：https://oss.sonatype.org/

登录后查看左侧的`Build Promotion`->`Staging Repositories`

你的提交，会出现在`最下面`，至于其他是啥，我也不太清楚😂


![sonatype-staging-repository-close](https://user-gold-cdn.xitu.io/2019/1/25/168853a9a0e19796?w=2084&h=1556&f=jpeg&s=496834)

### Close

你的提交在未处理前，是`open`状态，然后点击`Close`按钮。

> 一开始不太明白这个`Close`是啥意思，后来看了下，主要按照中央仓库的要求来验证下你上传到文件
> 签名就是其中一个步骤，会去公钥服务器拉取公钥，然后来验证你所有的文件

需要等待一会，等它执行完毕，状态会从`open`变成`closed`

> 如果碰到错误的话，仔细看下`Activity`选项卡，执行了什么步骤，那个步骤出现了什么错误，很清晰的

### Release

一般情况下，感觉如果顺利`close`后，再次选中点击`Release`，耐心等待一会，就大功告成了！

可以在侧边栏`Artifact Search`中搜索下你的`groupId`，此时应该能看到对应的构件名称和版本了


## 7、回复Issue

但是很抱歉，到此为止，你的jar包，还不会同步到maven仓库中

你需要在你原先创建issue中，告诉下管理人员，你已经完成了第一次发布

我用我蹩脚的英文，回复如下：

> I have already completed the first release.

然后管理人员给我回复了：

> Central sync is activated for org.zhangxiao.paladin2. After you successfully release, your component will be published to Central, typically within 10 minutes, though updates to search.maven.org can take up to two hours.

OK，至此，你的构建就会同步到`Maven Central Repository`了。

## 8、同步需要多久

可能你会像我一样很着急，啥时候可以用，怎么还搜不到呢~。~  
从sonatype同步到中央仓库，的确是需要一定的时间  

不过根据我的观察，文件的同步，会早于索引的同步  

比如，比如你等了半天，然后 https://search.maven.org/ 上搜一下依然搜不到
那么，你可以去 https://repo1.maven.org/maven2/ 上按照坐标找下，是否能找到你的包

如果能找到，那你就可以开始引用它了


## 9、新版本更新

只要完成第一个发布，后续就不需要再创建`issue`了，只要重复`5-6`步骤可以了。

你可以在`groupId范围内`发布`任意名称`的构件

## 结束语
以前在`NPM`仓库中发布过自己的东西。相比之下，`Maven`仓库的发布流程，让人感觉严谨很多。

赶紧一起在Maven全球中央仓库中，留下你自己专属的印记吧！

一定会让你感觉棒棒的^_^

## 参考文章

https://central.sonatype.org/pages/ossrh-guide.html
