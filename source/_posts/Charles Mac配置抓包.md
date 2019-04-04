---

title: "Charles Mac配置抓包"
date: 2019-04-03T18:31:23+08:00
draft: false
tags: 
    - Charles 
    - 抓包
    - Mac
categories : 
    - Android
    - 工具
---

> 笔者是Android 开发，所以本篇文章主要针对Android对Android设备的抓包

### 工具准备

在[官网](<https://www.charlesproxy.com/>) 点击右边的**Download** 可以下载



### 配置Mac端

1. 依次点击在顶部导航的 **help–>SSLProxying–> Install Charles Root Ceriticate** 
    ![图1](https://raw.githubusercontent.com/Grieey/Grieey.github.io/source/img/charles1.png)

2. 然后 **cmd+空格** 输入钥匙串，在钥匙串中选择**登录->所有项目** 找到**Charles Proxy CA**的条目，右键显示简介，在简介中的信任选择始终信任
![图2](https://raw.githubusercontent.com/Grieey/Grieey.github.io/source/img/charles2.png)
![图3](https://raw.githubusercontent.com/Grieey/Grieey.github.io/source/img/charles3.png)
3. 回到Charles中，点击导航栏的**Proxy –>Proxying Settings ** ,输入端口为 **8888** 并勾选下面的选项
![图4](https://raw.githubusercontent.com/Grieey/Grieey.github.io/source/img/charles4.png)
4. 再点击 **Proxy –>SSL Proxying Settings ** ，在弹出的对话框中add一个，http一栏输入*，端口输入443，这样会监听所有的**https**的请求
![图5](https://raw.githubusercontent.com/Grieey/Grieey.github.io/source/img/charles5.png)

### 配置移动端

1. 在电脑端的charles中，点击**help–>SSLProxying–> Install Charles Root Ceriticate on a Mobile Device or Remote Browser**  ，完成后charles会有一个提示框，大概意思就是下面要进行的操作
2. 首先保证手机和电脑处于同一个局域网中，在手机的网络设置中修改当前连接的网络，点开高级，设置手动代理，**IP** 为电脑的局域网IP地址，端口为刚刚设置的 **8888** ，保存网络
![图6](https://raw.githubusercontent.com/Grieey/Grieey.github.io/source/img/charles7.jpeg)
3. 在手机浏览器中输入 **http://charlesproxy.com/getssl** ，会自动下载一个证书，记住这个证书在手机存储中的位置。
4. 这步中如果是非华为手机，可以尝试直接点击3中下载的证书看是否能安装。如果是华为手机会提示没有能打开此文件的应用，那么需要到 **设置-> 安全和隐私-> 从SD卡安装** 找到刚刚下载的证书的目录，选择证书就可以安装
![图7](https://raw.githubusercontent.com/Grieey/Grieey.github.io/source/img/charles6.jpeg)
5. 安装证书时需要注意，名称可以随意取，但是**凭据用途** 这一项会有两个选项，`VPN和应用`、`WLAN`。这里是**Android就一定要选择WLAN** 

这样整个抓包就配置好了，在手机上发起请求就可以在charles看到具体的请求了。

如果需要更高级的应用场景(弱网请求模拟、服务器压力测试等)可以参考[Charles 从入门到精通](https://blog.devtang.com/2015/11/14/charles-introduction/)

### 参考文档

- [android使用Charles抓包https请求](https://blog.csdn.net/honjane/article/details/54602926)

- [Android安装Charles证书（华为手机测试）](https://blog.csdn.net/weixin_42034554/article/details/86669159)