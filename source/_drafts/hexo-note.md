title: hexo note
tags:
    - hexo
    - blog
-----------------

记录学习hexo的一些命令
-------------------
### HEXO 的安装

首先需要安装的是git和node.js             
[ubuntu] git
```
    $ sudo apt-get install git-core
```
[ubuntu] node.js
```
wget -qO- https://raw.github.com/creationix/nvm/master/install.sh | sh

nvm install stable
```

最后是hexo的安装
```
npm install -g hexo-cli
```

### 下面是记录一些hexo命令
这里主要是记录使用hexo记录的一些命令，为了防止在忘记了命令后快速的找到记录

#### 初始化目录为hexo文件目录
```
hexo init
```

#### 新建文章

```
Usage: hexo new [layout] <title>

Arguments:
  layout  Post layout. Use post, page, draft or whatever you want.
  title   Post title. Wrap it with quotations to escape.

```
当我们新建的


#### 在浏览器中查看

在hexo初始化的根目录执行
```
hexo server
```
