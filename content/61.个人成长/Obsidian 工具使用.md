---
title: Obsidian 工具使用
draft: false
tags:
  - 个人
date: 2025-07-03
---
# 引子

一直以来，个人知识记录采用的工具是 `notion`，坚持 `all in one notion` 4年有余最近发现了 `noiton` 的一些缺陷：

- 网络问题难以解决。
	- 老生常谈，`notion` 到现在还没国内服务器，有时候不挂梯子访问 `notion` 甚至会被墙
	- `notion` 不支持离线模式 (据说在测试了)
- 对于长篇文章的体验不好。
	- 当单篇文章过长时 `notion` 客户端会明显的卡顿
	- 当你尝试将一篇文章分为多个 `page` 去写作时你会发现来回切换的体验感并不好
- 数据并不在自己手中。
- 不利于搭建个人知识库。
	- 即使许多知识库已经支持了从 notion 导入数据，但体验下来并不好。不如本地的 `md` 文件。


于是想着搜寻一款适用于记录个人笔记，搭建个人知识库的工具。 `obsidian` 引起的我的关注。

- 开源，具有丰富的社区插件（第三方主题、各种功能插件）
- 完全本地化，`md` 的源文件格式对程序猿友好



当然 `notion` 并没有那么不堪：
 
- 它的颜值在众多的笔记软件中依旧吸睛（个人观点）。
- `notion` 支持多人协作，利用多视图的数据库可以玩起各种骚操作（团队协作、任务看板、攻略、个人管理。。。）
- `notion api`、`notion 脚本`、 `notion 日历`。。。



# 自定义配置

## 快捷键



## 插件

### Git

功能：利用 `git` 来管理 `value` 实现对笔记的版本控制。

### Image converter

功能：对粘贴到笔记的图片重命名、设置目录；实现对图片的裁剪、涂鸦、旋转等操作。

### Templater

功能：更强的模板文档管理。

### Style setting

功能：主题管理，控制自定义主题的各种色彩、字体。


## 主题

### Underwater

`underwater` 主题。 



# 搭建博客系统

## 使用到的内容：

- Git： 版本管理工具
- Github
- Quartz 4：[官网地址](https://quartz.jzhao.xyz/plugins/CustomOgImages)
- Node
- Vercel：托管网站

## 实现步骤

1. `fork` 官方项目：[jackyzha0/quartz: 🌱 a fast, batteries-included static-site generator that transforms Markdown content into fully functional websites](https://github.com/jackyzha0/quartz)
2. 拉取代码
3. 引入依赖，创建工程
	
	```bash
	npm i
	npx quartz create
	```

4. `obsidian` 打开 `docs` 文件夹作为仓库，并编写一个测试文档。
5. 本地运行预览

	```bash
	 npx quartz build --serve
	```

6. 配置 `git` 插件，将 `.git` 文件设置为上层目录

	![[Obsidian 工具使用-1751520005624.jpg|489x404]]
7. 项目根路径下编写 vercel.json 文件
	```
	{ "cleanUrls": true }
	```
	
8. `git` 提交编写的文档

	![[Obsidian 工具使用-1751520067398.png]]

9. 登录 vercel 平台，导入 git 账号下的 fork 项目
10. 配置项目名与其他配置
	![[Obsidian 工具使用-1751520246708.png]]

## 相关配置

关于博客系统的配置在项目根路径下的 `quartz.conf.ts` 文件下，具体的配置项目可参考官方文档：[Configuration](https://quartz.jzhao.xyz/configuration)


## 备注

- `oss` 上的图片在打包后不能预览，可能是跨域原因
- `obsidian` 中的文件夹不能排序，第三方的插件即使实现了排序在 `quartz` 可能也会出现问题，于是暂时采用编号的形式控制排序。



