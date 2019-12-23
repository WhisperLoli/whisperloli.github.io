---
layout: post
title:  "大麦抢票"
date:   2019-12-23 22:05:21 +0800
tags: Python，Selenium
color: rgb(255,90,90)
cover: '/blog/Halloween/2019_05_06_06.webp'
subtitle: '简要介绍大麦网抢票机制'
---
废话不多说，大部分都是为了把妹才研究的！
如README里面写的，首先需要安装selenium，我使用的浏览器及环境如下

```
浏览器：谷歌
chromedriver下载地址：http://chromedriver.storage.googleapis.com/index.html
操作系统：Mac OS
解压后执行命令，perl -pi -e 's/cdc_/dog_/g' /Applications/chromedriver，/Applications/chromedriver为解压后的位置
上述命令并未在Windows上测试，可能Windows需要自己安装perl
```
写脚本时是用jupyter notebook编写的，方便调试代码，可能你还需要安装Anaconda，上百度搜索一个下载，然后安装，完成后在命令行输入jupyter notebook就可以打开浏览器一个编辑页面，左上角File中可以把代码文件GetTicket.ipynb加载进来，如下图
![image](/blog/blog_damai_ticket/notebook.jpg)

整个抢票流程分为如下几个步骤：

```
1. 数据设置，大麦网账号上需要有身份证信息，默认地址信息之类
2. 账号密码登录
3. 搜索定位明星
4. 详情页需求选择
5. 下单页下单
6. 失败页面处理
```

抢票需要登录账号，而大麦网登录系统和淘宝是公用的一套登录系统，会识别selenium，干扰验证码认证，从而无法登录，使用selenium打开的浏览器，即使人工登录也会出现此问题，所以需要修改chromedriver的源码，使用perl修改会很方便，替换字符串cdc\_为dog\_，mac自带了perl命令，windows可能是需要自己安装的

```
perl -pi -e 's/cdc_/dog_/g' /Applications/chromedriver
```
注意登录的时候，有时会有验证码框，有时没有，所以需要自己判断是否出现验证码，及滑动滑块破解验证码，代码如下，滑动滑块验证码算是最简单的验证码之一

```python
if browser.find_element_by_id("nocaptcha-password").is_displayed():
    #滑动验证码,获取平移距离,移动解锁
    start=browser.find_element_by_id("nc_1_n1z")
    inner_size=start.size["width"]
    outer_size=browser.find_element_by_class_name("nc-lang-cnt").size["width"]
    move_size=outer_size-inner_size
    ActionChains(browser).drag_and_drop_by_offset(start, move_size, 0).perform()
    time.sleep(2)#滑完了之后稍等下，让系统判断完毕
```

着重讲解一下判断是否开售，如果没开售，则图片中【立即购买】的内容会是【即将开抢】，而开售后会是【立即购买】，票卖完后是【提交缺货登记】，我们需要根据这个框中的内容去判断程序需要做什么。票卖完时，就退出程序，当票还没开售时，就不停的刷新页面，开售后，程序就不停的下单直到下单成功。有时我们想买380块钱的票，结果卖完了，而又很想带小姐姐去看演唱会，那么可以在配置文件中配置auto_upgrade=True，该配置的意思是当该档位的票抢完了，则自动升级抢下一个档位的票，也就是图中680块钱的票
![image](/blog/blog_damai_ticket/buy.jpg)

有时，已经进入下单页面了，点击【同意以上协议并提交订单】后，会弹出一个div框，框中内容是【亲，同一时间下单人数过多，建议您稍后再试】，点击框中的确认按钮，就会把用户踢到选择价位、购票数量的详情页，我用的解决办法是直接刷新浏览器，再操作一遍，然后点击同意，只要不弹框提示票不够，就一只刷，如果弹框提示票不够的话，也只能被踢到选择价位页重来。之前抢票时，光记着改代码，忘记截图保存，所以在此没办法展示

需要注意的事项，**有时明星过于火爆，每笔订单会限购2张票，需要对应两张身份证信息，需要提前在大麦网中输入好身份证信息**

我用该脚本抢到过梁静茹的2020年2月14号情人节演唱会的票2张，但是奈何没有小姐姐啊，单子已经完成，就差付款后领小姐姐看演唱会

代码还是有很多需要改进的地方，有实际应用才好改进，也就是bug复现

完整代码参见[github](https://github.com/WhisperLoli/DaMai)
