# Rust ndarray 高性能计算：从元素操作到矩阵运算的优化实践

## 一、迭代与映射：用 mapv 释放元素级处理潜力

在 Rust 的 `ndarray` 库中，迭代与映射是对数组元素进行操作的基础。

通过灵活运用这些操作，可以高效地处理数组中的每个元素，实现各种复杂的数据处理任务。

### 1.1 mapv 基础：逐元素映射与类型转换

`mapv`是 `ndarray` 中高效的元素级映射工具，接收闭包作为参数，返回与原数组维度相同的新数组。与惰性的`map`不同，`mapv`立即分配内存并计算结果，适合需要新数组的场景。

```rust

use ndarray::array;

fn main() {
    let sensor_readings = array![102.3, 0.0, 511.5, 1023.0f64];

    // 使用 mapv 进行归一化
    let normalized = sensor_readings.mapv(|x| x / 1023.0);

    println!("原始读数: \n{}", sensor_readings);
    println!("归一化后: \n{}", normalized);

    // 也可以执行更复杂的操作，比如 Sigmoid
    let activations = normalized.mapv(|x| 1.0 / (1.0 + (-x).exp()));
    println!("激活值: \n{}", activations);
}
```

**输出：**

```shell
原始读数:
[102.3, 0, 511.5, 1023]
归一化后:
[0.09999999999999999, 0, 0.5, 1]
激活值:
[0.5249791874789399, 0.5, 0.6224593312018546, 0.7310585786300049]
```

在具身智能场景中，当处理传感器数据时，`mapv`可用于对传感器读数进行预处理。

比如，将温度传感器的原始读数从摄氏度转换为华氏度，或者对压力传感器数据进行校准。

### 1.2 并行加速：par_mapv_inplace 应对大规模数据

借助 rayon 库，`par_mapv_inplace`支持多核并行处理，显著提升计算密集型任务效率（需启用`rayon`特性）。

`par_mapv_inplace` 被称之为**就地并行修改**， 是 `ndarray` 提供的最直接的并行 `map` 方法。它会启动一个线程池，并行地修改数组中的每一个元素，不返回任何东西 (())。

Cargo.toml配置：

```toml
[dependencies]
ndarray = { version = "0.17.1", features = ["rayon"] }
```

```rust

use ndarray::Array3;

fn main() {
    let mut matrix =
        Array3::from_shape_vec((2, 2, 2), vec![1.0f64, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0]).unwrap();
    // 并行计算每个元素的平方
    matrix.par_mapv_inplace(|x| x.powi(2));
    println!("并行计算值: \n{}", matrix);
}
```

**输出：**

```shell
并行计算值:
[[[1, 4],
  [9, 16]],

 [[25, 36],
  [49, 64]]]
```

## 二、元素级运算：简洁高效的逐元素操作

元素级运算是 `ndarray` 的核心功能之一，它允许我们对数组中的每个元素进行操作，而无需显式的循环。

这种向量化的操作方式不仅提高了代码的简洁性，还显著提升了执行效率。

### 2.1 基础算术运算：运算符与函数双支持

`ndarray` 支持直接使用`+`、`-`、`*`、`/`进行元素级运算，操作简单直观，代码可读性强。这些运算符会自动应用到数组的每个元素上，生成一个新的数组。

同时，`ndarray` 也提供了`add`、`sub`、`mul`、`div`等函数来实现相同的运算。使用函数形式可以在一些需要更灵活操作的场景中，确保类型安全和更好的错误处理。

```rust

use ndarray::array;
use std::ops::Mul;

fn main() {
    let a = array![[1, 2, 3], [4, 5, 6]];
    let b = array![[7, 8, 9], [10, 11, 12]];

    let a_clone = a.clone();
    let b_clone = b.clone();
    // 使用运算符进行元素级加法
    let c = a + b;
    println!("相加: \n{}", c);

    // 使用函数进行元素级乘法
    let d = a_clone.mul(&b_clone);
    println!("相乘: \n{}", d);
}
```

