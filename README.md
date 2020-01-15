# 7天用Go从零实现Web框架Gee教程

转载自原作者：https://geektutu.com/post/gee.html  github：https://github.com/geektutu/7days-golang

在原作者的基础上做了一些小修改，算练手吧，具体参考后面的 `扩展` 章节。

**目录**

[TOC]

## 设计一个框架
大部分时候，我们需要实现一个 Web 应用，第一反应是应该使用哪个框架。不同的框架设计理念和提供的功能有很大的差别。比如 Python 语言的 django和flask，前者大而全，后者小而美。Go语言/golang 也是如此，新框架层出不穷，比如Beego，Gin，Iris等。那为什么不直接使用标准库，而必须使用框架呢？在设计一个框架之前，我们需要回答框架核心为我们解决了什么问题。只有理解了这一点，才能想明白我们需要在框架中实现什么功能。

我们先看看标准库net/http如何处理一个请求。
```go
func main() {
    http.HandleFunc("/", handler)
    http.HandleFunc("/count", counter)
    log.Fatal(http.ListenAndServe("localhost:8000", nil))
}

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "URL.Path = %q\n", r.URL.Path)
}
```
net/http提供了基础的Web功能，即监听端口，映射静态路由，解析HTTP报文。一些Web开发中简单的需求并不支持，需要手工实现。
- 动态路由：例如hello/:name，hello/*这类的规则。
- 鉴权：没有分组/统一鉴权的能力，需要在每个路由映射的handler中实现。
- 模板：没有统一简化的HTML机制。
- …

当我们离开框架，使用基础库时，需要频繁手工处理的地方，就是框架的价值所在。但并不是每一个频繁处理的地方都适合在框架中完成。Python有一个很著名的Web框架，名叫bottle，整个框架由bottle.py一个文件构成，共4400行，可以说是一个微框架。那么理解这个微框架提供的特性，可以帮助我们理解框架的核心能力。
- 路由(Routing)：将请求映射到函数，支持动态路由。例如'/hello/:name。
- 模板(Templates)：使用内置模板引擎提供模板渲染机制。
- 工具集(Utilites)：提供对 cookies，headers 等处理机制。
- 插件(Plugin)：Bottle本身功能有限，但提供了插件机制。可以选择安装到全局，也可以只针对某几个路由生效。
- …

## Gee 框架
这个教程将使用 Go 语言实现一个简单的 Web 框架，起名叫做 Gee，geektutu.com 的前三个字母。我第一次接触的 Go 语言的 Web 框架是 Gin，Gin 的代码总共是14K，其中测试代码9K，也就是说实际代码量只有5K。Gin 也是我非常喜欢的一个框架，与Python 中的 Flask 很像，小而美。

7天实现 Gee 框架这个教程的很多设计，包括源码，参考了 Gin，大家可以看到很多 Gin 的影子。

时间关系，同时为了尽可能地简洁明了，这个框架中的很多部分实现的功能都很简单，但是尽可能地体现一个框架核心的设计原则。例如 Router 的设计，虽然支持的动态路由规则有限，但为了性能考虑匹配算法是用 Trie 树实现的，Router最重要的指标之一便是性能。

# Go语言动手写Web框架 - Gee第一天 http.Handler
本文是 7天用Go从零实现Web框架Gee教程系列的第一篇。

- 简单介绍net/http库以及http.Handler接口。
- 搭建Gee框架的雏形，代码约50行。

## 标准库启动Web服务
Go语言内置了 net/http库，封装了HTTP网络编程的基础的接口，我们实现的Gee Web 框架便是基于net/http的。我们接下来通过一个例子，简单介绍下这个库的使用。

day1-http-base/base1/main.go
```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/", indexHandler)
	http.HandleFunc("/hello", helloHandler)
	log.Fatal(http.ListenAndServe(":9999", nil))
}

// handler echoes r.URL.Path
func indexHandler(w http.ResponseWriter, req *http.Request) {
	fmt.Fprintf(w, "URL.Path = %q\n", req.URL.Path)
}

// handler echoes r.URL.Header
func helloHandler(w http.ResponseWriter, req *http.Request) {
	for k, v := range req.Header {
		fmt.Fprintf(w, "Header[%q] = %q\n", k, v)
	}
}
```

我们设置了2个路由，/和/hello，分别绑定 indexHandler 和 helloHandler ， 根据不同的HTTP请求会调用不同的处理函数。访问/，响应是URL.Path = /，而/hello的响应则是请求头(header)中的键值对信息。

用 curl 这个工具测试一下，将会得到如下的结果。
```
$ curl http://localhost:9999/
URL.Path = "/"
$ curl http://localhost:9999/hello
Header["Accept"] = ["*/*"]
Header["User-Agent"] = ["curl/7.54.0"]
```

main 函数的最后一行，是用来启动 Web 服务的，第一个参数是地址，:9999表示在 9999 端口监听。而第二个参数则代表处理所有的HTTP请求的实例，nil 代表使用标准库中的实例处理。第二个参数，则是我们基于net/http标准库实现Web框架的入口。

## 实现http.Handler接口
```
package http

type Handler interface {
    ServeHTTP(w ResponseWriter, r *Request)
}

func ListenAndServe(address string, h Handler) error
```

第二个参数的类型是什么呢？通过查看net/http的源码可以发现，Handler是一个接口，需要实现方法 ServeHTTP ，也就是说，只要传入任何实现了 ServerHTTP 接口的实例，所有的HTTP请求，就都交给了该实例处理了。马上来试一试吧。

day1-http-base/base2/main.go
```
package main

import (
	"fmt"
	"log"
	"net/http"
)

// Engine is the uni handler for all requests
type Engine struct{}

func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	switch req.URL.Path {
	case "/":
		fmt.Fprintf(w, "URL.Path = %q\n", req.URL.Path)
	case "/hello":
		for k, v := range req.Header {
			fmt.Fprintf(w, "Header[%q] = %q\n", k, v)
		}
	default:
		fmt.Fprintf(w, "404 NOT FOUND: %s\n", req.URL)
	}
}

func main() {
	engine := new(Engine)
	log.Fatal(http.ListenAndServe(":9999", engine))
}
```

- 我们定义了一个空的结构体Engine，实现了方法ServeHTTP。这个方法有2个参数，第二个参数是 Request ，该对象包含了该HTTP请求的所有的信息，比如请求地址、Header和Body等信息；第一个参数是 ResponseWriter ，利用 ResponseWriter 可以构造针对该请求的响应。

- 在 main 函数中，我们给 ListenAndServe 方法的第二个参数传入了刚才创建的engine实例。至此，我们走出了实现Web框架的第一步，即，将所有的HTTP请求转向了我们自己的处理逻辑。还记得吗，在实现Engine之前，我们调用 http.HandleFunc 实现了路由和Handler的映射，也就是只能针对具体的路由写处理逻辑。比如/hello。但是在实现Engine之后，我们拦截了所有的HTTP请求，拥有了统一的控制入口。在这里我们可以自由定义路由映射的规则，也可以统一添加一些处理逻辑，例如日志、异常处理等。

- 代码的运行结果与之前的是一致的。

## Gee框架的雏形

我们接下来重新组织上面的代码，搭建出整个框架的雏形。

最终的代码目录结构是这样的。

```
gee/
  |--gee.go
main.go
```

day1-http-base/base3/main.go
```
package main

import (
	"fmt"
	"net/http"

	"./gee"
)

func main() {
	r := gee.New()
	r.GET("/", func(w http.ResponseWriter, req *http.Request) {
		fmt.Fprintf(w, "URL.Path = %q\n", req.URL.Path)
	})

	r.GET("/hello", func(w http.ResponseWriter, req *http.Request) {
		for k, v := range req.Header {
			fmt.Fprintf(w, "Header[%q] = %q\n", k, v)
		}
	})

	r.Run(":9999")
}
```

看到这里，如果你使用过gin框架的话，肯定会觉得无比的亲切。gee框架的设计以及API均参考了gin。使用New()创建 gee 的实例，使用 GET()方法添加路由，最后使用Run()启动Web服务。这里的路由，只是静态路由，不支持/hello/:name这样的动态路由，动态路由我们将在下一次实现。

day1-http-base/base3/gee/gee.go
```
package gee

import (
	"fmt"
	"net/http"
)

// HandlerFunc defines the request handler used by gee
type HandlerFunc func(http.ResponseWriter, *http.Request)

