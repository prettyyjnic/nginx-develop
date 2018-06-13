# 基本数据结构

nginx的作者为追求极致的高效，自己实现了很多颇具特色的nginx风格的数据结构以及公共函数。比如，nginx提供了带长度的字符串，根据编译器选项优化过的字符串拷贝函数ngx\_copy等。所以，在我们写nginx模块时，应该尽量调用nginx提供的api，尽管有些api只是对glibc的宏定义。本节，我们介绍string、list、buffer、chain等一系列最基本的数据结构及相关api的使用技巧以及注意事项。

#### ngx\_str\_t\(100%\)

在nginx源码目录的src/core下面的ngx\_string.h\|c里面，包含了字符串的封装以及字符串相关操作的api。nginx提供了一个带长度的字符串结构ngx\_str\_t，它的原型如下：

```text
typedef struct {
    size_t      len;
    u_char     *data;
} ngx_str_t;
```

在结构体当中，data指向字符串数据的第一个字符，字符串的结束用长度来表示，而不是由’\0’来表示结束。所以，在写nginx代码时，处理字符串的方法跟我们平时使用有很大的不一样，但要时刻记住，字符串不以’\0’结束，尽量使用nginx提供的字符串操作的api来操作字符串。 那么，nginx这样做有什么好处呢？首先，通过长度来表示字符串长度，减少计算字符串长度的次数。其次，nginx可以重复引用一段字符串内存，data可以指向任意内存，长度表示结束，而不用去copy一份自己的字符串\(因为如果要以’\0’结束，而不能更改原字符串，所以势必要copy一段字符串\)。我们在ngx\_http\_request\_t结构体的成员中，可以找到很多字符串引用一段内存的例子，比如request\_line、uri、args等等，这些字符串的data部分，都是指向在接收数据时创建buffer所指向的内存中，uri，args就没有必要copy一份出来。这样的话，减少了很多不必要的内存分配与拷贝。 正是基于此特性，在nginx中，必须谨慎的去修改一个字符串。在修改字符串时需要认真的去考虑：是否可以修改该字符串；字符串修改后，是否会对其它的引用造成影响。在后面介绍ngx\_unescape\_uri函数的时候，就会看到这一点。但是，使用nginx的字符串会产生一些问题，glibc提供的很多系统api函数大多是通过’\0’来表示字符串的结束，所以我们在调用系统api时，就不能直接传入str-&gt;data了。此时，通常的做法是创建一段str-&gt;len + 1大小的内存，然后copy字符串，最后一个字节置为’\0’。比较hack的做法是，将字符串最后一个字符的后一个字符backup一个，然后设置为’\0’，在做完调用后，再由backup改回来，但前提条件是，你得确定这个字符是可以修改的，而且是有内存分配，不会越界，但一般不建议这么做。 接下来，看看nginx提供的操作字符串相关的api。

```text
#define ngx_string(str)     { sizeof(str) - 1, (u_char *) str }

```

ngx\_string\(str\)是一个宏，它通过一个以’\0’结尾的普通字符串str构造一个nginx的字符串，鉴于其中采用sizeof操作符计算字符串长度，因此参数必须是一个常量字符串。

```text
#define ngx_null_string     { 0, NULL }

```

定义变量时，使用ngx\_null\_string初始化字符串为空字符串，符串的长度为0，data为NULL。

```text
#define ngx_str_set(str, text)                                               \
    (str)->len = sizeof(text) - 1; (str)->data = (u_char *) text

```

ngx\_str\_set用于设置字符串str为text，由于使用sizeof计算长度，故text必须为常量字符串。

```text
#define ngx_str_null(str)   (str)->len = 0; (str)->data = NULL

```

ngx\_str\_null用于设置字符串str为空串，长度为0，data为NULL。

上面这四个函数，使用时一定要小心，ngx\_string与ngx\_null\_string是“{_，_}”格式的，故只能用于赋值时初始化，如：

```text
ngx_str_t str = ngx_string("hello world");
ngx_str_t str1 = ngx_null_string;
```

如果向下面这样使用，就会有问题，这里涉及到c语言中对结构体变量赋值操作的语法规则，在此不做介绍。

```text
ngx_str_t str, str1;
str = ngx_string("hello world");    // 编译出错
str1 = ngx_null_string;                // 编译出错

```

这种情况，可以调用ngx\_str\_set与ngx\_str\_null这两个函数来做:

```text
ngx_str_t str, str1;
ngx_str_set(&str, "hello world");
ngx_str_null(&str1);
```

按照C99标准，您也可以这么做：

```text
ngx_str_t str, str1;
str  = (ngx_str_t) ngx_string("hello world");
str1 = (ngx_str_t) ngx_null_string;
```

另外要注意的是，ngx\_string与ngx\_str\_set在调用时，传进去的字符串一定是常量字符串，否则会得到意想不到的错误\(因为ngx\_str\_set内部使用了sizeof\(\)，如果传入的是u\_char\*，那么计算的是这个指针的长度，而不是字符串的长度\)。如：

```text
ngx_str_t str;
u_char *a = "hello world";
ngx_str_set(&str, a);    // 问题产生

```

此外，值得注意的是，由于ngx\_str\_set与ngx\_str\_null实际上是两行语句，故在if/for/while等语句中单独使用需要用花括号括起来，例如：

```text
ngx_str_t str;
if (cond)
   ngx_str_set(&str, "true");     // 问题产生
else
   ngx_str_set(&str, "false");    // 问题产生

```

```text
void ngx_strlow(u_char *dst, u_char *src, size_t n);
```

将src的前n个字符转换成小写存放在dst字符串当中，调用者需要保证dst指向的空间大于等于n，且指向的空间必须可写。操作不会对原字符串产生变动。如要更改原字符串，可以：

```text
ngx_strlow(str->data, str->data, str->len);
```

```text
ngx_strncmp(s1, s2, n)
```

区分大小写的字符串比较，只比较前n个字符。

```text
ngx_strcmp(s1, s2)
```

区分大小写的不带长度的字符串比较。

```text
ngx_int_t ngx_strcasecmp(u_char *s1, u_char *s2);
```

不区分大小写的不带长度的字符串比较。

```text
ngx_int_t ngx_strncasecmp(u_char *s1, u_char *s2, size_t n);
```

不区分大小写的带长度的字符串比较，只比较前n个字符。

```text
u_char * ngx_cdecl ngx_sprintf(u_char *buf, const char *fmt, ...);
u_char * ngx_cdecl ngx_snprintf(u_char *buf, size_t max, const char *fmt, ...);
u_char * ngx_cdecl ngx_slprintf(u_char *buf, u_char *last, const char *fmt, ...);
```

