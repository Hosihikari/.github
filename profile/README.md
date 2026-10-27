# Hosihikari Projection

<table align="right">
  <tr>
    <td align="right">
      <img src="assets/mascot.png" width="190" alt="Hosi Hikari"><br>
      <sub>Art by 紫芈</sub>
    </td>
  </tr>
</table>

[English](README.md) | [简体中文](README.zh_hans.md)

Hosihikari Projection (星光投影) projects the Minecraft Bedrock native APIs into .NET bindings. It owns the native-to-C-ABI and native-to-managed projection only; plugin lifecycles, the framework, and game logic live elsewhere. A libTooling frontend turns C++ declarations and debug symbols into a versioned intermediate representation; emitters turn that IR into a native-style Interop assembly and a C ABI binding shim, which the shim loader resolves at run time by module offset, vtable slot, or exported symbol.

Three interop surfaces are generated from the same IR pipeline: the Bedrock interop surface from BDS 1.21.3.01 symbols, and the LeviLamina (LLAPI) and Endstone interop surfaces from their public headers. Each repository carries its own coverage contract, and CI verifies that contract against the emitted sources on every push.

**Frontend**: LLVM 23.1.3 libTooling · IR 1.5.1 · Generator 0.3.0  
**Platforms**: Windows x64 · Linux x64 · `win-x64-server` and `linux-x64-server` configurations  
**Carriers**: BDS 1.21.3.01 (direct MCAPI) · LeviLamina 26.51 (LLAPI) · Endstone 0.11

## Coverage

Coverage counts come from the committed `coverage/manifest.jsonl` of each interop surface. "Callable" excludes members that the emitter keeps as reasoned `NotSupportedException` stubs, and `excluded` covers private members.

**Rate** is callable over bound members. **Net rate** is callable over all declared members, private ones included; it is the share of the whole interop surface that a caller can actually invoke.

| Interop surface | Source | Members | Callable | Stubbed | Excluded | Rate | Net rate |
|---|---|---:|---:|---:|---:|---:|---:|
| Endstone | Endstone headers | 1,831 | 1,688 | 143 | 268 | 92.2% | **80.4%** |
| Bedrock (MCAPI) | 1.21.3.01 debug symbols; v0.13.5 header supplement | 113,084 | 86,608 | 26,476 | 5,141 | 76.6% | **73.3%** |
| LeviLamina (LLAPI) | LeviLamina headers | 5,902 | 3,053 | 2,849 | 243 | 51.7% | **49.7%** |

Bedrock binds 3,885 selected classes; closing over the records those classes reference by value, the surface reaches 4,245 of the 5,952 types that have both a symbol and a header declaration — 53,092 of 57,731 symbols. Remaining MCAPI stubs are dominated by members with no symbol or vtable slot (12,727) and constructors with no symbol (7,259).

### MCAPI symbol coverage

The MCAPI numbers above describe the selected set. Symbol coverage describes how much of the discovered server interop surface that set reaches.

Discovery covers 23,410 classes, but about 18,000 of them are standard-library instantiations, Itanium-encoded RTTI names, and anonymous-namespace artifacts. The figures below count a type as bindable only when it has both a symbol and a header declaration, so the denominator is the set a class can actually be selected from.

| Measure | Covered | Discovered | Coverage |
|---|---:|---:|---:|
| Classes | 4,245 | 5,952 | 71.3% |
| Public symbols | 53,092 | 57,731 | **92.0%** |

`BindingGenerator/artifacts/symbol-classes-1.21.3.tsv` is the discovery list; `scripts/classes-1.21.3.txt` in the Bedrock repository is the selected subset. Coverage moves only when a class is added.

The remaining 7.4% is a long tail: 4,639 symbols spread over 1,707 classes, averaging 2.7 symbols each.

LLAPI stubs are overwhelmingly template handling: rvalue-qualified members (1,612), inaccessible or incomplete template arguments (1,015), template constructors without a visible definition (826), and assignment operators on specializations (814).

## Interop surfaces

| Repository | Purpose |
|---|---|
| [BedrockBinding](https://github.com/Hosihikari/BedrockBinding) | MCAPI Interop and symbol-dispatched binding shim for BDS 1.21.3.01 |
| [LeviLaminaBinding](https://github.com/Hosihikari/LeviLaminaBinding) | LLAPI Interop and binding shim over LeviLamina |
| [EndstoneBinding](https://github.com/Hosihikari/EndstoneBinding) | Endstone C++ API Interop and binding shim |
| [BindingGenerator](https://github.com/Hosihikari/BindingGenerator) | libTooling frontend, IR emitters, symbol discovery, validation tooling |
| [BindingIr](https://github.com/Hosihikari/BindingIr) | IR schema, records, JSONL reader/writer, canonical hash, validator |
| [BindingAbi](https://github.com/Hosihikari/BindingAbi) | C ABI header, value types, shim loader |

## Supported ABI

Scalars and enum underlying types; `std::string` and `std::string_view` values and const references; single-level `T*` and `T&` as `void*`; by-value records through registers or the hidden return buffer; `optional_ref<T>`, `std::vector<T>`, `std::optional<T>`, `std::shared_ptr<T>` and `std::weak_ptr<T>`; fields whose type is not projected surface their storage address as a readonly `nint`.

Maps with named key/value types, callbacks, variants, and custom owning wrappers remain stubs. Records without a resolvable copy constructor or destructor stay stubbed.

## Provenance

Symbol data is recovered from the proprietary 1.21.3.01 `bedrock_server.pdb`, `bedrock_server.exe`, and `bedrock_server_symbols.debug`; only derived facts and hashes are published. The Bedrock header supplement comes from the public LeviLamina v0.13.5 headers; the LLAPI surface is parsed from LeviLamina 26.51 headers. Each interop surface records the generator version, IR manifest hash, and carrier hashes in `coverage/provenance.json`, and `scripts/verify_generated.py` checks provenance, the coverage baseline, shim counts, and every generated file hash.

## License and trust boundary

LGPL-3.0-or-later for the IR, ABI, and the three binding interop surfaces; the generator is GPL-3.0. Bindings run in the server process through the binding shim, so a wrong ABI match faults the server process rather than throwing a managed exception.