// Engine implement the interface of ServeHTTP
type Engine struct {
	router map[string]HandlerFunc
}

// New is the constructor of gee.Engine
func New() *Engine {
	return &Engine{router: make(map[string]HandlerFunc)}
}

func (engine *Engine) addRoute(method string, pattern string, handler HandlerFunc) {
	key := method + "-" + pattern
	engine.router[key] = handler
}

// GET defines the method to add GET request
func (engine *Engine) GET(pattern string, handler HandlerFunc) {
	engine.addRoute("GET", pattern, handler)
}

// POST defines the method to add POST request
func (engine *Engine) POST(pattern string, handler HandlerFunc) {
	engine.addRoute("POST", pattern, handler)
}

// Run defines the method to start a http server
func (engine *Engine) Run(addr string) (err error) {
	return http.ListenAndServe(addr, engine)
}

func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	key := req.Method + "-" + req.URL.Path
	if handler, ok := engine.router[key]; ok {
		handler(w, req)
	} else {
		fmt.Fprintf(w, "404 NOT FOUND: %s\n", req.URL)
	}
}
```

那么gee.go就是重头戏了。我们重点介绍一下这部分的实现。

- 首先定义了类型HandlerFunc，这是提供给框架用户的，用来定义路由映射的处理方法。我们在Engine中，添加了一张路由映射表router，key 由请求方法和静态路由地址构成，例如GET-/、GET-/hello、POST-/hello，这样针对相同的路由，如果请求方法不同,可以映射不同的处理方法(Handler)，value 是用户映射的处理方法。

- 当用户调用(*Engine).GET()方法时，会将路由和处理方法注册到映射表 router 中，(*Engine).Run()方法，是 ListenAndServe 的包装。

- Engine实现的 ServeHTTP 方法的作用就是，解析请求的路径，查找路由映射表，如果查到，就执行注册的处理方法。如果查不到，就返回 404 NOT FOUND 。

执行go run main.go，再用 curl 工具访问，结果与最开始的一致。

```
$ curl http://localhost:9999/
URL.Path = "/"
$ curl http://localhost:9999/hello
Header["Accept"] = ["*/*"]
Header["User-Agent"] = ["curl/7.54.0"]
curl http://localhost:9999/world
404 NOT FOUND: /world
```

至此，整个Gee框架的原型已经出来了。实现了路由映射表，提供了用户注册静态路由的方法，包装了启动服务的函数。当然，到目前为止，我们还没有实现比net/http标准库更强大的能力，不用担心，很快就可以将动态路由、中间件等功能添加上去了。

# Go语言动手写Web框架 - Gee第二天 上下文Context
本文是 7天用Go从零实现Web框架Gee教程系列的第二篇。

- 将路由(router)独立出来，方便之后增强。
- 设计上下文(Context)，封装 Request 和 Response ，提供对 JSON、HTML 等返回类型的支持。
- 动手写 Gee 框架的第二天，框架代码140行，新增代码约90行

## 使用效果

为了展示第二天的成果，我们看一看在使用时的效果。

main.go
```
func main() {
	r := gee.New()
	r.GET("/", func(c *gee.Context) {
		c.HTML(http.StatusOK, "<h1>Hello Gee</h1>")
	})
	r.GET("/hello", func(c *gee.Context) {
		// expect /hello?name=geektutu
		c.String(http.StatusOK, "hello %s, you're at %s\n", c.Query("name"), c.Path)
	})

	r.POST("/login", func(c *gee.Context) {
		c.JSON(http.StatusOK, gee.H{
			"username": c.PostForm("username"),
			"password": c.PostForm("password"),
		})
	})

	r.Run(":9999")
}
```

- Handler的参数变成成了gee.Context，提供了查询Query/PostForm参数的功能。
- gee.Context封装了HTML/String/JSON函数，能够快速构造HTTP响应。

## 设计Context
必要性
- 对Web服务来说，无非是根据请求*http.Request，构造响应http.ResponseWriter。但是这两个对象提供的接口粒度太细，比如我们要构造一个完整的响应，需要考虑消息头(Header)和消息体(Body)，而 Header 包含了状态码(StatusCode)，消息类型(ContentType)等几乎每次请求都需要设置的信息。因此，如果不进行有效的封装，那么框架的用户将需要写大量重复，繁杂的代码，而且容易出错。针对常用场景，能够高效地构造出 HTTP 响应是一个好的框架必须考虑的点。

用返回 JSON 数据作比较，感受下封装前后的差距。
封装前
```
obj = map[string]interface{}{
    "name": "geektutu",
    "password": "1234",
}
w.Header().Set("Content-Type", "application/json")
w.WriteHeader(http.StatusOK)
encoder := json.NewEncoder(w)
if err := encoder.Encode(obj); err != nil {
    http.Error(w, err.Error(), 500)
}
```

VS 封装后：
```
c.JSON(http.StatusOK, gee.H{
    "username": c.PostForm("username"),
    "password": c.PostForm("password"),
})
```

- 针对使用场景，封装*http.Request和http.ResponseWriter的方法，简化相关接口的调用，只是设计 Context 的原因之一。对于框架来说，还需要支撑额外的功能。例如，将来解析动态路由/hello/:name，参数:name的值放在哪呢？再比如，框架需要支持中间件，那中间件产生的信息放在哪呢？Context 随着每一个请求的出现而产生，请求的结束而销毁，和当前请求强相关的信息都应由 Context 承载。因此，设计 Context 结构，扩展性和复杂性留在了内部，而对外简化了接口。路由的处理函数，以及将要实现的中间件，参数都统一使用 Context 实例， Context 就像一次会话的百宝箱，可以找到任何东西。

具体实现

day2-context/gee/context.go
```
type H map[string]interface{}

type Context struct {
	// origin objects
	Writer http.ResponseWriter
	Req    *http.Request
	// request info
	Path   string
	Method string
	// response info
	StatusCode int
}

func newContext(w http.ResponseWriter, req *http.Request) *Context {
	return &Context{
		Writer: w,
		Req:    req,
		Path:   req.URL.Path,
		Method: req.Method,
	}
}

func (c *Context) PostForm(key string) string {
	return c.Req.FormValue(key)
}

func (c *Context) Query(key string) string {
	return c.Req.URL.Query().Get(key)
}

func (c *Context) Status(code int) {
	c.StatusCode = code
	c.Writer.WriteHeader(code)
}

func (c *Context) SetHeader(key string, value string) {
	c.Writer.Header().Set(key, value)
}

func (c *Context) String(code int, format string, values ...interface{}) {
	c.Status(code)
	c.SetHeader("Content-Type", "text/plain")
	c.Writer.Write([]byte(fmt.Sprintf(format, values...)))
}

func (c *Context) JSON(code int, obj interface{}) {
	c.Status(code)
	c.SetHeader("Content-Type", "application/json")
	encoder := json.NewEncoder(c.Writer)
	if err := encoder.Encode(obj); err != nil {
		http.Error(c.Writer, err.Error(), 500)
	}
}

func (c *Context) Data(code int, data []byte) {
	c.Status(code)
	c.Writer.Write(data)
}

func (c *Context) HTML(code int, html string) {
	c.Status(code)
	c.SetHeader("Content-Type", "text/html")
	c.Writer.Write([]byte(html))
}
```

- 代码最开头，给map[string]interface{}起了一个别名gee.H，构建JSON数据时，显得更简洁。
- Context目前只包含了http.ResponseWriter和*http.Request，另外提供了对 Method 和 Path 这两个常用属性的直接访问。
- 提供了访问Query和PostForm参数的方法。
- 提供了快速构造String/Data/JSON/HTML响应的方法。

## 路由(Router)
我们将和路由相关的方法和结构提取了出来，放到了一个新的文件中router.go，方便我们下一次对 router 的功能进行增强，例如提供动态路由的支持。 router 的 handle 方法作了一个细微的调整，即 handler 的参数，变成了 Context。

day2-context/gee/router.go
```
type router struct {
	handlers map[string]HandlerFunc
}

func newRouter() *router {
	return &router{handlers: make(map[string]HandlerFunc)}
}

func (r *router) addRoute(method string, pattern string, handler HandlerFunc) {
	log.Printf("Route %4s - %s", method, pattern)
	key := method + "-" + pattern
	r.handlers[key] = handler
}

