# clipboard 0.8: The Clipboard Library Goes Cgo-Free on Desktop

v0.8.0 — 社区里程碑 🎉

这是项目历史上规模最大的一次发布——它的诞生离不开你们每一个人。下面的每一项功能都源自社区 issue、原型设计、设计讨论或 bug 报告。感谢所有提交报告、提出建议、开发原型、参与评审并推动这些功能落地的人——你们的参与直接塑造了这个版本。🙏

请注意——有一项破坏性变更：

`Watch` 现在返回 `<-chan Data` 而非 `<-chan []byte`。迁移只需一行：读取 `data.Bytes`（同时你现在可以检查 `data.Format`）。

✨ 亮点

- **所有桌面平台均无需 Cgo。** macOS（purego/Obj-C）、Linux/X11 以及各 BSD 系统现在无需 C 工具链即可在构建时与操作系统剪贴板通信，运行时也不依赖 `libX11`/`libwayland`（Windows 此前已无需 Cgo）。交叉编译到桌面平台"开箱即用"。(#69, #25, #55)
- **原生 Wayland 支持。** 纯 Go 实现的 data-control 后端（`ext-data-control-v1`/`zwlr_data_control_manager_v1`），事件驱动的 `Watch`，可用时自动优先于 X11 选用，并支持 XWayland 回退。(#6)
- **自定义 MIME 类型格式。** `Register(mime) Format` + `ReadAs[T]` 提供对任意 MIME 类型的原始、透传读写/监听——无转换、无新的强制依赖、无平台细节泄漏到你的代码中。(#17, 涵盖 #40)
- **剪贴板枚举。** `Formats() []Format` 报告当前剪贴板上的内容（自动注册它发现的 MIME 类型），`Format.MIME()` 告诉你某个标记对应什么类型。(#89)
- **可变参数、格式标记的 `Watch`。** `Watch(ctx, ...Format) <-chan Data` 可同时监听多种格式，并为每次变更标记其格式（无参数时监听所有内置格式）。(#89)
- **灵活的图像输入。** `Write(FmtImage, …)` 现在接受 JPEG/GIF/WebP/… 格式（需空白导入对应解码器），并统一转换为标准 PNG；`Read(FmtImage)` 始终返回 PNG。(#155)

🐛 修复

- Windows：透明图像不再粘贴后偏暗——图像以非预乘 alpha（straight alpha）方式存储/读取，以匹配 `CF_DIBV5`。已在 CI 中通过真实 Windows 消费者进行端到端验证。(#105)
- Windows：24 位（及其他非 32 位）DIB 图像现在能正常解码，不再返回 nil。(#65)
- Windows：`OpenClipboard` 不再无限忙等——当其他应用持有剪贴板时，采用有界退避 + 超时机制。(#144)
- `Watch` 不再泄漏——定时器已正确停止，发送操作遵循 context 取消。(#153)

📦 依赖

- 新增第一方纯 Go X11 编解码库 `golang.design/x/x11` v0.2.0（从本仓库中提取；可在其他项目中复用）。
- iOS/Android 继续使用 `gomobile`（Cgo）；移动端保持仅支持文本。

🙏 致谢

这个版本凝聚了众多贡献者的心血。特别感谢：

- Wayland：@trading-peter (#6)、@ajayd-san (#61)
- 移除 Cgo：@TotallyGamerJet（purego 原型，#83）、@gen2brain (#25)、@sarumaj (#69)、@arturbrzoz (#55)、@microo8 (#20)
- 自定义格式与原始访问：@dmzlingyin、@HuakunShen、@MarvinJWendt (#17)、@rams3sh (#40)
- 全格式监听与枚举：@xiebruce (#89)
- Windows 图像 / 图像格式：@allanpk716 (#65)、@matwachich (#48)、@AlejandroSuero (#58)、@jmmemo (#38)、@c17abab (#45)
- Bug 报告与功能建议（塑造了项目路线图）：@fanybook (#59)、@dongziyudongziyu (#37)、@Esword618 (#50)、@jinmao88 (#31)、@gucio321 (#67)、@ValorZard (#64)、@qjerome (#22)

……以及所有参与讨论、测试预发布版本、提交清晰可操作 issue 的每一个人。感谢你们。❤️

迁移

```
// 之前 (v0.7.x)：
for data := range clipboard.Watch(ctx, clipboard.FmtText) {
    use(data) // data 是 []byte
}

// 之后 (v0.8.0)：
for data := range clipboard.Watch(ctx, clipboard.FmtText) {
    use(data.Bytes) // data 是 clipboard.Data{Format, Bytes}
}
```

完整变更日志：v0.7.1...v0.8.0
