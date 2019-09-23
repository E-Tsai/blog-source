---
title: Linux内核补丁的哲学
date: 2019-03-11 10:38:22
tags: [technics, systems, CN, translated]
categories: CS
---

[英文原文](https://kernelnewbies.org/PatchPhilosophy)

<!-- more -->

## 什么是补丁

补丁 (patch) 是一个短commit (50字以内)，是对你所做修改的一个自然段的描述，一个代码的diff. 以下是一个linux kernel的文档：
```
From 2c97a63f6fec91db91241981808d099ec60a4688 Mon Sep 17 00:00:00 2001
From: Sarah Sharp <sarah.a.sharp@linux.intel.com>
To: linux-doc@vger.kernel.org
Date: Sat, 13 Apr 2013 18:40:55 -0700
Subject: [PATCH] Docs: Add info on supported kernels to REPORTING-BUGS.
One of the most common frustrations maintainers have with bug reporters
is the email that starts with "I have a two year old kernel from an
embedded vendor with some random drivers and fixes thrown in, and it's
crashing".
Be specific about what kernel versions the upstream maintainers will fix
bugs in, and direct bug reporters to their Linux distribution or
embedded vendor if the bug is in an unsupported kernel.
Suggest that bug reporters should reproduce their bugs on the latest -rc
kernel.
Signed-off-by: Sarah Sharp <sarah.a.sharp@linux.intel.com>
---
 REPORTING-BUGS |   22 ++++++++++++++++++++++
 1 files changed, 22 insertions(+), 0 deletions(-)
diff --git a/REPORTING-BUGS b/REPORTING-BUGS
index f86e500..c1f6e43 100644
--- a/REPORTING-BUGS
+++ b/REPORTING-BUGS
@@ -1,3 +1,25 @@
+Background
+==========
+
+The upstream Linux kernel maintainers only fix bugs for specific kernel
+versions.  Those versions include the current "release candidate" (or -rc)
+kernel, any "stable" kernel versions, and any "long term" kernels.
+
+Please see https://www.kernel.org/ for a list of supported kernels.  Any
+kernel marked with [EOL] is "end of life" and will not have any fixes
+backported to it.
+
+If you've found a bug on a kernel version isn't listed on kernel.org,
+contact your Linux distribution or embedded vendor for support.
+Alternatively, you can attempt to run one of the supported stable or -rc
+kernels, and see if you can reproduce the bug on that.  It's preferable
+to reproduce the bug on the latest -rc kernel.
+
+
+How to report Linux kernel bugs
+===============================
+
+
 Identify the problematic subsystem
 ----------------------------------
 
-- 
1.7.9
```
看一看这个patch的描述部分：
```
Subject: [PATCH] Docs: Add info on supported kernels to REPORTING-BUGS.
```
这句话让读者一目了然地知道这个patch的目的。通常来说描述之前会有一个前缀，让读者知道它适用于哪个子系统。在这个例子里，它指的是文档子系统。

接下来的是这个patch的目的：

……

你所有的patch描述都需要以"sign-off-by"结束。这意味着你的代码需要满足[Developer's certificate of origin](https://developercertificate.org/)，请确保你提交的代码属于你。

## 补丁格式
就像之前的例子一样，每一个git commit都有一个简短的描述，这就是patch subject line. 每一个patch都需要有一个"drive prefix" (驱动前缀)，在上面的例子里，这个前缀是"Docs"。它取决于你做了更改的内核驱动。当选择你使用的驱动前缀的时候，查看你的git log里你修改的文件所对应的前缀。

例如，当我们改变staging驱动的文件时，你可以查看git log：
```bash
sarah@xanatos:~/git/kernels/xhci$ git log --pretty=oneline --abbrev-commit drivers/staging/et131x/et131x.c
015851c34202 Staging: et131x: Fix warning of prefer ether_addr_copy() in et131x.c
a9f488831cb4 staging: et131x: fix allocation failures
0b15862d46fd staging: et131x: remove spinlock adapter->lock
a863a15bf24a staging: et131x: stop read when hit max delay in et131x_phy_mii_read
```
在这个例子里，"Staging: et131x" 看起来是正确的前缀。请确保在冒号后加空格。

## 针对checkpatch的格式

当创建一个解决checkpatch.pl warning或是error的patch时，指明相应错误或是警告是很重要的，特别是当你有好几条报错的时候。

下面的例子对于内核开发者们来说太过宽泛了：
```
staging: rtl8712: Fix checkpatch warning
```
而以下这个例子告诉了我们你在修改什么：
```
staging: rtl8712: Add blank line after variable declarations.
```
## 针对checkpatch修改的补丁描述格式

对于checkpatch警告，我们有几个可行的策略。其中之一是将整个checkpatch.pl警告/错误包含在你的patch主体中。确保它们不会自动换行，例如：
```bash
Date: Sun, 14 Sep 2014 19:24:03 +0530
From: Vaishali Thakkar <vthakkar1994@gmail.com>
To: opw-kernel@googlegroups.com
Subject: [OPW kernel] [PATCH] Staging: rtl8192e: Fix printk() warning style
List-ID: <opw-kernel.googlegroups.com>
This patch fixes the checkpatch.pl warning:
WARNING: Prefer [subsystem eg: netdev]_info([subsystem]dev, ... then dev_info(dev, ... then pr_info(...  to printk(KERN_INFO ...
Signed-off-by: Vaishali Thakkar<vthakkar1994@gmail.com>
---
```
当checkpatch报错很明确地告诉我们被修改的地方是，这个策略是可行的。然而，在一些情况下，包括上面的例子里，我们需要决定如何修改这些错误。我们推荐你解释你所做的修改以及原因。将功劳归功与checkpatch也不错。所以你的msg应该像这样:
```bash
Replace printk(KERN_INFO...) by netdev_info for more uniform error reporting.  Issue found by checkpatch.
```

## 升级/重新发送补丁

你可以使用`git rebase -i`来重组，分离或者编辑你的patchset；你也许想要保存tag或是指向你第一个patch版本之前的branch. 如果你需要回到你没有保存的patchset的老版本，使用`git reflog`.

一旦你重组了你的patchset，你应该用email重新发送它。你需要额外做一些事情：在subject line里使用`PATCHv2` （或是`PATCHv3`，诸如此类）而不是`PATCH`，对于单个patch增加一个patch changelog，对patchset增加一个cover letter，对新的patch series发送一个回复。

To update the subject lines, add the  -v 2  (or  -v 3, etc) options to  git format-patch. (If you have an older version of  git  that doesn't  understand the  -v  option, you may need to use  --subject-prefix=PATCHv2instead.)

To document what changed in the new patchset, put a patch changelog at  the bottom of your cover letter, just above the diffstat. A patch  changelog should consist of a series of entries like this:
```
v2: Fixed ..., noted by Some Person <some.person@example.org>
    Reworked to use ..., as suggested by ...
v3: ...
```
If you're just sending a single patch, put the patch changelog after  your commit message,  _below_  the  --  line, but above the diffstat.

Finally, to send your new patch series as a reply to the previous one,  first look up the Message-Id of the cover letter (or the one-and-only  patch) in your previous patch series, and then pass that to the  --in-reply-to=  option of either  git format-patch  or  git send-email.