###一 多线程编程概述 

从资源的利用角度看，使用多线程的原因主要有两个:**IO阻塞和多CPU**。

当前线程进行IO处理的时候，会被阻塞释放CPU等待IO操作完成，由于IO操纵(磁盘IO或者网络IO）通常都需要较长的时间，这时CPU可以调度其它的线程进行处理。理想的系统Load是既没有进程（线程）等待也没有CPU空闲，利用多线程IO阻塞与执行交替进行，可最大限度的利用CPU资源。

使用多线程的另外一个原因是服务器有多个CPU，在这个连手机都是八核的时代，除了最低配置的虚拟机，一般数据中心的服务器至少16核CPU，要想最大限度的使用这些CPU，必须启动多线程。

**启动线程数 ＝ ［任务执行时间/(任务执行时间－IO等待时间)] x CPU 内核数**
最佳启动线程数和CPU内核数量成正比，和IO阻塞时间成反比。如果任务都是CPU计算型任务，那么线程最多不超过CPU内核数，因为启动再多线程，CPU也来不及调度；相反如果任务需要等待磁盘操作，网络响应，那么启动多线程有助于提高任务并发度，提高系统吞吐能力，改善系统性能。

多线程意味着程序在运行时有多个执行上下文，对应着多个调用栈。每个进程在运行时，都有自己的调用栈和堆，有一个完整的上下文，而操作系统在调用进程的时候，会保存被调度进程的上下文环境，等该进程获得时间片后，再恢复该进程在上下文到系统中。

多线程安全主要手段：将对象设计为无状态对象、使用局部对象、并发访问资源时使用锁

###二 并发实现模型

1. **多进程**   
	多进程是在操作系统层面进行并发的基本模式。同时也是开销最大的模式。在Linux平台上，很多工具链正式采用这种模式在工作。比如某个Web服务器，它会有专门的进程负责网络端口的监听和链接的管理，还会有专门的进程负责事务和运算。这种的好处在于简单、进程间无不影响，坏处在于系统的开销大，因为所有的进程都是由内核管理的。    
2. **多线程**  
    多线程在大多数的系统上都属于系统层面的开发模式，也是我们使用最多最有效的一种是模式。它比多进程的开销小很多，但是其开销依旧比较大，且在高并发模式下，效率会有影响。
3. **基于回调的非阻塞/异步IO**  
	这种架构的诞生实际上来源于多线程模式的危机。在很多高并发服务器开发实践中，使用多线程模式会很快耗尽服务器内存和CPU资源。而这种模式通过事件驱动的方式使用异步IO，使服务器持续运转，且尽可能的减少对线程的使用，降低开销，它目前在Node.js中得到了很好的实践。但是使用这种模式编程比多线程要复杂，因为它把流程做了分割，对于问题本身的反应不够自然。
4. **协程**  
	协程的本质上是一种用户态线程，不需要操作系统来进行抢占式的调度，且在真正的实现中寄予于线程中，因此，系统的开销很小，可以有效的提高线程的任务并发性，而避免多线程的缺点。使用协程的优点是编程简单，结构清晰；缺点是需要语言的支持，如果不支持，则需要用户在程序中自行实现调度器。目前，原生支持协程的语言还很少（个人认为IOS中的GCD也是一种协程的实现，它本质上也是一种用户态的线程)。

###三 Go语言中协程  

