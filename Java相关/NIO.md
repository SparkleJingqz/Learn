

# Channel通道

Channel可以进行读取/写入数据。 Channel中数据总是先读到一个Buffer或者从一个Buffer写入。

- 区别于流String: 全双工双向通信。



## 2.1 Channel类别

FileChannel: 从文件读写数据

DatagreamChannel: 通过UDP读写网络数据

SocketChannel: 通过TCP读写网络数据

ServerSocketChannel: 监听套接口，监听新进TCP连接，对于每个新进连接可创建一个SocketChannel



## 2.2 FileChannel

`position(int)` : 在FileChannel中特定位置进行数据的读/写

- lseek() ? 

`size()` : 返回FileChannel关联文件大小

`truncate(int)` : 截取一个文件，文件将指定长度后面部分删除。

`force() ` 将通道中尚未写入磁盘中数据强制写入磁盘 类似flush()

**`transferTo/From(channel)` 可进行通道之间的数据传输**



写/读代码实例

```java
    public static void main(String[] args) throws Exception {
        //打开FileChannel
        RandomAccessFile aFile = new RandomAccessFile("d:\\IdeaProject\\BasicJava\\test1.txt", "rw");
        final FileChannel channel = aFile.getChannel();

        //创建Buffer对象
        ByteBuffer buffer = ByteBuffer.allocate(1024);

        //读取channel中数据到buffer中，若读到文件末尾则返回-1
        int read = channel.read(buffer);
        while (read != -1) {
            buffer.flip(); //对buffer由写模式切换到读模式读取内容
            while (buffer.hasRemaining()) {
                System.out.print((char)buffer.get());
            }
            buffer.clear(); //buffer中数据读完了，写入时重新指定position
            read = channel.read(buffer);
        }
        

        String msg = "new data";
        //写入数据 clear习惯
        buffer.clear();
        buffer.put(msg.getBytes());

        buffer.flip(); //切换读写模式 - 读buffer以写channel
        while (buffer.hasRemaining()) {
            channel.write(buffer);
        }

        aFile.close();
        System.out.println(channel.isOpen());
    }
```

## 2.3 WebChannel

避免单连接单线程，使得few线程管理多个连接 *via selector选择器*。

网络Channel都继承自AbstractSelectableChannel，可通过Selector选择器来对网络Channel进行就绪选择。



DataGreamChannel与SocketChannel都定义了读写功能，而ServerSocketChannel仅负责监听传入的连接和创建新的SocketChannel，其本身不传输数据。



Socket系Channel可通过 `configureBlocking(boolean)`方法设置阻塞模式



### 2.3.1 ServerSocketChannel

基于通道的socket监听器，调用accept()方法可返回一个SocketChannel()对象，若null则代表无就绪，可非阻塞。



### 2.3.2 SocketChannel

通过Selector选择器可实现多路复用



## 2.4 Scatter/Gather

Scatter分散：从Channel中读取数据，将读取的数据写入多个Buffer中。

- `channel.read(bufferArray)`, 按照bufferArray顺序写入buffer，写满再写下一个

Gather聚集：将多个Buffer中的数据写入同一个Channel。

- `Channel.write(bufferArray)`, 仅写入buffer中position~limit之间的数据。



场景eg: 消息头与消息体分散处理聚集



# Buffer缓冲区

Buffer与Channel交互，将数据写入通道 / 从通道读取到数据



| get()            | put()          | Capacity                                               | Limit     | Position                  | Mark                           |
| ---------------- | -------------- | ------------------------------------------------------ | --------- | ------------------------- | ------------------------------ |
| 读取缓冲区的数据 | 写数据到缓冲区 | 缓冲区能容纳的数据元素的最大数量(*创建时限定且不再变*) | 读/写上界 | 下一个要被读/写的元素位置 | 记录上一次读写的位置(Position) |



## 3.1 api

| 方法      | 作用                                        |
| --------- | ------------------------------------------- |
| flip()    | 切换到读模式 limit=position position=0      |
| clear()   | 清空缓冲区 position=0 limit=capacity        |
| compact() | 清除已读数据，未读数据移到0                 |
| rewind()  | position=0，limit不变，读模式重读buffer数据 |
| mark()    | 标记buffer中特定position，通过reset恢复     |
| reset()   | 恢复到mark()标记 *类似于回滚*               |
|           |                                             |



## 3.2 从buffer读/写数据

读：

- buffer读取到channel中：`channel.write(buffer)`
- `buffer.get()`

写：

- channel读入buffer:  `channel.read(buffer)`
- `buffer.put()`



## 3.3 缓冲区类型

### 3.3.1 缓冲区分片slice()

根据现有缓冲区分片出一个子缓冲区共享现有缓冲区数据，视图窗口。



操作:  **重设原缓冲区position与limit边界，调用slice()方法制造子缓冲区操作position~limit数据。**



