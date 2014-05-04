###一 反射的规则
反射是程序运行时检查其所拥有的结构，尤其是类型的一种能力；这是元编程的一种形式。它同时也是造成混淆的重要来源。

每个语言的反射模型都不同（同时许多语言根本不支持反射）。本节将试图明确解释在 Go 中的反射是如何工作的。

###1. 从接口值到反射对象的反射    
**在基本的层面上，反射只是一个检查存储在接口变量中的类型和值的算法。**在 reflect 包中有两个类型需要了解：**Type** 和 **Value**。这两个类型使得可以访问接口变量的内容，还有两个简单的函数，**reflect.TypeOf** 和 **reflect.ValueOf，从接口值中分别获取 reflect.Type 和 reflect.Value。**（**注：从 reflect.Value 也很容易能够获得 reflect.Type，不过这里让 Value 和 Type 在概念上是分离的**）

从 TypeOf 开始：

	package main
 
	import (
       	 "fmt"
        "reflect"
	)
 
	func main() {
        var x float64 = 3.4
        fmt.Println("type:", reflect.TypeOf(x))
	}
这个程序打印 `type: float64`    
接口在哪里呢，读者可能会对此有疑虑，看起来程序传递了一个 float64 类型的变量 x，而不是一个接口值，到 reflect.TypeOf。但是，它确实就在那里：如同 godoc 报告的那样，reflect.TypeOf 的声明包含了空接口：

	// TypeOf 返回 interface{} 中的值反射的类型。
	   func TypeOf(i interface{}) Type
**当调用 reflect.TypeOf(x) 的时候，x 首先存储于一个作为参数传递的空接口中；reflect.TypeOf 解包这个空接口来还原类型信息。**

reflect.ValueOf 函数，当然就是还原那个值（从这里开始将会略过那些概念示例，而聚焦于可执行的代码）：

	var x float64 = 3.4
	fmt.Println("value:", reflect.ValueOf(x))
打印

	value: <float64 Value>
除了reflect.Type 和 reflect.Value外，都有许多方法用于检查和操作它们。一个重要的例子是 Value 有一个 Type 方法返回 reflect.Value 的 Type。另一个是 Type 和 Value 都有 Kind 方法返回一个常量来表示类型：Uint、Float64、Slice 等等。同样 Value 有叫做 Int 和 Float 的方法可以获取存储在内部的值（跟 int64 和 float64 一样）：

	var x float64 = 3.4
	v := reflect.ValueOf(x)
	fmt.Println("type:", v.Type())
	fmt.Println("kind is float64:", v.Kind() == reflect.Float64)
	fmt.Println("value:", v.Float())   				      
打印					
					             
	type: float64					    
	kind is float64: true				
	value: 3.4						
									
同时也有类似 SetInt 和 SetFloat 的方法，不过在使用它们之前需要理解可设置性，这部分的主题在下面的第三条军规中讨论。

反射库有着若干特性值得特别说明。



- 为了保持 API 的简洁，“获取者”和“设置者”用 Value 的最宽泛的类型来处理值：例如，int64 可用于所有带符号整数。也就是说 Value 的 Int 方法返回一个 int64，而 SetInt 值接受一个 int64；所以可能必须转换到实际的类型：

		var x uint8 = 'x'
		v := reflect.ValueOf(x)
		fmt.Println("type:", v.Type()) // uint8.
		fmt.Println("kind is uint8: ", v.Kind() == reflect.Uint8) // true.
		x = uint8(v.Uint()) // v.Uint 返回一个 uint64.

- 反射对象的 Kind 描述了底层类型，而不是静态类型。如果一个反射对象包含了用户定义的整数类型的值，就像	
					
		type MyInt int
		var x MyInt = 7
		v := reflect.ValueOf(x)‘

v 的 Kind 仍然是 reflect.Int，尽管 x 的静态类型是 MyInt，而不是 int。换句话说，Kind 无法从 MyInt 中区分 int，而 Type 可以。


###2. 从反射对象到接口值的反射

如同物理中的反射，在 Go 中的反射也存在它自己的镜像。

从 reflect.Value 可以使用 Interface 方法还原接口值；
此方法可以高效地打包类型和值信息到接口表达中，并返回这个结果：

	// Interface 以 interface{} 返回 v 的值。
	func (v Value) Interface() interface{}

可以这样作为结果

	y := v.Interface().(float64) // y 将为类型 float64。
	fmt.Println(y)

通过反射对象 v 可以打印 float64 的表达值。

然而，还可以做得更好。`fmt.Println`,`fmt.Printf` 等其他所有传递一个空接口值作为参数的函数，在 fmt 包内部解包的方式就像之前的例子这样。因此正确的打印 reflect.Value 的内容的方法就是将 Interface 方法的结果进行格式化打印(formatted print routine).

	fmt.Println(v.Interface())

为什么不是 fmt.Println(v)？因为 v 是一个 reflect.Value；这里希望获得的是它保存的实际的值。
   
由于值是 float64，如果需要的话，甚至可以使用浮点格式化：

	fmt.Printf("value is %7.1e\n", v.Interface())

输出: `3.4e+00`    
再次强调，**对于 v.Interface() 无需类型断言其为 float64；空接口值在内部有实际值的类型信息，而 Printf 会发现它。**    
简单来说，Interface 方法是 ValueOf 函数的镜像，除了返回值总是静态类型 interface{}。  
回顾：**反射可以从接口值到反射对象，也可以反过来。**

