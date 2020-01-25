# Gin学习笔记

## Gin常用方法

```go
r.GET("/demo", func(c *gin.Context) {
		// func (c *Context) Data(code int, contentType string, data []byte) 将字节数据写入并更新状态码
		// c.SetCookie()
		// c.Cookie()
		c.Header("Zabbix", "tools-linux") // 响应请求头部增加一个kv键
		c.Header("Zabbix", "")            // 如果不写v 则删除响应请求头中的key，(Content-Type,Content-Length,Data)这3个字段无法删除
		c0 := c.GetHeader("User-Agent")   // 获取请求头部的key,没有则是空字符串
		c1 := c.ClientIP()                // 获取用户请求的ip
		c2 := c.ContentType()             // 获取headers里的ContentType字段
		c3 := c.HandlerName()             // 获取执行函数的名字
		c4 := c.FullPath()                // 获取请求的路径 /demo
		// key str, v interface{} 设置一个上下文访问的变量
		c.Set("Nstring", "cby")
		c.Set("Nbool", true)
		// GetInt GetInt64 GetFloat64 GetStringMap GetStringSlice GetDuration GetTime...
		v := c.GetString("Nstring")
		v1 := c.GetBool("Nbool")
        // c.GetQuery()(string, bool) 判断获取的key是否存在，返回值和布尔值
		// c.QueryArray()
        // 省略...
    
        // 输出更好看的json，但是消耗cpu和带宽
		c.IndentedJSON(200, gin.H{
			// c.SecureJSON(200, gin.H{
			"msg":  v,
			"name": v1,
		})
}
```

## 路由组

```go
func add(c *gin.Context) {} //伪代码
func del(c *gin.Context) {} //伪代码

func main() {
	r := gin.Default()
	v1 := r.Group("/v1")
    // {}书写规范，把同一个路由组的路由用{}包起来
	{
		v1.GET("/add", add)
		v1.GET("/del", del)
	}
	v2 := r.Group("/v2")
	{
		v2.POST("/add", add)
		v2.POST("/del", del)
	}
    r.Run("0.0.0.0:80")
}
```

```bash
curl -X GET "http://127.0.0.1/v1/add"
curl -X POST "http://127.0.0.1/v2/del"
```

## 设置默认路由，匹配不到的路由返回404

```go
// Any可以匹配所有请求方法(get,post,put.delete等)
r.Any("/test", func(c *gin.Context) {...}) //伪代码
// 可以匹配到没有的请求方法,捕获405
r.NoMethod(func(c *gin.Context){...}) //伪代码
// 匹配不到的路由返回404html
r.NoRoute(func(c *gin.Context) {
		c.HTML(http.StatusNotFound, "404.html", nil)
})
```

## HTTP重定向

```go
// 内部、外部重定向都支持
r.GET("/test", func(c *gin.Context) {
	c.Redirect(http.StatusMovedPermanently, "https://www.baidu.com/")
})
```

## 通过url传递参数

```go
package main

import (
	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()
	r.GET("/:name/:id", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"name": c.Param("name"),
			"id":   c.Param("id"),
		})
	})
	r.Run("0.0.0.0:80")
}
```

```bash
curl -XGET "http://127.0.0.1/cby/27"
{"id":"27","name":"cby"} #浏览器返回
```

## 泛绑定，所有访问user/*路径的请求都会被匹配到

```go
import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()
    // 注册/user/*路由
	r.GET("/user/*action", func(c *gin.Context) {
		action := c.Param("action")
		c.JSON(http.StatusOK, gin.H{
			"action": action,
		})
	})
	r.Run("0.0.0.0:80")
}
```

```bash
curl -XGET "http://127.0.0.1/user/cby/age/27"
{"action":"/cby/age/27"} #浏览器返回
```

## 通过API ? 传参的值

```go
func main() {
	r := gin.Default()
	r.GET("/test", func(c *gin.Context) {
		name := c.Query("name")             // 获取?name=xxx的值,若不存在返回字符串空值
		age := c.DefaultQuery("age", "18")  // 获取?age=xxx的值,默认值为“18”
		c.JSON(200, gin.H{
			"name": name,
			"age":  age,
		})
	})
	r.Run("0.0.0.0:80")
}
```

```bash
curl -X GET "http://127.0.0.1/test?name=cby&age=27"
{"age": "27", "name": "cby"} #POSTMAN返回
```

## 获取form表单数据

```go
r.POST("/form", func(c *gin.Context) {
        // 获取form表单里的key=username的值,默认值为"root"
		username := c.DefaultPostForm("username", "root")
        // 获取form表单里的key=password的值
		password := c.PostForm("password")
        // 获取form表单里的key=hobbys的值,hobbys是多选框,获取的值为数组
		hobbys := c.PostFormArray("hobbys")
		c.String(200, "user is %s, passwd is %s, hobbys is %v",
            username, password, hobbys)
})
```

<img src="C:\Users\cby\AppData\Roaming\Typora\typora-user-images\1571644457877.png" alt="1571644457877" style="zoom:80%;" />

## 参数绑定，提供开发效率

| 使用PostForm这种单个获取属性和字段的方式，代码量较多，需要一个一个属性进行获取。而表单数据的提交，往往对应着完整的数据结构体定义，其中对应着表单的输入项。gin框架提供了数据结构体和表单提交数据绑定的功能，提高表单数据获取的效率。如下所示： |
| ------------------------------------------------------------ |
| 以一个用户注册功能来进行讲解表单实体绑定操作。用户注册需要提交表单数据，假设注册时表单数据包含2项，分别为：username 和 password。 |