上面这三个函数用于字符串格式化，ngx\_snprintf的第二个参数max指明buf的空间大小，ngx\_slprintf则通过last来指明buf空间的大小。推荐使用第二个或第三个函数来格式化字符串，ngx\_sprintf函数还是比较危险的，容易产生缓冲区溢出漏洞。在这一系列函数中，nginx在兼容glibc中格式化字符串的形式之外，还添加了一些方便格式化nginx类型的一些转义字符，比如%V用于格式化ngx\_str\_t结构。在nginx源文件的ngx\_string.c中有说明：

```text
/*
 * supported formats:
 *    %[0][width][x][X]O        off_t
 *    %[0][width]T              time_t
 *    %[0][width][u][x|X]z      ssize_t/size_t
 *    %[0][width][u][x|X]d      int/u_int
 *    %[0][width][u][x|X]l      long
 *    %[0][width|m][u][x|X]i    ngx_int_t/ngx_uint_t
 *    %[0][width][u][x|X]D      int32_t/uint32_t
 *    %[0][width][u][x|X]L      int64_t/uint64_t
 *    %[0][width|m][u][x|X]A    ngx_atomic_int_t/ngx_atomic_uint_t
 *    %[0][width][.width]f      double, max valid number fits to %18.15f
 *    %P                        ngx_pid_t
 *    %M                        ngx_msec_t
 *    %r                        rlim_t
 *    %p                        void *
 *    %V                        ngx_str_t *
 *    %v                        ngx_variable_value_t *
 *    %s                        null-terminated string
 *    %*s                       length and string
 *    %Z                        '\0'
 *    %N                        '\n'
 *    %c                        char
 *    %%                        %
 *
 *  reserved:
 *    %t                        ptrdiff_t
 *    %S                        null-terminated wchar string
 *    %C                        wchar
 */
```

这里特别要提醒的是，我们最常用于格式化ngx\_str\_t结构，其对应的转义符是%V，传给函数的一定要是指针类型，否则程序就会coredump掉。这也是我们最容易犯的错。比如：

```text
ngx_str_t str = ngx_string("hello world");
char buffer[1024];
ngx_snprintf(buffer, 1024, "%V", &str);    // 注意，str取地址

```

```text
void ngx_encode_base64(ngx_str_t *dst, ngx_str_t *src);
ngx_int_t ngx_decode_base64(ngx_str_t *dst, ngx_str_t *src);
```

这两个函数用于对str进行base64编码与解码，调用前，需要保证dst中有足够的空间来存放结果，如果不知道具体大小，可先调用ngx\_base64\_encoded\_length与ngx\_base64\_decoded\_length来预估最大占用空间。

```text
uintptr_t ngx_escape_uri(u_char *dst, u_char *src, size_t size,
    ngx_uint_t type);
```

对src进行编码，根据type来按不同的方式进行编码，如果dst为NULL，则返回需要转义的字符的数量，由此可得到需要的空间大小。type的类型可以是：

```text
#define NGX_ESCAPE_URI         0
#define NGX_ESCAPE_ARGS        1
#define NGX_ESCAPE_HTML        2
#define NGX_ESCAPE_REFRESH     3
#define NGX_ESCAPE_MEMCACHED   4
#define NGX_ESCAPE_MAIL_AUTH   5

```

```text
void ngx_unescape_uri(u_char **dst, u_char **src, size_t size, ngx_uint_t type);
```

对src进行反编码，type可以是0、NGX\_UNESCAPE\_URI、NGX\_UNESCAPE\_REDIRECT这三个值。如果是0，则表示src中的所有字符都要进行转码。如果是NGX\_UNESCAPE\_URI与NGX\_UNESCAPE\_REDIRECT，则遇到’?’后就结束了，后面的字符就不管了。而NGX\_UNESCAPE\_URI与NGX\_UNESCAPE\_REDIRECT之间的区别是NGX\_UNESCAPE\_URI对于遇到的需要转码的字符，都会转码，而NGX\_UNESCAPE\_REDIRECT则只会对非可见字符进行转码。

```text
uintptr_t ngx_escape_html(u_char *dst, u_char *src, size_t size);
```

对html标签进行编码。

当然，我这里只介绍了一些常用的api的使用，大家可以先熟悉一下，在实际使用过程中，遇到不明白的，最快最直接的方法就是去看源码，看api的实现或看nginx自身调用api的地方是怎么做的，代码就是最好的文档。

#### ngx\_pool\_t\(100%\)

ngx\_pool\_t是一个非常重要的数据结构，在很多重要的场合都有使用，很多重要的数据结构也都在使用它。那么它究竟是一个什么东西呢？简单的说，它提供了一种机制，帮助管理一系列的资源（如内存，文件等），使得对这些资源的使用和释放统一进行，免除了使用过程中考虑到对各种各样资源的什么时候释放，是否遗漏了释放的担心。

例如对于内存的管理，如果我们需要使用内存，那么总是从一个ngx\_pool\_t的对象中获取内存，在最终的某个时刻，我们销毁这个ngx\_pool\_t对象，所有这些内存都被释放了。这样我们就不必要对对这些内存进行malloc和free的操作，不用担心是否某块被malloc出来的内存没有被释放。因为当ngx\_pool\_t对象被销毁的时候，所有从这个对象中分配出来的内存都会被统一释放掉。

再比如我们要使用一系列的文件，但是我们打开以后，最终需要都关闭，那么我们就把这些文件统一登记到一个ngx\_pool\_t对象中，当这个ngx\_pool\_t对象被销毁的时候，所有这些文件都将会被关闭。

从上面举的两个例子中我们可以看出，使用ngx\_pool\_t这个数据结构的时候，所有的资源的释放都在这个对象被销毁的时刻，统一进行了释放，那么就会带来一个问题，就是这些资源的生存周期（或者说被占用的时间）是跟ngx\_pool\_t的生存周期基本一致（ngx\_pool\_t也提供了少量操作可以提前释放资源）。从最高效的角度来说，这并不是最好的。比如，我们需要依次使用A，B，C三个资源，且使用完B的时候，A就不会再被使用了，使用C的时候A和B都不会被使用到。如果不使用ngx\_pool\_t来管理这三个资源，那我们可能从系统里面申请A，使用A，然后在释放A。接着申请B，使用B，再释放B。最后申请C，使用C，然后释放C。但是当我们使用一个ngx\_pool\_t对象来管理这三个资源的时候，A，B和C的释放是在最后一起发生的，也就是在使用完C以后。诚然，这在客观上增加了程序在一段时间的资源使用量。但是这也减轻了程序员分别管理三个资源的生命周期的工作。这也就是有所得，必有所失的道理。实际上是一个取舍的问题，要看在具体的情况下，你更在乎的是哪个。

可以看一下在nginx里面一个典型的使用ngx\_pool\_t的场景，对于nginx处理的每个http request, nginx会生成一个ngx\_pool\_t对象与这个http request关联，所有处理过程中需要申请的资源都从这个ngx\_pool\_t对象中获取，当这个http request处理完成以后，所有在处理过程中申请的资源，都将随着这个关联的ngx\_pool\_t对象的销毁而释放。

