## 一 函数式编程概论

在过去近十年时间里，面向对象编程大行其道，以至于在大学的教育里，老师也只会教给我们两种编程模型，面向过程和面向对象。孰不知，在面向对象思想产生之前，函数式编程已经有了数十年的历史。就让我们回顾这个古老又现代的编程模型，看看究竟是什么魔力将这个概念在21世纪的今天再次拉入我们的视野。

随着硬件性能的提升以及编译技术和虚拟机技术的改进，一些曾被性能问题所限制的动态语言开始受到关注，Python、Ruby 和 Lua 等语言都开始在应用中崭露头角。动态语言因其方便快捷的开发方式成为很多人喜爱的编程语言，伴随动态语言的流行，函数式编程也再次进入了我们的视野。

究竟什么是函数式编程呢？

在维基百科中，对函数式编程有很详细的介绍。Wiki上对Functional Programming的定义：

> In computer science, functional programming is a programming paradigm that treats computation as the evaluation of mathematical functions and avoids state and mutable data.

简单地翻译一下，也就是说函数式编程是一种编程模型，他将计算机运算看做是数学中函数的计算，并且避免了状态以及变量的概念。

## 二 闭包
在函数编程中经常用到闭包，闭包是什？它是怎么产生的及用来解决什么问题呢?先给出闭包的字面定义：**闭包是由函数及其相关引用环境组合而成的实体(即：闭包=函数+引用环境)。**这个从字面上很难理解，特别对于一直使用命令式语言进行编程的程序员们。  

闭包只是在形式和表现上像函数，但实际上不是函数。函数是一些可执行的代码，这些代码在函数被定义后就确定了，不会在执行时发生变化，所以一个函数只有一个实例。闭包在运行时可以有多个实例，**不同的引用环境和相同的函数组合可以产生不同的实例。所谓引用环境是指在程序执行中的某个点所有处于活跃状态的约束所组成的集合。其中的约束是指一个变量的名字和其所代表的对象之间的联系。**那么为什么要把引用环境与函数组合起来呢？这主要是因为在支持嵌套作用域的语言中，有时不能简单直接地确定函数的引用环境。这样的语言一般具有这样的特性：
> 函数是一等公民（First-class value），即函数可以作为另一个函数的返回值或参数，还可以作为一个变量的值。  
> 函数可以嵌套定义，即在一个函数内部可以定义另一个函数。


在面向对象编程中，我们把对象传来传去，那在函数式编程中，要做的是把函数传来传去，说成术语，把他叫做高阶函数。在数学和计算机科学中，高阶函数是至少满足下列一个条件的函数:

> 接受一个或多个函数作为输入  
> 输出一个函数

在函数式编程中，函数是基本单位，是第一型，他几乎被用作一切，包括最简单的计算，甚至连变量都被计算所取代。

**闭包小结:**  
函数只是一段可执行代码，编译后就“固化”了，每个函数在内存中只有一份实例，得到函数的入口点便可以执行函数了。在函数式编程语言中，函数是一等公民（First class value）：第一类对象，我们不需要像命令式语言中那样借助函数指针，委托操作函数，函数可以作为另一个函数的参数或返回值，可以赋给一个变量。函数可以嵌套定义，即在一个函数内部可以定义另一个函数，有了嵌套函数这种结构，便会产生闭包问题。如：

	package main
	
	import (
		"fmt"
	)
	
	func adder() func(int) int {
		sum := 0
		innerfunc := func(x int) int {
			sum += x
			return sum
		}
		return innerfunc
	}
	
	func main() {
		pos, neg := adder(), adder()
		for i := 0; i < 10; i++ {
			fmt.Println(pos(i), neg(-2*i))
		}
	
	}
在这段程序中，函数innerfunc是函数adder的内嵌函数，并且是adder函数的返回值。我们注意到一个问题：内嵌函数innerfunc中引用到外层函数中的局部变量sum，Go会这么处理这个问题呢？先让我们来看看这段代码的运行结果：
     
	0 0  
	1 -2  
	3 -6  
	6 -12   
	10 -20  
	15 -30  
	21 -42  
	28 -56  
	36 -72  
	45 -90  
	