**输出：**

```shell
相加:
[[8, 10, 12],
 [14, 16, 18]]
相乘:
[[7, 16, 27],
 [40, 55, 72]]
```

### 2.2 数学函数应用：从基础到复杂运算

`ndarray` 支持`sqrt`、`sin`、`cos`等丰富的数学函数，这些函数可以直接作用于数组的每个元素，避免了手动迭代数组来应用这些函数的繁琐过程。

```rust

use ndarray::prelude::*;

fn main() {
    let a = array![1.0, 4.0, 9.0];
    // 计算每个元素的平方根
    let b = a.sqrt();
    println!("平方根: \n{}", b);

    let angles = array![0.0, std::f64::consts::PI / 2.0, std::f64::consts::PI];
    // 计算每个角度的正弦值
    let sines = angles.sin();
    println!("正弦值: \n{}", sines);
}
```

**输出：**

```shell
平方根:
[1, 2, 3]
正弦值:
[0, 1, 0.00000000000000012246467991473532]
```

## 三、广播机制：维度适配的隐形助手

在 `ndarray` 的世界里，广播机制是一种强大而又神奇的特性，它允许不同形状的数组在进行运算时自动适配维度，大大简化了代码的编写。

### 3.1 广播规则：从后缘维度对齐到自动扩展

`ndarray` 广播遵循「后缘对齐」原则，当两个数组进行运算时，如果它们的维度数不同，`ndarray` 会在较小数组的前面补 1，使其维度数与较大数组相同。

例如，一个形状为 (5, 3) 的数组与一个形状为 (3,) 的数组进行广播时，形状为 (3,) 的数组会被视为 (1, 3)，然后再与 (5, 3) 进行对齐，最终广播为 (5, 3)。

在维度对齐后，单维度（长度为 1）的维度会自动复制扩展，以匹配另一个数组的维度。这种扩展是逻辑上的，无需显式的数据复制，因此效率非常高。

### 3.2 实战场景：环境常数广播与状态计算

在具身智能的应用中，将环境常数广播到所有状态向量是一个常见的需求。

假设我们有一个机器人，它在不同的状态下需要考虑重力加速度的影响。重力加速度是一个常数，我们可以将其广播到机器人的所有状态向量上，从而在计算中考虑重力的作用。

```rust

use ndarray::array;

fn main() {
    // 定义重力加速度 (1,)
    let gravity = array![9.81]; 

    // 假设机器人有三个状态，每个状态包含位置和速度信息 (3, 2, 2)
    let states = array![
        [[0.0, 0.0], [1.0, 1.0]],
        [[2.0, 2.0], [3.0, 3.0]],
        [[4.0, 4.0], [5.0, 5.0]]
    ]; 

    // 将重力加速度广播到所有状态向量上
    let new_states = states + gravity; 
    println!("New states:\n {:?}", new_states); 
}
```

**输出：**

```shell
New states:
 [[[9.81, 9.81],
  [10.81, 10.81]],

 [[11.81, 11.81],
  [12.81, 12.81]],

 [[13.81, 13.81],
  [14.81, 14.81]]], shape=[3, 2, 2], strides=[4, 2, 1], layout=Cc (0x5), const ndim=3
```

在这个例子中，`gravity`是一个形状为 (1,) 的数组，`states`是一个形状为 (3, 2, 2) 的数组。通过广播机制，`gravity`会自动扩展为 (3, 2, 2) 的形状，与`states`进行匹配，然后进行元素级加法运算。

## 四、连接与堆叠：灵活组合多维数据

在处理多维数据时，我们常常需要将多个数组合并成一个更大的数组，或者将一个数组分割成多个小数组。ndarray 提供了`stack`和`concatenate`函数来满足这些需求，它们在具身智能中也有着广泛的应用，比如在批量生成控制信号时，就需要将多个控制信号数组合并成一个大的数组。

### 4.1 concatenate：沿现有轴连接数组

