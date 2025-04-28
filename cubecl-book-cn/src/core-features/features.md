# 查询硬件特性

某些特性和数据类型仅在某些硬件或后端上受支持。可以通过以下方式查询：

```rust, ignore
client.properties().feature_enabled(feature)
```

另请参阅 [`Feature`](https://docs.rs/cubecl/latest/cubecl/enum.Feature.html)。

## 概述

### 特性

还需要设备支持

| 特性   | CUDA | ROCm | WGPU (WGSL) | WGPU (SPIR-V) |
| ------ | ---- | ---- | ----------- | ------------- |
| Plane  | ✔️   | ✔️   | ✔️          | ✔️            |
| CMMA   | ✔️   | ✔️   | ❌          | ✔️            |

### 数据类型

`flex32` 在 SPIR-V 以外的所有地方都表示为 `f32`，且没有降低精度。`f64` 不支持所有操作。

| 类型    | CUDA | ROCm | WGPU (WGSL) | WGPU (SPIR-V) |
| ------- | ---- | ---- | ----------- | ------------- |
| u8      | ✔️   | ✔️   | ❌          | ✔️            |
| u16     | ✔️   | ✔️   | ❌          | ✔️            |
| u32     | ✔️   | ✔️   | ✔️          | ✔️            |
| u64     | ✔️   | ✔️   | ❌          | ✔️            |
| i8      | ✔️   | ✔️   | ❌          | ✔️            |
| i16     | ✔️   | ✔️   | ❌          | ✔️            |
| i32     | ✔️   | ✔️   | ✔️          | ✔️            |
| i64     | ✔️   | ✔️   | ❌          | ✔️            |
| f16     | ✔️   | ✔️   | ❌          | ✔️            |
| bf16    | ✔️   | ✔️   | ❌          | ❌            |
| flex32  | ❔    | ❔    | ❔          | ✔️            |
| tf32    | ✔️   | ❌   | ❌          | ❌            |
| f32     | ✔️   | ✔️   | ✔️          | ✔️            |
| f64     | ❔    | ❔    | ❌          | ❔            |
| bool    | ✔️   | ✔️   | ✔️          | ✔️            |

## 数据类型细节

### Flex32

放宽精度的 32 位浮点数。最小范围和精度等同于 `f16`，但可能更高。当不支持放宽精度时，默认为 `f32`。

### Tensor-Float32

仅限 CUDA 的 19 位类型，仅应作为 CMMA 矩阵类型使用。可能可以从 `f32` 重新解释，但官方未定义。请使用 `Cast::cast_from` 安全转换。

## 特性细节

### Plane

平面级操作，例如：
[`plane_sum`](https://docs.rs/cubecl/latest/cubecl/frontend/fn.plane_sum.html),
[`plane_elect`](https://docs.rs/cubecl/latest/cubecl/frontend/fn.plane_elect.html)。

### 合作矩阵乘加 (CMMA)

平面级合作矩阵乘加操作。映射到 CUDA 中的 `wmma` 和 SPIR-V 中的 `CooperativeMatrixMultiply`。硬件支持的每个大小和数据类型都会注册特性。支持的函数详见：
[`cmma`](https://docs.rs/cubecl/latest/cubecl/frontend/cmma/index.html)。