---
title: Blog搭建总结
date: 2026-01-20 00:06:00 +0800
categories: [随手记, 博客搭建]
tags: [Blog，环境配置]
pin: false
math: false
mermaid: false
---


选择了[**Chirpy**](https://github.com/cotes2020/chirpy-starter)这个主题，参照[**Getting Started**](https://chirpy.cotes.page/posts/getting-started/)的教程进行配置，基本流程如下：

## 首次部署
### 创建仓库
1. 访问GitHub上Chirpy的[**starter**](https://github.com/cotes2020/chirpy-starter)仓库
2. 点击右上角的绿色按钮<kbd>Use this template</kbd>，然后选择<kbd>Create a new repository</kbd>
3. 使用`username.github.io`的格式给仓库命名

### 设置部署方式
Chirpy需要编译才能运行，所以不能用默认的部署方式，必须改用 **GitHub Actions**，采用以下步骤：
1. 进入刚刚创建好的仓库，点击上方的<kbd>Settings</kbd>
2. 点击**Code and automation**下的<kbd>Pages</kbd>按钮
3. 将**Build and deployment**下面的**Source**选项设置为<kbd>GitHub Actions</kbd>

### 等待自动运行
1. 点击仓库上方的<kbd>Actions</kbd> 标签页。
2. 看到一个正在转圈的任务，名字通常叫**Build and Deploy**（对应标题**Initial commit**）
3. 等待它变成绿色对勾时，博客渲染完成，可直接通过浏览器访问`https://username.github.io`

## 本地环境搭建

### 环境准备
* WSL 2
* Docker Desktop（选择好要使用的发行版，这里使用的**Ubuntu22**，充当**Client**）
* VS Code **Dev Containers**插件

### 容器构建
1. 在**Ubuntu22**中clone刚刚创建的仓库，然后用VS Code打开
2. 打开后检测到`.devcontainer`文件夹，然后选择<kbd>Reopen in Container</kbd>
3. VS Code会指挥Docker启动一个新的容器，并将刚刚clone的仓库目录挂载到新容器里，VS Code重新连接到这个**新容器**内部
4. 左下角状态栏变成：**Dev Container: Jekyll**

### 运行
此时已经位于容器内部的Shell了，``Ctrl + ` `` 调出终端，然后运行：
```shell
bundle exec jekyll serve
```
然后访问 `localhost:4000` 即可。

## 博客创建

## 代码提交


