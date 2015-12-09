# Gevent

### 进程 线程 协程 异步
并发编程（不是并行）目前有四种方式：多进程、多线程、协程和异步。
* 多进程编程在python中有类似C的os.fork,更高层封装的有multiprocessing标准库
* 多线程编程python中有Thread和threading
* 异步编程在linux下主+要有三种实现select，poll，epoll
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

#### 总结

* 进程拥有自己独立的堆和栈，既不共享堆，亦不共享栈，进程由操作系统调度
* 线程拥有自己独立的栈和共享的堆，共享堆，不共享栈，线程亦由操作系统调度(标准线程是的)
* 协程和线程一样共享堆，不共享栈，协程由程序员在协程的代码里显示调度

### 聊聊协程
协程，又称微线程，纤程。
Python的线程并不是标准线程，是系统级进程，线程间上下文切换有开销，而且Python在执行多线程时默认加了一个全局解释器锁（GIL），因此Python的多线程其实是串行的，所以并不能利用多核的优势，也就是说一个进程内的多个线程只能使用一个CPU。

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

 传统的生产者-消费者模型是一个线程写消息，一个线程取消息，通过锁机制控制队列和等待，但容易死锁。
如果改用协程，生产者生产消息后，直接通过yield跳转到消费者开始执行，待消费者执行完毕后，切换回生产者继续生产，效率极高。
  
