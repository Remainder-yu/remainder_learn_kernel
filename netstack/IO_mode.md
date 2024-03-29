# 概述
IO模型的选择在Linux网络编程中十分重要。在linux环境中主要提供了五种不同的IO模型，分别是：
* 阻塞式IO
* 非阻塞式IO
* IO多路复用
* 信号驱动方式IO
* 异步IO

通常一个输入操作包含两个不同阶段：
1. 等待数据准备好
2. 从内核向进程复制数据
![Alt text](../vx_images/IO操作模型.png)
例如，一个网络套接字上的输入操作，第一步通常涉及到发生系统调用，用户态切换到内核态并等待数据从网络中到达，当所有等待分组到达时，数据会复制到内核中的某个缓冲区。第二步则是数据从内核缓冲区复制到应用进程缓冲区。
## 基本概念

# 阻塞IO模型
在linux中，所有socket默认情况下都是阻塞的。这里有必要辨析以下阻塞和非阻塞这两个概念，这两个概念描述的是用户线程调用内核IO操作方式，其中阻塞时指I/O操作需要彻底完成后才返回到用户空间；而非阻塞式指IO操作北调用后立即返回给用户一个状态值，不需要等待IO操作彻底完成。

![阻塞IO模型](../vx_images/阻塞IO模型.png)




# 深入理解IO
date: 2023.10.07
# 概念介绍
## 阻塞blocking和非阻塞non-blocking：
可以理解为需要做一件事能不能立即得到返回应答，如果不能立即获得返回，需要等待，此时就会造成阻塞。否则就可以理解为非阻塞。

