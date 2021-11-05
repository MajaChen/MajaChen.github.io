---
layout: post
title: 基于vscode 和linux构建golang 开发环境
subtitle: golang vscode linux
cover-img: /assets/img/5.jpg
thumbnail-img: /assets/img/animals/LoepaOberthuri.jpg
share-img: /assets/img/path.jpg
tags: [go]

---


# 基于vscode 和linux构建golang 开发环境

&emsp;&emsp;**写在前面的话：磨刀不费砍柴功，构建一个“无缝”开发环境能真正提升开发效率。**我这里构建的golang开发环境是基于linux，代码的编辑、调试、运行都发生在linux服务器上。而编辑器vscode运行在windows侧，vscode基于ssh协议连接远程linux服务器，开发、调试远程代码。

![image-20210725140945141](https://gitee.com/xinyuanchen/image_collection/raw/master/image-20210725140945141.png)

## 1.windows侧安装vscode并连接远程linux服务器

&emsp;&emsp;到[这个地址](https://code.visualstudio.com/)下载vscode安装包并安装vscode。vscode安装成功之后，利用Remote-SSH扩展接远程服务器，操作步骤见[这篇博客]()，我在此过程中先后遇到并解决了两个问题，见文章末尾的WIKI。完成这些操作、解决这些问题之后，vscode可以正常地连接远程服务器，但是每次都需要输入密码，这不符合“无缝”要求。可以通过密钥解决这个问题。操作步骤可以参考[这篇博客]()。

## linux侧安装goalng和开发插件

&emsp;&emsp;按照[这篇文章]()的指导在linux服务器上安装golang，然后安装开发golang程序必需的插件，比如dlv——用于调试golang程序、godef——用于golang程序的跳转、gopls——用于golang程序代码补全等。但是不必大费周章自己一个一个安装，如果上一步vscode可以连接远程linux服务器，先安装一个扩展——Go

![image-20210725121503335](https://gitee.com/xinyuanchen/image_collection/raw/master/image-20210725121503335.png)





&emsp;&emsp;这款扩展安装成功之后，屏幕地右下角会出现一个对话框，要求你安装所有golang开发必备的插件，点击Install,扩展会自动地调用go get命令下载并安装插件，可以在输出窗口查看安装信息：

![golang 工具](https://gitee.com/xinyuanchen/image_collection/raw/master/golang 工具.png)

&emsp;&emsp;我们看到这些插件都被安装到/home/chxy/go/bin目录下，这是由环境变量GOPATH决定的。**插件安装完成之后切换到终端，会发现命令一个都找不到。那vscode能将这些插件集成到IDE吗？事实上是可以的。**若实在想在命令行运行这些插件以求心安，可以把/home/chxy/go/bin路径加入path，然后**重启linux服务器。**

## 配置git，托管代码

&emsp;&emsp;可以通过vscode的界面把代码托管于git，而不是通过git命令行。当本地仓库建好之后，vscode可以主动识别.git文件，任何本地仓库的更改都会反映在vscode的版本控制界面，如果本地仓库已经关联好远程仓库，还可以直接点击Push按钮把代码发布到远程仓库：

![image-20210725134254144](https://gitee.com/xinyuanchen/image_collection/raw/master/image-20210725134254144.png)



&emsp;&emsp;想了解配置git仓库的细节可以参考[这篇文章]()的git章节。

## 运行主程序和进行单元测试

&emsp;&emsp;有了基于vscode的集成开发环境，我们再也不用通过命令行来调试和运行golang程序了，点击运行和调试按钮即可。但是该功能依赖于launch.json配置文件，首次运行需要先创建这个文件。该文件内容非常简单，最主要的指定main函数所在的文件，我们可以在.vscode文件夹中找到这个文件。

![image-20210725112724048](https://gitee.com/xinyuanchen/image_collection/raw/master/image-20210725112724048.png)

&emsp;&emsp;对某些函数我们希望运行单元测试以验证其功能的正确性。golang提供了go test命令行来实现单元测试。但是有了vscode，只要我们遵守go test命名规范(测试文件以_test结尾，测试函数以Test开头)，那么这个命令行也可以省去。

![image-20210725135543277](https://gitee.com/xinyuanchen/image_collection/raw/master/image-20210725135543277.png)

## 7.环境验收

&emsp;&emsp;实现以下全部功能则代表无缝开发环境构建完成：代码补全/代码跳转/代码调试/界面运行主函数/界面执行单元测试/界面管理git仓库。

## WIKI

&emsp;&emsp;记录在构建和使用golang无缝开发环境中遇到的问题和解决过程。

&emsp;&emsp;1.vscode连接远程服务器时遭遇了两个问题：<br>
&emsp;&emsp;问题1：vscode报“vscode Server installation process already in progress - waiting and retrying”问题。<br>
&emsp;&emsp;解决方案：进入远程服务器的/home/{username}这个文件夹，比如，我是以chxy这个用户连接远程服务器的，那么就进入/home/chxy这个目录，删除.vscode-server这个文件夹，再重新连接，问题解决。

![image-20210725113831353](https://gitee.com/xinyuanchen/image_collection/raw/master/image-20210725113831353.png)

&emsp;&emsp;问题2：vscode报“Error: Running the contributed command: ‘_workbench.downloadResource‘ failed”错误。<br>
&emsp;&emsp;解决这个问题的步骤比较繁琐，网上这篇博客给出了非常详尽的解决方案，在此我不再复述。方案的其中一步要求查看vscode的版本是stable还是insider，我按照它给的指示没有操作成功，所以就默认使用stable版本下载对应文件，结果证明这对问题的解决没有影响。

