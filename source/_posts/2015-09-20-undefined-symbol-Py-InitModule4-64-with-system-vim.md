title: 'undefined symbol: Py_InitModule4_64 with system vim'

date: 2015-09-20 23:24:29

tags:
    - Linux
    - Ubuntu
    - Vim
---

### 背景
由于给系统安装了最新的python而导致了一些比较奇特的现象，运行apt-get安装的vim老是出现段错误，并提示undefined symbol: Py_InitModule4_64

### 原因
利用ldd命令查看/usr/bin/vim, 发现了libpython2.7.so.1.0 指向的动态库不是存在与/usr/lib中的，而是存在与新编译安装的python(/usr/local/)中的

(/usr/local中的文件在系统中的命令有着相对于 /usr目录中的目录有着优先权，这是因为PATH和/etc/ld.so.conf文件与/etc/ld.so.conf.d目录中把/usr/local/lib放在/usr/lib的前面)

### 修改方法
修改一下/etc/ld.so.conf.d/libc.conf文件中的内容，在/usr/local/lib前添加一条
        /usr/lib
然后在运行ldconfig命令进行动态库连接的更新，这样才能统治系统修改了动态库的一些规则。
