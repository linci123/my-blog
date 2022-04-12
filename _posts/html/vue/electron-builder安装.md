---
title: 使用vue搭建一个electron程序
date: 2020-06-12 16:13:20
tags: 
  - electron
  - vue
  - nodejs
categories:
  - 编程
img: /images/article-banner/WechatIMG38.png
---

折腾了两天，尝试了electron-vue和electron-forge，效果都不怎么理想，electron-vue的electron版本太老，升级坑实在是太多，electron-forge版本不知道怎么更新，最后还是选择原生的vue框架来搭建，查了不少资料，解决的一部分问题

## 创建一个vue+electron项目

```bash
vue create app
```
一路回车下来

```bash
cd app
vue add electron-builder
```
## 修改electron-builder安装源

这里装了一晚上electron-builder都没成功，魔法上网和替换cache中的文件都试了一遍，安装的效果不是很满意，今天突然在csdn看到一个评论，完美解决

先修改npm的配置，也就是.npmrc这个文件

```bash
npm config edit
```
将下面两行加到文件中
```bash
electron_mirror=https://npm.taobao.org/mirrors/electron/
electron-builder-binaries_mirror=https://npm.taobao.org/mirrors/electron-builder-binaries/
```

之后就可以```vue add electron-builder```了

我是在```npm install electron -g```之后才运行的，不知道不装全局electron有没有影响

![](/images/QQ截图20200612162107.png)

这里还有个坑，可能会导致build失败

```cannot move downloaded in to final location```

![](/images/QQ截图20200612162205.png)

进到path提示的路径中，把一串数字命名的文件夹再重命名为path中的名称即可

## 运行electron

```bash
npm run electron:serve
```

大概率是可以成功运行了

![](/images/QQ截图20200613153557.png)

## 修改background配置

接下来的配置才是天坑，你会发现node的api在vue中支楞不起来，原因是electron禁止了在网页中操作node

在background.js中修改webPreferences中的nodeIntegration，原先的```process.env.ELECTRON_NODE_INTEGRATION```打印出来是false

```javascript
  win = new BrowserWindow({ 
    width: 800, 
    height: 600, 
    webPreferences: {
      nodeIntegration: true //process.env.ELECTRON_NODE_INTEGRATION
  } })
```

## 依旧无法执行？？？

原因是electron暴露出来的api需要在函数前面加个window才可以运行

例：

```javascript
const fs = window.require('fs');
```

可以在vue中对window解构

```javascript
const { require } = window
const fs = require('fs')
console.log(fs.readdirSync('.'))
```


成功打印

![](/images/QQ截图20200613154748.png)

