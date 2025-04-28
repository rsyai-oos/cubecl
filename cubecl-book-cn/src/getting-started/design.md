# 设计

CubeCL 的设计围绕着一个核心概念——立方体！更具体地说，它是基于长方体的，因为并非所有轴的大小都相同。由于所有的计算 API 都需要映射到硬件上（这些硬件可以通过三维表示访问），我们的拓扑结构可以轻松映射到其他 API 的概念。

<div align="center">

### CubeCL - Topology

<img src="./cubecl.drawio.svg" width="100%"/>
<br />
</div>
<br />

> 一个立方体由单元组成，因此一个 3x3x3 的立方体有 27 个单元，可以通过它们在 x、y 和 z 轴上的位置访问。类似地，一个超立方体由立方体组成，就像一个立方体由单元组成一样。超立方体中的每个立方体都可以通过其相对于超立方体在 x、y 和 z 轴上的位置访问。因此，一个 3x3x3 的超立方体将包含 27 个立方体。在这个例子中，总的工作单元数为 27 x 27 = 729.

### 拓扑等价性

由于所有拓扑变量在内核入口点处都是常量，我们选择使用 Rust 的大写字母常量语法。在创建内核时，我们并不总是关心单元在立方体中沿每个轴的相对位置，而通常只关心它的总体位置。因此，每种变量也有其自己的轴独立变量，这种特性在其他语言中通常不存在，除了 WebGPU 中的 `local_invocation_index`。

<br />

| CubeCL         | CUDA        | WebGPU                 |
| -------------- | ----------- | ---------------------- |
| CUBE_COUNT     | N/A         | N/A                    |
| CUBE_COUNT_X   | gridDim.x   | num_workgroups.x       |
| CUBE_COUNT_Y   | gridDim.y   | num_workgroups.y       |
| CUBE_COUNT_Z   | gridDim.z   | num_workgroups.z       |
| CUBE_POS       | N/A         | N/A                    |
| CUBE_POS_X     | blockIdx.x  | workgroup.x            |
| CUBE_POS_Y     | blockIdx.y  | workgroup.y            |
| CUBE_POS_Z     | blockIdx.z  | workgroup.z            |
| CUBE_DIM       | N/A         | N/A                    |
| CUBE_DIM_X     | blockDim.x  | workgroup_size.x       |
| CUBE_DIM_Y     | blockDim.y  | workgroup_size.y       |
| CUBE_DIM_Z     | blockDim.z  | workgroup_size.z       |
| UNIT_POS       | N/A         | local_invocation_index |
| UNIT_POS_X     | threadIdx.x | local_invocation_id.x  |
| UNIT_POS_Y     | threadIdx.y | local_invocation_id.y  |
| UNIT_POS_Z     | threadIdx.z | local_invocation_id.z  |
| PLANE_DIM      | warpSize    | subgroup_size          |
| ABSOLUTE_POS   | N/A         | N/A                    |
| ABSOLUTE_POS_X | N/A         | global_id.x            |
| ABSOLUTE_POS_Y | N/A         | global_id.y            |
| ABSOLUTE_POS_Z | N/A         | global_id.z            |

</details>
