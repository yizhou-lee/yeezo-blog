[TOC]

# Go 参数校验库 validator

## 概述

`validator` 是一个功能强大的 Go 语言结构体字段校验库，广泛应用于 API、表单数据等场景中的参数校验。通过简单的结构体标签定义规则，`validator` 帮助我们实现了易读、可复用的参数校验逻辑，减少了手写的繁琐校验代码。

## 为什么后端要做参数校验

1. **安全性**：防止恶意用户绕过前端校验，保护系统免受注入攻击等安全威胁。
2. **数据一致性**：统一校验标准，确保各客户端传入的数据符合要求，避免数据不一致。
3. **复杂业务逻辑处理**：处理多字段关联、数据库依赖等复杂校验，前端无法独立完成。

## 快速安装

在项目中使用 `validator`，首先需要安装该库：

```shell
# 第一次安装使用如下命令
$ go get github.com/go-playground/validator/v10
# 项目中引入包
import "github.com/go-playground/validator/v10"
```

## 基础用法

### 1. 定义结构体字段标签

我们可以通过结构体字段上的 validate 标签来指定校验规则。例如：

```go
type User struct {
    Name  string `validate:"required"`       // 必填项
    Email string `validate:"required,email"` // 必填且必须是合法的邮箱格式
    Age   int    `validate:"gte=18,lte=60"`  // 年龄必须在18到60之间
}
```

### 2. 运行校验

使用 validator.New() 创建校验器，然后通过 validate.Struct() 对结构体实例进行校验。示例如下：

```go
package main

import (
    "fmt"
    "github.com/go-playground/validator/v10"
)

type User struct {
    Name  string `validate:"required"`
    Email string `validate:"required,email"`
    Age   int    `validate:"gte=18,lte=60"`
}

func main() {
    user := User{
        Name:  "Alice",
        Email: "alice@example.com",
        Age:   25,
    }
    
    validate := validator.New()
    
    err := validate.Struct(user)
    if err != nil {
        for _, err := range err.(validator.ValidationErrors) {
            fmt.Printf("字段 %s 校验失败，原因：%s\n", err.StructNamespace(), err.Tag())
        }
    } else {
        fmt.Println("所有字段校验通过！")
    }
}
```

### 3. 常用校验规则

| **Rule**   | **Description**                                              | **Example**                      |
| ---------- | ------------------------------------------------------------ | -------------------------------- |
| `required` | Field must not be empty or nil.                              | `validate:"required"`            |
| `min`      | Minimum value for numbers or minimum length for strings/arrays. | `validate:"min=3"`               |
| `max`      | Maximum value for numbers or maximum length for strings/arrays. | `validate:"max=10"`              |
| `len`      | Exact length for strings, arrays, or exact value for numbers. | `validate:"len=5"`               |
| `email`    | Checks if the field is a valid email format.                 | `validate:"email"`               |
| `url`      | Ensures the field is a valid URL format.                     | `validate:"url"`                 |
| `gte`      | Greater than or equal to a specified value.                  | `validate:"gte=18"`              |
| `lte`      | Less than or equal to a specified value.                     | `validate:"lte=100"`             |
| `oneof`    | Field must match one of the specified values.                | `validate:"oneof=red blue"`      |
| `numeric`  | Field must be numeric (string or number).                    | `validate:"numeric"`             |
| `uuid`     | Checks if the field is a valid UUID.                         | `validate:"uuid"`                |
| `datetime` | Field must match a specified date-time format.               | `validate:"datetime=2006-01-02"` |
| `ipv4`     | Field must be a valid IPv4 address.                          | `validate:"ipv4"`                |
| `ipv6`     | Field must be a valid IPv6 address.                          | `validate:"ipv6"`                |
| `contains` | String field must contain a specified substring.             | `validate:"contains=abc"`        |
| `excludes` | Field must not contain a specified substring.                | `validate:"excludes=@!"`         |
| `alpha`    | Field must only contain alphabetic characters.               | `validate:"alpha"`               |
| `alphanum` | Field must contain only alphanumeric characters.             | `validate:"alphanum"`            |
| `bool`     | Field must be a boolean.                                     | `validate:"bool"`                |

These rules can be combined within a single tag, for example: `validate:"required,min=3,max=20"`.

