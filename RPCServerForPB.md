	
    //
	//  rpcserver.go
	//  golang
	//  
	//  Created by zhangzhe on 2014-3-25
	//

	package rpcserver

	import (
		protorpc "code.google.com/p/protorpc"
		"fmt"
		"log"
		"net"
		"net/rpc"
		"os"
	)

	type Server struct {
		Address  string
		Network  string
		Services map[string]interface{}
	}

	/************************************
	函数描述::
  	 	用于创建Server的工厂方法

	参数说明:
    	 netwok  "TCP" 或 "UDP"
	 	addr    设置 Server IP
	***********************************/
	func NewServer(network string, addr string) *Server {

		rpcserver := new(Server)
		rpcserver.init(network, addr)

		return rpcserver
	}

	/************************************
	函数描述::
    	 初始化Server
	***********************************/
	func (this *Server) init(network string, addr string) {

		if this.Services == nil {
			this.Network = network
			this.Address = addr
			this.Services = make(map[string]interface{})
		}
	}

	/***********************************
	函数描述::
   		 为Server添加服务

	参数说明:
     	serverName  服务名称
	 	server      服务实体
	***********************************/
	func (this *Server) AddService(serverName string, server interface{}) {

		_, ok := this.Services[serverName]
		if ok { //serverName已经存在
			return
		}

		this.Services[serverName] = server
	}

	/***********************************
	函数描述::
     	启动服务
	***********************************/
	func (this *Server) StartServer() error {
		
		if len(this.Services) == 0 {
			return errors.New("No Registed Service.")
		}
		lis, err := net.Listen(this.Network, this.Address)

		if err != nil {
			return err
		}
		defer lis.Close()

		srv := rpc.NewServer()

		//注册服务
		for key, value := range this.Services {
			if err := srv.RegisterName(key, value); err != nil {
				return err
			}
		}

		for {
			conn, err := lis.Accept()
			if err != nil {
				log.Fatalf("lis.Accept(): %v\n", err)
			}
			go srv.ServeCodec(protorpc.NewServerCodec(conn))
		}
	}
	func (this *Server) checkError(err error) {
		if err != nil {
			fmt.Println("error:", err.Error())
			os.Exit(1)
		}
	}
