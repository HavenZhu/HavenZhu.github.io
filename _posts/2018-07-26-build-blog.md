---
layout: post
title: 此博客搭建过程记录
category: mix
tags: [mix]
---

这几天跟我老婆聊天，她说无聊时想写写东西，我说我给你搭个博客，你就在自己的博客里写吧。这一篇就记录一下搭建的过程。

主要分成以下几个步骤:
1. jekyll 生成博客框架
2. github 托管
3. nginx 配置访问静态页面
4. crontab 定时拉取并build

## jekyll

博客是基于[jekyll](https://jekyllrb.com/)搭建的，jekyll 是个博客生成工具，类似于 WordPress，不同的是 jekyll 只生成静态网页。支持 markdown 语法，写起来非常舒服。
按照官网的说明：
```text
~ $ gem install bundler jekyll
~ $ jekyll new my-awesome-site
~ $ cd my-awesome-site
~/my-awesome-site $ bundle exec jekyll serve
# => Now browse to http://localhost:4000
```
这样博客的主页就出来了，略显简陋，找了一套主题：[Yummy Jekyll Theme](https://github.com/DONGChuan/Yummy-Jekyll)，按照说明配置一下。

## github

Yummy Jekyll 是将代码托管到 github page 上的，把那一坨代码 push 上去之后，就可以感受一下啦。
<p align="center">
    <img src="http://betterzn.com/assets/images/2018_07_26/blog.png" />
</p>

后来发现在国内访问 github 太慢了，刷新个页面要几分钟。还是得放到国内的服务器上更靠谱，之前阿里云打折的时候买了3年，刚好派上用场。

## nginx

先在阿里云上把 jekyll 和 git 的环境搭了一下。安装好 nginx 之后，直接访问阿里云的ip会进入 nginx 的默认页面。现在把默认页面修改为博客主页。
1. clone 代码到阿里云
2. 使用 jekyll build 生成好静态的网页，build 之后所有的内容都在 _site 文件夹下。
3. 配置 nginx，修改 /etc/nginx/site-enabled/ 中的 default 文件，将 root 指向 _site 文件夹。这样阿里云默认的页面就是博客主页了。

## crontab

配置完上面这些内容之后，更新博客的过程是这样的：
1. 在本地把博客内容写好，push上去
2. 阿里云上pull下来
3. jekyll build

第2和第3步是重复工作，可以放到一个脚本里，像这样：
```text
#!/bin/sh
cd /path/to/project/
/usr/bin/git pull origin master
/usr/local/bin/jekyll build
```
但是即便放到脚本里了，还是每次都得执行一下，还不够自动化。

linux crontab的作用就是定时执行某个命令。

```text
# 1h between 8am and 11pm
0 8-23 * * * /usr/crontab/xxxxxx.sh >> /usr/crontab/xxxxxxx.log 2>&1
```
这里就是在上午8点到晚上11点之间每隔一个小时执行一次上面的脚本，并输出日志。
> 关于crontab的用法可以参考：[crontab定时任务](http://linuxtools-rst.readthedocs.io/zh_CN/latest/tool/crontab.html)

搞定！以后就只需要写好博客并push上去就可以啦。