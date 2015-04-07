---
layout: post
title: 用nginx反向代理GitHub Page
description: "用nginx反向代理GitHub Page"
modified: 2014-12-24
tags: [nginx]
---
### 起因

直接看结论请跳到文末。。

昨天心血来潮，想做个博客玩玩，一方面可以记录些东西，另一方面又能把闲置了两年的域名用起来（之前太懒了不愿搞），指不定还能找到些志同道合的朋友呢。

手上有一个新加坡的vps，一个搬瓦工的vps，还有一个台湾的vps做shadowsocks代理用，最开始是设想搭建个静态服务器，直接把Jekyll生成的内容往上一丢。但是同步文件是件麻烦事，索性直接用GitHub的Page来做。

找了个称心的主题，folk完修改了下丢到github.io上，绑定域名后发现不开代理访问速度依旧很渣.

解决方法很简单，只要给每个上这博客的人都开个代理就行。

现在万事俱备，只差钱了。

想想我又不是那个v2ex上自己花钱代理youtube视频，发布私人DNS免费给国人代理的土豪，这种想法也只能是想想。

言归正传，当然只能把代理给想上我博客的人，想去别的网站？没门...

### 代理过程

好了，现在我们有三个设备

*  A：我自己的电脑-->当然在国内，要不然访问github.io怎么会慢。。。
*  B: 代理服务器-->用dns解析到
*  C: 网页服务器，也就是GitHub Page的网页

先要做几个准备工作

1. 创建mofelee.github.io的Git仓库，用CNAME文件把C服务器（GitHub Page的服务器）绑定到blog.mofe.me域名上，修改blog.mofe.me的DNS直接指向mofelee.github.io的ip上
1. 将www.mofe.me绑定到B服务器上，为了域名的统一性，我把mofe.me域名直接显性跳转到www.mofe.me上了。

然后测试C服务器能正常打开后，在B服务器上安装nginx，最开始加入了如下代码
{% highlight nginx %}
server
{
  listen 80;
  server_name www.mofe.me;

  location / {
    proxy_set_header Host $host;
    proxy_set_header X-Real-Ip $remote_addr;
    proxy_set_header X-Forwarded-For $remote_addr;
    proxy_pass http://blog.mofe.me/;
  }

}
{% endhighlight %}

看起来好像没啥大问题，重启nginx后打开站提示404..

百思不得骑姐。上百度搜了半天啥也没搜到【别BS我用百度。。百度没有这文章我才想写的】，google后发现[Stackoverflow](http://stackoverflow.com/questions/1057239/nginx-proxy-for-a-github-page)上有个更酷的例子，用xxx.xxx/blog/映射到blog.xxx.xxx网站上，并且xxx.xxx/blog/下访问到的静态文件也不会和根目录冲突。

具体的代码如下（用于nginx 0.7.59）

{% highlight nginx %}
location /blog {
  set $blog 1;
  rewrite  ^/blog(.*)$ /$1 last;
}
location /css {
  set $blog 1;
  error_page 402 = @blog;
  return 402;
}
location / {
  if ($blog) {
        error_page 402 = @blog;
        return 402;
  }
  # here is where default settings for / should be
  root /usr/local/www/nginx/;
}
location @blog {
        proxy_pass http://blog.superfeedr.com;

        # the rest of proxying parameters should be here

        proxy_set_header   Host                   blog.superfeedr.com;
        proxy_set_header   X-Host                 blog.superfeedr.com;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
}
{% endhighlight %}


因为nginx不是特别懂，大致的意思就是如果是访问blog目录下就设置个标记，并且修改访问的url到根目录，判断是否是从blog目录访问过来的根目录，如果不是的话就用本地文件服务于根目录。如果是从/blog访问过来的，通通由github.io提供页面，以及相关的静态资源。

遂由此改造出了我自己的反向代理【羞。。我的这个简直像个玩具】

{% highlight nginx %}
server
{
  listen 80;
  server_name www.mofe.me;

  location / {
    proxy_pass http://blog.mofe.me;
    proxy_set_header   Host                   blog.mofe.me;
    proxy_set_header   X-Host                 blog.mofe.me;
    proxy_set_header   X-Real-IP $remote_addr;
    proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
  }
}
{% endhighlight %}