### 3.3.2 只读缓冲区

`asReadOnlyBuffer()`返回与原缓冲区完全相同缓冲区数据共享但只读，原缓冲区数据变化则只读也会变化。



### 3.3.3 直接缓冲区

JDK：给定一个直接字节缓冲区，则Java虚拟机将最大努力直接对其执行本机I/O操作，调用底层操作系统I/O操作时避免将缓冲区内容拷贝到一个中间缓冲区 or 从一个中间缓冲区拷贝数据。



`allocateDirect()`, 使用方法与普通缓冲区无差。调用Unsafe直接向内存申请缓冲区； 而`allocat()`申请JVM内存。





### 3.3.4 内存映射文件I/O

读/写文件方法，比常规的基于流/通道的I/O快很多。 基于内存的操作。  内存映射文件I/O并不直接将文件读取到内存，而是将实际读取/写入内容映射到内存。

`MapperByteBuffer m = channel.map(...)`





# Selector选择器

用于检查一个/多个Channel是否处于可读/可写等状态，实现单线程管理多个Channels(多个网络连接)

SelectableChannel - 继承该抽象类的Chanel才可被Selector注册复用；**一个通道可以被注册到多个选择器而一个选择器只能被注册一次。？注册时指定通道的哪些操作是selector感兴趣的**

- Channel注册到Selector：`Channel.register(Selector, Int)` 分别指定选择器与查询的通道操作
  - SelectionKey. [**1 OP_READ, 4 OP_WRITE, 8 OP_CONNECT, 16 OP_ACCEPT**]  可通过 | 指定多个参兴趣操作类型
- 选择键SelectionKey : Channel注册后且通道处于某种就绪状态时可被选择器查询到 `Selector.select()`; Selector可轮询Channel中的操作有关的就绪状态将选中放入到选择键集合中。
- 处理完就绪事件后需及时删除对应事件(selectedKeys不会自己删除)  *若不手动删除下次判断时调用对应api (accept)但返回null，异常*





## Selector操作

1. 开启通道 `Selector.open()`
2. 通道注册到选择器 `Channel.register(selector, key_type, Object att)`

   - 注册可附带att对象，该key被select()监听可用时附带att对象。
3. 查询就绪操作于SelectionKey中 `selector.select()`返回int就绪数量
   - 可指定阻塞时长 or now非阻塞立即返回
   - 对于select()阻塞的线程，可通过其他线程wakeup() / close()方法唤醒/关闭
4. select返回值不为0时 `selector.selectedKeys()`访问已选择集合

注：

1. 与Selector使用的Channel必须支持非阻塞模式 （不支持FileChannel）
2. 通道不一定支持所有四种操作 `channel.validOps()`查询支持操作



关注操作系统底层对于selector操作的实现 epoll/poll/select?

# NIO socket相关流程

1. 创建ServerSocketChannel, 监听绑定端口
2. 设置通道非阻塞
3. 创建Selector选择器，注册响应Channel
4. 调用Selector选择器监听通道对应就绪状态
5. 获取SelectionKeys，遍历根据类型进行操作，移除



# NIO others

## 1. Pipe管道

管道进行两个线程间单向的数据连接 

- A -> SinkChannel -> SourceChannel -> B



流程

1. 打开管道 `Pipe.open();`
2. 写入管道 `pipe.sink()`;  `channel.write(buffer)`;
3. 读取管道 `pipe.souce()`; `channel.read(buffer)`;



## 2. FileLock文件锁

避免文件数据不同步问题，给文件加锁。

- OS中文件锁为进程级别，不能解决多线程并发访问修改问题
- Java文件锁 - 当前程序所属JVM实例持有，获取文件锁时需调用release() / 关闭对应FileChannel对象 / JVM退出。  不可重复加锁  **通过FileChannel使用文件锁**





使用方法

- `lock()` 对整个文件加锁，默认排它锁
- `lock(position, size ,shared)` 指定文件内容以及是否共享锁
- `tryLock()`系列类似，非阻塞式，获取锁失败时不会阻塞而是返回null





# Linux下I/O多路复用模型

## 1. select

I/O事件发生时通知，**但不知哪个流是就绪的，需轮询所有流获取就绪流**



select(maxfd+1, readEvent, writeEvent, exceptEvent, time_out)

- Event参数指定为bitmap, *FD_SET(fds[i], &eventSet)*指定要监听的event事件  **bitmap默认最大1024位**
- max+1 ，在max+1个文件描述符中取出要监听的文件描述符  （max为所监听的文件描述符的最大值）



```c
while(1) {
    //rset:bitmap  监听的就绪事件,位置0
    FD_ZERO(&rset);
    for (int i = 0; i < 5; i++) {
        //给rset置位要监听的fd，根据器fd编号int值
        FD_SET (fds[i], &rset);
    }
    
    //调用select函数返回就绪的fds (将对应的rset事件位置位表示就绪状态)
    select(max+1, &rset, NULL, NULL, NULL);

    //使用FD_ISSET判断fd是否就绪，进行相应处理
    for (int i = 0; i < 5; i++) {
        if (FD_ISSET(fds[i], &rset)) {
            //handler;
        }
    }
}
```

