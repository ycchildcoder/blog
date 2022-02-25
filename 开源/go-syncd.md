---
title: go 注意事项
date: 2022-01-27 17:50:31
tags:
---


Syncd是一款开源的代码部署工具，它具有简单、高效、易用等特点，可以提高团队的工作效率。官网地址：[https://syncd.cc/](https://links.jianshu.com/go?to=https%3A%2F%2Fsyncd.cc%2F)

### 特性

-   Go语言开发，编译简单、运行高效
-   Web界面访问，交互友好
-   权限模型灵活自由
-   支持自定义构建
-   支持Git仓库
-   支持分支、Tag上线
-   部署Hook支持，可扩展性强
-   完善的上线工作流
-   邮件通知机制

### 部署流程

这是我自己通过测试发现的syncd部署上线的流程，看完这个流程再结合自己的需求是否使用该工具

-   从git仓库clone代码到syncd所属服务器上
-   通过tar命令将项目压缩成一个文件
-   通过scp命令把压缩文件拷贝到配置好的服务器上
-   在目标服务器上解压文件
-   完成

### 安装syncd

#### 环境需求

**操作系统**
 Linux / macOS + Bash. 需要注意的是Syncd不支持Win系统。
 **Go 编译环境**
 Syncd依赖 `Go1.11+` 编译环境，可前往[官方网站](https://links.jianshu.com/go?to=https%3A%2F%2Fgolang.org%2Fdl%2F) 或 [国内镜像](https://links.jianshu.com/go?to=https%3A%2F%2Fgolang.google.cn%2Fdl%2F) 下载安装。
 **MySQL**
 MySQL 5.6+
 **Git**
 升级操作系统Git到最新版本。

#### 安装

通过命令即可快速安装，如果出现报错，检查一下环境是否满足需求



```cpp
 curl https://syncd.cc/install.sh | bash
```

https://github.com/dreamans/syncd git 仓库下 docs/ 下有install.sh

#### 导入数据库

数据库文件位于syncd安装目录下的resource/sql文件夹中，通过数据库导入命令，将数据导入数据库中。

mysql -uroot -p123456 < syncd.sql 

#### 配置文件

配置文件为`syncd-deploy/etc/syncd.ini`，其中的配置简单易懂，主要修改数据库相关配置即可

#### 启动

进入到syncd-deploy目录下的bin文件夹中，执行`./syncd`即可运行，在浏览器中打开`http://IP:8878`即可进入到登录页。登录账号：`syncd` 密码：`111111`

### 使用

#### 项目空间

项目空间是项目的基本组织单元，是进行项目和多用户隔离和访问控制的主要边界。
 `项目 -> 空间管理 -> 新增项目空间`

![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/2836332-82eff5ba45b1cf3a.png)

#### 项目管理

```
项目 -> 项目管理 -> [切换项目空间] -> 新增项目
```

![2836332-461729d465026bbc111](https://raw.githubusercontent.com/ycchildcoder/markdown/main/2836332-461729d465026bbc111.webp)



#### 成员管理

管理成员所属项目
 `项目 -> 成员管理 -> [切换项目空间] -> 添加新成员`

![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/2836332-f7010d34b2527b36.png)

image.png



#### 集群管理

管理服务器集群
 `服务器 -> 集群管理 -> 新增集群`

![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/2836332-2f9b203616c766b4.png)



#### 服务器管理

管理集群下的服务器，部署服务器（Syncd服务所在的服务器）与生产服务器（代码部署目标机）之间通过ssh协议通信，所以需要将部署服务器的公钥 (一般在这里: `$HOME/.ssh/id_rsa.pub`)加入到生产机的信任列表中（一般在这里 `$HOME/.ssh/authorized_keys`）

![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/2836332-bf1eac05d703f913.png)



#### 构建配置

配置支持的变量只有两个

-   `${env_workspace}`
     代码仓库本地副本目录
-   `${env_pack_file}`
     打包文件绝对地址，构建完成后将需要部署到线上的代码打包到此文件中，必须使用 tar -zcf 命令进行打包。
     部署模块会将此压缩包分发到目标主机并解压缩到指定目录，请按照要求打包，否则会部署失败。
     配置示例

```bash
cd ${env_workspace}
tar -zcvf ${env_pack_file} *
```

#### 新建上线申请单

-   选择项目

    ![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/2836332-045f607db8291730.png)

    

-   填写上线单

    ![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/2836332-d1881672999765c1.png)

    

#### 上线

![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/2836332-81d2216b2c144668.png)



![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/2836332-fbea7dfa050688da.png)



### 总结

Syncd看上去功能比较简单，但是针对小项目的集群发布比较容易。



