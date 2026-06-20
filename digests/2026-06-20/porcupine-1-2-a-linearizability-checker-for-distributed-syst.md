# Porcupine 1.2: A Linearizability Checker for Distributed Systems

Porcupine 是一个快速的线性一致性检查器，在学术界和工业界均被广泛用于测试分布式系统的正确性。它接受以可执行 Go 代码编写的顺序规约以及一段并发历史记录，然后判断该历史记录相对于顺序规约是否满足线性一致性。Porcupine 还实现了历史记录和线性化点的可视化工具。
（点击查看交互版本）
Porcupine 实现了论文《Faster linearizability checking via P-compositionality》中描述的算法，该算法是对论文《Testing for Linearizability》中描述算法的优化。
Porcupine 比 Knossos 的线性一致性检查器更快，并且能处理更多的历史记录。在 `test_data/jepsen/` 中的数据上进行测试，Porcupine 通常快 1,000 到 10,000 倍，且内存占用小得多。在能够利用 P-可组合性的历史记录上，Porcupine 可以快数百万倍。
Porcupine 接受一个系统的可执行模型和一段历史记录，然后运行判定过程来确定该历史记录相对于模型是否满足线性一致性。Porcupine 支持两种方式指定历史记录：一种是以带有给定调用和返回时间的操作列表形式，另一种是按时间顺序排列的调用/返回事件列表形式。Porcupine 还可以可视化历史记录以及部分线性化结果，这有助于调试。
请参阅文档了解如何编写模型和指定历史记录。你也可以查看测试中的一些模型实现示例。
编写好模型并获得历史记录后，你可以使用 `CheckOperations` 和 `CheckEvents` 函数来判断你的历史记录是否满足线性一致性。如果你想可视化历史记录及其部分线性化结果，可以使用 `Visualize` 函数。
假设我们要测试一个初始值为 `0` 的读写寄存器上操作的线性一致性。我们为该寄存器编写如下顺序规约：
```go
type registerInput struct {
	op    bool // false = put, true = get
	value int
}

// 寄存器的顺序规约
registerModel := porcupine.Model{
	Init: func() interface{} {
		return 0
	},
	// 步进函数：接受一个状态、输入和输出，返回该操作是否合法以及新状态
	Step: func(state, input, output interface{}) (bool, interface{}) {
		regInput := input.(registerInput)
		if regInput.op == false {
			return true, regInput.value // put 操作总是合法的
		} else {
			readCorrectValue := output == state
			return readCorrectValue, state // 状态不变
		}
	},
}
```
假设我们有以下来自 3 个客户端的并发历史记录。在每一行中，第一个 `|` 表示操作被调用的时间，第二个 `|` 表示操作返回的时间。
```
C0:  |-------- put('100') --------|
C1:      |--- get() -> '100' ---|
C2:          |- get() -> '0' -|
```
我们将这段历史记录编码如下：
```go
events := []porcupine.Event{
	// C0: put('100')
	{Kind: porcupine.CallEvent, Value: registerInput{false, 100}, Id: 0, ClientId: 0},
	// C1: get()
	{Kind: porcupine.CallEvent, Value: registerInput{true, 0}, Id: 1, ClientId: 1},
	// C2: get()
	{Kind: porcupine.CallEvent, Value: registerInput{true, 0}, Id: 2, ClientId: 2},
	// C2: 完成 get() -> '0'
	{Kind: porcupine.ReturnEvent, Value: 0, Id: 2, ClientId: 2},
	// C1: 完成 get() -> '100'
	{Kind: porcupine.ReturnEvent, Value: 100, Id: 1, ClientId: 1},
	// C0: 完成 put('100')
	{Kind: porcupine.ReturnEvent, Value: 0, Id: 0, ClientId: 0},
}
```
我们可以让 Porcupine 检查该历史记录的线性一致性：
```go
ok := porcupine.CheckEvents(registerModel, events)
// 返回 true
```
Porcupine 还可以可视化线性化点：
现在，假设我们有另一段历史记录：
```
C0:  |---------------- put('200') ----------------|
C1:      |- get() -> '200' -|
C2:                             |- get() -> '0' -|
```
我们可以用 Porcupine 检查该历史记录，发现它不满足线性一致性：
```go
events := []porcupine.Event{
	// C0: put('200')
	{Kind: porcupine.CallEvent, Value: registerInput{false, 200}, Id: 0, ClientId: 0},
	// C1: get()
	{Kind: porcupine.CallEvent, Value: registerInput{true, 0}, Id: 1, ClientId: 1},
	// C1: 完成 get() -> '200'
	{Kind: porcupine.ReturnEvent, Value: 200, Id: 1, ClientId: 1},
	// C2: get()
	{Kind: porcupine.CallEvent, Value: registerInput{true, 0}, Id: 2, ClientId: 2},
	// C2: 完成 get() -> '0'
	{Kind: porcupine.ReturnEvent, Value: 0, Id: 2, ClientId: 2},
	// C0: 完成 put('200')
	{Kind: porcupine.ReturnEvent, Value: 0, Id: 0, ClientId: 0},
}

ok := porcupine.CheckEvents(registerModel, events)
// 返回 false
```
更多关于如何编写模型和历史记录的示例，请参阅 `porcupine_test.go`。
Porcupine 提供了可视化历史记录的功能，包括线性化结果（或者在非线性一致历史记录的情况下，显示部分线性化和非法线性化点）。输出结果是一个使用 JavaScript 绘制交互式可视化的 HTML 页面。输出效果如下：
你可以在此处查看完整的交互版本。
可视化按分区展示：所有分区本质上是独立的，因此在上面的键值存储示例中，与每个唯一键相关的操作位于独立的分区中。
在静态视图中，可视化展示所有历史元素，以及每个分区的线性化点。如果某个分区没有完整的线性化，可视化会显示最长的部分线性化。它还会为每个历史元素显示包含该事件的最长部分线性化，即使它不是整体最长的部分线性化；这些默认以灰色显示。它还会显示非法线性化点，即那些被检查是否可以作为下一个发生但根据模型在该点进行线性化是非法的历史元素。
当鼠标悬停在某个历史元素上时，可视化会高亮最相关的部分线性化。如果存在包含该事件的最长部分线性化，则显示该线性化。否则，如果存在以当前悬停事件作为非法线性化点结尾的最长部分线性化，则显示该线性化。悬停在事件上还会显示一个工具提示，展示额外信息，如状态机的前一个和当前状态，以及操作的调用时间和返回时间。这些信息来源于当前选中的线性化。
点击某个历史元素会选中它，并用粗边框高亮该事件。这使得部分线性化的选择变为"固定"状态，因此可以在历史记录中移动而不会取消选择。点击另一个历史元素会改为选中该元素，点击背景则会取消选择。
可视化历史记录只需使用 `CheckOperationsVerbose` / `CheckEventsVerbose` 函数（它们返回可视化所需的额外信息）和 `Visualize` 函数（生成可视化结果）。为了获得良好的可视化效果，建议填写模型的 `DescribeOperation` 和 `DescribeState` 字段。请参阅 `visualization_test.go` 了解如何使用 Porcupine 可视化历史记录的完整示例。
你还可以向可视化添加自定义注解，如上面的示例所示。这对于附加调试信息（例如来自服务器或测试框架的信息）非常有帮助。你可以使用 `AddAnnotations` 方法来实现这一点。
- 如果 Porcupine 在你的模型/历史记录上运行非常缓慢，这可能是不可避免的，原因是状态空间爆炸。请参阅此 issue 了解在特定模型和历史记录场景下这一挑战的讨论。
- 在记录操作的时间戳时，特别是在 ARM 和其他弱内存序架构上，你可能需要使用内存屏障或原子操作来确保测量的准确性，避免虚假的线性一致性违规。请参阅此 issue 了解详情。
Porcupine 在学术界和工业界均有广泛应用。查看 Porcupine 的现有使用案例有助于了解如何设计
