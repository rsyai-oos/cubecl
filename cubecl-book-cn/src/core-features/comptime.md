# 编译时计算 (Comptime)

CubeCL 不仅仅是一种新的计算语言：虽然它看起来像是在编写 GPU 内核，但实际上，您正在编写可以完全自定义的编译器插件！编译时计算（Comptime）是一种在首次编译内核时修改编译器中间表示（IR）的方法。

这种方法能够在不编写多个相同内核变体的情况下实现大量优化和灵活性，从而确保最佳性能。

## 循环展开

在 CubeCL 中，您可以轻松地通过在 `for` 循环上使用 `unroll` 属性来展开循环。

```rust
#[cube(launch)]
fn sum<F: Float>(input: &Array<F>, output: &mut Array<F>, #[comptime] end: Option<u32>) {
    let unroll = end.is_some();
    let end = end.unwrap_or_else(|| input.len());
    let mut sum = F::new(0.0);

    #[unroll(unroll)]
    for i in 0..end {
        sum += input[i];
    }

    output[ABSOLUTE_POS] = sum;
}
```

请注意，如果您提供了一个无法在编译时确定的变量 `end`，则在尝试执行该内核时会引发错误。

## 特性专业化

您还可以通过平面操作来实现求和功能。我们将编写一个内核，当基于编译时特性标志可用时，使用该指令；当不可用时，它将回退到之前的实现，从而使其具有可移植性。

```rust
#[cube(launch)]
fn sum_plane<F: Float>(
    input: &Array<F>,
    output: &mut Array<F>,
    #[comptime] plane: bool,
    #[comptime] end: Option<u32>,
) {
    if plane {
        output[UNIT_POS] = plane_sum(input[UNIT_POS]);
    } else {
        sum_basic(input, output, end);
    }
}
```

请注意，在 GPU 上不会实际发生分支操作，因为可以从上述代码片段生成三个不同的内核。您还可以使用 [特性系统](../language-support/trait.md) 来实现类似的行为。