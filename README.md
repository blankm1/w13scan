# w13scan
长亭的xray挺不错的，可惜没有开源，而且很多地方我都有不同的想法，网络上开源的被动扫描器都不够好，所以造此轮子。

## 简介
w13scan是一款插件化基于流量分析的扫描器，通过编写插件它会从访问流量中自动扫描，基于Python3。

## 安装
- git clone https://github.com/boy-hack/w13scan & cd w13scan
- pip3 install -r requirements.txt
- python3 main.py

## Https支持
设置代理服务器(默认127.0.0.1:7778)后，访问`http://w13scan.ca`下载根证书并信任它。

## 相关配置
`config.py`保存了扫描器使用的各种配置，按照注释修改即可。

## 检测插件
- [x] 敏感文件扫描
    - [x] 基于目录git,svn扫描
    - [x] 目录未授权访问
- [x] PHP真实路径泄漏
- [ ] 备份文件扫描
- [x] SQL注入(GET 型)
    - [x] 数字型SQL注入
    - [x] 报错型SQL注入
    - [x] 时间盲注
    - [x] 布尔类型SQL注入
- [ ] 命令注入
- [ ] 文件包含漏洞

### 其他插件
- [ ] 被动子域名搜索
- [ ] 被动E-mail,Phone等信息搜索

## Thx
- https://github.com/qiyeboy/BaseProxy  代理框架基于它
- https://github.com/chaitin/xray  灵感来源，部分规则基于它
- https://github.com/knownsec/pocsuite3  代码框架模仿自它

## 法律
本程序仅用于学习交流,在使用之前请遵守当地相关法律进行，切勿用于非法用途。

# w13scan 设计思路
吸取w12scan的教训，w12scan写到后面就懒得写自己的思路了，而且设计这种东西每天都不一样，有时候写了篇文章，第二天把架构又换了。为防止这种情况发生，所以记载将以时间顺序，也当作日记来看吧，也可以让大家明白，我为什么这么做。

- 2019.6.28 周五  自从又了w13scan的想法后，想了一周的如何设计，看到开源的`https://github.com/qiyeboy/BaseProxy`，太好了，这就是我需要的代理框架。然后在它基础上做了些符合我设计的调整。用xray的时候访问外网会非常的卡,所以我设想的是代理框架在返回给浏览器的时候再来调用插件而不是在截获请求的时候就调用,代理访问和插件扫描是分离开的。
- 2019.6.29 周六  初步制定了插件化的调用框架，插件结构，整个代码结构模仿pocsuite3。插件可以从返回的源码中获取链接用于组合payload。完善了整个框架，编写了一些测试插件，0.1版本发布🎉🎉,因为插件架构的原因，每个插件中发送的请求要尽可能少，因为每个插件只开一个线程运行，当插件中请求较多时，会比较慢，所以对插件要求粒度拆分更细。增加了几个简单的扫描插件，基于当前返回的网页源码以及从流量分析中获取的目录作为扫描payload。
- 2019.6.30 周日  在健壮了框架的一些功能后，开始编写SQL注入插件，首先得实现一个网页相似度对比算法，SQL注入中很多网页的比对都要基于该算法，在`w3af`中找到了该算法，这个算法挺特别的，它不是基于dom树，它根据一些特殊的标签`<'"`来分隔文本在进行比较，比基于dom树简单但效果挺好～太厉害了,sqlmap的是基于单词的对比，会去掉所有网页元素，也集成上来了，不过效果不太理想。抓取了xray的payload，将SQL注入插件（数字型，报错型，时间盲注型，布尔类型）都写完了，不过只完成了`GET`类型的SQL注入判断，`POST`类型看情况再加吧（主要没有靶机测试，不知道效果..）等插件多了，我想再将插件的调度流程优化一下，目前还是有些繁琐。还有一个框架的去重策略也不够好，准备继续优化。

## 如何编写插件
以内置的文件扫描插件为例
```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
# @Time    : 2019/6/29 12:16 AM
# @Author  : w8ay
# @File    : filescan.py
from lib.plugins import PluginBase
import requests


class W13SCAN(PluginBase):
    name = '为插件起个名字？'
    desc = '''基于流量动态生成敏感目录及文件，进行扫描'''

    def audit(self):
        method = self.requests.command  # 请求方式 GET or POST
        headers = self.requests.get_headers()  # 请求头 dict类型
        url = self.build_url()  # 请求完整URL
        data = self.requests.get_body_data().decode()  # POST 数据

        resp_data = self.response.get_body_data()  # 返回数据 byte类型
        resp_str = self.response.get_body_str()  # 返回数据 str类型 自动解码
        resp_headers = self.response.get_headers()  # 返回头 dict类型

```
需要声明名称为`W13SCAN`的类名，并基于`PluginBase`类，在其中声明`audit`方法，即可获得请求内容以及返回内容。

