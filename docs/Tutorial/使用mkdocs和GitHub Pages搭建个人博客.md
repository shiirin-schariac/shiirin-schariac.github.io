## 前言

鉴于网上关于GitHub pages的建站教程有点少，而且其中大部分对小白不是很友好，所以想着写一篇面向大众的零基础教程，因为我使用的是Mac，本文以macOS为主，Windows操作是差不多的，遇到不互通的指令Google一下就行。

### 在开始之前你需要准备的材料

1. 一个GitHub账号

2. 一个准备完全的终端
   
      macOS上使用⌘+space打开聚焦搜索，输入terminal或终端打开终端.app，输入以下指令
```shell
   $ pip --version
   # 如果返回pip not found 请执行下面的步骤安装pip
   $ python3 --version # 确保你安装了python3（macOS应该自带了的
   $ curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
   $ python3 get-pip.py
   $ pip --version
   > pip 23.0.1 from /opt/homebrew/lib/python3.11/site-packages/pip (python 3.11)
   $ git --version
   # 安装命令行或Xcode
```
         

## 正文

### 如何将你的GitHub与你的设备建立SSH连接

检查你是否已有GitHub的SSH连接

```shell
   $ ls -al ~/.ssh
   # 观察返回的ssh中有无id_rsa.pub, id_ecdsa.pub, id_ed25519.pub, 若无则该设备上没有与GitHub的ssh连接
```



**生成新的SSH密钥**

```shell
   $ ssh-keygen -t ed25519 -C "your_email@example.com" # 用你的邮件地址替换your_email@example.com
   > Generating public/private ALGORITHM key pair. #ALGORITHM会是一段乱码
   > Enter a file in which to save the key (/Users/YOU/.ssh/id_ALGORITHM: [Press enter] 
   # 直接按回车就行
   > Enter passphrase (empty for no passphrase): [Type a passphrase]
   > Enter same passphrase again: [Type passphrase again]
   # 按提示输入密码 要输入两次
```

   
**将新的SSH密钥添加到SSH代理中**

```shell
   $ eval "$(ssh-agent -s)"
   > Agent pid 59566
   $ vim ~/.ssh/config
   # 按I进入编辑模式 复制粘贴以下行
   Host github.com
     AddKeysToAgent yes
     UseKeychain yes
     IdentityFile ~/.ssh/id_ed25519
   # 按ESC后输入 :wq 保存并退出编辑
   $ ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

   

**向GitHub账户添加SSH密钥**

   1. 登入你的GitHub账户
   2. 右上角个人头像 => Settings => SSH and GPG keys => New SSH key button
   3. 在终端中输入`pbcopy < ~/.ssh/id_ed25519.pub`
   4. 在Key部分粘贴刚刚复制的部分，单击Add SSH key

**检查连接是否成功**

```shell
   $ ssh -T git@github.com
   > Hi USERNAME! You've successfully authenticated, but GitHub does not
   > provide shell access.
```

   

### 下载mkdocs与material

**分别在终端中输入以下指令**

```shell
   $ pip install mkdocs
   $ pip install mkdocs-material
```

**新建mkdocs项目，并尝试在本地运行**

```shell
   $ mkdocs new my_site
   $ cd my_site
   $ mkdocs serve
   INFO    -  Building documentation...
   INFO    -  Cleaning site directory
   [I 160402 15:50:43 server:271] Serving on http://127.0.0.1:8000
   # 复制到浏览器，运行，是最初的mkdocs界面
   [I 160402 15:50:43 handlers:58] Start watching changes
   [I 160402 15:50:43 handlers:60] Start detecting changes
```
**用你的ide打开my_site下的mkdocs.yml, 在其中添加如下行代码**



```shell
   theme:
     name: material
```

   

   刷新刚才的网页，你会发现它已经变成了material提供的主题

### 使用git关联本地仓库与远程仓库

1. 在GitHub上新建一个仓库(New repository), 命名为Username.github.io(Username一定要和用户名一致)
   在该仓库中找到Settings => Actions => General, 更改如下设置并**点击Save保存**
   1.  Actions permissions: Allow all actions and reusable workflows
   2.  Workflow permissions: Read and write permissions

2. 打开终端，输入如下指令

```shell
   # 首先你应该在my_site文件夹中，如果不在，请先cd my_site
   $ mkdir .github
   $ cd .github
   $ mkdir workflows
   $ cd workflows
   $ vim PublishMySite.yml
   # 按I进入编辑模式，复制粘贴以下行
   name: ci # 这个可以改
   on:
     push:
       branches:
         - master 
         - main
   permissions:
     contents: write
   jobs:
     deploy:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v3
         - uses: actions/setup-python@v4
           with:
             python-version: 3.x
         - uses: actions/cache@v2
           with:
             key: ${{ github.ref }}
             path: .cache
         - run: pip install mkdocs-material 
         - run: mkdocs gh-deploy --force
   # 按ESC后输入 :wq 保存并退出编辑
   $ git init
   $ git add .
   $ git commit -m "init"
   
   $ git remote add origin git@github.com:Username/Username.github.io.git
   # 把Username替换成你的用户名
   $ git branch -M main
   $ git push -u origin main
   # 如果报错，可以执行git push -f origin main
```

   进入Settings => Pages 中提示的网址，应该已经构建成功（可能需要等几分钟

   如果没有成功，请检查Settings => Pages => Source 中是否选择gh pages作为发布源

### 后续如何更新博客

进入你的ide，编辑/写博客后，建议先使用`mkdocs serve`看看效果，如果决定更新博客，可以在终端中输入如下指令

```shell
   $ cd my_site
   $ git add .
   $ git commit -m "tip" # 双引号内写备注
   $ git push -u origin main
```

进行刷新，应该已经实现更新



### 推荐阅读

[GitHub_SSH link](https://squidfunk.github.io/mkdocs-material/)

[mkdocs_material documents](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/about-ssh)