ngx\_pool\_t相关结构及操作被定义在文件src/core/ngx\_palloc.h\|c中。

```text
typedef struct ngx_pool_s        ngx_pool_t;

struct ngx_pool_s {
    ngx_pool_data_t       d;
    size_t                max;
    ngx_pool_t           *current;
    ngx_chain_t          *chain;
    ngx_pool_large_t     *large;
    ngx_pool_cleanup_t   *cleanup;
    ngx_log_t            *log;
};
```

从ngx\_pool\_t的一般使用者的角度来说，可不用关注ngx\_pool\_t结构中各字段作用。所以这里也不会进行详细的解释，当然在说明某些操作函数的使用的时候，如有必要，会进行说明。

下面我们来分别解释下ngx\_pool\_t的相关操作。

```text
ngx_pool_t *ngx_create_pool(size_t size, ngx_log_t *log);
```

创建一个初始节点大小为size的pool，log为后续在该pool上进行操作时输出日志的对象。 需要说明的是size的选择，size的大小必须小于等于NGX\_MAX\_ALLOC\_FROM\_POOL，且必须大于sizeof\(ngx\_pool\_t\)。

选择大于NGX\_MAX\_ALLOC\_FROM\_POOL的值会造成浪费，因为大于该限制的空间不会被用到（只是说在第一个由ngx\_pool\_t对象管理的内存块上的内存，后续的分配如果第一个内存块上的空闲部分已用完，会再分配的）。

选择小于sizeof\(ngx\_pool\_t\)的值会造成程序崩溃。由于初始大小的内存块中要用一部分来存储ngx\_pool\_t这个信息本身。

当一个ngx\_pool\_t对象被创建以后，该对象的max字段被赋值为size-sizeof\(ngx\_pool\_t\)和NGX\_MAX\_ALLOC\_FROM\_POOL这两者中比较小的。后续的从这个pool中分配的内存块，在第一块内存使用完成以后，如果要继续分配的话，就需要继续从操作系统申请内存。当内存的大小小于等于max字段的时候，则分配新的内存块，链接在d这个字段（实际上是d.next字段）管理的一条链表上。当要分配的内存块是比max大的，那么从系统中申请的内存是被挂接在large字段管理的一条链表上。我们暂且把这个称之为大块内存链和小块内存链。

```text
void *ngx_palloc(ngx_pool_t *pool, size_t size);
```

从这个pool中分配一块为size大小的内存。注意，此函数分配的内存的起始地址按照NGX\_ALIGNMENT进行了对齐。对齐操作会提高系统处理的速度，但会造成少量内存的浪费。

```text
void *ngx_pnalloc(ngx_pool_t *pool, size_t size);
```

从这个pool中分配一块为size大小的内存。但是此函数分配的内存并没有像上面的函数那样进行过对齐。

```text
void *ngx_pcalloc(ngx_pool_t *pool, size_t size);
```

该函数也是分配size大小的内存，并且对分配的内存块进行了清零。内部实际上是转调用ngx\_palloc实现的。

```text
void *ngx_pmemalign(ngx_pool_t *pool, size_t size, size_t alignment);
```

按照指定对齐大小alignment来申请一块大小为size的内存。此处获取的内存不管大小都将被置于大内存块链中管理。

```text
ngx_int_t ngx_pfree(ngx_pool_t *pool, void *p);
```

对于被置于大块内存链，也就是被large字段管理的一列内存中的某块进行释放。该函数的实现是顺序遍历large管理的大块内存链表。所以效率比较低下。如果在这个链表中找到了这块内存，则释放，并返回NGX\_OK。否则返回NGX\_DECLINED。

由于这个操作效率比较低下，除非必要，也就是说这块内存非常大，确应及时释放，否则一般不需要调用。反正内存在这个pool被销毁的时候，总归会都释放掉的嘛！

```text
ngx_pool_cleanup_t *ngx_pool_cleanup_add(ngx_pool_t *p, size_t size);
```

ngx\_pool\_t中的cleanup字段管理着一个特殊的链表，该链表的每一项都记录着一个特殊的需要释放的资源。对于这个链表中每个节点所包含的资源如何去释放，是自说明的。这也就提供了非常大的灵活性。意味着，ngx\_pool\_t不仅仅可以管理内存，通过这个机制，也可以管理任何需要释放的资源，例如，关闭文件，或者删除文件等等。下面我们看一下这个链表每个节点的类型:

```text
typedef struct ngx_pool_cleanup_s  ngx_pool_cleanup_t;
typedef void (*ngx_pool_cleanup_pt)(void *data);

struct ngx_pool_cleanup_s {
    ngx_pool_cleanup_pt   handler;
    void                 *data;
    ngx_pool_cleanup_t   *next;
};
```

| data: | 指明了该节点所对应的资源。 |
| --- | --- | --- |
| handler: | 是一个函数指针，指向一个可以释放data所对应资源的函数。该函数只有一个参数，就是data。 |
| next: | 指向该链表中下一个元素。 |

看到这里，ngx\_pool\_cleanup\_add这个函数的用法，我相信大家都应该有一些明白了。但是这个参数size是起什么作用的呢？这个 size就是要存储这个data字段所指向的资源的大小，该函数会为data分配size大小的空间。

比如我们需要最后删除一个文件。那我们在调用这个函数的时候，把size指定为存储文件名的字符串的大小，然后调用这个函数给cleanup链表中增加一项。该函数会返回新添加的这个节点。我们然后把这个节点中的data字段拷贝为文件名。把hander字段赋值为一个删除文件的函数（当然该函数的原型要按照void \(\*ngx\_pool\_cleanup\_pt\)\(void \*data\)）。

```text
void ngx_destroy_pool(ngx_pool_t *pool);
```

该函数就是释放pool中持有的所有内存，以及依次调用cleanup字段所管理的链表中每个元素的handler字段所指向的函数，来释放掉所有该pool管理的资源。并且把pool指向的ngx\_pool\_t也释放掉了，完全不可用了。

```text
void ngx_reset_pool(ngx_pool_t *pool);
```

该函数释放pool中所有大块内存链表上的内存，小块内存链上的内存块都修改为可用。但是不会去处理cleanup链表上的项目。

#### ngx\_array\_t\(100%\)

ngx\_array\_t是nginx内部使用的数组结构。nginx的数组结构在存储上与大家认知的C语言内置的数组有相似性，比如实际上存储数据的区域也是一大块连续的内存。但是数组除了存储数据的内存以外还包含一些元信息来描述相关的一些信息。下面我们从数组的定义上来详细的了解一下。ngx\_array\_t的定义位于src/core/ngx\_array.c\|h里面。

```text
typedef struct ngx_array_s       ngx_array_t;
struct ngx_array_s {
    void        *elts;
    ngx_uint_t   nelts;
    size_t       size;
    ngx_uint_t   nalloc;
    ngx_pool_t  *pool;
};
```

