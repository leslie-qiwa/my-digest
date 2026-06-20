# goja: An ECMAScript/JavaScript Engine in Pure Go

Go 语言实现的 ECMAScript 5.1(+)。

Goja 是一个用纯 Go 语言实现的 ECMAScript 5.1，注重标准合规性和性能。

该项目在很大程度上受到了 otto 的启发。

最低要求的 Go 版本为 1.25。

- 完整的 ECMAScript 5.1 支持（包括正则表达式和严格模式）。
- 几乎通过了目前已实现功能的所有 tc39 测试。目标是通过全部测试。最新可用的 commit id 请参见 .tc39_test262_checkout.sh。
- 能够运行 Babel、TypeScript 编译器以及几乎所有用 ES5 编写的代码。
- 支持 Sourcemaps。
- 大部分 ES6 功能，仍在开发中，详见 https://github.com/dop251/goja/milestone/1?closed=1

WeakMap 的实现方式是将值的引用嵌入到键中。这意味着只要键是可达的，与其在任何弱映射中关联的所有值也将保持可达状态，因此即使没有其他引用，甚至在 WeakMap 本身被销毁后，这些值也无法被垃圾回收。值的引用只有在键被显式地从 WeakMap 中删除，或者键变得不可达时才会被释放。

举例说明：

var m = new WeakMap();
var key = {};
var value = {/* 一个非常大的对象 */};
m.set(key, value);
value = undefined;
m = undefined; // 此时值不会变为可垃圾回收状态
key = undefined; // 现在可以了
// m.delete(key); // 这种方式也可以

原因在于 Go 运行时的限制。在撰写本文时（版本 1.15），如果一个对象设置了终结器（finalizer）并且该对象属于引用循环的一部分，则整个循环都将无法被垃圾回收。上述方案是我能想到的在不涉及终结器的情况下唯一合理的方式。这是第三次尝试（更多细节请参见 #250 和 #199）。

请注意，这不会影响应用逻辑，但可能导致内存使用量高于预期。

基于上述原因，在当前阶段实现 WeakRef 和 FinalizationRegistry 似乎是不可能的。

JSON.parse() 使用标准 Go 库，该库以 UTF-8 方式运行。因此，它无法正确解析损坏的 UTF-16 代理对，例如：

JSON.parse(`"\\uD800"`).charCodeAt(0).toString(16) // 返回 "fffd" 而不是 "d800"

从日历日期到纪元时间戳的转换使用标准 Go 库，该库使用 int 而非 ECMAScript 规范所规定的 float。这意味着如果你向 Date() 构造函数传递了溢出 int 的参数，或者发生了整数溢出，结果将不正确，例如：

Date.UTC(1970, 0, 1, 80063993375, 29, 1, -288230376151711740) // 返回 29256 而不是 29312

尽管它比我所见过的许多 Go 语言中的脚本语言实现都要快（例如，它平均比 otto 快 6-7 倍），但它不能替代 V8、SpiderMonkey 或任何其他通用 JavaScript 引擎。你可以在这里找到一些基准测试。

这很大程度上取决于你的使用场景。如果大部分工作是在 JavaScript 中完成的（例如加密或任何其他大量计算），那么使用 V8 绝对是更好的选择。

如果你需要一种脚本语言来驱动用 Go 编写的引擎，并且需要在 Go 和 JavaScript 之间频繁调用并传递复杂数据结构，那么 cgo 的开销可能会抵消拥有更快 JavaScript 引擎所带来的好处。

因为它是用纯 Go 编写的，没有 cgo 依赖，构建非常简单，并且可以在 Go 支持的任何平台上运行。

它让你能更好地控制执行环境，因此可用于研究目的。

不能。一个 goja.Runtime 实例在同一时间只能被单个 goroutine 使用。你可以创建任意数量的 Runtime 实例，但无法在运行时之间传递对象值。

setTimeout() 和 setInterval() 是 ECMAScript 环境中提供并发执行的常见函数，但这两个函数并不属于 ECMAScript 标准。浏览器和 NodeJS 只是恰好提供了类似但不完全相同的函数。宿主应用程序需要控制并发执行的环境（例如事件循环），并向脚本代码提供相应功能。