```go
type Login struct {
    Username string form:"username" binding:"required"
    Password string form:"password" binding:"required"
}
```

| 创建了Login结构体用于接收表单数据，通过tag标签的方式设置每个字段对应的form表单中的属性名，通过binding属于设置属性是否是必须。 |
| ------------------------------------------------------------ |
|                                                              |

```go
// 使用ShouldBind可以实现Post方式的提交数据的绑定工作
r.POST("/loginForm", func(c *gin.Context) {
	var login Login
	// ShouldBind()会根据请求的Content-Type自行选择绑定器
	if err := c.ShouldBind(&login); err == nil {
		c.JSON(http.StatusOK, gin.H{
			"username": login.Username,
			"password": login.Password,
		})
	} else {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
	}
})

// 使用ShouldBindQuery可以实现Get方式的数据请求的绑定
r.GET("/loginAPI", func(c *gin.Context) {
	var login Login
	if err := c.ShouldBindQuery(&login); err == nil {
		fmt.Printf("login info:%#v\n", login)
        c.Writer.Write([]byte(login.Username))
	} else {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
	}
})

// 当客户端使用Json格式进行数据提交时，可以采用ShouldBindJson对数据进行绑定并自动解析
// 需要在结构体里面增加json的tag
r.POST("/loginJSON", func(c *gin.Context) {
	var login Login
	if err := c.ShouldBindJSON(&login); err == nil {
		fmt.Printf("login info:%#v\n", login)
		c.JSON(http.StatusOK, gin.H{
			"username": login.Username,
			"password": login.Password,
		})
	} else {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
	}
})
```

## 返回数据渲染到HTML模板

```go
r := gin.Default()
r.LoadHTMLGlob("templates/*") // 指定html目录
r.StaticFS("/static", http.Dir("/var/www/static")) // 指定静态文件目录
// r.Static("/static", "/var/www/static")
r.GET("/", func(c *gin.Context) {
	// 返回3个参数，状态码，html文件，渲染的数据(没有可以用nil代替)
    c.HTML(200, "upload.html", gin.H{
		"message": fmt.Sprintf("%s upload Success!", file.Filename),
	})
}
```

## HTML模板格式

```html
<html>
    <head></head>
    <body>
        <div>{{ .name }}</div> // .名称 表示map或结构体
        <div>
            {{range .data }}   // 遍历data map里面的字段，其中ID就是其中之一
            <a href="/book/list?id={{.ID}}">查询</a>
        </div>
    </body>
```

## form表单上传文件

```go
// 上传文件需要把form表单enctype属性设置为 enctype="multipart/form-data"
// 处理multipart form提交文件时默认的内存限制是32 MiB, 改成2048MiB
r.MaxMultipartMemory = 2048 << 20
r.POST("/upload", func(c *gin.Context) {
	file, err := c.FormFile("files") // 获取form表单input标签type=file name=files的标签
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{
			"message":  err.Error(),
			"StatusOk": "error",
		})
		return
	}
	log.Println(file.Filename)
	dst := fmt.Sprintf("/home/aiadmin/%s", file.Filename)
	err = c.SaveUploadedFile(file, dst)  // 上传到指定路径 dst
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{
			"message":  err.Error(),
			"StatusOk": "error",
		})
		return
	}
	c.HTML(200, "upload.html", nil)
})
```

## form表单上传多个文件

```go
r.POST("/upload", func(c *gin.Context) {
	form, _ := c.MultipartForm()
	files := form.File["file"]
    
	for index, file := range files {
		log.Println(file.Filename)
		dst := fmt.Sprintf("C:/tmp/%s_%d", file.Filename, index)
        _ := c.SaveUploadedFile(file, dst) // 上传文件到指定的目录
	}
	c.JSON(http.StatusOK, gin.H{
		"message": fmt.Sprintf("%d files uploaded!", len(files)),
	})
})
```

## Gin中间件

```go
package main

import (
	"log"
	"net/http"
	"time"

	"github.com/gin-gonic/gin"
)

// IpAllow 自定义中间件，限制用户请求的ip
func IpAllow() gin.HandlerFunc {
	return func(c *gin.Context) {
		ipList := []string{
			"127.0.0.1",
		}
		flag := false
		clientIp := c.ClientIP()
		for _, host := range clientIp {
			if clientIp == host {
				flag = true
				break
			}
		}
		if !flag {
			c.AbortWithStatusJSON(404, gin.H{
				"msg":      "error",
				"StatusOk": "no",
			})
		}
	}
}

// StatCost 是一个统计耗时请求耗时的中间件
func StatCost() gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()
		c.Set("name", "小王子")
		c.Next() // 执行下一个视图函数
		cost := time.Since(start)
		log.Println(cost)
	}
}

func main() {
	// 新建一个没有任何默认中间件的路由
	r := gin.New()
	// 注册一个全局中间件
	r.Use(IpAllow())
	r.GET("/test", func(c *gin.Context) {
		name := c.MustGet("name").(string)
		log.Println(name)
		c.JSON(http.StatusOK, gin.H{
			"message": "Hello world!",
		})
	})
	// 给/test2路由单独注册中间件（可注册多个）
	r.GET("/test2", StatCost(), func(c *gin.Context) {
		name := c.MustGet("name").(string)
		log.Println(name)
		c.JSON(http.StatusOK, gin.H{
			"message": "Hello world!",
		})
	})
	r.Run("0.0.0.0:80")
}
```