| elts: | 指向实际的数据存储区域。 |
| --- | --- | --- | --- | --- |
| nelts: | 数组实际元素个数。 |
| size: | 数组单个元素的大小，单位是字节。 |
| nalloc: | 数组的容量。表示该数组在不引发扩容的前提下，可以最多存储的元素的个数。当nelts增长到达nalloc 时，如果再往此数组中存储元素，则会引发数组的扩容。数组的容量将会扩展到原有容量的2倍大小。实际上是分配新的一块内存，新的一块内存的大小是原有内存大小的2倍。原有的数据会被拷贝到新的一块内存中。 |
| pool: | 该数组用来分配内存的内存池。 |

下面介绍ngx\_array\_t相关操作函数。

```text
ngx_array_t *ngx_array_create(ngx_pool_t *p, ngx_uint_t n, size_t size);
```

创建一个新的数组对象，并返回这个对象。

| p: | 数组分配内存使用的内存池； |
| --- | --- | --- |
| n: | 数组的初始容量大小，即在不扩容的情况下最多可以容纳的元素个数。 |
| size: | 单个元素的大小，单位是字节。 |

```text
void ngx_array_destroy(ngx_array_t *a);
```

销毁该数组对象，并释放其分配的内存回内存池。

```text
void *ngx_array_push(ngx_array_t *a);
```

在数组a上新追加一个元素，并返回指向新元素的指针。需要把返回的指针使用类型转换，转换为具体的类型，然后再给新元素本身或者是各字段（如果数组的元素是复杂类型）赋值。

```text
void *ngx_array_push_n(ngx_array_t *a, ngx_uint_t n);
```

在数组a上追加n个元素，并返回指向这些追加元素的首个元素的位置的指针。

```text
static ngx_inline ngx_int_t ngx_array_init(ngx_array_t *array, ngx_pool_t *pool, ngx_uint_t n, size_t size);
```

如果一个数组对象是被分配在堆上的，那么当调用ngx\_array\_destroy销毁以后，如果想再次使用，就可以调用此函数。

如果一个数组对象是被分配在栈上的，那么就需要调用此函数，进行初始化的工作以后，才可以使用。

**注意事项:** 由于使用ngx\_palloc分配内存，数组在扩容时，旧的内存不会被释放，会造成内存的浪费。因此，最好能提前规划好数组的容量，在创建或者初始化的时候一次搞定，避免多次扩容，造成内存浪费。

#### ngx\_hash\_t\(100%\)

ngx\_hash\_t是nginx自己的hash表的实现。定义和实现位于src/core/ngx\_hash.h\|c中。ngx\_hash\_t的实现也与数据结构教科书上所描述的hash表的实现是大同小异。对于常用的解决冲突的方法有线性探测，二次探测和开链法等。ngx\_hash\_t使用的是最常用的一种，也就是开链法，这也是STL中的hash表使用的方法。

但是ngx\_hash\_t的实现又有其几个显著的特点:

1. ngx\_hash\_t不像其他的hash表的实现，可以插入删除元素，它只能一次初始化，就构建起整个hash表以后，既不能再删除，也不能在插入元素了。
2. ngx\_hash\_t的开链并不是真的开了一个链表，实际上是开了一段连续的存储空间，几乎可以看做是一个数组。这是因为ngx\_hash\_t在初始化的时候，会经历一次预计算的过程，提前把每个桶里面会有多少元素放进去给计算出来，这样就提前知道每个桶的大小了。那么就不需要使用链表，一段连续的存储空间就足够了。这也从一定程度上节省了内存的使用。

从上面的描述，我们可以看出来，这个值越大，越造成内存的浪费。就两步，首先是初始化，然后就可以在里面进行查找了。下面我们详细来看一下。

ngx\_hash\_t的初始化。

```text
   ngx_int_t ngx_hash_init(ngx_hash_init_t *hinit, ngx_hash_key_t *names,
ngx_uint_t nelts);
```

首先我们来看一下初始化函数。该函数的第一个参数hinit是初始化的一些参数的一个集合。 names是初始化一个ngx\_hash\_t所需要的所有key的一个数组。而nelts就是key的个数。下面先看一下ngx\_hash\_init\_t类型，该类型提供了初始化一个hash表所需要的一些基本信息。

```text
typedef struct {
    ngx_hash_t       *hash;
    ngx_hash_key_pt   key;

    ngx_uint_t        max_size;
    ngx_uint_t        bucket_size;

    char             *name;
    ngx_pool_t       *pool;
    ngx_pool_t       *temp_pool;
} ngx_hash_init_t;
```

| hash: | 该字段如果为NULL，那么调用完初始化函数后，该字段指向新创建出来的hash表。如果该字段不为NULL，那么在初始的时候，所有的数据被插入了这个字段所指的hash表中。 |
| --- | --- | --- | --- | --- | --- | --- |
| key: | 指向从字符串生成hash值的hash函数。nginx的源代码中提供了默认的实现函数ngx\_hash\_key\_lc。 |
| max\_size: | hash表中的桶的个数。该字段越大，元素存储时冲突的可能性越小，每个桶中存储的元素会更少，则查询起来的速度更快。当然，这个值越大，越造成内存的浪费也越大，\(实际上也浪费不了多少\)。 |
| bucket\_size: | 每个桶的最大限制大小，单位是字节。如果在初始化一个hash表的时候，发现某个桶里面无法存的下所有属于该桶的元素，则hash表初始化失败。 |
| name: | 该hash表的名字。 |
| pool: | 该hash表分配内存使用的pool。 |
| temp\_pool: | 该hash表使用的临时pool，在初始化完成以后，该pool可以被释放和销毁掉。 |

下面来看一下存储hash表key的数组的结构。

```text
typedef struct {
    ngx_str_t         key;
    ngx_uint_t        key_hash;
    void             *value;
} ngx_hash_key_t;
```

key和value的含义显而易见，就不用解释了。key\_hash是对key使用hash函数计算出来的值。 对这两个结构分析完成以后，我想大家应该都已经明白这个函数应该是如何使用了吧。该函数成功初始化一个hash表以后，返回NGX\_OK，否则返回NGX\_ERROR。

```text
void *ngx_hash_find(ngx_hash_t *hash, ngx_uint_t key, u_char *name, size_t len);
```

在hash里面查找key对应的value。实际上这里的key是对真正的key（也就是name）计算出的hash值。len是name的长度。

如果查找成功，则返回指向value的指针，否则返回NULL。

#### ngx\_hash\_wildcard\_t\(100%\)

nginx为了处理带有通配符的域名的匹配问题，实现了ngx\_hash\_wildcard\_t这样的hash表。他可以支持两种类型的带有通配符的域名。一种是通配符在前的，例如：“\*.abc.com”，也可以省略掉星号，直接写成”.abc.com”。这样的key，可以匹配www.abc.com，qqq.www.abc.com之类的。另外一种是通配符在末尾的，例如：“mail.xxx.\*”，请特别注意通配符在末尾的不像位于开始的通配符可以被省略掉。这样的通配符，可以匹配mail.xxx.com、mail.xxx.com.cn、mail.xxx.net之类的域名。

