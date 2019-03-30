---
title: Hexo 使用
categories: hexo
tags: 
    - hexo
    - Travis-CI
---


## 准备环境

- **Node.JS**
- **npm**
- **Hexo**

1. 在[Node.js](<http://nodejs.cn/download/>) 的下载页面下载对应的版本，**macOS** 版本下载下来的**pkg**文件直接双击就可以安装，安装好了后可以使用命令`npm -v`查看npm的版本。

2. 更改npm源，`$ npm config set registry https://registry.npm.taobao.org npm info underscore`

3. 使用npm安装hexo `npm install -g hexo-cli `

   在安装hexo时报错了`permission denied`的问题，这里我采取的方案是将npm的默认目录修改掉，然后再安装即可。

   ```shell
   # 创建一个新目录
   mkdir ~/.npm-global
   # 将npm配置新目录
   npm config set prefix '~/.npm-global'
   # 打开或者创建 ~/.profile
   # 将这句添加到 .profile文件中
   export PATH=~/.npm-global/bin:$PATH
   # 执行
   source ~/.profile
   ```

   

4. 如果安装后第二次打开命令行使用`hexo -v`出现`hexo: command not found`，则说明hexo的位置没有添加到环境变量中，找到上面重新配置的npm-global路径，查看lib下的**node_modules**中是否有hexo的文件夹，存在就打开hexo/bin，将该路径添加到上面的`.profile`文件中

   ```shell
   export PATH=~/路径/hexo/bin:$PATH
   ```

   

5. 在安装好hexo后，如果是hexo3.0版本以上，还需要安装hexo-server(3.0以上将server单独弄了出来),`npm install hexo-server --save`

## 初始化hexo

新建一个文件夹，用命令行打开这个目录后执行`hexo init`。这样就将一个hexo项目初始化好了，执行`hexo server`启动服务器就可以在浏览器上输入`http://localhost:4000`来预览默认主题。



## 配置主题

这次选中的主题是 [melody](<https://molunerfinn.com/hexo-theme-melody-doc/zh-Hans/quick-start.html#%E5%AE%89%E8%A3%85>)

坑：直接用git拉取到本地的主题中有也有git ，在上传时会被忽略