注意: **Go不能在函数内部显式嵌套定义函数，但是可以定义一个匿名函数。** 如上面所示，我们定义了一个匿名函数对象，然后将其赋值给innerfunc，最后将其作为返回值返回。

当用不同的参数调用adder函数得到（pos(i)，neg(i)）函数时，得到的结果是隔离的，也就是说每次调用adder返回的函数都将生成并保存一个新的局部变量sum。其实这里adder函数返回的就是闭包。  
这个就是Go中的闭包，一个函数和与其相关的引用环境组合而成的实体。一句关于闭包的名言: **对象是附有行为的数据，而闭包是附有数据的行为。**
 
## 三 闭包使用

闭包经常用于回调函数，当IO操作（例如从网络获取数据、文件读写)完成的时候，会对获取的数据进行某些操作，这些操作可以交给函数对象处理。

除此之外，在一些公共的操作中经常会包含一些差异性的特殊操作，而这些差异性的操作可以用函数来进行封装。看下面的例子：

	package main
	
	import (
		"errors"
		"fmt"
	)
	
	type Traveser func(ele interface{})
	/*
		Process:封装公共切片数组操作
	*/
	func Process(array interface{}, traveser Traveser) error {
	
		if array == nil {
			return errors.New("nil pointer")
		}
		var length int //数组的长度
		switch array.(type) {
		case []int:
			length = len(array.([]int))
		case []string:
			length = len(array.([]string))
		case []float32:
			length = len(array.([]float32))
			default:
			return errors.New("error type")
		}
		if length == 0 {
			return errors.New("len is zero.")
		}
		traveser(array)
		return nil
	}
	/*
		具体操作:升序排序数组元素
	*/
	func SortByAscending(ele interface{}) {
		intSlice, ok := ele.([]int)
		if !ok {
			return
		}
		length := len(intSlice)

		for i := 0; i < length-1; i++ {
			isChange := false
			for j := 0; j < length-1-i; j++ {
	
				if intSlice[j] > intSlice[j+1] {
					isChange = true
					intSlice[j], intSlice[j+1] = intSlice[j+1], intSlice[j]
				}
			}
	
			if isChange == false {
				return
			}
	
		}
	}
	/*
		具体操作:降序排序数组元素
	*/
	func SortByDescending(ele interface{}) {

		intSlice, ok := ele.([]int)
		if !ok {
			return
		}
		length := len(intSlice)
		for i := 0; i < length-1; i++ {
			isChange := false
			for j := 0; j < length-1-i; j++ {
				if intSlice[j] < intSlice[j+1] {
					isChange = true
					intSlice[j], intSlice[j+1] = intSlice[j+1], intSlice[j]
				}
			}
	
			if isChange == false {
				return
			}
	
		}
	}
	
	func main() {
	
		intSlice := make([]int, 0)
		intSlice = append(intSlice, 3, 1, 4, 2)
	
		Process(intSlice, SortByDescending)
		fmt.Println(intSlice) //[4 3 2 1]
		Process(intSlice, SortByAscending)
		fmt.Println(intSlice) //[1 2 3 4]
	
		stringSlice := make([]string, 0)
		stringSlice = append(stringSlice, "hello", "world", "china")
		
		/*
		   具体操作:使用匿名函数封装输出操作
		*/
		Process(stringSlice, func(elem interface{}) {
	
			if slice, ok := elem.([]string); ok {
				for index, value := range slice {
					fmt.Println("index:", index, "  value:", value)
				}
			}
		})
		floatSlice := make([]float32, 0)
		floatSlice = append(floatSlice, 1.2, 3.4, 2.4)
	    
		/*
		   具体操作:使用匿名函数封装自定义操作
		*/
		Process(floatSlice, func(elem interface{}) {
	
			if slice, ok := elem.([]float32); ok {
				for index, value := range slice {
					slice[index] = value * 2
				}
			}
		})
		fmt.Println(floatSlice) //[2.4 6.8 4.8]
	}

