# Validate 1.6: A Generic Data Validation and Filtering Library

`validate` 是一个通用的 Go 数据验证和过滤工具库。

- 支持快速验证 `Map`、`Struct`、`Request`（`Form`、`JSON`、`url.Values`、`UploadedFile`）数据
  - 验证 `http.Request` 时会根据请求的 `Content-Type` 值自动收集数据
  - 支持检查切片中的每个子值。例如：`v.StringRule("tags.*", "required|string")`
- 支持在验证之前对数据进行过滤/清理/转换
- 支持添加自定义过滤器/验证器函数
- 支持场景设置，在不同场景中验证不同的字段
- 支持自定义错误消息、字段翻译。
  - 可在结构体中使用 `message`、`label` 标签
- 可自定义国际化感知的错误消息，内置 `en`、`zh-CN`、`zh-TW`
- 内置常用数据类型过滤器/转换器。参见内置过滤器
- 已内置许多常用验证器（> 70 个），参见内置验证器
- 可在任何框架中使用 `validate`，如 Gin、Echo、Chi 等
- 支持直接使用规则验证值。例如：`validate.Val("xyz@mail.com", "required|email")`

v2.0：模块路径现在是 `github.com/gookit/validate/v2`（需要 Go 1.21+）。从 v1.x 迁移请参阅升级指南。

性能：v1.5.7 → v1.6.0 → v2.0.0 的基准测试结果请参见基准对比。

安装：

```
go get github.com/gookit/validate/v2
```

使用结构体的 `validate` 标签，可以快速配置一个结构体。

结构体的字段翻译和错误消息可以通过 `message` 和 `label` 标签快速配置。

- 支持通过结构体标签配置字段映射，默认读取 `json` 标签的值
- 支持通过结构体的 `message` 标签配置错误消息
- 支持通过结构体的 `label` 标签配置字段翻译

```go
package main

import (
	"fmt"
	"time"

	"github.com/gookit/validate/v2"
)

// UserForm 结构体
type UserForm struct {
	Name     string `validate:"required|min_len:7" message:"required:{field} is required" label:"用户名"`
	Email    string `validate:"email" message:"email is invalid" label:"用户邮箱"`
	Age      int    `validate:"required|int|min:1|max:99" message:"int:age must int|min:age min value is 1"`
	CreateAt int    `validate:"min:1"`
	Safe     int    `validate:"-"`
	UpdateAt time.Time `validate:"required" message:"update time is required"`
	Code     string `validate:"customValidator"`
	// ExtInfo 嵌套结构体
	ExtInfo struct{
		Homepage string `validate:"required" label:"主页"`
		CityName string
	} `validate:"required" label:"主页"`
}

// CustomValidator 源结构体中的自定义验证器。
func (f UserForm) CustomValidator(val string) bool {
	return len(val) == 4
}
```

`validate` 提供了扩展功能：

结构体可以实现三个接口方法，便于进行一些自定义：

- `ConfigValidation(v *Validation)` 将在验证器实例创建后被调用
- `Messages() map[string]string` 可以自定义验证器错误消息
- `Translates() map[string]string` 可以自定义字段翻译

```go
package main

import (
	"fmt"
	"time"

	"github.com/gookit/validate/v2"
)

// UserForm 结构体
type UserForm struct {
	Name     string `validate:"required|min_len:7"`
	Email    string `validate:"email"`
	Age      int    `validate:"required|int|min:1|max:99"`
	CreateAt int    `validate:"min:1"`
	Safe     int    `validate:"-"`
	UpdateAt time.Time `validate:"required"`
	Code     string `validate:"customValidator"`
	// ExtInfo 嵌套结构体
	ExtInfo struct{
		Homepage string `validate:"required"`
		CityName string
	} `validate:"required"`
}

// CustomValidator 源结构体中的自定义验证器。
func (f UserForm) CustomValidator(val string) bool {
	return len(val) == 4
}

// ConfigValidation 配置验证
// 例如：
// - 定义验证场景
func (f UserForm) ConfigValidation(v *validate.Validation) {
	v.WithScenes(validate.SValues{
		"add":    []string{"ExtInfo.Homepage", "Name", "Code"},
		"update": []string{"ExtInfo.CityName", "Name"},
	})
}

// Messages 可以自定义验证器错误消息。
func (f UserForm) Messages() map[string]string {
	return validate.MS{
		"required":      "噢！{field} 是必填的",
		"email":         "邮箱格式无效",
		"Name.required": "特定字段的消息",
		"Age.int":       "年龄必须是整数",
		"Age.min":       "年龄最小值为 1",
	}
}

// Translates 可以自定义字段翻译。
func (f UserForm) Translates() map[string]string {
	return validate.MS{
		"Name":             "用户名",
		"Email":            "用户邮箱",
		"ExtInfo.Homepage": "主页",
	}
}
```

