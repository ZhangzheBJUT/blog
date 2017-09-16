## 一 接口概述
    
如果说gorountine和channel是支撑起Go语言的并发模型的基石，让Go语言在如今集群化与多核化的时代成为一道亮丽的风景，那么接口是Go语言整个类型系列的基石，让Go语言在基础编程哲学的探索上达到前所未有的高度。    

Go语言在编程哲学上是变革派，而不是改良派。这不是因为Go语言有gorountine和channel，而更重要的是因为Go语言的类型系统，更是因为Go语言的接口。Go语言的编程哲学因为有接口而趋于完美。
    
`C++`,`Java` 使用"侵入式"接口，主要表现在实现类需要明确声明自己实现了某个接口。这种强制性的接口继承方式是面向对象编程思想发展过程中一个遭受相当多质疑的特性。  
**Go语言采用的是“非侵入式接口",Go语言的接口有其独到之处：只要类型T的公开方法完全满足接口I的要求，就可以把类型T的对象用在需要接口I的地方，所谓类型T的公开方法完全满足接口I的要求，也即是类型T实现了接口I所规定的一组成员。**这种做法的学名叫做 `Structural Typing`，有人也把它看作是一种`静态的Duck Typing`。

**Go 是静态类型的。每一个变量有一个静态的类型**，也就是说，有一个已知类型并且在编译时就确定下来了

	type MyInt int
	var i int
	var j MyInt

那么 i 的类型为 int 而 j 的类型为 MyInt。即使变量 i 和 j 有相同的底层类型，它们仍然是有不同的静态类型的。未经转换是不能相互赋值的。


## 二 接口类型
**在类型中有一个重要的类别就是接口类型，表达了固定的一个方法集合。**一个接口变量可以存储任意实际值（非接口），只要这个值实现了接口的方法。

	type Reader interface {
    	Read(p []byte) (n int, err os.Error)
	}
 
	// Writer 是包裹了基础 Write 方法的接口。
	type Writer interface {
		Write(p []byte) (n int, err os.Error)
	}

	var r io.Reader
	r = os.Stdin
	r = bufio.NewReader(r)
	r = new(bytes.Buffer)


**有一个事情是一定要明确的，不论 r 保存了什么值，r 的类型总是 io.Reader,Go 是静态类型，而 r 的静态类型是 io.Reader。**


**接口类型的一个极端重要的例子是空接口：**interface{}****,**它表示空的方法集合，由于任何值都有零个或者多个方法，所以任何值都可以满足它。**

也有人说 Go 的接口是动态类型的，不过这是一种误解。
它们是静态类型的：接口类型的变量总是有着相同的静态类型，这个值总是满足空接口，只是存储在接口变量中的值运行时可能被改变。

对于所有这些都必须严谨的对待，因为反射和接口密切相关。


## 三 接口的特色

接口类型的变量存储了两个内容：赋值给变量实际的值和这个值的类型描述。更准确的说，值是底层实现了接口的实际数据项目，而类型描述了这个项目完整的类型。例如下面，

	var r io.Reader
	tty, err = os.OpenFile("/dev/tty", os.O_RDWR, 0)
	if err != nil { return nil, err }
	r = tty

用模式的形式来表达 r 包含了的是** (value, type)** 对，如 (tty, *os.File)。      
**注意: 类型 `*os.File` 除了 Read 方法还实现了其他方法：尽管接口值仅仅提供了访问 Read 方法的可能（即通过r 只能访问Read方法），但是内部包含了这个值的完整的类型信息（反射的依据）。**

这也就是为什么可以这样做：

	var w io.Writer
	w = r.(io.Writer) //接口查询

在这个赋值中的断言是一个类型断言：它断言了 r 内部的条目同时也实现了 io.Writer，因此可以赋值它到 w。在赋值之后，w 将会包含 (tty, *os.File)，跟在 r 中保存的一致。   
**接口的静态类型决定了哪个方法可以通过接口变量调用，即便内部实际的值可能有一个更大的方法集。**

接下来，可以这样做：

	var empty interface{}
	empty = w
而空接口值 e 也将包含同样的 (tty, *os.File)。这很方便：**空接口可以保存任何值同时保留关于那个值的所有信息。**

**注:这里无需类型断言，因为 w 肯定满足空接口的。在上面的个例子中，将一个值从 Reader 变为 Writer，由于 Writer 的方法不是 Reader 的子集，所以就必须明确使用类型断言。**