func (r *router) handle(c *Context) {
	key := c.Method + "-" + c.Path
	if handler, ok := r.handlers[key]; ok {
		handler(c)
	} else {
		c.String(http.StatusNotFound, "404 NOT FOUND: %s\n", c.Path)
	}
}
```

## 框架入口
day2-context/gee/gee.go
```
// HandlerFunc defines the request handler used by gee
type HandlerFunc func(*Context)

// Engine implement the interface of ServeHTTP
type Engine struct {
	router *router
}

// New is the constructor of gee.Engine
func New() *Engine {
	return &Engine{router: newRouter()}
}

func (engine *Engine) addRoute(method string, pattern string, handler HandlerFunc) {
	engine.router.addRoute(method, pattern, handler)
}

// GET defines the method to add GET request
func (engine *Engine) GET(pattern string, handler HandlerFunc) {
	engine.addRoute("GET", pattern, handler)
}

// POST defines the method to add POST request
func (engine *Engine) POST(pattern string, handler HandlerFunc) {
	engine.addRoute("POST", pattern, handler)
}

// Run defines the method to start a http server
func (engine *Engine) Run(addr string) (err error) {
	return http.ListenAndServe(addr, engine)
}

func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	c := newContext(w, req)
	engine.router.handle(c)
}
```

将router相关的代码独立后，gee.go简单了不少。最重要的还是通过实现了 ServeHTTP 接口，接管了所有的 HTTP 请求。相比第一天的代码，这个方法也有细微的调整，在调用 router.handle 之前，构造了一个 Context 对象。这个对象目前还非常简单，仅仅是包装了原来的两个参数，之后我们会慢慢地给Context插上翅膀。

如何使用，main.go一开始就已经亮相了。运行go run main.go，借助 curl ，一起看一看今天的成果吧。

```
$ curl -i http://localhost:9999/
HTTP/1.1 200 OK
Date: Mon, 12 Aug 2019 16:52:52 GMT
Content-Length: 18
Content-Type: text/html; charset=utf-8
<h1>Hello Gee</h1>

$ curl "http://localhost:9999/hello?name=geektutu"
hello geektutu, you're at /hello

$ curl "http://localhost:9999/login" -X POST -d 'username=geektutu&password=1234'
{"password":"1234","username":"geektutu"}

$ curl "http://localhost:9999/xxx"
404 NOT FOUND: /xxx
```

# Go语言动手写Web框架 - Gee第三天 前缀树路由Router
本文是 7天用Go从零实现Web框架Gee教程系列的第三篇。

- 使用 Tire 树实现动态路由(dynamic route)解析。
- 支持两种模式:name和*filepath，代码约150行。

## Trie 树简介
之前，我们用了一个非常简单的map结构存储了路由表，使用map存储键值对，索引非常高效，但是有一个弊端，键值对的存储的方式，只能用来索引静态路由。那如果我们想支持类似于/hello/:name这样的动态路由怎么办呢？所谓动态路由，即一条路由规则可以匹配某一类型而非某一条固定的路由。例如/hello/:name，可以匹配/hello/geektutu、hello/jack等。

动态路由有很多种实现方式，支持的规则、性能等有很大的差异。例如开源的路由实现gorouter支持在路由规则中嵌入正则表达式，例如/p/[0-9A-Za-z]+，即路径中的参数仅匹配数字和字母；另一个开源实现httprouter就不支持正则表达式。著名的Web开源框架gin 在早期的版本，并没有实现自己的路由，而是直接使用了httprouter，后来不知道什么原因，放弃了httprouter，自己实现了一个版本。

实现动态路由最常用的数据结构，被称为前缀树(Trie树)。看到名字你大概也能知道前缀树长啥样了：每一个节点的所有的子节点都拥有相同的前缀。这种结构非常适用于路由匹配，比如我们定义了如下路由规则：
- /:lang/doc
- /:lang/tutorial
- /:lang/intro
- /about
- /p/blog
- /p/related

我们用前缀树来表示，是这样的。

HTTP请求的路径恰好是由/分隔的多段构成的，因此，每一段可以作为前缀树的一个节点。我们通过树结构查询，如果中间某一层的节点都不满足条件，那么就说明没有匹配到的路由，查询结束。

接下来我们实现的动态路由具备以下两个功能。

- 参数匹配:。例如 /p/:lang/doc，可以匹配 /p/c/doc 和 /p/go/doc。
- 通配*。例如 /static/*filepath，可以匹配/static/fav.ico，也可以匹配/static/js/jQuery.js，这种模式常用于静态服务器，能够递归地匹配子路径。

## Trie 树实现
首先我们需要设计树节点上应该存储那些信息。

day3-router/gee/trie.go
```
type node struct {
	pattern  string // 待匹配路由，例如 /p/:lang
	part     string // 路由中的一部分，例如 :lang
	children []*node // 子节点，例如 [doc, tutorial, intro]
	isWild   bool // 是否精确匹配，part 含有 : 或 * 时为true
}
```

与普通的树不同，为了实现动态路由匹配，加上了isWild这个参数。即当我们匹配 /p/go/doc/这个路由时，第一层节点，p精准匹配到了p，第二层节点，go模糊匹配到:lang，那么将会把lang这个参数赋值为go，继续下一层匹配。我们将匹配的逻辑，包装为一个辅助函数。

```
// 第一个匹配成功的节点，用于插入
func (n *node) matchChild(part string) *node {
	for _, child := range n.children {
		if child.part == part || child.isWild {
			return child
		}
	}
	return nil
}
// 所有匹配成功的节点，用于查找
func (n *node) matchChildren(part string) []*node {
	nodes := make([]*node, 0)
	for _, child := range n.children {
		if child.part == part || child.isWild {
			nodes = append(nodes, child)
		}
	}
	return nodes
}
```

对于路由来说，最重要的当然是注册与匹配了。开发服务时，注册路由规则，映射handler；访问时，匹配路由规则，查找到对应的handler。因此，Trie 树需要支持节点的插入与查询。插入功能很简单，递归查找每一层的节点，如果没有匹配到当前part的节点，则新建一个，有一点需要注意，/p/:lang/doc只有在第三层节点，即doc节点，pattern才会设置为/p/:lang/doc。p和:lang节点的pattern属性皆为空。因此，当匹配结束时，我们可以使用n.pattern == ""来判断路由规则是否匹配成功。例如，/p/python虽能成功匹配到:lang，但:lang的pattern值为空，因此匹配失败。查询功能，同样也是递归查询每一层的节点，退出规则是，匹配到了*，匹配失败，或者匹配到了第len(parts)层节点。

```
func (n *node) insert(pattern string, parts []string, height int) {
	if len(parts) == height {
		n.pattern = pattern
		return
	}

	part := parts[height]
	child := n.matchChild(part)
	if child == nil {
		child = &node{part: part, isWild: part[0] == ':' || part[0] == '*'}
		n.children = append(n.children, child)
	}
	child.insert(pattern, parts, height+1)
}

func (n *node) search(parts []string, height int) *node {
	if len(parts) == height || strings.HasPrefix(n.part, "*") {
		if n.pattern == "" {
			return nil
		}
		return n
	}

	part := parts[height]
	children := n.matchChildren(part)

	for _, child := range children {
		result := child.search(parts, height+1)
		if result != nil {
			return result
		}
	}

	return nil
}
```

## Router
Trie 树的插入与查找都成功实现了，接下来我们将 Trie 树应用到路由中去吧。我们使用 roots 来存储每种请求方式的Trie 树根节点。使用 handlers 存储每种请求方式的 HandlerFunc 。getRoute 函数中，还解析了:和*两种匹配符的参数，返回一个 map 。例如/p/go/doc匹配到/p/:lang/doc，解析结果为：{lang: "go"}，/static/css/geektutu.css匹配到/static/*filepath，解析结果为{filepath: "css/geektutu.css"}。

day3-router/gee/router.go
```
type router struct {
	roots    map[string]*node
	handlers map[string]HandlerFunc
}

// roots key eg, roots['GET'] roots['POST']
// handlers key eg, handlers['GET-/p/:lang/doc'], handlers['POST-/p/book']
    
func newRouter() *router {
	return &router{
		roots:    make(map[string]*node),
		handlers: make(map[string]HandlerFunc),
	}
}

