\---

layout: post

title: 搭建GitHub个人博客—Chirpy模板

date: 2024-03-06 22:49 +0800

categories: [搭建, Blog] 

tags: [blog, jekyll]

\---

这篇帖子主要记录自己从0开始搭建这个博客平台的心路历程~ 希望能够对大家有所帮助😊
## 0x00 准备工作
由于我们的博客更新以及主页的初始化都需要在本地进行，因此我们必须先配备好该模板所需的环境: [ruby](https://link.zhihu.com/?target=https%3A//rubyinstaller.org/downloads/)，[git](https://git-scm.com/downloads) 以及[nodejs](https://nodejs.cn/download/)。其中，Git将用于模板的调用与上传，而由于Chirpy模板由Ruby编写，所以初始化的时候会用到Ruby命令行，nodejs在模板初始化时会用到！
#### 安装Git
直接下载64位版本，然后全部默认安装。安装成功后，可以打开`Git CMD`并运行下面命令查看版本，以验证是否安装成功:
```cmd
git --version
```
#### 安装Ruby
运行ruby的安装包（选择Ruby+Devkit），按照指引完成安装（全部设置都选默认的就行）。
>  Ruby下载完成以后直接双击安装，除了安装路径，其他一路默认选项就行。安装路径最好不要包含空格。Ruby安装完成以后会弹出一个窗口让你选择3个选项之一来安装，一般直接选3就是，安装过程需要一定的时间。
{: .prompt-tip }

然后，我们运行`Start Command Prompt with Ruby`（在开始菜单栏的ruby文件夹下可以看到），然后运行检查版本命令来查看是否运行成功：

```cmd
ruby -v
```

然后，输入以下命令进行安装模板所需的库组件：

```cmd
gem install jekyll bundler
```

同理，安装完成后输入查看版本命令：

```cmd
jekyII -v  
bundler -v
```

至此，环境已经基本准备就绪！！！

## 0x01 初始化模板

根据官方给出[^Chirpy]的调用模板有两种方式，一种是使用Chirpy Starter直接作为新模板编辑，这种方式比较简单，但是许多功能被阉割了，所以我采用了第二种方式：在GitHub上fork模板到仓库[^fange]。

其实我之前也按照了网上的方法走了好多遍，也按照官方给的也不行..... 我也不知道为啥，然后后面就按着Fange[^fange]的博客试了下，就成功啦！！！ 感谢感谢！

#### 克隆到本地

首先，我们先进入[官方的模板网址](https://github.com/cotes2020/jekyll-theme-chirpy)，然后点击右上角的fork选项。之后我们就会进入到fork构建仓库的页面，此时我们需要把仓库名`Repository name`改成`GitHub用户名.github.io`，记住一定要改成这样子！！！ 这是官方规定的。之后，我们访问的网址就会是这个域名了，然后确定fork。之后，我们运行`Git CMD`，然后输入以下命令进行克隆：

```bash
git clone git@github.com:[用户名]/[用户名].github.io.git
```

>这里我用的是ssh进行连接与克隆。想要像我一样用ssh进行克隆和后面的登陆的，可以参考附录的ssh建立连接部分。当然，不用ssh的可以直接用http方式！
{: .prompt-warning }

默认是在当前文件路径下自动创建名字位`[用户名].github.io`{: .filepath}的文件

#### 初始化到本地

运行`Start Command Prompt with Ruby`，然后切换目录到你克隆的文件夹：

```bash
cd xxxx # xxx为你模板的地址
```

然后输入以下命令为模板安装缺失的bundle:

```bash
bundle
```

然后输入以下命令初始化模板：

```bash
bash tools/init
```

>这是比较重要的一步，根据官方的说法，这个初始化主要是删除了一些文件，如果不执行这一步，发布网页的时候网页会显示异常。
>{: .prompt-tip }

>这里初始化时候，可能因为Windows的问题会出现`node_env不是系统命令`这个报错。这里，虽然不影响本地运行但是会导致我们在GitHub上部署时报错！ 大家可以先看看会不会报错，如果有报错，可以参考`bug记录中的dist文件夹缺失部分`。
>{: .prompt-danger }

## 0x02 更新配置并发布主页

在我们正式上传之前，首先需要配置本地模板根目录下的`config.yml`，并补全其中的url部分：

```sass
url: "https://[GitHub用户名].github.io"
```
{: file='_config.yml'}

这里，其实已经可以上传了！！但是我先提前介绍以下`config.yml`的其他配置，大家也可以先配置后再上传，当然也可以直接上传先看看，然后后续自己再更新也可以。

```sass
系统语言，格式对应根目录下的`_data/locales`。  
lang: zh-CN
  
title: Yuan's Website  #主标题  
tagline: 一个简简单单的个人主页  #主标题的简单介绍  
  
github:    
  username: gxu-yuan  #左下角GitHub标志对应的跳转页面       
twitter:  
  username: yuan  #左下角Twitter标志对应的跳转页面    
  
social:
  name: Yuan Zhang  #网页中间最下面的保留权利人姓名  
  email: zy959536@gmail.com #左下角Email标志对应的跳转页面  
  links:  
    # 点击前面姓名跳转的页面，-为开启，#为关闭，若开启多个，展示最前面的一个  
    # https://twitter.com/username      
    - https://github.com/gxu-yuan     
    # https://www.facebook.com/username  
    # https://www.linkedin.com/in/username   
  
theme_mode:  #色调模式，有黑和白两种，若不填写左下角会有切换按钮  
  
avatar: '/img/wtw.png'  #左上角头像  
```

{: file='_config.yml'}

接下来，我们正式开始把初始化的模板传至你的GitHub仓库之中并覆盖掉原来的模板。启动`Git CMD`，输入以下命令将路径转至模板下：

```bash
cd xxxx # xxx为你模板的地址
```

然后，按以下顺序输入下面四步命令：

```bash
将本地文件存到暂存区  
git commit -m “Yuan\'s Blog v1”  #将暂存区文件提交到本地仓库  
git remote add origin git@github.com:xxxx.github.io.git  #将本地仓库关联到GitHub仓库；这里也可以用http的那个链接  
git push -u origin master  #本地仓库上传到GitHub仓库（可能会要求输入用户名和密码）
```

>这里如果`git push -u origin master `时候报错说：
>! [rejected]        master -> master (non-fast-forward)
>error: failed to push some refs to 'github.com:gxu-yuan/gxu-yuan.github.io.git'
>hint: Updates were rejected because the tip of your current branch is behind
>hint: its remote counterpart. If you want to integrate the remote changes,
>hint: use 'git pull' before pushing again.
>hint: See the 'Note about fast-forwards' in 'git push --help' for details.
>则，我们直接使用这个命令：
{: .prompt-warning }

```bash
git push -f origin master
```

至此，我们就已经初始化完成的模板上传完成了。之后打开GitHub仓库网址，点击上访工具栏最右边的`Settings`，之后点击左侧菜单列的`Pages`，在`Build and Development`中将`Source`选项选择`GitHub Actions`。 这意味着可以通过文件的修改来触发主页的发布，之后一般等一段时间就可以在`Pages`页面右上角看到主页已发布的信息。

## 0x03 附录

### ssh建立连接部分

#### 1）创建ssh密钥

1、在git bash中输入：

```bash
cd ~/.ssh
ssh-keygen -t rsa -C "youremail@example.com"
```

注：如果没有该目录，则需要创建一个；创建key时，双引号中填写你自己注册时用的邮箱名，除此之外还需要输入存放地址（当然默认的话，回车即可）；然后创建完后，会提示你密钥存放地址。

2、进入你id_rsa和id_rsa.pub存放的目录下，复制id_rsa.pub的内容，然后登陆自己的GitHub账户，找到setting中的SSH keys and GPG keys选项，点击创建New SSH key。然后，将id_rsa.pub里的内容拷贝到Key内，Title内容随便填，确定即可。

#### 2）初始化本地仓库

1、右击我们之前创建的博客文件夹，然后选择git bash here选项，打bash命令窗口。

2、输入命令git init 把这个文件夹变成git本地可管理仓库：

```bash
git init
```

此时，文件夹内部就有了个.git的隐藏文件夹。

3、登陆配置：

```bash
git config --global user.email "你的Github邮箱地址"
git config --global user.name "你的Github用户名""
```

## 0x04 BUG记录

### dist文件夹缺失部分

由于之前再文件初始化部分有可能导致dist文件夹的缺失，而在GitHub上部署的时候会出现如下的部署错误：（在自己的仓库下面的`Action`下可以看见工作流的信息）

```markdown
* At _site/tags/index.html:1:

 internal script reference /assets/js/dist/commons.min.js does not exist
```

修复方法是： 我们可以从官方的[demo的dist文件夹](https://github.com/cotes2020/chirpy-demo/tree/main/assets/js/dist)中发现确实的`JS文件`，然后我们需要去重新上传到我们GitHub仓库中的`assets/js/dist`{.filepath}中。

这里，不知道为什么我不能本地上传这些文件。。。。 所以我决定直接在GitHub上操作。进入到`assets/js/`{.filepath}之中，然后点击右上角的`Add file`，然后点击`Create new file`。此时，在左上方我们可以看见路径，我们需要在那个`name your dile`处填入下面内容：

```markdown
dist/1.txt
```

下面的内容随便输入，然后点击`commit changes`即可。 最后，我们在进入这个`dist`目录处，点击上传文件，把我们之前下载的JS文件上传即可解决上述bug。



[^Chirpy]:https://chirpy.cotes.page/posts/getting-started/
[^fange]:https://fange12306.github.io