**一个很重要的细节是接口内部的总是 (value, 实际类型) 的格式，而不会有 (value, 接口类型) 的格式。接口不能保存接口值。**


## 四 接口赋值

接口赋值在Go语言中分为两种情况:
1.将对象实例赋值给接口
2.将一个接口赋值给另外一个接口

* 将对象实例赋值给接口        
看下面的例子:    


		package main

		import (
		"fmt"
		)

		type LesssAdder interface {
			Less(b Integer) bool
			Add(b Integer)
		}

		type Integer int

		func (a Integer) Less(b Integer) bool {
			return a < b
		}

		func (a *Integer) Add(b Integer) {
			*a += b
		}

		func main() {

			var a Integer = 1
			var b LesssAdder = &a
			fmt.Println(b)
			
			//var c LesssAdder = a
			//Error:Integer does not implement LesssAdder 
			//(Add method has pointer receiver)
		}

go语言可以根据下面的函数:		

	func (a Integer) Less(b Integer) bool 
自动生成一个新的Less()方法   
 
	func (a *Integer) Less(b Integer) bool 
这样，类型*Integer就既存在Less()方法，也存在Add()方法，满足LessAdder接口。
而根据      

	func (a *Integer) Add(b Integer)】

这个函数无法生成以下成员方法：

	func(a Integer) Add(b Integer) {
	    （&a).Add(b)
	}
因为(&a).Add()改变的只是函数参数a,对外部实际要操作的对象并无影响(值传递)，这不符合用户的预期。所以Go语言不会自动为其生成该函数。因此类型Integer只存在Less()方法，缺少Add()方法，不满足LessAddr接口。（可以这样去理解：指针类型的对象函数是可读可写的，非指针类型的对象函数是只读的）

*	 将一个接口赋值给另外一个接口   
	在Go语言中，只要两个接口拥有相同的方法列表(次序不同不要紧),那么它们就等同的，可以相互赋值。
    如果A接口的方法列表时接口B的方法列表的子集，那么接口B可以赋值给接口A，但是反过来是不行的，无法通过编译。

## 五 接口查询

接口查询是否成功，要在运行期才能够确定。他不像接口的赋值，编译器只需要通过静态类型检查即可判断赋值是否可行。

	var file1  Writer = ...
	if file5,ok := file1.(two.IStream);ok {
	...
	}
这个if语句检查file1接口指向的对象实例是否实现了two.IStream接口，如果实现了，则执行特定的代码。

在Go语言中，你可以询问它指向的对象是否是某个类型，比如，

	var file1 Writer = ...
	if file6,ok := file1.(*File);ok {
	...
	}
这个if语句判断file1接口指向的对象实例是否是*File类型，如果是则执行特定的代码。

    slice := make([]int, 0)
	slice = append(slice, 1, 2, 3)

	var I interface{} = slice

	
	if res, ok := I.([]int)；ok {
		fmt.Println(res) //[1 2 3]
	}
这个if语句判断接口I所指向的对象是否是[]int类型，如果是的话输出切片中的元素。

	func Sort(array interface{}, traveser Traveser) error {
	
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

通过使用.(type)方法可以利用switch来判断接口存储的类型。   

**小结:**

**查询接口所指向的对象是否为某个类型的这种用法可以认为是接口查询的一个特例。** 接口是对一组类型的公共特性的抽象，所以查询接口与查询具体类型区别好比是下面这两句问话的区别:
> 你是医生么？  
> 是。  
> 你是莫莫莫  
> 是

第一句问话查询的是一个群体，是查询接口；而第二个问句已经到了具体的个体，是查询具体类型。

除此之外利用反射也可以进行类型查询，会在反射中做详细介绍。


参考：   
 
* [http://mikespook.com/2011/09/%E5%8F%8D%E5%B0%84%E7%9A%84%E8%A7%84%E5%88%99/](http://mikespook.com/2011/09/%E5%8F%8D%E5%B0%84%E7%9A%84%E8%A7%84%E5%88%99/ "http://mikespook.com/2011/09/%E5%8F%8D%E5%B0%84%E7%9A%84%E8%A7%84%E5%88%99/")    
* [http://research.swtch.com/interfaces](http://research.swtch.com/interfaces "http://research.swtch.com/interfaces")