流程：

- 将fds从用户空间拷贝到内核空间；遍历fds对每个fd调用poll逻辑检查fd是否有就绪事件，将当前进程挂到fd等待队列中；若遍历完毕没有就绪事件则sleep；sleep时间到/sleep期间有就绪事件则select进程被唤醒继续执行上述逻辑。

1. copy_from_user 将文件描述符集合从用户空间拷贝到内核空间
2. 遍历fd集合(socket)，调用fd对应等待集合poll方法，将当前进程挂到设备的等待队列中（不同设备不同等待队列），设备收到一条消息/读写完文件数据后唤醒等待队列上的进程。
3. poll方法返回一个描述读写操作是否就绪的mask掩码，给对应的fd bitmap赋值，对其进行相应的操作。
4. 若无fd有一个就绪mask掩码，将调用select的进程阻塞一段时间，直至时间到达或者有就绪。



缺点：

1. 将fds从内核态拷贝到用户态开销大
2. bitmap位图数量有限，默认1024
3. bitmap位每轮循环都需要重新置位，select会改变bitmap位
4. fds有就绪事件就绪整个遍历fds，开销大。



## 2. poll

实现方式与select类似，描述fd集合的方式不同，poll使用pollfd，**没有最大文件描述符数量的限制**，



```c
struct pollfd{
    int fd; //监听的文件描述符
    short events; //监听的事件 POLLIN/POLLOUT
    short revents; //poll方法返回的就绪事件 初始0
}

while (1) {
    //pollfds监听的套接口，5数量，500000超时时间
    poll(pollfds, 5, 50000);
    
    for (i = 0; i < 5; i++) {
        if (pollfds[i].revents & POLLIN) {
            //处理就绪事件时同时将revents置位为0
            pollfds[i].revents = 0;
            //handler 处理对应事件
        }
    }
}
```



相较于select优化

1. pollfd数组，无限制
2. 不需要重复复位&rset



## 3. epoll  （windows IOCP）

epoll使用 "事件"的就绪通知方式，通过epoll_ctl注册fd，fd就绪时内核采用callback回调机制激活fd，epoll可收到对应激活fd通知避免无谓轮询操作。



```c
struct epoll_event events[5]; //epoll_event 有fd和events没有revents
int epfd = epoll_creat(10); //创建epoll文件句柄，红黑树结构存储监听文件描述符

for (int i = 0; i < 5; i++) {
    //....
    ev.events = EPOLLIN; //监听对应事件
    //epfd指定epoll文件句柄；EPOLL_CTL_ADD指定对epfd的操作；指定的操作fd；
    epoll_ctl(epfd, EPOLL_CTL_ADD, ev.data.fd, &ev);
}

while(1) {
    //epfd为epoll文件句柄，events指定要监听的事件，数量&超时时间；返回指定事件就绪fds
    //epoll_wait通过回调函数将就绪描述符加入一个链表，最终返回就绪链表。
    nfds = epoll_wait(epfd, events, 5, 10000);
    
    for(fd: nfds) {
        //handler fd
    }
}
```



实现方式 & 对应优点：

1. epoll_create: 创建epoll句柄，产生epoll专用文件描述符epfd
2. epoll_ctl: 注册函数 增/删/改文件描述符于epfd，注册要监听的事件类型
   - 每次注册新的事件到epoll句柄中时，将所有fd拷贝到内核。而非在epoll_wait时拷贝。 保证fd仅拷贝一次
   - epoll_ctl对每个fd指定一个回调函数，fd就绪时将对应fd加入指定事件的就绪队列。
3. epoll_wait: 等待事件的发生，收集已经就绪事件
   - sleep一会查看一会就绪队列而非整个的轮询。



EPOLL LT: 默认模式，只要fd就绪就返回，未处理完时仍会返回。（未读完状态）

EPOLL ET: 高速模式，就绪fd只会被提示一次，直到下次就绪态发生之前都不会做提示。效率高。



## 总结分析

连接数少且十分活跃： select or poll, 因为epoll需要更多的回调操作而select poll只需少量轮询个。



# Questions from now:

**https://zhuanlan.zhihu.com/p/444503354**

1. Channel中的read、write有关方法
2. Buffer中的内存映射、直接内存。 零拷贝?
3. Linux相关命令
   - epoll与selector？
4. Why FileChannel不可非阻塞
5. open()方法对应channel与selector？ why open?
6. Selector实践中的客户端为什么总是NotConnectedYet?
7. **为什么write操作使用ByteBuffer.wrap()可以，而是用原生Buffer写byte[]无法写入？**