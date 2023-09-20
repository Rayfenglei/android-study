> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_56326415/article/details/132651235)

1. 准备工作
=======

#### 1.1 下载插件

在浏览器中下载插件简悦

#### ![](https://img-blog.csdnimg.cn/9c0d8cf83da740c48005d8472e7a9db6.png)1.2 准备一个 [github](https://so.csdn.net/so/search?q=github&spm=1001.2101.3001.7020) 仓库

###### 1.2.1 为什么需要一个 github 仓库？

虽然这个插件自带了一个以 md 格式导出，我试了一下，[typora](https://so.csdn.net/so/search?q=typora&spm=1001.2101.3001.7020) 根本打不开他下载的文件（**可能是我不会用）**。所以我们先保存到 github 上，在从 github 下载到本地。

###### 1.2.2 创建仓库

打开 github，点击右上角加号，然后点击 New repository

![](https://img-blog.csdnimg.cn/1bbbeddb3e844d96b89108d31446b343.png)

给仓库起一个名字**（记住这个名字一会有用）**之后，选择创建一个 README file 文件，然后点击创建即可

![](https://img-blog.csdnimg.cn/04b317ebfc5d4ae2926a657d243485b8.png)

#### 1.3 创建 github 的 token

打开网址 [Sign in to GitHub · GitHub](https://github.com/settings/tokens/new "Sign in to GitHub · GitHub")

**Note**：随便起一个

勾选 **repo**

然后点击最下面的 **Generate token**，

![](https://img-blog.csdnimg.cn/9cd106f6dbb84ded9b8ca8f7d4d2397f.png)

保存生成的 token 信息

![](https://img-blog.csdnimg.cn/461208f3807c41db8673e350bc1c365c.png)

2. 保存文章为 md 格式到 github
======================

#### 2.1 打开文章

打开我们想要保存的文章

#### 2.2 打开插件

点击红色的插件图标

![](https://img-blog.csdnimg.cn/943c96df59a44f66b0df8b5632601212.png)

#### 2.3 给插件进行 github 授权

依次点击，点击 **保存到 GitHub** 后会出现一个授权窗口

![](https://img-blog.csdnimg.cn/601480ad9f2440e6813fff83486b6396.png)

#### 2.4  绑定 github 信息

 **第一个框**中输入我们生成的 **github token**

      **第二个框中**输入  **你的 github 用户名 / 仓库名称。**你也可以直接打开刚创建的 github 仓库，然后复制地址栏的 github.com/ 的后面的内容

      **第三个框**输入 **md**

 点击验证并绑定 Github 的信息

![](https://img-blog.csdnimg.cn/45f6f6060d9b423c9101d087d828b6d9.png)

#### 2.5 文章保存到仓库

重新做 2.3 步就可以成功保存到 github 仓库中了。

3. 将仓库中的 md 文章下载到本地
===================

#### 3.1 打开仓库

我们所有文章都保存在了 [md 文件](https://so.csdn.net/so/search?q=md%E6%96%87%E4%BB%B6&spm=1001.2101.3001.7020)夹中

![](https://img-blog.csdnimg.cn/a1dcf48222004b228f67c7b92f19cfa7.png)

#### 3.2 不 clone 仓库，直接下载压缩包

![](https://img-blog.csdnimg.cn/1520b793abb545aaaba16382b67dc76d.png)

#### 3.3 使用 clone 仓库的方式

**clone 需要下载 git，如果没有请先安装**

1. 复制这个链接

![](https://img-blog.csdnimg.cn/6033a12e48604cc183f04fe0ec13538f.png)

2. 打开 git bash 输入

> git clone 复制的网址

![](https://img-blog.csdnimg.cn/876bc3df0ff1492688d09ee0b0bcd033.png)

4. 查看文章
=======

可以看到图片、文章格式都非常好的保存了下来！

![](https://img-blog.csdnimg.cn/72ef788762354e9c85b96a42e9e2a157.png)

参考博客：[将 CSDN 或一般博客导出为 markdown 文件的通用方法_csdn 转 markdown_瑞哥 - RealWang 的博客 - CSDN 博客](https://blog.csdn.net/wangrui1573/article/details/124662720 "将CSDN或一般博客导出为markdown文件的通用方法_csdn转markdown_瑞哥-RealWang的博客-CSDN博客")