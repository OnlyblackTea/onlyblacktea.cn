---
title: 我的LINUX初始化环境配置
tags: []
id: '299'
categories:
  - - 随笔
abbrlink: b4677c6b
date: 2022-07-02 17:27:12
---

	% 我的Linux初始化环境配置
	% Onlyblacktea
	% 2022-07-02 17:27:12

从2019年起我就逐渐开始尝试使用Linux系统了，回想起我最早装ubuntu的日子，由于驱动问题导致在安装过程中死机、以及各种装输入法不成功、还有sudo apt remove python导致系统崩溃等等。后来我在腾讯云租了一台学生机，通过没有桌面环境的系统（不怎么容易崩）配了现在的这个网站，以及当时需要Linux环境做的一些开发。再到后来就是经过推荐后使用了KDE桌面环境，在工位上装了Kubuntu 21.10，一直是根据需求在做配置，用到今天。大三结束以后，我在家心血来潮安装了数个不同的Linux发行版，便觉Linux世界之广阔，而后开始思考：我如何做一套流程，以便为我下次安装新系统时配一套初始的开发环境。

### 一、zsh

安装zsh的方法比较多，可以根据具体的环境选择合适的方法：

*   [通过包管理器安装](https://github.com/ohmyzsh/ohmyzsh/wiki/Installing-ZSH#how-to-install-zsh-on-many-platforms)
*   [从源码编译安装](https://zsh.sourceforge.io/FAQ/zshfaq01.html#l7)（[source](https://zsh.sourceforge.io/Arc/source.html)）

安装完zsh以后，通过指令一键安装oh-my-zsh。当然也可以安装其他的框架，omz只是我用得比较熟悉的：

```
$ sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

简约风可以试试half-life主题，带个入字信仰：

```
λ omz theme set half-life
```

再推荐一个[p10k](http://www.github.com/romkatv/powerlevel10k)，可以根据github中的指示进行安装。非常好看！

然后装几个插件：

```
# zsh-syntax-highlight:
λ git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
λ omz plugin enable zsh-syntax-highlighting

# zsh-autosuggestions:
λ git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
λ omz plugin enable zsh-autosuggestions

# others
λ omz plugin enable extract
λ omz plugin enable z
```

### 二、neovim

neovim与vim不同，它推荐采用lua来配置。可以通过包管理器进行安装，也可以从源码编译安装。  
访问[neovim的github主页](https://github.com/neovim/neovim)来下载release。

* [一个轻量化的nvim配置](https://github.com/nvim-lua/kickstart.nvim)
* [ayamir的nvimdots](https://github.com/ayamir/nvimdots)

neovim各个插件的版本由~/.config/nvim/lazy-lock.json描述，在插件管理器更新插件时，它会采取类似paru -Sy，paru -Su的过程，先更新lazy-lock.json，再根据该文件对插件进行pull。因此如果插件更新以后出现问题，通过还原lazy-lock.json以后再去插件管理器对应commit对各个插件重新pull一下即可。

目前对于上述nvim的配置方案我还是处于即开即用状态。关于后续的使用和调整会新开一篇文章来讲。

#### 参考文档：

1.  Oh-My-Zsh：[https://github.com/ohmyzsh/ohmyzsh/wiki](https://github.com/ohmyzsh/ohmyzsh/wiki)
2.  neovim：[https://github.com/neovim/neovim/wiki](https://github.com/neovim/neovim/wiki)