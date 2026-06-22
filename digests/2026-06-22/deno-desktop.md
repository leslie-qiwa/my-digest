# Deno Desktop

# 桌面应用

`deno desktop` 可将 Deno 项目（从单个 TypeScript 文件到 Next.js 应用均可）转化为自包含的桌面应用程序。输出为可分发的二进制文件，将你的代码、Deno 运行时和 Web 渲染引擎打包在一起，每个平台生成一个包。

`deno desktop` 随 Deno v2.9.0 发布，目前尚未进入稳定版本。如需立即体验，请运行 `deno upgrade canary` 安装 canary 构建版本。命令、配置项和 TypeScript API 在功能稳定之前可能仍会变更。

## 为什么选择 deno desktop

Web 技术是世界上最广泛使用的 UI 工具包。基于 Web 技术栈构建的桌面应用（Electron、Tauri、Electrobun）利用了这一优势，但每种方案都有你必须接受的取舍：巨大的二进制体积、缺失的平台支持、无 JavaScript 生态系统、无内置更新方案、无框架集成。

`deno desktop` 对这些取舍有明确的立场：

- **默认体积小，完整 Node 兼容性。** 默认的 WebView 后端使用操作系统自带的 webview 以生成小体积二进制文件，同时你仍可通过 Deno 的 Node 兼容层使用整个 npm 生态系统。当你需要在 macOS、Windows 和 Linux 上实现一致的渲染效果时，可选择捆绑 Chromium（CEF）后端。
- **框架自动检测。** 将 `deno desktop` 指向 Next.js、Astro、Fresh、Remix、Nuxt、SvelteKit、SolidStart、TanStack Start 或 Vite SSR 项目即可运行：发布模式下运行生产服务器，`--hmr` 模式下运行带热重载的开发服务器。无需修改任何代码即可将现有 Web 项目搬上桌面。
- **进程内绑定而非 IPC。** 后端与 UI 之间的通信通过进程内通道进行，而非基于套接字的 IPC。值在跨越调用边界时仍会被编码，但 Deno 代码与 webview 之间不存在跨进程往返。
- **单机交叉编译。** 同一台机器可以为 macOS、Windows 和 Linux 构建。后端按需下载，无需本地构建。
- **内置二进制差分自动更新。** 只需发布一个 `latest.json` 清单文件和 bsdiff 补丁；运行时自动轮询、应用更新，并在启动失败时自动回滚。

## Hello, desktop

创建一个单文件桌面应用：

```js
Deno.serve(() =>
  new Response("<h1>Hello, desktop</h1>", {
    headers: { "content-type": "text/html" },
  })
);
```

```bash
deno desktop main.ts
```

编译后的二进制文件会打开一个窗口，指向绑定到你的 `Deno.serve()` 处理函数的本地 HTTP 服务器。直接运行：

```bash
./main        # macOS / Linux
.\main.exe    # Windows
```

`Deno.serve()` 会自动绑定到 webview 导航的地址，因此你无需传递端口或主机名。详见 HTTP 服务部分。

## 本节内容

- **配置：** `deno.json` 中的 `desktop` 配置块。
- **后端：** CEF、webview、raw；如何选择。
- **HTTP 服务：** `Deno.serve()` 集成与服务模型。
- **框架：** Next.js、Astro、Fresh、Remix、Nuxt、SvelteKit 等。
- **窗口：** `Deno.BrowserWindow` 生命周期、多窗口、事件。
- **绑定：** 通过 `bindings.<name>()` 从 webview 调用 Deno 代码。
- **菜单：** 应用菜单和上下文菜单。
- **托盘和 Dock：** 系统状态图标和 macOS Dock。
- **对话框：** `prompt()`、`alert()`、`confirm()` 作为原生弹窗。
- **通知：** 通过 Web `Notification` API 发送原生操作系统通知。
- **热模块替换：** `--hmr` 用于框架和非框架应用。
- **开发者工具：** 同时附加到 Deno 运行时和 webview 的统一 DevTools。
- **自动更新：** `Deno.autoUpdate()`、清单文件、bsdiff、回滚。
- **错误报告：** 捕获未处理的异常和 panic。
- **分发：** 交叉编译、输出格式、安装器。
- **对比：** `deno desktop` 与 Electron、Tauri、Electrobun、Dioxus 的关系。
- **`deno desktop` CLI 参考：** 命令、标志和 `deno.json` `desktop` 配置模式。
