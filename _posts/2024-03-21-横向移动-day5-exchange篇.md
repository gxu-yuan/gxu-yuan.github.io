---
title: 横向移动-day5-Exchange篇
date: 2024-03-21 17:40 +0800
categories: [内网,横向移动] 
tags: [横向移动,Exchange]
img_path: /img/penetration/
---
## 0x00 准备与介绍

Exchange Server是微软公司的一套电子邮件服务组件，是个消息与协作系统。简单而言， Exchange server 可以被用来构架应用于企业、学校的邮件系统。Exchange server 还是一个协作平台。在此基础上可以开发工作流，知识管理系统，Web 系统或者是其他消息系统。

Exchange不仅在内网中会有应用，在公网上也是有可能有的！！！

一般对此的主要渗透测试手段有：

1）信息收集（推荐SPN）；

2）看看能不能利用内网的账户密码来登录（该方法一般配合钓鱼！！！）；

3）漏洞利用

![截图](e1332a4c3412a746dbc7560f49570a2c.png)

#### 环境

使用0day.org。 ==> 开启了jack、Srv-DB、Exchange1010sp3 三台机器。

其中我们在jack里添加了外网网段的卡，并让其上线cs。并提权至sysytem权限！

开启socket5代理转发！！！

## 0x01信息收集

这里我们设置了三种的收集方式：

- 端口扫描

exchange会对外暴露接口如OWA,ECP等，会暴露在80端口，而且25/587/2525等端口上会有SMTP服务，所以可以通过一些端口特征来定位exchange。

比如：直接访问网址 https://192.168.3.142/owa/auth/logon.aspx

![截图](80699953e9dd176296fd5690e0fd916f.png)

也可以直接用cs进行扫描：
![截图](4e2dee845367c2d8b4438a3481d67ee8.png)

- 直接使用SPN（注意：这里可能需要在普通用户权限下使用）

```powershell
powershell setspn -T 0day.org -q */*
```

![截图](54aff563197d823398bcb1455a03e528.png)

- 直接使用脚本探测

Exchange_GetVersion_MatchVul.py ： 

```powershell
python .\Exchange_GetVersion_MatchVul.py 192.168.3.142
```

![截图](800dcdd03d3950d907bd73c627768085.png)

## 0x02 密码爆破

这一部分呢，主要是基于内网中使用。 这是因为当我们拿一下一台机器的权限后，我们可以直接利用cs抓取机器中存取的账号与密码，从而利用这个账号与密码进行爆破。

（一般来说，内网中exchange的账户密码都是默认内网成员的账号与密码）

#### 首先用cs抓取密码与账户：

![截图](07de7be55c56e8e20657fde04624418c.png)

![截图](c8b61a38c8de87d82d185593ac25f4f1.png)

#### 开启burp尝试爆破登录

![截图](59561ae5a4581e0fd82c6f0dbad3c253.png)

利用burp抓取，并发送到intruder模块：（使用cluster bomb爆破模式）

![截图](3270bd419a583df047798d19930e32ae.png)

添加前往中获取的账户与密码：

![截图](36ac58fc7acdeab8e3c62764383e4b95.png)

然后，开始爆破后发现，sqladmin以及jack这两个用户对应的admin!@#45密码下的数据包明显更长！ 且发现返回包中带有的了更多的session数据：

![截图](710ffe93774278f91f55309bddcb1d67.png)

## 0x03 漏洞

在前文中，我们已经利用了脚本探测到对应的版本信息了。 此时，我们可以去网上找该版本下存在的cve漏洞。

参考: https://www.cnblogs.com/xiaozi/p/14481595.html

这里，我们就演示一个：

CVE-2020-0688

```powershell
 python .\cve-2020-0688.py -s https://192.168.3.142 -u jack@0day.org  -p admin!@#45 -c "cmd /c calc.exe"
```

运行后会出现payload需要填写，此时启ysoserial.exe程序，将脚本提示的命令输入，就可以出现我们所需的脚本payload了！！

![截图](f6ee83d6cdf5d8d9d78191069930dc2c.png)

## 0x04 可能出现的bug

- 就是最后的cve-2020-0688 脚本可能出现readline模块无法安装，使用以下命令：


```powershell
python -m pip install pyreadline
```

- 可能经常出现网络通信问题.............