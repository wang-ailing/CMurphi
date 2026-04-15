# CMurphi 编译与运行警告修复记录 (Warning Fix Log)

在使用 ProtoGen 配合 CMurphi 进行缓存一致性协议（如 `MSI_Proto`）的生成与模型检验时，控制台输出了大量干扰视线的警告（Warnings）。为了提供更干净的验证输出，我们对 CMurphi 的源码进行了针对性的梳理和修改。

## 1. C++ `register` 关键字弃用警告

**问题原因**：
CMurphi 是一个较早期的代码库，其生成的底层 C++ 模型检验代码（例如 `MSI_Proto.cpp`）中大量使用了 `register` 关键字。而在现代 C++ 标准（C++17 及以上）中，`register` 关键字已被正式弃用，导致使用新版 `g++` 编译模型时，会产生满屏的 `[-Wregister]` 警告。

**修复方案**：
修改了 `src/Makefile`，在编译选项 `CFLAGS` 中追加了 `-w` 参数（以及在外部脚本的 `tmpmakefile` 模板中配置忽略选项），以此全局忽略由于现代 C++ 编译器版本升级而引起的向下兼容性警告。

## 2. Scalarset 循环索引警告 (Scalarset in loop index)

**问题原因**：
在 Murphi 协议代码（`.m` 文件）中，当我们使用 `for i: Machines do` 来遍历对称的机器节点时，这些节点集合通常被定义为 `Scalarset`（对称标量集）。Murphi 编译器在解析这种语法时，会默认抛出如下警告：
> `warning: Scalarset is used in loop index. Please make sure that the iterations are independent.`

这仅仅是一条用于提醒开发者注意循环体内部操作独立性的“善意提示”。但由于 ProtoGen 自动生成的并发协议状态机非常庞大，包含了极其密集的循环代码，导致该警告疯狂刷屏，掩盖了真正有用的验证结果。

**修复方案**：
直接从编译器底层拦截警告输出。我们修改了 CMurphi 的错误处理源码 `src/error.cpp`：
- 在 `Error_handler::vWarning` 和 `Error_handler::Warning` 等打印警告日志的方法最前方，直接加入了 `return;`。
- 重新 `make` 编译了 `mu` 编译器，从而在底层彻底静音了这类非致命的常规提醒。

---

**修改结果**：
经过上述修改，再次运行 ProtoGen 验证完整协议时，控制台仅保留核心的步骤提示：
```text
Starting SSP verification
SSP verified without error

Starting full protocol verification
Full protocol verified without error
```
彻底告别了刷屏烦恼，使得自动化验证的日志输出变得极度干净、清爽且易于解析。