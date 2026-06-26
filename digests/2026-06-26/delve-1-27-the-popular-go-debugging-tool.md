# Delve 1.27: The Popular Go Debugging Tool

derekparker
发布于此
6 月 24 日 19:57
·
自此版本以来对 master 有 1 次提交
不可变
发布。仅可修改发布标题和说明。

更新内容
- service/dap：避免断点-断开连接后测试中的 Session.conn 竞态，由 @alexsaezm 提交于 #4317
- proc：实现帧指针栈展开，由 @alexsaezm 提交于 #4288
- proc：修复 arm64 上 sigpanic 帧的 SP 计算，由 @alexsaezm 提交于 #4319
- native：在 linux/ppc64le 上传播 PtraceSetRegs 错误，由 @cuiweixie 提交于 #4321
- native：在 linux/ppc64le 上传播 PtraceGetRegs 错误，由 @cuiweixie 提交于 #4322
- proc：在复合内存中传播 AddrPiece ReadMemory 错误，由 @cuiweixie 提交于 #4323
- proc/internal/ebpf：修复段加载时 AddressToOffset 的差一错误，由 @cuiweixie 提交于 #4324
- proc：修复 breakpointConditionSatisfiable 中的 OR 处理，由 @cuiweixie 提交于 #4325
- dwarf/reader：在 Reader 条目迭代过程中传播错误，由 @cuiweixie 提交于 #4327
- proc：在 stride 溢出后退出 loadArrayValues，由 @cuiweixie 提交于 #4328
- proc：仅编译一次命中条件正则表达式，由 @cuiweixie 提交于 #4335
- service/dap：为构建消息添加换行符，由 @sagg0t 提交于 #4340
- proc：修复 go1.27 正则表达式重构的测试，由 @aarzilli 提交于 #4339
- proc：修复 go1.27 的 range over func 单步执行，由 @aarzilli 提交于 #4343
- chore：修正部分注释以提升可读性，由 @box4wangjing 提交于 #4344
- Teamcity：重新启用 riscv64 构建，由 @aarzilli 提交于 #4346
- proc：修复栈回溯中的 hasInlines，修复内联时的 range 单步执行，由 @aarzilli 提交于 #4345
- chore：修正注释以提升可读性，由 @cuoguojida 提交于 #4350
- service/dap：放宽 TestBadLaunchRequest，由 @aarzilli 提交于 #4351
- TeamCity：调整执行超时，由 @vietage 提交于 #4354
- proc/internal/ebpf：切换到“头部+参数”事件环形缓冲区协议，由 @derekparker 提交于 #4352
- proc：为带嵌入字段选择器的结构体字面量添加测试，由 @aarzilli 提交于 #4362
- proc：在 riscv64 上禁用部分失败的测试，由 @aarzilli 提交于 #4361
- gobuild：增加在 Windows 上删除二进制文件的等待时长，由 @aarzilli 提交于 #4359
- proc：为访问全局 C 变量添加测试，由 @aarzilli 提交于 #4358
- proc：支持泛型方法，由 @aarzilli 提交于 #4356
- 还原“winarm64：移除实验性构建标签 (#4176)”，由 @aarzilli 提交于 #4281
- proc/test：将 GOEXPERIMENT 纳入 fixture 缓存键，由 @typesanitizer 提交于 #4367
- proc：修复 go1.27 / arm64 上的单步执行测试，由 @aarzilli 提交于 #4365
- service/dap：添加 dap 写内存请求处理器，由 @DrSergei 提交于 #4364
- proc：使 PushPackageVarOrSelect 优先检查局部变量，由 @aarzilli 提交于 #4181
- pkg/proc：添加 GOEXPERIMENT=mapsplitgroup 支持，由 @aarzilli 提交于 #4370
- service/dap：在 TerminatedEvent 之前发送 ExitedEvent，由 @derekparker 提交于 #4371
- service/dap：修复 ExitedEvent 改动导致的测试失败，由 @derekparker 提交于 #4373
- service/dap,pkg/debugdetect：修复测试，由 @derekparker 提交于 #4375
- v1.27.0，由 @derekparker 提交于 #4372

新贡献者
- @sagg0t 在 #4340 中首次贡献
- @box4wangjing 在 #4344 中首次贡献
- @cuoguojida 在 #4350 中首次贡献

完整更新日志：v1.26.3...v1.27.0
精选更新日志：https://github.com/go-delve/delve/blob/master/CHANGELOG.md#1270-2026-06-19