有一点必须说明，就是一个ngx\_hash\_wildcard\_t类型的hash表只能包含通配符在前的key或者是通配符在后的key。不能同时包含两种类型的通配符的key。ngx\_hash\_wildcard\_t类型变量的构建是通过函数ngx\_hash\_wildcard\_init完成的，而查询是通过函数ngx\_hash\_find\_wc\_head或者ngx\_hash\_find\_wc\_tail来做的。ngx\_hash\_find\_wc\_head是查询包含通配符在前的key的hash表的，而ngx\_hash\_find\_wc\_tail是查询包含通配符在后的key的hash表的。

下面详细说明这几个函数的用法。

```text
ngx_int_t ngx_hash_wildcard_init(ngx_hash_init_t *hinit, ngx_hash_key_t *names,
    ngx_uint_t nelts);
```

该函数迎来构建一个可以包含通配符key的hash表。

| hinit: | 构造一个通配符hash表的一些参数的一个集合。关于该参数对应的类型的说明，请参见ngx\_hash\_t类型中ngx\_hash\_init函数的说明。 |
| --- | --- | --- |
| names: | 构造此hash表的所有的通配符key的数组。特别要注意的是这里的key已经都是被预处理过的。例如：“\*.abc.com”或者“.abc.com”被预处理完成以后，变成了“com.abc.”。而“mail.xxx.\*”则被预处理为“mail.xxx.”。为什么会被处理这样？这里不得不简单地描述一下通配符hash表的实现原理。当构造此类型的hash表的时候，实际上是构造了一个hash表的一个“链表”，是通过hash表中的key“链接”起来的。比如：对于“\*.abc.com”将会构造出2个hash表，第一个hash表中有一个key为com的表项，该表项的value包含有指向第二个hash表的指针，而第二个hash表中有一个表项abc，该表项的value包含有指向\*.abc.com对应的value的指针。那么查询的时候，比如查询www.abc.com的时候，先查com，通过查com可以找到第二级的hash表，在第二级hash表中，再查找abc，依次类推，直到在某一级的hash表中查到的表项对应的value对应一个真正的值而非一个指向下一级hash表的指针的时候，查询过程结束。**这里有一点需要特别注意的，就是names数组中元素的value值低两位bit必须为0（有特殊用途）。如果不满足这个条件，这个hash表查询不出正确结果。** |
| nelts: | names数组元素的个数。 |

该函数执行成功返回NGX\_OK，否则NGX\_ERROR。

```text
void *ngx_hash_find_wc_head(ngx_hash_wildcard_t *hwc, u_char *name, size_t len);
```

该函数查询包含通配符在前的key的hash表的。

| hwc: | hash表对象的指针。 |
| --- | --- | --- |
| name: | 需要查询的域名，例如: www.abc.com。 |
| len: | name的长度。 |

该函数返回匹配的通配符对应value。如果没有查到，返回NULL。

```text
void *ngx_hash_find_wc_tail(ngx_hash_wildcard_t *hwc, u_char *name, size_t len);
```

该函数查询包含通配符在末尾的key的hash表的。 参数及返回值请参加上个函数的说明。

#### ngx\_hash\_combined\_t\(100%\)

组合类型hash表，该hash表的定义如下：

```text
typedef struct {
    ngx_hash_t            hash;
    ngx_hash_wildcard_t  *wc_head;
    ngx_hash_wildcard_t  *wc_tail;
} ngx_hash_combined_t;
```

从其定义显见，该类型实际上包含了三个hash表，一个普通hash表，一个包含前向通配符的hash表和一个包含后向通配符的hash表。

nginx提供该类型的作用，在于提供一个方便的容器包含三个类型的hash表，当有包含通配符的和不包含通配符的一组key构建hash表以后，以一种方便的方式来查询，你不需要再考虑一个key到底是应该到哪个类型的hash表里去查了。

构造这样一组合hash表的时候，首先定义一个该类型的变量，再分别构造其包含的三个子hash表即可。

对于该类型hash表的查询，nginx提供了一个方便的函数ngx\_hash\_find\_combined。

```text
void *ngx_hash_find_combined(ngx_hash_combined_t *hash, ngx_uint_t key,
u_char *name, size_t len);
```

该函数在此组合hash表中，依次查询其三个子hash表，看是否匹配，一旦找到，立即返回查找结果，也就是说如果有多个可能匹配，则只返回第一个匹配的结果。

| hash: | 此组合hash表对象。 |
| --- | --- | --- | --- |
| key: | 根据name计算出的hash值。 |
| name: | key的具体内容。 |
| len: | name的长度。 |

返回查询的结果，未查到则返回NULL。

#### ngx\_hash\_keys\_arrays\_t\(100%\)

大家看到在构建一个ngx\_hash\_wildcard\_t的时候，需要对通配符的哪些key进行预处理。这个处理起来比较麻烦。而当有一组key，这些里面既有无通配符的key，也有包含通配符的key的时候。我们就需要构建三个hash表，一个包含普通的key的hash表，一个包含前向通配符的hash表，一个包含后向通配符的hash表（或者也可以把这三个hash表组合成一个ngx\_hash\_combined\_t）。在这种情况下，为了让大家方便的构造这些hash表，nginx提供给了此辅助类型。

该类型以及相关的操作函数也定义在src/core/ngx\_hash.h\|c里。我们先来看一下该类型的定义。

```text
typedef struct {
    ngx_uint_t        hsize;

    ngx_pool_t       *pool;
    ngx_pool_t       *temp_pool;

    ngx_array_t       keys;
    ngx_array_t      *keys_hash;

    ngx_array_t       dns_wc_head;
    ngx_array_t      *dns_wc_head_hash;

    ngx_array_t       dns_wc_tail;
    ngx_array_t      *dns_wc_tail_hash;
} ngx_hash_keys_arrays_t;
```