##  Gevent
### 介绍
gevent是基于协程的Python网络库。特点：
* 基于libev的快速事件循环(Linux上epoll，FreeBSD上kqueue）。
* 基于greenlet的轻量级执行单元。
* API的概念和Python标准库一致(如事件，队列)。
* 可以配合socket，ssl模块使用。
* 能够使用标准库和第三方模块创建标准的阻塞套接字(gevent.monkey)。
* 默认通过线程池进行DNS查询,也可通过c-are(通过GEVENT_RESOLVER=ares环境变量开启）。
* TCP/UDP/HTTP服务器
* 子进程支持（通过gevent.subprocess）
* 线程池

### 安装和依赖
依赖于greenlet library
支持python 2.6+ 、3.3+

### 核心部分
* Greenlets
* 同步和异步执行
* 确定性
* 创建Greenlets
* Greenlet状态
* 程序停止
* 超时
* 猴子补丁

####Greenlets
gevent中的主要模式, 它是以C扩展模块形式接入Python的轻量级协程。 全部运行在主程序操作系统进程的内部，但它们被程序员协作式地调度。
![Alt text](./1449559519865.png)
> 在任何时刻，只有一个协程在运行。

区别于multiprocessing、threading等提供真正并行构造的库， 这些库轮转使用操作系统调度的进程和线程，是真正的并行。

#### 同步和异步执行
并发的核心思想在于，大的任务可以分解成一系列的子任务，后者可以被调度成 同时执行或异步执行，而不是一次一个地或者同步地执行。两个子任务之间的 切换也就是上下文切换。

在gevent里面，上下文切换是通过yielding来完成的. 


	import gevent

	def foo():
	    print('Running in foo')
	    gevent.sleep(0)
	    print('Explicit context switch to foo again')

	def bar():
	    print('Explicit context to bar')
	    gevent.sleep(0)
	    print('Implicit context switch back to bar')

	gevent.joinall([
	    gevent.spawn(foo),
	    gevent.spawn(bar),
	])

执行结果：
	
	Running in foo
	Explicit context to bar
	Explicit context switch to foo again
	Implicit context switch back to bar

代码执行过程：

![Alt text](http://xlambda.com/gevent-tutorial/flow.gif)

网络延迟或IO阻塞隐式交出greenlet上下文的执行权。

	import time
	import gevent
	from gevent import select

	start = time.time()
	tic = lambda: 'at %1.1f seconds' % (time.time() - start)

	def gr1():
	    print('Started Polling: %s' % tic())
	    select.select([], [], [], 1)
	    print('Ended Polling: %s' % tic())

	def gr2():
	    print('Started Polling: %s' % tic())
	    select.select([], [], [], 2)
	    print('Ended Polling: %s' % tic())

	def gr3():
	    print("Hey lets do some stuff while the greenlets poll, %s" % tic())
	    gevent.sleep(1)

	gevent.joinall([
	    gevent.spawn(gr1),
	    gevent.spawn(gr2),
	    gevent.spawn(gr3),
	])

执行结果：
	
	Started Polling: at 0.0 seconds
	Started Polling: at 0.0 seconds
	Hey lets do some stuff while the greenlets poll, at 0.0 seconds
	Ended Polling: at 1.0 seconds
	Ended Polling: at 2.0 seconds

同步vs异步

	import gevent
	import random

	def task(pid):
	    gevent.sleep(random.randint(0,2)*0.001)
	    print('Task %s done' % pid)

	def synchronous():
	    for i in xrange(5):
	        task(i)

	def asynchronous():
	    threads = [gevent.spawn(task, i) for i in xrange(5)]
	    gevent.joinall(threads)

	print('Synchronous:')
	synchronous()

	print('Asynchronous:')
	asynchronous()

执行结果：
	
	Synchronous:
	Task 0 done
	Task 1 done
	Task 2 done
	Task 3 done
	Task 4 done
	Asynchronous:
	Task 2 done
	Task 0 done
	Task 1 done
	Task 3 done
	Task 4 done

####确定性
greenlet具有确定性。在相同配置相同输入的情况下，它们总是会产生相同的输出。


	import time

	def echo(i):
	    time.sleep(0.001)
	    return i

	# Non Deterministic Process Pool

	from multiprocessing.pool import Pool

	p = Pool(10)
	run1 = [a for a in p.imap_unordered(echo, xrange(10))]
	run2 = [a for a in p.imap_unordered(echo, xrange(10))]
	run3 = [a for a in p.imap_unordered(echo, xrange(10))]
	run4 = [a for a in p.imap_unordered(echo, xrange(10))]

	print(run1 == run2 == run3 == run4)

	# Deterministic Gevent Pool

	from gevent.pool import Pool

	p = Pool(10)
	run1 = [a for a in p.imap_unordered(echo, xrange(10))]
	run2 = [a for a in p.imap_unordered(echo, xrange(10))]
	run3 = [a for a in p.imap_unordered(echo, xrange(10))]
	run4 = [a for a in p.imap_unordered(echo, xrange(10))]

	print(run1 == run2 == run3 == run4)

执行结果：
	
	False
	True

即使gevent通常带有确定性，当开始与如socket或文件等外部服务交互时， 不确定性也可能溜进你的程序中。因此尽管gevent线程是一种“确定的并发”形式， 使用它仍然可能会遇到像使用POSIX线程或进程时遇到的那些问题。

涉及并发长期存在的问题就是竞争条件(race condition)(当两个并发线程/进程都依赖于某个共享资源同时都尝试去修改它的时候， 就会出现竞争条件),这会导致资源修改的结果状态依赖于时间和执行顺序。 这个问题，会导致整个程序行为变得不确定。

解决办法: 始终避免所有全局的状态.

####创建Greenlets
gevent对Greenlet初始化提供了一些封装.

	import gevent
	from gevent import Greenlet

	def foo(message, n):
	    gevent.sleep(n)
	    print(message)

	thread1 = Greenlet.spawn(foo, "Hello", 1)
	thread2 = gevent.spawn(foo, "I live!", 2)
	thread3 = gevent.spawn(lambda x: (x+1), 2)
	threads = [thread1, thread2, thread3]
	gevent.joinall(threads)

执行结果：

	Hello
	I live!
除使用基本的Greenlet类之外，你也可以子类化Greenlet类，重载它的_run方法。

	import gevent
	from gevent import Greenlet

	class MyGreenlet(Greenlet):

	    def __init__(self, message, n):
	        Greenlet.__init__(self)
	        self.message = message
	        self.n = n

	    def _run(self):
	        print(self.message)
	        gevent.sleep(self.n)

	g = MyGreenlet("Hi there!", 3)
	g.start()
	g.join()

执行结果：
		
	Hi there!
####Greenlet状态
greenlet的状态通常是一个依赖于时间的参数：
* started -- Boolean, 指示此Greenlet是否已经启动
* ready() -- Boolean, 指示此Greenlet是否已经停止
* successful() -- Boolean, 指示此Greenlet是否已经停止而且没抛异常
* value -- 任意值, 此Greenlet代码返回的值
* exception -- 异常, 此Greenlet内抛出的未捕获异常
	
####程序停止
程序
当主程序(main program)收到一个SIGQUIT信号时，不能成功做yield操作的 Greenlet可能会令意外地挂起程序的执行。这导致了所谓的僵尸进程， 它需要在Python解释器之外被kill掉。

通用的处理模式就是在主程序中监听SIGQUIT信号，调用gevent.shutdown退出程序。

	import gevent
	import signal

	def run_forever():
	    gevent.sleep(1000)

	if __name__ == '__main__':
	    gevent.signal(signal.SIGQUIT, gevent.shutdown)
	    thread = gevent.spawn(run_forever)
	    thread.join()

####超时
通过超时可以对代码块儿或一个Greenlet的运行时间进行约束。


	import gevent
	from gevent import Timeout

	seconds = 10

	timeout = Timeout(seconds)
	timeout.start()

	def wait():
	    gevent.sleep(10)

	try:
	    gevent.spawn(wait).join()
	except Timeout:
	    print('Could not complete')

超时类

	import gevent
	from gevent import Timeout

	time_to_wait = 5 # seconds

	class TooLong(Exception):
	    pass

	with Timeout(time_to_wait, TooLong):
	    gevent.sleep(10)

另外，对各种Greenlet和数据结构相关的调用，gevent也提供了超时参数。


	import gevent
	from gevent import Timeout

	def wait():
	    gevent.sleep(2)

	timer = Timeout(1).start()
	thread1 = gevent.spawn(wait)

	try:
	    thread1.join(timeout=timer)
	except Timeout:
	    print('Thread 1 timed out')

	# --

	timer = Timeout.start_new(1)
	thread2 = gevent.spawn(wait)

	try:
	    thread2.get(timeout=timer)
	except Timeout:
	    print('Thread 2 timed out')

	# --

	try:
	    gevent.with_timeout(1, wait)
	except Timeout:
	    print('Thread 3 timed out')

执行结果：

	Thread 1 timed out
	Thread 2 timed out
	Thread 3 timed out
####猴子补丁(Monkey patching)
gevent的死角.

	import socket
	print(socket.socket)

	print("After monkey patch")
	from gevent import monkey
	monkey.patch_socket()
	print(socket.socket)

	import select
	print(select.select)
	monkey.patch_select()
	print("After monkey patch")
	print(select.select)

执行结果：

	class 'socket.socket'
	After monkey patch
	class 'gevent.socket.socket'

	built-in function select
	After monkey patch
	function select at 0x1924de8

Python的运行环境允许我们在运行时修改大部分的对象，包括模块，类甚至函数。 这是个一般说来令人惊奇的坏主意，因为它创造了“隐式的副作用”，如果出现问题 它很多时候是极难调试的。虽然如此，在极端情况下当一个库需要修改Python本身 的基础行为的时候，猴子补丁就派上用场了。在这种情况下，gevent能够修改标准库里面大部分的阻塞式系统调用，包括socket、ssl、threading和 select等模块，而变为协作式运行。

例如，Redis的python绑定一般使用常规的tcp socket来与redis-server实例通信。 通过简单地调用gevent.monkey.patch_all()，可以使得redis的绑定协作式的调度 请求，与gevent栈的其它部分一起工作。

这让我们可以将一般不能与gevent共同工作的库结合起来，而不用写哪怕一行代码。 虽然猴子补丁仍然是邪恶的(evil)，但在这种情况下它是“有用的邪恶(useful evil)”。

### 数据结构
* 事件
* 队列
* 组和池
* 锁和信号量
* 线程局部变量
* 子进程
* Actors
#### 事件
事件(event)是一个在Greenlet之间异步通信的形式。

	import gevent
	from gevent.event import Event
	
	evt = Event()
	
	def setter():
	    print('A: Hey wait for me, I have to do something')
	    gevent.sleep(3)
	    print("Ok, I'm done")
	    evt.set()

	def waiter():
	    print("I'll wait for you")
	    evt.wait()  # blocking
	    print("It's about time")

	def main():
	    gevent.joinall([
	        gevent.spawn(setter),
	        gevent.spawn(waiter),
	        gevent.spawn(waiter),
	        gevent.spawn(waiter)
	    ])

	if __name__ == '__main__': 
		main()

执行结果：

	A: Hey wait for me, I have to do something
	I'll wait for you
	I'll wait for you
	I'll wait for you
	Ok, I'm done
	It's about time
	It's about time
	It's about time

事件对象的一个扩展是AsyncResult，它允许你在唤醒调用上附加一个值。 它有时也被称作是future或defered，因为它持有一个指向将来任意时间可设置为任何值的引用。

	import gevent
	from gevent.event import AsyncResult
	a = AsyncResult()

	def setter():
	    gevent.sleep(3)
	    a.set('Hello!')

	def waiter():
	    print(a.get())

	gevent.joinall([
	    gevent.spawn(setter),
	    gevent.spawn(waiter),
	])

#### 队列
队列是一个排序的数据集合，它有常见的put / get操作， 但是它是以在Greenlet之间可以安全操作的方式来实现的。

	import gevent
	from gevent.queue import Queue

	tasks = Queue()

	def worker(n):
	    while not tasks.empty():
	        task = tasks.get()
	        print('Worker %s got task %s' % (n, task))
	        gevent.sleep(0)
	    print('Quitting time!')

	def boss():
	    for i in xrange(1,10):
	        tasks.put_nowait(i)

	gevent.spawn(boss).join()

	gevent.joinall([
	    gevent.spawn(worker, 'steve'),
	    gevent.spawn(worker, 'john'),
	    gevent.spawn(worker, 'nancy'),
	])

执行结果：
	
	Worker steve got task 1
	Worker john got task 2
	Worker nancy got task 3
	Worker steve got task 4
	Worker john got task 5
	Worker nancy got task 6
	Worker steve got task 7
	Worker john got task 8
	Worker nancy got task 9
	Quitting time!
	Quitting time!
	Quitting time!

put和get操作都是阻塞的，put_nowait和get_nowait不会阻塞， 然而在操作不能完成时抛出gevent.queue.Empty或gevent.queue.Full异常。

####组和池
组(group)是一个运行中greenlet集合，集合中的greenlet像一个组一样会被共同管理和调度。 它也兼饰了像Python的multiprocessing库那样的平行调度器的角色，主要用在在管理异步任务的时候进行分组。


	import gevent
	from gevent.pool import Group

	def talk(msg):
	    for i in xrange(2):
	        print(msg)

	g1 = gevent.spawn(talk, 'bar')
	g2 = gevent.spawn(talk, 'foo')
	g3 = gevent.spawn(talk, 'fizz')

	group = Group()
	group.add(g1)
	group.add(g2)
	group.join()

	group.add(g3)
	group.join()

执行结果：
	
	bar
	bar
	foo
	foo
	fizz
	fizz

池(pool)是一个为处理数量变化并且需要限制并发的greenlet而设计的结构。

	import gevent
	from gevent.pool import Pool

	pool = Pool(2)

	def hello_from(n):
	    print('Size of pool %s' % len(pool))

	pool.map(hello_from, xrange(3))

执行结果：

	Size of pool 2
	Size of pool 2
	Size of pool 1
构造一个socket池的类，在各个socket上轮询。

	from gevent.pool import Pool

	class SocketPool(object):

	    def __init__(self):
	        self.pool = Pool(10)
	        self.pool.start()

	    def listen(self, socket):
	        while True:
	            socket.recv()

	    def add_handler(self, socket):
	        if self.pool.full():
	            raise Exception("At maximum pool size")
	        else:
	            self.pool.spawn(self.listen, socket)

	    def shutdown(self):
	        self.pool.kill()


####锁和信号量
信号量是一个允许greenlet相互合作，限制并发访问或运行的低层次的同步原语。 信号量有两个方法，acquire和release。在信号量是否已经被 acquire或release，和拥有资源的数量之间不同，被称为此信号量的范围 (the bound of the semaphore)。如果一个信号量的范围已经降低到0，它会 阻塞acquire操作直到另一个已经获得信号量的greenlet作出释放。


	from gevent import sleep
	from gevent.pool import Pool
	from gevent.coros import BoundedSemaphore

	sem = BoundedSemaphore(2)

	def worker1(n):
	    sem.acquire()
	    print('Worker %i acquired semaphore' % n)
	    sleep(0)
	    sem.release()
	    print('Worker %i released semaphore' % n)

	def worker2(n):
	    with sem:
	        print('Worker %i acquired semaphore' % n)
	        sleep(0)
	    print('Worker %i released semaphore' % n)

	pool = Pool()
	pool.map(worker1, xrange(0,2))

执行结果：

	Worker 0 acquired semaphore
	Worker 1 acquired semaphore
	Worker 0 released semaphore
	Worker 1 released semaphore
锁(lock)是范围为1的信号量。它向单个greenlet提供了互斥访问。 信号量和锁常被用来保证资源只在程序上下文被单次使用。

####线程局部变量
Gevent允许程序员指定局部于greenlet上下文的数据。 在内部，它被实现为以greenlet的getcurrent()为键， 在一个私有命名空间寻址的全局查找。

	import gevent
	from gevent.local import local

	stash = local()

	def f1():
	    stash.x = 1
	    print(stash.x)

	def f2():
	    stash.y = 2
	    print(stash.y)

	    try:
	        stash.x
	    except AttributeError:
	        print("x is not local to f2")

	g1 = gevent.spawn(f1)
	g2 = gevent.spawn(f2)

	gevent.joinall([g1, g2])

执行结果：
	
	1
	2
	x is not local to f2
很多集成了gevent的web框架将HTTP会话对象以线程局部变量的方式存储在gevent内。 例如使用Werkzeug实用库和它的proxy对象，我们可以创建Flask风格的请求对象。

	from gevent.local import local
	from werkzeug.local import LocalProxy
	from werkzeug.wrappers import Request
	from contextlib import contextmanager

	from gevent.wsgi import WSGIServer

	_requests = local()
	request = LocalProxy(lambda: _requests.request)

	@contextmanager
	def sessionmanager(environ):
	    _requests.request = Request(environ)
	    yield
	    _requests.request = None

	def logic():
	    return "Hello " + request.remote_addr

	def application(environ, start_response):
	    status = '200 OK'

	    with sessionmanager(environ):
	        body = logic()

	    headers = [
	        ('Content-Type', 'text/html')
	    ]

	    start_response(status, headers)
	    return [body]

	WSGIServer(('', 8000), application).serve_forever()

####子进程
从gevent 1.0起，支持gevent.subprocess，支持协作式的等待子进程。

	import gevent
	from gevent.subprocess import Popen, PIPE

	def cron():
	    while True:
	        print("cron")
	        gevent.sleep(0.2)

	g = gevent.spawn(cron)
	sub = Popen(['sleep 1; uname'], stdout=PIPE, shell=True)
	out, err = sub.communicate()
	g.kill()
	print(out.rstrip())
 
 执行结果：
 
	cron
	cron
	cron
	cron
	cron
	Linux

很多人也想将gevent和multiprocessing一起使用。最明显的挑战之一 就是multiprocessing提供的进程间通信默认不是协作式的。由于基于 multiprocessing.Connection的对象(例如Pipe)暴露了它们下面的 文件描述符(file descriptor)，gevent.socket.wait_read和wait_write 可以用来在直接读写之前协作式的等待ready-to-read/ready-to-write事件。


	import gevent
	from multiprocessing import Process, Pipe
	from gevent.socket import wait_read, wait_write

	# To Process
	a, b = Pipe()

	# From Process
	c, d = Pipe()

	def relay():
	    for i in xrange(5):
	        msg = b.recv()
	        c.send(msg + " in " + str(i))

	def put_msg():
	    for i in xrange(5):
	        wait_write(a.fileno())
	        a.send('hi')

	def get_msg():
	    for i in xrange(5):
	        wait_read(d.fileno())
	        print(d.recv())

	if __name__ == '__main__':
	    proc = Process(target=relay)
	    proc.start()

	    g1 = gevent.spawn(get_msg)
	    g2 = gevent.spawn(put_msg)
	    gevent.joinall([g1, g2], timeout=1)
	  
执行结果：
	
	hi in 0
	hi in 1
	hi in 2
	hi in 3
	hi in 4

然而要注意，组合multiprocessing和gevent必定带来 依赖于操作系统(os-dependent)的缺陷，其中有：

在兼容POSIX的系统创建子进程(forking)之后， 在子进程的gevent的状态是不适定的(ill-posed)。一个副作用就是， multiprocessing.Process创建之前的greenlet创建动作，会在父进程和子进程两方都运行。

上例的put_msg()中的a.send()可能依然非协作式地阻塞调用的线程：一个 ready-to-write事件只保证写了一个byte。在尝试写完成之前底下的buffer可能是满的。

上面表示的基于wait_write()/wait_read()的方法在Windows上不工作 (IOError: 3 is not a socket (files are not supported))，因为Windows不能监视 pipe事件。

Python包gipc以大体上透明的方式在 兼容POSIX系统和Windows上克服了这些挑战。它提供了gevent感知的基于 multiprocessing.Process的子进程和gevent基于pipe的协作式进程间通信。

####Actors
actor模型是一个由于Erlang变得普及的更高层的并发模型。 简单的说它的主要思想就是许多个独立的Actor，每个Actor有一个可以从 其它Actor接收消息的收件箱。Actor内部的主循环遍历它收到的消息，并根据它期望的行为来采取行动。

Gevent没有原生的Actor类型，但在一个子类化的Greenlet内使用队列， 我们可以定义一个非常简单的。

	import gevent
	from gevent.queue import Queue

	class Actor(gevent.Greenlet):

	    def __init__(self):
	        self.inbox = Queue()
	        Greenlet.__init__(self)

	    def receive(self, message):
	        """
	        Define in your subclass.
	        """
	        raise NotImplemented()

	    def _run(self):
	        self.running = True

	        while self.running:
	            message = self.inbox.get()
	            self.receive(message)


下面是一个使用的例子：

	import gevent
	from gevent.queue import Queue
	from gevent import Greenlet

	class Pinger(Actor):
	    def receive(self, message):
	        print(message)
	        pong.inbox.put('ping')
	        gevent.sleep(0)

	class Ponger(Actor):
	    def receive(self, message):
	        print(message)
	        ping.inbox.put('pong')
	        gevent.sleep(0)

	ping = Pinger()
	pong = Ponger()

	ping.start()
	pong.start()

	ping.inbox.put('start')
	gevent.joinall([ping, pong])

###实际应用

* Gevent ZeroMQ
* 简单server
* WSGI Servers
* 流式server
* Long Polling
* Websockets

##简单server

    # On Unix: Access with ``$ nc 127.0.0.1 5000``
    # On Window: Access with ``$ telnet 127.0.0.1 5000``

    from gevent.server import StreamServer

    def handle(socket, address):
        socket.send("Hello from a telnet!\n")
        for i in range(5):
            socket.send(str(i) + '\n')
        socket.close()

    server = StreamServer(('127.0.0.1', 5000), handle)
    server.serve_forever()

##WSGI Servers And Websockets
Gevent为HTTP内容服务提供了两种WSGI server。从今以后就称为 wsgi和pywsgi：

* gevent.wsgi.WSGIServer
* gevent.pywsgi.WSGIServer

glb中使用
	
	import click
	from flask import Flask
	from gevent.pywsgi import WSGIServer
	from geventwebsocket.handler import WebSocketHandler

	import v1
	from .settings import Config
	from .sockethandler import handle_websocket


	def create_app(config=None):
	    app = Flask(__name__, static_folder='static')
	    if config:
	        app.config.update(config)
	    else:
	        app.config.from_object(Config)

	    app.register_blueprint(
	        v1.bp,
	        url_prefix='/v1')
	    return app


	def wsgi_app(environ, start_response):
	    path = environ['PATH_INFO']
	    if path == '/websocket':
	        handle_websocket(environ['wsgi.websocket'])
	    else:
	        return create_app()(environ, start_response)


	@click.command()
	@click.option('-h', '--host_port', type=(unicode, int),
	              default=('0.0.0.0', 5000), help='Host and port of server.')
	@click.option('-r', '--redis', type=(unicode, int, int),
	              default=('127.0.0.1', 6379, 0),
	              help='Redis url of server.')
	@click.option('-p', '--port_range', type=(int, int),
	              default=(50000, 61000),
	              help='Port range to be assigned.')
	def manage(host_port, redis=None, port_range=None):
	    Config.REDIS_URL = 'redis://%s:%s/%s' % redis
	    Config.PORT_RANGE = port_range
	    http_server = WSGIServer(host_port,
	                             wsgi_app, handler_class=WebSocketHandler)
	    print '----GLB Server run at %s:%s-----' % host_port
	    print '----Redis Server run at %s:%s:%s-----' % redis
	    http_server.serve_forever()

  
## 缺陷

和其他异步I/O框架一样,gevent也有一些缺陷:

* 阻塞(真正的阻塞,在内核级别)在程序中的某个地方停止了所有的东西.这很像C代码中monkey patch没有生效
* 保持CPU处于繁忙状态.greenlet不是抢占式的,这可能导致其他greenlet不会被调度.
* 在greenlet之间存在死锁的可能.

一个gevent回避的缺陷是,你几乎不会碰到一个和异步无关的Python库--它将阻塞你的应用程序,因为纯Python库使用的是monkey patch的stdlib.





