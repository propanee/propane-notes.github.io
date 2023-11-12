# RPC

## 原理



[Go RPC](https://www.jianshu.com/p/5ade587dbc58)**<=** **该小节简单讲解了RPC，这里推荐这篇文章。或者可以去了解下gRPC。**

Goal: RPC ![img](data:image/svg+xml;utf8,%3Csvg%20xmlns%3Axlink%3D%22http%3A%2F%2Fwww.w3.org%2F1999%2Fxlink%22%20width%3D%221.808ex%22%20height%3D%221.509ex%22%20style%3D%22vertical-align%3A%200.125ex%3B%20margin-bottom%3A%20-0.297ex%3B%22%20viewBox%3D%220%20-576.1%20778.5%20649.8%22%20role%3D%22img%22%20focusable%3D%22false%22%20xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%20aria-labelledby%3D%22MathJax-SVG-1-Title%22%3E%0A%3Ctitle%20id%3D%22MathJax-SVG-1-Title%22%3EEquation%3C%2Ftitle%3E%0A%3Cdefs%20aria-hidden%3D%22true%22%3E%0A%3Cpath%20stroke-width%3D%221%22%20id%3D%22E1-MJMAIN-2248%22%20d%3D%22M55%20319Q55%20360%2072%20393T114%20444T163%20472T205%20482Q207%20482%20213%20482T223%20483Q262%20483%20296%20468T393%20413L443%20381Q502%20346%20553%20346Q609%20346%20649%20375T694%20454Q694%20465%20698%20474T708%20483Q722%20483%20722%20452Q722%20386%20675%20338T555%20289Q514%20289%20468%20310T388%20357T308%20404T224%20426Q164%20426%20125%20393T83%20318Q81%20289%2069%20289Q55%20289%2055%20319ZM55%2085Q55%20126%2072%20159T114%20210T163%20238T205%20248Q207%20248%20213%20248T223%20249Q262%20249%20296%20234T393%20179L443%20147Q502%20112%20553%20112Q609%20112%20649%20141T694%20220Q694%20249%20708%20249T722%20217Q722%20153%20675%20104T555%2055Q514%2055%20468%2076T388%20123T308%20170T224%20192Q164%20192%20125%20159T83%2084Q80%2055%2069%2055Q55%2055%2055%2085Z%22%3E%3C%2Fpath%3E%0A%3C%2Fdefs%3E%0A%3Cg%20stroke%3D%22currentColor%22%20fill%3D%22currentColor%22%20stroke-width%3D%220%22%20transform%3D%22matrix(1%200%200%20-1%200%200)%22%20aria-hidden%3D%22true%22%3E%0A%20%3Cuse%20xlink%3Ahref%3D%22%23E1-MJMAIN-2248%22%20x%3D%220%22%20y%3D%220%22%3E%3C%2Fuse%3E%0A%3C%2Fg%3E%0A%3C%2Fsvg%3E)(local)PC

使得RPC在编程使用方面和栈上本地过程调用无差异。比如gRPC等实现，看上去Client和Server都像是在调用本地方法，对开发者隐藏了底层的网络通信环节流程。

<img src="https://cdn.nlark.com/yuque/0/2023/png/38520300/1691891070092-7c76c899-b8eb-4472-8d70-0ea83dcd0b3f.png" alt="img" style="zoom: 33%;" />

<img src="https://cdn.nlark.com/yuque/0/2023/png/38520300/1691891606088-9e4b9687-083a-4777-ad06-81b638dd3d95.png" alt="img" style="zoom:33%;" />

<img src="https://cdn.nlark.com/yuque/0/2023/png/38520300/1691892086203-9349b0a0-8a43-413d-8422-af3d073bedf1.png" alt="img" style="zoom:33%;" />

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1029%20rpc%E5%8E%9F%E7%90%86.png" alt="image-20230804131507445" style="zoom:50%;" />

客户端存根（client stub）

服务端存根（server stub）

1. 客户端调用客户端stub（client stub）。这个调用是在本地，并将调用参数push到栈（stack）中。
2. 客户端存根将方法、参数等打包编码marshalling（消息体对象序列化为二进制）
3. 客户端通过sockets发送消息到服务器；
4. 服务器系统将信息发送到服务器存根(Server Stub)；
5.  服务端存根解析信息，也就是解码；
6. 服务端存根调用真正的服务端程序(Server)；
7. 服务端(Server) 处理后，通过同样的方式，把结果返回给客户端(Client)（**the same stub**）。



RPC semantics under failures (RPC失败时的语义) ：

- at-least-once：至少执行一次（客户端自动重试，问题：同一操作可能被执行多次）
- at-most-once：至多执行一次，即重复请求不再处理

- （过滤重复duplicate，检测发送多次的请求，并只执行一次）
- Go 的RPC就是这样

- Exactly-once：正好执行一次，很难做到

## rpc库

https://www.liwenzhou.com/posts/Go/rpc/

Go RPC 库提供了通过网络访问一个对象方法的能力，服务器需要注册对象， 通过对象的类型名暴露这个服务。注册后这个对象的输出方法就可以远程调用，这个库封装了底层传输的细节，包括序列化（默认 GOB 序列化器）。

服务器可以注册多个不同类型的对象，但是注册相同类型的多个对象的时候会出错。

同时，如果对象的方法要能远程访问，它们必须满足一定的条件，否则这个对象的方法会被忽略，这些条件为以下内容：

- 首先，方法必须是导出的（名字首字母大写）；

- 其次，方法接受两个参数，必须是导出的或内置类型。第一个参数表示客户端传递过来的请求参数，第二个是需要返回给客户端的响应。第二个参数必须为指针类型（需要修改）；
- 最后，方法必须返回一个`error`类型的值。返回非`nil`的值，表示调用出错。

```go
func (t *T) MethodName(argType T1, replyType *T2) error
```

这个方法的第一个参数代表调用者（client）提供的参数，第二个参数代表要返回给调用者的计算结果

- 方法的返回值如果不为空， 则它作为一个字符串返回给调用者；
- 如果返回 error ，则 reply 参数不会返回给调用者；
- 这里的 T、T1、T2 能够被 encoding/gob 序列化，即使使用其它的序列化框架。

1. 服务器通过调用 ServeConn 在一个连接上处理请求，更典型地， 它可以创建一个 network listener 然后 accept 请求。
   对于 HTTP listener 来说，可以调用 HandleHTTP() 方法 和 http.Serve() 函数 。

2. 客户端可以调用 Dial 和 DialHTTP 建立连接， 客户端有两个方法调用服务（Call 和 Go），可以同步地或者异步地调用服务，调用服务时需要把服务名、方法名和参数传递给服务器。异步方法调用 Go 通过 Done channel 通知调用结果返回。

示例：

- 定义rpc相关接口传入参数和返回参数的数据结构；

- 定义一个服务对象，以及被调用方法
- 实现 RPC 服务器，使用 rpc.Register() 函数注册这个服务，通过调用 net.Listen() 方法来监听对应的 socket 并对外提供服务，具体的代码如下所示：
- 编写客户端程序来远程调用定义的服务，

```go
// mr.rpc
// 定义传入参数和返回参数的数据结构
type ExampleArgs struct {
	X int
}
type ExampleReply struct {
	Y int
}
// Cook up a unique-ish UNIX-domain socket name
// in /var/tmp, for the coordinator.
// Can't use the current directory since
// Athena AFS doesn't support UNIX-domain sockets.
func coordinatorSock() string {
	s := "/var/tmp/5840-mr-"
	s += strconv.Itoa(os.Getuid())
	return s
}
```

```go
// mr.coordinator
// 定义服务对象
type Coordinator struct {}
// 实现被调用方法
func (c *Coordinator) Example(args *ExampleArgs, reply *ExampleReply) error {
	reply.Y = args.X + 1
	return nil
}
// 实现服务器
func (c *Coordinator) server() {
	rpc.Register(c)
	rpc.HandleHTTP()
	//l, e := net.Listen("tcp", ":1234")
	sockname := coordinatorSock()
	os.Remove(sockname)
	l, e := net.Listen("unix", sockname)
	if e != nil {
		log.Fatal("listen error:", e)
	}
	go http.Serve(l, nil)
}
```

```go
// mr.worker
// 调用
func CallExample() {

	// declare an argument structure.
	args := ExampleArgs{}

	// fill in the argument(s).
	args.X = 99

	// declare a reply structure.
	reply := ExampleReply{}

	// send the RPC request, wait for the reply.
	// the "Coordinator.Example" tells the
	// receiving server that we'd like to call
	// the Example() method of struct Coordinator.
	ok := call("Coordinator.Example", &args, &reply)
	if ok {
		// reply.Y should be 100.
		fmt.Printf("reply.Y %v\n", reply.Y)
	} else {
		fmt.Printf("call failed!\n")
	}
}

// send an RPC request to the coordinator, wait for the response.
// usually returns true.
// returns false if something goes wrong.
func call(rpcname string, args interface{}, reply interface{}) bool {
	// c, err := rpc.DialHTTP("tcp", "127.0.0.1"+":1234")
	sockname := coordinatorSock()
	c, err := rpc.DialHTTP("unix", sockname)
	if err != nil {
		log.Fatal("dialing:", err)
	}
	defer c.Close()

	err = c.Call(rpcname, args, reply)
	if err == nil {
		return true
	}

	fmt.Println(err)
	return false
}
```



### rpc.Register() 

rpc.Register() 函数调用会将对象类型中所有满足 RPC 规则的对象方法注册为 RPC 函数。

### rpc.HandleHTTP()

`rpc.HandleHTTP()`注册 HTTP 路由。

```go
rpc.Register(arith)
rpc.HandleHTTP()
if err := http.ListenAndServe(":1234", nil); err != nil {
	log.Fatal("serve error:", err)
}
```