// Only one * is allowed
func parsePattern(pattern string) []string {
	vs := strings.Split(pattern, "/")

	parts := make([]string, 0)
	for _, item := range vs {
		if item != "" {
			parts = append(parts, item)
			if item[0] == '*' {
				break
			}
		}
	}
	return parts
}

func (r *router) addRoute(method string, pattern string, handler HandlerFunc) {
	parts := parsePattern(pattern)

	key := method + "-" + pattern
	_, ok := r.roots[method]
	if !ok {
		r.roots[method] = &node{}
	}
	r.roots[method].insert(pattern, parts, 0)
	r.handlers[key] = handler
}

func (r *router) getRoute(method string, path string) (*node, map[string]string) {
	searchParts := parsePattern(path)
	params := make(map[string]string)
	root, ok := r.roots[method]

	if !ok {
		return nil, nil
	}

	n := root.search(searchParts, 0)

	if n != nil {
		parts := parsePattern(n.pattern)
		for index, part := range parts {
			if part[0] == ':' {
				params[part[1:]] = searchParts[index]
			}
			if part[0] == '*' && len(part) > 1 {
				params[part[1:]] = strings.Join(searchParts[index:], "/")
				break
			}
		}
		return n, params
	}

	return nil, nil
}
```

## Context与handle的变化
在 HandlerFunc 中，希望能够访问到解析的参数，因此，需要对 Context 对象增加一个属性和方法，来提供对路由参数的访问。我们将解析后的参数存储到Params中，通过c.Param("lang")的方式获取到对应的值。

day3-router/gee/context.go
```
type Context struct {
	// origin objects
	Writer http.ResponseWriter
	Req    *http.Request
	// request info
	Path   string
	Method string
	Params map[string]string
	// response info
	StatusCode int
}

func (c *Context) Param(key string) string {
	value, _ := c.Params[key]
	return value
}
```

day3-router/gee/router.go
```
func (r *router) handle(c *Context) {
	n, params := r.getRoute(c.Method, c.Path)
	if n != nil {
		c.Params = params
		key := c.Method + "-" + n.pattern
		r.handlers[key](c)
	} else {
		c.String(http.StatusNotFound, "404 NOT FOUND: %s\n", c.Path)
	}
}
```

router.go的变化比较小，比较重要的一点是，在调用匹配到的handler前，将解析出来的路由参数赋值给了c.Params。这样就能够在handler中，通过Context对象访问到具体的值了。

## 单元测试
```
func newTestRouter() *router {
	r := newRouter()
	r.addRoute("GET", "/", nil)
	r.addRoute("GET", "/hello/:name", nil)
	r.addRoute("GET", "/hello/b/c", nil)
	r.addRoute("GET", "/hi/:name", nil)
	r.addRoute("GET", "/assets/*filepath", nil)
	return r
}

func TestParsePattern(t *testing.T) {
	ok := reflect.DeepEqual(parsePattern("/p/:name"), []string{"p", ":name"})
	ok = ok && reflect.DeepEqual(parsePattern("/p/*"), []string{"p", "*"})
	ok = ok && reflect.DeepEqual(parsePattern("/p/*name/*"), []string{"p", "*name"})
	if !ok {
		t.Fatal("test parsePattern failed")
	}
}

func TestGetRoute(t *testing.T) {
	r := newTestRouter()
	n, ps := r.getRoute("GET", "/hello/geektutu")

	if n == nil {
		t.Fatal("nil shouldn't be returned")
	}

	if n.pattern != "/hello/:name" {
		t.Fatal("should match /hello/:name")
	}

	if ps["name"] != "geektutu" {
		t.Fatal("name should be equal to 'geektutu'")
	}

	fmt.Printf("matched path: %s, params['name']: %s\n", n.pattern, ps["name"])

}
```

## 使用Demo
看看框架使用的样例吧。

day3-router/main.go
```
func main() {
	r := gee.New()
	r.GET("/", func(c *gee.Context) {
		c.HTML(http.StatusOK, "<h1>Hello Gee</h1>")
	})

	r.GET("/hello", func(c *gee.Context) {
		// expect /hello?name=geektutu
		c.String(http.StatusOK, "hello %s, you're at %s\n", c.Query("name"), c.Path)
	})

	r.GET("/hello/:name", func(c *gee.Context) {
		// expect /hello/geektutu
		c.String(http.StatusOK, "hello %s, you're at %s\n", c.Param("name"), c.Path)
	})

	r.GET("/assets/*filepath", func(c *gee.Context) {
		c.JSON(http.StatusOK, gee.H{"filepath": c.Param("filepath")})
	})

	r.Run(":9999")
}
```

使用curl工具，测试结果。
```
$ curl "http://localhost:9999/hello/geektutu"
hello geektutu, you're at /hello/geektutu

$ curl "http://localhost:9999/assets/css/geektutu.css"
{"filepath":"css/geektutu.css"}
```

# Go语言动手写Web框架 - Gee第四天 分组控制Group

本文是 7天用Go从零实现Web框架Gee教程系列的第四篇。

- 实现路由分组控制(Route Group Control)，代码约50行

## 分组的意义
分组控制(Group Control)是 Web 框架应提供的基础功能之一。所谓分组，是指路由的分组。如果没有路由分组，我们需要针对每一个路由进行控制。但是真实的业务场景中，往往某一组路由需要相似的处理。例如：

- 以/post开头的路由匿名可访问。
- 以/admin开头的路由需要鉴权。
- 以/api开头的路由是 RESTful 接口，可以对接第三方平台，需要三方平台鉴权。

大部分情况下的路由分组，是以相同的前缀来区分的。因此，我们今天实现的分组控制也是以前缀来区分，并且支持分组的嵌套。例如/post是一个分组，/post/a和/post/b可以是该分组下的子分组。作用在/post分组上的中间件(middleware)，也都会作用在子分组，子分组还可以应用自己特有的中间件。

中间件可以给框架提供无限的扩展能力，应用在分组上，可以使得分组控制的收益更为明显，而不是共享相同的路由前缀这么简单。例如/admin的分组，可以应用鉴权中间件；/分组应用日志中间件，/是默认的最顶层的分组，也就意味着给所有的路由，即整个框架增加了记录日志的能力。

提供扩展能力支持中间件的内容，我们将在下一节当中介绍。

## 分组嵌套
一个 Group 对象需要具备哪些属性呢？首先是前缀(prefix)，比如/，或者/api；要支持分组嵌套，那么需要知道当前分组的父亲(parent)是谁；当然了，按照我们一开始的分析，中间件是应用在分组上的，那还需要存储应用在该分组上的中间件(middlewares)。还记得，我们之前调用函数(*Engine).addRoute()来映射所有的路由规则和 Handler 。如果Group对象需要直接映射路由规则的话，比如我们想在使用框架时，这么调用：
```
r := gee.New()
v1 := r.Group("/v1")
v1.GET("/", func(c *gee.Context) {
	c.HTML(http.StatusOK, "<h1>Hello Gee</h1>")
})
```

那么Group对象，还需要有访问Router的能力，为了方便，我们可以在Group中，保存一个指针，指向Engine，整个框架的所有资源都是由Engine统一协调的，那么就可以通过Engine间接地访问各种接口了。

所以，最后的 Group 的定义是这样的：

day4-group/gee/gee.go
```
RouterGroup struct {
	prefix      string
	middlewares []HandlerFunc // support middleware
	parent      *RouterGroup  // support nesting
	engine      *Engine       // all groups share a Engine instance
}
```

我们还可以进一步地抽象，将Engine作为最顶层的分组，也就是说Engine拥有RouterGroup所有的能力。
```
Engine struct {
	*RouterGroup
	router *router
	groups []*RouterGroup // store all groups
}
```

那我们就可以将和路由有关的函数，都交给RouterGroup实现了。
```
// New is the constructor of gee.Engine
func New() *Engine {
	engine := &Engine{router: newRouter()}
	engine.RouterGroup = &RouterGroup{engine: engine}
	engine.groups = []*RouterGroup{engine.RouterGroup}
	return engine
}

// Group is defined to create a new RouterGroup
// remember all groups share the same Engine instance
func (group *RouterGroup) Group(prefix string) *RouterGroup {
	engine := group.engine
	newGroup := &RouterGroup{
		prefix: group.prefix + prefix,
		parent: group,
		engine: engine,
	}
	engine.groups = append(engine.groups, newGroup)
	return newGroup
}

