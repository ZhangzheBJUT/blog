并发意味着程序在运行时有多个执行上下文，对应着多个调用栈。每个进程在运行时，都有自己的调用栈和堆，有一个完整的上下文，而操作系统在调用进程的时候，会保存被调度进程的上下文环境，等该进程获得时间片后，再恢复该进程在上下文到系统中。

并发的实现模型

1.多进程
	多进程是在操作系统层面进行并发的基本模式。同时也是开销最大的模式。在Linux平台上，很多工具链正式采用这种模式在工作。比如某个Web服务器，它会有专门的进程负责网络端口的监听和链接的管理，还会有专门的进程负责事务和运算。这种的好处在于简单、进程间无不影响，坏处在于系统的开销大，因为所有的进程都是由内核管理的。
2.多线程
    多线程在大多数的系统上都属于系统层面的开发模式，也是我们使用最多最有效的一种是模式。它比多进程的开销小很多，但是其开销依旧比较大，且在高并发模式下，效率会有影响。
3.基于回调的非阻塞/异步IO
	这种架构的诞生实际上来源于多线程模式的危机。在很多高并发服务器开发实践中，使用多线程模式会很快耗尽服务器内存和CPU资源。而这种模式通过事件驱动的方式使用异步IO，使服务器持续运转，且尽可能的减少对线程的使用，降低开销，它目前在Node.js中得到了很好的实践。但是使用这种模式编程比多线程要复杂，因为它把流程做了分割，对于问题本身的反应不够自然。
4.协程
	协程的本质上是一种用户态线程，不需要操作系统来进行抢占式的调度，且在真正的实现中寄予于线程中，因此，系统的开销很小，可以有效的提高线程的任务并发性，而避免多线程的缺点。使用协程的优点是编程简单，结构清晰；缺点是需要语言的支持，如果不支持，则需要用户在程序中自行实现调度器。目前，原生支持协程的语言还很少（个人认为IOS中的GCD也是一种协程的实现，它本质上也是一种用户态的线程)。

协程的最大优势是其轻量级，可以轻松地创建上百万个而不会导致系统资源耗竭，而线程和进程通常最多也不能超过1万个。这也是协程也叫轻量级线程的原因。
Go语言在语言级别支持轻量级线程，叫goroutine.Go语言标准库提供的所有系统调用操作(当然也包括同步IO操作），都会出让CPU给其他goroutine。这让事情变得非常简单，让轻量级线程的切换管理不依赖于系统的线程和进程，也不依赖于CPU的核心数量。

goroutine是Go语言中的轻量级线程的实现，由Go运行时(runtime)管理。在一个函数调用前加上go关键字，这次调用就会有一个新的goroutine中并发执行，当被调用的函数返回时，这个goroutine也自动结束了。需要注意的是，如果这个函数有返回值，那么这个返回值被丢弃。

并发编程的难度在于协调，而协调是通过交流。在工程上有两种并发通信的模型：共享数据和消息。
共享内存是指多个并发单元分别保持对同一个数据的引用，实现对数据的共享。被共享的数据可能有多种形式，比如内存数据块、磁盘文件、网络数据等。实际在工程应用中最常见的是内存。

不要通过共享内存来通信，而是通过通信来共享内存。

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




共享数据
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

1.缓冲的channel
保证往里放数据‘先于
	// By default channels are _unbuffered_, meaning that they
	// will only accept sends (`chan <-`) if there is a
	// corresponding receive (`<- chan`) ready to receive the
	// sent value. _Buffered channels_ accept a limited
	// number of  values without a corresponding receiver for
	// those values.

	package main

	import "fmt"

	func main() {

    	// Here we `make` a channel of strings buffering up to
    	// 2 values.
    	messages := make(chan string, 2)

    	// Because this channel is buffered, we can send these
    	// values into the channel without a corresponding
    	// concurrent receive.
    	messages <- "buffered"
   	    messages <- "channel"

       // Later we can receive these two values as usual.
       fmt.Println(<-messages)
       fmt.Println(<-messages)
}
’对应的取数据。说白了就是保证在取的时候里面肯定有数据，否则就因取不到而阻塞。

2.非缓冲的channel
保证取数据‘先于’放数据。同样说白了就是保证放的时候肯定有另外的goroutine在取，否则就因放不进去而阻塞。
	
	package main

	import (
		"fmt"
		"time"
	)

	var c = make(chan int, 10)

	func Producer() {

		var counter int = 0

		for {
			c <- counter
			counter++
			time.Sleep(time.Second * 2)
		}	

	}

	func Consumer() {

		for {
			counter := <-c
			time.Sleep(time.Second * 3)
			fmt.Println("counter:", counter)
			}

		}		

	func main() {
		go Producer()
		go Consumer()

		for {
			time.Sleep(time.Second * 1000)
		}
	}       
 

带缓冲的channel
                      

	var a string
	var c = make(chan int, 10)
 
	func f() {
   		a = "hello, world"
    	c <- 0
	}
 
	func main() {
   	 	go f()
    	<-c
   	    print(a)
	}

在goroutine f()中，赋值先于c<-0。在goroutine main()中，<-c先于print。而对于channel c来说，它是一个缓冲的channel，从而‘放’先于‘取’，所以c<-0先于<-c。所以串起来便是：赋值先于c<-0先于<-c先于print。从而保证了不同的goroutine中对a的读写如期望的顺序发生。

	var a string
	var c = make(chan int, 10)
 
	func f() {
    	a = "hello, world"
    	c <- 0
	}
 
	func main() {
    	go f()
    	<-c
    	print(a)
	}

再来排排‘先于’关系。在f()中，赋值先于<-c，在main()中c<-0先于print，而c是个非缓冲channel，从而‘取’先于‘放’，所以<-c先于c<-0。串起来便是：赋值先于<-c先于c<-0先于print。同样达到了目的。


同步执行
	// We can use channels to synchronize execution
	// across goroutines. Here's an example of using a
	// blocking receive to wait for a goroutine to finish.

	package main

	import "fmt"
	import "time"

	// This is the function we'll run in a goroutine. The
	// `done` channel will be used to notify another
	// goroutine that this function's work is done.
	func worker(done chan bool) {
    	fmt.Print("working...")
    	time.Sleep(time.Second)
    	fmt.Println("done")

    	// Send a value to notify that we're done.
    	done <- true
	}

	func main() {

    	// Start a worker goroutine, giving it the channel to
    	// notify on.
    	done := make(chan bool, 1)
    	go worker(done)
	
    	// Block until we receive a notification from the
    	// worker on the channel.
    	<-done
	}


参考：https://gobyexample.com/channel-directions