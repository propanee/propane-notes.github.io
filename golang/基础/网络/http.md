# HTTP

[一文读懂 Go Http Server 原理](https://blog.csdn.net/RA681t58CJxsgCkJ31/article/details/128660811)

[Golang HTTP 标准库实现原理](https://mp.weixin.qq.com/s?__biz=MzkxMjQzMjA0OQ==&mid=2247484040&idx=1&sn=b710f4429188ea5f49f6a9155381b67f)

## http request method ![img](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/0929http%20request.png)



## http.HandleFunc

```go
func HandleFunc(pattern string, handler func(ResponseWriter, *Request))
```

注册一个处理器函数handler和对应的模式pattern（注册到DefaultServeMux）

第一个参数指的是请求路径，第二个参数是一个函数类型，表示这个请求需要处理的事情。没有处理复杂的逻辑，而是直接给DefaultServeMux处理，如源码：

```go
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
   DefaultServeMux.HandleFunc(pattern, handler)
}
```

DefaultServeMux 是ServeMux一个全局实例