func (group *RouterGroup) addRoute(method string, comp string, handler HandlerFunc) {
	pattern := group.prefix + comp
	log.Printf("Route %4s - %s", method, pattern)
	group.engine.router.addRoute(method, pattern, handler)
}

// GET defines the method to add GET request
func (group *RouterGroup) GET(pattern string, handler HandlerFunc) {
	group.addRoute("GET", pattern, handler)
}

// POST defines the method to add POST request
func (group *RouterGroup) POST(pattern string, handler HandlerFunc) {
	group.addRoute("POST", pattern, handler)
}
```

可以仔细观察下addRoute函数，调用了group.engine.router.addRoute来实现了路由的映射。由于Engine从某种意义上继承了RouterGroup的所有属性和方法，因为 (*Engine).engine 是指向自己的。这样实现，我们既可以像原来一样添加路由，也可以通过分组添加路由。

## 使用 Demo
测试框架的Demo就可以这样写了：

```
func main() {
	r := gee.New()
	r.GET("/index", func(c *gee.Context) {
		c.HTML(http.StatusOK, "<h1>Index Page</h1>")
	})
	v1 := r.Group("/v1")
	{
		v1.GET("/", func(c *gee.Context) {
			c.HTML(http.StatusOK, "<h1>Hello Gee</h1>")
		})

		v1.GET("/hello", func(c *gee.Context) {
			// expect /hello?name=geektutu
			c.String(http.StatusOK, "hello %s, you're at %s\n", c.Query("name"), c.Path)
		})
	}
	v2 := r.Group("/v2")
	{
		v2.GET("/hello/:name", func(c *gee.Context) {
			// expect /hello/geektutu
			c.String(http.StatusOK, "hello %s, you're at %s\n", c.Param("name"), c.Path)
		})
		v2.POST("/login", func(c *gee.Context) {
			c.JSON(http.StatusOK, gee.H{
				"username": c.PostForm("username"),
				"password": c.PostForm("password"),
			})
		})

	}

	r.Run(":9999")
}
```

通过 curl 简单测试：
```
$ curl "http://localhost:9999/v1/hello?name=geektutu"
hello geektutu, you're at /v1/hello

$ curl "http://localhost:9999/v2/hello/geektutu"
hello geektutu, you're at /hello/geektutu
```

# Go语言动手写Web框架 - Gee第五天 中间件Middleware
本文是 7天用Go从零实现Web框架Gee教程系列的第五篇。
- 设计并实现 Web 框架的中间件(Middlewares)机制。
- 实现通用的Logger中间件，能够记录请求到响应所花费的时间，代码约50行

## 中间件是什么
中间件(middlewares)，简单说，就是非业务的技术类组件。Web 框架本身不可能去理解所有的业务，因而不可能实现所有的功能。因此，框架需要有一个插口，允许用户自己定义功能，嵌入到框架中，仿佛这个功能是框架原生支持的一样。因此，对中间件而言，需要考虑2个比较关键的点：

- 插入点在哪？使用框架的人并不关心底层逻辑的具体实现，如果插入点太底层，中间件逻辑就会非常复杂。如果插入点离用户太近，那和用户直接定义一组函数，每次在 Handler 中手工调用没有多大的优势了。
- 中间件的输入是什么？中间件的输入，决定了扩展能力。暴露的参数太少，用户发挥空间有限。那对于一个 Web 框架而言，中间件应该设计成什么样呢？接下来的实现，基本参考了 Gin 框架。

## 中间件设计
Gee 的中间件的定义与路由映射的 Handler 一致，处理的输入是Context对象。插入点是框架接收到请求初始化Context对象后，允许用户使用自己定义的中间件做一些额外的处理，例如记录日志等，以及对Context进行二次加工。另外通过调用(*Context).Next()函数，中间件可等待用户自己定义的 Handler处理结束后，再做一些额外的操作，例如计算本次处理所用时间等。即 Gee 的中间件支持用户在请求被处理的前后，做一些额外的操作。举个例子，我们希望最终能够支持如下定义的中间件，c.Next()表示等待执行其他的中间件或用户的Handler：

day4-group/gee/logger.go
```
func Logger() HandlerFunc {
	return func(c *Context) {
		// Start timer
		t := time.Now()
		// Process request
		c.Next()
		// Calculate resolution time
		log.Printf("[%d] %s in %v", c.StatusCode, c.Req.RequestURI, time.Since(t))
	}
}
```

另外，支持设置多个中间件，依次进行调用。

我们上一篇文章分组控制 Group Control中讲到，中间件是应用在RouterGroup上的，应用在最顶层的 Group，相当于作用于全局，所有的请求都会被中间件处理。那为什么不作用在每一条路由规则上呢？作用在某条路由规则，那还不如用户直接在 Handler 中调用直观。只作用在某条路由规则的功能通用性太差，不适合定义为中间件。

我们之前的框架设计是这样的，当接收到请求后，匹配路由，该请求的所有信息都保存在Context中。中间件也不例外，接收到请求后，应查找所有应作用于该路由的中间件，保存在Context中，依次进行调用。为什么依次调用后，还需要在Context中保存呢？因为在设计中，中间件不仅作用在处理流程前，也可以作用在处理流程后，即在用户定义的 Handler 处理完毕后，还可以执行剩下的操作。

为此，我们给Context添加了2个参数，定义了Next方法：

day4-group/gee/context.go
```
type Context struct {
	// origin objects
	Writer http.ResponseWriter
	Req    *http.Request
	// request info
	Path   string
	Method string
	Params map[string]string
	// response info
	StatusCode int
	// middleware
	handlers []HandlerFunc
	index    int
}

func newContext(w http.ResponseWriter, req *http.Request) *Context {
	return &Context{
		Path:   req.URL.Path,
		Method: req.Method,
		Req:    req,
		Writer: w,
		index:  -1,
	}
}

func (c *Context) Next() {
	c.index++
	s := len(c.handlers)
	for ; c.index < s; c.index++ {
		c.handlers[c.index](c)
	}
}
```

index是记录当前执行到第几个中间件，当在中间件中调用Next方法时，控制权交给了下一个中间件，直到调用到最后一个中间件，然后再从后往前，调用每个中间件在Next方法之后定义的部分。如果我们将用户在映射路由时定义的Handler添加到c.handlers列表中，结果会怎么样呢？想必你已经猜到了。

```
func A(c *Context) {
    part1
    c.Next()
    part2
}
func B(c *Context) {
    part3
    c.Next()
    part4
}
```

假设我们应用了中间件 A 和 B，和路由映射的 Handler。c.handlers是这样的[A, B, Handler]，c.index初始化为-1。调用c.Next()，接下来的流程是这样的：

- c.index++，c.index 变为 0
- 0 < 3，调用 c.handlers[0]，即 A
- 执行 part1，调用 c.Next()
- c.index++，c.index 变为 1
- 1 < 3，调用 c.handlers[1]，即 B
- 执行 part3，调用 c.Next()
- c.index++，c.index 变为 2
- 2 < 3，调用 c.handlers[2]，即Handler
- Handler 调用完毕，返回到 B 中的 part4，执行 part4
- part4 执行完毕，返回到 A 中的 part2，执行 part2
- part2 执行完毕，结束。

一句话说清楚重点，最终的顺序是part1 -> part3 -> Handler -> part 4 -> part2。恰恰满足了我们对中间件的要求，接下来看调用部分的代码，就能全部串起来了。

## 代码实现
定义Use函数，将中间件应用到某个 Group 。

day4-group/gee/gee.go
```
// Use is defined to add middleware to the group
func (group *RouterGroup) Use(middlewares ...HandlerFunc) {
	group.middlewares = append(group.middlewares, middlewares...)
}

