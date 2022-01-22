---
title: 创建自己的blog
date: 2022-01-21 10:50:31
tags:
---



### 1、创建一个repository

打开自己的github,创建一个名为“username.github.io”的项目，***注意：username必须是自己的github的用户名\***
<img src=https://raw.githubusercontent.com/ycchildcoder/markdown/main/pages_create_repo.png width=700 height=500 />)

#### 1.1 Clone the repository

克隆项目

```
git clone https://github.com/username/username.github.io
```

#### 1.2  Hello World

进入到你克隆下来的项目目录里，添加一个html文件

```
cd username.github.io
echo "Hello World" > index.html
```

#### 1.3 Push it

把它推到github上，Add, commit, and push your changes:

```
git add --all
git commit -m "Initial commit"
git push -u origin master
```

#### 1.4 访问

最后访问你的website ***[http://username.github.io](http://username.github.io/)\***

### 2、安装hexo

一个免费的域名外加一个只能写静态页面的server。现在你只需要更新一下你的html，git push到你对应的gitHub的repository上，外网就可以访问到你最新的内容。有人会说：“假如每次写博客都手动写一遍前端代码，太麻烦了，写完了还得自己发布到github,有没有一种工具可以帮我们干这件事呢？” 答案是有的，例如[***Hexo\***](https://hexo.io/zh-cn/)。

安装 Hexo 相当简单，只需要先安装下列应用程序即可：

Node.js (Node.js 版本需不低于 10.13，建议使用 Node.js 12.0 及以上版本)
Git

#### 2.1 安装nodejs
你可以根据不同平台系统选择你需要的 Node.js 安装包。

Node.js 历史版本下载地址：https://nodejs.org/dist/

注意：Linux 上安装 Node.js 需要安装 Python 2.6 或 2.7 ，不建议安装 Python 3.0 以上版本。

#### 2.2 安装

安装 Hexo 相当简单。然而在安装前，您必须检查电脑中是否安装了Node.js和Git。
如果电脑上已经装完了Node.js和Git,则：

```
npm install -g hexo-cli
npm install -g hexo
```

#### 2.3 建站

安装 Hexo 完成后，请执行下列命令，Hexo 将会在指定文件夹中新建所需要的文件。

```
$ hexo init <folder>
$ cd <folder>
$ npm install 
```

输入一下指令

```
hexo generate  (此指令可以简化为:hexo g)
hexo server （此指令可以简化为:hexo s）
```

启动服务器。默认情况下，访问网址为： http://localhost:4000/

#### 2.4 部署

推送生成的html代码到github，以便外网访问。

到blog的根目录下。打开_config.yml配置文件。翻到最后一行，编辑deploy相关的信息。内容如下:

```
deploy:
  type: git
  repo: https://your_git_token@github.com/i6448038/i6448038.github.io.git   你的github仓库地址
  branch: master
```

配置结束以后。输入以下指令，把代码自动部署到github。

```
hexo deploy （此指令可以简化为：hexo d）
```

如果你输入以上指令后没有反应，那么你应该：

```
npm install hexo-deployer-git --save
```

成功后，你就可以访问自己的gitPages了，[https://username.github.io](https://username.github.io/)

#### 2.5 写博客

打开gitPages后发现都是程序默认生成的东西，现在需要自己写自己博客的内容了。

```
hexo new "文章题目"  (此指令可以简化:hexo n "文章题目")
```

关于MarkDown的编辑器，也多种多样，Atom、sublime、Mou、甚至vim也可以。

写完内容后，如果想改变当前博客的风格，可以去Hexo Themes的官网选择自己喜欢的风格。 https://hexo.io/themes/
打开网址：
![img](https://raw.githubusercontent.com/ycchildcoder/markdown/main/website.png)
点击喜欢的内容，会自动打开到该theme的gitHub仓库。

Aloha对应的gitHub仓库：

 https://github.com/henryhuang/hexo-theme-aloha.git 

然后你需要做如下操作:

```
$ cd $YOUR_BLOG_ROOT_DIR 
$ git clone https://github.com/henryhuang/hexo-theme-aloha.git themes/aloha
```

然后更改当前文件夹下的_config.yml，把该文件中theme的值改为aloha。除此之外，如果你想更改该风格的站点菜单栏字段等文本内容，可以去theme文件夹下的aloha子文件夹下的__config.yml文件中修改。
然后执行

```
$ hexo g
$ hexo s
```

浏览器打开 [http://localhost:4000](http://localhost:4000/) 看看风格是否改变

### 3 自动化

#### 3.1 启动本地hexo
bat脚本启动hexo本地服务器
```
@echo off
E:
cd E:\blog\blog
hexo s
```

#### 3.2 部署hexo 博客
```
@echo off
E:
cd E:\blog\blog
hexo clean && hexo g && hexo d
```
