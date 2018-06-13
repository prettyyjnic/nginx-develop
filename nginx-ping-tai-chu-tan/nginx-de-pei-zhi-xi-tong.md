# nginx的配置系统

nginx的配置系统由一个主配置文件和其他一些辅助的配置文件构成。这些配置文件均是纯文本文件，全部位于nginx安装目录下的conf目录下。

配置文件中以\#开始的行，或者是前面有若干空格或者TAB，然后再跟\#的行，都被认为是注释，也就是只对编辑查看文件的用户有意义，程序在读取这些注释行的时候，其实际的内容是被忽略的。

由于除主配置文件nginx.conf以外的文件都是在某些情况下才使用的，而只有主配置文件是在任何情况下都被使用的。所以在这里我们就以主配置文件为例，来解释nginx的配置系统。

在nginx.conf中，包含若干配置项。每个配置项由配置指令和指令参数2个部分构成。指令参数也就是配置指令对应的配置值。

#### 指令概述

配置指令是一个字符串，可以用单引号或者双引号括起来，也可以不括。但是如果配置指令包含空格，一定要引起来。

#### 指令参数

指令的参数使用一个或者多个空格或者TAB字符与指令分开。指令的参数有一个或者多个TOKEN串组成。TOKEN串之间由空格或者TAB键分隔。

TOKEN串分为简单字符串或者是复合配置块。复合配置块即是由大括号括起来的一堆内容。一个复合配置块中可能包含若干其他的配置指令。

如果一个配置指令的参数全部由简单字符串构成，也就是不包含复合配置块，那么我们就说这个配置指令是一个简单配置项，否则称之为复杂配置项。例如下面这个是一个简单配置项：

```text
error_page   500 502 503 504  /50x.html;
```

对于简单配置，配置项的结尾使用分号结束。对于复杂配置项，包含多个TOKEN串的，一般都是简单TOKEN串放在前面，复合配置块一般位于最后，而且其结尾，并不需要再添加分号。例如下面这个复杂配置项：

```text
location / {
    root   /home/jizhao/nginx-book/build/html;
    index  index.html index.htm;
}
```

#### 指令上下文

nginx.conf中的配置信息，根据其逻辑上的意义，对它们进行了分类，也就是分成了多个作用域，或者称之为配置指令上下文。不同的作用域含有一个或者多个配置项。

当前nginx支持的几个指令上下文：

| main: | nginx在运行时与具体业务功能（比如http服务或者email服务代理）无关的一些参数，比如工作进程数，运行的身份等。 |
| --- | --- | --- | --- | --- |
| http: | 与提供http服务相关的一些配置参数。例如：是否使用keepalive啊，是否使用gzip进行压缩等。 |
| server: | http服务上支持若干虚拟主机。每个虚拟主机一个对应的server配置项，配置项里面包含该虚拟主机相关的配置。在提供mail服务的代理时，也可以建立若干server.每个server通过监听的地址来区分。 |
| location: | http服务中，某些特定的URL对应的一系列配置项。 |
| mail: | 实现email相关的SMTP/IMAP/POP3代理时，共享的一些配置项（因为可能实现多个代理，工作在多个监听地址上）。 |

指令上下文，可能有包含的情况出现。例如：通常http上下文和mail上下文一定是出现在main上下文里的。在一个上下文里，可能包含另外一种类型的上下文多次。例如：如果http服务，支持了多个虚拟主机，那么在http上下文里，就会出现多个server上下文。

我们来看一个示例配置：

```text
user  nobody;
worker_processes  1;
error_log  logs/error.log  info;

events {
    worker_connections  1024;
}

http {
    server {
        listen          80;
        server_name     www.linuxidc.com;
        access_log      logs/linuxidc.access.log main;
        location / {
            index index.html;
            root  /var/www/linuxidc.com/htdocs;
        }
    }

    server {
        listen          80;
        server_name     www.Androidj.com;
        access_log      logs/androidj.access.log main;
        location / {
            index index.html;
            root  /var/www/androidj.com/htdocs;
        }
    }
}

mail {
    auth_http  127.0.0.1:80/auth.php;
    pop3_capabilities  "TOP"  "USER";
    imap_capabilities  "IMAP4rev1"  "UIDPLUS";

    server {
        listen     110;
        protocol   pop3;
        proxy      on;
    }
    server {
        listen      25;
        protocol    smtp;
        proxy       on;
        smtp_auth   login plain;
        xclient     off;
    }
}
```

在这个配置中，上面提到个五种配置指令上下文都存在。

存在于main上下文中的配置指令如下:

* user
* worker\_processes
* error\_log
* events
* http
* mail

存在于http上下文中的指令如下:

* server

存在于mail上下文中的指令如下：

* server
* auth\_http
* imap\_capabilities

存在于server上下文中的配置指令如下：

* listen
* server\_name
* access\_log
* location
* protocol
* proxy
* smtp\_auth
* xclient

存在于location上下文中的指令如下：

* index
* root

当然，这里只是一些示例。具体有哪些配置指令，以及这些配置指令可以出现在什么样的上下文中，需要参考nginx的使用文档。

