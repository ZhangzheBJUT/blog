### 一 channel介绍
channel是Go语言在语言级别提供的goroutine间的通信方式。我们可以使用channel在两个或多个goroutine直间传递消息哦。channel是进程内的通信方式，因此通过channel传递对象的过程和用函数时的参数传递行为比较一致，比如也可以传递指针等。如果需要跨进程通信，我们建议用分布式的方法来解决，比如使用socket或者http等通信协议。
channel是类型相关的，也就是说，一个channel只能传递一种类型的值，这个类型需要在声明channel指定。如果对Unix管道有所了解的话，不难理解channel，可以将其认为是一种类型的管道。


### 二 channel类型


1. 缓冲的channel 保证往里放数据‘先于’对应的取数据。说白了就是保证在取的时候里面肯定有数据，否则就因取不到而阻塞。带缓冲的channel相当于信号量。    
	
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


2. 非缓冲的channel 保证取数据‘先于’放数据。同样说白了就是保证放的时候肯定有另外的goroutine在取，否则就因放不进去而阻塞。非缓冲的channel又叫做同步channel
	
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


	
	

3. 单向channel
    

4. 需要注意的问题    
   在使用channel的时候经常使用死循环，应该在调用channel函数时候加上go关键字，看下面的程序：

		package main
	
		import "fmt"
	
		func foo() {
			defer fmt.Println("World")
			fmt.Println("Hello")
		}
	
		func sum(x, y int, c chan int) {
			c <- x + y
		}
	
		func main() {
			foo()
			c := make(chan int)
			sum(24, 18, c)
			fmt.Println(<-c)
		}
	运行时会报错，c是一个无缓冲的Channel，当执行函数sum(24, 18, c)的时候函数会被阻塞，如果没有使用go Channel就一直在阻塞的状态，执行就死循环了。应该改为：           

		package main
		import "fmt"
	
		func foo() {
			defer fmt.Println("World")
			fmt.Println("Hello")
		}
	
		func sum(x, y int, c chan int) {
			c <- x + y
		}
	
		func main() {
			foo()
			c := make(chan int)
			go sum(24, 18, c)
			fmt.Println(<-c)
		}
	


### 三 超时问题

多线程编程中经常遇到的问题是等待超时问题，Go语言中没有直接可用的方法来避免超时的发生，但是我们可以借助select机制来解决超时等待的问题。

	package main

	import (
		"fmt"
		"time"
	)

	func ProduceData(c chan int64) {

		time.Sleep(time.Second * 6)
		//time.Sleep(time.Second * 3)
		c <- 32
	}
	func main() {

		//首先实现一个匿名的超时等待函数
		timeout := make(chan bool, 1)
		defer close(timeout)

		go func() {
			time.Sleep(time.Second * 4)
			timeout <- true
		}()

		c := make(chan int64, 2)
	
		go ProduceData(c)
	
		select {
		case data := <-c:
			fmt.Println("Read a data", data)
		case <-timeout:
			fmt.Println("timeout......")
		}
	
	}

### 四 总结
    
参考：    
    [http://blog.golang.org/pipelines](http://blog.golang.org/pipelines "http://blog.golang.org/pipelines")   
    [http://golang.org/doc/effective_go.html#channels](http://golang.org/doc/effective_go.html#channels "http://golang.org/doc/effective_go.html#channels")
