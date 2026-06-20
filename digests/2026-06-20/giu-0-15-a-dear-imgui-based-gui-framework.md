# giu 0.15: A Dear ImGui-Based GUI Framework

简介
大家好！
距离上次 giu 发布已经过去一年多了，我们带来了一系列更新和修复。
发布亮点
- 移除了以下枚举：
StyleColorNavHighlight
- 新增了通过
`(*MasterWindow).SetUserFile(...)` 指定 `.ini` 文件名的功能
- `MasterWindow`：可通过 `GetStyle` 获取当前全局样式
- 修复了多个字体缩放问题（macOS）
- `InputText`：新增 `InputTextFlagsWordWrap`，允许自动换行。
- `FontAtlas` 重构：
  - 字体大小不再由 `FontAtlas` 管理，因此 `(*FontAtlas).SetSize` 已被移除
  - 不再需要注册字符串，因此 `(*FontAtlas).RegisterString`（及其变体）已被移除。
- `Markdown`：修改了 `(*MarkdownWidget).Header`，使其可以设置字体大小。
- `MasterWindow`：新增浅色主题
- `CodeEditor`：新增不同的配色方案
- `MasterWindow`：新增 `SizeChangeCallback`，在主窗口大小调整时调用。
- `ContextMenu`：`MouseButton` 已重命名为 `Flags`。同时，标志类型从 `MouseButton` 变更为 `PopupFlags`
注意
本次发布使用 cimgui-go `v1.5.0` 版本，该版本使用 imgui `1.92.8`。
重要
请注意，giu 必须与正确的 cimgui-go 版本配对使用。混合使用不同版本可能会导致编译问题。
变更内容
- 将用户 .ini 文件设为可选 由 @theaino 提交于 #989
- readme：更新安装说明 由 @gucio321 提交于 #994
- 移除 .travis.yml 由 @gucio321 提交于 #988
- 移除 StyleColorNavHighlight 由 @theaino 提交于 #995
- 调整弹出窗口打开时间 | #986 由 @theaino 提交于 #990
- StyleSetter 的 getter 方法及 MasterWindow 的 GetStyle() 由 @theaino 提交于 #998
- github/issue_template：改进 bug 报告模板 由 @gucio321 提交于 #999
- build(deps)：将 golang.org/x/image 从 0.27.0 升级到 0.28.0 由 @dependabot[bot] 提交于 #1002
- build(deps)：将 golang.org/x/image 从 0.28.0 升级到 0.29.0 由 @dependabot[bot] 提交于 #1005
- build(deps)：将 golang.org/x/image 从 0.29.0 升级到 0.30.0 由 @dependabot[bot] 提交于 #1014
- 在 macOS 上不缩放字体大小 由 @ioxenus 提交于 #1015
- 添加额外功能键（F13-F25）由 @ioxenus 提交于 #1017
- build(deps)：将 actions/checkout 从 4 升级到 5 由 @dependabot[bot] 提交于 #1016
- build(deps)：将 github.com/stretchr/testify 从 1.10.0 升级到 1.11.1 由 @dependabot[bot] 提交于 #1020
- workflows：在 setup-go 调用中引用 go.mod 由 @gucio321 提交于 #1027
- build(deps)：将 golang.org/x/image 从 0.30.0 升级到 0.31.0 由 @dependabot[bot] 提交于 #1024
- build(deps)：将 actions/setup-go 从 5 升级到 6 由 @dependabot[bot] 提交于 #1021
- 将 cimgui-go 更新到 1.4.0 由 @gucio321 提交于 #1028
- Flags：添加 InputTextFlagsWordWrap 标志 由 @gucio321 提交于 #1031
- 破坏性变更：Font Atlas 大小处理重构 由 @gucio321 提交于 #1032
- 破坏性变更：markdown：修复 Header 无法设置字体大小的问题 由 @gucio321 提交于 #1034
- 将 golangci-lint 更新到 v2.5.0 由 @gucio321 提交于 #1033
- Font atlas：移除字形范围 由 @gucio321 提交于 #1036
- themes：添加设置浅色主题的功能 由 @gucio321 提交于 #1037
- build(deps)：将 golang.org/x/image 从 0.31.0 升级到 0.32.0 由 @dependabot[bot] 提交于 #1039
- 代码编辑器：添加不同配色方案支持 由 @gucio321 提交于 #1038
- build(deps)：将 golangci/golangci-lint-action 从 8 升级到 9 由 @dependabot[bot] 提交于 #1042
- build(deps)：将 golang.org/x/image 从 0.32.0 升级到 0.33.0 由 @dependabot[bot] 提交于 #1043
- build(deps)：将 actions/checkout 从 5 升级到 6 由 @dependabot[bot] 提交于 #1044
- build(deps)：将 golang.org/x/image 从 0.33.0 升级到 0.34.0 由 @dependabot[bot] 提交于 #1045
- build(deps)：将 actions/cache 从 4 升级到 5 由 @dependabot[bot] 提交于 #1046
- build(deps)：将 github.com/mazznoer/csscolorparser 从 0.1.6 升级到 0.1.7 由 @dependabot[bot] 提交于 #1047
- build(deps)：将 golang.org/x/image 从 0.34.0 升级到 0.35.0 由 @dependabot[bot] 提交于 #1049
- build(deps)：将 github.com/mazznoer/csscolorparser 从 0.1.7 升级到 0.1.8 由 @dependabot[bot] 提交于 #1050
- 为主窗口添加 SetSizeChangeCallback 由 @Orygin 提交于 #1051
- build(deps)：将 golang.org/x/image 从 0.35.0 升级到 0.36.0 由 @dependabot[bot] 提交于 #1053
- 允许将焦点设置到 InputText 控件 由 @Orygin 提交于 #1052
- build(deps)：将 golang.org/x/image 从 0.36.0 升级到 0.38.0 由 @dependabot[bot] 提交于 #1056
- build(deps)：将 golang.org/x/image 从 0.38.0 升级到 0.39.0 由 @dependabot[bot] 提交于 #1057
- build(deps)：将 github.com/sahilm/fuzzy 从 0.1.1 升级到 0.1.2 由 @dependabot[bot] 提交于 #1058
- build(deps)：将 golang.org/x/image 从 0.39.0 升级到 0.40.0 由 @dependabot[bot] 提交于 #1059
- build(deps)：将 golang.org/x/image 从 0.40.0 升级到 0.41.0 由 @dependabot[bot] 提交于 #1061
- deps：将 cimgui-go 更新到 v1.5.0 由 @gucio321 提交于 #1063
- workflows：将 golangci-lint 更新到 v2.11.5 由 @gucio321 提交于 #1064
- 破坏性变更(ContextMenu)：将 MouseButton 重命名为 Flags + 更改标志类型 由 @gucio321 提交于 #1066
新贡献者
衷心感谢！❤️ 🎉 🚀
完整变更日志：v0.14.1...v0.15.0
