# Heimdall 7.2: An Enhanced HTTP Client for Go

Heimdall 是一个 HTTP 客户端，帮助您的应用程序大规模发起大量请求。使用 Heimdall，您可以：
- 使用类似 hystrix 的熔断器来控制失败的请求
- 为每个请求添加同步的内存重试机制，并可选择设置自定义的重试策略
- 为每个请求创建具有不同超时时间的客户端
所有 HTTP 方法都以流式接口的方式暴露。
go get -u github.com/gojek/heimdall/v7
您可以通过在 .go 文件中添加以下导入语句来使用此包。
import "github.com/gojek/heimdall/v7/httpclient"
以下示例将打印 Google 首页的内容：
// 创建一个带有默认超时时间的新 HTTP 客户端
timeout := 1000 * time.Millisecond
client := httpclient.NewClient(httpclient.WithHTTPTimeout(timeout))
// 使用客户端的 GET 方法创建并执行请求
res, err := client.Get("http://google.com", nil)
if err != nil{
panic(err)
}
// Heimdall 返回标准的 *http.Response 对象
body, err := io.ReadAll(res.Body)
fmt.Println(string(body))
您也可以将 *http.Request 对象与 http.Do 接口配合使用：
timeout := 1000 * time.Millisecond
client := httpclient.NewClient(httpclient.WithHTTPTimeout(timeout))
// 创建一个 http.Request 实例
req, _ := http.NewRequest(http.MethodGet, "http://google.com", nil)
// 调用 `Do` 方法，其接口与 `http.Do` 方法类似
res, err := client.Do(req)
if err != nil {
panic(err)
}
body, err := io.ReadAll(res.Body)
fmt.Println(string(body))
要导入 heimdall 的 hystrix 包：
import "github.com/gojek/heimdall/v7/hystrix"
您可以使用 hystrix.NewClient 函数创建一个包裹在类 hystrix 熔断器中的客户端：
// 创建一个新的 hystrix 包裹的 HTTP 客户端，指定命令名称及其他必需选项
client := hystrix.NewClient(
hystrix.WithHTTPTimeout(10 * time.Millisecond),
hystrix.WithCommandName("google_get_request"),
hystrix.WithHystrixTimeout(1000 * time.Millisecond),
hystrix.WithMaxConcurrentRequests(30),
hystrix.WithErrorPercentThreshold(20),
hystrix.WithStatsDCollector("localhost:8125", "myapp.hystrix"),
)
// 其余部分与前面的示例相同
在上面的示例中，使用了两个超时值：一个用于 hystrix 配置，另一个用于 HTTP 客户端配置。前者决定 hystrix 应在何时记录错误，后者决定客户端本身应在何时返回超时错误。除非您有特殊需求，否则这两个值通常应设置为相同。
您可以在初始化客户端时使用 hystrix.WithStatsDCollector(<statsd 地址>, <指标前缀>) 选项，将 hystrix 指标导出到 statsD 收集器，如上所示。
您可以使用 hystrix.NewClient 函数，通过传入自定义的降级回调来创建一个包裹在类 hystrix 熔断器中的客户端：
当您的代码返回错误，或由于各种健康检查而无法完成时，降级函数将被触发。
您的降级函数应如下所示，您需要传入一个签名如下的函数：
func(err error) error {
// 处理错误/故障条件的逻辑
return err
}
示例
// 创建一个新的降级函数
fallbackFn := func(err error) error {
_, postErr := http.Post("http://post_to_channel_two")
return postErr
}
timeout := 10 * time.Millisecond
// 创建一个新的 hystrix 包裹的 HTTP 客户端，使用 fallbackFunc 作为降级函数
client := hystrix.NewClient(
hystrix.WithHTTPTimeout(timeout),
hystrix.WithCommandName("MyCommand"),
hystrix.WithHystrixTimeout(1100 * time.Millisecond),
hystrix.WithMaxConcurrentRequests(100),
hystrix.WithErrorPercentThreshold(20),
hystrix.WithSleepWindow(10),
hystrix.WithRequestVolumeThreshold(10),
hystrix.WithFallbackFunc(fallbackFn),
)
// 其余部分与前面的示例相同
在上面的示例中，fallbackFunc 是一个函数，当向通道一发送失败时，它会向通道二发送请求。
// 首先设置退避机制。常量退避以固定速率增加退避时间
backoffInterval := 500 * time.Millisecond
// 定义最大抖动间隔，必须大于 1*time.Millisecond
maximumJitterInterval := 5 * time.Millisecond
backoff := heimdall.NewConstantBackoff(backoffInterval, maximumJitterInterval)
// 使用该退避策略创建新的重试机制
retrier := heimdall.NewRetrier(backoff)
timeout := 1000 * time.Millisecond
// 创建新客户端，设置重试机制和重试次数
client := httpclient.NewClient(
httpclient.WithHTTPTimeout(timeout),
httpclient.WithRetrier(retrier),
httpclient.WithRetryCount(4),
)
// 其余部分与第一个示例相同
或者创建一个使用指数退避的客户端
// 首先设置退避机制。指数退避以指数速率增加退避时间
initialTimeout := 2 * time.Millisecond // 初始超时时间
maxTimeout := 9 * time.Millisecond // 最大超时时间
exponentFactor := float64(2) // 倍增因子
maximumJitterInterval := 2 * time.Millisecond // 最大抖动间隔，必须大于 1 * time.Millisecond
backoff := heimdall.NewExponentialBackoff(initialTimeout, maxTimeout, exponentFactor, maximumJitterInterval)
// 使用该退避策略创建新的重试机制
retrier := heimdall.NewRetrier(backoff)
timeout := 1000 * time.Millisecond
// 创建新客户端，设置重试机制和重试次数
client := httpclient.NewClient(
httpclient.WithHTTPTimeout(timeout),
httpclient.WithRetrier(retrier),
httpclient.WithRetryCount(4),
)
// 其余部分与第一个示例相同
这将创建一个 HTTP 客户端，在请求失败时每隔 500 毫秒进行重试。该库还自带指数退避功能。
Heimdall 支持自定义重试策略。为此，您需要实现 Backoff 接口：
type Backoff interface {
Next(retry int) time.Duration
}
让我们看一个创建线性递增退避时间客户端的示例：
首先，创建退避机制：
type linearBackoff struct {
backoffInterval int
}
func (lb *linearBackoff) Next(retry int) time.Duration{
if retry <= 0 {
return 0 * time.Millisecond
}
return time.Duration(retry * lb.backoffInterval) * time.Millisecond
}
这将创建一个退避机制，每次重试尝试时重试时间线性增加。我们可以像上一个示例一样使用它来创建客户端：
backoff := &linearBackoff{100}
retrier := heimdall.NewRetrier(backoff)
timeout := 1000 * time.Millisecond
// 创建新客户端，设置重试机制和重试次数
client := httpclient.NewClient(
httpclient.WithHTTPTimeout(timeout),
httpclient.WithRetrier(retrier),
httpclient.WithRetryCount(4),
)
// 其余部分与第一个示例相同
Heimdall 还允许您简单地传入一个返回重试超时时间的函数。可以这样使用它来创建客户端：
linearRetrier := heimdall.NewRetrierFunc(func(retry int) time.Duration {
if retry <= 0 {
return 0 * time.Millisecond
}
return time.Duration(retry) * time.Millisecond
})
timeout := 1000 * time.Millisecond
client := httpclient.NewClient(
httpclient.WithHTTPTimeout(timeout),
httpclient.WithRetrier(linearRetrier),
httpclient.WithRetryCount(4),
)
Heimdall 支持自定义 HTTP 客户端。如果您使用的是从其他库导入的客户端，和/或希望为每个请求实现自定义日志记录、Cookie、请求头等功能，这将非常有用。
在底层，Client 结构体现在接受 Doer，这是 HTTP 客户端（包括标准库的 net/*http.Client）实现的标准接口。
假设我们希望为所有请求添加授权请求头。
我们可以定义我们的客户端 myHTTPClient
type myHTTPClient struct {
client *http.Client
}
func (c *myHTTPClient) Do(request *http.Request) (*http.Response, error) {
request.SetBasicAuth("username",
