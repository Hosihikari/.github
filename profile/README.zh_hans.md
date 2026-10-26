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

三条能力面共用同一条 IR 管线：Bedrock 能力面来自 BDS 1.21.3.01 的符号，LeviLamina（LLAPI）与 Endstone 能力面来自各自公开的头文件。每个仓库都带自己的覆盖率契约，CI 在每次推送时对照发射产物校验该契约。

**前端**：LLVM 23.1.3 libTooling · IR 1.5.1 · 生成器 0.3.0  
**平台**：Windows x64 · Linux x64 · `win-x64-server` 与 `linux-x64-server` 配置  
**载体**：BDS 1.21.3.01（直连 MCAPI）· LeviLamina v0.13.5（LLAPI，26.51）· Endstone 0.11

## 覆盖

覆盖数字取自各能力面提交的 `coverage/manifest.jsonl`。「可调用」不含发射器保留为带原因 `NotSupportedException` stub 的成员，`excluded` 为私有成员。

「比例」是「可调用」占已绑定成员的比例。「净比例」是「可调用」占全部声明成员（含私有）的比例，即调用方真正能调到的能力面份额。

| 能力面 | 来源 | 成员 | 可调用 | stub | excluded | 比例 | 净比例 |
|---|---|---:|---:|---:|---:|---:|---:|
| Endstone | Endstone 头文件 | 1,831 | 1,688 | 143 | 268 | 92.2% | **80.4%** |
| Bedrock（MCAPI） | 1.21.3.01 调试符号 | 65,715 | 56,060 | 9,655 | 3,903 | 85.3% | **80.5%** |
| LeviLamina（LLAPI） | LeviLamina 头文件 | 5,902 | 3,053 | 2,849 | 243 | 51.7% | **49.7%** |

Bedrock 从过滤后的 6,677 个可发现类中选出 1,403 个，覆盖过滤后 49,635 个公开符号中的 30,402 个。其余 MCAPI stub 主要是无符号或无 vtable 槽的方法（4,456）与无符号的构造函数（2,495）。

### MCAPI 符号覆盖

上面的 MCAPI 数字描述的是已选类集。符号覆盖描述的是该类集对已发现服务端能力面的到达程度。

发现结果共 24,632 个类，其中约 1.8 万个是标准库实例化、Itanium 编码的 RTTI 名与匿名命名空间产物。下表与选类使用同一套过滤，分母是真正可供选类的集合。

| 度量 | 已覆盖 | 已发现 | 覆盖率 |
|---|---:|---:|---:|
| 类 | 1,403 | 6,677 | 21.0% |
| 公开符号 | 30,402 | 49,635 | **61.3%** |

`BindingGenerator/artifacts/symbol-classes-1.21.3.tsv` 是发现清单，Bedrock 仓库的 `scripts/classes-1.21.3.txt` 是其中被选中的子集。只有新增类时覆盖率才会变动。

剩下 38.7% 是长尾：19,233 个符号分散在 5,274 个类里，平均每类 3.6 个。

LLAPI 的 stub 高度集中在模板处理：rvalue 限定成员（1,612）、不可访问或不完整的模板参数（1,015）、无可见定义的模板构造（826）、模板特化的赋值运算符（814）。

## 能力面

| 仓 | 作用 |
|---|---|
| [BedrockBinding](https://github.com/Hosihikari/BedrockBinding) | BDS 1.21.3.01 的 MCAPI Interop 与按符号分发的 binding shim |
| [LeviLaminaBinding](https://github.com/Hosihikari/LeviLaminaBinding) | 基于 LeviLamina 的 LLAPI Interop 与 binding shim |
| [EndstoneBinding](https://github.com/Hosihikari/EndstoneBinding) | Endstone C++ API 的 Interop 与 binding shim |
| [BindingGenerator](https://github.com/Hosihikari/BindingGenerator) | libTooling 前端、IR 发射器、符号发现与校验工具 |
| [BindingIr](https://github.com/Hosihikari/BindingIr) | IR schema、记录模型、JSONL 读写、规范哈希与校验器 |
| [BindingAbi](https://github.com/Hosihikari/BindingAbi) | C ABI 头、值类型与 shim 加载器 |

## 支持的 ABI

标量与枚举底层类型；`std::string` 与 `std::string_view` 的值与 const 引用；单层 `T*` 与 `T&` 以 `void*` 传递；按值 record 走寄存器或隐藏返回缓冲区；`optional_ref<T>`、`std::vector<T>`、`std::optional<T>`、`std::shared_ptr<T>` 与 `std::weak_ptr<T>`；类型未投影的字段以只读 `nint` 暴露存储地址。

map（需具名 key/value 类型）、回调、variant 与自定义拥有型包装仍为 stub。缺少可解析拷贝构造或析构的 record 保持 stub。

## 数据来源

符号数据取自专有的 1.21.3.01 `bedrock_server.pdb`、`bedrock_server.exe` 与 `bedrock_server_symbols.debug`，只发布派生事实与哈希。头文件补充来自公开的 LeviLamina 头。各能力面把生成器版本、IR manifest 哈希与载体哈希记录在 `coverage/provenance.json`，`scripts/verify_generated.py` 校验来源、覆盖基线、shim 数量与每个生成文件的哈希。

## 许可与信任边界

IR、ABI 与三条绑定能力面为 LGPL-3.0-or-later；生成器为 GPL-3.0。绑定经 binding shim 在服务器进程内运行，ABI 匹配错误会让服务器进程崩溃，而不是抛出托管异常。