输出结果:

	[4 3 2 1]	
	[1 2 3 4]
	index: 0   value: hello
	index: 1   value: world
	index: 2   value: china
	[2.4 6.8 4.8]

在上面的例子中，Process函数负责对切片(数组）数据进行操作，在操作切片(数组)时候，首先要做一些参数检测，例如指针是否为空、数组长度是否大于0等。这些是操作数据的公共操作。具体针对数据可以有自己特殊的操作，包括排序(升序、降序）、输出等。针对这些特殊的操作可以使用函数对象来进行封装。    
再看下面的例子，这个例子没什么实际意义，只是为了说明闭包的使用方式。


	package main
	
	import (
		"fmt"
	)
	
	type FilterFunc func(ele interface{}) interface{}
	
	/*
	  公共操作:对数据进行特殊操作
	*/
	func Data(arr interface{}, filterFunc FilterFunc) interface{} {
	
		slice := make([]int, 0)
		array, _ := arr.([]int)

		for _, value := range array {
	
			integer, ok := filterFunc(value).(int)
			if ok {
				slice = append(slice, integer)
			}
	
		}
		return slice
	}
	/*
	  具体操作:奇数变偶数（这里可以不使用接口类型,直接使用int类型)
    */
	func EvenFilter(ele interface{}) interface{} {
	
		integer, ok := ele.(int)
		if ok {
			if integer%2 == 1 {
				integer = integer + 1
			}
		}
		return integer
	}
	/*
	  具体操作:偶数变奇数（这里可以不使用接口类型,直接使用int类型)
    */
	func OddFilter(ele interface{}) interface{} {
	
		integer, ok := ele.(int)
	
		if ok {
			if integer%2 != 1 {
				integer = integer + 1
			}
		}
	
		return integer
	}

	func main() {
		sliceEven := make([]int, 0)
		sliceEven = append(sliceEven, 1, 2, 3, 4, 5)
		sliceEven = Data(sliceEven, EvenFilter).([]int)
		fmt.Println(sliceEven) //[2 2 4 4 6]
	
		sliceOdd := make([]int, 0)
		sliceOdd = append(sliceOdd, 1, 2, 3, 4, 5)
		sliceOdd = Data(sliceOdd, OddFilter).([]int)
		fmt.Println(sliceOdd) //[1 3 3 5 5]
	
	}
输出结果: 
  
	[2 2 4 4 6]  
	[1 3 3 5 5]

Data作为公共函数，然后分别定义了两个具体的特殊函数:偶数和奇数的过滤器，定义具体的操作。


## 四 总结
上面例子中闭包的使用有点类似于面向对象设计模式中的模版模式，在模版模式中是在父类中定义公共的行为执行序列，然后子类通过重载父类的方法来实现特定的操作，而在Go语言中我们使用闭包实现了同样的效果。  
其实理解闭包最方便的方法就是将闭包函数看成一个类，一个闭包函数调用就是实例化一个类（在Objective-c中闭包就是用类来实现的），然后就可以从类的角度看出哪些是“全局变量”，哪些是“局部变量”。例如在第一个例子中，pos和neg分别实例化了两个“闭包类”，在这个“闭包类”中有个“闭包全局变量”sum。所以这样就很好理解返回的结果了。 


参考:  
[http://www.ibm.com/developerworks/cn/linux/l-cn-closure/index.html](http://www.ibm.com/developerworks/cn/linux/l-cn-closure/index.html "http://www.ibm.com/developerworks/cn/linux/l-cn-closure/index.html")  
[http://www.cnblogs.com/kym/archive/2011/03/07/1976519.html](http://www.cnblogs.com/kym/archive/2011/03/07/1976519.html "http://www.cnblogs.com/kym/archive/2011/03/07/1976519.html")
