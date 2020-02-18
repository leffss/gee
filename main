package main

/*
$ curl "http://localhost:9999"
Hello Geektutu
$ curl "http://localhost:9999/panic"
{"message":"Internal Server Error"}
$ curl "http://localhost:9999"
Hello Geektutu

>>> log
2020/01/09 01:00:10 Route  GET - /
2020/01/09 01:00:10 Route  GET - /panic
2020/01/09 01:00:22 [200] / in 25.364µs
2020/01/09 01:00:32 runtime error: index out of range
Traceback:
        /usr/local/Cellar/go/1.12.5/libexec/src/runtime/panic.go:523
        /usr/local/Cellar/go/1.12.5/libexec/src/runtime/panic.go:44
        /Users/7days-golang/day7-panic-recover/main.go:47
        /Users/7days-golang/day7-panic-recover/gee/context.go:41
        /Users/7days-golang/day7-panic-recover/gee/recovery.go:37
        /Users/7days-golang/day7-panic-recover/gee/context.go:41
        /Users/7days-golang/day7-panic-recover/gee/logger.go:15
        /Users/7days-golang/day7-panic-recover/gee/context.go:41
        /Users/7days-golang/day7-panic-recover/gee/router.go:99
        /Users/7days-golang/day7-panic-recover/gee/gee.go:130
        /usr/local/Cellar/go/1.12.5/libexec/src/net/http/server.go:2775
        /usr/local/Cellar/go/1.12.5/libexec/src/net/http/server.go:1879
        /usr/local/Cellar/go/1.12.5/libexec/src/runtime/asm_amd64.s:1338

2020/01/09 01:00:32 [500] /panic in 395.846µs
2020/01/09 01:00:38 [200] / in 6.985µs
*/

import (
	"context"
	"log"
	"net/http"
	"os"
	"os/signal"
	"time"

	"gee/gee"
)

func Test(v string) gee.HandlerFunc {
	log.Printf("Init Test Middleware")
	return func(c *gee.Context) {
		c.Set("test", v)
		// Process request
		c.Next()
	}
}

func addServerName(v string) gee.HandlerFunc {
	log.Printf("Init addServerName Middleware")
	return func(c *gee.Context) {
		c.SetHeader("Server", v)
		c.Next()
	}
}

func main() {
	listen := ":9999"
	r := gee.Default()
	v := "30"
	server := "MyServer"
	r.Use(Test(v))
	r.Use(addServerName(server))
	r.GET("/", func(c *gee.Context) {
		c.String(http.StatusOK, "Hello Geektutu\n")
	})

	r.GET("/test1/:name/test2", func(c *gee.Context) {
		x := c.Param("name")
		c.String(http.StatusOK, x)
	})

	r.GET("/test2/*name", func(c *gee.Context) {
		x := c.Param("name")
		c.String(http.StatusOK, x)
	})

	r.GET("/re1/{id:\\d+}", func(c *gee.Context) {
		id:= c.Param("id")
		c.String(http.StatusOK, "re1 id: %s", id)
	})

	r.GET("/re2/{id:[a-z]+}", func(c *gee.Context) {
		id:= c.Param("id")
		c.String(http.StatusOK, "re2 id: %s", id)
	})

	r.GET("/re3/{year:[12][0-9]{3}}/{month:[1-9]{2}}/{day:[1-9]{2}}/{hour:(12|[3-9])}", func(c *gee.Context) {
		year := c.Param("year")
		month := c.Param("month")
		day := c.Param("day")
		hour := c.Param("hour")
		c.String(http.StatusOK, "re3 year: %s, month: %s, day: %s, hour: %s", year, month, day, hour)
	})

	// index out of range for testing Recovery()
	r.Any("/panic", func(c *gee.Context) {
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
