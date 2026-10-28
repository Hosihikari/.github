# 星光 · 星光投影

<table align="right">
  <tr>
    <td align="right">
      <img src="assets/mascot.png" width="190" alt="星光"><br>
      <sub>立绘：紫芈</sub>
    </td>
  </tr>
</table>

[English](README.md) | [简体中文](README.zh_hans.md)

星光投影把 Minecraft 基岩版原生 API 投影成 .NET 绑定。它只负责“原生事实 → C ABI / 托管投影”，不负责插件生命周期、框架本身或游戏逻辑。libTooling 前端把 C++ 声明与调试符号转成带版本的中间表示；发射器再把 IR 转成贴近原生风格的 Interop 程序集与 C ABI binding shim，shim 加载器在运行时按模块偏移、vtable 槽或导出符号解析。

三条互操作面共用同一条 IR 管线：Bedrock 互操作面来自 BDS 1.21.3.01 的符号，LeviLamina（LLAPI）与 Endstone 互操作面来自各自公开的头文件。每个仓库都带自己的覆盖率契约，CI 在每次推送时对照发射产物校验该契约。

**前端**：LLVM 23.1.3 libTooling · IR 1.5.1 · 生成器 0.3.0  
**平台**：Windows x64 · Linux x64 · `win-x64-server` 与 `linux-x64-server` 配置  
**载体**：BDS 1.21.3.01（直连 MCAPI）· LeviLamina 26.51（LLAPI）· Endstone 0.11

## 覆盖

覆盖数字取自各互操作面提交的 `coverage/manifest.jsonl`。「可调用」不含发射器保留为带原因 `NotSupportedException` stub 的成员，`excluded` 为私有成员。

「比例」是「可调用」占已绑定成员的比例。「净比例」是「可调用」占全部声明成员（含私有）的比例，即调用方真正能调到的互操作面份额。

| 互操作面 | 来源 | 成员 | 可调用 | stub | excluded | 比例 | 净比例 |
|---|---|---:|---:|---:|---:|---:|---:|
| Endstone | Endstone 头文件 | 1,831 | 1,688 | 143 | 268 | 92.2% | **80.4%** |
| Bedrock（MCAPI） | 1.21.3.01 调试符号与 BDS 头文件 | 113,084 | 86,608 | 26,476 | 5,141 | 76.6% | **73.3%** |
| LeviLamina（LLAPI） | LeviLamina 头文件 | 5,902 | 3,053 | 2,849 | 243 | 51.7% | **49.7%** |

Bedrock 选出 3,885 个类；把这些类按值引用的记录一并闭包后，覆盖面到达 5,952 个「既有符号又有头文件声明」的类型中的 4,245 个，覆盖 57,731 个符号中的 53,092 个。其余 MCAPI stub 主要是无符号或无 vtable 槽的成员（12,727）与无符号的构造函数（7,259）。

### MCAPI 符号覆盖

上面的 MCAPI 数字描述的是已选类集。符号覆盖描述的是该类集对已发现服务端互操作面的到达程度。

发现结果共 23,410 个类，其中约 1.8 万个是标准库实例化、Itanium 编码的 RTTI 名与匿名命名空间产物。下表只把「同时有符号与头文件声明」的类型算作可绑定，分母是真正可供选类的集合。

| 度量 | 已覆盖 | 已发现 | 覆盖率 |
|---|---:|---:|---:|
| 类 | 4,245 | 5,952 | 71.3% |
| 公开符号 | 53,092 | 57,731 | **92.0%** |

`BindingGenerator/artifacts/symbol-classes-1.21.3.tsv` 是发现清单，Bedrock 仓库的 `scripts/classes-1.21.3.txt` 是其中被选中的子集。只有新增类时覆盖率才会变动。

剩下 7.4% 是长尾：4,639 个符号分散在 1,707 个类里，平均每类 2.7 个。

LLAPI 的 stub 高度集中在模板处理：rvalue 限定成员（1,612）、不可访问或不完整的模板参数（1,015）、无可见定义的模板构造（826）、模板特化的赋值运算符（814）。

## 互操作面

| 仓 | 作用 |
|---|---|
| [BedrockBinding](https://github.com/HosihikariProjection/BedrockBinding) | BDS 1.21.3.01 的 MCAPI Interop 与按符号分发的 binding shim |
| [LeviLaminaBinding](https://github.com/HosihikariProjection/LeviLaminaBinding) | 基于 LeviLamina 的 LLAPI Interop 与 binding shim |
| [EndstoneBinding](https://github.com/HosihikariProjection/EndstoneBinding) | Endstone C++ API 的 Interop 与 binding shim |
| [BindingGenerator](https://github.com/HosihikariProjection/BindingGenerator) | libTooling 前端、IR 发射器、符号发现与校验工具 |
| [BindingIr](https://github.com/HosihikariProjection/BindingIr) | IR schema、记录模型、JSONL 读写、规范哈希与校验器 |
| [BindingAbi](https://github.com/HosihikariProjection/BindingAbi) | C ABI 头、值类型与 shim 加载器 |

## 支持的 ABI

标量与枚举底层类型；`std::string` 与 `std::string_view` 的值与 const 引用；单层 `T*` 与 `T&` 以 `void*` 传递；按值 record 走寄存器或隐藏返回缓冲区；`optional_ref<T>`、`std::vector<T>`、`std::optional<T>`、`std::shared_ptr<T>` 与 `std::weak_ptr<T>`；类型未投影的字段以只读 `nint` 暴露存储地址。

map（需具名 key/value 类型）、回调、variant 与自定义拥有型包装仍为 stub。缺少可解析拷贝构造或析构的 record 保持 stub。

## 数据来源

符号数据取自专有的 1.21.3.01 `bedrock_server.pdb`、`bedrock_server.exe` 与 `bedrock_server_symbols.debug`，只发布派生事实与哈希。Bedrock 的头文件补充是 LeviLamina 转载的 BDS 头，只用来提供声明与布局，不为它发射任何东西。LLAPI 互操作面解析的是 LeviLamina 26.51 的头。各互操作面把生成器版本、IR manifest 哈希与载体哈希记录在 `coverage/provenance.json`，`scripts/verify_generated.py` 校验来源、覆盖基线、shim 数量与每个生成文件的哈希。

## 许可与信任边界

IR、ABI 与三条绑定互操作面为 LGPL-3.0-or-later；生成器为 GPL-3.0。绑定经 binding shim 在服务器进程内运行，ABI 匹配错误会让服务器进程崩溃，而不是抛出托管异常。