func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	var middlewares []HandlerFunc
	for _, group := range engine.groups {
		if strings.HasPrefix(req.URL.Path, group.prefix) {
			middlewares = append(middlewares, group.middlewares...)
		}
	}
	c := newContext(w, req)
	c.handlers = middlewares
	engine.router.handle(c)
}
```

ServeHTTP 函数也有变化，当我们接收到一个具体请求时，要判断该请求适用于哪些中间件，在这里我们简单通过 URL 的前缀来判断。得到中间件列表后，赋值给 c.handlers。

- handle 函数中，将从路由匹配得到的 Handler 添加到 c.handlers列表中，执行c.Next()。

day4-group/gee/router.go
```
func (r *router) handle(c *Context) {
	n, params := r.getRoute(c.Method, c.Path)

	if n != nil {
		key := c.Method + "-" + n.pattern
		c.Params = params
		c.handlers = append(c.handlers, r.handlers[key])
	} else {
		c.handlers = append(c.handlers, func(c *Context) {
			c.String(http.StatusNotFound, "404 NOT FOUND: %s\n", c.Path)
		})
	}
	c.Next()
}
```

## 使用 Demo
```
func onlyForV2() gee.HandlerFunc {
	return func(c *gee.Context) {
		// Start timer
		t := time.Now()
		// if a server error occurred
		c.Fail(500, "Internal Server Error")
		// Calculate resolution time
		log.Printf("[%d] %s in %v for group v2", c.StatusCode, c.Req.RequestURI, time.Since(t))
	}
}

func main() {
	r := gee.New()
	r.Use(gee.Logger()) // global midlleware
	r.GET("/", func(c *gee.Context) {
		c.HTML(http.StatusOK, "<h1>Hello Gee</h1>")
	})

	v2 := r.Group("/v2")
	v2.Use(onlyForV2()) // v2 group middleware
	{
		v2.GET("/hello/:name", func(c *gee.Context) {
			// expect /hello/geektutu
			c.String(http.StatusOK, "hello %s, you're at %s\n", c.Param("name"), c.Path)
		})
	}

	r.Run(":9999")
}
```

gee.Logger()即我们一开始就介绍的中间件，我们将这个中间件和框架代码放在了一起，作为框架默认提供的中间件。在这个例子中，我们将gee.Logger()应用在了全局，所有的路由都会应用该中间件。onlyForV2()是用来测试功能的，仅在v2对应的 Group 中应用了。

接下来使用 curl 测试，可以看到，v2 Group 2个中间件都生效了。
```
$ curl http://localhost:9999/
>>> log
2019/08/17 01:37:38 [200] / in 3.14µs

(2) global + group middleware
$ curl http://localhost:9999/v2/hello/geektutu
>>> log
2019/08/17 01:38:48 [200] /v2/hello/geektutu in 61.467µs for group v2
2019/08/17 01:38:48 [200] /v2/hello/geektutu in 281µs
```

# Go语言动手写Web框架 - Gee第六天 模板(HTML Template)
本文是 7天用Go从零实现Web框架Gee教程系列的第六篇。

- 实现静态资源服务(Static Resource)。
- 支持HTML模板渲染。

## 服务端渲染
现在越来越流行前后端分离的开发模式，即 Web 后端提供 RESTful 接口，返回结构化的数据(通常为 JSON 或者 XML)。前端使用 AJAX 技术请求到所需的数据，利用 JavaScript 进行渲染。Vue/React 等前端框架持续火热，这种开发模式前后端解耦，优势非常突出。后端童鞋专心解决资源利用，并发，数据库等问题，只需要考虑数据如何生成；前端童鞋专注于界面设计实现，只需要考虑拿到数据后如何渲染即可。使用 JSP 写过网站的童鞋，应该能感受到前后端耦合的痛苦。JSP 的表现力肯定是远不如 Vue/React 等专业做前端渲染的框架的。而且前后端分离在当前还有另外一个不可忽视的优势。因为后端只关注于数据，接口返回值是结构化的，与前端解耦。同一套后端服务能够同时支撑小程序、移动APP、PC端 Web 页面，以及对外提供的接口。随着前端工程化的不断地发展，Webpack，gulp 等工具层出不穷，前端技术越来越自成体系了。

但前后分离的一大问题在于，页面是在客户端渲染的，比如浏览器，这对于爬虫并不友好。Google 爬虫已经能够爬取渲染后的网页，但是短期内爬取服务端直接渲染的 HTML 页面仍是主流。

今天的内容便是介绍 Web 框架如何支持服务端渲染的场景。

## 静态文件(Serve Static Files)
网页的三剑客，JavaScript、CSS 和 HTML。要做到服务端渲染，第一步便是要支持 JS、CSS 等静态文件。还记得我们之前设计动态路由的时候，支持通配符*匹配多级子路径。比如路由规则/assets/*filepath，可以匹配/assets/开头的所有的地址。例如/assets/js/geektutu.js，匹配后，参数filepath就赋值为js/geektutu.js。

那如果我么将所有的静态文件放在/usr/web目录下，那么filepath的值即是该目录下文件的相对地址。映射到真实的文件后，将文件返回，静态服务器就实现了。

找到文件后，如何返回这一步，net/http库已经实现了。因此，gee 框架要做的，仅仅是解析请求的地址，映射到服务器上文件的真实地址，交给http.FileServer处理就好了。

day6-template/gee/gee.go
```
// create static handler
func (group *RouterGroup) createStaticHandler(relativePath string, fs http.FileSystem) HandlerFunc {
	absolutePath := path.Join(group.prefix, relativePath)
	fileServer := http.StripPrefix(absolutePath, http.FileServer(fs))
	return func(c *Context) {
		file := c.Param("filepath")
		// Check if file exists and/or if we have permission to access it
		if _, err := fs.Open(file); err != nil {
			c.Status(http.StatusNotFound)
			return
		}

		fileServer.ServeHTTP(c.Writer, c.Req)
	}
}

// serve static files
func (group *RouterGroup) Static(relativePath string, root string) {
	handler := group.createStaticHandler(relativePath, http.Dir(root))
	urlPattern := path.Join(relativePath, "/*filepath")
	// Register GET handlers
	group.GET(urlPattern, handler)
}
```

我们给RouterGroup添加了2个方法，Static这个方法是暴露给用户的。用户可以将磁盘上的某个文件夹root映射到路由relativePath。例如：
```
r := gee.New()
r.Static("/assets", "/usr/geektutu/blog/static")
// 或相对路径 r.Static("/assets", "./static")
r.Run(":9999")
```

用户访问localhost:9999/assets/js/geektutu.js，最终返回/usr/geektutu/blog/static/js/geektutu.js。

## HTML 模板渲染
Go语言内置了text/template和html/template2个模板标准库，其中html/template为 HTML 提供了较为完整的支持。包括普通变量渲染、列表渲染、对象渲染等。gee 框架的模板渲染直接使用了html/template提供的能力。
```
Engine struct {
	*RouterGroup
	router        *router
	groups        []*RouterGroup     // store all groups
	htmlTemplates *template.Template // for html render
	funcMap       template.FuncMap   // for html render
}

func (engine *Engine) SetFuncMap(funcMap template.FuncMap) {
	engine.funcMap = funcMap
}

func (engine *Engine) LoadHTMLGlob(pattern string) {
	engine.htmlTemplates = template.Must(template.New("").Funcs(engine.funcMap).ParseGlob(pattern))
}
```

首先为 Engine 示例添加了 *template.Template 和 template.FuncMap对象，前者将所有的模板加载进内存，后者是所有的自定义模板渲染函数。

另外，给用户分别提供了设置自定义渲染函数funcMap和加载模板的方法。

接下来，对原来的 (*Context).HTML()方法做了些小修改，使之支持根据模板文件名选择模板进行渲染。

day6-template/gee/context.go
```
func (c *Context) HTML(code int, name string, data interface{}) {
	c.Writer.WriteHeader(code)
	c.Writer.Header().Set("Content-Type", "text/html")
	if err := c.engine.htmlTemplates.ExecuteTemplate(c.Writer, name, data); err != nil {
		c.Fail(500, err.Error())
	}
}
```

## 使用Demo
最终的目录结构
```
---gee/
---static/
   |---css/
        |---geektutu.css
   |---file1.txt
---templates/
   |---arr.tmpl
   |---css.tmpl
   |---custom_func.tmpl
---main.go
```

```
<!-- day6-template/templates/css.tmpl -->
<html>
    <link rel="stylesheet" href="/assets/css/geektutu.css">
    <p>geektutu.css is loaded</p>
</html>
```

day6-template/main.go
```
type student struct {
	Name string
	Age  int8
}

func formatAsDate(t time.Time) string {
	year, month, day := t.Date()
	return fmt.Sprintf("%d-%02d-%02d", year, month, day)
}

