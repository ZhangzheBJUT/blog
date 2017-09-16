
## 一 接口与nil
前面讲解了go语言中接口的基本使用方法，下面将说一说nil在接口中的使用。  
从上面一节我们知道在底层，interface作为两个成员实现：一个类型和一个值。该值被称为接口的动态值， 它是一个任意的具体值，而该接口的类型则为该值的类型。对于 int 值3， 一个接口值示意性地包含(int, 3)。

只有在内部值和类型都未设置时(nil, nil)，一个接口的值才为 nil。特别是，一个 nil 接口将总是拥有一个 nil 类型。若我们在一个接口值中存储一个 int 类型的指针，则内部类型将为 int，无论该指针的值是什么，这样的接口值会是非 nil 的，即使在该指针的内部值为 nil，形如(*int, nil)。

下面是一个错误的错误返回方式：


	func returnsError() error {
	    var p *MyError = nil
	    if bad() {
	        p = ErrBad
	    }
	
	    return p // Will always return a non-nil error.
	}
这里 p 返回的是一个有效值（非nil），值为 nil。 

因此，下面判断错误的代码会有问题：

	func main() {
	    if err := returnsError(); err != nil {
	        panic(nil)
	    }
	}
上面的if判断永远为真，因为resturnsError返回的值的类型不为nil。

针对 returnsError 的问题，可以这样处理（不建议的方式）：

	func main() {
	    if err := returnsError(); err.(*MyError) != nil {
	        panic(nil)
	    }
	}
在判断前先将err转型为*MyError，然后再判断err的值。 
类似的C语言空字符串可以这样判断：


	bool IsEmptyStr(const char* str) {
	    return !(str && str[0] != '\0');
	}

但是Go语言中标准的错误返回方式不是returnsError这样。 
下面是改进的returnsError：

	func returnsError() error {
	
	    var p *MyError = nil
	    if bad() {
	        return nil
	    }
	    return p // Will always return a non-nil error.
	}

## 二 小结
  在处理错误返回值的时候，一定要将正常的错误值转换为 nil。

参考:   
[http://my.oschina.net/chai2010/blog/117923](http://my.oschina.net/chai2010/blog/117923 "http://my.oschina.net/chai2010/blog/117923")  
[http://golang.org/doc/faq#nil_error](http://golang.org/doc/faq#nil_error "http://golang.org/doc/faq#nil_error")
