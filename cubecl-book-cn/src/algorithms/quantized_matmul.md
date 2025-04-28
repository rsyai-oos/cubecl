# 量化矩阵乘法

为了加速矩阵乘法，我们将使用 `f32` 的浮点运算替换为使用 `u8`、`u16` 和 `i32` 的整数运算。这种做法有两个好处。
首先，我们将 `Tensor<f32>` 替换为 `Tensor<u8>`，从而将内存成本降低 4 倍。这导致更快的全局内存读写操作。
其次，整数运算通常比相应的浮点运算更快。

在本节中，我们将首先介绍算法的数学公式，然后讨论其实现。

## 数学公式

### 标量量化

一个实数 \\(a\\) 可以通过以下公式用一个整数 \\(q\\) 近似：
\\[
    a \approx s(q - z).
\\]
在这个公式中，\\(s\\) 是缩放因子，也是一个实数，而 \\(z\\) 称为零偏移，是一个整数。
理论上，通过这种近似，我们可以精确表示所有是 \\(s\\) 的整数倍的实数。
所有其他实数将被四舍五入到最近的可表示值。
然而，在实践中，\\(q\\) 的范围受到其表示方式（例如 `u8`、`i32`）的限制。
因此，零偏移 \\(z\\) 允许我们将可表示数值的区间滑动到我们感兴趣的特定应用区间。
此外，通过为 \\(q\\) 和 \\(z\\) 使用相同的类型，我们确保 0 是可以精确表示的。

两个实数的乘法等价于
\\[
  a b = s_a s_b (q_a - z_a) (q_b - z_b).
\\]
然而，我们更感兴趣的是 \\(c = ab\\) 的量化版本 \\(q_c\\)。
如果我们希望用缩放因子 \\(s_c\\) 和零偏移 \\(z_c\\) 来近似 \\(c\\)，则有
\\[
  q_c =
  z_c + \frac{s_a s_b}{s_c} (q_a - z_a) (q_b - z_b).
\\]
除了因子 \\( (s_a s_b) / s_c \\) 之外，上述公式仅涉及整数运算。
然而，我们总能找到两个整数 \\(u, v\\) 使得
\\[
  \frac uv \approx \frac{s_a s_b}{s_c}
\\]
是一个满意的近似值。
这导致了量化乘法的最终近似公式
\\[
  q_c \approx z_c + \frac uv (q_a - z_a)(q_b - z_b)
\\]
仅涉及整数运算。

### 矩阵量化

同样的想法适用于矩阵乘法。
为了区分矩阵和标量，我们使用大写字母表示矩阵，小写字母表示标量。

一个实矩阵 \\( A \\) 可以通过以下公式用一个整数矩阵 \\( Q \\) 近似：
\\[
  A \approx s (Q - z N).
\\]
这里 \\( N \\) 是一个与 \\( A \\) 大小相同的全 1 矩阵。
对于两个形状分别为 \\(m \times k\\) 和 \\(k \times n\\) 的矩阵 \\(A\\) 和 \\(B\\) 及其乘积 \\(C\\) 形状为 \\(m \times n\\)，我们有，类似于标量情况，
\\[
  Q_c \approx z_c N_c + \frac uv (Q_a - z_a N_a)(Q_b - z_b N_b).
\\]

## 实现

作为示例，我们将描述如何实现量化矩阵乘法，其中 \\(Q_a\\)、\\(Q_b\\) 和 \\(Q_c\\) 的元素以及零偏移量表示为 `u8`。

为了计算 \\(Q_a - z_a N_a\\)，我们首先将值转换为 `i16`，然后再进行减法运算。
然后，我们可以通过将值转换为 `i32` 来计算乘积 \\((Q_a - z_a N_a)(Q_b - z_b N_b)\\)。
当然，在实际操作中，我们会在运行时进行这些转换，以避免浪费地分配新矩阵。

