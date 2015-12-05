# share
### 进程 线程 协程 异步
并发编程（不是并行）编程目前有四种方式：多进程、多线程、协程和异步。
* 多进程编程在python中有类似C的os.fork,更高层封装的有multiprocessing标准库
* 多线程编程python中有Thread和threading
* 异步编程在linux下主要有三种实现select，poll，epoll
* 协程在python中通常会说到yield，关于协程的库主要有greenlet,stackless,gevent,eventlet等实现。

#### 进程
* 不共享任何状态
* 调度由操作系统完成
* 有独立的内存空间（上下文切换的时候需要保存栈、cpu寄存器、虚拟内存、以及打开的相关句柄等信息，开销大）
* 通讯主要通过信号传递的方式来实现（实现方式有多种，信号量、管道、事件等，通讯都需要过内核，效率低）

#### 线程
* 共享变量（解决了通讯麻烦的问题，但是对于变量的访问需要加锁）
* 调度由操作系统完成（由于共享内存，上下文切换变得高效）
* 一个进程可以有多个线程，每个线程会共享父进程的资源（创建线程开销占用比进程小很多，可创建的数量也会很多）
* 通讯除了可使用进程间通讯的方式，还可以通过共享内存的方式进行通信（通过共享内存通信比通过内核要快很多）

#### 协程
* 调度完全由用户控制
* 一个线程（进程）可以有多个协程
* 每个线程（进程）循环按照指定的任务清单顺序完成不同的任务（当任务被堵塞时，执行下一个任务；当恢复时，再回来执行这个任务；任务间切换只需要保存任务的上下文，没有内核的开销，可以不加锁的访问全局变量）
* 协程需要保证是非堵塞的且没有相互依赖
* 协程基本上不能同步通讯，多采用异步的消息通讯，效率比较高

### 总结

* 进程拥有自己独立的堆和栈，既不共享堆，亦不共享栈，进程由操作系统调度
* 线程拥有自己独立的栈和共享的堆，共享堆，不共享栈，线程亦由操作系统调度(标准线程是的)
* 协程和线程一样共享堆，不共享栈，协程由程序员在协程的代码里显示调度

进程和其他两个的区别还是很明显的。
协程和线程的区别是：协程避免了无意义的调度，由此可以提高性能，但也因此，程序员必须自己承担调度的责任，同时，协程也失去了标准线程使用多CPU的能力。


gevent 是基于 greenlet 的一个 python 库，它可以把 python 的内置线程用 greenlet 包装，这样在我们使用线程的时候，实际上使用的是协程，在上一个协程的例子里，协程 A 结束时，由协程 A 让位给协程 B ，而在 gevent 里，所有需要让位的协程都让位给主协程，由主协程决定运行哪一个协程，gevent 也会包装一些可能需要阻塞的方法，比如 sleep ，比如读 socket ，比如等待锁，等等，在这些方法里会自动让位给主协程，而不是由程序员显示让位，这样程序员就可以按照线程的模式进行线性编程，不需要考虑切换的逻辑。

gevent 版的命令发生了 3 次切换：主协程 -> A -> 主协程 -> B

### 聊聊协程
协程，又称微线程，纤程。英文名Coroutine。
协程的概念很早就提出来了，但直到最近几年才在某些语言（如Lua）中得到广泛应用。
Python的线程并不是标准线程，是系统级进程，线程间上下文切换有不小的开销，而且Python在执行多线程时默认加了一个全局解释器锁（GIL），因此Python的多线程其实是串行的，所以并不能利用多核的优势，也就是说一个进程内的多个线程只能使用一个CPU。

    def coroutine(func):
        def ret():
            f = func()
            f.next()
            return f
        return ret
  
  
    @coroutine
    def consumer():
        print "Wait to getting a task"
        while True:
            n = (yield)
            print "Got %s",n
                
    
    import time
    def producer():
        c = consumer()
        task_id = 0
        while True:
            time.sleep(1)
            print "Send a task to consumer" % task_id
            c.send("task %s" % task_id)
            
    if __name__ == "__main__":
        producer()
  
  结果：
    
    Wait to getting a task
    Send a task 0 to consumer
    Got task 0
    Send a task 1 to consumer
    Got task 1
    Send a task 2 to consumer
    Got task 2
    ...

  传统的生产者-消费者模型是一个线程写消息，一个线程取消息，通过锁机制控制队列和等待，但一不小心就可能死锁。
