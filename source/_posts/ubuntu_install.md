---
title: "Ubuntu18.04.1的化妆之术"
date: 2018-11-23T14:30:57+08:00
draft: false
---

> 我那12年的老朋友现在带Windows系统已经吃力了，跑个IDE内存开个浏览器啥的，内存就很吃紧了，但是岁数在那儿了再优化也没办法啦，只好换个装变个性换下血啥。

# 准备工具

- 容量大于4G的U盘一只
- 笔记本一台
- Ubuntu镜像一坨（ [官网传送门](https://www.ubuntu.com/index_kylin) ）

# 制作系统启动盘

网上有很多Ubuntu系统盘的制作教程，这里介绍一种简单方便的方法，首先下载工具[Rufus官网](https://rufus.ie/en_IE.html)，这个工具使用非常简单，下载后安装，然后运行起来，选择你的U盘，刚刚下载的ISO镜像，其他的默认点击**[Start]**就可以了。

# 安装系统

把U盘插入电脑，启动电脑时进入BOIS菜单，切换为U盘启动，然后保存重启。重启后就会进入Ubuntu的安装，这里没有啥特别的，基本和普通的软件一样点击下一步设置完就好了。

# 配置系统环境

### 基础设置
启动了系统后呢先别急着搞事情，有些东西需要先初始化下，一般在进入系统前，如果你安装时选择的是中文，那么这时候会提示不完整的语言支持，点击**[现在执行此动作]**，就可以了。然后再使用命令行工具进行下更新,接下来就是悄悄咪咪的等待了。。。

```bash
sudo apt-get update
sudo apt-get upgrade
```

更改系统的时区和设置好终端的样式

```bash
sudo apt-get install git
sudo apt-get install zsh
wget https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh -O - | sh
chsh -s /usr/bin/zsh
```

### 工具配置

首先是输入法，这里选择搜狗的输入法[传送门在此](https://pinyin.sogou.com/mac/)。安装输入法需要先安装**[Fcitx]**框架。

```bash
sudo apt install fcitx
```

然后到搜狗官网选择Linux版本的输入法下载,会得到一个deb包，下载成功后到下载目录打开终端执行以下的命令。

```bash
# 首先移步到文件管理器的下载目录，终端下输入以下命令进行安装
sudo dpkg -i sogoupinyin_2.2.0.0108_amd64.deb

# 一般情况下会提示安装失败，缺失依赖，所以先解决依赖问题
sudo apt install -f

# 接着重复第一步安装搜狗输入法的命令
# 一般 deb 包都是如此安装的，如果失败就去解决依赖问题
```

然后移步到 **设置 → 区域和语言** ，删除一部分输入源，只保留 **汉语** 。接着选择 **管理已安装的语言** ，修改 **键盘输入法系统** 为 **fcitx** 。关闭窗口，打开所有程序，选择软件 **Fctix 配置**，选择加号添加搜狗输入法(`如果添加输入法中没有搜狗输入法，重启下系统`)

### 主题配置

在Ubuntu18.04上使用的**gnome**，所以先下载安装**gnome-tweak-tools** 和 **gnome-shell**两个东东

```bash
sudo apt install gnome-tweak-tool
sudo apt install gnome-shell
```

**gnome**有一个主题网站，上面有各种各样的主题，可以根据自己的喜好选择，[gnome-look传送门](https://www.gonme-look.org)，
**flat-remix-gtk**系列的主题个人感觉挺酷的，于是就打算装这个啦，在其github主页也有详细的安装教程[Flat-remix-gtk传送门](https://github.com/daniruiz/flat-remix-gtk)

```bash
sudo add-apt-repository ppa:daniruiz/flat-remix
sudo apt-get update
sudo apt-get install flat-remix-gtk
sudo apt-get install flat-remix-icon
```

如果在终端无法拉取下来安装，可以直接用git将主题包拉取到本地，然后将对应主题的文件夹移动到相应目录

    主题存放目录：/usr/share/themes 或 ~/.themes
    图标存放目录：/usr/share/icons 或 ~/.icons
    字体存放目录：/usr/share/fonts 或 ~/.fonts

其中如果需要使用界面进行拷贝复制到 **/usr/share/** 路径下，需要 **root** 权限

```bash
# 终端下打开一个具有管理员权限的文件管理器
# 打开后终端最小化，不要关闭
sudo nautilus
```

主题搞定了后在全部应用打开gnome(`中文名叫做优化`),就可以在外观中选择主题了，但是这个时候会发现 **Shell** 对应的选项有红色的感叹号，这个时候需要安装 **User theme** 的插件才能使用，一个简单的方法是 在Extension的网站[传送门](https://extensions.gnome.org/extension/19/user-themes/) 点击 **Click here to install browser extension** 安装后刷新网页，在 **User theme** 右边就有一个开关，打开它就可以了，再回到优化中，就可以在外观中选择shell theme了。这个时候顶部栏和全部应用的样式就改好了。(`这步骤中如果进入网页没有 Install browser extension 这个提示，检测下 gnome-shell 是否安装`)

然后是 **Dock** 的样式，其实 **Docky** 蛮好用的，但是他没有全部应用，也不好添加应用图标。所以这里还是使用主题推荐的扩展 **Dash to Dock** 在刚刚的扩展网页可以直接搜索这个扩展，和上面一样的操作打开 **ON** 就可以啦，然后如果需要设置为主题中的样式，需要把在扩展的设置中把透明度设置为固定大小，然后下面的数值才能拖动为0。

还有朋友喜欢应用全面屏的，可以先设置 **Dock** 智能隐藏， 然后添加扩展 **Hide top bar**。

这样呢，使用时的主题样式就差不多了，但是呢（没错，但是前都是废话），开机的时候如果是多系统会发现Ubuntu原生的系统选择不好看，还有登录用户的界面也差那么多点意思。我找了一款看起来清新简洁的 [没错我是传送门](https://www.gnome-look.org/p/1176413/) ,这个主题只需要执行以下的命令就可以使用了。

```bash
# 稳的方法
wget -P /tmp https://github.com/shvchk/poly-light/raw/master/install.sh
bash /tmp/install.sh
# 浪的方法
wget -O - https://github.com/shvchk/poly-light/raw/master/install.sh | bash
```

最后就是装饰下门面啦，修改掉系统开机动画和登录界面。登录界面修改掉背景就会好很多啦
主要修改`/etc/alternatives/gdm3.css`文件中的 **lockDialogGroup**属性，将里面的background 的url指向你想设置的图片就OK了。

开机动画需要先下好一个，我选了个酷酷的[传送门](https://www.gnome-look.org/p/1009456/)，直接下载下来解压，然后将里面的 **ubuntu-spinner-logo** 文件夹拷贝到 `/usr/share/plymouth/themes/`下，然后修改`/etc/alternatives/default.plymouth`中的属性为你刚刚下载的动画的

```bash
[script]
ImageDir=/usr/share/plymouth/themes/ubuntu-spinner-logo
ScriptFile=/usr/share/plymouth/themes/ubuntu-spinner-logo//usr/share/plymouth/themes/ubuntu-spinner-logo.script
```
保存关机重启应该就能看到效果啦。

# 参考链接
- https://zhuanlan.zhihu.com/p/41708902
- https://www.k-xzy.xyz/archives/5577/comment-page-6