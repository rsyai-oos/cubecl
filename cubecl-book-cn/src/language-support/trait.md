# 特性支持 (Trait Support)

CubeCL 部分支持特性（trait），以便在不产生任何开销的情况下模块化您的内核代码。目前，大多数功能都已支持，除了有状态的函数。

```rust
#[cube]
trait MyTrait {
    /// 支持
    fn my_function(x: &Array<f32>) -> f32;
    /// 不支持
    fn my_function_2(&self, x: &Array<f32>) -> f32;
}
```

特性系统允许您轻松实现专业化。我们以 [编译时计算部分](../core-features/comptime.md) 的相同示例为例。

首先，您可以定义您的特性。请注意，如果您从启动函数中使用该特性，则需要添加 `'static + Send + Sync`。

```rust
#[cube]
trait SumKind: 'static + Send + Sync {
    fn sum<F: Float>(input: &Slice<F>, #[comptime] end: Option<u32>) -> F;
}
```

接下来，我们可以定义一些实现：

```rust
struct SumBasic;
struct SumPlane;

#[cube]
impl SumKind for SumBasic {
    fn sum<F: Float>(input: &Slice<F>, #[comptime] end: Option<u32>) -> F {
        let unroll = end.is_some();
        let end = end.unwrap_or_else(|| input.len());

        let mut sum = F::new(0.0);

        #[unroll(unroll)]
        for i in 0..end {
            sum += input[i];
        }

        sum
    }
}

#[cube]
impl SumKind for SumPlane {
    fn sum<F: Float>(input: &Slice<F>, #[comptime] _end: Option<u32>) -> F {
        plane_sum(input[UNIT_POS])
    }
}
```

关联类型也受到支持。假设您想从一个求和结果创建一个序列。

```rust
#[cube]
trait CreateSeries: 'static + Send + Sync {
    type SumKind: SumKind;

    fn execute<F: Float>(input: &Slice<F>, #[comptime] end: Option<u32>) -> F;
}
```

您可能希望通过实现来定义要创建的序列类型。

```rust
struct SumThenMul<K: SumKind> {
    _p: PhantomData<K>,
}

#[cube]
impl<K: SumKind> CreateSeries for SumThenMul<K> {
    type SumKind = K;

    fn execute<F: Float>(input: &Slice<F>, #[comptime] end: Option<u32>) -> F {
        let val = Self::SumKind::sum(input, end);
        val * input[UNIT_POS]
    }
}
```

实际上，这并不是使用关联类型的最佳示例，但它展示了 CubeCL 对关联类型的完全支持。