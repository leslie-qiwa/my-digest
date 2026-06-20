# Figurine: A Go App to Print Your Name in Style

以风格化方式打印你的名字

你可以从这里下载最新的二进制文件，也可以从源码编译：

go install github.com/arsham/figurine/v2@latest

每次调用该应用时，它会随机选择一种字体来渲染消息。将你想要装饰的消息作为参数传入。

figurine Some Text

你可以查看可用的字体：

figurine -l
figurine -l -s
figurine -ls Sample Text

设置字体：

figurine -f "Poison.flf" Some Text

获取可用参数列表：

figurine -h

这个应用非常轻量，你可以将它添加到你的 .zshrc/.bashrc 文件中，这样每次打开新的终端时都会显示一条精美的消息。

另请参阅 Rainbow，这是用于为输出添加颜色的库。

本源代码的使用受 Apache 2.0 许可证约束。许可证可在 LICENSE 文件中找到。
