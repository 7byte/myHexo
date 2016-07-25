---
title: 基于hexo搭建github个人博客
date: 2016-07-22 10:38:04
tags:
---
@(github博客)[github]

## 为什么要写个人博客
>“好记性不如烂笔头”

一直倔强地认为只要我记性足够好，就根本不需要把时间花在做笔记、总结这种多余的事情上面。然而事实证明，我的这个想法的确没错，问题出就出在我的记性还不够好，至少还没好到完全不用做笔记的地步。当我意识到这点的时候，结合自身程序员的身份，很自然地萌生了“创建一个只属于自己的个人博客”的想法，于是有了现在你看到的这篇文章。

## 配置Hexo
>[Hexo](https://hexo.io/)是一个快速、简洁且高效的博客框架。Hexo 使用 Markdown（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。

- 下载安装[Node.js](https://nodejs.org/en/)
- 安装 Hexo
```
$ npm install -g hexo-cli
```
不知道是不是网络原因，上面的命令我在家里自己电脑上卡了二十多分钟才执行完，不过从最终结果来看，并没有什么问题。
- 创建hexo工作目录
新建一个文件夹Hexo，进入文件夹后
```
$ npm install
```
- 启动本地预览
```
$ hexo init
```
我执行上面这条命令时在最后一步报了个警告：
```
$ npm WARN deprecated minimatch@0.3.0: Please update to minimatch 3.0.2 or higher to avoid a RegExp DoS issue
```

这时候`Ctrl+C`就可以了，这个警告不会对后续流程有什么影响，但本着追求完美的态度（没错我是个处女座），我们执行下面的命令更新这个有问题的包
```
$ npm install minimatch@"3.0.2"
```
- 然后生成静态文件并启动服务
```
$ hexo g
$ hexo s
```
启动服务后的默认访问网址为： http://localhost:4000/
如果以上各步都没问题，就能在浏览器看到新建的网页了，这里看到的是默认主题，将来根据自己的需要修改，参考官方[主题](https://hexo.io/themes/)

## 部署到github
- 安装hexo git部署插件 [hexo-deployer-git](https://github.com/hexojs/hexo-deployer-git)
```
$ npm install hexo-deployer-git --save
```

- 修改hexo配置
工作目录中的 _config.yml 是网站的配置信息，包括网站标题、作者名等等。这里我们修改deploy参数为github仓库路径。
```
deploy: 
  type: git
  repo: https://github.com/7byte/7byte.github.com.git
```
这里有一点需要注意：**deploy参数冒号后面一定要带一个空格**，否则后果自负。

- 部署github
```
$ hexo deploy
```
第一次部署的时候需要输入github账号和密码，以后再部署不用重新输入。
执行上面的命令后会把 \public 目录下的文件全部同步到指定的github pages仓库，仓库中原来的**所有文件都会被清空**，然后替换成我们新提交的文件。
