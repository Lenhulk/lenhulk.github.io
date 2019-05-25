---
title: “git add .” not worked - “诡异”问题解决
date: 2019-05-26 03:25:19
tags: git
categories: git
---

{% blockquote %}
听终端的话，别留下warning。
{% endblockquote %}

{% codeblock lang:bash %}
$ git add .
$ git commit -m "fuckit"
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)
  (commit or discard the untracked or modified content in submodules)

	modified:   xxx/themes/next (modified content, untracked content)

no changes added to commit (use "git add" and/or "git commit -a")
{% endcodeblock %}

<!-- more -->
---

## 起因 ##
自从把博客源码托管到github仓库之后，腰也不酸了，腿也不痛了，一口气上五楼..咳咳

之后我给hexo换了一个NEXT主题，爽炸了！可是今天修改了主题某个js文件，
“**git status**”之后却显示*“nothing to commit”* ???

上github上看源码仓库，却发现主题文件夹下空空如也..仓库却显示没问题，**git log**，一直停留在上个提交版本。

于是一直死循环操作“**git add .**”&&“**git commit -m "xxx"**”，
终端却一直无情回复我“Changes not staged for commit”，如上面那段代码所示。

尝试按照终端提示的办法无法解决。

这就很诡异了？于是我花了一个晚上疯狂踩坑..

## 爬坑 ##
搜索了各个关键词句，看了stackoverflow和几篇无用博文还是没有找到合适的解决。
直觉是文件引入的问题，一口气把next文件 “**git rm --f xxx/themes/next**” 再add进去，仔细一看却发现有如下提示：
``` bash
$ git add .
warning: adding embedded git repository: xxx/themes/next
hint: You've added another git repository inside your current repository.
hint: Clones of the outer repository will not contain the contents of
hint: the embedded repository and will not know how to obtain it.
hint: If you meant to add a submodule, use:
hint:
hint: 	git submodule add <url> xxx/themes/next
hint:
hint: If you added this path by mistake, you can remove it from the
hint: index with:
hint:
hint: 	git rm --cached xxx/themes/next
hint:
hint: See "git help submodule" for more information.
```
还好我认识一点英文，想起当时是从git仓库直接克隆next的，却因为粗心没有看warning，径直commit了（居然也没事- -!），才会引发以上的惨剧。
于是乎赶紧把主题下的git仓库删除了。

偷了个懒，把next文件夹移走，commit，再移进来，add之后总算能看到一排排的主题相关文件辣～

## 总结 ##
git仓库下要提交A文件夹，A/B文件夹下也包含一个git仓库（.git）。使用**git add .** 就只能加入一个空的B文件夹。

---

#### 这个故事告诉我们以LINUX“没有回应就是最好的回应”的尿性来说，还是好好看终端回复吧。 ####