有一个独立的项目旨在提供一些 NodeJS 功能，其中包含一个事件循环。

我会按照依赖顺序并在时间允许的情况下尽快添加功能。请不要询问预计完成时间。里程碑中处于开放状态的功能要么正在进行中，要么将是下一个要处理的。

正在进行的工作在独立的功能分支中完成，在适当时合并到 master。这些分支中的每次提交都代表一个相对稳定的状态（即可以编译并通过所有已启用的 tc39 测试），但由于我使用的 tc39 测试版本较旧，它可能没有 ES5.1 功能那样经过充分测试。因为 ECMAScript 修订版之间（通常）没有重大破坏性更改，所以它不应该破坏你现有的代码。欢迎你尝试并报告发现的任何错误。但是，请在提交修复之前先进行讨论，因为代码可能在此期间已经发生了变化。

在提交拉取请求之前，请确保：

- 你尽可能地遵循了 ECMA 标准。如果要添加新功能，请确保你已经阅读了规范，不要仅仅基于几个运行正常的示例。
- 你的更改不会对性能产生显著的负面影响（除非是修复错误且无法避免）。
- 它通过了所有相关的 tc39 测试。
- API 不应有破坏性更改，但可以进行扩展。
- 部分 AnnexB 功能尚未实现。

运行 JavaScript 并获取结果值。

vm := goja.New()
v, err := vm.RunString("2 + 2")
if err != nil {
panic(err)
}
if num := v.Export().(int64); num != 4 {
panic(num)
}

任何 Go 值都可以使用 Runtime.ToValue() 方法传递给 JS。更多细节请参阅该方法的文档。

JS 值可以使用 Value.Export() 方法导出为其默认的 Go 表示形式。

或者，可以使用 Runtime.ExportTo() 方法将其导出到特定的 Go 变量中。

在单次导出操作中，同一个 Object 将由相同的 Go 值表示（相同的 map、slice 或指向相同结构体的指针）。这包括循环对象，使得导出循环对象成为可能。

有两种方式：

- 使用 AssertFunction()：

const SCRIPT = `
function sum(a, b) {
return +a + b;
}
`

vm := goja.New()
_, err := vm.RunString(SCRIPT)
if err != nil {
panic(err)
}
sum, ok := goja.AssertFunction(vm.Get("sum"))
if !ok {
panic("Not a function")
}

res, err := sum(goja.Undefined(), vm.ToValue(40), vm.ToValue(2))
if err != nil {
panic(err)
}
fmt.Println(res)
// 输出: 42

- 使用 Runtime.ExportTo()：

const SCRIPT = `
function sum(a, b) {
return +a + b;
}
`

vm := goja.New()
_, err := vm.RunString(SCRIPT)
if err != nil {
panic(err)
}

var sum func(int, int) int
err = vm.ExportTo(vm.Get("sum"), &sum)
if err != nil {
panic(err)
}

fmt.Println(sum(40, 2)) // 注意，函数中的 _this_ 值将是 undefined。
// 输出: 42

第一种方式更底层，允许指定 this 值，而第二种方式使函数看起来像一个普通的 Go 函数。

默认情况下，名称按原样传递，这意味着它们是大写的。这不符合标准的 JavaScript 命名约定，所以如果你需要让 JS 代码看起来更自然，或者你在处理第三方库，可以使用 FieldNameMapper：

vm := goja.New()
vm.SetFieldNameMapper(TagFieldNameMapper("json", true))
type S struct {
Field int `json:"field"`
}
vm.Set("s", S{Field: 42})
res, _ := vm.RunString(`s.field`) // 不使用映射器的话应该是 s.Field
fmt.Println(res.Export())
// 输出: 42

有两个标准映射器：TagFieldNameMapper 和 UncapFieldNameMapper，你也可以使用自己的实现。

要在 Go 中实现构造函数，请使用 `func (goja.ConstructorCall) *goja.Object`。

请参阅 Runtime.ToValue() 文档了