| hsize: | 将要构建的hash表的桶的个数。对于使用这个结构中包含的信息构建的三种类型的hash表都会使用此参数。 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| pool: | 构建这些hash表使用的pool。 |
| temp\_pool: | 在构建这个类型以及最终的三个hash表过程中可能用到临时pool。该temp\_pool可以在构建完成以后，被销毁掉。这里只是存放临时的一些内存消耗。 |
| keys: | 存放所有非通配符key的数组。 |
| keys\_hash: | 这是个二维数组，第一个维度代表的是bucket的编号，那么keys\_hash\[i\]中存放的是所有的key算出来的hash值对hsize取模以后的值为i的key。假设有3个key,分别是key1,key2和key3假设hash值算出来以后对hsize取模的值都是i，那么这三个key的值就顺序存放在keys\_hash\[i\]\[0\],keys\_hash\[i\]\[1\], keys\_hash\[i\]\[2\]。该值在调用的过程中用来保存和检测是否有冲突的key值，也就是是否有重复。 |
| dns\_wc\_head: | 放前向通配符key被处理完成以后的值。比如：“\*.abc.com” 被处理完成以后，变成 “com.abc.” 被存放在此数组中。 |
| dns\_wc\_tail: | 存放后向通配符key被处理完成以后的值。比如：“mail.xxx.\*” 被处理完成以后，变成 “mail.xxx.” 被存放在此数组中。 |
| dns\_wc\_head\_hash: |  |
|   | 该值在调用的过程中用来保存和检测是否有冲突的前向通配符的key值，也就是是否有重复。 |
| dns\_wc\_tail\_hash: |  |
|   | 该值在调用的过程中用来保存和检测是否有冲突的后向通配符的key值，也就是是否有重复。 |

在定义一个这个类型的变量，并对字段pool和temp\_pool赋值以后，就可以调用函数ngx\_hash\_add\_key把所有的key加入到这个结构中了，该函数会自动实现普通key，带前向通配符的key和带后向通配符的key的分类和检查，并将这个些值存放到对应的字段中去， 然后就可以通过检查这个结构体中的keys、dns\_wc\_head、dns\_wc\_tail三个数组是否为空，来决定是否构建普通hash表，前向通配符hash表和后向通配符hash表了（在构建这三个类型的hash表的时候，可以分别使用keys、dns\_wc\_head、dns\_wc\_tail三个数组）。

构建出这三个hash表以后，可以组合在一个ngx\_hash\_combined\_t对象中，使用ngx\_hash\_find\_combined进行查找。或者是仍然保持三个独立的变量对应这三个hash表，自己决定何时以及在哪个hash表中进行查询。

```text
ngx_int_t ngx_hash_keys_array_init(ngx_hash_keys_arrays_t *ha, ngx_uint_t type);
```

初始化这个结构，主要是对这个结构中的ngx\_array\_t类型的字段进行初始化，成功返回NGX\_OK。

| ha: | 该结构的对象指针。 |
| --- | --- |
| type: | 该字段有2个值可选择，即NGX\_HASH\_SMALL和NGX\_HASH\_LARGE。用来指明将要建立的hash表的类型，如果是NGX\_HASH\_SMALL，则有比较小的桶的个数和数组元素大小。NGX\_HASH\_LARGE则相反。 |

```text
ngx_int_t ngx_hash_add_key(ngx_hash_keys_arrays_t *ha, ngx_str_t *key,
void *value, ngx_uint_t flags);
```

一般是循环调用这个函数，把一组键值对加入到这个结构体中。返回NGX\_OK是加入成功。返回NGX\_BUSY意味着key值重复。

| ha: | 该结构的对象指针。 |
| --- | --- | --- | --- |
| key: | 参数名自解释了。 |
| value: | 参数名自解释了。 |
| flags: | 有两个标志位可以设置，NGX\_HASH\_WILDCARD\_KEY和NGX\_HASH\_READONLY\_KEY。同时要设置的使用逻辑与操作符就可以了。NGX\_HASH\_READONLY\_KEY被设置的时候，在计算hash值的时候，key的值不会被转成小写字符，否则会。NGX\_HASH\_WILDCARD\_KEY被设置的时候，说明key里面可能含有通配符，会进行相应的处理。如果两个标志位都不设置，传0。 |

有关于这个数据结构的使用，可以参考src/http/ngx\_http.c中的ngx\_http\_server\_names函数。

#### ngx\_chain\_t\(100%\)

nginx的filter模块在处理从别的filter模块或者是handler模块传递过来的数据（实际上就是需要发送给客户端的http response）。这个传递过来的数据是以一个链表的形式\(ngx\_chain\_t\)。而且数据可能被分多次传递过来。也就是多次调用filter的处理函数，以不同的ngx\_chain\_t。

该结构被定义在src/core/ngx\_buf.h\|c。下面我们来看一下ngx\_chain\_t的定义。

```text
typedef struct ngx_chain_s       ngx_chain_t;

struct ngx_chain_s {
    ngx_buf_t    *buf;
    ngx_chain_t  *next;
};
```

就2个字段，next指向这个链表的下个节点。buf指向实际的数据。所以在这个链表上追加节点也是非常容易，只要把末尾元素的next指针指向新的节点，把新节点的next赋值为NULL即可。

```text
ngx_chain_t *ngx_alloc_chain_link(ngx_pool_t *pool);
```

该函数创建一个ngx\_chain\_t的对象，并返回指向对象的指针，失败返回NULL。

```text
#define ngx_free_chain(pool, cl)                                             \
    cl->next = pool->chain;                                                  \
pool->chain = cl

```

该宏释放一个ngx\_chain\_t类型的对象。如果要释放整个chain，则迭代此链表，对每个节点使用此宏即可。

**注意:** 对ngx\_chaint\_t类型的释放，并不是真的释放了内存，而仅仅是把这个对象挂在了这个pool对象的一个叫做chain的字段对应的chain上，以供下次从这个pool上分配ngx\_chain\_t类型对象的时候，快速的从这个pool-&gt;chain上取下链首元素就返回了，当然，如果这个链是空的，才会真的在这个pool上使用ngx\_palloc函数进行分配。

#### ngx\_buf\_t\(99%\)

这个ngx\_buf\_t就是这个ngx\_chain\_t链表的每个节点的实际数据。该结构实际上是一种抽象的数据结构，它代表某种具体的数据。这个数据可能是指向内存中的某个缓冲区，也可能指向一个文件的某一部分，也可能是一些纯元数据（元数据的作用在于指示这个链表的读取者对读取的数据进行不同的处理）。

该数据结构位于src/core/ngx\_buf.h\|c文件中。我们来看一下它的定义。

```text
struct ngx_buf_s {
    u_char          *pos;
    u_char          *last;
    off_t            file_pos;
    off_t            file_last;

    u_char          *start;         /* start of buffer */
    u_char          *end;           /* end of buffer */
    ngx_buf_tag_t    tag;
    ngx_file_t      *file;
    ngx_buf_t       *shadow;


    /* the buf's content could be changed */
    unsigned         temporary:1;

    /*
     * the buf's content is in a memory cache or in a read only memory
     * and must not be changed
     */
    unsigned         memory:1;

    /* the buf's content is mmap()ed and must not be changed */
    unsigned         mmap:1;

    unsigned         recycled:1;
    unsigned         in_file:1;
    unsigned         flush:1;
    unsigned         sync:1;
    unsigned         last_buf:1;
    unsigned         last_in_chain:1;

    unsigned         last_shadow:1;
    unsigned         temp_file:1;

    /* STUB */ int   num;
};
```

