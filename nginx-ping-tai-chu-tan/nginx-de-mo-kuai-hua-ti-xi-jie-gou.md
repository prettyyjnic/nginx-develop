# nginx的模块化体系结构

nginx的内部结构是由核心部分和一系列的功能模块所组成。这样划分是为了使得每个模块的功能相对简单，便于开发，同时也便于对系统进行功能扩展。为了便于描述，下文中我们将使用nginx core来称呼nginx的核心功能部分。

nginx提供了web服务器的基础功能，同时提供了web服务反向代理，email服务反向代理功能。nginx core实现了底层的通讯协议，为其他模块和nginx进程构建了基本的运行时环境，并且构建了其他各模块的协作基础。除此之外，或者说大部分与协议相关的，或者应用相关的功能都是在这些模块中所实现的。

### 模块概述

nginx将各功能模块组织成一条链，当有请求到达的时候，请求依次经过这条链上的部分或者全部模块，进行处理。每个模块实现特定的功能。例如，实现对请求解压缩的模块，实现SSI的模块，实现与上游服务器进行通讯的模块，实现与FastCGI服务进行通讯的模块。

有两个模块比较特殊，他们居于nginx core和各功能模块的中间。这两个模块就是http模块和mail模块。这2个模块在nginx core之上实现了另外一层抽象，处理与HTTP协议和email相关协议（SMTP/POP3/IMAP）有关的事件，并且确保这些事件能被以正确的顺序调用其他的一些功能模块。

目前HTTP协议是被实现在http模块中的，但是有可能将来被剥离到一个单独的模块中，以扩展nginx支持SPDY协议。

#### 模块的分类

nginx的模块根据其功能基本上可以分为以下几种类型：

| event module: | 搭建了独立于操作系统的事件处理机制的框架，及提供了各具体事件的处理。包括ngx\_events\_module， ngx\_event\_core\_module和ngx\_epoll\_module等。nginx具体使用何种事件处理模块，这依赖于具体的操作系统和编译选项。 |
| --- | --- | --- | --- | --- |
| phase handler: | 此类型的模块也被直接称为handler模块。主要负责处理客户端请求并产生待响应内容，比如ngx\_http\_static\_module模块，负责客户端的静态页面请求处理并将对应的磁盘文件准备为响应内容输出。 |
| output filter: | 也称为filter模块，主要是负责对输出的内容进行处理，可以对输出进行修改。例如，可以实现对输出的所有html页面增加预定义的footbar一类的工作，或者对输出的图片的URL进行替换之类的工作。 |
| upstream: | upstream模块实现反向代理的功能，将真正的请求转发到后端服务器上，并从后端服务器上读取响应，发回客户端。upstream模块是一种特殊的handler，只不过响应内容不是真正由自己产生的，而是从后端服务器上读取的。 |
| load-balancer: | 负载均衡模块，实现特定的算法，在众多的后端服务器中，选择一个服务器出来作为某个请求的转发服务器。 |

### nginx的请求处理

nginx使用一个多进程模型来对外提供服务，其中一个master进程，多个worker进程。master进程负责管理nginx本身和其他worker进程。

所有实际上的业务处理逻辑都在worker进程。worker进程中有一个函数，执行无限循环，不断处理收到的来自客户端的请求，并进行处理，直到整个nginx服务被停止。

worker进程中，ngx\_worker\_process\_cycle\(\)函数就是这个无限循环的处理函数。在这个函数中，一个请求的简单处理流程如下：

1. 操作系统提供的机制（例如epoll, kqueue等）产生相关的事件。
2. 接收和处理这些事件，如是接受到数据，则产生更高层的request对象。
3. 处理request的header和body。
4. 产生响应，并发送回客户端。
5. 完成request的处理。
6. 重新初始化定时器及其他事件。

#### 请求的处理流程

为了让大家更好的了解nginx中请求处理过程，我们以HTTP Request为例，来做一下详细地说明。

从nginx的内部来看，一个HTTP Request的处理过程涉及到以下几个阶段。

