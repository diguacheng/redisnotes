# redis 第二篇 原理篇

[toc]

## 2. 1线程io模型

redis是个单线程程序（在6.0版本引入了多线程，需要手动开启）

redis运行快的原因是所有的数据都在内存中，所有的运算都是内存级别的运算。

单线程的redis能处理大量的并发客户端的原因是 多路复用，

### 2.1.1 阻塞i/o与非阻塞I/o

- 阻塞i/o

编程中调用套接字方法，往往是阻塞的，比如使用read方法去读，但是此时没有数据，线程就会阻塞，直到新的数据到来或者连接关闭，read方法才返回。

![阻塞io流程](C:\Users\46782\Desktop\redis\redisnotes\redistheory.assets\阻塞io流程.png)

在需要处理多个客户端任务时，往往不使用阻塞I/o

- 非阻塞i/o

非阻塞i/o的读写方法不会阻塞，而是能读多少读多少，能写多少写多少，取决于内核为套接字分配的空间。

但是非阻塞i/o有个问题，当一个线程读了一部分数据返回后，线程不知道何时继续读

或者当线程写完一部分数据，缓冲区满了，剩下的数据线程不知道何时去写。

即 线程需要得到通知。----事件轮询api可以解决这个问题

### 2.1.3事件轮询（多路复用）

![多路复用io](C:\Users\46782\Desktop\redis\redisnotes\redistheory.assets\多路复用io.png)

select函数是操作系统提供给用户程序的api,

输入读写描述符列表read_fds write_fds,和timeout参数，

输出可读可写事件。

若没有任何事件到来，最多等该timeout时间，线程处于阻塞状态。

redis拿到时间后，会依次处理完相应事件，处理完继续轮询，于是线程就进入了死循环，这个死循环就是事件循环。

#### redis io多路复用程序的实现

