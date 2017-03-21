title标题: A Web Crawler With asyncio Coroutines
author作者: A. Jesse Jiryu Davis and Guido van Rossum
<markdown>
_A. Jesse Jiryu Davis is a staff engineer at MongoDB in New York. He wrote Motor, the async MongoDB Python driver, and he is the lead developer of the MongoDB C Driver and a member of the PyMongo team. He contributes to asyncio and Tornado. He writes at [http://emptysqua.re](http://emptysqua.re).
A. Jesse Jiryu Davis在纽约为MongoDB工作。他编写了Motor，异步MongoDB Python驱动器，他也是MongoDB C驱动器的首席开发者， 同时他也是PyMango组织的成员之一。他对asyncio和Tornado同样有着杰出贡献。他的博客是 [http://emptysqua.re](http://emptysqua.re/)。

Guido van Rossum is the creator of Python, one of the major programming languages on and off the web. The Python community refers to him as the BDFL (Benevolent Dictator For Life), a title straight from a Monty Python skit.  Guido's home on the web is [http://www.python.org/~guido/](http://www.python.org/~guido/).
Guido van Rossum，Python之父，Python是目前主要的编程语言之一，无论线上线下。 他在社区里一直是一位仁慈的独裁者，一个来自Monty Python短剧的标题。Guido网址是[http://www.python.org/~guido/](http://www.python.org/~guido/) .
</markdown>
## Introduction 介绍

Classical computer science emphasizes efficient algorithms that complete computations as quickly as possible. But many networked programs spend their time not computing, but holding open many connections that are slow, or have infrequent events. These programs present a very different challenge: to wait for a huge number of network events efficiently. A contemporary approach to this problem is asynchronous I/O, or "async".
经典计算机科学看重高效的算法以便能尽快完成计算。但是许多网络程序消耗的时间不是在计算上，它们通常维持着许多打开的缓慢的连接，或者期待着一些不频繁发生的事件发生。这些程序代表了另一个不同的挑战：如何高效的监听大量网络事件。解决这个问题的一个现代方法是采用异步I/O.

This chapter presents a simple web crawler. The crawler is an archetypal async application because it waits for many responses, but does little computation. The more pages it can fetch at once, the sooner it completes. If it devotes a thread to each in-flight request, then as the number of concurrent requests rises it will run out of memory or other thread-related resource before it runs out of sockets. It avoids the need for threads by using asynchronous I/O.
这一章节实现了一个简单的网络爬虫。这个爬虫是一个异步调用的原型应用程序，因为它需要等待许多响应，而极少有CPU计算。它每次可以抓取的页面越多，它运行结束的时间越快。 如果它为每一个运行的请求分发一个线程，那么随着并发请求数量的增加，它最终会在耗尽系统套接字之前，耗尽内存或者其他线程相关的资源。 它通过使用异步I/O来避免对大量线程依赖。
We present the example in three stages. First, we show an async event loop and sketch a crawler that uses the event loop with callbacks: it is very efficient, but extending it to more complex problems would lead to unmanageable spaghetti code. Second, therefore, we show that Python coroutines are both efficient and extensible. We implement simple coroutines in Python using generator functions. In the third stage, we use the full-featured coroutines from Python's standard "asyncio" library[^16], and coordinate them using an async queue.
我们通过三步来实现这个例子。首先，我们展示一个异步的事件循环，并且完成一个带有回掉函数并且使用这个循环的爬虫：它非常的高效，但是当我们想扩展它来适应更复杂的问题时会带来很多难以处理的代码。因此，接下来我们展示一个即高效又容易扩展的python的协程的程序。第三步，我们使用python标准库中的“asyncio”库中的全功能的协程程序，然后通过async异步队列来组合他们。
## The Task

A web crawler finds and downloads all pages on a website, perhaps to archive or index them. Beginning with a root URL, it fetches each page, parses it for links to unseen pages, and adds these to a queue. It stops when it fetches a page with no unseen links and the queue is empty.
一个网络爬虫会寻找并且下载一个网站上的所有页面，可能会存档或者对他们建立索引。从一个根节点开始，它爬取每一个页面，解析页面并且寻找从未访问过的链接，然后把他们加入到队列中。当解析到一个没有从未访问过的链接的页面并且队列是空的时候，爬虫会停止。
We can hasten this process by downloading many pages concurrently. As the crawler finds new links, it launches simultaneous fetch operations for the new pages on separate sockets. It parses responses as they arrive, adding new links to the queue. There may come some point of diminishing returns where too much concurrency degrades performance, so we cap the number of concurrent requests, and leave the remaining links in the queue until some in-flight requests complete.
我们可以通过同时下载许多页面来加快这个过程。当爬虫发现新的链接时，它在单独的sockets上同时启动抓取新页面的操作。当抓取结果抵达时，它开始解析响应，并往队列里添加新解析到的链接。 大量的并发请求可能导致一些性能降低，因而我们限制同一时间内请求的数量，把其他的链接加入队列直到一些运行中的请求完成。
## The Traditional Approach 传统的实现方法

How do we make the crawler concurrent? Traditionally we would create a thread pool. Each thread would be in charge of downloading one page at a time over a socket. For example, to download a page from `xkcd.com`:
我们该如何让爬虫并发处理请求呢？传统方法是建立一个线程池。每个进程每次将负责通过一个socket下载一个页面。比如，下载“xkcd.com”的一个页面：
```python
def fetch(url):
    sock = socket.socket()
    sock.connect(('xkcd.com', 80))
    request = 'GET {} HTTP/1.0\r\nHost: xkcd.com\r\n\r\n'.format(url)
    sock.send(request.encode('ascii'))
    response = b''
    chunk = sock.recv(4096)
    while chunk:
        response += chunk
        chunk = sock.recv(4096)
    
    # Page is now downloaded.
    links = parse_links(response)
    q.add(links)
```

By default, socket operations are *blocking*: when the thread calls a method like `connect` or `recv`, it pauses until the operation completes.[^15] Consequently to download many pages at once, we need many threads. A sophisticated application amortizes the cost of thread-creation by keeping idle threads in a thread pool, then checking them out to reuse them for subsequent tasks; it does the same with sockets in a connection pool.
默认情况下，socket操作是阻塞的：当一个线程调用像connect或者recv之类Socket相关的方法时，它会被阻塞直至操作完成。 因此，一次性并行下载很多页面，我们得需要更多的线程。一个复杂点的程序通过将线程池中的空闲线程保持在线程池中，然后将他们检出以重用他们来用于后续任务中分摊线程创建的成本，它对连接池中的套接字执行相同的操作。
And yet, threads are expensive, and operating systems enforce a variety of hard caps on the number of threads a process, user, or machine may have. On Jesse's system, a Python thread costs around 50k of memory, and starting tens of thousands of threads causes failures. If we scale up to tens of thousands of simultaneous operations on concurrent sockets, we run out of threads before we run out of sockets. Per-thread overhead or system limits on threads are the bottleneck.
然而，线程的开销是很昂贵的，并且操作系统对进程，线程的数量进行各种限制。在Jesse的系统上，Python线程需要大约50k的内存，并且启动数以万计的线程会导致失败。 如果我们在并发socket上扩展到数万个并发操作，我们就耗尽了线程，然后我们用完了socket。 每个线程的开销或系统对线程的限制是瓶颈。
In his influential article "The C10K problem"[^8], Dan Kegel outlines the limitations of multithreading for I/O concurrency. He begins,
在他那篇颇有影响力的文章《The C10K problem》中，Dan Kegel概述了用多线程并行处理I/O问题的局限性。
> It's time for web servers to handle ten thousand clients simultaneously, don't you think? After all, the web is a big place now.是时候让web服务器同时处理数万客户端请求了，不是吗？毕竟，web那么大。

Kegel coined the term "C10K" in 1999. Ten thousand connections sounds dainty now, but the problem has changed only in size, not in kind. Back then, using a thread per connection for C10K was impractical. Now the cap is orders of magnitude higher. Indeed, our toy web crawler would work just fine with threads. Yet for very large scale applications, with hundreds of thousands of connections, the cap remains: there is a limit beyond which most systems can still create sockets, but have run out of threads. How can we overcome this?
Kegel在1999年发明了“C10K”这个词。一万连接现在听起来觉得很少，但问题的关键点在于连接的数量而不在于类型。回到那个年代，一个连接使用一个线程来处理C10K问题是不实际的。现在容量已经是当初的好几个数量级了。说实话，我们的爬虫小玩具使用线程的方式也能运行的很好。但对于需要面对成百上千连接的大规模应用程序来说，使用线程的缺陷还是依旧在这儿：大部分操作系统还能创建Socket，但是不能再继续创建线程了。我们如何克服这个难题呢？

## Async 异步

Asynchronous I/O frameworks do concurrent operations on a single thread using
*non-blocking* sockets. In our async crawler, we set the socket non-blocking
before we begin to connect to the server:
异步I / O框架使用*非阻塞*socket在单个线程上执行并行操作。 在我们的异步爬虫中，我们在开始连接到服务器之前设置socket无阻塞：
```python
sock = socket.socket()
sock.setblocking(False)
try:
    sock.connect(('xkcd.com', 80))
except BlockingIOError:
    pass
```

Irritatingly, a non-blocking socket throws an exception from `connect`, even when it is working normally. This exception replicates the irritating behavior of the underlying C function, which sets `errno` to `EINPROGRESS` to tell you it has begun.
非阻塞套接字从`connect`抛出一个异常，即使它正常工作。 这个异常复制了底层C函数的行为，它将`errno`设置为`EINPROGRESS`来告诉你它已经开始。
Now our crawler needs a way to know when the connection is established, so it can send the HTTP request. We could simply keep trying in a tight loop:
现在我们的爬虫需要一种方法来知道连接何时建立，然后它可以发送HTTP请求。 我们可以简单地在一个循环中尝试：
```python
request = 'GET {} HTTP/1.0\r\nHost: xkcd.com\r\n\r\n'.format(url)
encoded = request.encode('ascii')

while True:
    try:
        sock.send(encoded)
        break  # Done.
    except OSError as e:
        pass

print('sent')
```

This method not only wastes electricity, but it cannot efficiently await events on *multiple* sockets. In ancient times, BSD Unix's solution to this problem was `select`, a C function that waits for an event to occur on a non-blocking socket or a small array of them. Nowadays the demand for Internet applications with huge numbers of connections has led to replacements like `poll`, then `kqueue` on BSD and `epoll` on Linux. These APIs are similar to `select`, but perform well with very large numbers of connections.
这种方法不仅浪费电力，而且不能有效地等待*多个socket上的事件。 在过去，BSD Unix的这个问题的解决方案是`select`，一个C函数，等待一个事件发生在一个非阻塞的套接字或者一个小的数组。 如今，对具有大量连接的互联网应用的需求导致了诸如`poll'，然后是BSD上的`kqueue'和Linux上的`epoll'的替换。 这些API类似于“select”，但是对于非常大量的连接执行得很好。
Python 3.4's `DefaultSelector` uses the best `select`-like function available on your system. To register for notifications about network I/O, we create a non-blocking socket and register it with the default selector:
Python 3.4的“DefaultSelector”使用你的系统上最好的类似select的函数。 为了注册有关网络I / O的通知，我们创建一个非阻塞socket并使用默认选择器注册它：
```python
from selectors import DefaultSelector, EVENT_WRITE

selector = DefaultSelector()

sock = socket.socket()
sock.setblocking(False)
try:
    sock.connect(('xkcd.com', 80))
except BlockingIOError:
    pass

def connected():
    selector.unregister(sock.fileno())
    print('connected!')

selector.register(sock.fileno(), EVENT_WRITE, connected)
```

We disregard the spurious error and call `selector.register`, passing in the socket's file descriptor and a constant that expresses what event we are waiting for. To be notified when the connection is established, we pass `EVENT_WRITE`: that is, we want to know when the socket is "writable". We also pass a Python function, `connected`, to run when that event occurs. Such a function is known as a *callback*.
我们忽略了错误并调用了selector.register，传递了socket的文件描述符和一个表示我们正在等待什么事件的常量。 要在连接建立时获得通知，我们传递`EVENT_WRITE`：也就是说，我们想知道套接字何时是可写的。 我们还传递一个Python函数`connected`，当事件发生时运行。 这样的函数称为*回调*。
We process I/O notifications as the selector receives them, in a loop:
当选择器接收到它们时，我们在一个循环中处理I / O通知：
```python
def loop():
    while True:
        events = selector.select()
        for event_key, event_mask in events:
            callback = event_key.data
            callback()
```

The `connected` callback is stored as `event_key.data`, which we retrieve and execute once the non-blocking socket is connected.
`connected`回调被存储为`event_key.data`，我们在非阻塞socket连接时取回并执行。

Unlike in our fast-spinning loop above, the call to `select` here pauses, awaiting the next I/O events. Then the loop runs callbacks that are waiting for these events. Operations that have not completed remain pending until some future tick of the event loop.
与上面的快速循环不同，这里`select`调用暂停，等待下一个I / O事件。 然后循环运行等待这些事件的回调。 尚未完成的操作将保持挂起，直到事件循环的某个未来时间点为止
What have we demonstrated already? We showed how to begin an operation and execute a callback when the operation is ready. An async *framework* builds on the two features we have shown&mdash;non-blocking sockets and the event loop&mdash;to run concurrent operations on a single thread.
我们已经证明了什么？ 我们展示了当操作准备好时如何开始操作和执行回调。 异步框架建立在我们所展示的两个特性（非阻塞socket和事件循环）上，以在单个线程上运行并发操作。
We have achieved "concurrency" here, but not what is traditionally called "parallelism". That is, we built a tiny system that does overlapping I/O. It is capable of beginning new operations while others are in flight. It does not actually utilize multiple cores to execute computation in parallel. But then, this system is designed for I/O-bound problems, not CPU-bound ones.[^14]
我们在这里实现了“并发”，但不是传统上被称为“并行性”。 也就是说，我们构建了一个重叠I / O的小系统。 它能够开始新的操作，而其他人在飞行。 它实际上并不利用多个核来并行执行计算。 但是，这个系统是为I / O绑定的问题设计的，而不是CPU绑定的。[^ 14]
So our event loop is efficient at concurrent I/O because it does not devote thread resources to each connection. But before we proceed, it is important to correct a common misapprehension that async is *faster* than multithreading. Often it is not&mdash;indeed, in Python, an event loop like ours is moderately slower than multithreading at serving a small number of very active connections. In a runtime without a global interpreter lock, threads would perform even better on such a workload. What asynchronous I/O is right for, is applications with many slow or sleepy connections with infrequent events.[^11]<latex>[^bayer]</latex>
因此，我们的事件循环在并发I / O方面是高效的，因为它不会将线程资源分配给每个连接。 但在我们继续前，重要的是纠正一个常见的误解，即异步的速度比多线程快。 通常不是，事实上，在Python中，像我们这样的事件循环比服务少且非常活跃的连接的多线程慢。 在没有全局解释器锁的运行时，线程在这样的工作负载上表现更好。 什么时候异步I / O是正确的，是与许多慢或困连接与罕见的事件的应用程序。[^ 11]<latex>[^bayer]</latex>
## Programming With Callbacks 回调

With the runty async framework we have built so far, how can we build a web crawler? Even a simple URL-fetcher is painful to write.
随着我们构建的runty异步框架到目前为止，我们如何构建一个网络爬虫？ 即使一个简单的URL爬虫写起来也是痛苦的。
We begin with global sets of the URLs we have yet to fetch, and the URLs we have seen:
我们从全局的还没有抓取URL集合和我们看到的URL开始：
```python
urls_todo = set(['/'])
seen_urls = set(['/'])
```

The `seen_urls` set includes `urls_todo` plus completed URLs. The two sets are initialized with the root URL "/".
`seen_urls`集合包括`urls_todo`和完成的URL。 这两个集合由根URL“/”初始化。

Fetching a page will require a series of callbacks. The `connected` callback fires when a socket is connected, and sends a GET request to the server. But then it must await a response, so it registers another callback. If, when that callback fires, it cannot read the full response yet, it registers again, and so on.
获取页面将需要一系列回调。 当连接socket时，连接回调触发，并向服务器发送GET请求。 但是它必须等待响应，所以它注册另一个回调。 如果，当回调触发时，它不能读取完整的响应，它再次注册，等等。
Let us collect these callbacks into a `Fetcher` object. It needs a URL, a socket object, and a place to accumulate the response bytes:
让我们将这些回调收集到一个`Fetcher`对象中。 它需要一个URL，一个socket对象和一个地方来累积response bytes：
```python
class Fetcher:
    def __init__(self, url):
        self.response = b''  # Empty array of bytes.
        self.url = url
        self.sock = None
```

We begin by calling `Fetcher.fetch`:

```python
    # Method on Fetcher class.
    def fetch(self):
        self.sock = socket.socket()
        self.sock.setblocking(False)
        try:
            self.sock.connect(('xkcd.com', 80))
        except BlockingIOError:
            pass
            
        # Register next callback.
        selector.register(self.sock.fileno(),
                          EVENT_WRITE,
                          self.connected)
```

The `fetch` method begins connecting a socket. But notice the method returns before the connection is established. It must return control to the event loop to wait for the connection. To understand why, imagine our whole application was structured so:
`fetch`方法开始连接socket。 但请注意，该方法在建立连接之前返回。 它必须将控制权返回到事件循环，以等待连接。 为了理解为什么，想象我们的整个应用程序的结构如下：
```python
# Begin fetching http://xkcd.com/353/
fetcher = Fetcher('/353/')
fetcher.fetch()

while True:
    events = selector.select()
    for event_key, event_mask in events:
        callback = event_key.data
        callback(event_key, event_mask)
```

All event notifications are processed in the event loop when it calls `select`. Hence `fetch` must hand control to the event loop, so that the program knows when the socket has connected. Only then does the loop run the `connected` callback, which was registered at the end of `fetch` above.
当调用`select`时，所有事件通知都在事件循环中处理。 因此，“fetch”必须手动控制事件循环，以便程序知道套接字何时连接。 只有这样，循环才会运行`connected`回调，它在上面的`fetch`结尾处注册。
Here is the implementation of `connected`:这里是`connected`的实现：

```python
    # Method on Fetcher class.
    def connected(self, key, mask):
        print('connected!')
        selector.unregister(key.fd)
        request = 'GET {} HTTP/1.0\r\nHost: xkcd.com\r\n\r\n'.format(self.url)
        self.sock.send(request.encode('ascii'))
        
        # Register the next callback.
        selector.register(key.fd,
                          EVENT_READ,
                          self.read_response)
```

The method sends a GET request. A real application would check the return value of `send` in case the whole message cannot be sent at once. But our request is small and our application unsophisticated. It blithely calls `send`, then waits for a response. Of course, it must register yet another callback and relinquish control to the event loop. The next and final callback, `read_response`, processes the server's reply:
该方法发送GET请求。 一个真正的应用程序将检查`send`的返回值，以防整个消息不能立即发送。 但我们的要求很低，我们的应用程序不复杂。 它调用`send`，然后等待响应。 当然，它必须注册另一个回调，并放弃对事件循环的控制。 下一个和最后一个回调，`read_response`，处理服务器的回复：
```python
    # Method on Fetcher class.
    def read_response(self, key, mask):
        global stopped

        chunk = self.sock.recv(4096)  # 4k chunk size.
        if chunk:
            self.response += chunk
        else:
            selector.unregister(key.fd)  # Done reading.
            links = self.parse_links()
            
            # Python set-logic:
            for link in links.difference(seen_urls):
                urls_todo.add(link)
                Fetcher(link).fetch()  # <- New Fetcher.

            seen_urls.update(links)
            urls_todo.remove(self.url)
            if not urls_todo:
                stopped = True
```

The callback is executed each time the selector sees that the socket is "readable", which could mean two things: the socket has data or it is closed.
每当选择器看到socket是“可读的”时，就会执行回调，这可能意味着两件事情：套接字有数据或关闭。
The callback asks for up to four kilobytes of data from the socket. If less is ready, `chunk` contains whatever data is available. If there is more, `chunk` is four kilobytes long and the socket remains readable, so the event loop runs this callback again on the next tick. When the response is complete, the server has closed the socket and `chunk` is empty.
回调从socket请求最多4K字节的数据。 如果less准备好，`chunk`包含任何可用的数据。 如果有更多，`chunk`是4K字节长，并且socket保持可读，所以事件循环在下一个tick时再次运行这个回调。 当响应完成时，服务器已关闭socket，并且“chunk”为空。
The `parse_links` method, not shown, returns a set of URLs. We start a new fetcher for each new URL, with no concurrency cap. Note a nice feature of async programming with callbacks: we need no mutex around changes to shared data, such as when we add links to `seen_urls`. There is no preemptive multitasking, so we cannot be interrupted at arbitrary points in our code.
`parse_links`方法（未显示）返回一组URL。 我们为每个新网址开始一个新的抓取器，没有并发上限。 注意使用回调的异步编程的一个很好的功能：我们不需要围绕共享数据的变化的互斥，例如当我们添加链接到`seen_urls`。 没有抢先的多任务，所以我们不能在我们的代码中的任意点被打断。
We add a global `stopped` variable and use it to control the loop:
我们添加一个全局`stopped`变量，并使用它来控制循环：
```python
stopped = False

def loop():
    while not stopped:
        events = selector.select()
        for event_key, event_mask in events:
            callback = event_key.data
            callback()
```

Once all pages are downloaded the fetcher stops the global event loop and the program exits.
一旦所有页面被下载，fetcher会停止全局事件循环，程序退出。
This example makes async's problem plain: spaghetti code. We need some way to express a series of computations and I/O operations, and schedule multiple such series of operations to run concurrently. But without threads, a series of operations cannot be collected into a single function: whenever a function begins an I/O operation, it explicitly saves whatever state will be needed in the future, then returns. You are responsible for thinking about and writing this state-saving code.
这个例子使async的问题很简单：spaghetti code。 我们需要一些方法来表达一系列计算和I / O操作，并且调度多个这样的一系列操作以并发运行。 但是没有线程，一系列操作不能被收集到单个函数中：每当一个函数开始I / O操作时，它显式地保存将来需要的任何状态，然后返回。 你负责思考和编写这个state-saving代码。
Let us explain what we mean by that. Consider how simply we fetched a URL on a thread with a conventional blocking socket:
让我们解释一下我们的意思。 考虑我们如何简单地在一个具有常规阻塞socket的线程上获取一个URL：
```python
# Blocking version.
def fetch(url):
    sock = socket.socket()
    sock.connect(('xkcd.com', 80))
    request = 'GET {} HTTP/1.0\r\nHost: xkcd.com\r\n\r\n'.format(url)
    sock.send(request.encode('ascii'))
    response = b''
    chunk = sock.recv(4096)
    while chunk:
        response += chunk
        chunk = sock.recv(4096)
    
    # Page is now downloaded.
    links = parse_links(response)
    q.add(links)
```

What state does this function remember between one socket operation and the next? It has the socket, a URL, and the accumulating `response`.  A function that runs on a thread uses basic features of the programming language to store this temporary state in local variables, on its stack. The function also has a "continuation"&mdash;that is, the code it plans to execute after I/O completes. The runtime remembers the continuation by storing the thread's instruction pointer. You need not think about restoring these local variables and the continuation after I/O. It is built in to the language.
这个函数在一个socket操作和下一个socket操作之间记住什么状态？ 它有socket，一个URL和累积的“响应”。 在线程上运行的函数使用编程语言的基本特性将该临时状态存储在其堆栈中的局部变量中。 该函数还有一个“continuation”，即它计划在I / O完成后执行的代码。 运行时通过存储线程的指令指针来记住continuation。 您不必考虑恢复这些局部变量和I / O后的继续。 它是内置的语言。
But with a callback-based async framework, these language features are no help. While waiting for I/O, a function must save its state explicitly, because the function returns and loses its stack frame before I/O completes. In lieu of local variables, our callback-based example stores `sock` and `response` as attributes of `self`, the Fetcher instance. In lieu of the instruction pointer, it stores its continuation by registering the callbacks `connected` and `read_response`. As the application's features grow, so does the complexity of the state we manually save across callbacks. Such onerous bookkeeping makes the coder prone to migraines.
但是使用基于回调的异步框架，这些语言功能没有帮助。 在等待I / O时，函数必须显式地保存其状态，因为函数在I / O完成之前返回并丢失其堆栈。 代替局部变量，我们的基于回调的例子将`sock`和`response`存储为`self`的属性，Fetcher实例。 代替指令指针，它通过注册回调`connected`和`read_response`来存储它的continuation。 随着应用程序的功能增长，我们在回调中手动保存的状态的复杂性也在增加。 这样繁重的记录使得编码器倾向于偏头痛。
Even worse, what happens if a callback throws an exception, before it schedules the next callback in the chain? Say we did a poor job on the `parse_links` method and it throws an exception parsing some HTML:
更糟的是，如果回调引发异常，在调度链中的下一个回调之前会发生什么？ 我们在`parse_links`方法上做了一个不好的工作，它抛出一个解析一些HTML的异常：
```
Traceback (most recent call last):
  File "loop-with-callbacks.py", line 111, in <module>
    loop()
  File "loop-with-callbacks.py", line 106, in loop
    callback(event_key, event_mask)
  File "loop-with-callbacks.py", line 51, in read_response
    links = self.parse_links()
  File "loop-with-callbacks.py", line 67, in parse_links
    raise Exception('parse error')
Exception: parse error
```

The stack trace shows only that the event loop was running a callback. We do not remember what led to the error. The chain is broken on both ends: we forgot where we were going and whence we came. This loss of context is called "stack ripping", and in many cases it confounds the investigator. Stack ripping also prevents us from installing an exception handler for a chain of callbacks, the way a "try / except" block wraps a function call and its tree of descendents.[^7]
堆栈跟踪仅显示事件循环正在运行回调。 我们不记得是什么导致的错误。 链条在两端都断了：我们忘了我们去哪里，我们从哪来。 这种上下文的丢失被称为“堆栈翻录”，并且在许多情况下它混淆了研究者。 堆栈翻录还防止我们为回调链安装异常处理程序，“try / except”块包装函数调用及其后代树。[^ 7]
So, even apart from the long debate about the relative efficiencies of multithreading and async, there is this other debate regarding which is more error-prone: threads are susceptible to data races if you make a mistake synchronizing them, but callbacks are stubborn to debug due to stack ripping. 
因此，除了关于多线程和异步的相对效率的长期争论之外，还有另一个争论是更容易出错的：如果你错误的同步它们，线程就容易受到数据竞争，但是回调由于堆栈翻录，固执的调试 。

## Coroutines 协程

We entice you with a promise. It is possible to write asynchronous code that combines the efficiency of callbacks with the classic good looks of multithreaded programming. This combination is achieved with a pattern called "coroutines". Using Python 3.4's standard asyncio library, and a package called "aiohttp", fetching a URL in a coroutine is very direct[^10]:
我们用一个承诺诱惑你。 可以编写异步代码，将回调的效率与多线程编程的经典好看结合起来。 这种组合通过称为“协程”的模式来实现。 使用Python 3.4的标准asyncio库和一个名为“aiohttp”的包，在协程中获取一个URL是非常直接的[^ 10]：
```python
    @asyncio.coroutine
    def fetch(self, url):
        response = yield from self.session.get(url)
        body = yield from response.read()
```

It is also scalable. Compared to the 50k of memory per thread and the operating system's hard limits on threads, a Python coroutine takes barely 3k of memory on Jesse's system. Python can easily start hundreds of thousands of coroutines.
它也是可扩展的。 与每个线程的50k内存和操作系统对线程的硬限制相比，Python协程在Jesse系统上只需要3k的内存。 Python可以轻松启动数十万个协程。
The concept of a coroutine, dating to the elder days of computer science, is simple: it is a subroutine that can be paused and resumed. Whereas threads are preemptively multitasked by the operating system, coroutines multitask cooperatively: they choose when to pause, and which coroutine to run next.
协程的概念，可以追溯到计算机科学的祖先，很简单：它是一个可以暂停和恢复的子程序。 而线程是由操作系统抢占式多任务，协同多任务协作：他们选择何时暂停，以及哪个协程运行下一步。
There are many implementations of coroutines; even in Python there are several. The coroutines in the standard "asyncio" library in Python 3.4 are built upon generators, a Future class, and the "yield from" statement. Starting in Python 3.5, coroutines are a native feature of the language itself[^17]; however, understanding coroutines as they were first implemented in Python 3.4, using pre-existing language facilities, is the foundation to tackle Python 3.5's native coroutines.
有很多协同的实现; 即使在Python有几个。 Python 3.4中的标准“asyncio”库中的协程是基于generator，Future类和“yield from”语句构建的。 从Python 3.5开始，协程是语言本身的一个本地特性[^ 17]; 然而，了解协同程序，因为他们第一次在Python 3.4中实现，使用预先存在的语言设施，是解决Python 3.5的本地协同程序的基础。
To explain Python 3.4's generator-based coroutines, we will engage in an exposition of generators and how they are used as coroutines in asyncio, and trust you will enjoy reading it as much as we enjoyed writing it. Once we have explained generator-based coroutines, we shall use them in our async web crawler.
为了解释Python 3.4的基于生成器的协程，我们将介绍一些生成器，以及它们如何在asyncio中用作协同程序，并且相信你会喜欢阅读它，就像我们喜欢写它一样。 一旦我们解释了基于生成器的协同程序，我们将使用它们在我们的异步Web爬虫。
## How Python Generators Work Python生成器如何工作

Before you grasp Python generators, you have to understand how regular Python functions work. Normally, when a Python function calls a subroutine, the subroutine retains control until it returns, or throws an exception. Then control returns to the caller:
在掌握Python生成器之前，您必须了解常规Python函数的工作原理。 通常，当Python函数调用子例程时，子例程保留控制权，直到返回或抛出异常。 然后控制权返回给调用者：
```python
>>> def foo():
...     bar()
...
>>> def bar():
...     pass
```

The standard Python interpreter is written in C. The C function that executes a Python function is called, mellifluously, `PyEval_EvalFrameEx`. It takes a Python stack frame object and evaluates Python bytecode in the context of the frame. Here is the bytecode for `foo`:
标准的Python解释器是用C编写的。执行Python函数的C函数被称为`PyEval_EvalFrameEx`。 它需要一个Python栈框架对象，并在框架的上下文中评估Python字节码。 这里是`foo`的字节码：
```python
>>> import dis
>>> dis.dis(foo)
  2           0 LOAD_GLOBAL              0 (bar)
              3 CALL_FUNCTION            0 (0 positional, 0 keyword pair)
              6 POP_TOP
              7 LOAD_CONST               0 (None)
             10 RETURN_VALUE
```

The `foo` function loads `bar` onto its stack and calls it, then pops its return value from the stack, loads `None` onto the stack, and returns `None`.
`foo`函数将`bar`加载到它的堆栈上并调用它，然后从堆栈中弹出其返回值，将`None`加载到堆栈中，并返回`None`。
When `PyEval_EvalFrameEx` encounters the `CALL_FUNCTION` bytecode, it creates a new Python stack frame and recurses: that is, it calls `PyEval_EvalFrameEx` recursively with the new frame, which is used to execute `bar`.
当`PyEval_EvalFrameEx`遇到`CALL_FUNCTION`字节码时，它创建一个新的Python栈框架和递归：也就是说，它用新的框架递归调用`PyEval_EvalFrameEx`，用来执行`bar`。
It is crucial to understand that Python stack frames are allocated in heap memory! The Python interpreter is a normal C program, so its stack frames are normal stack frames. But the *Python* stack frames it manipulates are on the heap. Among other surprises, this means a Python stack frame can outlive its function call. To see this interactively, save the current frame from within `bar`:
了解Python堆栈在堆内存中分配是至关重要的！ Python解释器是一个正常的C程序，所以它的堆栈是正常的堆栈帧。 但是* Python *堆栈框架操纵是在堆上。 除此之外，这意味着Python堆栈帧可以超过其函数调用。 要以交互方式查看，请从`bar`中保存当前帧：
```python
>>> import inspect
>>> frame = None
>>> def foo():
...     bar()
...
>>> def bar():
...     global frame
...     frame = inspect.currentframe()
...
>>> foo()
>>> # The frame was executing the code for 'bar'.
>>> frame.f_code.co_name
'bar'
>>> # Its back pointer refers to the frame for 'foo'.
>>> caller_frame = frame.f_back
>>> caller_frame.f_code.co_name
'foo'
```

\aosafigure[240pt]{crawler-images/function-calls.png}{Function Calls}{500l.crawler.functioncalls}

The stage is now set for Python generators, which use the same building blocks&mdash;code objects and stack frames&mdash;to marvelous effect.
该阶段现在设置为Python生成器，它使用相同的构建块 - 代码对象和堆栈帧 - 以奇妙的效果。
This is a generator function: 这是一个生成器函数：

```python
>>> def gen_fn():
...     result = yield 1
...     print('result of yield: {}'.format(result))
...     result2 = yield 2
...     print('result of 2nd yield: {}'.format(result2))
...     return 'done'
...     
```

When Python compiles `gen_fn` to bytecode, it sees the `yield` statement and knows that `gen_fn` is a generator function, not a regular one. It sets a flag to remember this fact:
当Python将`gen_fn`编译成字节码时，它看到`yield`语句，并且知道`gen_fn`是一个生成器函数，而不是一个常规函数。 它设置一个标志，以记住这个事实：
```python
>>> # The generator flag is bit position 5.
>>> generator_bit = 1 << 5
>>> bool(gen_fn.__code__.co_flags & generator_bit)
True
```

When you call a generator function, Python sees the generator flag, and it does not actually run the function. Instead, it creates a generator:
当你调用一个生成器函数，Python看到生成器标志，它实际上不运行该函数。 相反，它创建一个生成器：
```python
>>> gen = gen_fn()
>>> type(gen)
<class 'generator'>
```

A Python generator encapsulates a stack frame plus a reference to some code, the body of `gen_fn`:
Python生成器封装了一个栈帧加上一些代码的引用，gen_fn的主体：
```python
>>> gen.gi_code.co_name
'gen_fn'
```

All generators from calls to `gen_fn` point to this same code. But each has its own stack frame. This stack frame is not on any actual stack, it sits in heap memory waiting to be used:
来自`gen_fn`调用的所有生成器都指向这个相同的代码。 但每个都有自己的堆栈帧。 这个堆栈帧不在任何实际堆栈，它在堆内存等待被使用：
\aosafigure[240pt]{crawler-images/generator.png}{Generators}{500l.crawler.generators}

The frame has a "last instruction" pointer, the instruction it executed most recently. In the beginning, the last instruction pointer is -1, meaning the generator has not begun:
该帧具有“最后指令”指针，它是最近执行的指令。 开始时，最后一个指令指针是-1，表示生成器尚未开始：
```python
>>> gen.gi_frame.f_lasti
-1
```

When we call `send`, the generator reaches its first `yield`, and pauses. The return value of `send` is 1, since that is what `gen` passes to the `yield` expression:
当我们调用'send'时，生成器到达它的第一个“yield”，并暂停。 `send`的返回值是1，因为这是`gen`传递给`yield`表达式：
```python
>>> gen.send(None)
1
```

The generator's instruction pointer is now 3 bytecodes from the start, part way through the 56 bytes of compiled Python:
生成器的指令指针现在是3个字节码，部分通过编译的Python的56个字节：
```python
>>> gen.gi_frame.f_lasti
3
>>> len(gen.gi_code.co_code)
56
```

The generator can be resumed at any time, from any function, because its stack frame is not actually on the stack: it is on the heap. Its position in the call hierarchy is not fixed, and it need not obey the first-in, last-out order of execution that regular functions do. It is liberated, floating free like a cloud.
生成器可以在任何时候从任何函数恢复，因为它的堆栈帧实际上不在堆栈上：它在堆上。 它在调用层次结构中的位置不是固定的，并且它不需要遵守常规函数执行的先进先出顺序。 它是解放的，浮动自由像云。
We can send the value "hello" into the generator and it becomes the result of the `yield` expression, and the generator continues until it yields 2:
我们可以发送值“hello”到生成器，它成为`yield`表达式的结果，生成器继续，直到它产生2：
```python
>>> gen.send('hello')
result of yield: hello
2
```

Its stack frame now contains the local variable `result`:
它的堆栈帧现在包含局部变量result：
```python
>>> gen.gi_frame.f_locals
{'result': 'hello'}
```

Other generators created from `gen_fn` will have their own stack frames and local variables.
从`gen_fn`创建的其他生成器将有自己的堆栈帧和局部变量
When we call `send` again, the generator continues from its second `yield`, and finishes by raising the special `StopIteration` exception:
当我们再次调用`send`时，生成器从它的第二个`yield`继续，并且通过提高特殊的StopIteration异常来结束：
```python
>>> gen.send('goodbye')
result of 2nd yield: goodbye
Traceback (most recent call last):
  File "<input>", line 1, in <module>
StopIteration: done
```

The exception has a value, which is the return value of the generator: the string `"done"`.
异常有一个值，它是生成器的返回值：字符串`"done"`。

## Building Coroutines With Generators 构造带有生成器的协程

So a generator can pause, and it can be resumed with a value, and it has a return value. Sounds like a good primitive upon which to build an async programming model, without spaghetti callbacks! We want to build a "coroutine": a routine that is cooperatively scheduled with other routines in the program. Our coroutines will be a simplified version of those in Python's standard "asyncio" library. As in asyncio, we will use generators, futures, and the "yield from" statement.
因此，发生器可以暂停，并且可以使用值恢复，并且它具有返回值。 听起来像一个很好的原语，构建一个异步编程模型，没有意大利面条回调！ 我们想建立一个“协程”：一个与程序中的其他程序合作安排的程序。 我们的协程将是Python标准“asyncio”库中的简化版本。 在asyncio中，我们将使用generator，futures和“yield from”语句。
First we need a way to represent some future result that a coroutine is waiting for. A stripped-down version:
首先，我们需要一种方法来表示协程正在等待的future结果。 精简版本：
```python
class Future:
    def __init__(self):
        self.result = None
        self._callbacks = []

    def add_done_callback(self, fn):
        self._callbacks.append(fn)

    def set_result(self, result):
        self.result = result
        for fn in self._callbacks:
            fn(self)
```

A future is initially "pending". It is "resolved" by a call to `set_result`.[^12]
future 最初是“待定”。 它是通过调用`set_result`来“解析”的。[^ 12]
Let us adapt our fetcher to use futures and coroutines. We wrote `fetch` with a callback:
让我们调整我们的fetcher使用futures 和协程。 我们用回调写了`fetch`：
```python
class Fetcher:
    def fetch(self):
        self.sock = socket.socket()
        self.sock.setblocking(False)
        try:
            self.sock.connect(('xkcd.com', 80))
        except BlockingIOError:
            pass
        selector.register(self.sock.fileno(),
                          EVENT_WRITE,
                          self.connected)

    def connected(self, key, mask):
        print('connected!')
        # And so on....
```

The `fetch` method begins connecting a socket, then registers the callback, `connected`, to be executed when the socket is ready. Now we can combine these two steps into one coroutine:
`fetch`方法开始连接一个socket，然后注册回调，`connect`，当socket就绪时执行。 现在我们可以将这两个步骤组合成一个协程：
```python
    def fetch(self):
        sock = socket.socket()
        sock.setblocking(False)
        try:
            sock.connect(('xkcd.com', 80))
        except BlockingIOError:
            pass

        f = Future()

        def on_connected():
            f.set_result(None)

        selector.register(sock.fileno(),
                          EVENT_WRITE,
                          on_connected)
        yield f
        selector.unregister(sock.fileno())
        print('connected!')
```

Now `fetch` is a generator function, rather than a regular one, because it contains a `yield` statement. We create a pending future, then yield it to pause `fetch` until the socket is ready. The inner function `on_connected` resolves the future.
现在fetch是一个生成器函数，而不是一个常规的函数，因为它包含一个yield语句。 我们创建一个待定的Future，然后让它暂停抓取，直到socket准备就绪。 内部函数on_connected解析Future。
But when the future resolves, what resumes the generator? We need a coroutine *driver*. Let us call it "task":
但是，当future 解决，用什么来恢复生成器？ 我们需要一个协程*driver*程序。 让我们称之为“任务”：
```python
class Task:
    def __init__(self, coro):
        self.coro = coro
        f = Future()
        f.set_result(None)
        self.step(f)

    def step(self, future):
        try:
            next_future = self.coro.send(future.result)
        except StopIteration:
            return

        next_future.add_done_callback(self.step)

# Begin fetching http://xkcd.com/353/
fetcher = Fetcher('/353/')
Task(fetcher.fetch())

loop()
```

The task starts the `fetch` generator by sending `None` into it. Then `fetch` runs until it yields a future, which the task captures as `next_future`. When the socket is connected, the event loop runs the callback `on_connected`, which resolves the future, which calls `step`, which resumes `fetch`.
任务通过发送`None`来启动`fetch`生成器。 然后`fetch`运行，直到它产生一个future，任务捕获为`next_future`。 当套接字连接时，事件循环运行回调`on_connected`，它解析future，它调用`step`，它恢复`fetch`。

## Factoring Coroutines With `yield from`

Once the socket is connected, we send the HTTP GET request and read the server response. These steps need no longer be scattered among callbacks; we gather them into the same generator function:
一旦socket连接，我们发送HTTP GET请求并读取服务器响应。 这些步骤不再分散在回调中; 我们将它们收集到相同的生成器函数中：
```python
    def fetch(self):
        # ... connection logic from above, then:
        sock.send(request.encode('ascii'))

        while True:
            f = Future()

            def on_readable():
                f.set_result(sock.recv(4096))

            selector.register(sock.fileno(),
                              EVENT_READ,
                              on_readable)
            chunk = yield f
            selector.unregister(sock.fileno())
            if chunk:
                self.response += chunk
            else:
                # Done reading.
                break
```

This code, which reads a whole message from a socket, seems generally useful. How can we factor it from `fetch` into a subroutine? Now Python 3's celebrated `yield from` takes the stage. It lets one generator *delegate* to another.
这个代码，从socket读取整个消息，似乎很有用。 我们如何将它从fetch转换为子程序？ 现在Python 3的`yield from`走上舞台。 它让一个生成器委托给另一个。
To see how, let us return to our simple generator example:
为了看看怎么做，让我们回到我们简单的生成器示例：
```python
>>> def gen_fn():
...     result = yield 1
...     print('result of yield: {}'.format(result))
...     result2 = yield 2
...     print('result of 2nd yield: {}'.format(result2))
...     return 'done'
...     
```

To call this generator from another generator, delegate to it with `yield from`:
要从另一个生成器调用这个生成器，使用`yield from`来委托它：
```python
>>> # Generator function:
>>> def caller_fn():
...     gen = gen_fn()
...     rv = yield from gen
...     print('return value of yield-from: {}'
...           .format(rv))
...
>>> # Make a generator from the
>>> # generator function.
>>> caller = caller_fn()
```

The `caller` generator acts as if it were `gen`, the generator it is delegating to:
 `caller`生成器就像是`gen`，它被委托给：
```python
>>> caller.send(None)
1
>>> caller.gi_frame.f_lasti
15
>>> caller.send('hello')
result of yield: hello
2
>>> caller.gi_frame.f_lasti  # Hasn't advanced.
15
>>> caller.send('goodbye')
result of 2nd yield: goodbye
return value of yield-from: done
Traceback (most recent call last):
  File "<input>", line 1, in <module>
StopIteration
```

While `caller` yields from `gen`, `caller` does not advance. Notice that its instruction pointer remains at 15, the site of its `yield from` statement, even while the inner generator `gen` advances from one `yield` statement to the next.[^13] From our perspective outside `caller`, we cannot tell if the values it yields are from `caller` or from the generator it delegates to. And from inside `gen`, we cannot tell if values are sent in from `caller` or from outside it. The `yield from` statement is a frictionless channel, through which values flow in and out of `gen` until `gen` completes.
虽然`caller`从 `gen`产生，`caller` 不会前进。 注意，它的指令指针保持在15，即它的yield from语句的位置，即使内部生成器`gen`从一个yield语句前进到下一个。[^ 13]从我们的角度看， 我们不能知道它产生的值是从`caller`还是从它委派的生成器。 从`gen`里面，我们不能知道值是从`caller`还是从外部发送。 “yield from”语句是一个无摩擦的通道，尽管值通过它流入和离开`gen`，直到`gen`完成。
A coroutine can delegate work to a sub-coroutine with `yield from` and receive the result of the work. Notice, above, that `caller` printed "return value of yield-from: done". When `gen` completed, its return value became the value of the `yield from` statement in `caller`:
协程可以将工作委托给具有`yield from` 的子协程，并接收工作的结果。 注意，上面的`caller`打印“return value of yield-from: done”。 当`gen`完成时，其返回值成为`caller'中`yield from`语句的值：
```python
    rv = yield from gen
```

Earlier, when we criticized callback-based async programming, our most strident complaint was about "stack ripping": when a callback throws an exception, the stack trace is typically useless. It only shows that the event loop was running the callback, not *why*. How do coroutines fare?
早些时候，当我们批评基于回调的异步编程时，我们最强烈的投诉是关于“stack ripping”：当回调抛出异常时，堆栈跟踪通常是无用的。 它只显示事件循环正在运行回调，而不是*为什么*。 协程如何运行？
```python
>>> def gen_fn():
...     raise Exception('my error')
>>> caller = caller_fn()
>>> caller.send(None)
Traceback (most recent call last):
  File "<input>", line 1, in <module>
  File "<input>", line 3, in caller_fn
  File "<input>", line 2, in gen_fn
Exception: my error
```

This is much more useful! The stack trace shows `caller_fn` was delegating to `gen_fn` when it threw the error. Even more comforting, we can wrap the call to a sub-coroutine in an exception handler, the same is with normal subroutines:
这更有用！ 堆栈跟踪显示 `caller_fn`在委托`gen_fn` 时抛出错误。 更令人欣慰的是，我们可以将调用包装到异常处理程序中的子协程，同样的是使用正常的子程序：
```python
>>> def gen_fn():
...     yield 1
...     raise Exception('uh oh')
...
>>> def caller_fn():
...     try:
...         yield from gen_fn()
...     except Exception as exc:
...         print('caught {}'.format(exc))
...
>>> caller = caller_fn()
>>> caller.send(None)
1
>>> caller.send('hello')
caught uh oh
```

So we factor logic with sub-coroutines just like with regular subroutines. Let us factor some useful sub-coroutines from our fetcher. We write a `read` coroutine to receive one chunk:
因此，我们使用子协程，就像使用常规子程序一样。 让我们从我们的fetcher中得到一些有用的子协程。 我们写一个`read`协程来接收一个块：
```python
def read(sock):
    f = Future()

    def on_readable():
        f.set_result(sock.recv(4096))

    selector.register(sock.fileno(), EVENT_READ, on_readable)
    chunk = yield f  # Read one chunk.
    selector.unregister(sock.fileno())
    return chunk
```

We build on `read` with a `read_all` coroutine that receives a whole message:
我们使用`read`协程构建`read_all`，它接收一条完整的消息：
```python
def read_all(sock):
    response = []
    # Read whole response.
    chunk = yield from read(sock)
    while chunk:
        response.append(chunk)
        chunk = yield from read(sock)

    return b''.join(response)
```

If you squint the right way, the `yield from` statements disappear and these look like conventional functions doing blocking I/O. But in fact, `read` and `read_all` are coroutines. Yielding from `read` pauses `read_all` until the I/O completes. While `read_all` is paused, asyncio's event loop does other work and awaits other I/O events; `read_all` is resumed with the result of `read` on the next loop tick once its event is ready.
如果你以正确的方式看待，`yield from`语句消失，这些看起来像阻塞I / O的常规函数。 但事实上，`read`和`read_all`是协程。 从 `read`读取暂停`read_all`，直到I / O完成。 当`read_all`被暂停时，asyncio的事件循环执行其他工作，并等待其他I / O事件; `read_all`在其事件准备就绪后，在下一个循环中返回“read”的结果。
At the stack's root, `fetch` calls `read_all`:
在栈的根，`fetch`调用`read_all`：
```python
class Fetcher:
    def fetch(self):
		 # ... connection logic from above, then:
        sock.send(request.encode('ascii'))
        self.response = yield from read_all(sock)
```

Miraculously, the Task class needs no modification. It drives the outer `fetch` coroutine just the same as before:
奇怪的是，Task类不需要修改。 它与之前一样驱动外部`fetch`协程：
```python
Task(fetcher.fetch())
loop()
```

When `read` yields a future, the task receives it through the channel of `yield from` statements, precisely as if the future were yielded directly from `fetch`. When the loop resolves a future, the task sends its result into `fetch`, and the value is received by `read`, exactly as if the task were driving `read` directly:
当`read`产生future时，任务通过`yield from`语句的通道接收它，就好像future直接从`fetch`中获得。 当循环解决future时，任务将其结果发送到`fetch`，并且值由`read`接收，就像任务直接驱动`read`：

\aosafigure[240pt]{crawler-images/yield-from.png}{Yield From}{500l.crawler.yieldfrom}

To perfect our coroutine implementation, we polish out one mar: our code uses `yield` when it waits for a future, but `yield from` when it delegates to a sub-coroutine. It would be more refined if we used `yield from` whenever a coroutine pauses. Then a coroutine need not concern itself with what type of thing it awaits.
为了完善我们的协程实现，我们抛弃一个mar：我们的代码在等待future时使用`yield`，而在委托给一个子协程时使用`yield from`。 如果我们每当一个协程暂停时使用`yield from`，它会更精确。 然后协同程序不需要关心它等待什么类型的事情。
We take advantage of the deep correspondence in Python between generators and iterators. Advancing a generator is, to the caller, the same as advancing an iterator. So we make our Future class iterable by implementing a special method:
我们利用Python在生成器和迭代器之间的深层对应。 推进生成器对于调用者，与推进迭代器相同。 所以我们通过实现一个特殊的方法使我们的Future类可迭代：
```python
    # Method on Future class.
    def __iter__(self):
        # Tell Task to resume me here.
        yield self
        return self.result
```

The future's `__iter__` method is a coroutine that yields the future itself. Now when we replace code like this:
future的`__iter__`方法是一个协同程序，它产生future本身。 现在当我们像这样替换代码：
```python
# f is a Future.
yield f
```

...with this:

```python
# f is a Future.
yield from f
```

...the outcome is the same! The driving Task receives the future from its call to `send`, and when the future is resolved it sends the new result back into the coroutine.
...结果是一样的！ 驱动任务从其对`send`的调用接收future ，并且当future 被解决时，它将新的结果发送回协程。

What is the advantage of using `yield from` everywhere? Why is that better than waiting for futures with `yield` and delegating to sub-coroutines with `yield from`? It is better because now, a method can freely change its implementation without affecting the caller: it might be a normal method that returns a future that will *resolve* to a value, or it might be a coroutine that contains `yield from` statements and *returns* a value. In either case, the caller need only `yield from` the method in order to wait for the result.
使用`yield from` 的优势是什么？ 为什么比等待具有 `yield`的future，并委托给具有`yield from`的子协程更好？ 它是更好的，因为现在，一个方法可以自由地改变其实现，而不影响调用者：它可能是一个正常的方法，返回一个future将*解析*一个值，或者它可能是一个协程包含`yield from`语句 和*返回*一个值。 在任一情况下，调用者只需要`yield from` 方法来等待结果。
Gentle reader, we have reached the end of our enjoyable exposition of coroutines in asyncio. We peered into the machinery of generators, and sketched an implementation of futures and tasks. We outlined how asyncio attains the best of both worlds: concurrent I/O that is more efficient than threads and more legible than callbacks. Of course, the real asyncio is much more sophisticated than our sketch. The real framework addresses zero-copy I/O, fair scheduling, exception handling, and an abundance of other features.
温柔的读者，我们已经到达了我们愉快的在asyncio的协程的终点。 我们探讨了生成器的机制，并草拟了一个futures 和tasks的实现。 我们概述了asyncio如何实现两个中最好的：并发I / O比线程更有效，比回调更清晰。 当然，真正的asyncio比我们的草图更复杂。 真正的框架解决了零拷贝I / O，公平调度，异常处理和大量其他功能。
To an asyncio user, coding with coroutines is much simpler than you saw here. In the code above we implemented coroutines from first principles, so you saw callbacks, tasks, and futures. You even saw non-blocking sockets and the call to ``select``. But when it comes time to build an application with asyncio, none of this appears in your code. As we promised, you can now sleekly fetch a URL:
对于asyncio用户，使用协程的编码比你在这里看到的要简单得多。 在上面的代码中，我们从第一个原则实现协程，所以你看到回调，tasks和futures。 你甚至看到非阻塞socket和调用 ``select``。 但是当使用asyncio构建应用程序时，这些都不会出现在您的代码中。 正如我们承诺的，你现在可以顺利地获取一个URL：
```python
    @asyncio.coroutine
    def fetch(self, url):
        response = yield from self.session.get(url)
        body = yield from response.read()
```

Satisfied with this exposition, we return to our original assignment: to write an async web crawler, using asyncio.
我们回到我们原来的任务：写一个异步的网络爬虫，使用asyncio。


## Coordinating Coroutines

We began by describing how we want our crawler to work. Now it is time to implement it with asyncio coroutines.

Our crawler will fetch the first page, parse its links, and add them to a queue. After this it fans out across the website, fetching pages concurrently. But to limit load on the client and server, we want some maximum number of workers to run, and no more. Whenever a worker finishes fetching a page, it should immediately pull the next link from the queue. We will pass through periods when there is not enough work to go around, so some workers must pause. But when a worker hits a page rich with new links, then the queue suddenly grows and any paused workers should wake and get cracking. Finally, our program must quit once its work is done.

Imagine if the workers were threads. How would we express the crawler's algorithm? We could use a synchronized queue[^5] from the Python standard library. Each time an item is put in the queue, the queue increments its count of "tasks". Worker threads call `task_done` after completing work on an item. The main thread blocks on `Queue.join` until each item put in the queue is matched by a `task_done` call, then it exits.

Coroutines use the exact same pattern with an asyncio queue! First we import it[^6]:

```python
try:
    from asyncio import JoinableQueue as Queue
except ImportError:
    # In Python 3.5, asyncio.JoinableQueue is
    # merged into Queue.
    from asyncio import Queue
```

We collect the workers' shared state in a crawler class, and write the main logic in its `crawl` method. We start `crawl` on a coroutine and run asyncio's event loop until `crawl` finishes:

```python
loop = asyncio.get_event_loop()

crawler = crawling.Crawler('http://xkcd.com',
                           max_redirect=10)

loop.run_until_complete(crawler.crawl())
```

The crawler begins with a root URL and `max_redirect`, the number of redirects it is willing to follow to fetch any one URL. It puts the pair `(URL, max_redirect)` in the queue. (For the reason why, stay tuned.)

```python
class Crawler:
    def __init__(self, root_url, max_redirect):
        self.max_tasks = 10
        self.max_redirect = max_redirect
        self.q = Queue()
        self.seen_urls = set()
        
        # aiohttp's ClientSession does connection pooling and
        # HTTP keep-alives for us.
        self.session = aiohttp.ClientSession(loop=loop)
        
        # Put (URL, max_redirect) in the queue.
        self.q.put((root_url, self.max_redirect))
```

The number of unfinished tasks in the queue is now one. Back in our main script, we launch the event loop and the `crawl` method:

```python
loop.run_until_complete(crawler.crawl())
```

The `crawl` coroutine kicks off the workers. It is like a main thread: it blocks on `join` until all tasks are finished, while the workers run in the background.

```python
    @asyncio.coroutine
    def crawl(self):
        """Run the crawler until all work is done."""
        workers = [asyncio.Task(self.work())
                   for _ in range(self.max_tasks)]

        # When all work is done, exit.
        yield from self.q.join()
        for w in workers:
            w.cancel()
```

If the workers were threads we might not wish to start them all at once. To avoid creating expensive threads until it is certain they are necessary, a thread pool typically grows on demand. But coroutines are cheap, so we simply start the maximum number allowed.

It is interesting to note how we shut down the crawler. When the `join` future resolves, the worker tasks are alive but suspended: they wait for more URLs but none come. So, the main coroutine cancels them before exiting. Otherwise, as the Python interpreter shuts down and calls all objects' destructors, living tasks cry out:

```
ERROR:asyncio:Task was destroyed but it is pending!
```

And how does `cancel` work? Generators have a feature we have not yet shown you. You can throw an exception into a generator from outside:

```python
>>> gen = gen_fn()
>>> gen.send(None)  # Start the generator as usual.
1
>>> gen.throw(Exception('error'))
Traceback (most recent call last):
  File "<input>", line 3, in <module>
  File "<input>", line 2, in gen_fn
Exception: error
```

The generator is resumed by `throw`, but it is now raising an exception. If no code in the generator's call stack catches it, the exception bubbles back up to the top. So to cancel a task's coroutine:

```python
    # Method of Task class.
    def cancel(self):
        self.coro.throw(CancelledError)
```

Wherever the generator is paused, at some `yield from` statement, it resumes and throws an exception. We handle cancellation in the task's `step` method:

```python
    # Method of Task class.
    def step(self, future):
        try:
            next_future = self.coro.send(future.result)
        except CancelledError:
            self.cancelled = True
            return
        except StopIteration:
            return

        next_future.add_done_callback(self.step)
```

Now the task knows it is cancelled, so when it is destroyed it does not rage against the dying of the light.

Once `crawl` has canceled the workers, it exits. The event loop sees that the coroutine is complete (we shall see how later), and it too exits:

```python
loop.run_until_complete(crawler.crawl())
```

The `crawl` method comprises all that our main coroutine must do. It is the worker coroutines that get URLs from the queue, fetch them, and parse them for new links. Each worker runs the `work` coroutine independently:

```python
    @asyncio.coroutine
    def work(self):
        while True:
            url, max_redirect = yield from self.q.get()

            # Download page and add new links to self.q.
            yield from self.fetch(url, max_redirect)
            self.q.task_done()
```

Python sees that this code contains `yield from` statements, and compiles it into a generator function. So in `crawl`, when the main coroutine calls `self.work` ten times, it does not actually execute this method: it only creates ten generator objects with references to this code. It wraps each in a Task. The Task receives each future the generator yields, and drives the generator by calling `send` with each future's result when the future resolves. Because the generators have their own stack frames, they run independently, with separate local variables and instruction pointers.

The worker coordinates with its fellows via the queue. It waits for new URLs with:

```python
    url, max_redirect = yield from self.q.get()
```

The queue's `get` method is itself a coroutine: it pauses until someone puts an item in the queue, then resumes and returns the item.

Incidentally, this is where the worker will be paused at the end of the crawl, when the main coroutine cancels it. From the coroutine's perspective, its last trip around the loop ends when `yield from` raises a `CancelledError`.

When a worker fetches a page it parses the links and puts new ones in the queue, then calls `task_done` to decrement the counter. Eventually, a worker fetches a page whose URLs have all been fetched already, and there is also no work left in the queue. Thus this worker's call to `task_done` decrements the counter to zero. Then `crawl`, which is waiting for the queue's `join` method, is unpaused and finishes.

We promised to explain why the items in the queue are pairs, like:

```python
# URL to fetch, and the number of redirects left.
('http://xkcd.com/353', 10)
```

New URLs have ten redirects remaining. Fetching this particular URL results in a redirect to a new location with a trailing slash. We decrement the number of redirects remaining, and put the next location in the queue:

```python
# URL with a trailing slash. Nine redirects left.
('http://xkcd.com/353/', 9)
```

The `aiohttp` package we use would follow redirects by default and give us the final response. We tell it not to, however, and handle redirects in the crawler, so it can coalesce redirect paths that lead to the same destination: if we have already seen this URL, it is in ``self.seen_urls`` and we have already started on this path from a different entry point:

\aosafigure[240pt]{crawler-images/redirects.png}{Redirects}{500l.crawler.redirects}

The crawler fetches "foo" and sees it redirects to "baz", so it adds "baz" to
the queue and to ``seen_urls``. If the next page it fetches is "bar", which
also redirects to "baz", the fetcher does not enqueue "baz" again. If the
response is a page, rather than a redirect, `fetch` parses it for links and
puts new ones in the queue.

```python
    @asyncio.coroutine
    def fetch(self, url, max_redirect):
        # Handle redirects ourselves.
        response = yield from self.session.get(
            url, allow_redirects=False)

        try:
            if is_redirect(response):
                if max_redirect > 0:
                    next_url = response.headers['location']
                    if next_url in self.seen_urls:
                        # We have been down this path before.
                        return
    
                    # Remember we have seen this URL.
                    self.seen_urls.add(next_url)
                    
                    # Follow the redirect. One less redirect remains.
                    self.q.put_nowait((next_url, max_redirect - 1))
    	     else:
    	         links = yield from self.parse_links(response)
    	         # Python set-logic:
    	         for link in links.difference(self.seen_urls):
                    self.q.put_nowait((link, self.max_redirect))
                self.seen_urls.update(links)
        finally:
            # Return connection to pool.
            yield from response.release()
```

If this were multithreaded code, it would be lousy with race conditions. For example, the worker checks if a link is in `seen_urls`, and if not the worker puts it in the queue and adds it to `seen_urls`. If it were interrupted between the two operations, then another worker might parse the same link from a different page, also observe that it is not in `seen_urls`, and also add it to the queue. Now that same link is in the queue twice, leading (at best) to duplicated work and wrong statistics.

However, a coroutine is only vulnerable to interruption at `yield from` statements. This is a key difference that makes coroutine code far less prone to races than multithreaded code: multithreaded code must enter a critical section explicitly, by grabbing a lock, otherwise it is interruptible. A Python coroutine is uninterruptible by default, and only cedes control when it explicitly yields.

We no longer need a fetcher class like we had in the callback-based program. That class was a workaround for a deficiency of callbacks: they need some place to store state while waiting for I/O, since their local variables are not preserved across calls. But the `fetch` coroutine can store its state in local variables like a regular function does, so there is no more need for a class.

When `fetch` finishes processing the server response it returns to the caller, `work`. The `work` method calls `task_done` on the queue and then gets the next URL from the queue to be fetched.

When `fetch` puts new links in the queue it increments the count of unfinished tasks and keeps the main coroutine, which is waiting for `q.join`, paused. If, however, there are no unseen links and this was the last URL in the queue, then when `work` calls `task_done` the count of unfinished tasks falls to zero. That event unpauses `join` and the main coroutine completes.

The queue code that coordinates the workers and the main coroutine is like this[^9]:

```python
class Queue:
    def __init__(self):
        self._join_future = Future()
        self._unfinished_tasks = 0
        # ... other initialization ...
    
    def put_nowait(self, item):
        self._unfinished_tasks += 1
        # ... store the item ...

    def task_done(self):
        self._unfinished_tasks -= 1
        if self._unfinished_tasks == 0:
            self._join_future.set_result(None)

    @asyncio.coroutine
    def join(self):
        if self._unfinished_tasks > 0:
            yield from self._join_future
```

The main coroutine, `crawl`, yields from `join`. So when the last worker decrements the count of unfinished tasks to zero, it signals `crawl` to resume, and finish.

The ride is almost over. Our program began with the call to `crawl`:

```python
loop.run_until_complete(self.crawler.crawl())
```

How does the program end? Since `crawl` is a generator function, calling it returns a generator. To drive the generator, asyncio wraps it in a task:

```python
class EventLoop:
    def run_until_complete(self, coro):
        """Run until the coroutine is done."""
        task = Task(coro)
        task.add_done_callback(stop_callback)
        try:
            self.run_forever()
        except StopError:
            pass

class StopError(BaseException):
    """Raised to stop the event loop."""

def stop_callback(future):
    raise StopError
```

When the task completes, it raises `StopError `, which the loop uses as a signal that it has arrived at normal completion.

But what's this? The task has methods called `add_done_callback` and `result`? You might think that a task resembles a future. Your instinct is correct. We must admit a detail about the Task class we hid from you: a task is a future.

```python
class Task(Future):
    """A coroutine wrapped in a Future."""
```

Normally a future is resolved by someone else calling `set_result` on it. But a task resolves *itself* when its coroutine stops. Remember from our earlier exploration of Python generators that when a generator returns, it throws the special `StopIteration` exception:

```python
    # Method of class Task.
    def step(self, future):
        try:
            next_future = self.coro.send(future.result)
        except CancelledError:
            self.cancelled = True
            return
        except StopIteration as exc:

            # Task resolves itself with coro's return
            # value.
            self.set_result(exc.value)
            return

        next_future.add_done_callback(self.step)
```

So when the event loop calls `task.add_done_callback(stop_callback)`, it prepares to be stopped by the task. Here is `run_until_complete` again:

```python
    # Method of event loop.
    def run_until_complete(self, coro):
        task = Task(coro)
        task.add_done_callback(stop_callback)
        try:
            self.run_forever()
        except StopError:
            pass
```

When the task catches `StopIteration` and resolves itself, the callback raises `StopError` from within the loop. The loop stops and the call stack is unwound to `run_until_complete`. Our program is finished.

## Conclusion

Increasingly often, modern programs are I/O-bound instead of CPU-bound. For such programs, Python threads are the worst of both worlds: the global interpreter lock prevents them from actually executing computations in parallel, and preemptive switching makes them prone to races. Async is often the right pattern. But as callback-based async code grows, it tends to become a dishevelled mess. Coroutines are a tidy alternative. They factor naturally into subroutines, with sane exception handling and stack traces.

If we squint so that the `yield from` statements blur, a coroutine looks like a thread doing traditional blocking I/O. We can even coordinate coroutines with classic patterns from multi-threaded programming. There is no need for reinvention. Thus, compared to callbacks, coroutines are an inviting idiom to the coder experienced with multithreading.

But when we open our eyes and focus on the `yield from` statements, we see they mark points when the coroutine cedes control and allows others to run. Unlike threads, coroutines display where our code can be interrupted and where it cannot. In his illuminating essay "Unyielding"[^4], Glyph Lefkowitz writes, "Threads make local reasoning difficult, and local reasoning is perhaps the most important thing in software development." Explicitly yielding, however, makes it possible to "understand the behavior (and thereby, the correctness) of a routine by examining the routine itself rather than examining the entire system."

This chapter was written during a renaissance in the history of Python and async. Generator-based coroutines, whose devising you have just learned, were released in the "asyncio" module with Python 3.4 in March 2014. In September 2015, Python 3.5 was released with coroutines built in to the language itself. These native coroutinesare declared with the new syntax "async def", and instead of "yield from", they use the new "await" keyword to delegate to a coroutine or wait for a Future.

Despite these advances, the core ideas remain. Python's new native coroutines will be syntactically distinct from generators but work very similarly; indeed, they will share an implementation within the Python interpreter. Task, Future, and the event loop will continue to play their roles in asyncio.

Now that you know how asyncio coroutines work, you can largely forget the details. The machinery is tucked behind a dapper interface. But your grasp of the fundamentals empowers you to code correctly and efficiently in modern async environments.

[^4]: [https://glyph.twistedmatrix.com/2014/02/unyielding.html](https://glyph.twistedmatrix.com/2014/02/unyielding.html)

[^5]: [https://docs.python.org/3/library/queue.html](https://docs.python.org/3/library/queue.html)

[^6]: [https://docs.python.org/3/library/asyncio-sync.html](https://docs.python.org/3/library/asyncio-sync.html)

[^7]: For a complex solution to this problem, see [http://www.tornadoweb.org/en/stable/stack_context.html](http://www.tornadoweb.org/en/stable/stack_context.html)

[^8]: [http://www.kegel.com/c10k.html](http://www.kegel.com/c10k.html)

[^9]: The actual `asyncio.Queue` implementation uses an `asyncio.Event` in place of the Future shown here. The difference is an Event can be reset, whereas a Future cannot transition from resolved back to pending.

[^10]: The `@asyncio.coroutine` decorator is not magical. In fact, if it decorates a generator function and the `PYTHONASYNCIODEBUG` environment variable is not set, the decorator does practically nothing. It just sets an attribute, `_is_coroutine`, for the convenience of other parts of the framework. It is possible to use asyncio with bare generators not decorated with `@asyncio.coroutine` at all.

<latex>
[^11]: Jesse listed indications and contraindications for using async in "What Is Async, How Does It Work, And When Should I Use It?", available at pyvideo.org. 
[^bayer]: Mike Bayer compared the throughput of asyncio and multithreading for different workloads in his "Asynchronous Python and Databases": http://techspot.zzzeek.org/2015/02/15/asynchronous-python-and-databases/
</latex>
<markdown>
[^11]: Jesse listed indications and contraindications for using async in ["What Is Async, How Does It Work, And When Should I Use It?":](http://pyvideo.org/video/2565/what-is-async-how-does-it-work-and-when-should). Mike Bayer compared the throughput of asyncio and multithreading for different workloads in ["Asynchronous Python and Databases":](http://techspot.zzzeek.org/2015/02/15/asynchronous-python-and-databases/)
</markdown>

[^12]: This future has many deficiencies. For example, once this future is resolved, a coroutine that yields it should resume immediately instead of pausing, but with our code it does not. See asyncio's Future class for a complete implementation.

[^13]: In fact, this is exactly how "yield from" works in CPython. A function increments its instruction pointer before executing each statement. But after the outer generator executes "yield from", it subtracts 1 from its instruction pointer to keep itself pinned at the "yield from" statement. Then it yields to *its* caller. The cycle repeats until the inner generator throws `StopIteration`, at which point the outer generator finally allows itself to advance to the next instruction.

[^14]: Python's global interpreter lock prohibits running Python code in parallel in one process anyway. Parallelizing CPU-bound algorithms in Python requires multiple processes, or writing the parallel portions of the code in C. But that is a topic for another day.

[^15]: Even calls to `send` can block, if the recipient is slow to acknowledge outstanding messages and the system's buffer of outgoing data is full.

<markdown>
[^16]: Guido introduced the standard asyncio library, called "Tulip" then, at [PyCon 2013](http://pyvideo.org/video/1667/keynote).
</markdown>
<latex>
[^16]: Guido introduced the standard asyncio library, called "Tulip" then, at PyCon 2013.
</latex>

[^17]: Python 3.5's built-in coroutines are described in [PEP 492](https://www.python.org/dev/peps/pep-0492/), "Coroutines with async and await syntax."
