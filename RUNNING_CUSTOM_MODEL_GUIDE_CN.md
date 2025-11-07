# OpenFAST 运行自定义模型完整指南

本指南详细介绍如何准备和运行自己的 OpenFAST 模型，从零开始到成功运行仿真的完整流程。

---

## 📋 目录

1. [准备工作](#准备工作)
2. [理解输入文件结构](#理解输入文件结构)
3. [从现有案例开始](#从现有案例开始)
4. [创建自定义模型](#创建自定义模型)
5. [运行仿真](#运行仿真)
6. [验证和调试](#验证和调试)
7. [常见问题解决](#常见问题解决)
8. [最佳实践](#最佳实践)

---

## 准备工作

### 1. 确认 OpenFAST 已正确编译

```bash
# 检查可执行文件是否存在
cd ~/openfast/build
ls -la glue-codes/openfast/openfast

# 检查版本信息
./glue-codes/openfast/openfast -h
```

### 2. 准备模型数据

运行 OpenFAST 需要以下数据：

**必需数据**：
- 风机结构参数（叶片、塔架、机舱等）
- 初始条件（初始转速、变桨角等）

**可选数据**（取决于仿真类型）：
- 翼型数据（气动分析）
- 控制器参数（控制系统仿真）
- 风场数据（湍流风仿真）
- 波浪数据（海上风机仿真）

### 3. 创建工作目录

```bash
# 创建您的工作目录
mkdir -p ~/openfast/my_models/my_turbine
cd ~/openfast/my_models/my_turbine
```

---

## 理解输入文件结构

### OpenFAST 主输入文件（.fst）

主输入文件是 OpenFAST 的入口点，它指定了：
- 仿真控制参数
- 启用的模块
- 各模块的输入文件路径
- 输出设置

**文件结构**：

```
------- OpenFAST INPUT FILE -----------------------------------------------
[标题行1 - 用户自定义]
[标题行2 - 用户自定义]
---------------------- SIMULATION CONTROL --------------------------------------
[仿真控制参数]
---------------------- FEATURE SWITCHES AND FLAGS ------------------------------
[模块开关]
---------------------- ENVIRONMENTAL CONDITIONS --------------------------------
[环境条件]
---------------------- INPUT FILES ---------------------------------------------
[各模块输入文件路径]
---------------------- OUTPUT --------------------------------------------------
[输出设置]
---------------------- LINEARIZATION -------------------------------------------
[线性化设置]
---------------------- VISUALIZATION ------------------------------------------
[可视化设置]
```

### 必需的模块输入文件

根据启用的模块，需要准备相应的输入文件：

| 模块 | 开关 | 输入文件 | 说明 |
|------|------|---------|------|
| ElastoDyn | CompElast=1 | `EDFile` | 结构动力学模块 |
| BeamDyn | CompElast=2 | `BDBldFile(1-3)` | 高级叶片模型 |
| InflowWind | CompInflow=1 | `InflowFile` | 风场模块 |
| AeroDyn | CompAero=2 | `AeroFile` | 气动力学模块 |
| ServoDyn | CompServo=1 | `ServoFile` | 控制系统模块 |
| HydroDyn | CompHydro=1 | `HydroFile` | 水动力学模块 |
| SubDyn | CompSub=1 | `SubFile` | 子结构模块 |
| MoorDyn | CompMooring=3 | `MooringFile` | 系泊系统模块 |

### 文件路径规则

- **相对路径**：相对于当前 `.fst` 文件所在目录
- **绝对路径**：完整路径（推荐用于避免路径问题）
- **路径分隔符**：Linux/Mac 使用 `/`，Windows 使用 `\` 或 `/`

---

## 从现有案例开始

### 方法 1：复制并修改现有案例（推荐）

这是最简单、最稳定的方法：

```bash
# 1. 选择一个合适的测试案例作为模板
# 例如：简单的陆地风机案例
cp -r ~/openfast/reg_tests/r-test/glue-codes/openfast/AWT_YFix_WSt \
      ~/openfast/my_models/my_turbine

cd ~/openfast/my_models/my_turbine

# 2. 重命名主输入文件
mv AWT_YFix_WSt.fst my_turbine.fst

# 3. 查看文件列表
ls -la
```

**文件结构示例**：
```
my_turbine/
├── my_turbine.fst                    # 主输入文件
├── AWT_YFix_WSt_ElastoDyn.dat        # ElastoDyn 输入文件
├── AWT_YFix_WSt_InflowWind.dat       # InflowWind 输入文件
├── AWT_YFix_WSt_AeroDyn.dat          # AeroDyn 输入文件
├── AWT_YFix_WSt_ServoDyn.dat         # ServoDyn 输入文件
└── [其他数据文件]
```

### 方法 2：使用最小示例

OpenFAST 提供了一个最小示例，适合初学者：

```bash
# 复制最小示例
cp -r ~/openfast/reg_tests/r-test/glue-codes/openfast/MinimalExample \
      ~/openfast/my_models/my_turbine_minimal

cd ~/openfast/my_models/my_turbine_minimal

# 查看文件
ls -la
```

**最小示例包含**：
- `Main.fst`：主输入文件
- `ElastoDyn.dat`：结构动力学输入
- `ElastoDyn_Blade.dat`：叶片数据
- `ElastoDyn_Tower.dat`：塔架数据

### 方法 3：从 IEA Wind Task 37 模型开始

IEA Wind Task 37 提供了标准化的参考风机模型：

```bash
# 克隆 IEA Wind Task 37 仓库（如果尚未克隆）
git clone https://github.com/IEAWindTask37/IEA-3.4-130-RWT.git
cd IEA-3.4-130-RWT

# 查看模型文件
ls -la
```

---

## 创建自定义模型

### 步骤 1：修改主输入文件（.fst）

#### 1.1 更新文件路径

编辑 `.fst` 文件，更新输入文件路径：

```bash
# 使用文本编辑器打开
nano my_turbine.fst
# 或
vim my_turbine.fst
```

**修改示例**：

```fortran
---------------------- INPUT FILES ---------------------------------------------
"my_turbine_ElastoDyn.dat"    EDFile          - Name of file containing ElastoDyn input parameters (quoted string)
"unused"                      BDBldFile(1)   - Name of file containing BeamDyn input parameters for blade 1 (quoted string)
"my_turbine_InflowWind.dat"   InflowFile      - Name of file containing inflow wind input parameters (quoted string)
"my_turbine_AeroDyn.dat"      AeroFile        - Name of file containing aerodynamic input parameters (quoted string)
"my_turbine_ServoDyn.dat"     ServoFile       - Name of file containing control and electrical-drive input parameters (quoted string)
```

#### 1.2 设置仿真参数

**关键参数**：

```fortran
---------------------- SIMULATION CONTROL --------------------------------------
False         Echo            - Echo input data to <RootName>.ech (flag)
"FATAL"       AbortLevel      - Error level when simulation should abort (string)
        30    TMax            - Total run time (s)          # 仿真总时长
     0.01     DT              - Recommended module time step (s)  # 时间步长
```

**时间步长选择指南**：
- 简单结构：0.01-0.05 秒
- 复杂结构（BeamDyn）：0.001-0.005 秒
- 浮式风机：0.005-0.01 秒

#### 1.3 配置模块开关

根据仿真需求启用/禁用模块：

```fortran
---------------------- FEATURE SWITCHES AND FLAGS ------------------------------
          1   CompElast       - Compute structural dynamics (switch) {1=ElastoDyn; 2=ElastoDyn + BeamDyn}
          1   CompInflow      - Compute inflow wind velocities (switch) {0=still air; 1=InflowWind}
          2   CompAero        - Compute aerodynamic loads (switch) {0=None; 1=AeroDisk; 2=AeroDyn}
          1   CompServo       - Compute control and electrical-drive dynamics (switch) {0=None; 1=ServoDyn}
          0   CompHydro       - Compute hydrodynamic loads (switch) {0=None; 1=HydroDyn}
```

**常见配置组合**：

| 仿真类型 | CompElast | CompInflow | CompAero | CompServo | CompHydro |
|---------|-----------|------------|----------|-----------|-----------|
| 结构模态分析 | 1 | 0 | 0 | 0 | 0 |
| 简单气动仿真 | 1 | 1 | 2 | 0 | 0 |
| 完整风机仿真 | 1 | 1 | 2 | 1 | 0 |
| 海上风机仿真 | 1 | 1 | 2 | 1 | 1 |

### 步骤 2：准备 ElastoDyn 输入文件

#### 2.1 复制模板文件

```bash
# 从现有案例复制
cp AWT_YFix_WSt_ElastoDyn.dat my_turbine_ElastoDyn.dat
```

#### 2.2 修改关键参数

**必须修改的参数**：

1. **叶片数据文件路径**：
```fortran
"my_turbine_Blade.dat"  BldFile(1)  - Name of file containing properties for blade 1 (quoted string)
"my_turbine_Blade.dat"  BldFile(2)  - Name of file containing properties for blade 2 (quoted string)
"my_turbine_Blade.dat"  BldFile(3)  - Name of file containing properties for blade 3 (quoted string)
```

2. **塔架数据文件路径**：
```fortran
"my_turbine_Tower.dat"  TwrFile     - Name of file containing tower properties (quoted string)
```

3. **几何参数**：
```fortran
        63   TipRad         - The distance from the rotor apex to the blade tip (m)
         1.5 HubRad         - The distance from the rotor apex to the blade root (m)
        90   HubHt          - The distance from the ground [onshore] or MSL [offshore] to the rotor apex (m)
```

4. **初始条件**：
```fortran
        0.0 RotSpeed        - Initial or fixed rotor speed (rpm) [used only when GenDOF or DrTrDOF is False]
        0.0 NacYaw          - Initial or fixed nacelle-yaw angle (deg)
        0.0 TTDspFA         - Initial fore-aft tower-top displacement (m)
        0.0 TTDspSS         - Initial side-to-side tower-top displacement (m)
```

#### 2.3 准备叶片数据文件

```bash
# 复制模板
cp AWT_YFix_WSt_Blade.dat my_turbine_Blade.dat
```

**叶片文件结构**：
```fortran
------- ElastoDyn Blade Input File -------------------------------------------
[标题行]
---------------------- BLADE PARAMETERS --------------------------------------
[叶片参数]
---------------------- BLADE ADJUSTMENT FACTORS ------------------------------
[调整因子]
---------------------- DISTRIBUTED BLADE PROPERTIES --------------------------
[分布式属性表格]
```

**关键参数**：
- `BlFract`：叶片径向位置（0=根部，1=叶尖）
- `PitchAxis`：变桨轴位置
- `StrcTwst`：结构扭角（度）
- `BMassDen`：单位长度质量（kg/m）
- `FlpStff`：挥舞刚度（N·m²）
- `EdgStff`：摆振刚度（N·m²）

#### 2.4 准备塔架数据文件

```bash
# 复制模板
cp AWT_YFix_WSt_Tower.dat my_turbine_Tower.dat
```

**塔架文件结构**：
```fortran
------- ElastoDyn Tower Input File -------------------------------------------
[标题行]
---------------------- TOWER PARAMETERS ----------------------------------------
[塔架参数]
---------------------- DISTRIBUTED TOWER PROPERTIES ---------------------------
[分布式属性表格]
```

**关键参数**：
- `HtFract`：塔架高度位置（0=底部，1=顶部）
- `TMassDen`：单位长度质量（kg/m）
- `TwFAStif`：前后方向弯曲刚度（N·m²）
- `TwSSStif`：侧向弯曲刚度（N·m²）

### 步骤 3：准备 InflowWind 输入文件

#### 3.1 复制模板文件

```bash
cp AWT_YFix_WSt_InflowWind.dat my_turbine_InflowWind.dat
```

#### 3.2 选择风场类型

**常见风场类型**：

1. **均匀风（Uniform）**：
```fortran
---------------------- WIND FILE PARAMETERS ------------------------------------
          1   WindType       - Wind file type (switch) {0=none; 1=uniform wind file; 2=steady wind; 3=uniform wind file with time-varying direction; 4=HAWC format; 5=User-defined from Simulink/LabVIEW; 6=User-defined from an external source; 7=Bladed-style binary file; 8=Bladed-style ASCII file; 9=InflowWind binary file; 10=InflowWind ASCII file; 11=InflowWind binary file with tower data; 12=InflowWind ASCII file with tower data; 13=InflowWind binary file with tower data and coherent turbulence; 14=InflowWind ASCII file with tower data and coherent turbulence; 15=InflowWind binary file with tower data and coherent turbulence and coherent wind direction; 16=InflowWind ASCII file with tower data and coherent turbulence and coherent wind direction}
"unused"      FilenameRoot   - Root name of the full-field wind file(s) (quoted string) [used only for WindType=1,3,9-16]
```

2. **稳态风（Steady）**：
```fortran
          2   WindType       - Wind file type (switch)
        10.0  HWindSp        - Horizontal mean wind speed (m/s)
        0.0   RefHt           - Reference height for horizontal wind speed (m)
        0.0   PLexp           - Power law exponent (-) [used only for WindType=2]
```

3. **TurbSim 风场**：
```fortran
          9   WindType       - Wind file type (switch)
"wind_file"   FilenameRoot   - Root name of the full-field wind file(s) (quoted string)
```

#### 3.3 设置参考高度和轮毂高度

```fortran
        90.0  RefHt           - Reference height for horizontal wind speed (m) [used only for WindType=2]
```

**重要**：`RefHt` 应该与 ElastoDyn 中的 `HubHt` 一致。

### 步骤 4：准备 AeroDyn 输入文件

#### 4.1 复制模板文件

```bash
cp AWT_YFix_WSt_AeroDyn.dat my_turbine_AeroDyn.dat
```

#### 4.2 设置翼型数据文件

```fortran
---------------------- AIRFOIL INFORMATION ------------------------------------
          2   NumAFfiles     - Number of airfoil files (-)
"airfoil_1.dat" AFNames(1)   - Airfoil file names (NumAFfiles lines) (quoted string)
"airfoil_2.dat" AFNames(2)   - Airfoil file names (NumAFfiles lines) (quoted string)
```

#### 4.3 设置叶片气动数据文件

```fortran
---------------------- BLADE PARAMETERS ----------------------------------------
          2   NumBlades      - Number of blades (-)
          3   NumBlNds       - Number of blade nodes used in the analysis (-)
"my_turbine_AeroDyn_Blade.dat" BldFile(1)  - Name of file containing properties for blade 1 (quoted string)
```

#### 4.4 准备翼型数据文件

翼型文件包含攻角-升力/阻力系数数据：

```fortran
------- AeroDyn Airfoil File -------------------------------------------
[标题行]
---------------------- AIRFOIL COEFFICIENTS ------------------------------------
[攻角] [升力系数] [阻力系数] [力矩系数]
```

**示例**：
```fortran
------- AeroDyn Airfoil File -------------------------------------------
NACA 64-618 Airfoil
---------------------- AIRFOIL COEFFICIENTS ------------------------------------
-180.0  -0.5  0.5  0.0
-170.0  -0.3  0.3  0.0
...
```

#### 4.5 准备叶片气动数据文件

```bash
cp AWT_YFix_WSt_AeroDyn_Blade.dat my_turbine_AeroDyn_Blade.dat
```

**关键参数**：
- `BlSpn`：叶片径向位置（m）
- `BlCrvAC`：变桨轴位置
- `BlSwpAC`：扫掠位置
- `BlCrvAng`：预弯角（度）
- `BlTwist`：扭角（度）
- `BlChord`：弦长（m）
- `BlAFID`：翼型ID

### 步骤 5：准备 ServoDyn 输入文件（可选）

#### 5.1 复制模板文件

```bash
cp AWT_YFix_WSt_ServoDyn.dat my_turbine_ServoDyn.dat
```

#### 5.2 选择控制模式

**常见控制模式**：

1. **无控制**：
```fortran
          0   PCMode         - Pitch control mode (switch) {0=none; 1=user-defined from Simulink/LabVIEW; 2=user-defined from DLL; 3=user-defined from Bladed-style DLL; 4=user-defined from external source}
          0   VSContrl       - Variable-speed control mode (switch) {0:none; 1:pitch-to-feather; 2:user-defined from Simulink/LabVIEW; 3:user-defined from DLL; 4:user-defined from Bladed-style DLL; 5:user-defined from external source}
```

2. **简单内置控制器**：
```fortran
          1   PCMode         - Pitch control mode (switch)
          1   VSContrl       - Variable-speed control mode (switch)
```

3. **DLL 控制器**：
```fortran
          3   PCMode         - Pitch control mode (switch)
          3   VSContrl       - Variable-speed control mode (switch)
"controller.dll" DLL_FileName - Name/location of the dynamic library {.dll [Windows] or .so [Linux]} (quoted string)
```

---

## 运行仿真

### 基本运行命令

```bash
# 确保在正确的目录
cd ~/openfast/my_models/my_turbine

# 运行仿真
~/openfast/build/glue-codes/openfast/openfast my_turbine.fst
```

### 使用相对路径

```bash
# 从 build 目录运行
cd ~/openfast/build
./glue-codes/openfast/openfast ../my_models/my_turbine/my_turbine.fst
```

### 检查运行状态

运行时会显示进度信息：

```
 Running OpenFAST (v3.2.0, 2023-01-01)
 
 Computing: 0.00 of 30.00 seconds (0.00%)
 Computing: 5.00 of 30.00 seconds (16.67%)
 Computing: 10.00 of 30.00 seconds (33.33%)
 ...
```

### 成功运行的标志

1. **终端输出**：
   - 显示 "OpenFAST completed successfully"
   - 没有错误信息

2. **生成的文件**：
   ```bash
   ls -la
   ```
   
   应该看到：
   - `my_turbine.out` 或 `my_turbine.outb`：输出文件
   - `my_turbine.sum`：摘要文件（如果 `SumPrint=True`）
   - `my_turbine.ech`：回显文件（如果 `Echo=True`）

---

## 验证和调试

### 1. 检查输出文件

#### 读取输出文件

```python
#!/usr/bin/env python3
"""检查 OpenFAST 输出文件"""

import sys
sys.path.append('/home/timi/openfast/reg_tests/lib')
from fast_io import load_output
import numpy as np

# 读取输出文件
data, info, _ = load_output('my_turbine.outb')

# 显示基本信息
print(f"仿真时长: {data[-1, 0]:.2f} 秒")
print(f"时间步数: {len(data)}")
print(f"输出通道数: {len(info['attribute_names'])}")

# 检查是否有 NaN 或异常值
for i, name in enumerate(info['attribute_names']):
    values = data[:, i+1]
    if np.any(np.isnan(values)):
        print(f"警告: 通道 '{name}' 包含 NaN 值")
    if np.any(np.abs(values) > 1e10):
        print(f"警告: 通道 '{name}' 包含异常大的值")
```

### 2. 可视化结果

```python
#!/usr/bin/env python3
"""可视化 OpenFAST 输出"""

import sys
sys.path.append('/home/timi/openfast/reg_tests/lib')
from fast_io import load_output
import matplotlib.pyplot as plt

# 读取数据
data, info, _ = load_output('my_turbine.outb')
time = data[:, 0]

# 绘制关键通道
channels = ['RotSpeed', 'GenPwr', 'BlPitch1', 'TTDspFA']
fig, axes = plt.subplots(len(channels), 1, figsize=(10, 8), sharex=True)

for i, channel in enumerate(channels):
    if channel in info['attribute_names']:
        idx = info['attribute_names'].index(channel)
        unit = info['attribute_units'][idx]
        values = data[:, idx+1]
        
        axes[i].plot(time, values)
        axes[i].set_ylabel(f'{channel}\n({unit})')
        axes[i].grid(True)
        axes[i].set_title(channel)

axes[-1].set_xlabel('Time (s)')
plt.tight_layout()
plt.savefig('results.png', dpi=150)
plt.show()
```

### 3. 检查摘要文件

```bash
# 查看摘要文件
cat my_turbine.sum
```

摘要文件包含：
- 模型参数摘要
- 质量分布
- 输出通道列表
- 模块配置信息

### 4. 常见验证检查

#### 检查 1：结构合理性

```python
# 检查位移是否合理
if 'TTDspFA' in info['attribute_names']:
    idx = info['attribute_names'].index('TTDspFA')
    max_disp = np.max(np.abs(data[:, idx+1]))
    print(f"最大塔顶前后位移: {max_disp:.3f} m")
    if max_disp > 10:  # 假设塔高约 90m
        print("警告: 位移过大，可能存在问题")
```

#### 检查 2：转速稳定性

```python
# 检查转速是否稳定
if 'RotSpeed' in info['attribute_names']:
    idx = info['attribute_names'].index('RotSpeed')
    rot_speed = data[:, idx+1]
    mean_speed = np.mean(rot_speed[-100:])  # 最后 100 个时间步的平均值
    std_speed = np.std(rot_speed[-100:])
    print(f"平均转速: {mean_speed:.2f} rpm")
    print(f"转速标准差: {std_speed:.2f} rpm")
    if std_speed / mean_speed > 0.1:
        print("警告: 转速波动较大")
```

#### 检查 3：功率输出

```python
# 检查功率输出
if 'GenPwr' in info['attribute_names']:
    idx = info['attribute_names'].index('GenPwr')
    power = data[:, idx+1]
    max_power = np.max(power)
    mean_power = np.mean(power[-100:])
    print(f"最大功率: {max_power:.2f} kW")
    print(f"平均功率: {mean_power:.2f} kW")
```

---

## 常见问题解决

### 问题 1：文件未找到

**错误信息**：
```
The input file, "xxx.dat", was not found.
```

**解决方案**：

1. **检查文件路径**：
   ```bash
   # 列出所有文件
   ls -la
   
   # 检查文件是否存在
   ls -la my_turbine_ElastoDyn.dat
   ```

2. **使用绝对路径**：
   ```fortran
   "/home/timi/openfast/my_models/my_turbine/my_turbine_ElastoDyn.dat"  EDFile
   ```

3. **检查路径分隔符**：
   - Linux/Mac：使用 `/`
   - Windows：使用 `\` 或 `/`

### 问题 2：输入文件格式错误

**错误信息**：
```
Invalid input in file "xxx.dat" while trying to read VAR
```

**解决方案**：

1. **检查 OpenFAST 版本**：
   ```bash
   ./glue-codes/openfast/openfast -h
   ```

2. **查看 API 变更文档**：
   - 检查 `docs/source/user/api_change.rst`
   - 或查看 GitHub 上的变更日志

3. **使用最新版本的测试案例**：
   ```bash
   # 从 r-test 仓库获取最新案例
   git pull origin main
   ```

4. **检查回显文件**：
   ```bash
   # 设置 Echo=True，查看回显文件
   cat my_turbine.ech
   ```

### 问题 3：仿真崩溃或产生 NaN

**错误信息**：
```
FAST encountered an error at simulation time T
```

**解决方案**：

1. **简化模型**：
   - 关闭气动：`CompAero=0`
   - 关闭控制：`CompServo=0`
   - 使用稳态风：`WindType=2`

2. **检查初始条件**：
   ```fortran
   # 确保初始条件合理
   RotSpeed = 0.0  # 如果从静止开始
   BlPitch1 = 0.0  # 初始变桨角
   ```

3. **减小时间步长**：
   ```fortran
   0.001   DT  # 从 0.01 减小到 0.001
   ```

4. **检查结构参数**：
   - 质量是否合理
   - 刚度是否合理
   - 阻尼是否设置

5. **添加阻尼**（用于调试）：
   ```fortran
   # 在 ElastoDyn 中
   1000   BldDamp(1)  # 叶片阻尼
   1000   TwrDamp(1)  # 塔架阻尼
   ```

### 问题 4：模块初始化失败

**错误信息**：
```
FAST encountered an error during module initialization.
```

**解决方案**：

1. **检查模块依赖**：
   - AeroDyn 需要 InflowWind
   - ServoDyn 需要 ElastoDyn
   - 确保所有必需的模块都已启用

2. **检查输入文件完整性**：
   ```bash
   # 检查所有必需的文件是否存在
   ls -la my_turbine_*.dat
   ```

3. **查看详细错误信息**：
   - 错误信息会显示具体是哪个模块失败
   - 检查该模块的输入文件

### 问题 5：路径问题

**错误信息**：
```
The blade file, "xxx.dat", was not found.
```

**解决方案**：

1. **统一使用相对路径**：
   ```fortran
   # 所有文件在同一目录
   "my_turbine_Blade.dat"  BldFile(1)
   ```

2. **或统一使用绝对路径**：
   ```fortran
   "/home/timi/openfast/my_models/my_turbine/my_turbine_Blade.dat"  BldFile(1)
   ```

3. **检查工作目录**：
   ```bash
   # 确保在正确的目录运行
   pwd
   cd ~/openfast/my_models/my_turbine
   ```

---

## 最佳实践

### 1. 文件组织

**推荐的目录结构**：

```
my_models/
├── my_turbine/
│   ├── my_turbine.fst                    # 主输入文件
│   ├── my_turbine_ElastoDyn.dat          # ElastoDyn 输入
│   ├── my_turbine_Blade.dat              # 叶片结构数据
│   ├── my_turbine_Tower.dat              # 塔架结构数据
│   ├── my_turbine_InflowWind.dat         # 风场输入
│   ├── my_turbine_AeroDyn.dat            # AeroDyn 输入
│   ├── my_turbine_AeroDyn_Blade.dat      # 叶片气动数据
│   ├── airfoil_1.dat                     # 翼型数据
│   ├── airfoil_2.dat                     # 翼型数据
│   ├── my_turbine_ServoDyn.dat           # 控制器输入
│   ├── wind/                              # 风场数据目录
│   │   └── wind_file.bts
│   └── results/                           # 结果目录
│       ├── my_turbine.outb
│       └── my_turbine.sum
```

### 2. 版本控制

```bash
# 使用 Git 管理模型文件
cd ~/openfast/my_models/my_turbine
git init
git add *.fst *.dat
git commit -m "Initial model setup"
```

### 3. 参数化研究

创建脚本批量运行不同参数：

```python
#!/usr/bin/env python3
"""参数化研究示例"""

import subprocess
import os

# 参数范围
wind_speeds = [8, 10, 12, 14, 16]  # m/s

executable = '/home/timi/openfast/build/glue-codes/openfast/openfast'

for ws in wind_speeds:
    # 修改输入文件中的风速
    # ... (使用 openfast_toolbox 或手动修改)
    
    # 运行仿真
    input_file = f'my_turbine_ws{ws}.fst'
    subprocess.run([executable, input_file])
    
    # 重命名输出文件
    os.rename('my_turbine.outb', f'results/my_turbine_ws{ws}.outb')
```

### 4. 文档化

为每个模型创建 README：

```markdown
# My Turbine Model

## 模型描述
- 额定功率：5 MW
- 轮毂高度：90 m
- 转子直径：126 m

## 文件说明
- `my_turbine.fst`: 主输入文件
- `my_turbine_ElastoDyn.dat`: 结构参数
- ...

## 运行方法
```bash
~/openfast/build/glue-codes/openfast/openfast my_turbine.fst
```

## 参数来源
- 叶片数据：来自 [来源]
- 翼型数据：来自 [来源]
- ...
```

### 5. 验证流程

**推荐的验证步骤**：

1. **结构模态分析**（无气动、无控制）：
   ```fortran
   CompInflow=0
   CompAero=0
   CompServo=0
   ```

2. **静态分析**（稳态风、无控制）：
   ```fortran
   CompInflow=1
   WindType=2  # 稳态风
   CompAero=2
   CompServo=0
   ```

3. **动态分析**（湍流风、有控制）：
   ```fortran
   CompInflow=1
   WindType=9  # TurbSim 风场
   CompAero=2
   CompServo=1
   ```

### 6. 性能优化

**提高运行速度**：

1. **使用二进制输出**：
   ```fortran
   2   OutFileFmt  # 二进制格式更快
   ```

2. **减少输出通道**：
   ```fortran
   # 只输出需要的通道
   "RotSpeed GenPwr BlPitch1"  OutList
   ```

3. **增大输出时间步**：
   ```fortran
   0.1   DT_Out  # 如果不需要高频率输出
   ```

4. **关闭可视化**：
   ```fortran
   0   WrVTK  # 关闭 VTK 输出
   ```

---

## 总结

### 快速检查清单

在运行自定义模型前，确保：

- [ ] OpenFAST 已正确编译
- [ ] 所有输入文件存在且路径正确
- [ ] 文件格式与 OpenFAST 版本匹配
- [ ] 初始条件合理
- [ ] 时间步长合适
- [ ] 模块开关配置正确
- [ ] 输出目录有写权限

### 获取帮助

1. **官方文档**：
   - `docs/source/user/`：用户指南
   - `docs/source/working.rst`：运行指南

2. **测试案例**：
   - `reg_tests/r-test/`：回归测试案例
   - `reg_tests/r-test/glue-codes/openfast/MinimalExample/`：最小示例

3. **社区资源**：
   - GitHub Issues：https://github.com/OpenFAST/openfast/issues
   - 论坛：https://wind.nrel.gov/forum/wind/

4. **工具**：
   - `openfast_toolbox`：Python 工具
   - `matlab-toolbox`：MATLAB 工具

---

**祝你成功运行自己的模型！** 🚀

如有问题，请参考本文档的"常见问题解决"部分，或查阅官方文档和社区资源。

