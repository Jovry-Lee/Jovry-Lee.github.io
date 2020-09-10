---
title: Ubuntu16.04-搭建Hexo-Blog
date: 2020-08-20 17:19:56
tags: ["Ubuntu","Config","Hexo"]
categories: ["Ubuntu", "Config"]
---

#### 1 简介

GitHub Pages 是一项静态站点托管服务，它直接从 GitHub 上的仓库获取 HTML、CSS 和 JavaScript 文件，通过构建过程运行文件，然后发布网站。 Hexo是高效的静态站点生成框架，它基于Node.js. 通过Hexo，可以直接使用Markdown语法来撰写博客。

<!--more-->

#### 2 配置Git

配置方式参考：[Ubuntu16.04 Github配置及使用](https://jovry-lee.github.io/2020/08/18/Ubuntu16-04-Github%E9%85%8D%E7%BD%AE%E5%8F%8A%E4%BD%BF%E7%94%A8/#more)

#### 3 安装node.js

安装方式参考：[Ubuntu16.04 Node.js安装.note](https://jovry-lee.github.io/2020/08/21/Ubuntu16-04-Nodejs%E5%AE%89%E8%A3%85/)

#### 4 安装Hexo

安装命令：

```bash
$ npm install -g hexo-cli
```

检查hexo是否安装成功：

```bash
$ hexo -v
```

*得到hexo-cli：4.2.0等一串数据，安装成功。*

#### 5 Hexo创建本地博客

- 初始化本地站点

  本地博客路径在`~/ProjectWork/githubBlog`下,本地博客搭建操作在此路径下进行操作.

  ```bash
  $ cd ~/ProjectWork
  $ mkdir githubBlog # 创建本地博客路径
  $ hexo init # 初始化本地站点
  ```

*注：hexo init 命令要求该当前文件夹为空文件夹。*

- 安装依赖包

```bash
$ npm install
```

- 生成网页

```bash
$ hexo g
```

*注：由于我们还没创建任何博客，生成的网页会展示 Hexo 里面自带了一个 Hello World 的博客。*

- 将网页放在本地服务器

```bash
$ hexo s
```

- 测试本地博客

  在浏览器里输入http://localhost:4000/ 

- 发布一篇博客

  在本地博客路径下执行以下命令, 此时source/\_posts下会生成一个“日志名.md”的文件，该文件即是日志文件。

```bash
$ hexo new "<日志名>"
```

- 生成网页并放到本地服务器

```bash
$ hexo g 
$ hexo s
```

#### 6 将本地Hexo博客部署到Github上

##### 6.1 创建代码仓库

- 在Github中创建一个以.github.io结尾的Repository。

- - ①、Repository name:  `Jovry-Lee.github.io`
  - ②、勾选 Initialize this repository with a README
  - ③、Create repository

- 简单地编辑一下 README.md 这个文档。 比如添加：I am trying to create my own blog.. 保存(Commit changes)。

- 打开网页：`jovry-Lee.github.io` 这里就可以看到 README.md 里的内容了。


##### 6.2 配置本地代码仓库

- 获取Github对应的Repository的链接。（git@github.com:Jovry-Lee/Jovry-Lee.github.io.git）


- 修改本地站点配置文件。


```bash
$ sudo vim _config.yml # 打开配置文件
```

- 找到#Deployment，填入以下内容：


```
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repository: https://github.com/Jovry-Lee/Jovry-Lee.github.io.git
  branch: master
```

- 部署

```bash
$ npm install hexo-deployer-git --save # 安装hexo-deployer-git，该步骤只需要做一次
$ hexo d
```

*得到 INFO Deploy done: git 即为部署成功*

此时访问`Jovry-Lee.github.io`即可看到博客页面。

#### 7 使用Next主题

##### 7.1 配置NexT主题

- 获取主题代码

​	克隆主题代码到本地博客`themes/next`路径下.	

```bash
$ git clone https://github.com/next-theme/hexo-theme-next.git themes/next
```

- 修改博客配置文件
  - 打开 `~/ProjectWork/githubBlog/_config.yml`

  - 找到 theme:

  - 把 Hexo 默认的 lanscape 修改成 next。 即 theme: next

  - 找到 # Site，添加博客名称，作者名字等。

  - 在 language 后面填入 en 或者 zh-Hans，选择英文或者中文。

  - 找到 # URL, 填入 url。比如 url:https://jovry-lee.github.io/

    

    当前本地配置如下:

```
# Extensions
## Plugins: https://hexo.io/plugins/
## Themes: https://hexo.io/themes/
theme: next

...

# Site
title: Jovry's blog
subtitle: ''
description: Keeping learning and improving!
keywords:
author: Jovry Lee
language: en
timezone: 'Asia/Shanghai'

...

# URL
## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'
url: https://jovry-Lee.github.io
```

- 清除旧配置

​	*注意：修改配置后最好都进行一下清理操作，不然可能不生效。*

```bash
$ hexo clean
```

- 重新生成部署


```bash
$ hexo g -d
```

##### 7.2 NexT主题优化

###### 7.2.1 修改NexT主题Scheme

​	NexT当前支持4种风格,默认为Muse,在NexT主题配置文件(`themes/next/_config.yml`)中可以修改Scheme,配置如下:

```
# Schemes
#scheme: Muse
#scheme: Mist
#scheme: Pisces
scheme: Gemini
```

###### 7.2.2 添加访问统计

​	通过配置NexT主题配置文件,修改busuanzi_count.配置如下:

```
# Show Views / Visitors of the website / page with busuanzi.
# Get more information on http://ibruce.info/2015/04/04/busuanzi
busuanzi_count:
  enable: true
  total_visitors: true
  total_visitors_icon: fa fa-user
  total_views: true
  total_views_icon: fa fa-eye
  post_views: true
  post_views_icon: far fa-eye
```

###### 7.2.3 添加头像

​	添加的头像可以保存到主站目录下,或者主题目录下.

- Site路径: 保存到`~/ProjectWork/Jovry-Lee.github.io/public/uploads`路径下.
- theme路径下:`~/ProjectWork/Jovry-Lee.github.io/themes/next/source/images`路径下

配置NexT主题配置文件,修改avatar项:

```
# Sidebar Avatar
avatar:
  # in theme directory(source/images): /images/avatar.gif
  # in site  directory(source/uploads): /uploads/avatar.gif
  # Replace the default image and set the url here.
  url: /uploads/princess.jpeg
  # 配置为true时,头像会在一个圈圈中.
  rounded: true
  # 配置为true时,鼠标放在头像上,头像有旋转效果.
  rotated: true
```

###### 7.2.4 添加About/Tags/Categories等页面

​	默认情况下,About页面是不存在的,即使将主页展示了About的图标,若不进行页面配置,点击跳转会报404. 配置方式如下:

- 进入博客Bash路径,生成about等页面.

```bash
$ cd ~/ProjectWork/Jovry-Lee.github.io
$ hexo new page "about" # 生成about页面
$ hexo new page "tags" # 生成tags页面
$ hexo new page "categories" # 生成分类页面
```

- 配置NexT主题配置文件,修改`munu`项,启用图标, 修改如下:

```
menu:
  home: / || fa fa-home
  about: /about/ || fa fa-user
  tags: /tags/ || fa fa-tags
  categories: /categories/ || fa fa-th
  archives: /archives/ || fa fa-archive
  #schedule: /schedule/ || fa fa-calendar
  #sitemap: /sitemap.xml || fa fa-sitemap
  #commonweal: /404/ || fa fa-heartbeat
```

- ​	添加about个人介绍,直接在`~/ProjectWork/Jovry-Lee.github.io/source/about/index.md`文件上进行编辑即可.

###### 7.2.5 分类和标签页自动生成categories&tags

- 分类页面配置

​	在生成的分类页面`~/ProjectWork/Jovry-Lee.github.io/source/categories/index.md`上进行如下修改:

```
---
title: categories
date: 2020-08-21 15:08:25
type: "categories"
---
```

- 标签页面配置

​	在生成的分类页面`~/ProjectWork/Jovry-Lee.github.io/source/tags/index.md`上进行如下修改:	

```
---
title: tags
date: 2020-08-19 15:30:01
type: tags
---
```

###### 7.2.6 去掉目录栏序号

由于本地写文章时，已经对文章标题进行编号，而NexT主题默认是自动添加目录编号，因此，这里选择关闭该功能．

修改NexT主题配置文件`toc`项,将`number`项设置为`false`, 修改如下:

```
# Table of Contents in the Sidebar
# Front-matter variable (unsupport wrap expand_all).
toc:
  enable: true
  # Automatically add list number to toc.
  number: false
```

###### 7.2.7 设置侧边栏社交链接

社交连接也是在NexT配置文件中进行修改, 关键字`social`,进行修改,去掉`#`,添加个人链接即可.

###### 7.2.8 设置Post Body字体大小

由于NexT主题中Post Body部分的字体大小默认为1.125em，个人觉得不是很好看，因此自定义修改该字体的大小．

通过查看`<site>/themes/next/source/css/_variables/base.styl`文件中定义了一系列`Font Size`．其中`post-body`默认使用的font-size-large，即1.125em大小．对`<site>/themes/next/source/css/_common/components/post/post-body.styl`进行设置．

```css <site>/themes/next/source/css/_common/components/post/post-body.styl
.post-body {
  font-family: $font-family-posts;
  word-wrap();

  +desktop-large() {
    // font-size: $font-size-large;
    font-size: font-size-medium;
  }
```

###### 7.2.9 设置代码高亮样式

在[NexT Highlight Theme Preview](https://theme-next.js.org/highlight/#)中找到自己喜欢的格式，然后进行配置．

当前使用默认的Highlight.js插件，选用atelier-seaside-light样式，对NexT主题配置如下：

```tex /themes/next/_config.yml
codeblock:
  # Code Highlight theme
  # All available themes: https://theme-next.js.org/highlight/
  theme:
    light: atelier-seaside-light
    dark: atelier-seaside-dark
  ．．．
```

##### 7.3 基于NexT主题开启评论功能——Gitalk

该评论功能使用Gitalk服务实现。

###### 7.3.1 注册OAuth Application

- 登录GitHub
- 前往 `https://github.com/settings/profile`
- 点击左侧下方的 `Developer settings`
- 点击绿色 `Register a new application`
- 填写以下内容：

```
Application name：gitalk-comment
Homepage URL：https://jovry-lee.github.io/
Application description：Blog comment system
Authorization callback URL：https://jovry-lee.github.io/
```

- 点击 Register application

- 得到：

```
Client ID：xxx 

Client Secret： xxxx
```

###### 7.3.2 创建存放Gitalk-comments的repository

- 创建 repository。 Repository name 为：`gitalk-comments`
- 地址：`https://github.com/Jovry-Lee/gitalk-comments`
- 注意稍后配置中填的是 `gitalk-comments`，<u>不是地址</u>。

###### 7.3.3 添加Gitalk到博客

- 打开本地博客路径下next主题的配置文件

```bash
$ vim ~/ProjectWork/Jovry-Lee.github.io/themes/next/_config.yml
```

- 找到gitalk，进行如下修改：

```
# Gitalk
# For more information: https://gitalk.github.io, https://github.com/gitalk/gitalk
gitalk:
  enable: true
  github_id: Jovry-Lee # GitHub repo owner
  repo: gitalk-comments # Repository name to store issues
  client_id: xxx # GitHub Application Client ID
  client_secret: xxxx # GitHub Application Client Secret
  admin_user: Jovry-Lee # GitHub repo owner and collaborators, only these guys can initialize gitHub issues
  distraction_free_mode: true # Facebook-like distraction free mode
  # Gitalk's display language depends on user's browser or system environment
  # If you want everyone visiting your site to see a uniform language, you can set a force language value
  # Available values: en | es-ES | fr | ru | zh-CN | zh-TW
  language: en
```

- 重新部署

```bash
$ hexo clean 

$ hexo g -d
```



#### 8 多端使用Hexo博客

首先应该确保某一台电脑搭建好了Hexo博客，然后进行后续操作。

##### 8.1 主端配置

- ①、登录Github，在`username.github.io`仓库上新建hexo分支。

  在博客仓库上新建一个分支，例如“hexo”，切换到该分支，并设置该分支为默认分支（Setting->Branches->Default branch）

- ②、克隆博客仓库到本地


*注：不是本地Hexo目录。*

```shell
$ git clone git@github.com:Jovry-Lee/Jovry-Lee.github.io.git
```

查看当前分支，确保为新建的hexo分支

```shell
$ cd Jovry-Lee.github.io/ $ git branch * hexo
```

`后续操作均是在Jovry-Lee.github.io目录(Jovry-Lee.github.io与githubBlog目录同级)下完成。`

- ③、拷贝本地博客的部署文件（Hexo目录下的全部文件到`username.github.io`文件目录中），然后删除`themes`目录中主题下的`.git`目录（如果存在的话），因为一个git仓库中不能包含另一个git仓库，提交主题文件夹会失败。然后提交。


```bash
# 拷贝Hexo目录内容
$ sudo cp -r ~/ProjectWork/githubBlog/* ./
# 删除themes目录下主题的.git目录，我本地用的next主题
$ sudo rm -r themes/next/.git
# 提交修改
$ git add .
$ git commit -m "back up hexo files"
$ git push
```

- ④、后续写博客，即在`username.github.io`文件目录中进行了，由于仓库有个`gitignore`文件，里面忽略掉了`node_modules`文件夹，也就是说仓库的`hexo分支`并没有存储该目录，所以需要重新install一下。


```bash
$ npm install
```

##### 8.2 其他电脑端配置

​	安装Hexo环境，然后克隆`username.github.io`仓库的hexo分支到本地，此时本地git仓库处于hexo分支；切换到username.github.io目录，安装依赖包。

```bash
 $ git clone git@github.com:Jovry-Lee/Jovry-Lee.github.io.git
 $ cd Jovry-Lee.github.io
 $ npm install
```

这里，如果npm install出错，如”npm ERR! Unexpected end of JSON input while parsing near”,可尝试：

- 删除package-lock.json文件
- 清除cache: npm cache clean --force
- 不要用淘宝镜像：npm set registry https://registry.npmjs.org/

##### 8.3 发布更新博客

更新博客内容后提交到github,执行以下操作进行提交及其部署.

```bash
$ git add .
$ git commint -m "注释"
$ git push
$ hexo d -g
```

*注：每次操作时，最好先git pull 一下。*



#### 9 设置图床

为了解决图片的存储问题，使用第三方静态资源库，即图床，获取图片Url，目前可供选择的图床很多，小众一些的容易挂，大厂存储服务又需要花钱，因此，这里使用Github + jsDelivr + PicGo + Imagine打造自己的图床。

##### 9.1 配置Github

- 创建仓库

- - 输入项目名称
  - 选择权限为公开
  - 初始化一个READMEmd文件
  - 创建项目

- 生成一个Token

  - 点击用户头像->选择”Settings“->点击”Developer settings“->点击”Personal access tokens“->点击”Generate new token“
  - 在“Node”栏键入token的备注
  - 在“Select scopes”中勾选“repo”
  - 点击“Generate token”按钮

- 获取Token秘钥：<u>该秘钥只会显示一次，注意自己保存一下，方便后续使用</u>。

##### 9.2 配置PicGo，并使用jsdelivr作为CDN加速

- 下载PicGo

- - [下载地址](https://github.com/Molunerfinn/PicGo)
  - Linux系统下载`AppImage`文件，更改其权限为可执行，双击即可显示应用图标。

- 右键点击图标选择“打开详细窗口”->点击“图床设置”->选择“GitHub图床”，进行GitHub设置

  - 设定仓库名：Jovry-Lee/cdn

  - 设定分支名：master

  - 设定Token：<Github配置时生成的那个秘钥>

  - 指定存储路径：img/   *(注: 指定存储路径，将会在仓库下创建设置名称的文件夹（eg：img），上传的图片将保存在里面。)*

  - 设定自定义域名： eg：`https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn`  *(注: 自定义域名的作用是，在图片上传后，PicGo会按照自定义域名+上传图片名的方式生成访问链接，放到粘贴板上。因为我们要使用 jsDelivr 加速访问，所以可以设置为https://cdn.jsdelivr.net/gh/用户名/图床仓库名。)*


##### 9.3 图片压缩工具

通常情况下，图片大小都是超过200KB的，网页加载会比较慢，所以需要对图片进行压缩。

- 在线网站压缩

  [网站地址](https://tinypng.com/)

- Imagine工具压缩

  [下载地址](https://github.com/meowtec/Imagine)

##### 9.4 图片上传/获取

- 上传: 上传区进行图片上传，PicGo工具支持多个图床,需要选择上传的图床,选择`GitHub图床`。


- 获取: PicGo应用点击`相册`，选择图床，即可显示该图床下的所有图片。




生成的图片的链接示例：`https://cdn.jsdelivr.net/gh/Jovry-Lee/cdn/img/%E7%9F%A9%E5%BD%A2@3x.png`



#### Markdown编辑器安装

GitHub page支持Markdown语法,推荐使用Markdown进行编辑.

##### Typora

[Typora官网](https://typora.io/)

安装方法：

- 命令行安装

```bash
# or run:
# sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys BA300B7755AFCFAE
$ wget -qO - https://typora.io/linux/public-key.asc | sudo apt-key add -

# add Typora's repository
$ sudo add-apt-repository 'deb https://typora.io/linux ./'
$ sudo apt-get update

# install typora
$ sudo apt-get install typora
```

- 源文件安装
  - 官网下载二进制文件
  - 解压到指定目录
  - 配置环境变量

```bash
$ wget https://typora.io/linux/Typora-linux-x64.tar.gz
$ sudo tar zxvf Typora-linux-x64.tar.gz -d /usr/local
$ sudo vim ~/.bashrc
```

填入以下信息:

```
#set typora
export TYPORA_HOME=/usr/local/Typora-linux-x64
export PAHT=$TYPORA_HOME:$PATH
```

```bash
$ source ~/.bashrc
```



---

#### 参考资料

[用 Hexo 和 GitHub Pages 搭建博客](https://ryanluoxu.github.io/2017/11/24/用-Hexo-和-GitHub-Pages-搭建博客/)

[Ubuntu 16.04下Github配置](https://lrscy.github.io/2017/05/01/Ubuntu-Github-config/) 

[Next使用文档](http://theme-next.iissnan.com/getting-started.html)

[Getting Started](https://theme-next.js.org/docs/getting-started/)

[Hexo中如何给一篇文章加多个tags？](https://www.zhihu.com/question/43517242)

[创建分类页面](https://github.com/iissnan/hexo-theme-next/wiki/创建分类页面)

[hexo之next主题添加分类](https://blog.csdn.net/u011240016/article/details/79422462)

[使用多台电脑写Hexo博客](https://cccshuang.github.io/2018/09/28/使用多台电脑写Hexo博客/)

[在WSL下快速搭建hexo](https://www.vivatakethat.com/2016/07/07/在Windows下快速搭建hexo/)

[Hexo官网](https://hexo.io/zh-cn/)

[Hexo Blog折腾笔记](https://ghamster0.github.io/2019/03/12/Hexo Blog折腾笔记/)

[GitHub + jsDelivr + PicGo + Imagine 打造稳定快速、高效免费图床 ](https://www.cnblogs.com/sitoi/p/11848816.html)

[Hexo 的 Next 主题优化](https://ryanluoxu.github.io/2017/11/26/Hexo-的-Next-主题优化)

[Hexo+Next主题优化](https://zhuanlan.zhihu.com/p/30836436)