`concatenate`函数是沿指定轴连接数组，它允许输入数组在其他轴上的形状一致，只有连接轴上的长度可以不同。这使得`concatenate`在合并具有不同长度但相同结构的数据时非常灵活。

```rust

use ndarray::array;
use ndarray::Axis;

fn main() {
    let a = array![[1, 2], [3, 4]];
    let b = array![[5, 6]];

    // 沿轴0连接数组
    let c = ndarray::concatenate(Axis(0), &[a.view(), b.view()]).unwrap();
    println!("concatenate axis0:\n {:?}", c);

    let d = array![[7, 8], [9, 10]];
    // 沿轴1连接数组
    let e = ndarray::concatenate(Axis(1), &[a.view(), d.view()]).unwrap();
    println!("concatenate axis1:\n {:?}", e);
}
```

**输出：**

```shell
concatenate axis0:
 [[1, 2],
 [3, 4],
 [5, 6]], shape=[3, 2], strides=[2, 1], layout=Cc (0x5), const ndim=2
concatenate axis1:
 [[1, 2, 7, 8],
 [3, 4, 9, 10]], shape=[2, 4], strides=[1, 2], layout=Ff (0xa), const ndim=2
```

在实际应用中，当我们需要将不同时间段的传感器数据连接起来时，`concatenate`就派上用场了。

比如，一个机器人在不同时间段采集到的位置数据，我们可以使用`concatenate`将这些数据按时间顺序连接起来，以便分析机器人的运动轨迹。

### 4.2 stack：新增维度堆叠数组

`stack`函数用于在指定轴上堆叠数组，生成一个更高维度的新数组。它要求所有输入数组的形状必须一致，否则会导致错误。通过`stack`，我们可以轻松地将多个相同形状的数组合并成一个更高维度的数组，这在处理多个样本的相同特征数据时非常有用。

堆叠 (stack) 在概念上，完全等同于以下两步操作：

- “Reshape” (增加维度): 先把你要堆叠的每一个数组，在你指定的 Axis 位置上，增加一个大小为 1 的新维度。

- “Concatenate” (拼接): 然后，沿着那个刚刚新增的 Axis，把这些“升维”后的数组拼接起来。

```rust

use ndarray::array;

fn main() {
    let a = array![1, 2, 3];
    let b = array![4, 5, 6];

    // 在新轴（轴0）上堆叠数组
    let c = ndarray::stack(Axis(0), &[a.view(), b.view()]).unwrap();
    println!("stack axis0:\n {:?}", c);

    let a_2d = a.insert_axis(Axis(1)); // (3,1)
    let b_2d = b.insert_axis(Axis(1)); // (3,1)
    // 在轴1上堆叠数组, 先升维 （3，1）-> (3, 1, 1), 然后在轴1上拼接
    let d = ndarray::stack(Axis(1), &[a_2d.view(), b_2d.view()]).unwrap();
    println!("stack axis1:\n {:?}", d);
}
```

**输出：**

```shell
stack axis0:
 [[1, 2, 3],
 [4, 5, 6]], shape=[2, 3], strides=[3, 1], layout=Cc (0x5), const ndim=2
stack axis1:
 [[[1],
  [4]],

 [[2],
  [5]],

 [[3],
  [6]]], shape=[3, 2, 1], strides=[1, 3, 1], layout=Ff (0xa), const ndim=3
```

## 五、聚合与沿轴操作：数据降维与统计

在数据分析和科学计算中，聚合操作是对数据进行总结和概括的重要手段。`ndarray` 提供了丰富的聚合函数，如`sum`、`mean`等，这些函数可以快速计算数组的总和、平均值等统计量。

同时，通过指定轴参数，我们还可以沿特定的维度进行聚合操作，实现数据的降维与分析。

### 5.1 基础聚合：sum、mean 快速统计

`sum`和`mean`是最常用的聚合函数之一，它们可以直接对数组进行操作，返回一个标量结果，表示整个数组的总和或平均值。

