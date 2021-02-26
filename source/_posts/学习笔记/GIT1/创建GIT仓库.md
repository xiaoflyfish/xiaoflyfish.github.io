---
title: 创建GIT仓库
categories: 
- 学习笔记
- GIT1
---

**创建版本库**

使用`git init`命令初始化一个仓库了

初始化版本库会在当前目录中一个`.git`的文件夹，然后通过命令`ls -al`进入版本库根目录查看

可以通过命令：`cd .git && ls -al`进入 `.git `文件夹中查看

**查看配置信息**

```
git config user.name
```

```
git config user.email
```

**设置配置信息**

```
git config --global user.name "你的昵称"
```

```
git config --global user.email "你的邮箱"
```

不能通过重复执行上面的设置昵称命令，来修改昵称的，邮箱修改同理

修改的时候，可以通过特定的方式去修改，介绍两种方法， 第一种是通过命令行，第二种是通过修改配置文件

**命令行修改配置**

```
git config --global --replace-all user.name "your user name"
```

```
git config --global --replace-all user.email"your user email"
```

**修改配置文件**

修改文件的方式，主要是修改位于主目录下` .gitconfig` 文件。在 Linux 和 Mac 中，可以通过 vim 命令进行直接编 辑

比如 `vim ~/.gitconfig`

如果之前已经配置过昵称和邮箱的情况下，当使用 vim 或者记事本打开配置文件之后，可以看到如下配置：

```
[user]
	name = daxia 
	email = 78778443@qq.com
```

在如果有重复的 name 或 email，可以将其删掉，只剩下一个就好

修改完，通过 `git bash` 输入 `git config –list` 可以 查看是否修改成功了