可以使用 `validate.Struct(ptr)` 快速创建验证实例，然后调用 `v.Validate()` 进行验证。

```go
package main

import (
	"fmt"

	"github.com/gookit/validate/v2"
)

func main() {
	u := &UserForm{
		Name: "inhere",
	}

	v := validate.Struct(u)
	// v := validate.New(u)

	if v.Validate() { // 验证通过
		// 执行某些操作 ...
	} else {
		fmt.Println(v.Errors)             // 所有错误消息
		fmt.Println(v.Errors.One())        // 返回一条随机错误消息文本
		fmt.Println(v.Errors.OneError())   // 返回一个随机错误
		fmt.Println(v.Errors.Field("Name")) // 返回该字段的错误消息
	}
}
```

默认情况下（`CheckSubOnParentMarked=true`），命名的子结构体字段（`struct` / `*struct` / 结构体切片 / 结构体映射）只有在携带 `validate` 标签时才会被递归进入以收集其内部规则。标签值可以为空——`validate:""` 足以标记该字段。没有任何 `validate` 标签的命名字段不会被递归进入。这是 Java `@Valid` 风格的按需级联方式，避免了对整个结构体树的无意义递归。

匿名嵌入结构体不受此限制：`type Bar struct { Foo }`（字段被提升，属于父结构体的一部分）始终会级联，无需标签。只有命名的嵌套字段需要标记。

```go
type Address struct {
	City string `validate:"required"`
}

type User struct {
	Name string  `validate:"required"`
	Home Address `validate:"required"` // 有标签 -> 递归进入，验证 Home.City
	Work Address `validate:""`         // 空标签也是标记 -> 递归进入
	Temp Address                        // 无标签 -> 不递归进入，Temp.City 被忽略
}
```

若要全局恢复 v1 的无条件级联行为：

```go
validate.Config(func(o *validate.GlobalOption) {
	o.CheckSubOnParentMarked = false
})
```

更多关于此行为变更的信息请参阅升级指南。

也可以直接验证 MAP 数据。

```go
package main

import (
	"fmt"

	"github.com/gookit/validate/v2"
)

func main() {
	m := map[string]any{
		"name":  "inhere",
		"age":   100,
		"oldSt": 1,
		"newSt": 2,
		"email": "some@email.com",
		"tags":  []string{"go", "php", "java"},
	}

	v := validate.Map(m)
	// v := validate.New(m)
	v.AddRule("name", "required")
	v.AddRule("name", "minLen", 7)
	v.AddRule("age", "max", 99)
	v.AddRule("age", "min", 1)
	v.AddRule("email", "email")

	// 也可以这样
	v.StringRule("age", "required|int|min:1|max:99")
	v.StringRule("name", "required|minLen:7")
	v.StringRule("tags", "required|slice|minlen:1")

	// 功能：支持检查切片中的子项
	v.StringRule("tags.*", "required|string|min_len:7")

	// v.WithScenes(map[string]string{
	//   "create": []string{"name", "email"},
	//   "update": []string{"name"},
	// })

	r := v.ValidateR() // 返回 *ValidResult，与 v 解耦
	if r.IsOK() {      // 验证通过
		safeData := r.SafeData()
		// 执行某些操作 ...
	} else {
		fmt.Println(r.Errors)       // 所有错误消息
		fmt.Println(r.Errors.One()) // 返回一条随机错误消息文本
	}
}
```

> 提示：对于结构体验证，推荐使用顶层的 `validate.Check(&u)` ——它是无状态的，内部使用了对象池，并返回相同的 `*ValidResult`。

如果是 HTTP 请求，可以快速验证数据并通过验证。然后将安全数据绑定到结构体。

```go
package main

import (
	"fmt"
	"net/http"
	"time"

	"github.com/gookit/validate/v2"
)

// UserForm 结构体
type UserForm struct {
	Name     string
	Email    string
	Age      int
	CreateAt int
	Safe     int
	UpdateAt time.Time
	Code     string
}

func main() {
	handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		data, err := validate.FromRequest(r)
		if err != nil {
			panic(err)
		}

		v := data.Create()
		// 设置规则
		v.FilterRule("age", "int") // 将值转换为 int
		v.AddRule("name", "required")
		v.AddRule("name", "minLen", 7)
		v.AddRule("age", "max", 99)
		v.StringRule("code", `required|regex:\d{4,6}`)

		vr := v.ValidateR()
		if vr.IsOK() { //
```
