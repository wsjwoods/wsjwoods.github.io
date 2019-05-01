---
layout:     post
title:      "2019-05-01最新Linux安装Ruby 安装Jekyll"
subtitle:   "github自建博客 GitHub Pages + Jekyll"
date:       2019-05-01
author:     "Woods"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Linux
---


网上很多安装方式都有问题，
这里汇总一下并亲自试验且安装成功

我的linux环境：centos7

## 1.安装rvm
```
gpg2 --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB
curl -sSL https://get.rvm.io | bash -s stable
```

查看rvm安装路径
```
[root@bigdata-003 user]# find / -name rvm
/usr/local/rvm
/usr/local/rvm/scripts/rvm
/usr/local/rvm/bin/rvm
/usr/local/rvm/lib/rvm
/usr/local/rvm/src/rvm
/usr/local/rvm/src/rvm/scripts/rvm
/usr/local/rvm/src/rvm/bin/rvm
/usr/local/rvm/src/rvm/lib/rvm
```
发现在 `/usr/local/rvm `路径 
并将`/usr/local/rvm/bin`添加到环境变量中


修改 RVM 的 Ruby 安装源到 Ruby China 的 Ruby 镜像服务器，这样能提高安装速度
`echo "ruby_url=https://cache.ruby-china.com/pub/ruby" > /usr/local/rvm/user/db`
这里的 `> /usr/local/rvm/user/db` 路径为上一步查找到的rvm路径下的`user/db`


## 2.Ruby 的安装与切换
列出已知的 Ruby 版本
`rvm list known`

安装一个 Ruby 版本（最新版）
`rvm install 2.6.0 --disable-binary`

这里安装了最新的 2.6.0, rvm list known 列表里面的都可以拿来安装。
切换 Ruby 版本
`rvm use 2.6.0`

如果想设置为默认版本，这样一来以后新打开的控制台默认的 Ruby 就是这个版本
`rvm use 2.6.0 --default `

查询已经安装的ruby
`rvm list`

卸载一个已安装版本（如果有的话）
`rvm remove 1.8.7`



## 3.安装jekyll
切换gem数据源 https://rubygems.org/ 国内被墙 需要更换成 https://gems.ruby-china.com/ 
注意：https://gems.ruby-china.com/   这个国内的ruby源换过很多次，截至2019.5.1 只有这个是可以连接上的。
```
gem sources --remove https://rubygems.org/
gem sources -a https://gems.ruby-china.com/ 
```
查看gem数据源
`gem sources -l`
```
*** CURRENT SOURCES ***

https://gems.ruby-china.com/
```

安装jekyll
`gem install jekyll`



## 4.安装GIT

安装新版本git
```
yum remove git -y
yum install http://opensource.wandisco.com/centos/6/git/x86_64/wandisco-git-release-6-1.noarch.rpm
```
安装新版本git
```

yum remove git -y
yum install http://opensource.wandisco.com/centos/6/git/x86_64/wandisco-git-release-6-1.noarch.rpm
```
```
yum install git -y
git --version
```

```
email="wsjwoods555@163.com"
owner="wsjwoods"
git config --global user.email "$email"
git config --global user.name "$owner"
```

git免密配置
```
cd ~/.ssh
ssh-keygen -t rsa -C "$email"
```


查看秘钥
```
cat ~/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDKRB54hYeRfEr5Os8Y79afx5Y3G1TwcLAx6kiNYmpnPoDsPxOUVCvApCS7cIG5Yd6Bo0iiugOm4xabxGgvQuFjuP6EzWcE5ZwWtV2ncybVi2xhWZbx7Xf+mDSvoI.....
```

复制秘钥配置到github的SSH KEY中

## 5.启动jekyll调试你的Blog
假设你已经在git上有了blog了
需要 git clone git地址到linux上
cd到博客文件夹
```cd /home/blog/wsjwoods.github.io```

