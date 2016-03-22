---
title: HEXO-USE
date: 2016-01-14 11:05:18
tags: GIT
---

## 安装node.js  

	qq微云中有

## 安装git
	官网上下载,要设置path, 否则hexo deploy失败

## 安装hexo

	npm install -g hexo-cli

## 博客hexo工作目录创建

	mkdir blog
	cd blog
	hexo init
	npm install
	创建出几个目录
	source 下面放源文件md
	public  hexo genrate生成静态文件在下面, hexo deploy的时候上传到github
	themes  hexo genrate生成静态文件,会使用这里主题,这里可能需要自定义
	
## 自定义设置

	修改banner高度,文件位置:
	\themes\landscape\source\css\_variables.styl
	// Header
	logo-size = 10px
	subtitle-size = 16px
	banner-height = 80px
	banner-url = "images/banner.jpg"
	修改jquery位置,google cnd报错
	\themes\landscape\layout\_partial\after-footer.ejs
	<script src="/js/jquery.js"></script>
	需要将jquery.js文件放到source/js文件夹下

## 配置git地址

	type: git
	repo: https://github.com/qifanyang/qifanyang.github.io.git
	branch: master
	需要配置sshkey,将pub key放入github中
	

	