如果改用协程，生产者生产消息后，直接通过yield跳转到消费者开始执行，待消费者执行完毕后，切换回生产者继续生产，效率极高：
  服务编程范式以这样的方式进化多进程--->多线程--->非阻塞--->协程
  
### 正题Gevent
Python通过yield提供了对协程的基本支持，但是不完全。而第三方库gevent为Python提供了比较完善的协程支持。
gevent是通过greenlet实现协程，其基本思想是：
> 当一个greenlet遇到IO操作时，比如访问网络，就自动切换到其他的greenlet，等到IO操作完成，
> 再在适当的时候切换回来继续执行。由于IO操作非常耗时，经常使程序处于等待状态，
> 有了gevent为我们自动切换协程，就保证总有greenlet在运行，而不是等待IO。

由于切换是在IO操作时自动完成，所以gevent需要修改Python自带的一些标准库，这一过程在启动时通过monkey patch完成：
  
  而在Java、C#这样的语言中，多线程真的是并发的，虽然可以利用多核优势，但由于线程的切换是由调度器控制的，不论是用户级线程还是系统级线程，调度器都会由于IO操作、时间片用完等原因强制夺取某个线程的控制权，又由于线程间共享状态的不可控性，同时也会带来安全问题。所以我们在写多线程程序的时候都会加各种锁，很是麻烦，一不小心就会造成死锁，而且锁对性能，总是有些影响的。

然后说到协程，与线程的抢占式调度不同，它是协作式调度，之前忘了看哪个技术分享视频说过，纤程（跟协程一样，只是叫法不同），大致上就是一种可控制的回调，是比线程更轻量级的一种实现异步编程的方式。协程在Python中可以用generator(生成器)实现，生成器主要由yeild关键字实现（yeild在C#中也有，之前在《深入理解C#》就有看到，然而当时一知半解，真是惭愧）。生成器是一个可迭代的对象，每次调用next方法就返回一个值，或者用for in （C#中是foreach）这样的语法糖，可以自动调用next方法。

而用yeild实现协程的话，其实有挺多不同的方式，不过大致的思想都是用yeild来暂停当前执行的程序，转而执行另一个，再在恰当的时候（可以控制）回来执行。用法很灵活。

在此以大家都耳熟能详的生产者-消费者模型为例：
</p>
## 初识gevent

话说gevent也没个logo啥的，于是就摆了这张图= =|||，首先这是一种叫做greenlet的鸟，而在python里，按照官方解释greenlet是轻量级的并行编程，而gevent呢，就是利用greenlet实现的基于协程的python的网络library，好了，关系理清了。。。

话说pycon没有白去阿，了解了很多以前不知道的东西，比如说协程，gevent，greenlet，eventlet。说说协程，进程和线程大家平时了解的都比较多，而协程算是一种轻量级进程，但又不能叫进程，因为操作系统并不知道它的存在。什么意思呢，就是说，协程像是一种在程序级别来模拟系统级别的进程，由于是单进程，并且少了上下文切换，于是相对来说系统消耗很少，而且网上的各种测试也表明，协程确实拥有惊人的速度。并且在实现过程中，协程可以用以前同步思路的写法，而运行起来确是异步的，也确实很有意思。话说有一种说法就是说进化历程是多进程->多线程->异步->协程，暂且不论说的对不对，单从诸多赞誉来看，协程还是有必要理解一下的。

比较惭愧，greenlet没怎么看就直接看gevent，官方文档还是可以看看的，尤其是源码里的examples都相当不错，有助于理解gevent的使用。

