---
title: "Gin multipart form binding"
date: 2020-05-20T22:36:24+08:00
draft: false
---

Gin 文件上传时, 若不指定需要上传的文件会遇到 ```unexpected end of JSON input``` 错误, 核心代码如下

```go
type BindFile struct {
    Name       string                   `form:"name" binding:"required"`
    Email string                        `form:"email" binding:"required"`
    File       *multipart.FileHeader    `form:"file"`
}

var bindFile BindFile
if err := c.Bind(&bindFile); err != nil {
    c.String(http.StatusBadRequest, fmt.Sprintf("err: %s", err.Error()))
    return
}
```