```rust

use ndarray::array;

fn main() {
    let a = array![1, 2, 3, 4, 5];

    // 计算数组的总和
    let sum = a.sum();
    println!("sum:\n {:?}", sum);

    // 计算数组的平均值
    let mean = a.mean().unwrap();
    println!("mean:\n {:?}", mean); 
}
```

**输出：**

```shell
sum:
 15
mean:
 3.0
```

### 5.2 沿轴计算：按维度聚合数据

通过`axis`参数，我们可以指定聚合操作沿哪个轴进行，从而实现按维度聚合数据。在处理多维数据时非常有用，可以快速获取不同维度上的统计信息。

```rust

use ndarray::array;
use ndarray::Axis;

fn main() {
    let matrix = array![[1, 2, 3], [4, 5, 6]];

    // 计算每列的总和（轴0）
    let column_sums = matrix.sum_axis(Axis(0));
    println!("column_sums:\n {:?}", column_sums);

    // 计算每行的平均值（轴1）
    let row_means = matrix.mean_axis(Axis(1)).unwrap();
    println!("row_means:\n {:?}", row_means);
}
```

**输出：**

```shell
column_sums:
 [5, 7, 9], shape=[3], strides=[1], layout=CFcf (0xf), const ndim=1
row_means:
 [2, 5], shape=[2], strides=[1], layout=CFcf (0xf), const ndim=1
```

## 六、矩阵代数：ndarray-linalg 与线性代数基础

在人工智能的算法实现中，矩阵代数是不可或缺的一部分。`ndarray` 库本身提供了基本的矩阵乘法操作，而 `ndarray-linalg` 库则进一步扩展了其线性代数功能，为解决复杂的数学问题提供了强大的工具。

### 6.1 矩阵乘法：.dot () 与维度匹配

在 `ndarray` 中，使用`.dot()`方法执行矩阵乘法，它严格遵循线性代数中的维度规则。对于两个矩阵`A`和`B`，只有当`A`的列数等于`B`的行数时，矩阵乘法`A.dot(B)`才是有效的。

```rust

use ndarray::array;

fn main() {
    let a = array![[1, 2], [3, 4]];
    let b = array![[5, 6], [7, 8]];

    // 执行矩阵乘法
    let c = a.dot(&b);
    println!("dot:\n {:?}", c);
}
```

**输出：**

```shell
dot:
 [[19, 22],
 [43, 50]], shape=[2, 2], strides=[2, 1], layout=Cc (0x5), const ndim=2
```

### 6.2 ndarray-linalg 扩展：特征值、矩阵分解

`ndarray-linalg`库为 `ndarray` 提供了丰富的线性代数扩展，包括矩阵求逆、奇异值分解（SVD）、特征值计算等高级操作。

这些功能在人工智能中对于解决复杂的优化和估计问题非常关键。

Cargo.toml 配置：

```toml
[dependencies]
ndarray = { version = "0.17.1" }
ndarray-linalg = "0.18.0"
```

需要额外安装 `openblas`，地址：https://github.com/OpenMathLib/OpenBLAS 

```rust

use ndarray::array;
use ndarray_linalg::Inverse;

fn main() {
    let matrix = array![
        [1.0, 2.0],
        [3.0, 4.0]
    ];

    // 求矩阵的逆
    match matrix.inv() {
        Ok(inverse) => {
            println!("Inverse matrix: {:?}", inverse); 
        }
        Err(e) => {
            println!("Error: {}", e); 
        }
    }
}
```

**输出**

```shell
Inverse matrix:
 [[-2.0, 1.0],
 [1.5, -0.5]], shape=[2, 2], strides=[2, 1], layout=Cc (0x5), const ndim=2
```

## 总结：ndarray 助力智能系统高效计算

Rust ndarray 凭借元素级操作的简洁性、广播机制的智能维度适配、线性代数的高效支持，成为AI与具身智能开发的得力工具。

无论是传感器数据的实时处理，还是复杂算法的矩阵运算，`ndarray` 都能在保证内存安全的同时，提供接近原生的性能。
