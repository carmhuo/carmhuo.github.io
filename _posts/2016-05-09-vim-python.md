---
layout: post
title: Vim下配置Python开发环境
subtitle: 备忘录
author: carm
header-img: img/home-bg.jpg
categories: linux
tag:
  - vim
---
# Vim配置Python开发环境

### 系统环境
* ubuntu15.10
* vim7.4
* python2.7
> 确保vim支持python，查看方式 `vim --version | grep python` ,若出现`+python`字样，则vim支持python

### 安装Vundle插件管理器

Vundle其特色在于使用git来管理插件,更新方便，支持搜索，一键更新，从此只需要一个vimrc走天下。更换机器时，只需安装Vundle，并使用原先的配置，即可配置好vim

项目托管在github上 [https://github.com/gmarik/vundle](https://github.com/gmarik/vundle)

##### 安装命令
	
	$ git clone http://github.com/gmarik/vundle.git ~/.vim/bundle/vundle
  
### 配置
	touch ~/.vimrc

添加内容：

	set nocompatible              " required
	filetype off                  " required
	  
	" set the runtime path to include Vundle and initialize
	set rtp+=~/.vim/bundle/Vundle.vim
	call vundle#begin() 
	" alternatively, pass a path where Vundle should install plugins
	"call vundle#begin('~/some/path/here')
	 	
	" let Vundle manage Vundle, required
	Plugin 'gmarik/Vundle.vim'
	 
	" Add all your plugins here (note older versions of Vundle used Bundle instead of Plugin)
	Plugin 'vim-scripts/indentpython.vim'
	Bundle 'Valloric/YouCompleteMe'
	Plugin 'scrooloose/syntastic'
	Plugin 'scrooloose/nerdtree'
	Plugin 'jistr/vim-nerdtree-tabs'
	Plugin 'tmhedberg/SimpylFold'
	 
	let NERDTreeIgnore=['\.pyc$', '\~$'] "ignore files in NERDTree
	let g:SimpylFold_docstring_preview=1

	" All of your Plugins must be added before the following line
	call vundle#end()            " required
	filetype plugin indent on    " required
	  
	"split navigations
	nnoremap <C-J> <C-W><C-J>
	nnoremap <C-K> <C-W><C-K>
	nnoremap <C-L> <C-W><C-L>
	nnoremap <C-H> <C-W><C-H>

	set encoding=utf-8
	set nu
	let python_highlight_all=1
	syntax on

接着打开vim,运行命令`:PluginInstall`

##### 说明：
* vim-scripts/indentpython.vim：自动缩进插件
* Valloric/YouCompleteMe：自动补全插件
* scrooloose/syntastic：语法检查/高亮
* scrooloose/nerdtree：文件树形结构
* jistr/vim-nerdtree-tabs：tab键
* tmhedberg/SimpylFold：代码折叠

##### 手动配置自动补全功能
github中有安装文档，可进行参考。[YouCompleteMe: a code-completion engine for Vim](https://github.com/Valloric/YouCompleteMe)

### 结束
抽点时间，写写文章，以备日后需要

### 参考文章：
1. [Vim与Python真乃天作之合](http://codingpy.com/article/vim-and-python-match-in-heaven/)







