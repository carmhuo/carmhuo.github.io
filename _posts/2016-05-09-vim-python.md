---
layout: post
title: 配置Vim
subtitle: Vim下配置Python开发环境
author: carm
header-img: img/home-bg.jpg
categories: linux
tag:
  - vim
  - linux
  - python
---
# Vim配置Python开发环境

### 系统环境
* ubuntu15.10
* vim7.4
* python2.7

> 确保vim支持python，查看方式 `vim --version | grep python` ,若出现`+python`字样，则vim支持python

### 安装Vundle插件管理器

Vundle其特色在于使用git来管理插件,更换机器时,只需安装Vundle,并使用原先的配置，即可配置好vim

项目托管在github上，链接：[https://github.com/gmarik/vundle](https://github.com/gmarik/vundle)

##### 安装命令
	
	$ git clone https://github.com/VundleVim/Vundle.vim.git ~/.vim/bundle/Vundle.vim
  
### 配置vim
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
	Plugin 'VundleVim/Vundle.vim'
	 
	" Add all your plugins here (note older versions of Vundle used Bundle instead of Plugin)
	
	Plugin 'vim-scripts/indentpython.vim'
	Plugin 'scrooloose/syntastic'
	Plugin 'scrooloose/nerdtree'
	Plugin 'jistr/vim-nerdtree-tabs'
	Plugin 'tmhedberg/SimpylFold'
	"Plugin 'git://git.wincent.com/command-t.git'
	"Plugin 'Valloric/YouCompleteMe'
	
	" All of your Plugins must be added before the following line
	call vundle#end()            " required
	filetype plugin indent on    " required
	
	let NERDTreeIgnore=['\.pyc$', '\~$'] "ignore files in NERDTree
	let g:SimpylFold_docstring_preview=1
	
	"split navigations
	nnoremap <C-J> <C-W><C-J>
	nnoremap <C-K> <C-W><C-K>
	nnoremap <C-L> <C-W><C-L>
	nnoremap <C-H> <C-W><C-H>

	set encoding=utf-8
	set nu
	let python_highlight_all=1
	syntax on
	
	"Enable folding
	set foldmethod=indent
	set foldlevel=99
	" Enable folding with the spacebar
	nnoremap <space> za

	"在 vim 启动的时候默认开启 NERDTree（autocmd 可以缩写为 au）
	"autocmd VimEnter * NERDTree
	"按下 F1 调出/隐藏 NERDTree
	map <F1> :NERDTreeToggle<CR>
	"将 NERDTree 的窗口设置在 vim 窗口的右侧（默认为左侧）
	"let NERDTreeWinPos="right"
	"当打开 NERDTree 窗口时，自动显示 Bookmarks
	let NERDTreeShowBookmarks=1
	"不显示帮助面板
	let NERDTreeMinimalUI=1 
	
接着打开vim,运行命令`:PluginInstall`

##### 说明：
* vim-scripts/indentpython.vim：自动缩进插件
* Valloric/YouCompleteMe：自动补全插件
* scrooloose/syntastic：语法检查/高亮
* scrooloose/nerdtree：文件树形结构
* jistr/vim-nerdtree-tabs：tab键
* tmhedberg/SimpylFold：代码折叠

##### YouCompleteMe配置详细过程
YouCompleteMe项目托管在github上，且有详细安装文档，参考链接:[YouCompleteMe: a code-completion engine for Vim](https://github.com/Valloric/YouCompleteMe#full-installation-guide)

### 结束
抽点时间，写写文章，以备日后需要。

### 参考文章：
1. [Vim与Python真乃天作之合](http://codingpy.com/article/vim-and-python-match-in-heaven/)
2. [VIM NERDTree 插件配置](https://www.douban.com/note/225250638/)