func main() {
	r := gee.New()
	r.Use(gee.Logger())
	r.SetFuncMap(template.FuncMap{
		"formatAsDate": formatAsDate,
	})
	r.LoadHTMLGlob("templates/*")
	r.Static("/assets", "./static")

	stu1 := &student{Name: "Geektutu", Age: 20}
	stu2 := &student{Name: "Jack", Age: 22}
	r.GET("/", func(c *gee.Context) {
		c.HTML(http.StatusOK, "css.tmpl", nil)
	})
	r.GET("/students", func(c *gee.Context) {
		c.HTML(http.StatusOK, "arr.tmpl", gee.H{
			"title":  "gee",
			"stuArr": [2]*student{stu1, stu2},
		})
	})

	r.GET("/date", func(c *gee.Context) {
		c.HTML(http.StatusOK, "custom_func.tmpl", gee.H{
			"title": "gee",
			"now":   time.Date(2019, 8, 17, 0, 0, 0, 0, time.UTC),
		})
	})

	r.Run(":9999")
}
```

访问下主页，模板正常渲染，CSS 静态文件加载成功。


# Go语言动手写Web框架 - Gee第七天 错误恢复(Panic Recover)
本文是7天用Go从零实现Web框架Gee教程系列的第七篇。

- 实现错误处理机制。

## panic
Go 语言中，比较常见的错误处理方法是返回 error，由调用者决定后续如何处理。但是如果是无法恢复的错误，可以手动触发 panic，当然如果在程序运行过程中出现了类似于数组越界的错误，panic 也会被触发。panic 会中止当前执行的程序，退出。

下面是主动触发的例子：
```
// hello.go
func main() {
	fmt.Println("before panic")
	panic("crash")
	fmt.Println("after panic")
}
```

```
$ go run hello.go

before panic
panic: crash

goroutine 1 [running]:
main.main()
        ~/go_demo/hello/hello.go:7 +0x95
exit status 2
```

下面是数组越界触发的 panic
```
// hello.go
func main() {
	arr := []int{1, 2, 3}
	fmt.Println(arr[4])
}
```

```
$ go run hello.go
panic: runtime error: index out of range [4] with length 3
```

## defer
panic 会导致程序被中止，但是在退出前，会先处理完当前协程上已经defer 的任务，执行完成后再退出。效果类似于 java 语言的 try...catch。
```
// hello.go
func main() {
	defer func() {
		fmt.Println("defer func")
	}()

	arr := []int{1, 2, 3}
	fmt.Println(arr[4])
}
```

```
$ go run hello.go 
defer func
panic: runtime error: index out of range [4] with length 3
```

可以 defer 多个任务，在同一个函数中 defer 多个任务，会逆序执行。即先执行最后 defer 的任务。

在这里，defer 的任务执行完成之后，panic 还会继续被抛出，导致程序非正常结束。

## recover
Go 语言还提供了 recover 函数，可以避免因为 panic 发生而导致整个程序终止，recover 函数只在 defer 中生效。
```
// hello.go
func test_recover() {
	defer func() {
		fmt.Println("defer func")
		if err := recover(); err != nil {
			fmt.Println("recover success")
		}
	}()

	arr := []int{1, 2, 3}
	fmt.Println(arr[4])
	fmt.Println("after panic")
}

func main() {
	test_recover()
	fmt.Println("after recover")
}
```

```
$ go run hello.go 
defer func
recover success
after recover
```

我们可以看到，recover 捕获了 panic，程序正常结束。test_recover() 中的 after panic 没有打印，这是正确的，当 panic 被触发时，控制权就被交给了 defer 。就像在 java 中，try代码块中发生了异常，控制权交给了 catch，接下来执行 catch 代码块中的代码。而在 main() 中打印了 after recover，说明程序已经恢复正常，继续往下执行直到结束。

## Gee 的错误处理机制
对一个 Web 框架而言，错误处理机制是非常必要的。可能是框架本身没有完备的测试，导致在某些情况下出现空指针异常等情况。也有可能用户不正确的参数，触发了某些异常，例如数组越界，空指针等。如果因为这些原因导致系统宕机，必然是不可接受的。

我们在第六天实现的框架并没有加入异常处理机制，如果代码中存在会触发 panic 的 BUG，很容易宕掉。

例如下面的代码：
```
func main() {
	r := gee.New()
	r.GET("/panic", func(c *gee.Context) {
		names := []string{"geektutu"}
		c.String(http.StatusOK, names[100])
	})
	r.Run(":9999")
}
```

在上面的代码中，我们为 gee 注册了路由 /panic，而这个路由的处理函数内部存在数组越界 names[100]，如果访问 localhost:9999/panic，Web 服务就会宕掉。

今天，我们将在 gee 中添加一个非常简单的错误处理机制，即在此类错误发生时，向用户返回 Internal Server Error，并且在日志中打印必要的错误信息，方便进行错误定位。

我们之前实现了中间件机制，错误处理也可以作为一个中间件，增强 gee 框架的能力。

新增文件 gee/recovery.go，在这个文件中实现中间件 Recovery。
```
func Recovery() HandlerFunc {
	return func(c *Context) {
		defer func() {
			if err := recover(); err != nil {
				message := fmt.Sprintf("%s", err)
				log.Printf("%s\n\n", trace(message))
				c.Fail(http.StatusInternalServerError, "Internal Server Error")
			}
		}()

		c.Next()
	}
}
```

Recovery 的实现非常简单，使用 defer 挂载上错误恢复的函数，在这个函数中调用 recover()，捕获 panic，并且将堆栈信息打印在日志中，向用户返回 Internal Server Error。

你可能注意到，这里有一个 trace() 函数，这个函数是用来获取触发 panic 的堆栈信息，完整代码如下：

day7-panic-recover/gee/recovery.go
```
package gee

import (
	"fmt"
	"log"
	"net/http"
	"runtime"
	"strings"
)

// print stack trace for debug
func trace(message string) string {
	var pcs [32]uintptr
	n := runtime.Callers(3, pcs[:]) // skip first 3 caller

	var str strings.Builder
	str.WriteString(message + "\nTraceback:")
	for _, pc := range pcs[:n] {
		fn := runtime.FuncForPC(pc)
		file, line := fn.FileLine(pc)
		str.WriteString(fmt.Sprintf("\n\t%s:%d", file, line))
	}
	return str.String()
}

func Recovery() HandlerFunc {
	return func(c *Context) {
		defer func() {
			if err := recover(); err != nil {
				message := fmt.Sprintf("%s", err)
				log.Printf("%s\n\n", trace(message))
				c.Fail(http.StatusInternalServerError, "Internal Server Error")
			}
		}()

		c.Next()
	}
}
```

在 trace() 中，调用了 runtime.Callers(3, pcs[:])，Callers 用来返回调用栈的程序计数器, 第 0 个 Caller 是 Callers 本身，第 1 个是上一层 trace，第 2 个是再上一层的 defer func。因此，为了日志简洁一点，我们跳过了前 3 个 Caller。

接下来，通过 runtime.FuncForPC(pc) 获取对应的函数，在通过 fn.FileLine(pc) 获取到调用该函数的文件名和行号，打印在日志中。

至此，gee 框架的错误处理机制就完成了。

## 使用 Demo
day7-panic-recover/main.go
```
package main

import (
	"net/http"

	"./gee"
)

func main() {
	r := gee.Default()
	r.GET("/", func(c *gee.Context) {
		c.String(http.StatusOK, "Hello Geektutu\n")
	})
	// index out of range for testing Recovery()
	r.GET("/panic", func(c *gee.Context) {
		names := []string{"geektutu"}
		c.String(http.StatusOK, names[100])
	})

	r.Run(":9999")
}
```

接下来进行测试，先访问主页，访问一个有BUG的 /panic，服务正常返回。接下来我们再一次成功访问了主页，说明服务完全运转正常。
```
$ curl "http://localhost:9999"
Hello Geektutu
$ curl "http://localhost:9999/panic"
{"message":"Internal Server Error"}
$ curl "http://localhost:9999"
Hello Geektutu
```

我们可以在后台日志中看到如下内容，引发错误的原因和堆栈信息都被打印了出来，通过日志，我们可以很容易地知道，在day7-panic-recover/main.go:47 的地方出现了 index out of range 错误。
```
2020/01/09 01:00:10 Route  GET - /
2020/01/09 01:00:10 Route  GET - /panic
2020/01/09 01:00:22 [200] / in 25.364µs
2020/01/09 01:00:32 runtime error: index out of range
Traceback:
        /usr/local/Cellar/go/1.12.5/libexec/src/runtime/panic.go:523
        /usr/local/Cellar/go/1.12.5/libexec/src/runtime/panic.go:44
        /tmp/7days-golang/day7-panic-recover/main.go:47
        /tmp/7days-golang/day7-panic-recover/gee/context.go:41
        /tmp/7days-golang/day7-panic-recover/gee/recovery.go:37
        /tmp/7days-golang/day7-panic-recover/gee/context.go:41
        /tmp/7days-golang/day7-panic-recover/gee/logger.go:15
        /tmp/7days-golang/day7-panic-recover/gee/context.go:41
        /tmp/7days-golang/day7-panic-recover/gee/router.go:99
        /tmp/7days-golang/day7-panic-recover/gee/gee.go:130
        /usr/local/Cellar/go/1.12.5/libexec/src/net/http/server.go:2775
        /usr/local/Cellar/go/1.12.5/libexec/src/net/http/server.go:1879
        /usr/local/Cellar/go/1.12.5/libexec/src/runtime/asm_amd64.s:1338