**协程的最大优势是其轻量级，可以轻松地创建上百万个而不会导致系统资源耗竭，而线程和进程通常最多也不能超过1万个，这也是协程也叫轻量级线程的原因。**  
Go语言在语言级别支持轻量级线程，叫goroutine。Go语言标准库提供的所有系统调用操作(当然也包括同步IO操作），都会出让CPU给其他goroutine。这让事情变得非常简单，让轻量级线程的切换管理不依赖于系统的线程和进程，也不依赖于CPU的核心数量。

goroutine是Go语言中的轻量级线程的实现，由Go运行时(runtime)管理。在一个函数调用前加上go关键字，这次调用就会有一个新的goroutine中并发执行，当被调用的函数返回时，这个goroutine也自动结束了。需要注意的是，如果这个函数有返回值，那么这个返回值被丢弃。下面的程序：

	package main

	import "fmt"

	func f(from string) {
    	for i := 0; i < 3; i++ {
       	    fmt.Println(from, ":", i)
    	}
	}

	func main() {

    	// Suppose we have a function call `f(s)`. Here's how
    	// we'd call that in the usual way, running it
    	// synchronously.
    	f("direct")

    	// To invoke this function in a goroutine, use
   		// `go f(s)`. This new goroutine will execute
    	// concurrently with the calling one.
   	    go f("goroutine")

    	// You can also start a goroutine for an anonymous
    	// function call.
    	go func(msg string) {
        	fmt.Println(msg)
    	}("going")

    	// Our two goroutines are running asynchronously in
    	// separate goroutines now, so execution falls through
    	// to here. This `Scanln` code requires we press a key
    	// before the program exits.
    	var input string

   	 	fmt.Scanln(&input)
   	 	fmt.Println("done")
	}

输出结果：   
     
	go run goroutines.go
	direct : 0
	direct : 1
	direct : 2
	goroutine : 0
	going
	goroutine : 1
	goroutine : 2
	<enter>
	done

通过输出结果发现，不加go关键字的函数执行时同步的。下面的两个函数在调用语句前都加了go关键字，这两个函数的执行就会放到一个新的协程中去，从而实现异步的执行。

由上面的例子，我们可以看到在Go中使用多线程是如此的简单。

###四 Go语言协程间通信
并发编程的难度在于协调，而协调是通过交流。在工程上有两种并发通信的模型：**共享数据(内存)和消息**。
共享内存是指多个并发单元分别保持对同一个数据的引用，实现对数据的共享。被共享的数据可能有多种形式，比如内存数据块、磁盘文件、网络数据等，实际在工程应用中最常见的是内存。  
首先，来看一个通过共享内存来实现线程间通信的例子

	package main

	import (
		"fmt"
		"runtime"
		"sync"
	)

	var counter int = 0
    //线程执行时候，加锁的原因是为了保证线程函数执行的原子性
	func Count(lock *sync.Mutex) {

		lock.Lock()
		counter++
		fmt.Print(counter)
		lock.Unlock()
	}

	func main() {

		lock := &sync.Mutex{}

		for i := 0; i < 10; i++ {
			go Count(lock)
		}

		for {
			lock.Lock()
			c := counter
			lock.Unlock()
			runtime.Gosched()
			if c >= 10 {
			break
			}
		}
	}

	注:使用Gosched()函数可以控制goroutine主动出让时间片给其他gorountine


**在Go语言中使用的是后者，Go中的多线程通信:不要通过共享内存来通信，而是通过通信来共享内存。(Do not communicate by sharing memory; instead, share memory by communicating.)**  

	// _Channels_ are the pipes that connect concurrent
	// goroutines. You can send values into channels from one
	// goroutine and receive those values into another
	// goroutine.

	package main

	import "fmt"

	func main() {

    	// Create a new channel with `make(chan val-type)`.
    	// Channels are typed by the values they convey.
    	messages := make(chan string)

    	// _Send_ a value into a channel using the `channel <-`
    	// syntax. Here we send `"ping"`  to the `messages`
    	// channel we made above, from a new goroutine.
    	go func() { messages <- "ping" }()

    	// The `<-channel` syntax _receives_ a value from the
    	// channel. Here we'll receive the `"ping"` message
   	    // we sent above and print it out.
   	    msg := <-messages
        fmt.Println(msg)
	}
首先创建了一个无缓冲的同步类型的字符串类型channel，在匿名函数中存入一个值"ping",此处程序会阻塞，直到执行<-message时，匿名函数才会返回，此时整个程序结束。

通过上面的例子我们发现，我们通过channel不仅实现了两个协程之间的通信，而且还传递了数据。这就是Go语言中并发编程的实现如此的简单和优雅，下一节将重点讲解channel的使用。