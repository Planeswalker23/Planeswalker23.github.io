---
layout: post
title: MacOS 安装 homebrew 搭建 Git 环境
categories: [Others]
description: MacOS 安装 homebrew 搭建 Git 环境
keywords: MacOS, homebrew, Git
---

# 一、Homebrew介绍

homebrew是一款Mac平台的软件包管理工具，官方对于它能做什么的回答是：“Homebrew 使 macOS 更完整。使用 gem 来安装 gems、用 brew 来安装那些依赖包。”

# 二、Homebrew安装
 在终端中输入以下命令
```python
usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

![终端返回的结果1](https://planeswalker23.github.io/images/posts/2018040601.png)

按照他的要求按下RETURN(即回车)然后等待它自己安装完成就可以了。
事实上我装了好多次，都没有成功，有时候是没有连接到某地址，有时候是直接返回false、error，多试几次然后就行了。当终端出现以下信息时，就是说安装完成了。

![终端返回的结果2](https://planeswalker23.github.io/images/posts/2018040602.png)

然后就可以用下面的代码测试Homebrew是否安装成功。
```
brew
```
如果安装成功，会返回下面的命令。

![终端返回的结果3](https://planeswalker23.github.io/images/posts/2018040603.png)

Ps：Homebrew的卸载请看参考中第一条。

# 三、用Homebrew安装Git
只需要输入一行命令:
```
brew install git
```
接下来就是漫长的等待，好像要很久，把我给等困了，安装完成会返回下面的界面。

![终端返回的结果4](https://planeswalker23.github.io/images/posts/2018040604.png)

# 四、配置用户名和用户邮箱
该信息以后与Git交互时需要用到。
```
git config --global user.name "your_name"  
git config --global user.email "your_email@gmail.com"
```
Ps1：使用`git config --list`来查看Git的配置信息；
Ps2：使用`git config core.ignorecase false`设置Git为大小写敏感。

# 五、生成密钥
Git关联远端仓库时候需要提供公钥，本地保存私钥，每次与远端仓库交互时候，远端仓库会用公钥来验证交互者身份。
使用以下指令生成密钥。
```
ssh-keygen -t rsa -C "your_email@youremail.com"
```
在`/Users/当前电脑用户/.ssh`目录下会生成两个文件`id_rsa`、`id_rsa.pub`，`id_rsa`文件保存的是私钥，保存于本地，`id_rsa.pub`文件保存的是公钥，可于远端仓库添加公钥。

# 六、向远端仓库添加密钥
首先获取公钥
> 在终端中输入`cd /Users/当前电脑用户/.ssh`进入该目录
> 输入`cat id_rsa.pub`指令，查看`id_rsa.pub`文件中的内容

![这里写图片描述](https://planeswalker23.github.io/images/posts/2018040605.png)

然后打开自己的Github，鼠标移动到右上角自己的头像，点击`Settings`->`SSH and GPG keys`。

![这里写图片描述](https://planeswalker23.github.io/images/posts/2018040606.png)

然后点击`New SSH Key`，`Title`中的内容可任意编辑，`Key`中填写刚刚获取的`id_rsa.pub`文件中的内容，全部复制粘贴进去，然后点击`Add SSH Key`就添加公钥完成了。

接下来就可以往Github里创建仓库并与本地仓库同步了，具体步骤请移步：[MacOS如何将本地项目同步到Github](https://blog.csdn.net/qq_29401491/article/details/79834271)

# 参考
[安装卸载homebrew](http://www.cnblogs.com/chenjunbiao/archive/2011/07/11/2102899.html%20%E5%AE%89%E8%A3%85%E5%8D%B8%E8%BD%BDhomebrew)

[Homebrew简介和基本使用](https://blog.csdn.net/andanlan/article/details/51589800#fn:xcode)

[添加远程库 - 廖雪峰的官方网站](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/0013752340242354807e192f02a44359908df8a5643103a000)

[Homebrew macOS 缺失的软件包管理器](https://brew.sh/index_zh-cn.html)

> 版权声明：本文为[Planeswalker23](https://github.com/Planeswalker23)所创，转载请带上原文链接，感谢