###3. 为了修改反射对象，其值必须可设置

	var x float64 = 3.4
	v := reflect.ValueOf(x)
	v.SetFloat(7.1) // Error: will panic.

如果运行这个代码，它报出神秘的 panic 消息

	panic: reflect.Value.SetFloat using unaddressable value

问题不在于值 7.1 不能地址化；在于 v 不可设置。**设置性是反射值的一个属性，并不是所有的反射值有此特性。**

Value的 CanSet 方法提供了值的设置性；在这个例子中，

	var x float64 = 3.4
	v := reflect.ValueOf(x)
	fmt.Println("settability of v:" , v.CanSet())
打印

	settability of v: false
对不可设置值调用 Set 方法会有错误。 
 
**但是什么是设置性？**   
设置性有一点点像地址化，但是更严格。这是用于创建反射对象的时候，能够修改实际存储的属性。设置性用于决定反射对象是否保存原始项目。当这样

	var x float64 = 3.4
	v := reflect.ValueOf(x)
就传递了一个 x 的副本到 reflect.ValueOf，所以接口值作为 reflect.ValueOf 参数创建了 x 的副本，而不是 x 本身。因此，如果语句

	v.SetFloat(7.1)
允许执行，虽然 v 看起来是从 x 创建的，它也无法更新 x。反之，如果在反射值内部允许更新 x 的副本，那么 x 本身不会收到影响。这会造成混淆，并且毫无意义，因此这是非法的，而设置性是用于解决这个问题的属性。

这很神奇？其实不是。这实际上是一个常见的非同寻常的情况。考虑传递 x 到函数：

f(x)
由于传递的是 x 的值的副本，而不是 x 本身，所以并不期望 f 可以修改 x。如果想要 f 直接修改 x，必须向函数传递 x 的地址（也就是，指向 x 的指针）：

f(&x)
这是清晰且熟悉的，而反射通过同样的途径工作。如果希望通过反射来修改 x，必须向反射库提供一个希望修改的值的指针。

来试试吧。首先像平常那样初始化 x，然后创建指向它的反射值，叫做 p。

	var x float64 = 3.4
	p := reflect.ValueOf(&x) // 注意：获取 X 的地址。
	fmt.Println("type of p:", p.Type())
	fmt.Println("settability of p:" , p.CanSet())
这样输出为

	type of p: *float64
	settability of p: false
反射对象 p 并不是可设置的，而且我们也不希望设置 p，实际上是 *p。为了获得 p 指向的内容，调用值上的 Elem 方法，从指针间接指向，然后保存反射值的结果叫做 v：

	v := p.Elem()
	fmt.Println("settability of v:" , v.CanSet())
现在 v 是可设置的反射对象，如同示例的输出，

	settability of v: true
而由于它来自 x，最终可以使用 v.SetFloat 来修改 x 的值：

	v.SetFloat(7.1)
	fmt.Println(v.Interface())
	fmt.Println(x)
得到期望的输出

	7.1
	7.1
反射可能很难理解，但是语言做了它应该做的，尽管底层的实现被反射的 Type 和 Value 隐藏了。务必记得反射值需要某些内容的地址来修改它指向的东西。

### 二结构体

在之前的例子中 v 本身不是指针，它只是从一个指针中获取的。这种情况更加常见的是当使用反射修改结构体的字段的时候。也就是当有结构体的地址的时候，可以修改它的字段。

这里有一个分析结构值 t 的简单例子。由于希望对结构体进行修改，所以从它的地址创建了反射对象。设置了 typeOfT 为其类型，然后用直白的方法调用来遍历其字段（参考 reflect 包了解更多信息）。注意从结构类型中解析了字段名字，但是字段本身是原始的 reflect.Value 对象。

	type T struct {
  	  A int
  	  B string
	}
	t := T{23, "skidoo"}
	s := reflect.ValueOf(&t).Elem()
	typeOfT := s.Type()
	for i := 0; i < s.NumField(); i++ {
   	 	f := s.Field(i)
    	fmt.Printf("%d: %s %s = %v\n", i,
        typeOfT.Field(i).Name, f.Type(), f.Interface())
	}
程序输出:

	0: A int = 23
	1: B string = skidoo
还有一个关于设置性的要点：**T 的字段名要大写（可导出），因为只有可导出的字段是可设置的。**

由于 s 包含可设置的反射对象，所以可以修改结构体的字段。

	s.Field(0).SetInt(77)
	s.Field(1).SetString("Sunset Strip")
	fmt.Println("t is now", t)
这里是结果：

	t is now {77 Sunset Strip}
如果修改程序使得 s 创建于 t，而不是 &t，调用 SetInt 和 SetString 会失败，因为 t 的字段不可设置。

###三 总结

反射的规则如下：    
从接口值到反射对象的反射    
从反射对象到接口值的反射    
为了修改反射对象，其值必须可设置     
一旦理解了 Go 中的反射的这些规则，就会变得容易使用了，虽然它仍然很微妙。这是一个强大的工具，除非真得有必要，否则应当避免使用或小心使用。

还有大量的关于反射的内容没有涉及到——channel 上的发送和接收、分配内存、使用 slice 和 map、调用方法和函数。

事例代码：
[https://github.com/ZhangzheBJUT/GoProject/blob/master/reflect/main.go](https://github.com/ZhangzheBJUT/GoProject/blob/master/reflect/main.go "https://github.com/ZhangzheBJUT/GoProject/blob/master/reflect/main.go")

参考:  
[http://blog.golang.org/laws-of-reflection](http://blog.golang.org/laws-of-reflection "http://blog.golang.org/laws-of-reflection")