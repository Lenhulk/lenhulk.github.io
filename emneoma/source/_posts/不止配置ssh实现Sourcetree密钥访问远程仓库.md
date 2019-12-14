---
title: 不止配置ssh实现Sourcetree密钥访问远程仓库
date: 2019-12-14 23:44:24
tags: [ssh,shell,Sourcetree,Automator]
Categories: ssh
---

{% cq %}

每次推拉合并代码到远程仓库都需要输入账号密码，那也太慢了太糗了吧？

配置好了ssh私钥，但是为何报错port 22: Connection refused呢？

终于免密操作了，咋电脑重启又不不好使了？

{% endcq %}



<!-- more -->

# 背景

团队实施了组件化开发，项目分成多个组件。

难以避免的会频繁使用pod及远程仓库操作，每天十来位同事在疯狂输出，一不留神项目git就几十个更新，如果每次pull/rebase/merge/push/代码都需要输入账号密码，那绝对1000%影响开发进度。



# 配置ssh

> “SSH 为 Secure Shell 的缩写，由 IETF 的网络小组（Network Working Group）所制定；SSH 为建立在应用层基础上的安全协议。SSH 是目前较可靠，专为远程登录会话和其他网络服务提供安全性的协议。利用 SSH 协议可以有效防止远程管理过程中的信息泄露问题。

几乎每一个远程仓库服务商（github, gitlab, bitbucket...）都有如何生成以及将密钥配置到远程仓库的[教程](https://help.github.com/en/github/authenticating-to-github/connecting-to-github-with-ssh)，这里简单看看步骤



### 常规操作

* 生成新的ssh key

  ```shell
  ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
  ```

* 或查看已存在的key

  ```shell
  ls -al ~/.ssh
  ```

* 将您的SSH私钥添加到ssh-agent（该agent为您实现身份验证操作哦）并将密码短语存储在Mac钥匙串中

* ```shell
  ssh-add -K ~/.ssh/id_rsa
  ```

* 将ssh密钥复制到剪贴板后，粘到远程仓库的配置中

  ```shell
  pbcopy < ~/.ssh/id_rsa.pub
  ```



> `-K`选项是Apple的标准版本`ssh-add`，当您将ssh密钥添加到ssh-agent时，它将密码短语存储在您的钥匙串中



### 报错：ssh: connect to host gitlab.com port 22: Connection refused

你已经将私钥添加到了代理和钥匙串，远程也配置好了，然鹅不管终端还是sourcetree操作还是会报错。

* 测试链接，如下是假如你想配置多个平台.. 此时你输入哪一个都会报错

  ```shell
  ssh -T git@github.com
  ssh -T bitbucket.org
  ssh -T 192.160.2.xxx   //工作的私有远程仓库
  ```

  为什么呢？

  > 如果您使用的是macOS Sierra 10.12.2或更高版本，需要`~/.ssh/config`文件以将密钥自动加载到ssh-agent中，并将密码短语存储在密钥链中

也就是说ssh-add还不够，我还需要配置ssh的config文件，config文件涉及到很多不同字段有不同的作用，可以在：[SSH的config配置之多账号简单管理](https://jingwei.link/2018/12/15/ssh-config-multi-app-manager.html)，学习一哈！

最后，根据我的需求，创建&编辑了一个config文件，不同网站对应了不同的key：

```
# Github
Host opensource   
  Hostname ssh.github.com  
  PreferredAuthentications publickey  
  IdentityFile ~/.ssh/id_rsa_github
  User xxxxx@xx.com 
  
# Bitbucket
Host private
  HostName bitbucket.org
  Preferredauthentications publickey
  IdentityFile ~/.ssh/id_ed25519_bb
  User git

# Private GitLab instance
Host work
  HostName 192.160.2.xxx
  port 1888
  Preferredauthentications publickey
  IdentityFile ~/.ssh/id_rsa_work
  User git
```





### 报错：Bad owner or permissions on ~/xxx/.ssh/config

配置完config文件，再次通过ssh -T测试连接，却又报错，这次是配置文件的权限不够，我们给他权限

```shell
sudo chmod 600 ~/xxx/.ssh/config
```

再次连接就成功了，会提示你successfully autheticated或者Hi ,welcome to XXX。

通过ssh这个通行证，您可以用sourcetree任意玩耍不再需要输入账号密码，也不再需要输入密钥的安全码了！😄



---

## 重启了Mac，sourcetree操作又报权限错误

soucetree为啥会报错呢，因为他操作命令是不会帮你输入密码的，也就是rsa密钥的安全码！

当用你用terminal操作git你就会发现，又需要输入rsa的passphase了。



在重启之后，ssh-agent估计（可能？）也是重启了，于是密钥丢失了。

意味着每次重启Mac你就要去执行ssh-add -K！



我们把这几个命令自动化吧，在每次启动时执行shell，步骤：

1. 找到自动操作.app，创建一个Automator 应用程序

![1](https://s2.ax1x.com/2019/12/15/QWBxMj.png)



2. 选择“运行shell脚本”，输入命令，点击顶部未命名保存

![2](https://s2.ax1x.com/2019/12/15/QWBjzQ.png)



3. 系统偏好设置->用户与群组->登录项，选择我们保存的app文件，实现开机自启动

   ![3](https://s2.ax1x.com/2019/12/15/QWBXRg.png)