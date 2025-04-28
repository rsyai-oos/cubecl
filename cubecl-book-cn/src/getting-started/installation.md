# 安装
安装 CubeCL 非常简单。它作为一个 Rust crate 提供，您可以通过更新您的 `Cargo.toml` 文件将其添加到项目中：

```toml
[dependencies]
cubecl = { version = "{version}", features = ["cuda", "wgpu"] }
```

更具挑战性的部分是确保您拥有运行所选运行时所需的必要驱动程序。

对于 Linux 和 Windows 上的 `wgpu`，需要 Vulkan 驱动程序。这些驱动程序通常包含在默认的操作系统安装中。然而，在某些配置下（例如 Windows 的 Linux 子系统 (WSL)），如果缺少驱动程序，则可能需要手动安装。

对于 `cuda`，只需在您的设备上安装 CUDA 驱动程序即可。您可以按照 [NVIDIA 官方网站](https://developer.nvidia.com/cuda-downloads) 提供的安装说明进行操作。