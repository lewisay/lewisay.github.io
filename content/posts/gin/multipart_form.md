---
title: "Gin框架文件上传异常"
date: 2020-05-20T22:36:24+08:00
draft: false
---

#### 问题描述

Gin 文件上传时, 指定文件内容为空会遇到 ```unexpected end of JSON input``` 错误, 核心代码如下
<!--more-->

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

#### 产生原因

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
        // file 为 reflect.Struct 执行 default 分支
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

#### 解决方案

若需要文件为可选，上传时不应传递指定字段

#### 意外收获

```go
func tryToSetValue(value reflect.Value, field reflect.StructField, setter setter, tag string) (bool, error) {
	var tagValue string
	var setOpt setOptions

	// tag == form
	tagValue = field.Tag.Get(tag)
	tagValue, opts := head(tagValue, ",")

	if tagValue == "" { // default value is FieldName
		tagValue = field.Name
	}
	if tagValue == "" { // when field is "emptyField" variable
		return false, nil
	}

	var opt string
	for len(opts) > 0 {
		// opts = default=xxxx
		opt, opts = head(opts, ",")

		// 获取 default 对应的值
		if k, v := head(opt, "="); k == "default" {
			setOpt.isDefaultExists = true
			setOpt.defaultValue = v
		}
	}

	return setter.TrySet(value, field, tagValue, setOpt)
}
```
这里判断 field tag 中是否存在 default 标记，意味着可以给字段增加默认值，使用方式如下

```go
type BindFile struct {
    Name       string                   `form:"name,default=lewisay" binding:"required"`
    Email string                        `form:"email,default=lewisay@163.com" binding:"required"`
    File       *multipart.FileHeader    `form:"file,default={\"Size\":0}"`
}
```