1. 初始化HTTP Request（读取来自客户端的数据，生成HTTP Request对象，该对象含有该请求所有的信息）。
2. 处理请求头。
3. 处理请求体。
4. 如果有的话，调用与此请求（URL或者Location）关联的handler。
5. 依次调用各phase handler进行处理。

在这里，我们需要了解一下phase handler这个概念。phase字面的意思，就是阶段。所以phase handlers也就好理解了，就是包含若干个处理阶段的一些handler。

在每一个阶段，包含有若干个handler，再处理到某个阶段的时候，依次调用该阶段的handler对HTTP Request进行处理。

通常情况下，一个phase handler对这个request进行处理，并产生一些输出。通常phase handler是与定义在配置文件中的某个location相关联的。

一个phase handler通常执行以下几项任务：

1. 获取location配置。
2. 产生适当的响应。
3. 发送response header。
4. 发送response body。

当nginx读取到一个HTTP Request的header的时候，nginx首先查找与这个请求关联的虚拟主机的配置。如果找到了这个虚拟主机的配置，那么通常情况下，这个HTTP Request将会经过以下几个阶段的处理（phase handlers）：

| NGX\_HTTP\_POST\_READ\_PHASE: | 读取请求内容阶段 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
|  |  |
| NGX\_HTTP\_SERVER\_REWRITE\_PHASE: |  |
|   | Server请求地址重写阶段 |
| NGX\_HTTP\_FIND\_CONFIG\_PHASE: |  |
|   | 配置查找阶段: |
| NGX\_HTTP\_REWRITE\_PHASE: |  |
|   | Location请求地址重写阶段 |
| NGX\_HTTP\_POST\_REWRITE\_PHASE: |  |
|   | 请求地址重写提交阶段 |
| NGX\_HTTP\_PREACCESS\_PHASE: |  |
|   | 访问权限检查准备阶段 |
| NGX\_HTTP\_ACCESS\_PHASE: |  |
|   | 访问权限检查阶段 |
| NGX\_HTTP\_POST\_ACCESS\_PHASE: |  |
|   | 访问权限检查提交阶段 |
| NGX\_HTTP\_TRY\_FILES\_PHASE: |  |
|   | 配置项try\_files处理阶段 |
| NGX\_HTTP\_CONTENT\_PHASE: |  |
|   | 内容产生阶段 |
| NGX\_HTTP\_LOG\_PHASE: |  |
|   | 日志模块处理阶段 |

在内容产生阶段，为了给一个request产生正确的响应，nginx必须把这个request交给一个合适的content handler去处理。如果这个request对应的location在配置文件中被明确指定了一个content handler，那么nginx就可以通过对location的匹配，直接找到这个对应的handler，并把这个request交给这个content handler去处理。这样的配置指令包括像，perl，flv，proxy\_pass，mp4等。

如果一个request对应的location并没有直接有配置的content handler，那么nginx依次尝试:

1. 如果一个location里面有配置 random\_index on，那么随机选择一个文件，发送给客户端。
2. 如果一个location里面有配置 index指令，那么发送index指令指明的文件，给客户端。
3. 如果一个location里面有配置 autoindex on，那么就发送请求地址对应的服务端路径下的文件列表给客户端。
4. 如果这个request对应的location上有设置gzip\_static on，那么就查找是否有对应的.gz文件存在，有的话，就发送这个给客户端（客户端支持gzip的情况下）。
5. 请求的URI如果对应一个静态文件，static module就发送静态文件的内容到客户端。

内容产生阶段完成以后，生成的输出会被传递到filter模块去进行处理。filter模块也是与location相关的。所有的fiter模块都被组织成一条链。输出会依次穿越所有的filter，直到有一个filter模块的返回值表明已经处理完成。

这里列举几个常见的filter模块，例如：

1. server-side includes。
2. XSLT filtering。
3. 图像缩放之类的。
4. gzip压缩。

在所有的filter中，有几个filter模块需要关注一下。按照调用的顺序依次说明如下：

| write: | 写输出到客户端，实际上是写到连接对应的socket上。 |
| --- | --- | --- |
| postpone: | 这个filter是负责subrequest的，也就是子请求的。 |
| copy: | 将一些需要复制的buf\(文件或者内存\)重新复制一份然后交给剩余的body filter处理。 |

