# -Linux多线程服务端编程笔记-
## 第一章 线程安全的对象生命周期管理 
* 1.构造函数的作用是构造对象，初始化，不要泄露对象的指针，调用成员函数。
* 2.一个函数如果要锁住相同类型的多个对象，为了避免死锁（多个锁，顺序不同可能出现），可以比较mutex对象的地址，始终先加锁地址较小的muex。
* 3.使用智能指针 shared_ptr 做函数参数最好用常引用 防止拷贝 延长对象生命周期
## 第二章 线程同步精要
* 1.最低限度地共享对象，减少需要同步场合。
* 2.尽量用高层同步设施，不得已使用底层同步原语时，只用非递归的互斥器和条件变量，慎用读写锁，不要用信号量。<br>
（递归锁：同一个线程可以多次获取同一个递归锁，不会产生死锁。<br>
  非递归锁：如果一个线程多次获取同一个非递归锁，则会产生死锁。Linux下pthread_mutex_t锁是默认是非递归的。）
* 3.条件变量的使用。<br>
等待端（消费者）<br>
 a.mutex lock<br>
 b.while(empty){ //没有任务<br>
     cond.wait()<br>
 }<br>
 c.if(!empty) dowork remove//再判断 完成任务 移除任务<br>
 通知端（生产者）<br>
 a.mutex lock<br>
 b.insert<br>
 c.cond.notify<br>
 ## 第三章 多线程服务器的适用场合与常用编程模型
 * 1.进程间通信首选Sockets(TCP) 可以跨主机，具有伸缩性
## 第四章 C++多线程系统编程精要
* 1.凡是非共享对象都是彼此独立的，如果一个对象始终只被一个线程用到，那么该线程是安全的。
* 2.如果是共享对象，只读不写是安全的，一旦有写，读必须加锁。
* 3.在posix线程api中，通过pthread_self(void) 函数获取当前线程的id 定义为<br>
extern pthread_t pthread_self (void) __ THROW __ attribute__ ((__ const__));<br>
返回值是pthread_t 定义为 typedef unsigned long int pthread_t;//声明为无符号长整型<br>
pthread_t不一定是一个数值类型（整数或指针），也有可能是一个结构体，可通过pthread_equal函数判断是否相等。
* 4.pthread_t并不适合做函数的标识符，可使用gettid返回pid_t来区分。
* 6.程序中线程的创建最好能在初始化阶段全部完成，这样线程是不必销毁的，伴随程序一直运行。如果线程需要退出，让其自然死亡（return）
* 7.__ thread变量每一个线程有一份独立实体，各个线程的值互不干扰。可以用来修饰那些带有全局性且值可能变，但是又不值得用全局锁保护的变量。
* 8.每个fd只由一个线程操作，一个线程可以操作多个fd，但一个线程不能操作其他线程拥有的fd。
* 9.用socket对象封装文件描述符，在该对象析构函数关闭文件描述符，这样的话只要该对象活着，就不会发生串话。
* 10.fork()之后子进程只有一个线程，其他线程都消失，如果有线程刚好持某个锁，而其突然死亡没得解锁，这时候fork出来的子进程再对同一个加锁会造成死锁。（不要在多线程环境使用fork()）。
## 第五章 高效的多线程日志
* 1.Log Every All The Time<br>
 a.收到的每条内部消息的id（还可以包括关键字段、长度、hash等）<br>
 b.收到的每条外部消息的全文<br>
 c.发出的每条消息的全文，每条消息都有全局唯一的id<br>
 d.关键内部状态的变更，等等
* 2.muduo日志库采用的是双缓冲技术，前段负责往buufer写数据，当buufer满时交换数据给后端（移动而非复制），后端再负责将buffer写入本地。
## 第六章 muduo网络库简介
* 1.三个半事件<br>
 a.连接的建立，包括服务端接受连接，客户端发起连接。TCP连接一旦建立，服务端和客户端是平等的，相互发送和接收数据。<br>
 b.连接的断开，包括主动断开（close、shutdown）和被动断开（read()返回0）。<br>
 c.消息到达，文件描述符可读。这是最重要的一个事件，对它的处理方式决定了网络风格的风格（阻塞还是非阻塞，如何处理分包，如何设计应用层缓冲等等）。<br>
 d.消息发送完毕，这算是半个。对于低流量的服务可以不关心这个事件；“发送完毕”是指将数据写入操作系统的缓冲区，将由TCP协议栈负责数据的发送与重传，不代表对方已接收到数据。
 * 2.TcpConnection::setContext() getContext()<br>
  void setContext(const boost::any& context)<br>
  { context_ = context; }<br>
  const boost::any& getContext() const<br>
  { return context_; }<br>
 * 3.短连接 <br>
 连接->传输数据->关闭连接<br>
 * 4.长连接 <br>
 连接->传输数据->保持连接 -> 传输数据-> ...........->直到一方关闭连接，多是客户端关闭连接。<br>
 * 5.长连接短连接区别 <br>
 长连接多用于操作频繁，点对点的通讯，而且连接数不能太多情况。<br>
 而像WEB网站的http服务一般都用短链接，因为长连接对于服务端来说会耗费一定的资源，而像WEB网站这么频繁的成千上万甚至上亿客户端的连接用短连接会更省一些资源。<br>
 * 6.muduo Buffer 在头部预留了8个字节的空间，可调用prepend()操作往前追加数据，效率高。
