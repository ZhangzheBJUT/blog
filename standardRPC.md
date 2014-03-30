**//  
//  main.go  
//  golang  
//
//  Created by zhangzhe on 2014-2-20
//**

###一 HTTP RPC

服务端代码
	
	package main

    import (
    "errors"
	"fmt"
	"net/http"
	"net/rpc"
	)

	const (
		URL = "192.168.2.172:12981"
	)

	type Args struct {
		A, B int
	}
	
	type Quotient struct {
		Quo, Rem int
	}

	type Arith int

	func (t *Arith) Multiply(args *Args, reply *int) error {
		*reply = args.A * args.B
		return nil
	}
	func (t *Arith) Divide(args *Args, quo *Quotient) error{
		if args.B == 0 {
			return errors.New("divide by zero!")
		}

		quo.Quo = args.A / args.B
		quo.Rem = args.A % args.B

		return nil
	}
	func main() {

		arith := new(Arith)
		rpc.Register(arith)
		rpc.HandleHTTP()

		err := http.ListenAndServe(URL, nil)
		if err != nil {
			fmt.Println(err.Error())
		}
	}


客户端代码

	package main
	
	import (
    	"fmt"    
    	"net/rpc”
    )
    
    const (
		URL = "192.168.2.172:12982"
	)
	
	func main() {
	
		client, err := rpc.DialHTTP("tcp", URL)
		if err != nil {
			fmt.Println(err.Error())
		}
		
		args := Args{2, 4}
		var reply int
		err = client.Call("Arith.Multiply", &args, &reply)

		if err != nil {
			fmt.Println(err.Error())
		} else {
			fmt.Println(reply)
		}
	}	
	
### 二 JSON-RPC	

	
服务器端代码
	
	package main

    import (
    "errors"
	"fmt"
	"net"
	"net/rpc"
	"net/rpc/jsonrpc"
	)

	const (
		URL= "192.168.2.172:12981"
	)

	type Args struct {
		A, B int
	}
	type Quotient struct {
		Quo, Rem int
	}

	type Arith int

	func (t *Arith) Multiply(args *Args, reply *int) error {
		*reply = args.A * args.B
		return nil
	}
	func (t *Arith) Divide(args *Args, quo *Quotient) error {
		if args.B == 0 {
			return errors.New("divide by zero!")
		}

		quo.Quo = args.A / args.B
		quo.Rem = args.A % args.B

		return nil
	}
	func main() {

		arith := new(Arith)
		rpc.Register(arith)
		
		tcpAddr, err := net.ResolveTCPAddr("tcp", URL)
		if err != nil {
			fmt.Println(err)
		}
		listener, err := net.ListenTCP("tcp", tcpAddr)

		for {
			conn, err := listener.Accept()
			if err != nil {
				continue
			}
			go jsonrpc.ServeConn(conn)
		}
	}
	
客户端代码

	package main
	
	import (
    	"fmt"    
    	"net/rpc”
    )
    
    const (
		URL = "192.168.2.172:12982"
	)
	
	func main() {
	
		client, err := jsonrpc.Dial("tcp", URL)
		defer client.Close()

		if err != nil {
			fmt.Println(err)
		}

		args := Args{7, 2}
		var reply int
		err = client.Call("Arith.Multiply", &args, &reply)
		if err != nil {
			fmt.Println(err)
		}
		fmt.Println(reply)	
	}