2020/01/09 01:00:32 [500] /panic in 395.846µs
2020/01/09 01:00:38 [200] / in 6.985µs
```

# 扩展
## 补充 HTTP 请求方法
原作者只实现了 GET, POST 路由添加，其他的 PUT, DELETE 等标准 HTTP 方法未实现，实现方法也很简单。

gee.go 中新增以下代码：
```
// PUT defines the method to add PUT request
func (group *RouterGroup) PUT(pattern string, handler HandlerFunc) {
	group.addRoute(http.MethodPut, pattern, handler)
}


// DELETE defines the method to add DELETE request
func (group *RouterGroup) DELETE(pattern string, handler HandlerFunc) {
	group.addRoute(http.MethodDelete, pattern, handler)
}


// PATCH defines the method to add PATCH request
func (group *RouterGroup) PATCH(pattern string, handler HandlerFunc) {
	group.addRoute(http.MethodPatch, pattern, handler)
}

// HEAD defines the method to add HEAD request
func (group *RouterGroup) HEAD(pattern string, handler HandlerFunc) {
	group.addRoute(http.MethodHead, pattern, handler)
}

// OPTIONS defines the method to add OPTIONS request
func (group *RouterGroup) OPTIONS(pattern string, handler HandlerFunc) {
	group.addRoute(http.MethodOptions, pattern, handler)
}

// TRACE defines the method to add TRACE request
func (group *RouterGroup) TRACE(pattern string, handler HandlerFunc) {
	group.addRoute(http.MethodTrace, pattern, handler)
}

// Any registers a route that matches all the HTTP methods.
// GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS.
func (group *RouterGroup) Any(pattern string, handler HandlerFunc) {
	group.GET(pattern, handler)
	group.POST(pattern, handler)
	group.PUT(pattern, handler)
	group.DELETE(pattern, handler)
	group.PATCH(pattern, handler)
	group.HEAD(pattern, handler)
	group.OPTIONS(pattern, handler)
	group.TRACE(pattern, handler)
}
```

## 优雅地重启或停止
由于 gee 参考 gin 的 api，故可以使用 gin 官方文档中的方式实现优雅地重启或停止。

适用于 go 1.8+
```
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"time"

	"gee/gee"
)

func main() {
	listen := ":9999"
	r := gee.Default()
	r.GET("/", func(c *gee.Context) {
		c.String(http.StatusOK, "Hello Geektutu\n")
	})

	// index out of range for testing Recovery()
	r.GET("/panic", func(c *gee.Context) {
		names := []string{"geektutu"}
		c.String(http.StatusOK, names[100])
	})
	
	srv := &http.Server{
		Addr:              listen,
		Handler:           r,
	}
	
	go func() {
		log.Println("Server Start @", listen)
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("Server Start Error: %s\n", err)
		}
	}()
	
	// 等待中断信号以优雅地关闭服务器（设置 5 秒的超时时间）
	quit := make(chan os.Signal)
	signal.Notify(quit, os.Interrupt)
	<-quit
	ctx, cancel := context.WithTimeout(context.Background(), 5 * time.Second)
	defer cancel()
	if err := srv.Shutdown(ctx); err != nil {
		log.Fatal("Server Shutdown Error:", err)
	}
	log.Println("Server Shutdown")
	//r.Run(":9999")
}

```

- ctrl+c 或者 kill pid 就可以关闭服务器了


## 参考 gin 实现 cookie
context.go 中加入以下代码：
```
// SetCookie adds a Set-Cookie header to the ResponseWriter's headers.
// The provided cookie must have a valid Name. Invalid cookies may be
// silently dropped.
func (c *Context) SetCookie(name, value string, maxAge int, path, domain string, secure, httpOnly bool) {
	if path == "" {
		path = "/"
	}
	http.SetCookie(c.Writer, &http.Cookie{
		Name:     name,
		Value:    url.QueryEscape(value),
		MaxAge:   maxAge,
		Path:     path,
		Domain:   domain,
		Secure:   secure,
		HttpOnly: httpOnly,
	})
}

// Cookie returns the named cookie provided in the request or
// ErrNoCookie if not found. And return the named cookie is unescaped.
// If multiple cookies match the given name, only one cookie will
// be returned.
func (c *Context) Cookie(name string) (string, error) {
	cookie, err := c.Req.Cookie(name)
	if err != nil {
		return "", err
	}
	val, _ := url.QueryUnescape(cookie.Value)
	return val, nil
}
```

使用方法：
```
r.GET("/setcookie", func(c *gee.Context) {
	c.SetCookie("gee_cookie", "gee_cookie", 3600, "/", "localhost", false, true)
	c.String(http.StatusOK, "set cookie: gee_cookie")
})

r.GET("/getcookie", func(c *gee.Context) {
	cookie, _ := c.Cookie("gee_cookie")
	c.String(http.StatusOK, "get cookie: %s", cookie)
})
```


## 新增路由 Router 正则表达式匹配
trie.go 更新 insert 方法代码：
```
func (n *node) insert(pattern string, parts []string, height int) {
	if len(parts) == height {
		n.pattern = pattern
		return
	}

	part := parts[height]
	child := n.matchChild(part)
	if child == nil {
		// part[0] == '#' 用于正则匹配
		child = &node{part: part, isWild: part[0] == ':' || part[0] == '*' || part[0] == '#'}
		n.children = append(n.children, child)
	}
	child.insert(pattern, parts, height+1)
}
```

router.go 更新 getRoute 方法代码：
```
func (r *router) getRoute(method string, path string) (*node, map[string]string) {
	searchParts := parsePattern(path)
	params := make(map[string]string)
	root, ok := r.roots[method]

	if !ok {
		return nil, nil
	}

	n := root.search(searchParts, 0)

	if n != nil {
		parts := parsePattern(n.pattern)
		for index, part := range parts {
			if part[0] == ':' {
				params[part[1:]] = searchParts[index]
			}
			if part[0] == '*' && len(part) > 1 {
				params[part[1:]] = strings.Join(searchParts[index:], "/")
				break
			}
			
			// 正则匹配
			if part[0] == '#' {
				splitPart := strings.Split(part[1:], ":")
				rePattern := splitPart[1]
				if rePattern[0] != '^' {
					rePattern = "^" + rePattern
				}
				if rePattern[len(rePattern) - 1] != '$' {
					rePattern = rePattern + "$"
				}
				re := regexp.MustCompile(rePattern)
				if re.MatchString(searchParts[index]) {
					params[splitPart[0]] = searchParts[index]
				} else {
					return nil, nil
				}
			}

		}
		return n, params
	}
	return nil, nil
}
```


使用方法：
```
r.GET("/re1/#id:\\d+", func(c *gee.Context) {
	id:= c.Param("id")
	c.String(http.StatusOK, "re1 id: %s", id)
})

r.GET("/re2/#id:[a-z]+", func(c *gee.Context) {
	id:= c.Param("id")
	c.String(http.StatusOK, "re2 id: %s", id)
})

r.GET("/re3/#year:[12][0-9]{3}/#month:[1-9]{2}/#day:(12|[3-9])", func(c *gee.Context) {
	year := c.Param("year")
	month := c.Param("month")
	day := c.Param("day")
	c.String(http.StatusOK, "re3 year: %s, month: %s, day: %s", year, month, day)
})
```
- 对于性能的影响没测过，未知。
