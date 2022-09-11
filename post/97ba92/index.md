# Gin中间件执行顺序


### 初始化demo项目

```bash
mkdir -p gin-middleware-demo
cd gin-middleware-demo
go mod init github.com/kbsonlong/gin-middleware-demo
```
<!--more-->

#### middleware/middle.go
```go
package middleware

import (
	"fmt"
	"time"

	"github.com/gin-gonic/gin"
)

func MdOne() gin.HandlerFunc {
	return func(c *gin.Context) {
		fmt.Println("开始执行第一个 gin 中间件:" + time.Now().String())
		c.Next()

		fmt.Println("第一个 gin 中间件返回内容:" + time.Now().String())
	}
}

func MdTwo() gin.HandlerFunc {
	return func(c *gin.Context) {
		fmt.Println("开始执行第二个 gin 中间件:" + time.Now().String())
		c.Next()

		fmt.Println("第二个 gin 中间件返回内容:" + time.Now().String())
	}
}

func MdThree() gin.HandlerFunc {
	return func(c *gin.Context) {
		fmt.Println("开始执行第三个 gin 中间件:" + time.Now().String())
		c.Next()

		fmt.Println("第三个 gin 中间件返回内容:" + time.Now().String())
	}
}

```

#### routers/router.go
```go
package routers

import (
	"github.com/gin-gonic/gin"
	"github.com/kbsonlong/gin-middleware-demo/middleware"
)

func InitRouter() *gin.Engine {
	r := gin.New()
	r.Use(middleware.MdOne())
	r.Use(middleware.MdTwo())
	r.Use(middleware.MdThree())
	gin.SetMode("debug")

	r.GET("/", func(ctx *gin.Context) {
		ctx.JSON(200, gin.H{
			"message": "test",
		})
	})
	return r
}

```

#### main.go
```go
package main

import (
	"github.com/kbsonlong/gin-middleware-demo/routers"
)

func main() {

	router := routers.InitRouter()

	router.Run()
}

```

### 运行测试
```bash
go mod tidy
go run main.go
```

```text
[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:   export GIN_MODE=release
 - using code:  gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /                         --> github.com/kbsonlong/gin-middleware-demo/routers.InitRouter.func1 (4 handlers)
[GIN-debug] [WARNING] You trusted all proxies, this is NOT safe. We recommend you to set a value.
Please check https://pkg.go.dev/github.com/gin-gonic/gin#readme-don-t-trust-all-proxies for details.
[GIN-debug] Environment variable PORT is undefined. Using port :8080 by default
[GIN-debug] Listening and serving HTTP on :8080
```

另一终端执行
```bash
curl 127.0.0.1:8080
{"message":"test"}
```

可以看到控制台输出内容
```text
开始执行第一个 gin 中间件:2022-07-08 17:18:57.499033 +0800 CST m=+3.029327751
开始执行第二个 gin 中间件:2022-07-08 17:18:57.499215 +0800 CST m=+3.029509918
开始执行第三个 gin 中间件:2022-07-08 17:18:57.499218 +0800 CST m=+3.029513043
第三个 gin 中间件返回内容:2022-07-08 17:18:57.499295 +0800 CST m=+3.029589876
第二个 gin 中间件返回内容:2022-07-08 17:18:57.499298 +0800 CST m=+3.029593335
第一个 gin 中间件返回内容:2022-07-08 17:18:57.4993 +0800 CST m=+3.029594876
```

#### 调整中间件Use顺序
```golang
......
	r.Use(middleware.MdOne())

	r.Use(middleware.MdThree())
	r.Use(middleware.MdTwo())
	gin.SetMode("debug")
......
```
> 第二个中间件和第三个中间件顺序对调

```text
开始执行第一个 gin 中间件:2022-07-08 17:20:57.702026 +0800 CST m=+2.567931210
开始执行第三个 gin 中间件:2022-07-08 17:20:57.702215 +0800 CST m=+2.568120210
开始执行第二个 gin 中间件:2022-07-08 17:20:57.702218 +0800 CST m=+2.568123543
第二个 gin 中间件返回内容:2022-07-08 17:20:57.702305 +0800 CST m=+2.568210460
第三个 gin 中间件返回内容:2022-07-08 17:20:57.702308 +0800 CST m=+2.568214043
第一个 gin 中间件返回内容:2022-07-08 17:20:57.702311 +0800 CST m=+2.568216293
```
可以看到第三个中间件在第二个中间件之前执行


#### 注释第三个中间件Next
```golang
func MdThree() gin.HandlerFunc {
	return func(c *gin.Context) {
		fmt.Println("开始执行第三个 gin 中间件:" + time.Now().String())

		// c.Next()
		fmt.Println("第三个 gin 中间件返回内容:" + time.Now().String())
	}
}
```

```text
开始执行第一个 gin 中间件:2022-07-08 17:22:43.132983 +0800 CST m=+4.061931376
开始执行第三个 gin 中间件:2022-07-08 17:22:43.133126 +0800 CST m=+4.062074668
第三个 gin 中间件返回内容:2022-07-08 17:22:43.133128 +0800 CST m=+4.062076751
开始执行第二个 gin 中间件:2022-07-08 17:22:43.13313 +0800 CST m=+4.062078210
第二个 gin 中间件返回内容:2022-07-08 17:22:43.133203 +0800 CST m=+4.062151335
第一个 gin 中间件返回内容:2022-07-08 17:22:43.133205 +0800 CST m=+4.062153460
```

### 总结
Gin中间件的调用顺序与Use顺序有关，代码运行顺序和Next前后顺序有关。
- Next之前代码先进先出
- Next之后代码后进先出
- 没有引用Next代码直接运行


