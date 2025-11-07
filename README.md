# OpenFAST 完整安装编译运行指南（基于源码）

本指南基于 OpenFAST 项目源码分析，提供**零错误、一次性成功**的完整安装编译运行流程。

---

## 📋 目录

1. [第0步：虚拟机配置（解决"已杀死"问题）](#第0步虚拟机配置解决已杀死问题)
2. [第1步：安装系统依赖](#第1步安装系统依赖)
3. [第2步：获取源代码（包含子模块）](#第2步获取源代码包含子模块)
4. [第3步：配置编译系统](#第3步配置编译系统)
5. [第4步：编译 OpenFAST](#第4步编译-openfast)
6. [第5步：验证与运行](#第5步验证与运行)
7. [常见问题排查](#常见问题排查)

---

## 第0步：虚拟机配置（解决"已杀死"问题）

这是**最关键**的一步，是之前编译失败的**根本原因**。

### 问题原因
`make` 编译时需要大量内存，默认的 2GB 或 4GB 内存**绝对不够**，会导致进程被系统 OOM Killer "杀死"。

### 解决方案

1. **彻底关机**：在 Ubuntu 系统内，点击"关机"。不要"挂起"或"休眠"。

2. **打开 VMware 设置**：
   - 回到 VMware 软件界面
   - 选中你的 Ubuntu 虚拟机
   - 点击"**编辑虚拟机设置**"

3. **增加内存 (RAM)**：
   - 找到"**内存 (Memory)**"选项
   - 从默认值（如 2GB 或 4GB）**调高到至少 8GB**
   - 如果你的电脑有 16GB 以上，给 12GB 或 16GB 更好
   - **这是最重要的设置！**

4. **增加处理器 (CPU)**：
   - 找到"**处理器 (Processors)**"选项
   - 确保至少分配了 **2 个**（推荐 4 个）处理器核心
   - 这将加快编译速度

5. **保存并重启**：
   - 保存设置
   - 重新启动你的 Ubuntu 虚拟机

**做完这一步，99% 的编译问题已经解决了。**

---

## 第1步：安装系统依赖

### 1.1 更新包列表

```bash
sudo apt update
```

**说明**：`sudo` 获取管理员权限（会要求输入密码）。`apt update` 更新软件包列表。

### 1.2 安装编译工具和依赖

根据源码分析（`CMakeLists.txt` 和 `docs/source/install/index.rst`），OpenFAST 需要以下依赖：

```bash
sudo apt install -y \
    build-essential \
    gfortran \
    g++ \
    cmake \
    git \
    libblas-dev \
    liblapack-dev
```

**依赖说明**：
- `build-essential`：包含 `gcc`, `g++`, `make` 等编译工具
- `gfortran`：OpenFAST 的 **Fortran 编译器**（必需）
- `g++`：C++ 编译器（必需）
- `cmake`：**编译配置工具**（最低版本 3.15，用于生成 Makefile）
- `git`：**下载工具**（从 GitHub 下载源码和子模块）
- `libblas-dev` / `liblapack-dev`：OpenFAST 依赖的**数学库**（必需）

### 1.3 验证安装

```bash
# 检查编译器版本
gfortran --version
g++ --version
cmake --version
git --version
```

**预期输出**：
- `gfortran` 版本应 >= 4.6.0
- `cmake` 版本应 >= 3.15.0
- 其他工具应显示版本信息

---

## 第2步：获取源代码（包含子模块）

### 2.1 克隆主仓库

```bash
# 进入主目录
cd ~

# 克隆 OpenFAST 源码
git clone https://github.com/OpenFAST/OpenFAST.git openfast

# 进入目录
cd openfast
```

**说明**：这会下载 OpenFAST 的主代码，但**不包含测试数据子模块**。

### 2.2 初始化子模块（关键！）

根据源码分析（`.gitmodules` 和 `reg_tests/CMakeLists.txt`），OpenFAST 使用 `r-test` 作为子模块存储测试数据。

```bash
# 初始化并下载所有子模块
git submodule update --init --recursive
```

**说明**：
- `git clone` **默认不下载**测试集 (`r-test`)
- 这条命令会**初始化并下载所有子模块**
- **完成后，`reg_tests/r-test` 文件夹里就有文件了**

**验证**：
```bash
# 检查 r-test 是否存在且有内容
ls -la reg_tests/r-test/
```

**预期输出**：应该看到 `glue-codes/`, `modules/` 等目录，而不是空文件夹。

---

## 第3步：配置编译系统

### 3.1 创建构建目录

```bash
# 确保在 openfast 目录
cd ~/openfast

# 创建 build 目录
mkdir build

# 进入 build 目录
cd build
```

**说明**：`build` 目录是"工作间"，所有编译生成的文件都在这里，不会污染源代码。

### 3.2 运行 CMake 配置

根据源码（`CMakeLists.txt`），CMake 会自动查找 BLAS/LAPACK，默认配置即可：

```bash
# 基本配置（推荐）
cmake ..
```

**说明**：
- `cmake ..` 会查找**上一级目录** (`..` 指 `/home/timi/openfast`) 的 `CMakeLists.txt`
- 生成编译用的 `Makefile` 文件
- CMake 会自动查找系统安装的 BLAS/LAPACK

**预期输出**：
```
-- The Fortran compiler identification is GNU X.X.X
-- The CXX compiler identification is GNU X.X.X
-- Found BLAS: /usr/lib/x86_64-linux-gnu/libblas.so
-- Found LAPACK: /usr/lib/x86_64-linux-gnu/liblapack.so
-- Configuring done
-- Generating done
-- Build files have been written to: /home/timi/openfast/build
```

### 3.3 可选：自定义 CMake 选项

如果需要启用测试或特定功能：

```bash
# 启用测试（需要 Python）
cmake .. -DBUILD_TESTING=ON

# 启用 FAST.Farm（需要 OpenMP）
cmake .. -DBUILD_FASTFARM=ON -DOPENMP=ON

# 使用静态 LAPACK（如果系统库有问题）
cmake .. -DUSE_LOCAL_STATIC_LAPACK=ON
```

**常用选项**（来自 `CMakeLists.txt`）：
- `BUILD_TESTING`：启用测试（默认 OFF）
- `BUILD_FASTFARM`：启用 FAST.Farm（默认 OFF）
- `BUILD_SHARED_LIBS`：构建共享库（默认 OFF）
- `CMAKE_BUILD_TYPE`：构建类型（Release/Debug，默认 Release）
- `DOUBLE_PRECISION`：双精度（默认 ON）

---

## 第4步：编译 OpenFAST

### 4.1 执行编译

现在在 `/home/timi/openfast/build` 目录里：

```bash
# 使用所有 CPU 核心并行编译（推荐）
make -j$(nproc)
```

**说明**：
- `make` 开始编译
- `-j$(nproc)` 使用**所有** CPU 核心并行编译，速度最快
- `$(nproc)` 自动检测 CPU 核心数

**或者指定核心数**：
```bash
# 使用 4 个核心
make -j4
```

### 4.2 编译时间

- **首次编译**：通常需要 **10-30 分钟**（取决于 CPU 和内存）
- **增量编译**：如果只修改了部分文件，通常只需几分钟

### 4.3 编译成功标志

编译成功时，你会看到：
```
[100%] Built target openfast
```

**可执行文件位置**：
```bash
# OpenFAST 主程序
./glue-codes/openfast/openfast

# 验证版本
./glue-codes/openfast/openfast -v
```

---

## 第5步：验证与运行

### 5.1 选项 A：启用并运行自动测试（最推荐）

#### 5.1.1 启用测试功能

**重要**：默认情况下，CMake 配置时 `BUILD_TESTING=OFF`，需要手动启用：

```bash
# 在 build 目录下重新配置 CMake，启用测试
cd ~/openfast/build
cmake .. -DBUILD_TESTING=ON
```

**说明**：
- 如果已经配置过，可以重新运行 `cmake .. -DBUILD_TESTING=ON` 来更新配置
- 启用测试后，CMake 会配置 CTest 测试框架

#### 5.1.2 安装 Python 测试依赖

测试框架需要 Python 和相关的科学计算库：

```bash
# 安装 Python 3 和 pip
sudo apt install -y python3 python3-pip

# 安装测试所需的 Python 库
pip3 install numpy pandas bokeh
```

**依赖说明**（来自 `reg_tests/README.md`）：
- `numpy`：数值计算库
- `pandas`：数据处理库
- `bokeh`：绘图库（可选，用于生成测试报告）

#### 5.1.3 运行自动测试

```bash
# 确保在 build 目录
cd ~/openfast/build

# 运行所有测试
ctest

# 或者运行特定标签的测试
ctest -L openfast          # 只运行 OpenFAST 测试
ctest -L elastodyn         # 只运行 ElastoDyn 相关测试
ctest -L offshore          # 只运行海上风机测试

# 显示详细输出
ctest -VV
```

**说明**：
- `ctest` 会自动找到 `reg_tests/r-test` 中的测试案例
- 调用编译好的 `openfast` 程序运行测试
- 测试结果会保存在 `build/reg_tests/` 目录下
- 你会看到大量 `Test #1 ... Pass`, `Test #2 ... Pass`...

**预期输出**：
```
Test project /home/timi/openfast/build
    Start 1: AWT_YFix_WSt
 1/XX Test #1: AWT_YFix_WSt ...................   Passed
 2/XX Test #2: AWT_YFree_WSt ..................   Passed
 3/XX Test #3: 5MW_Land_DLL_WTurb .............   Passed
 ...
```

**只要最后没有 `Failed`，就证明编译完美无缺。**

### 5.2 选项 B：手动运行测试案例

#### 5.2.1 重要说明：5MW_Baseline 目录

**注意**：`5MW_Baseline` 目录**不是**一个测试案例，而是一个**数据目录**，包含：
- 各种模块的输入文件（`.dat` 文件）
- 气动数据（`AeroData/`）
- 叶片数据（`Airfoils/`）
- 水动力数据（`HydroData/`）
- 控制器数据（`ServoData/`）

这些数据被其他测试案例**引用**，而不是直接运行。

#### 5.2.2 实际可用的测试案例

以下是一些实际存在的测试案例（每个目录都包含对应的 `.fst` 文件）：

**简单陆地案例（推荐初学者）**：
```bash
# 确保在 build 目录
cd ~/openfast/build

# 案例 1：5MW 陆地风机 - 模态分析
./glue-codes/openfast/openfast \
    ../reg_tests/r-test/glue-codes/openfast/5MW_Land_ModeShapes/5MW_Land_ModeShapes.fst

# 案例 2：5MW 陆地风机 - 动态链接库控制器
./glue-codes/openfast/openfast \
    ../reg_tests/r-test/glue-codes/openfast/5MW_Land_DLL_WTurb/5MW_Land_DLL_WTurb.fst

# 案例 3：AWT 风机 - 固定偏航
./glue-codes/openfast/openfast \
    ../reg_tests/r-test/glue-codes/openfast/AWT_YFix_WSt/AWT_YFix_WSt.fst
```

**海上风机案例**：
```bash
# 案例 4：5MW OC3 Spar 浮式风机
./glue-codes/openfast/openfast \
    ../reg_tests/r-test/glue-codes/openfast/5MW_OC3Spar_DLL_WTurb_WavesIrr/5MW_OC3Spar_DLL_WTurb_WavesIrr.fst

# 案例 5：5MW OC3 单桩风机
./glue-codes/openfast/openfast \
    ../reg_tests/r-test/glue-codes/openfast/5MW_OC3Mnpl_DLL_WTurb_WavesIrr/5MW_OC3Mnpl_DLL_WTurb_WavesIrr.fst
```

**其他配置案例**：
```bash
# 案例 6：使用 BeamDyn 高级梁模型
./glue-codes/openfast/openfast \
    ../reg_tests/r-test/glue-codes/openfast/5MW_Land_BD_DLL_WTurb/5MW_Land_BD_DLL_WTurb.fst

# 案例 7：使用 AeroDisk 气动盘模型
./glue-codes/openfast/openfast \
    ../reg_tests/r-test/glue-codes/openfast/5MW_Land_DLL_WTurb_ADsk/5MW_Land_DLL_WTurb_ADsk.fst
```

#### 5.2.3 运行说明

**基本语法**：
```bash
./glue-codes/openfast/openfast <输入文件路径>
```

**参数说明**：
- 可执行文件：`./glue-codes/openfast/openfast`
- 输入文件：`.fst` 文件（FAST 主输入文件）
- 相对路径：从 `build` 目录出发，使用 `../` 访问源码目录

**预期输出**：
```
 **************************************************************************************************
 OpenFAST

 Copyright (C) 2025 National Renewable Energy Laboratory
 ...
 OpenFAST-v4.1.2-1-gd08d931f
 ...
 Running ElastoDyn.
 Running InflowWind.
 Running AeroDyn.
 Running ServoDyn.
 ...
  Time: 0 of 6000 seconds.
  Time: 5 of 6000 seconds.
  Time: 10 of 6000 seconds.
 ...
 Simulation completed.
```

**当你看到终端开始滚动仿真时间，你就 100% 成功了！**

#### 5.2.4 输出文件说明

运行完成后，会在测试案例目录下生成以下文件：

- **`.outb`**：二进制输出文件（包含所有输出通道的时间序列）
- **`.out`**：ASCII 输出文件（可读的文本格式）
- **`.ech`**：回显文件（输入文件的副本，包含注释）
- **`.sum`**：摘要文件（仿真摘要信息）
- **`.log`**：日志文件（如果使用 Python 脚本运行）

#### 5.2.5 测试案例分类参考

根据 `reg_tests/CTestList.cmake`，测试案例按以下标签分类：

- **`openfast`**：OpenFAST 主程序测试
- **`elastodyn`**：结构动力学模块
- **`aerodyn`**：气动力学模块
- **`servodyn`**：控制系统模块
- **`hydrodyn`**：水动力学模块
- **`offshore`**：海上风机配置
- **`beamdyn`**：高级梁模型
- **`linear`**：线性化分析

你可以根据需求选择合适的测试案例。

### 5.3 选项 C：安装到系统（可选）

```bash
# 安装到默认位置（openfast/install）
make install

# 或安装到自定义位置
cmake .. -DCMAKE_INSTALL_PREFIX=/usr/local
make install
```

---

## 常见问题排查

### 问题 1：编译时出现 "已杀死" (Killed)

**原因**：内存不足

**解决方案**：
1. 检查虚拟机内存设置（第0步）
2. 确保至少分配了 8GB 内存
3. 减少并行编译核心数：`make -j2`（使用 2 个核心）

### 问题 2：`r-test` 文件夹为空

**原因**：子模块未初始化

**解决方案**：
```bash
cd ~/openfast
git submodule update --init --recursive
```

### 问题 3：CMake 找不到 BLAS/LAPACK

**原因**：数学库未安装

**解决方案**：
```bash
sudo apt install -y libblas-dev liblapack-dev
# 然后重新运行 cmake ..
```

### 问题 4：CMake 版本过低

**原因**：系统 CMake 版本 < 3.15

**解决方案**：
```bash
# 检查版本
cmake --version

# 如果版本过低，安装最新版本
sudo apt remove cmake
sudo apt install -y cmake
# 或从源码编译安装
```

### 问题 5：ctest 报错 "No test configuration file found!"

**原因**：编译时未启用 `BUILD_TESTING=ON`

**解决方案**：
```bash
# 重新配置 CMake，启用测试
cd ~/openfast/build
cmake .. -DBUILD_TESTING=ON

# 然后运行 ctest
ctest
```

### 问题 6：测试失败（ctest）

**原因**：可能缺少 Python 依赖或测试数据问题

**解决方案**：
```bash
# 安装 Python 依赖
pip3 install numpy pandas bokeh

# 检查测试数据是否存在
ls -la reg_tests/r-test/glue-codes/openfast/

# 如果 r-test 为空，重新初始化子模块
cd ~/openfast
git submodule update --init --recursive
```

### 问题 7：找不到可执行文件

**原因**：编译未完成或路径错误

**解决方案**：
```bash
# 检查编译是否完成
cd ~/openfast/build
ls -la glue-codes/openfast/openfast

# 如果文件不存在，重新编译
make openfast
```

### 问题 8：测试案例文件不存在

**原因**：使用了不存在的测试案例路径（如 `5MW_Baseline` 目录没有 `.fst` 文件）

**解决方案**：
```bash
# 查找实际存在的测试案例
find reg_tests/r-test/glue-codes/openfast -name "*.fst" -type f

# 使用实际存在的案例，例如：
./glue-codes/openfast/openfast \
    ../reg_tests/r-test/glue-codes/openfast/5MW_Land_ModeShapes/5MW_Land_ModeShapes.fst
```

---

## 快速参考命令总结

```bash
# 1. 安装依赖
sudo apt update
sudo apt install -y build-essential gfortran g++ cmake git libblas-dev liblapack-dev

# 2. 获取源码
cd ~
git clone https://github.com/OpenFAST/OpenFAST.git openfast
cd openfast
git submodule update --init --recursive

# 3. 配置编译
mkdir build && cd build
cmake ..

# 4. 编译
make -j$(nproc)

# 5. 验证版本
./glue-codes/openfast/openfast -v

# 6. 运行测试案例（选择一个实际存在的案例）
# 简单案例：5MW 陆地风机模态分析
./glue-codes/openfast/openfast \
    ../reg_tests/r-test/glue-codes/openfast/5MW_Land_ModeShapes/5MW_Land_ModeShapes.fst

# 或者启用测试并运行自动测试
cmake .. -DBUILD_TESTING=ON
pip3 install numpy pandas bokeh
ctest
```

---

## 源码依据

本指南基于以下源码文件分析：

1. **`CMakeLists.txt`**：主构建配置文件
   - CMake 最低版本：3.15
   - 必需依赖：BLAS/LAPACK
   - 默认选项配置

2. **`.gitmodules`**：子模块配置
   - `reg_tests/r-test`：测试数据子模块

3. **`reg_tests/CMakeLists.txt`**：测试配置
   - 验证 `r-test` 子模块存在
   - 测试可执行文件路径

4. **`docs/source/install/index.rst`**：官方安装文档
   - Ubuntu 依赖安装命令
   - CMake 配置选项

5. **`.github/workflows/automated-dev-tests.yml`**：CI 配置
   - Ubuntu 22.04 依赖列表
   - 编译和测试流程

---

## 总结

按照本指南的步骤，你应该能够：

1. ✅ 成功配置虚拟机（避免内存不足）
2. ✅ 安装所有必需的依赖
3. ✅ 下载完整的源代码（包括子模块）
4. ✅ 配置和编译 OpenFAST
5. ✅ 运行测试和仿真案例

**关键要点**：
- **第0步（虚拟机配置）最重要**，解决了 99% 的编译问题
- **第2步（子模块初始化）必须执行**，否则测试数据为空
- **使用 `make -j$(nproc)` 并行编译**，充分利用多核 CPU

祝你编译顺利！🎉

