---
layout: post
title:  "Mac本地编译chromedriver"
date:   2019-07-04 21:01:32
tags: 
- Mac
- chromedriver
color: rgb(191, 239, 255)
cover: '/blog/Halloween/2019_05_06_04.webp'
subtitle: 'Mac OS编译chromedriver踩坑之路'
---

Mac OS本地编译chromedriver，参考[官方文档](https://chromium.googlesource.com/chromium/src/+/master/docs/mac_build_instructions.md)

chromedriver中自带的属性可以用来进行是否爬虫鉴定，所以被很多网站应用，这也是我为什么自己编译chromedriver的原因。耗费了自己很大的功夫，主要原因还是因为GFW的问题，大家都懂的。

电脑即使浏览器配置了翻墙，终端也是不能翻墙的，所以要先配置终端翻墙，如果没有VPN的话，建议买一个吧，或者参考我之前的文章[AWS搭建VPN](https://whisperloli.github.io/2019/02/16/AWS_VPN.html)，因为接下来的工作在翻墙的基础上完成的

配置终端翻墙

```shell
vi ~/.bash_profile

export http_proxy=http://127.0.0.1:1087
export https_proxy=http://127.0.0.1:1087
# export all_proxy=socks5://127.0.0.1:1089
#可以配置http_proxy和https_proxy，也可以配置所有的代理都用socks5，即上面注释掉的all_proxy，上面两种方式二选一即可

source ~/.bash_profile
```

切换系统的默认python为python2，我的因为安装了Anaconda，所以默认的换成了python3，后面需要使用python2

安装depot_tools工具

```shell
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
```

配置depot_tools到环境变量中

```shell
vi ~/.bash_profile

export PATH="$PATH:/depot_tools_path/depot_tools"

source ~/.bash_profile
```

配置git编码支持

```
git config --global core.precomposeUnicode true
```

创建一个目录用于获取chrome的源码

```
# 随便找个地方创建，当然得保证有读写权限
mkdir chromium && cd chromium

#获取代码
fetch --no-history chromium
```

获取代码这一步差不多花了我5、6个小时吧，只要网络正常，一般不会出现什么问题，可以夜里面慢慢下载，我就是这样做的。最后下载完，看了一下文件夹大小，26个G左右，太大了，每次终端进入到这个文件夹下执行操作都会卡顿

下载完成后会有个src目录，进入src目录

```shell
cd src
```

切换分支，chromedriver需要对应chrome浏览器版本，按自己的需要切换，大致相近即可

同步工程相关代码

```shell
gclient sync --force
```

上面这步同步代码完成后可能会报错
![image](/blog/blog_mac_compile_chromedriver/gclient_sysn_error.jpg)
红框中标出的内容说明代码同步完成，下面的错误可以不管它

写配置信息到args.gn中

```shell
vi args.gn

is_debug=false
is_component_build=true
symbol_level=0

```

执行runhooks操作，这一步也是最容易出问题的一步，会出现各种莫名的问题

```shell
gclient runhooks
```

错误记录及解决方案

1. 错误：buildtools/mac/gn文件不存在
	
	解决方案：[gn下载网站](https://chrome-infra-packages.appspot.com/p/gn/gn/mac-amd64/+/git_revision:c4a88ac93e44a1950ecb4b490ef63dea6d2bf3d3)，去网站上下载一个，放到需要的目录下，之后会在报错信息中告诉你，当前gn与需要的gn版本不一致，然后错误信息中会给出需要的版本的下载路径
2. 错误：下载https://commondatastorage.googleapis.com/chromium-browser-clang/Mac/clang-363790-d874c057-3.tgz失败
![image](/blog/blog_mac_compile_chromedriver/download_failure.jpg)

    解决方案:这个问题也是因为GFW导致的，需要打开tools/clang/scripts/update.py，配置urlopen翻墙，也就是挂代理
    ![image](/blog/blog_mac_compile_chromedriver/add_proxy.jpg)
    红框中的内容是我自己加进去的
    
    ```shell
    import urllib2      		
    proxies=urllib2.ProxyHandler({'https':'127.0.0.1:1087'})
    opener=urllib2.build_opener(proxies)
    urllib2.install_opener(opener)
    ```
    如果还是不行的话可能是网络不好，多试几遍
3. 错误：找不到.cipd\_bin/vpython文件
	解决方案: 在终端执行update\_depot\_tools命令，如果执行该命令报错
	```shell
	curl: (35) error:1400410B:SSL routines:CONNECT_CR_SRVR_HELLO:wrong version number
	```
	这是因为网络不好的原因，网络好的话，秒级时间就OK

之后runhooks就没什么错误了，一路通畅

上述结束后可以看到提示：Running hooks: 100% (75/75), done.
那么执行下面的命令吧

```shell
gn gen out/Release
```
可能会出现如下的错误
![image](/blog/blog_mac_compile_chromedriver/xcode_error.jpg)

这个是因为电脑中没有安装Xcode的缘故，赶紧去AppStore下一个，Xcode也挺大的，几个G的大小，下载完成后安装，然后按照错误中的提示执行命令

```shell
#最后的是Xcode.app安装的路径，如果不在/Applications下面，可以用find命令查找一下
sudo xcode-select -s /Applications/Xcode.app
```

马上就要大功告成了，激动吧

```shell
ninja -C out/Release chromedriver
```
最后一步差不多十来分钟，可以去喝个茶，休息会，等待执行完成后，进入out/Release目录下面就可以看到chromedriver了，如果你也是想用于爬虫的话，记得修改源码的识别码哦，我已经改了，在本博文中没写进去。最后，说一句Fucking GFW，因为浪费我时间最久的就是这个
