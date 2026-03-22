# rnnoise CMake 可配置选项

## 项目自定义选项

| 选项                       | 默认值 | 说明                                                                                      |
| -------------------------- | ------ | ----------------------------------------------------------------------------------------- |
| `USE_EXTERNAL_MODEL`       | `OFF`  | 是否启用外部模型加载模式。`ON` 时不编译内置 `rnnoise_data.c`，并定义 `USE_WEIGHTS_FILE`。 |
| `BUILD_EXAMPLES`           | `ON`   | 是否构建并安装 `rnnoise_demo` 示例程序。                                                  |
| `RNNOISE_ENABLE_X86_RTCD`  | `OFF`  | 是否启用 x86 RTCD 路径与相关源码（`src/x86/*`）。                                         |
| `RNNOISE_AUTO_FETCH_MODEL` | `ON`   | 当模型文件缺失时，是否自动读取 `model_version` 并下载、校验、解压模型归档。               |

## 常用 CMake 通用变量

| 变量                       | 默认值/行为                        | 说明                                                   |
| -------------------------- | ---------------------------------- | ------------------------------------------------------ |
| `CMAKE_BUILD_TYPE`         | 若未指定，脚本中默认设为 `Release` | 单配置生成器的构建类型（如 Ninja/Unix Makefiles）。    |
| `CMAKE_INSTALL_PREFIX`     | CMake 默认安装前缀                 | 安装根目录。                                           |
| `BUILD_SHARED_LIBS`        | CMake 默认 `OFF`                   | 控制 `add_library(rnnoise ...)` 生成共享库还是静态库。 |
| `CMAKE_C_COMPILER`         | 自动探测                           | 指定 C 编译器。                                        |
| `CMAKE_C_FLAGS`            | 编译器默认                         | 追加全局 C 编译参数。                                  |
| `CMAKE_INSTALL_LIBDIR`     | 由 `GNUInstallDirs` 给出           | 库安装目录（如 `lib` / `lib64`）。                     |
| `CMAKE_INSTALL_INCLUDEDIR` | 由 `GNUInstallDirs` 给出           | 头文件安装目录（通常 `include`）。                     |
| `CMAKE_INSTALL_BINDIR`     | 由 `GNUInstallDirs` 给出           | 可执行文件安装目录（通常 `bin`）。                     |

## 快速示例

```powershell
cmake -S . -B build `
  -DCMAKE_BUILD_TYPE=Release `
  -DCMAKE_INSTALL_PREFIX="C:/rnnoise/install" `
  -DUSE_EXTERNAL_MODEL=OFF `
  -DBUILD_EXAMPLES=ON `
  -DRNNOISE_ENABLE_X86_RTCD=OFF `
  -DRNNOISE_AUTO_FETCH_MODEL=ON
```