![digraph {    label = "图 IMAGE_MULTI_LIB    Redis 的 I/O 多路复用程序有多个 I/O 多路复用库实现可选";    node [shape = box];    io_multiplexing [label = "I/O 多路复用程序"];    subgraph cluster_imp {        style = dashed        label = "底层实现";        labelloc = "b";        kqueue [label = "kqueue"];        evport [label = "evport"];        epoll [label = "epoll"];        select [label = "select"];    }    //    edge [dir = back];    io_multiplexing -> select;    io_multiplexing -> epoll;    io_multiplexing -> evport;    io_multiplexing -> kqueue;}](C:\Users\46782\Desktop\redis\redisnotes\redistheory.assets\graphviz-840bfb6ea3cc590829fecd9b9062002d59dbf673.png)

Redis的I/O多路复用程序的所有功能是通过包装select、epoll、evport和kqueue这些I/O多路复用函数库来实现的，每个I/O多路复用函数库在Redis源码中都对应一个单独的文件，比如ae_select.c、ae_epoll.c、ae_kqueue.c等。

因为Redis为每个I/O多路复用函数库都实现了相同的API，所以I/O多路复用程序的底层实现是可以互换的

Redis在I/O多路复用程序的实现源码中用#include宏定义了相应的规则，程序会在编译时自动选择系统中性能最好的I/O多路复用函数库来作为Redis的I/O多路复用程序的底层实现：

```c++
/* Include the best multiplexing layer supported by this system.
 * The following should be ordered by performances, descending. */
#ifdef HAVE_EVPORT
#include "ae_evport.c"
#else
    #ifdef HAVE_EPOLL
    #include "ae_epoll.c"
    #else
        #ifdef HAVE_KQUEUE
        #include "ae_kqueue.c"
        #else
        #include "ae_select.c"
        #endif
    #endif
#endif

```

Redis 会优先选择时间复杂度为 O(1) 的 I/O 多路复用函数作为底层实现，包括 Solaries 10 中的 `evport`、Linux 中的 `epoll` 和 macOS/FreeBSD 中的 `kqueue`，上述的这些函数都使用了内核内部的结构，并且能够服务几十万的文件描述符。

但是如果当前编译环境没有上述函数，就会选择 `select` 作为备选方案，由于其在使用时会扫描全部监听的描述符，所以其时间复杂度较差 O(n)，并且只能同时服务 1024 个文件描述符，所以一般并不会以 `select` 作为第一方案使用。select作为保底供选择



### 2.1.4 指令队列

redis将客户端套接字都关联到一个指令队列里，客户端的指令通过队列来排队进行顺序处理，先到先服务。

### 2.1.5 响应队列

redis为每个客户端套接字关联一个响应队列。redis服务器通过响应队列将指令的返回结果恢复给客户端，

若队列为空，意味着连着暂时处于空闲状态，不需要获取写事件，即可将当前客户端描述符从write_fds里移出来，等有了数据，再放进去。目的是防止select系统调用立即返回写事件，但是缺没有数据可写，造成cpu消耗提升。

### 2.1.6 定时任务 

线程阻塞在select系统调用上，定时任务将无法得到准时调度。

redis通过将定时任务记录在一个称为”最小堆“的数据结构中 解决这个问题。

在堆中，最快要执行的任务排在堆的最上方。在每个循环周期里，redis会对最小堆里面已经到达时间点的任务进行处理。

处理完毕后，将最快要执行任务还需要的事件记录下来，作为select系统调用的timeout参数。



### 2.1.7 redis文件处理器 

![redis文件处理器](C:\Users\46782\Desktop\redis\redisnotes\redistheory.assets\redis文件处理器.png)

Redis基于Reactor模式开发了网络事件处理器，这个处理器被称为文件事件处理器。它的组成结构为4部分：

- 多个套接字、
- I/O多路复用程序、
- 文件事件分派器。因为文件事件分派器队列的消费是单线程的，所以Redis才叫单线程模型。
- 事件处理器。

尽管多个文件事件可能会并发地出现，但I/O多路复用程序总是会将所有产生事件的套接字都推到一个队列里面，然后通过这个队列，以有序（sequentially）、同步（synchronously）、每次一个套接字的方式向文件事件分派器传送套接字：当上一个套接字产生的事件被处理完毕之后（该套接字为事件所关联的事件处理器执行完毕）， I/O多路复用程序才会继续向文件事件分派器传送下一个套接字

#### 文件事件的类型

I/O 多路复用程序可以监听多个套接字的ae.h/AE_READABLE事件和ae.h/AE_WRITABLE事件，这两类事件和套接字操作之间的对应关系如下：

- 当套接字变得可读时（客户端对套接字执行write操作，或者执行close操作），或者有新的可应答（acceptable）套接字出现时（客户端对服务器的监听套接字执行connect操作），套接字产生AE_READABLE 事件。
- 当套接字变得可写时（客户端对套接字执行read操作），套接字产生AE_WRITABLE事件。I/O多路复用程序允许服务器同时监听套接字的AE_READABLE事件和AE_WRITABLE事件，如果一个套接字同时产生了这两种事件，那么文件事件分派器会优先处理AE_READABLE事件，等到AE_READABLE事件处理完之后，才处理AE_WRITABLE 事件。这也就是说，如果一个套接字又可读又可写的话，那么服务器将先读套接字，后写套接字。

#### 文件事件的处理器

Redis为文件事件编写了多个处理器，这些事件处理器分别用于实现不同的网络通讯需求，常用的处理器如下：

- 为了对连接服务器的各个客户端进行应答， 服务器要为监听套接字关联连接应答处理器。
- 为了接收客户端传来的命令请求， 服务器要为客户端套接字关联命令请求处理器。
- 为了向客户端返回命令的执行结果， 服务器要为客户端套接字关联命令回复处理器。

##### 连接应答处理器

networking.c中acceptTcpHandler函数是Redis的连接应答处理器，这个处理器用于对连接服务器监听套接字的客户端进行应答，具体实现为sys/socket.h/accept函数的包装。

当Redis服务器进行初始化的时候，程序会将这个连接应答处理器和服务器监听套接字的AE_READABLE事件关联起来，当有客户端用sys/socket.h/connect函数连接服务器监听套接字的时候， 套接字就会产生AE_READABLE 事件， 引发连接应答处理器执行， 并执行相应的套接字应答操作，如图所示。

![img](C:\Users\46782\Desktop\redis\redisnotes\redistheory.assets\16de751520935c90)



##### 命令请求处理器

networking.c中readQueryFromClient函数是Redis的命令请求处理器，这个处理器负责从套接字中读入客户端发送的命令请求内容， 具体实现为unistd.h/read函数的包装。

当一个客户端通过连接应答处理器成功连接到服务器之后， 服务器会将客户端套接字的AE_READABLE事件和命令请求处理器关联起来，当客户端向服务器发送命令请求的时候，套接字就会产生 AE_READABLE事件，引发命令请求处理器执行，并执行相应的套接字读入操作，如图所示。

![img](C:\Users\46782\Desktop\redis\redisnotes\redistheory.assets\16de7515208f8919)

在客户端连接服务器的整个过程中，服务器都会一直为客户端套接字的AE_READABLE事件关联命令请求处理器。

##### 命令回复处理器

networking.c中sendReplyToClient函数是Redis的命令回复处理器，这个处理器负责将服务器执行命令后得到的命令回复通过套接字返回给客户端，具体实现为unistd.h/write函数的包装。

当服务器有命令回复需要传送给客户端的时候，服务器会将客户端套接字的AE_WRITABLE事件和命令回复处理器关联起来，当客户端准备好接收服务器传回的命令回复时，就会产生AE_WRITABLE事件，引发命令回复处理器执行，并执行相应的套接字写入操作， 如图所示。

![img](C:\Users\46782\Desktop\redis\redisnotes\redistheory.assets\16de751520b9b2a1)

当命令回复发送完毕之后， 服务器就会解除命令回复处理器与客户端套接字的 AE_WRITABLE 事件之间的关联。



### 一次完整的客户端与服务器连接事件示例

假设Redis服务器正在运作，那么这个服务器的监听套接字的AE_READABLE事件应该正处于监听状态之下，而该事件所对应的处理器为连接应答处理器。

如果这时有一个Redis客户端向Redis服务器发起连接，那么监听套接字将产生AE_READABLE事件， 触发连接应答处理器执行：处理器会对客户端的连接请求进行应答， 然后创建客户端套接字，以及客户端状态，并将客户端套接字的 AE_READABLE 事件与命令请求处理器进行关联，使得客户端可以向主服务器发送命令请求。

之后，客户端向Redis服务器发送一个命令请求，那么客户端套接字将产生 AE_READABLE事件，引发命令请求处理器执行，处理器读取客户端的命令内容， 然后传给相关程序去执行。

执行命令将产生相应的命令回复，为了将这些命令回复传送回客户端，服务器会将客户端套接字的AE_WRITABLE事件与命令回复处理器进行关联：当客户端尝试读取命令回复的时候，客户端套接字将产生AE_WRITABLE事件， 触发命令回复处理器执行， 当命令回复处理器将命令回复全部写入到套接字之后， 服务器就会解除客户端套接字的AE_WRITABLE事件与命令回复处理器之间的关联。

![img](.\redistheory.assets\16de75153f46bf71)


参考： 

1 链接：https://juejin.im/post/6844903970511519758

2 链接：https://draveness.me/redis-io-multiplexing/



## 2.2 通信协议

### 2.2.1 RESP (Redis Serialization Protocol)

RESP 是 Redis 序列化协议的简写。它是一种直观的文本协议，优势在于实现异常简 单，解析性能极好

Redis 协议将传输的结构数据分为 5 种最小单元类型，单元结束时统一加上回车换行符 号\r\n。 

- 单行字符串 以 + 符号开头。 

eg：**单行字符串** hello world   `+hello world\r\n`

- 多行字符串 以 $ 符号开头，后跟字符串长度。 多行字符串当然也可以表示单行字符串。

eg1：**多行字符串** hello world  `$11\r\nhello world\r\n `

eg2:**NULL**  用多行字符串表示，不过长度要写成-1。 `$-1\r\n `

eg3:**空串**  用多行字符串表示，长度填 0。 `$0\r\n\r\n` 

 注意这里有两个\r\n的原因是两个\r\n 之间,隔的是空串。

- 整数值 以 : 符号开头，后跟整数的字符串形式。 

eg：**整数** 1024  ` :1024\r\n `

- 错误消息 以 - 符号开头。

eg:**错误**  参数类型错误

` -WRONGTYPE Operation against a key holding the wrong kind of value `

- 数组 以 * 号开头，后跟数组的长度。

eg:**数组** [1,2,3]   ` *3\r\n:1\r\n:2\r\n:3\r\n `



 ### 客户端->服务端

客户端向服务器发送的指令只有一种格式，多行字符串数组。比如一个简单的 set 指令 `set author codehole` 会被序列化成下面的字符串。

`*3\r\n$3\r\nset\r\n$6\r\nauthor\r\n$8\r\ncodehole\r\n`

在控制台输出这个字符串如下，可以看出这是很好阅读的一种格式。

```
$3
set
$6
author
$8
codehole
```

### 服务端->客户端

服务器向客户端回复的响应要支持多种数据结构，所以消息响应在结构上要复杂不少。 不过再复杂的响应消息也是以上 5 中基本类型的组合。

##### 单行字符串响应

```
127.0.0.1:6379> set author codehole 
OK
```

这里的 OK 就是单行响应，没有使用引号括起来。 +OK

##### 错误响应

```
127.0.0.1:6379> incr author
(error) ERR value is not an integer or out of range
```

试图对一个字符串进行自增，服务器抛出一个通用的错误。 

-ERR value is not an integer or out of range

##### 整数响应

```
127.0.0.1:6379> incr books
(integer) 1

```

这里的 1 就是整数响应 :1

##### 多行字符串响应

```
127.0.0.1:6379> get author
"codehole"
```

这里使用双引号括起来的字符串就是多行字符串响应

 $8 

codehole

##### 数据响应

```
127.0.0.1:6379> hset info name laoqian
(integer) 1
127.0.0.1:6379> hset info age 30
(integer) 1
127.0.0.1:6379> hset info sex male
(integer) 1
127.0.0.1:6379> hgetall info
1) "name"
2) "laoqian"
3) "age"
4) "30"
5) "sex"
6) "male"

```

这里的 hgetall 命令返回的就是一个数值，第 0|2|4 位置的字符串是 hash 表的 key，第 1|3|5 位置的字符串是 value，客户端负责将数组组装成字典再返回。

```
*6
$4
name
$6
laoqian
$3
age
$2
30
$3
sex
$4
male
```

##### 嵌套

```
127.0.0.1:6379> scan 0
1) "0"
2) 1) "info"
 2) "books"
 3) "author"
```

scan 命令用来扫描服务器包含的所有 key 列表，它是以游标的形式获取，一次只获取 一部分。

 scan 命令返回的是一个嵌套数组。数组的第一个值表示游标的值，如果这个值为零，说 明已经遍历完毕。如果不为零，使用这个值作为 scan 命令的参数进行下一次遍历。数组的 第二个值又是一个数组，这个数组就是 key 列表。

```
*2
$1
0
*3
$4
info
$5
books
$6
autho
```

## 2.3 持久化