| pos: | 当buf所指向的数据在内存里的时候，pos指向的是这段数据开始的位置。 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| last: | 当buf所指向的数据在内存里的时候，last指向的是这段数据结束的位置。 |
| file\_pos: | 当buf所指向的数据是在文件里的时候，file\_pos指向的是这段数据的开始位置在文件中的偏移量。 |
| file\_last: | 当buf所指向的数据是在文件里的时候，file\_last指向的是这段数据的结束位置在文件中的偏移量。 |
| start: | 当buf所指向的数据在内存里的时候，这一整块内存包含的内容可能被包含在多个buf中\(比如在某段数据中间插入了其他的数据，这一块数据就需要被拆分开\)。那么这些buf中的start和end都指向这一块内存的开始地址和结束地址。而pos和last指向本buf所实际包含的数据的开始和结尾。 |
| end: | 解释参见start。 |
| tag: | 实际上是一个void\*类型的指针，使用者可以关联任意的对象上去，只要对使用者有意义。 |
| file: | 当buf所包含的内容在文件中时，file字段指向对应的文件对象。 |
| shadow: | 当这个buf完整copy了另外一个buf的所有字段的时候，那么这两个buf指向的实际上是同一块内存，或者是同一个文件的同一部分，此时这两个buf的shadow字段都是指向对方的。那么对于这样的两个buf，在释放的时候，就需要使用者特别小心，具体是由哪里释放，要提前考虑好，如果造成资源的多次释放，可能会造成程序崩溃！ |
| temporary: | 为1时表示该buf所包含的内容是在一个用户创建的内存块中，并且可以被在filter处理的过程中进行变更，而不会造成问题。 |
| memory: | 为1时表示该buf所包含的内容是在内存中，但是这些内容却不能被进行处理的filter进行变更。 |
| mmap: | 为1时表示该buf所包含的内容是在内存中, 是通过mmap使用内存映射从文件中映射到内存中的，这些内容却不能被进行处理的filter进行变更。 |
| recycled: | 可以回收的。也就是这个buf是可以被释放的。这个字段通常是配合shadow字段一起使用的，对于使用ngx\_create\_temp\_buf 函数创建的buf，并且是另外一个buf的shadow，那么可以使用这个字段来标示这个buf是可以被释放的。 |
| in\_file: | 为1时表示该buf所包含的内容是在文件中。 |
| flush: | 遇到有flush字段被设置为1的的buf的chain，则该chain的数据即便不是最后结束的数据（last\_buf被设置，标志所有要输出的内容都完了），也会进行输出，不会受postpone\_output配置的限制，但是会受到发送速率等其他条件的限制。 |
| sync: |  |
| last\_buf: | 数据被以多个chain传递给了过滤器，此字段为1表明这是最后一个buf。 |
| last\_in\_chain: | 在当前的chain里面，此buf是最后一个。特别要注意的是last\_in\_chain的buf不一定是last\_buf，但是last\_buf的buf一定是last\_in\_chain的。这是因为数据会被以多个chain传递给某个filter模块。 |
| last\_shadow: | 在创建一个buf的shadow的时候，通常将新创建的一个buf的last\_shadow置为1。 |
| temp\_file: | 由于受到内存使用的限制，有时候一些buf的内容需要被写到磁盘上的临时文件中去，那么这时，就设置此标志 。 |

对于此对象的创建，可以直接在某个ngx\_pool\_t上分配，然后根据需要，给对应的字段赋值。也可以使用定义好的2个宏：

```text
#define ngx_alloc_buf(pool)  ngx_palloc(pool, sizeof(ngx_buf_t))
#define ngx_calloc_buf(pool) ngx_pcalloc(pool, sizeof(ngx_buf_t))

```

这两个宏使用类似函数，也是不说自明的。

对于创建temporary字段为1的buf（就是其内容可以被后续的filter模块进行修改），可以直接使用函数ngx\_create\_temp\_buf进行创建。

```text
ngx_buf_t *ngx_create_temp_buf(ngx_pool_t *pool, size_t size);
```

该函数创建一个ngx\_but\_t类型的对象，并返回指向这个对象的指针，创建失败返回NULL。

对于创建的这个对象，它的start和end指向新分配内存开始和结束的地方。pos和last都指向这块新分配内存的开始处，这样，后续的操作可以在这块新分配的内存上存入数据。

| pool: | 分配该buf和buf使用的内存所使用的pool。 |
| --- | --- |
| size: | 该buf使用的内存的大小。 |

为了配合对ngx\_buf\_t的使用，nginx定义了以下的宏方便操作。

```text
#define ngx_buf_in_memory(b)        (b->temporary || b->memory || b->mmap)

```

返回这个buf里面的内容是否在内存里。

```text
#define ngx_buf_in_memory_only(b)   (ngx_buf_in_memory(b) && !b->in_file)

```

返回这个buf里面的内容是否仅仅在内存里，并且没有在文件里。

```text
#define ngx_buf_special(b)                                                   \
    ((b->flush || b->last_buf || b->sync)                                    \
     && !ngx_buf_in_memory(b) && !b->in_file)

```

返回该buf是否是一个特殊的buf，只含有特殊的标志和没有包含真正的数据。

```text
#define ngx_buf_sync_only(b)                                                 \
    (b->sync                                                                 \
     && !ngx_buf_in_memory(b) && !b->in_file && !b->flush && !b->last_buf)

```

返回该buf是否是一个只包含sync标志而不包含真正数据的特殊buf。

```text
#define ngx_buf_size(b)                                                      \
    (ngx_buf_in_memory(b) ? (off_t) (b->last - b->pos):                      \
                            (b->file_last - b->file_pos))

```

返回该buf所含数据的大小，不管这个数据是在文件里还是在内存里。

#### ngx\_list\_t\(100%\)

ngx\_list\_t顾名思义，看起来好像是一个list的数据结构。这样的说法，算对也不算对。因为它符合list类型数据结构的一些特点，比如可以添加元素，实现自增长，不会像数组类型的数据结构，受到初始设定的数组容量的限制，并且它跟我们常见的list型数据结构也是一样的，内部实现使用了一个链表。

那么它跟我们常见的链表实现的list有什么不同呢？不同点就在于它的节点，它的节点不像我们常见的list的节点，只能存放一个元素，ngx\_list\_t的节点实际上是一个固定大小的数组。

在初始化的时候，我们需要设定元素需要占用的空间大小，每个节点数组的容量大小。在添加元素到这个list里面的时候，会在最尾部的节点里的数组上添加元素，如果这个节点的数组存满了，就再增加一个新的节点到这个list里面去。

好了，看到这里，大家应该基本上明白这个list结构了吧？还不明白也没有关系，下面我们来具体看一下它的定义，这些定义和相关的操作函数定义在src/core/ngx\_list.h\|c文件中。

```text
typedef struct {
    ngx_list_part_t  *last;
    ngx_list_part_t   part;
    size_t            size;
    ngx_uint_t        nalloc;
    ngx_pool_t       *pool;
} ngx_list_t;
```