gevent封装了很多很方便的接口，其中一个就是monkey

    from gevent import monkey
    monkey.patch_all()

这样两行，就可以使用python以前的socket之类的，因为gevent已经给你自动转化了，真是超级方便阿。

而且安装gevent也是很方便，首先安装依赖libevent和greenlet，再利用pypi安装即可

    sudo apt-get install libevent-dev
    sudo apt-get install python-dev
    sudo easy-install gevent

然后，gevent中的event，有wait，set等api，方便你可以让某些协程在某些地方等待条件，然后用另一个去唤醒他们。

再就是gevent实现了wsgi可以很方便的当作python的web server服务器使。

最后放送一个我利用gevent实现的一个带有缓存的dns server

    # -*- coding: UTF-8 -*-
    
    import gevent
    import dnslib
    from gevent import socket
    from gevent import event
    
    rev=socket.socket(socket.AF_INET,socket.SOCK_DGRAM)
    rev.bind(('',53))
    ip=[]
    cur=0
    
    def preload():
        for i in open('ip'):
            ip.append(i)
        print "load "+str(len(ip))+" ip"
 
    def send_request(data):
        global cur
        ret=rev.sendto(data,(ip[cur],53))
        cur=(cur+1)%len(ip)
 
    class Cache:
        def __init__(self):
            self.c={}
        def get(self,key):
            return self.c.get(key,None)
        def set(self,key,value):
            self.c[key]=value
        def remove(self,key):
            self.c.pop(key,None)
 
    cache=Cache()
 
    def handle_request(s,data,addr):
        req=dnslib.DNSRecord.parse(data)
        qname=str(req.q.qname)
        qid=req.header.id
        ret=cache.get(qname)
        if ret:
            ret=dnslib.DNSRecord.parse(ret)
            ret.header.id=qid;
            s.sendto(ret.pack(),addr)
        else:
            e=event.Event()
            cache.set(qname+"e",e)
            send_request(data)
            e.wait(60)
            tmp=cache.get(qname)
            if tmp:
                tmp=dnslib.DNSRecord.parse(tmp)
                tmp.header.id=qid;
                s.sendto(tmp.pack(),addr)
 
    def handle_response(data):
        req=dnslib.DNSRecord.parse(data)
        qname=str(req.q.qname)
        print qname
        cache.set(qname,data)
        e=cache.get(qname+"e")
        cache.remove(qname+"e")
        if e:
            e.set()
            e.clear()
 
    def handler(s,data,addr):
        req=dnslib.DNSRecord.parse(data)
        if req.header.qr:
            handle_response(data)
        else:
            handle_request(s,data,addr)
 
    def main():
        preload()
        while True:
            data,addr=rev.recvfrom(8192)
            gevent.spawn(handler,rev,data,addr)
 
    if __name__ == '__main__':
        main()
  
这个是直接利用了dict来作为缓存查询了，在这里还有我将dict换成redis持久化实现的另一个版本(话说redis的python api也可以利用pypi安装，pypi这真是个好东西阿)，话说可以将这段代码放到国外的服务器上去运行，然后修改dns的地址去指向它，然后你懂的。。。

比较惭愧，greenlet没怎么看就直接看gevent，官方文档还是可以看看的，尤其是源码里的examples都相当不错，有助于理解gevent的使用。
Python通过yield提供了对协程的基本支持，但是不完全。而第三方的gevent为Python提供了比较完善的协程支持。
gevent是第三方库，通过greenlet实现协程，其基本思想是：
当一个greenlet遇到IO操作时，比如访问网络，就自动切换到其他的greenlet，等到IO操作完成，再在适当的时候切换回来继续执行。由于IO操作非常耗时，经常使程序处于等待状态，有了gevent为我们自动切换协程，就保证总有greenlet在运行，而不是等待IO。
由于切换是在IO操作时自动完成，所以gevent需要修改Python自带的一些标准库，这一过程在启动时通过monkey patch完成：
