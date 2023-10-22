# overview

goctl 定位是代码生成脚手架，其核心思想是通过对 DSL (领域专用语言？)进行语法解析、语义分析到数据驱动实现完成代码生成功能，旨在帮助开发者从工程效率角度提高开发者的开发效率。以代码生成为主，开发规范为辅，goctl 是围绕 zero-api 为中心做代码生成的。

- 提高开发效率：goctl 的代码生成覆盖了Go、Java、Android、iOS、TS 等多门语言，通过 goctl 可以一键生成各端代码，让开发者将精力集中在业务开发上，从而加快了开发效率且降低了代码出错率。
- 降低沟通成本：使用 zero-api 替代 API 文档，各端以 zero-api 为中心生成各端代码，无需再通过会议、API 文档来进行接口信息传递。
- 降低耦合度：前端可以利用 goctl 去生成适量的 mock 数据来进行 UI联调。

# 常用指令

## rpc

rpc服务代码生成模块，支持proto模板生成和rpc服务代码生成

```shell
//创建proto参考模板文件
goctl rpc template -o ums.proto
```

```shell
//读取当前目录下的ums.proto文件，生成rpc项目代码，生成到当前目录下
goctl rpc protoc ums.proto --go_out=. --go-grpc_out=. --zrpc_out=.
```

## api
goctl api是goctl中的核心模块之一，其可以通过.api文件一键快速生成一个api服务。

在传统的api项目中，我们要创建各级目录，编写结构体， 定义路由，添加logic文件，这一系列操作，如果按照一条协议的业务需求计算，整个编码下来大概需要5～6分钟才能真正进入业务逻辑的编写。 而goctl api则可以完全替代你去做这一部分工作，不管你的协议要定多少个，最终来说，只需要花费10秒不到即可完成。

```shell
//在文件夹desc下创建一个index.api的模板文件
goctl api -o ./desc/index.api
//读取desc/index.api文件，生成api项目代码，生成到当前目录下
goctl api go -api ./desc/index.api -dir .
```

| **子命令选项** | **说明**                                               |
| -------------------- | ------------------------------------------------------------ |
| new                  | 在当前目录下，快速创建一个api服务                            |
| -o value             | 快速初始化一个api文件 value:为需要创建的api文件路径加名称  例：./desc/index.api |
| go            | 生成go代码的api服务                       |
| go -api value | 根据对应api文件生成 value:api源文件（路径加名称） |
| go -dir value | 生成的go代码目录 value:目录                       |

### api生成swagger文件

api项目开发中通常都需要与前端交互对接，此时有个文档会方便开发，goctl也带swagger功能，可直接生成swagger文件

```shell 
//读取当前目录下的index.api文件，根据swagger规则，生成api.json文件，生成到当前目录下
goctl api plugin -plugin goctl-swagger="swagger -filename api.json" -api index.api -dir .
```

## template

模板（Template）是数据驱动生成的基础，所有的代码（rest api、rpc、model、docker、kube）生成都会依赖模板， 默认情况下，模板生成器会选择内存中的模板进行生成，而对于有模板修改需求的开发者来讲，则需要将模板进行落盘， 从而进行模板修改，在下次代码生成时会加载指定路径下的模板进行生成。

```shell
//根据当前goctl版本，下载所有默认模板文件到goctl文件夹下
goctl template init --home=./goctl
```

# api文件格式

```go
//版本信息，import中的版本信息必须与被import的api版本信息一样。
syntax = "v1" //表示这是 zero-api 的 v1 语法
//随着api文件中定义的结构体和服务增多，必然会面临文件拆分问题。import语法块就负责文件路径引入。
import "xxx.api"
import "x/xxx.api"
//或者：
import(
    "xxx.api"
    "x/xxx.api"
)

//info负责对api文件的描述。且每个api文件最多只能有1个info语法块。
info(
    author: "xxx"
    date: "2022-01-01"
    desc: "xxx-api文档"
)

//结构体，由go语言的结构体演化而来，与go语言的结构体语法一致。
type (
    // 请求的结构体
    LoginReq {
        Username string `json:"username"`
        Password string `json:"password"`
    }

    // 响应的结构体
    LoginReply {
        Id           int64  `json:"id"`
        Name         string `json:"name"`
        Gender       string `json:"gender"`
        AccessToken  string `json:"accessToken"`
        AccessExpire int64  `json:"accessExpire"`
        RefreshAfter int64  `json:"refreshAfter"`
    }
)
//service用于定义api服务，其中可以包含服务中的各种信息，包括名称、metadata、中间件生成、handler、路由等。
// user-api服务里的metadata，比如url的前缀和group群组
@server(
    prefix: v1/user        // url的前缀
    group: user            // 群组
)
// user-api服务的请求路由
service user-api {
    @handler login        // 处理函数
    post /user/login (LoginReq) returns (LoginReply)    // 方法与URL路径
}
```

### 服务定义

```go
@server (
   prefix: /v1/user // 这个服务下的API端点的url前缀，如/info的完整路径是/v1/user/info
   signature: true //对请求进行签名验证。增加API请求的安全性，确保请求没有被篡改。
   jwt: Auth //使用JWT（JSON Web Token）进行身份验证。
)
```

JWT：

JSON Web Token（JWT）是目前最流行的跨域身份验证解决方案

JWT的原则是在服务器身份验证之后，将生成一个JSON对象并将其发送回用户