启动jekyll（必须在博客文件夹下面）
--host 0.0.0.0 表示局域网内其他机器可以访问
```jekyll server --watch --host=0.0.0.0```

watch为了检测文件夹内的变化，即修改后不需要重新启动jekyll

报错： `The 'gems' configuration option has been renamed to 'plugins'. Please update your config file accordingly.`

修改_config.yml

gems: [jekyll-paginate]--- 错误

plugins: [jekyll-paginate]----正确

报错：
```
Dependency Error: Yikes! It looks like you don't have jekyll-paginate or one of its dependencies installed. In order to use Jekyll as currently configured, you'll need to install this gem. The full error message from Ruby is: 'cannot load such file -- jekyll-paginate' If you run into trouble, you can find helpful resources at https://jekyllrb.com/help/! 
jekyll 3.8.5 | Error:  jekyll-paginate
```

安装jekyll-paginate 
```gem install jekyll-paginate```

在启动jekyll

```jekyll server --watch --host=0.0.0.0```
```
Configuration file: /home/blog/wsjwoods.github.io/_config.yml
            Source: /home/blog/wsjwoods.github.io
       Destination: /home/blog/wsjwoods.github.io/_site
 Incremental build: disabled. Enable with --incremental
      Generating... 
    Liquid Warning: Liquid syntax error (line 50): Unexpected character & in "site.duoshuo_share && site.duoshuo_username" in /_layouts/post.html
    Liquid Warning: Liquid syntax error (line 125): Unexpected character { in "tag[1].size > {{site.featured-condition-size}}" in /_layouts/post.html
    Liquid Warning: Liquid syntax error (line 50): Unexpected character & in "site.duoshuo_share && site.duoshuo_username" in /_layouts/post.html
    Liquid Warning: Liquid syntax error (line 125): Unexpected character { in "tag[1].size > {{site.featured-condition-size}}" in /_layouts/post.html
    Liquid Warning: Liquid syntax error (line 50): Unexpected character & in "site.duoshuo_share && site.duoshuo_username" in /_layouts/post.html
    Liquid Warning: Liquid syntax error (line 125): Unexpected character { in "tag[1].size > {{site.featured-condition-size}}" in /_layouts/post.html
    Liquid Warning: Liquid syntax error (line 58): Unexpected character & in "site.duoshuo_share && site.duoshuo_username" in /_layouts/keynote.html
    Liquid Warning: Liquid syntax error (line 133): Unexpected character { in "tag[1].size > {{site.featured-condition-size}}" in /_layouts/keynote.html
    Liquid Warning: Liquid syntax error (line 50): Unexpected character & in "site.duoshuo_share && site.duoshuo_username" in /_layouts/post.html
    Liquid Warning: Liquid syntax error (line 125): Unexpected character { in "tag[1].size > {{site.featured-condition-size}}" in /_layouts/post.html
    Liquid Warning: Liquid syntax error (line 38): Unexpected character { in "tag[1].size > {{site.featured-condition-size}}" in /_layouts/page.html
    Liquid Warning: Liquid syntax error (line 87): Unexpected character { in "tag[1].size > {{site.featured-condition-size}}" in /_layouts/page.html
    Liquid Warning: Liquid syntax error (line 38): Unexpected character { in "tag[1].size > {{site.featured-condition-size}}" in /_layouts/page.html
    Liquid Warning: Liquid syntax error (line 87): Unexpected character { in "tag[1].size > {{site.featured-condition-size}}" in /_layouts/page.html
                    done in 0.333 seconds.
 Auto-regeneration: enabled for '/home/blog/wsjwoods.github.io'
    Server address: http://127.0.0.1:4000
  Server running... press ctrl-c to stop.
```
表示启动成功

然后就可以登录  http://127.0.0.1:4000 查看你的博客了





参考
https://ruby-china.org/wiki/rvm-guide

https://www.cnblogs.com/ee2213/p/3915243.html?utm_source=tuicool

https://github.com/kunnan/kunnan.github.io