现在，假设 \\(x\\) 是结果矩阵中的一个元素，而 \\(y\\) 是 \\(Q_c\\) 中相同位置的元素。
我们仍然需要计算以下公式：
\\[
  y = z_c + \frac uv \cdot x.
\\]
这里的难点在于乘法运算。
首先，我们规定 \\( v \\) 是 2 的幂，这样除以 \\( v \\) 等价于将乘积 \\( u x \\) 右移。
然后，我们需要找到最佳的 \\( u \\) 和 \\( v \\) 值来表示缩放因子 \\( \sigma = \frac{s_a s_b}{s_c} \\)。
技巧在于巧妙地将 \\( \sigma \\) 乘以 1，以获得一种允许我们使用 2 的幂的形式：
\\[
  \sigma = \frac{2^{31 - f}}{2^{31 - f}} \sigma
\\]
其中 \\(2^f\\) 是大于 \\(\sigma\\) 的最小 2 的幂。
例如，如果 \\(\sigma = 0.3\\)，则 \\(f = -1\\)，因为 \\(2^{-1} = 0.5 > 0.3\\) 而 \\(2^{-2} = 0.25 < 0.3\\)。
从这里我们可以推断出我们可以使用 \\(u = 2^{31 - f} \sigma\\) 四舍五入到最近的 `i64` 值，并且 \\(v = 2^{31 - f}\\)。
这为我们提供了 31 位的乘以 \\(\sigma\\) 的近似值，这是当另一个乘数是 `i32` 时我们能达到的最佳效果。
确实，我们需要保留一位符号位。
为了正确地四舍五入乘积，可以在右移之前向乘积添加 \\(\frac v 2\\)。

上述算法的简单实现如下所示。
```rust
fn scaling_ratio(sigma: f32) -> (i64, u32) {
    let log = sigma.log2().ceil() as i32;
    let u = (sigma * 2.0_f32.powi(31 - log)).round() as i64;
    let v_shift = (31 - log) as u32;
    (u, v_shift)
}

fn approx_mul(x: i32, u: i64, v_shift: u32) -> i32 {
    let prod = (x as i64) * u;
    let rounding: i64 = 1 << (v_shift - 1);
    let prod_with_rounding = prod + rounding;
    (prod_with_rounding >> v_shift) as i32
}

fn clamp_to_u8(x: i32) -> u8 {
    if x < 0 {
        0
    } else if x > u8::MAX as i32 {
        u8::MAX
    } else {
        x as u8
    }
}

struct Matrix {
    scaling: f32,
    zero_offset: u8,
    // ... 其他字段用于存储矩阵元素。
}

impl Matrix {
    fn quantized_mul(&self, other: &Self, output: &mut Self) {
        // 假设矩阵的形状匹配。

        let sigma = self.scaling * other.scaling / output.scaling;
        let (u, v_shift) = scaling_ratio(sigma);

        for row in 0..self.row_count() {
            for col in 0..other.col_count() {
                let mut sum: i32 = 0;
                for middle in 0..self.col_count() {
                    let a = self.get(row, middle) as i16 - self.zero_offset as i16;
                    let b = other.get(middle, col) as i16 - other.zero_offset as i16;
                    sum += (a as i32) * (b as i32);
                }
                sum = approx_mul(sum, u, v_shift);

                output.update(row, col, clamp_to_u8(sum + output.zero_offset as i32));
            }
        }
    }

    // 返回 (row, col) 处的值
    fn get(&self, row: usize, col: usize) -> u8 { /* ... */ }

    // 将 (row, col) 处的值替换为给定值
    fn update(&mut self, row: usize, col: usize, value: u8) { /* ... */ }

    // 返回矩阵的行数
    fn row_count(&self) -> usize { /* ... */ }

    // 返回矩阵的列数
    fn col_count(&self) -> usize { /* ... */ }
}
```
当然，在 CubeCL 中，我们致力于为 GPU 设备提供最快的实现。
因此，该示例强调了正确的类型转换，以展示在 CubeCL 中如何实现这一点。