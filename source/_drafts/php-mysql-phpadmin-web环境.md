title: php+mysql+phpadmin web环境
tags:
---

## 环境的安装
需要安装的软件：
- php
- mysql
- apache2
- phpmyadmin


- 软件的安装
    ```
    $ sudo apt-get install mysql-client mysql-server apache2 php-mysql  libapache2-mod-php phpmyadmin
    ```
    其中这里需要安装mysql的服务端和客户端，分别是包mysql-client和mysql-server
- 其中安装好apache2后，apache2的默认的配置文件是/et/apache2/apache2.conf
根据`apache2.conf`我们可知网页的目录是在`/var/www`下，*其中根据网上说的，这里就是apache2的网页目录，其实并不然，因为`index.html`是在`/var/www/html/`下*

- 在上面的安装过程中，mysql是要设置root用户密码的，而phpmyadmin也是要设置root用户的密码的

- 首先测试php安装信息，这里我们在`/var/www/html/`下编写一个`info.php`文件，文件内容为：
```
<?php
phpinfo()
?>
```

然后在浏览器上输入`http://localhost/php.info`就可以查看有关php的安装信息，并且可以知道php是否正常的工作。这里我们可以通过观察这个页面我们就可以了解到apache2的文档路径

phpmyadmin是一个网络接口,通过它可以管理你的mysql数据库。

apt-get install phpmyadmin

phpmyadmin会自动安装在/usr/share/phpMyAdmin下，然后将phpMyAdmin拷贝到/var/www/html目录下面，运行http://localhost/phpmyadmin/即可通过phpmyadmin使用phpmyadmin
