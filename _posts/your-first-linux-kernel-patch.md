---
title: 你的第一个Linux内核补丁
date: 2019-04-18 10:17:15
tags: [Technics, Systems]
categories: cs
---

这是一个linux内核补丁提交教程，其中有不少内容来自[Kernel Newbies](https://kernelnewbies.org/)，也有部分自己创作的。
书籍推荐: [Linux Device Drivers](http://lwn.net/Kenel/LDD3/)

<!-- more -->
## 环境配置&初步编译

### 简介

这个教程的目的是教会你提交linux内核补丁 (patch)。为了成功编译linux kernel，你需要准备好以下工具：

Ubuntu/Debian:
```shell
sudo apt-get install vim libncurses5-dev gcc make git exuberant-ctags libssl-dev bison flex libelf-dev bc
```

ArchLinux/Manjaro:
```shell
sudo pacman -S vim gcc make git ctags openssl bison flex libelf bc
```
至此你已经安装好: vim, gcc, make, git, ncurses, libssl, bison, flex, libelf和bc.

### 建立你的Linux Kernel仓库

当安装完毕时，运行：
```shell
mkdir -p git/kernels; cd git/kernels

git clone -b staging-testing git://git.kernel.org/pub/scm/linux/kernel/git/gregkh/staging.git
```
这里我clone的是Greg KH的linux树，你也可选择其他维护者的linux树 (比如[Linus](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/)本人的).

Linux kernel更新很快，你也许想要常常
```
git fetch origin

# and check the patches applied:
git log --pretty=oneline --abbrev-commit origin 
# or any other branches you've been working on
```
并rebase你目前的branch
```shell
git rebase your-branch
```

此刻你也许想要configure你的kernel，步骤在下面的`config文件`里


### config文件

如果你想测试你是否成功修改某个bug，你也许想要把config文件复制到你正在运行的kernel上，这个config文件在`/boot`的某处，你可以用`uname -a`来找到那个config文件 (它应当以你的kernel版本结尾)，将那个文件复制到你的源文件目录里：

Ubuntu/Debian:

```shell
cp /boot/config-`uname -r`* .config 
```
ArchLinux/Manjaro:
```shell
zcat /proc/config.gz > .config
```
不要忘记把你的kernel版本 "CONFIG_LOCALVERSION" 在新的.config 重命名 . 如果你跳过了这个，你可能会覆盖你现有的kernel信息。

如果你需要改变configuration文件，运行这条命令将configuration到它们的默认值：
```shell
make olddefconfig
```

如果要改变编译信息 (进入GUI界面)：
```shell
make menuconfig
```

为了编译更快进行，你可以不编译你系统中当前没有使用的模块：
```shell
make localmodconfig
```

### 编译kernel

```shell
make -jn
```

安装kernel:
```shell
sudo make modules_install install 
```

运行kernel:
```shell
sudo vim /etc/default/grub 
```

(如果你的grub里有这些，你需要删除它们：)
```
GRUB_HIDDEN_TIMEOUT=0
GRUB_HIDDEN_TIMEOUT_QUIET=true
```

你会想把grub的timeout改长一点 (这样你可以在grub停留更长时间)：
```
GRUB_TIMEOUT=60
```

请确保：
```
GRUB_TIMEOUT_STYLE=menu
```

恭喜，你已经到了第一阶段的最后一步：
```shell
sudo update-grub2
```
此时你也许需要重启。

## patch编译&测试

### 代码规范

Linux kernel的代码有一套自己的 (非常复杂的) [规范](https://www.kernel.org/doc/html/v4.10/process/coding-style.html)，他们写了一个perl文件[checkpatch.pl](http://lxr.free-electrons.com/source/scripts/checkpatch.pl)来做检查。你应当用它来检查你修改后的代码，不过它有时也会给出false positive的警告 (例如一些“一行超过80字” —— 实际上断开更不利于阅读，“此处使用volitale一般是错误的” —— 实际上并无错误)，你需要自行判断。

```shell
perl scripts/checkpatch.pl -f the/path/to/your/file | less
```

**小建议**：

如果你不知道从何下手，你可以试着看看`drivers/staging`，它充满了需要被修改的代码，运行这个命令来看看在todo-list的任务有哪些：

```shell
find drivers/staging -name TODO
```

### 编译

如果你已经做了代码修改，在编译你的新kernel之前，你也许想要在
```
make menuconfig
```
里把你修改过的模块作为Module编译。(注：可以使用'/'来搜索你改动的地方，如果你在改动drivers，记得要把编译选项改为模块 ('M')，而不是内置 ('\*')，退出时记得保存你修改的configuration.)

如果你修改的部位在这里找不到，搜索它 ('/')，查看它的依赖，你需要递归地把它的依赖设置为'm' (instead of 'n').

如果你想知道，举个例子，`xhci-hcd`是什么文件编译成的，你可以运行这条命令来查看：
```shell
git grep xhci-hcd -- '*Makefile'
```
开始编译：

```shell
make -jn
```
请注意：引起warning的patch是不会被接受的。

编译太慢了？你可以只编译kernel的一部分：
```shell
make drivers/staging/ 
# does not link the modules

make M=drivers/staging 
# links modules in previously build vmlinux file: might fail when .config is changed or a newer git tree is rebased
```

**安装**：
```shell
sudo make modules_install install
```

### 测试
你已经编译好了一个新内核了！此时你需要重启。然后运行 (如果你改动的是/drivers)：
```shell
modprobe name-of-your-driver # load the driver
lsmod | less # see the drivers loaded
```

那么如何知道`name-of-your-driver`应该是什么？运行：
```shell
ls drivers/staging/your-path/*.ko
```
看看输出的名字是什么。

另：如果你运行的kernel和你修改的kernel的版本一样，你测试时可以不重启，你可以安装/lib/modules里的模块，unload再reload它：
```shell
make -jn && sudo make modules_install
sudo modprobe -r <module_name>
sudo modprobe <module_name> 
lsmod | less
```

**回滚**

Oops! 你的patch有问题？你也许想回滚到这个patch开始之前：
```shell
git reset --hard HEAD
# HEAD can be replaced any other commit hash
```

## 发送你的patch

你已经来到了最后一步：发送你的补丁。

### email软件

我们推荐你使用git-email或是mutt来发送你的patch：
```shell
sudo apt-get install esmtp
sudo apt-get install mutt
# or
sudo apt-get install git-email
```
你也许想要把vim作为你的默认text editor:
```shell
sudo update-alternatives --config editor
```
选择` /usr/bin/vim.basic`作为你的默认text editor.

(下面是一些vim格式调整，它能方便你编辑，统一的patch格式也方便邮件列表中其他维护者查看你的patch:)

```shell
vim ~/.vimrc
```

```
filetype plugin indent on
syntax on
set 

set tabstop=8
set softtabstop=8
set shiftwidth=8
set noexpandtab
```
### Gmail设置

在`Setting`里面的`Forwarding POP/IMAP`: 选择`Enable IMAP`并保存。

点击页面底部的"Configuration instructions"，把"Step 2"里的server信息复制到你的.esmtprc文件里 (我们用esmtp来发送邮件).

如果你的gmail邮箱开启了两步验证，你需要用到[App Password](https://support.google.com/accounts/answer/185833?hl=en).

### emstp设置

(你可以选择其他的MTA).
首先创建.esmptrc:

```shell
touch ~/.esmtprc
chmod g-rwx ~/.esmtprc
chmod o-rwx ~/.esmtprc
```
编辑这个文件：
```shell
identity "my.email@gmail.com"
hostname smtp.gmail.com:587 # or other mail server
username "my.email@gmail.com"
password "ThisIsNotARealPassWord"
starttls required 
```
(我的密码里有`'`，这给我造成了很大麻烦——我花了不少时间确定问题*居然*出在我的gmail密码上，如果你的密码也有这样的特殊字符，记得加'\').

然后设置git： 在.gitconfig里加上：

```shell
[user]
   name = Your Name
   email = your.email@example.com 

[sendemail]
   smtpserver = /usr/bin/esmtp
```

请注意，在这里的姓名必须是你的**原名** (有法律效益的)，不是你的github用户名或是网名。这将会填写你的patch中的'sign-off-by'栏，详情请见[Developer's Certificate of Origin](https://elixir.bootlin.com/linux/latest/source/Documentation/process/submitting-patches.rst).

### git post-commit钩 (hook)

Linux kernel的开发者对patch的格式也有要求 (They're so picky!)，所以你需要一个git post-commit hook.

如果你已经有一个`.git/hooks/post-commit`了，请把它重命名为`.git/hooks/post-commit.old`，然后创建一个新的：
```
#!/bin/sh
exec git show --format=email HEAD | ./scripts/checkpatch.pl --strict --codespell
```
然后:
```shell
chmod a+x .git/hooks/post-commit
```

(如果你没有/usr/share/codespell/dictionary.txt，请运行：)
```shell
apt-get install codespell
```

Linux kernel的补丁有自己的格式要求，**在你开始创建你的pactch之前，请阅读[Linux内核补丁的哲学](https://kernelnewbies.org/PatchPhilosophy)**，[这个](https://etsai.site/2019/03/11/philosophy-of-linux-kernel-patches/)是我翻译~~了一半~~的中文版。

### Alas

如果你在墙内，你可能需要通过ss (这里就不写怎么配置ss了) 来发送你的邮件，比如使用`proxychains`：

```shell
sudo apt-get install proxychains
```
配置好：
```shell
vim /etc/proxychains.conf

[ProxyList]

socks5 127.0.0.1 1080 ## Set socks5 proxy to your sslocal your_ip/port
```
然后使用类似下面的命令来发送你的patch:
```shell
proxychains git send-email ...
```

~~我之前用的light，一直发不出去，馒头一语点醒我：light代理的是http&https，邮件用的协议是smtp，机智的馒头！~~


### 关于邮件格式

注意：当你回复linux kernel相关邮件列表中的邮件时，请保证你的邮件风格与社区一致，使用这些["被批准的邮件客户端"](https://www.kernel.org/doc/html/v4.12/process/email-clients.html)。 **不要使用gmail 网页端**，它会给引用换行；**不要使用outlook**，它会搞乱patch，把tab换成空格 (outlook developers: 空格党的胜利).

(我发送了十来个patch之后才注意到这个:o，我注意到了每个人的引用和我的gmail客户端都不一样，可我没想过为什么……幸运的是大家都很友善 and nobody scold me for that :P)

**请使用inline回复**，而不是top-posting.
inline (把新内容回复到相应的文字下面):
```
> > Hi. I just submit a patch to fix the first __ATTR_NULL, but I have some 
> question about the latter one.
> I understand it is better to use read only attribute here, but by changing 
> __ATTR to __ATTR_RO, since the new macro does not need the _show_function  
> parameter:
> 
> > (linux/sysfs.h, line 115)
> > #define __ATTR_RO(_name) { \
> > .attr = { .name = __stringify(_name), .mode = 0444 }, \
> > .show = _name##_show, \
> > }
> 
> it seems to me that the second parameter in this macro should be deleted, 
> which means all other file which calls this macro should be revised. 
> I think it's a big code change, and don't know whether it's suitable to do 
> this. Could you give me some advice?

Yes, it is a big code change, and if you are not comfortable with it, I
would not recommend it at this point in time.

This is also why no one has done it yet :)

thanks,

greg k-h
```
top-posting (直接回复到最上面)：
```
Thanks greg! I'll be more careful with this next time!

On Saturday, March 9, 2019 at 2:12:36 AM UTC-8, gregkh wrote:
>That's 4 patches, which is great, but you sent out 5 patches in this 
>series :( 
>
>Odds are you had a file that ended in .patch in the directory when you 
>sent this out.  It happens to everyone :) 
>
>I'll just drop it and focus on the numbered patches. 
>
>thanks, 
>
>greg k-h 
```
I learned this one the hard way :(

### Patchset

如果你想一次提交多个patch (而且它们是相关的)，你可以把它们放进一个patchset里。 
```shell
git format-patch -n --subject-prefix="PATCH vY" --cover-letter -o /tmp/
# or you can use commit hash to do this: <hash1>^..<hash2>

git send-email /tmp/*.patch
```
patchset也要遵循其他的规则 (比如version)，你还需要编辑/tmp/里的cover-letter: 描述这个patchset做的事情。更多细节请阅读[Linux内核补丁的哲学](https://kernelnewbies.org/PatchPhilosophy)。

### 发送patch

**编写commit**
```shell
git add <modified-file>
git commit -s -v
```
请注意按照[Linux内核补丁的哲学](https://kernelnewbies.org/PatchPhilosophy)里的格式创建你的patch，**记得**要在标题和主体之前留一行空白，主体结束后也要留一行空白 (因为你的名字，发送时间还有邮箱会被自动加到patch里)。

运行下面这个命令来查看你的commit：
```
git show HEAD

# if you want to revise it:
git commit --amend -v

# You might want to make sure your commit looks fine
git log --pretty=oneline --abbrev-commit HEAD^
```

**维护者们**
你需要把这个patch发给你修改的部分的维护者们 (another perl file for this)：
```shell
git show HEAD | perl scripts/get_maintainer.pl --separator , --nokeywords --nogit --nogit-fallback --norolestats --nol

# or
perl scripts/get_maintainer.pl --separator , --nokeywords --nogit --nogit-fallback --norolestats --nol -f path/to/your/file
```
这个perl文件会告诉你这个patch应该发给哪些人。

**patch生成&发送**
```shell
git format-patch -o /tmp/ HEAD^
```
如果你是在修改一个已经发送过的patch，请将版本 (第y次修改版)加到标题里：
```shell
git format-patch --subject-prefix="PATCH vY" -o /tmp/ HEAD^
```

此刻你的patch被放在了/tmp/下，你可以使用mutt发送它：
```shell
mutt -H /tmp/0001-<name-of-your-patch>
```

你也可以使用git send-email用同样的方法发送/tmp下的补丁，也可直接发送HEAD:
```shell
git send-email --annotate HEAD^
```

在把你的第一个补丁发送给它的维护者之前，你也许想要先发送到自己的邮箱看看有没有问题。

**感谢你读到这里**，如果你的patch被加入原维护者的git tree里，它通常会出现在24小时之内发行的linux-next tree里。

Voila! 你的代码出现在船新版本的Linux kernel里了:)