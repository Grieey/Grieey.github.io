---

title: "Hugo主题使用"
date: 2018-11-02T15:31:23+08:00
draft: false
tags: 
    - 杂谈 
    - Hugo
    - Travis CI
categories : 
    - 博客
    - Hugo
    - 使用手册
---

# 准备工具

 - **Hugo**
 - **Github账号**

# 开始锯板子

**Hugo**的介绍这里不用细讲了，网上一大堆的都说的比较详细了，在**macOS**中可以直接使用`brew`下载安装

```bash
brew install hugo
hugo version # 检查安装成功
hugo new site my_site # 初始化一个hugo的目录
cd my_site
hugo init
```

执行命令后会在对应的目录生成hugo的基本文件和文件夹，该目录的结构如下

      ▸ archetypes/
      ▸ content/
      ▸ layouts/
      ▸ static/
        config.toml

- ``config.toml``，站点的配置文件，这里面可以定义博客，邮箱以及主题等信息。
- ``content/``，博客中的文章对应的.md文件都会生成在这个文件夹下。

然后使用`hugo new about.md`就会生成一个about.md在content中，在命令里加路径就会在content下对应的文件夹生成，如`hugo new /post/about.md`。

# 雕纹上色

其实选择**hugo**的原因之一就是主题比较好看，因此**theme**是很必要的。 在[themes.gohugo.io](https://themes.gohugo.io/)中,选择自己喜欢中意的，一般点进入主题中都会进入到**Github**，里面也有作者写的使用教程，参考着应该没有多大问题。在Terminal中进入到刚刚创建的工程根目录，直接**clone**下来,这里以`Even`主题为例。

```bash
git clone https://github.com/olOwOlo/hugo-theme-even themes/even
```

然后将`themes/even/example`中的`config.toml`文件复制到自己的工程根目录覆盖。再根据**Github**中作者的介绍或者指导修改`config.toml`的对应属性即可。

最后键入`hugo server -D`命令启动hugo中内置的服务器，在浏览器中输入`localhost:1313`查看网站的效果。

# 板上钉钉

>前面的步骤都是在本地运行的效果，要别人能够访问你的博客，肯定得部署到网络上，这里选择GitPages，不需要自己对后台服务器进行管理，比较省心。

首先在自己的github中新建一个仓库，仓库的名字为`xxx.github.io`，其中`xxx`替换为自己的github用户名。

然后在**Teminal**中打开本地的工程，执行`hugo -d blog`生成网站的网页内容。这里开始需要注意一下，一般会将hugo生成的内容提交到仓库的master分支，然后会将网站的源码提交到另一个分支。所以这里我是这样骚操作的。

```bash
cd blog  #进入blog目录
git init  #初始化git
git add -A
git commit -m "first commit"
git remote add origin git@github.com:RiceBrother/RiceBrother.github.io.git
git push -u origin master #提交到master分支
cd ../   #回到工程根目录
git init #再次初始化git
git checkout -b source #创建并切换到source分支
echo '/public/' >> .gitignore
sed -r -i '/^\/blog\/$/{$!d}' .gitignore  #添加忽略文件，忽略掉blog文件夹
git add -A
git commit -m "commit sources"
git remote add origin git@github.com:RiceBrother/RiceBrother.github.io.git
git push -u origin master #提交到source分支   
```

这样操作后在浏览器输入`xxx.github.io`应该就能看到效果了。

# 投入使用

如果每次新写了一篇文件，都需要手动的使用`hugo -d blog`然后提交到master分支，再回到根目录提交source分支，这个过程很繁琐，作为懒人的我第一时间是想到能不能使用脚本自动化，于是本着懒人的原则先google了一番，参考了几位仁兄的方案后，选择了使用**Travis CI**。

第一步呢就是在**Github**个人的**Setting**中找到**Developer setting**，然后生成一个**Personal assess tokens**。然后到**CI**这个网站先注册一番，[传送门在此](https://travis-ci.org/)，这个网站可以直接使用**Github**账号登录，非常方便。

![同步](../../img/sync.png)

注册成功后同步自己的仓库，将对应的`xxx.github.io`仓库激活，并点击**Setting**设置。找到下面的**Environment Variables**添加刚刚在自己的**Github**生成的**Token**，`**Name**必须为GITHUB_TOKEN这个是网站的规则，不能是其他的值`。

![仓库](../../img/res.png)
![增加变量](../../img/add_variable.png)

最后再返回到自己的github.io这个仓库，在source分支中添加一个`.travis.yml`，在里面输入以下脚本,**Travis CI**会自动检测哪个分支添加了这个文件并运行。

```python
# https://docs.travis-ci.com/user/deployment/pages/
# https://docs.travis-ci.com/user/languages/python/
language: python

python:
    - "3.6"

install:
    - uname -a
    - wget https://github.com/gohugoio/hugo/releases/download/v0.42.2/hugo_0.42.2_Linux-64bit.deb
    - sudo dpkg -i hugo*.deb
    - hugo version
    - rm -rf blog
    - ls
    - pwd

script:
    - hugo -d blog
    - echo 'Hugo build done!'
after_success:
    - git config --global user.email "yangqing43352@163.com"
    - git config --global user.name "griee"
    - git clone https://$GITHUB_TOKEN@github.com/RiceBrother/RiceBrother.github.io.git container
    - rm -rf container/blog
    - cp -r blog/* container
    - cd container
    - ls
    - git add .
    - git commit -m 'Travis upate blog'
    - git push -u origin master
```

这个时候切回到**Travis CI**网站查看脚本执行的日志，显示`Done. Your build exited with 0.`一般没有问题`(如果有问题可以查看job log，有可能是git 提交失败)`，刷新自己的`xxx.github.io`看看效果，这样每次只需要修改自己的文章并提交到**source**分支，就会自动触发脚本生成**blog**目录并提交到**master**分支。

# 展示自我

>正常情况下呢通过上面的步骤应该能正常访问到博客了，但是，你建立一个博客肯定不仅仅是给自己看的吧，所以要让别人在搜索引擎中能搜到自己，就需要做做下面的功夫了。

这里以**Google**搜索为例，在**Google**中输入`site:https://ricebrother.github.io/`提示查询不到相关内容，则说明你的站点并未被收录，所以别人也搜不到你的文章内容。

进入[Google搜索文档地址](https://www.google.com/webmasters/tools/home?hl=zh-CN)，登录自己的账号，然后添加自己的网址，会有一个验证，将该html下载下来，上传到master分支就可以了。通过验证可以在控制台查看你的站点的访问量等等。

最后将`https://xxx.github.io/sitemap.xml`添加到Google站点地图就可以啦。



# 参考链接
- https://justforuse.github.io/blog/2018/07/travisci-deploy-hugo/
- https://axdlog.com/zh/2018/using-hugo-and-travis-ci-to-deploy-blog-to-github-pages-automatically/
- https://segmentfault.com/a/1190000012975914
- https://www.jianshu.com/p/df46bca5889d
