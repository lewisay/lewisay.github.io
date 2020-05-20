---
title: "Gin multipart form binding"
date: 2020-05-20T22:36:24+08:00
draft: false
---

> 问题描述

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

> 产生原因

```go
func (r *multipartRequest) TrySet(value reflect.Value, field reflect.StructField, key string, opt setOptions) (isSetted bool, err error) {
    // 指定了 file 字段，但内容为空
	if files := r.MultipartForm.File[key]; len(files) != 0 {
		return setByMultipartFormFile(value, field, files)
	}

    // 由于内容为空，会执行以下逻辑
	return setByForm(value, field, r.MultipartForm.Value, key, opt)
}

func setByForm(value reflect.Value, field reflect.StructField, form map[string][]string, tagValue string, opt setOptions) (isSetted bool, err error) {
	vs, ok := form[tagValue]
	if !ok && !opt.isDefaultExists {
		return false, nil
	}

	switch value.Kind() {
	case reflect.Slice:
		if !ok {
			vs = []string{opt.defaultValue}
		}
		return true, setSlice(vs, value, field)
	case reflect.Array:
		if !ok {
			vs = []string{opt.defaultValue}
		}
		if len(vs) != value.Len() {
			return false, fmt.Errorf("%q is not valid value for %s", vs, value.Type().String())
		}
		return true, setArray(vs, value, field)
    default:
        // 实际执行 default 分支
		var val string
		if !ok {
			val = opt.defaultValue
		}

		if len(vs) > 0 {
			val = vs[0]
		}
		return true, setWithProperType(val, value, field)
	}
}

func setWithProperType(val string, value reflect.Value, field reflect.StructField) error {
	switch value.Kind() {
	...
	case reflect.String:
		value.SetString(val)
	case reflect.Struct:
		switch value.Interface().(type) {
		case time.Time:
			return setTimeField(val, field, value)
        }
        // 最终执行这里，由于 val 无法正确解析，导致 json.unmarshal 返回 unexpected end of JSON input
		return json.Unmarshal(bytesconv.StringToBytes(val), value.Addr().Interface())
	case reflect.Map:
		return json.Unmarshal(bytesconv.StringToBytes(val), value.Addr().Interface())
	default:
		return errUnknownType
	}
	return nil
}
```

> 解决方案
若需要文件为可选，上传时不应传递指定字段