| last: | 指向该链表的最后一个节点。 |
| --- | --- | --- | --- | --- |
| part: | 该链表的首个存放具体元素的节点。 |
| size: | 链表中存放的具体元素所需内存大小。 |
| nalloc: | 每个节点所含的固定大小的数组的容量。 |
| pool: | 该list使用的分配内存的pool。 |

好，我们在看一下每个节点的定义。

```text
typedef struct ngx_list_part_s  ngx_list_part_t;
struct ngx_list_part_s {
    void             *elts;
    ngx_uint_t        nelts;
    ngx_list_part_t  *next;
};
```

| elts: | 节点中存放具体元素的内存的开始地址。 |
| --- | --- | --- |
| nelts: | 节点中已有元素个数。这个值是不能大于链表头节点ngx\_list\_t类型中的nalloc字段的。 |
| next: | 指向下一个节点。 |

我们来看一下提供的一个操作的函数。

```text
ngx_list_t *ngx_list_create(ngx_pool_t *pool, ngx_uint_t n, size_t size);
```

该函数创建一个ngx\_list\_t类型的对象,并对该list的第一个节点分配存放元素的内存空间。

| pool: | 分配内存使用的pool。 |
| --- | --- | --- | --- |
| n: | 每个节点固定长度的数组的长度。 |
| size: | 存放的具体元素的个数。 |
| 返回值: | 成功返回指向创建的ngx\_list\_t对象的指针，失败返回NULL。 |

```text
void *ngx_list_push(ngx_list_t *list);
```

该函数在给定的list的尾部追加一个元素，并返回指向新元素存放空间的指针。如果追加失败，则返回NULL。

```text
static ngx_inline ngx_int_t
ngx_list_init(ngx_list_t *list, ngx_pool_t *pool, ngx_uint_t n, size_t size);
```

该函数是用于ngx\_list\_t类型的对象已经存在，但是其第一个节点存放元素的内存空间还未分配的情况下，可以调用此函数来给这个list的首节点来分配存放元素的内存空间。

那么什么时候会出现已经有了ngx\_list\_t类型的对象，而其首节点存放元素的内存尚未分配的情况呢？那就是这个ngx\_list\_t类型的变量并不是通过调用ngx\_list\_create函数创建的。例如：如果某个结构体的一个成员变量是ngx\_list\_t类型的，那么当这个结构体类型的对象被创建出来的时候，这个成员变量也被创建出来了，但是它的首节点的存放元素的内存并未被分配。

总之，如果这个ngx\_list\_t类型的变量，如果不是你通过调用函数ngx\_list\_create创建的，那么就必须调用此函数去初始化，否则，你往这个list里追加元素就可能引发不可预知的行为，亦或程序会崩溃!

#### ngx\_queue\_t\(100%\)

ngx\_queue\_t是nginx中的双向链表，在nginx源码目录src/core下面的ngx\_queue.h\|c里面。它的原型如下：

```text
typedef struct ngx_queue_s ngx_queue_t;

struct ngx_queue_s {
    ngx_queue_t  *prev;
    ngx_queue_t  *next;
};
```

不同于教科书中将链表节点的数据成员声明在链表节点的结构体中，ngx\_queue\_t只是声明了前向和后向指针。在使用的时候，我们首先需要定义一个哨兵节点\(对于后续具体存放数据的节点，我们称之为数据节点\)，比如：

```text
ngx_queue_t free;
```

接下来需要进行初始化，通过宏ngx\_queue\_init\(\)来实现：

```text
ngx_queue_init(&free);
```

ngx\_queue\_init\(\)的宏定义如下：

```text
#define ngx_queue_init(q)     \
    (q)->prev = q;            \
    (q)->next = q;

```

可见初始的时候哨兵节点的 prev 和 next 都指向自己，因此其实是一个空链表。ngx\_queue\_empty\(\)可以用来判断一个链表是否为空，其实现也很简单，就是：

```text
#define ngx_queue_empty(h)    \
    (h == (h)->prev)

```

那么如何声明一个具有数据元素的链表节点呢？只要在相应的结构体中加上一个 ngx\_queue\_t 的成员就行了。比如ngx\_http\_upstream\_keepalive\_module中的ngx\_http\_upstream\_keepalive\_cache\_t：

```text
typedef struct {
    ngx_http_upstream_keepalive_srv_conf_t  *conf;

    ngx_queue_t                        queue;
    ngx_connection_t                  *connection;

    socklen_t                          socklen;
    u_char                             sockaddr[NGX_SOCKADDRLEN];
} ngx_http_upstream_keepalive_cache_t;
```

对于每一个这样的数据节点，可以通过ngx\_queue\_insert\_head\(\)来添加到链表中，第一个参数是哨兵节点，第二个参数是数据节点，比如：

```text
ngx_http_upstream_keepalive_cache_t cache;
ngx_queue_insert_head(&free, &cache.queue);
```

相应的几个宏定义如下：

```text
#define ngx_queue_insert_head(h, x)                         \
    (x)->next = (h)->next;                                  \
    (x)->next->prev = x;                                    \
    (x)->prev = h;                                          \
    (h)->next = x

#define ngx_queue_insert_after   ngx_queue_insert_head

#define ngx_queue_insert_tail(h, x)                          \
    (x)->prev = (h)->prev;                                   \
    (x)->prev->next = x;                                     \
    (x)->next = h;                                           \
    (h)->prev = x

```

ngx\_queue\_insert\_head\(\)和ngx\_queue\_insert\_after\(\)都是往头部添加节点，ngx\_queue\_insert\_tail\(\)是往尾部添加节点。从代码可以看出哨兵节点的 prev 指向链表的尾数据节点，next 指向链表的头数据节点。另外ngx\_queue\_head\(\)和ngx\_queue\_last\(\)这两个宏分别可以得到头节点和尾节点。

那假如现在有一个ngx\_queue\_t [\*](http://tengine.taobao.org/book/chapter_02.html#id4)q 指向的是链表中的数据节点的queue成员，如何得到ngx\_http\_upstream\_keepalive\_cache\_t的数据呢？ nginx提供了ngx\_queue\_data\(\)宏来得到ngx\_http\_upstream\_keepalive\_cache\_t的指针，例如：

```text
ngx_http_upstream_keepalive_cache_t *cache = ngx_queue_data(q,
                                                 ngx_http_upstream_keepalive_cache_t,
                                                 queue);
```

也许您已经可以猜到ngx\_queue\_data是通过地址相减来得到的：

```text
#define ngx_queue_data(q, type, link)                        \
    (type *) ((u_char *) q - offsetof(type, link))

```

另外nginx也提供了ngx\_queue\_remove\(\)宏来从链表中删除一个数据节点，以及ngx\_queue\_add\(\)用来将一个链表添加到另一个链表。