> 详见[官方文档](https://pkg.go.dev/github.com/go-playground/validator?utm_source=godoc#hdr-Baked_In_Validators_and_Tags)

### 4. 自定义校验规则

在实际业务中，我们可能会遇到 validator 内置规则无法满足的校验需求。我们可以通过 `RegisterValidation` 方法来自定义校验逻辑。例如，要求用户名只能包含字母字符：

```go
package main

import (
    "fmt"
    "regexp"
    "github.com/go-playground/validator/v10"
)

type User struct {
    Name  string `validate:"alphaonly"`
    Email string `validate:"required,email"`
    Age   int    `validate:"gte=18,lte=60"`
}

func main() {
    validate := validator.New()

    // 注册自定义校验规则
    validate.RegisterValidation("alphaonly", func(fl validator.FieldLevel) bool {
        return regexp.MustCompile(`^[a-zA-Z]+$`).MatchString(fl.Field().String())
    })
    
    user := User{
        Name:  "Alice123",
        Email: "alice@example.com",
        Age:   25,
    }
    
    err := validate.Struct(user)
    if err != nil {
        fmt.Println("校验失败:", err)
    }
}
```

在这个示例中，如果 User 结构体的 Name 字段包含非字母字符，则会触发校验失败。

## 原理剖析

- 反射机制

  - 在 `validator` 中，反射机制用于动态访问结构体的字段及其标签，从而根据标签指定的规则进行验证。通过 `reflect` 包，`validator` 可以在运行时获取字段的值、类型及其标签，进而根据标签中的规则（如 `required`, `gte=18`）执行验证。反射使得验证过程能够适应各种类型和字段，而无需预先定义特定的验证逻辑，但也带来了性能上的开销。

- 缓存机制

  - `validator` 库使用了缓存机制来提高性能。具体来说，它通过以下几个方面来减少重复的计算：

    1. **字段验证规则缓存**：
       每次执行验证时，`validator` 会检查结构体的字段及其验证规则。为了避免每次都重新解析和处理字段标签，`validator` 会将解析后的规则缓存起来。这样，在同一个结构体上执行多次验证时，重复的解析和计算会被跳过。

    2. **结构体标签解析缓存**：
       在执行验证时，`validator` 会解析结构体字段的标签并根据标签生成相应的验证规则。这一过程会被缓存，避免重复解析相同的标签。

    3. **自定义规则缓存**：
       如果使用了自定义验证规则，`validator` 会缓存这些规则及其对应的执行逻辑。当规则被多次调用时，`validator` 会直接使用缓存的验证逻辑而不是每次都重新构建。

    这种缓存机制显著减少了重复的计算开销，提升了验证的性能，尤其是在对多个数据结构进行验证时。

  - 例子：

    ```go
    type Person struct {
        Name string `validate:"required"`
        Age  int    `validate:"gte=18"`
    }
    
    func main() {
        person := Person{Name: "Alice", Age: 25}
        validate := validator.New()
        
        // 第一次验证
        // 1. 反射结构体 Person，检查字段 Name 和 Age
        // 2. 解析验证规则，比如 required 和 gte=18
        // 3. 执行验证逻辑，返回验证结果
        if err := validate.Struct(person); err != nil {
            fmt.Println(err)
        }
        
        // 第二次验证
        // 1. 检查缓存中是否已存在结构体字段和验证规则的解析结果
        // 2. 如果存在缓存，它直接复用缓存的信息，而不需要再次解析和验证字段
        if err := validate.Struct(person); err != nil {
            fmt.Println(err)
        }
    }
    ```

  - 缓存失效：
    1. 结构体定义变化
    2. 自定义验证规则的变化
    3. 调用 `ClearCache()` 方法
    4. 不同的 `validator` 实例：多个 `validator` 实例之间的缓存是独立的

## 在公司项目中的应用

### 目前代码

- 部分直接使用 `httputil.ParseRequestToJson()` 读取 HTTP 请求的 body，将其解析为 JSON 并存储到指定的结构体中，未进行参数验证
- 还有部分使用了 `urlparam.Unmarshal()` ，通过 `req` 标签对参数进行简单的验证，然后将请求内容解析
  - `req` 标签支持功能：
    - 字段名称映射：`Field1 string req:"field1"`
    - 忽略字段：`Field2 string req:"-"`
    - 可选字段：`Field3 string req:"field3,omitempty`
  - 问题：
    - 支持解析的数据类型有缺（无法处理数组、切片、布尔值）
    - 每次调用 `FormValue` 都会遍历整个请求的表单数据，可能会导致性能问题

### 改进

在公司项目中，我们可以将 validator 应用于需要参数校验的模块，特别是在处理用户输入的 API 层。例如：

1. 定义结构体校验规则：在项目中的请求参数结构体上添加校验标签。

2. 解析请求体：将请求的数据解析到结构体中。

3. 执行校验并返回错误信息：校验不通过时，返回详细的错误信息，便于用户或开发人员快速定位问题。

4. 错误处理：在实际业务逻辑中调用校验函数，错误时返回统一格式的响应。

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"

    "github.com/go-playground/validator/v10"
    "github.com/gorilla/mux"
)

// 定义请求结构体
type CreateUserRequest struct {
    Name  string `json:"name" validate:"required"`
    Email string `json:"email" validate:"required,email"`
    Age   int    `json:"age" validate:"gte=0,lte=130"`
}

// 定义响应结构体
type ErrorResponse struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}

// 初始化验证器
var validate = validator.New()

// 处理创建用户请求的处理器
func CreateUserHandler(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest

    // 解析请求体
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }

    // 验证请求参数
    if err := validate.Struct(req); err != nil {
        var errors []ErrorResponse
        for _, err := range err.(validator.ValidationErrors) {
            errors = append(errors, ErrorResponse{
                Field:   err.Field(),
                Message: fmt.Sprintf("Validation failed on '%s' tag", err.Tag()),
            })
        }
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusBadRequest)
        json.NewEncoder(w).Encode(errors)
        return
    }

    // 处理业务逻辑
    // ...

    w.WriteHeader(http.StatusCreated)
    w.Write([]byte("User created successfully"))
}

func main() {
    r := mux.NewRouter()
    r.HandleFunc("/users", CreateUserHandler).Methods("POST")

    http.ListenAndServe(":8080", r)
}
```

## 参考资源

- [validator 官方文档](https://pkg.go.dev/github.com/go-playground/validator/v10)