### Q1：如何实现阻塞和非阻塞？
[知乎链接答案](https://www.zhihu.com/question/391359472)

阻塞本质上是资源调度过程中出现的一种现象。为了完成某个任务，需要在某个时间使用某个资源，但是却发现该资源不可用。对于这种现象，我们就说此任务被该资源阻塞了。
对待阻塞有两种处理模式，分别是同步方式和异步方式。
同步方式，感觉像是无限循环，但通常并不是。
异步方式，主要看代码实现方式。
如果用同步方式处理这个阻塞，那么对于这段程序来说，就是死等，直到数据被传输过来，然后程序读到数据，继续执行下去。站在这段程序的角度，感觉上像是进入了一个无限循环，循环里在不停的检查网络上是不是有数据，一旦有就读出数据，然后继续处理。真实情况是不是这样呢？
对于抢占式多任务操作系统来说，当一个任务被阻塞时，该任务所在的进程或线程会被暂停运行，将进程或线程状态保存下来，转去执行其他未被阻塞的其他任务。当阻塞条件解除后，该进程或线程会得到重新调度，得以继续执行。
也就是说，站在这段程序来说，感觉上好像在死等，但站在操作系统的角度上，并没在死等该程序的执行，该程序已经被暂停挂起了，并没在执行。

阻塞还有一种异步处理方式，细分下来也有多种处理方式：
1. 程序不是直接去读网络，而是先询问网络是否可读，可读就去读，不可读就先做别的事，做完后再来检查网络是否可读，如此循环。
2. 程序注册一个回调例程，然后去做别的事。当网络可读时回调例程被自动调用。
3. 程序使用多路复用技术，同时要管理多个网络读写，哪个可读就读哪个。


## 同步synchronous、异步asynchronous：

如果说做完一件事再做另一件，不管是否需要时间等待，这就是同步。（就是在发出一个功能调用时，在没有得到结果之前，该调用就不返回，即此时不能做下一件事）
异步：可以同时做几件事情，并非一定需要做完一件事件做另一件事情（当一个异步过程调用发出后，调用者不能立即得到结果，此时可以接着做其他事情）。

## 阻塞和同步：

同步调用:很多时候当前线程还是激活的，只是从逻辑上当前函数没有返回而已。例如：我们在socket中调用recv函数，如果缓冲区中没有数据，这个函数就会一直等待，直到有数据才返回。而此时，当前线程还会继续处理各种各样的信息。

## 进程的阻塞
正在执行的进程，由于期待的某些事件未发生，如请求资源失败，等待某种操作的完成、数据尚未到达或无新工作做等，则由系统自动执行阻塞原语（Block），使自己由运行状态变为阻塞状态。可见，进程的阻塞是进程自身的一种主动行为，也因此只有处于运行态的进程（获得CPU），才可能将其转为阻塞状态。当进程进入阻塞状态，是不占用CPU资源的。

## 文件描述符准备就绪通知方式
水平触发通知：如果文件描述符上可以非阻塞地执行I/O系统调用，此时认为其已经就绪。
边缘触发通知：如果文件描述符自上次状态检查以来有了新的I/O活动（如新的输入），此时需要触发通知。

### 触发方式对程序设计的影响
* 水平触发：当采用水平触发时，我们可以在任意时刻检查文件描述符的就绪状态。当确定其就绪状态后就可以对其进行I/O操作，然后重复检查文件描述符，已确定是否仍然处于就绪状态，此时就可以执行更多的I/O。也就是说，因为水平触发允许我们在任意时刻重复检查I/O状态，也就没有必要每次文件描述符就绪后就尽可能多地执行I/O（尽可能多地读取字节，亦或是不去执行I/O）。
* 边缘触发：当采用边缘触发时，只有当I/O事件发生时才会得到通知。在另一个I/O时间到来之前我们不会收到任何新的通知。此外当文件描述符收到I/O事件通知时，我们并不知道要处理多少I/O（有多少数据可读）。因此采用边缘触发通知的程序应该在接收到一个I/O事件通知后，程序在某个时刻（在有些时候我们确定文件描述符是就绪态时，此时不适合大量的I/O操作。因为如果我们仅对一个文件描述符执行大量I/O操作，可能会让其他文件描述符处于饥饿状态）应该在相应的文件描述符上尽可能多地执行I/O（尽可能多地读取字节，与水平触发相反）。但是若是程序如此设计，就可能失去执行I/O的机会。因为知道产生另一个I/O事件为止，在此之前程序都不会在接收到通知了，也就不知道何时执行I/O了。这也将导致数据丢失或者程序中出现阻塞。

如果程序采用循环来对文件描述符执行尽可能多的I/O，而文件描述符又被置为可阻塞的，那么最终当没有更多文件可执行时，I/O系统调用就会阻塞。因此，每个被检查的文件描述符通常都应该被设为非阻塞模式。在得到I/O事件通知后会重复执行I/O操作，直到相应的系统调用 (如read()、write()) 以错误码EAGAIN或EWOULDBLOCK的形式失败。

# linux环境中的五种IO模型

* 阻塞式IO
* 非阻塞式IO
* IO多路复用
* 信号驱动式IO
* 异步IO


![image](https://user-images.githubusercontent.com/87457873/127822245-eabba2a8-0f24-4f77-89c8-7b471a31bd83.png)


通过上面的图片，可以发现non-blocking IO和asynchronous IO的区别还是很明显的。在non-blocking IO中，虽然进程大部分时间都不会被block，但是它仍然要求进程去主动的check，并且当数据准备完成以后，也需要进程主动的再次调用recvfrom来将数据拷贝到用户内存。而asynchronous IO则完全不同。它就像是用户进程将整个IO操作交给了他人（kernel）完成，然后他人做完后发信号通知。在此期间，用户进程不需要去检查IO操作的状态，也不需要主动的去拷贝数据。

通常一个输入操作包含两个不同阶段：
1. 等待数据准备好
2. 从内核向进程复制数据

例如，对于一个网络套接字上的输入操作，第一步通常涉及到发生系统调用，用户态切换到内核态并等待数据从网络中到达，当所有等待分组到达时，数据会被复制到内核某个缓冲区。第二步则是将数据从内核缓冲区复制到进程缓冲区。

## 概念前提
分析是否阻塞的前提是，基于描述用户线程调用内核IO操作的方式，其中阻塞是指IO操作需要彻底完成后才返回到用户空间；而非阻塞则是指IO操作被调用后立即返回给用户一个状态值，不需要等到IO操作彻底完成。

其实在分析过程中，主要分析linux中socket创建及数据传输过程即可，可以参考《深入理解Linux网络》。
没有特殊指定参数，几乎所有的IO都是阻塞型的，即系统调用时不返回调用结果，只有当该系统系统调用获得结果或者超时出错才返回。带来的负面影响就是，线程将无法执行任何运算或者相应任何网络请求。

>如何避免？
在服务器端使用阻塞IO模型，结合多线程技术。让每一个连接都独立的进/线程，任何一个阻塞都不会影响。
可以采用线程池和连接池降低线程创建的频率，减少系统开销。

## 非阻塞IO
进程把一个套接字设置成非阻塞主要目的：
通知内核，当请求IO操作时不需要进入睡眠，等待数据可读返回，而是直接返回一个错误。因此，打开O_NONBLOCK标志，则会以阻塞方式打开文件。如果I/O系统化调用不能立即完成，则会返回错误而不是阻塞进程。
现象：非阻塞IO可以实现周期性检查（轮询）某个文件描述符是否可执行IO操作。

![image](https://user-images.githubusercontent.com/87457873/127650167-86e622e8-4e04-437d-9e29-aa3cacc78968.png)

### 如何实现非阻塞IO操作：
当设定一个文件描述符设置为非阻塞模型时，然后周期性的执行非阻塞式读操作。如果需要同时检测多个文件描述符，则将其都设为非阻塞，然后依次轮询。
缺点：**轮询的效率不高，程序响应IO事件的延迟将难以接受，紧凑循环会浪费CPU执行时间**

# IO多路复用模型
IO多路复用（也叫事件驱动IO），通过系统调用select、poll、epoll实现进程同时检查多个文件描述符状态，以快速找出其中任何一个可执行IO操作。
其实该情况下，使用了两次系统调用而阻塞IO模型只是用了一个系统调用。但是IO多路复用的优势在于可以同时处理多个连接。因此如果处理连接数不是特别多情况下使用IO多路复用模型并不比阻塞式IO好。

select和poll原理基本相同：
1. 注册待侦听的FD（建议创建的fd都是非阻塞）
2. 每次调用都去检查这些fd的状态，当有一个多个就绪返回
3. 返回结果中包括已就绪的fd和未就绪的fd

![image](https://user-images.githubusercontent.com/87457873/127650193-d91a7751-d9be-43e4-ace1-d153f18d82e7.png)

select，poll，epoll都是IO多路复用的机制。I/O多路复用就是通过一种机制，一个进程可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作。但select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。

## select篇
### 接口定义
```c
int select (int maxfdp, fd_set *readfds, fd_set *writefds, fd_set *errorfds, struct timeval *timeout);
```
#### 参数及返回值说明
接口函数定义：
* maxfdp: 指集合中所有文件描述符的范围，即所有文件描述符的最大值+1
* readfs、writefds、errorfds指向文件描述符集合的指针，分别检测输入、输出是否就绪和异常情况是否发生
* timeout时select()的超时时间，控制着select的阻塞行为

readfs、writefds、errorfds所指结构体都是保存结果的地方，在调用select之前，这些参数指向结构体必须初始化以包含我们所感兴趣的文件描述符集合。之后select会修改这些结构体，当其返回时他们包含的就是处于就绪态的文件描述符集合。

性能分析：
select：效率低，性能不太好。不能解决大量并发请求的问题。
它把1000个fd加入到fd_set（文件描述符集合），通过select监控fd_set里的fd是否有变化。如果有一个fd满足读写事件，就会依次查看每个文件描述符，那些发生变化的描述符在fd_set对应位设为1，表示socket可读或者可写。
Select通过轮询的方式监听，对监听的FD数量 t通过FD_SETSIZE限制。

两个问题：

1、select初始化时，要告诉内核，关注1000个fd， 每次初始化都需要重新关注1000个fd。前期准备阶段长。<br>
2、select返回之后，要扫描1000个fd。 后期扫描维护成本大，CPU开销大。

## poll篇
### 接口定义
```c
#include <poll.h>
int poll (struct pollfd fds[], nfds_t nfds, int timeout);
```
#### 参数及返回值说明


## select和poll对比总结

### 功能对比
linux实现中，select和poll都使用了内核轮询（poll）程序集合，与系统调用poll不同，内核每个poll都返回有关文件描述符就绪的信息，这个消息以掩码形式返回，其值同poll()系统调用返回的revent字段中的比特值相关。
对于系统调用select()则可以使用一组宏将内核poll例程返回的信息转化为由select()返回的与之对应的事件集合。
```c
#define POLLIN_SET (POLLIN | POLLRDNORM | POLLRDBAND | POLLHUP | POLLERR) /*读就绪*/
#define POLLOUT_SET (POLLOUT | POLLWRNORM | POLLWRBAND | POLLERR) /*写就绪*/
#define POLLEX_SET (POLLPRI) /*异常*/
```
以上宏定义展现了select()和poll()返回信息间的语义关系，唯一一点不同是如果被检查的文件描述符中有一个关闭了，poll()在revent字段中返回POLLNVAL，而select()返回-1并把错误码置为EBADF。

#### API接口层面不同：
编程调试然后补充：

#### 性能层面不同：
在待检查文件描述符范围较小（最大文件描述符较低），或者有大量文件描述符待检查，但是其分布比较密集时poll()和select()性能相似。
在被检查文件描述符集合很稀疏的情况，poll()要优于select()。

### select()和poll()的不足
1. IO效率随着文件描述符的数量增加而线性下降。每次调用select或poll内核都要检查所有被指定的文件描述符状态（实际上只有部分是活跃的），如果文件描述符集合增大，那么IO效率也随之下降。
2. 当检查大量文件描述符时，用户空间和内核空间消息传递速度较慢。每次调用select或poll时，程序都必须传递一个表示所有需要被检查的文件描述符的数据结构到内核，在内核完成检查后，然后修改这个数据结构返回给用户。数据拷贝消耗。
3. select或poll调用完成后，程序必须检查返回的数据结构中每个元素，已确定那个文件描述符处于就绪态。
4. select对一个进程打开的文件描述符数据上限值。

## epoll篇
### 接口定义

![image](https://user-images.githubusercontent.com/87457873/127650321-094aa0ad-c49b-46d9-8100-4b7c3cc60f53.png)


### 实现原理

epoll在内核中的实现不是通过轮询的方式，而是通过注册callback函数的方式。当某个文件描述符发现变化，就主动通知。成功解决了select的两个问题:
1. select的健忘症，一返回就不记得关注多少fd。api把告诉内核等哪些文件，和最终监听哪些文件，都是同一个api。而epoll，告诉内核等哪些文件和具体等哪些文件分成两个api系统调用，epoll_wait返回后，还知道关注了哪些fd。
2. select在返回后的维护开销很大，而epoll就可以直接直到需要等fd。

![image](https://user-images.githubusercontent.com/87457873/127650306-b4419ff5-164e-4678-80f4-6f839ad44245.png)

epoll获取事件的时候，无须遍历整个被侦听的描述符集，只要遍历那些被内核I/O事件异步唤醒而加入就绪队列的描述符集合。
1. epoll_create: 创建epoll池子。
2. epoll_ctl：向epoll注册事件。告诉内核epoll关心哪些文件，让内核没有健忘症。
3. epoll_wait：等待就绪事件的到来。专门等哪些文件，第2个参数 是输出参数，包含满足的fd，不需要再遍历所有的fd文件。

![image](https://user-images.githubusercontent.com/87457873/127650368-39c111d9-a074-4f0b-befd-3c0e9a29867b.png)
如上图，epoll在CPU的消耗上，远低于select，这样就可以在一个线程内监控更多的IO。


### 设计特点
epoll是在2.6内核中提出的，是之前的select和poll的增强版本。相对于select和poll来说，epoll更加灵活，没有描述符限制。epoll使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的copy只需一次。

### 实现细节

### epoll优点

1. 监视的文件描述符不受限制
2. IO效率不随监视fd上升而下降。因为epoll不是轮询的方式，而是通过每个fd定义的回调函数来实现的。只有就绪的fd才会执行回调函数。

如果没有大量的idle-connection或者dead-connection，epoll的效率并不会比select/poll高很多，但是当遇到大量的idle-connection，就会发现epoll的效率大大高于select/poll。

* [Epoll详解及源码分析](https://blog.csdn.net/weiyuefei/article/details/53006659)

# 异步IO
对于IO操作主要有两种，同步IO和异步IO，对于同步IO会导致请求进程阻塞，直到IO操作完成，即必须等到IO操作完成以后控制权才会返回给用户进程；而异步IO不会导致请求进程阻塞，则无需等待IO操作完成就将控制权返回给用户进程。

异步IO模型工作机制：告知内核启动某个操作，并让内核在整个操作（包括数据从内核复制到进程缓冲区）完成后通知我们。主要方式调用aio_read函数向内核传递描述符、缓冲区指针、缓冲区大小（与read相同的三个参数）和文件偏移，并告知内核当整个操作完成时如何通知用户进程。该系统调用立即返回，在等待I/O完成期间进程不被阻塞。
异步I/O模型则是由内核通知我们I/O操作何时完成。
信号驱动式I/O是由内核告诉我们何时可以启动一个I/O操作。

![image](https://user-images.githubusercontent.com/87457873/127822064-307fe305-e912-4c89-84f7-34858f4644ac.png)


# I/O模型的技术选择

《Linux中的五种IO模型》I/O模型的技术选择章节。

