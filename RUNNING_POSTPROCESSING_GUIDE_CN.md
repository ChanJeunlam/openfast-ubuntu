# OpenFAST 运行案例和后处理指南

本指南详细介绍如何运行 OpenFAST 测试案例，以及如何使用 Python 工具进行后处理和批量分析。

---

## 📋 目录

1. [运行测试案例](#运行测试案例)
2. [输出文件说明](#输出文件说明)
3. [Python 后处理工具（中级）](#python-后处理工具中级)
4. [自定义后处理脚本（高级）](#自定义后处理脚本高级)
5. [批量分析](#批量分析)
6. [实际案例](#实际案例)

---

## 运行测试案例

### 基本运行方法

#### 直接运行

```bash
# 基本语法
./glue-codes/openfast/openfast <输入文件路径>

# 示例：运行 5MW 陆地风机模态分析案例
cd ~/openfast/build
./glue-codes/openfast/openfast \
    ../reg_tests/r-test/glue-codes/openfast/5MW_Land_ModeShapes/5MW_Land_ModeShapes.fst
```

#### 命令行参数

OpenFAST 支持以下命令行参数：

```bash
# 显示版本信息
./glue-codes/openfast/openfast -v

# 显示帮助信息
./glue-codes/openfast/openfast -h

# 从检查点重启
./glue-codes/openfast/openfast -restart <检查点文件根名>

# 稳态分析
./glue-codes/openfast/openfast -steadystate <输入文件>
```

### 测试案例选择指南

#### 按配置分类

**陆地风机（简单，推荐初学者）**：
- `5MW_Land_ModeShapes`：模态分析
- `5MW_Land_DLL_WTurb`：动态链接库控制器
- `AWT_YFix_WSt`：固定偏航，稳态风
- `AWT_YFree_WTurb`：自由偏航，湍流风

**海上风机**：
- `5MW_OC3Spar_DLL_WTurb_WavesIrr`：Spar 浮式风机
- `5MW_OC3Mnpl_DLL_WTurb_WavesIrr`：单桩固定式风机
- `5MW_OC4Semi_WSt_WavesWN`：半潜式浮式风机

**高级配置**：
- `5MW_Land_BD_DLL_WTurb`：使用 BeamDyn 高级梁模型
- `5MW_Land_DLL_WTurb_ADsk`：使用 AeroDisk 气动盘模型
- `HelicalWake_OLAF`：使用 OLAF 自由涡尾流模型

#### 按复杂度分类

**简单案例**（运行时间 < 1 分钟）：
- `AWT_YFix_WSt`
- `5MW_Land_ModeShapes`

**中等案例**（运行时间 1-5 分钟）：
- `5MW_Land_DLL_WTurb`
- `5MW_OC3Mnpl_DLL_WTurb_WavesIrr`

**复杂案例**（运行时间 > 5 分钟）：
- `5MW_OC3Spar_DLL_WTurb_WavesIrr`
- `5MW_OC4Semi_WSt_WavesWN`

### 常见运行问题

#### 问题 1：文件路径错误

**错误信息**：
```
The input file, "xxx.fst", was not found.
```

**解决方案**：
```bash
# 检查文件是否存在
ls -la ../reg_tests/r-test/glue-codes/openfast/5MW_Land_ModeShapes/5MW_Land_ModeShapes.fst

# 使用绝对路径
./glue-codes/openfast/openfast \
    /home/timi/openfast/reg_tests/r-test/glue-codes/openfast/5MW_Land_ModeShapes/5MW_Land_ModeShapes.fst
```

#### 问题 2：依赖文件缺失

**错误信息**：
```
The ElastoDyn Blade file, "xxx.dat", was not found.
```

**解决方案**：
- 确保所有 `.dat` 文件都在正确的位置
- 检查 `.fst` 文件中的路径设置
- 某些案例需要 `5MW_Baseline` 目录中的数据文件

#### 问题 3：权限问题

**解决方案**：
```bash
# 确保可执行文件有执行权限
chmod +x ./glue-codes/openfast/openfast

# 确保输出目录有写权限
chmod 777 ../reg_tests/r-test/glue-codes/openfast/5MW_Land_ModeShapes/
```

---

## 输出文件说明

### 主要输出文件

运行完成后，会在测试案例目录下生成以下文件：

#### 1. `.outb` - 二进制输出文件（推荐）

**格式**：二进制格式，包含所有输出通道的时间序列

**优点**：
- 文件小，读写快
- 精度高
- 标准格式

**读取方法**：
```python
from reg_tests.lib.fast_io import load_output

data, info, pack = load_output('5MW_Land_ModeShapes.outb')
# data: numpy 数组，第一列是时间，后续列是各通道数据
# info: 字典，包含通道名称、单位等信息
```

#### 2. `.out` - ASCII 输出文件

**格式**：文本格式，可读

**结构**：
```
OpenFAST Output File
...
Time (s)  OoPDefl1 (m)  IPDefl1 (m)  TTDspFA (m)  ...
0.000     0.000         0.000         0.000        ...
0.005     0.001         0.000         0.000        ...
...
```

**读取方法**：
```python
import numpy as np

# 读取 ASCII 文件
data = np.loadtxt('5MW_Land_ModeShapes.out', skiprows=8)
time = data[:, 0]
channel1 = data[:, 1]  # 第一个通道
```

#### 3. `.ech` - 回显文件

**内容**：输入文件的副本，包含注释和解析后的值

**用途**：调试输入文件，查看实际使用的参数值

#### 4. `.sum` - 摘要文件

**内容**：
- 仿真参数摘要
- 输出通道列表
- 模块配置信息

#### 5. `.log` - 日志文件

**生成条件**：使用 Python 脚本运行时会生成

**内容**：运行时的标准输出和错误信息

### 输出通道说明

常见的输出通道包括：

**结构响应**：
- `OoPDefl1`, `OoPDefl2`, `OoPDefl3`：叶片面外变形（m）
- `IPDefl1`, `IPDefl2`, `IPDefl3`：叶片面内变形（m）
- `TTDspFA`, `TTDspSS`：塔顶前后/侧向位移（m）

**运动学**：
- `RotSpeed`：转子转速（rpm）
- `GenSpeed`：发电机转速（rpm）
- `NacYaw`：机舱偏航角（deg）

**载荷**：
- `RootMyb1`, `RootMyb2`, `RootMyb3`：叶片根部弯矩（kN-m）
- `TwrBsMyt`：塔底弯矩（kN-m）
- `GenPwr`：发电机功率（kW）

**控制**：
- `BlPitch1`, `BlPitch2`, `BlPitch3`：叶片变桨角（deg）
- `GenTrq`：发电机转矩（kN-m）

---

## Python 后处理工具（中级）

### 1. fast_io.py - 读取输出文件

**位置**：`reg_tests/lib/fast_io.py`

#### 基本使用

```python
import sys
sys.path.append('/home/timi/openfast/reg_tests/lib')
from fast_io import load_output

# 读取二进制输出文件
data, info, pack = load_output('5MW_Land_ModeShapes.outb')

# data: numpy 数组，形状为 (n_time_steps, n_channels+1)
#       第一列是时间，后续列是各通道数据
# info: 字典，包含：
#       - 'attribute_names': 通道名称列表
#       - 'attribute_units': 通道单位列表
#       - 'description': 文件描述
# pack: 压缩后的原始数据（用于调试）

# 获取时间序列
time = data[:, 0]

# 获取特定通道数据
channel_names = info['attribute_names']
rot_speed_idx = channel_names.index('RotSpeed')
rot_speed = data[:, rot_speed_idx]
```

#### 读取 ASCII 文件

```python
# fast_io 会自动识别文件格式
data, info, _ = load_output('5MW_Land_ModeShapes.out')  # ASCII 格式
```

#### 完整示例

```python
#!/usr/bin/env python3
"""读取并显示 OpenFAST 输出文件"""

import sys
sys.path.append('/home/timi/openfast/reg_tests/lib')
from fast_io import load_output
import numpy as np

# 读取输出文件
outfile = '5MW_Land_ModeShapes.outb'
data, info, _ = load_output(outfile)

# 显示文件信息
print(f"文件描述: {info['description']}")
print(f"通道数量: {len(info['attribute_names'])}")
print(f"时间步数: {data.shape[0]}")

# 显示所有通道名称
print("\n可用通道:")
for i, name in enumerate(info['attribute_names']):
    unit = info['attribute_units'][i]
    print(f"  {i:3d}: {name:20s} ({unit})")

# 提取特定通道
time = data[:, 0]
if 'RotSpeed' in info['attribute_names']:
    rot_speed_idx = info['attribute_names'].index('RotSpeed')
    rot_speed = data[:, rot_speed_idx]
    print(f"\n转子转速范围: {rot_speed.min():.2f} - {rot_speed.max():.2f} rpm")
```

---

### 2. errorPlotting.py - 误差对比绘图

**位置**：`reg_tests/lib/errorPlotting.py`

#### 功能

- 对比两个 OpenFAST 解（测试解 vs 基准解）
- 生成交互式 HTML 图表
- 计算误差和阈值

#### 基本使用

```python
import sys
sys.path.append('/home/timi/openfast/reg_tests/lib')
from errorPlotting import plotOpenfastError

# 绘制单个通道的误差图
plotOpenfastError(
    testSolution='test.outb',           # 测试解文件
    baselineSolution='baseline.outb',  # 基准解文件
    attribute='RotSpeed',               # 通道名称
    RTOL_MAGNITUDE=2,                   # 相对容差数量级
    ATOL_MAGNITUDE=1.9                  # 绝对容差数量级
)

# 这会生成 plots/RotSpeed_script.txt 和 plots/RotSpeed_div.txt
```

#### 批量生成所有通道的图表

```python
import sys
sys.path.append('/home/timi/openfast/reg_tests/lib')
from errorPlotting import plotOpenfastError, finalizePlotDirectory
from fast_io import load_output

# 读取文件信息
_, info, _ = load_output('test.outb')

# 为每个通道生成图表
for channel in info['attribute_names']:
    try:
        plotOpenfastError('test.outb', 'baseline.outb', channel, 2, 1.9)
    except Exception as e:
        print(f"Error plotting {channel}: {e}")

# 生成 HTML 导航页面
finalizePlotDirectory('test.outb', info['attribute_names'], 'CaseName')
```

#### 生成的图表说明

每个通道会生成两个图：
1. **对比图**：显示测试解和基准解的时间序列
2. **误差图**：显示绝对误差和阈值线

图表以 HTML 格式保存，可以在浏览器中打开查看。

---

### 3. pass_fail.py - 结果对比

**位置**：`reg_tests/lib/pass_fail.py`

#### 功能

- 比较测试解和基准解
- 计算各种范数（相对范数、L2 范数、最大范数）
- 判断通道是否通过测试

#### 基本使用

```python
import sys
sys.path.append('/home/timi/openfast/reg_tests/lib')
from pass_fail import readFASTOut, passing_channels, calculateNorms

# 读取两个输出文件
testData, testInfo, _ = readFASTOut('test.outb')
baselineData, baselineInfo, _ = readFASTOut('baseline.outb')

# 计算范数
norms = calculateNorms(testData, baselineData)
# norms 形状: (n_channels, 3)
# 列 0: 相对范数
# 列 1: 相对 L2 范数
# 列 2: 最大范数

# 判断通道是否通过
passing = passing_channels(testData.T, baselineData.T, RTOL_MAGNITUDE=2, ATOL_MAGNITUDE=1.9)
# passing: 布尔数组，True 表示通过

# 显示结果
for i, channel in enumerate(testInfo['attribute_names']):
    status = "PASS" if passing[i] else "FAIL"
    rel_norm = norms[i, 0]
    print(f"{channel:20s} {status:4s} 相对范数: {rel_norm:.2e}")
```

#### 完整示例

```python
#!/usr/bin/env python3
"""对比两个 OpenFAST 输出文件"""

import sys
sys.path.append('/home/timi/openfast/reg_tests/lib')
from pass_fail import readFASTOut, passing_channels, calculateNorms
import numpy as np

# 读取文件
test_file = 'test.outb'
baseline_file = 'baseline.outb'

testData, testInfo, _ = readFASTOut(test_file)
baselineData, baselineInfo, _ = readFASTOut(baseline_file)

# 计算范数和通过状态
norms = calculateNorms(testData, baselineData)
passing = passing_channels(testData.T, baselineData.T, RTOL_MAGNITUDE=2, ATOL_MAGNITUDE=1.9)

# 生成报告
print("=" * 80)
print("OpenFAST 结果对比报告")
print("=" * 80)
print(f"测试文件: {test_file}")
print(f"基准文件: {baseline_file}")
print(f"总通道数: {len(testInfo['attribute_names'])}")
print(f"通过通道数: {np.sum(passing)}")
print(f"失败通道数: {np.sum(~passing)}")
print("=" * 80)

# 显示失败通道
failed_channels = [name for i, name in enumerate(testInfo['attribute_names']) if not passing[i]]
if failed_channels:
    print("\n失败通道:")
    for i, name in enumerate(testInfo['attribute_names']):
        if not passing[i]:
            rel_norm = norms[i, 0]
            max_norm = norms[i, 2]
            print(f"  {name:20s}  相对范数: {rel_norm:.2e}  最大范数: {max_norm:.2e}")
```

---

### 4. openfastDrivers.py - 批量运行

**位置**：`reg_tests/lib/openfastDrivers.py`

#### 功能

提供统一的接口来运行各种 OpenFAST 驱动程序和案例。

#### 基本使用

```python
import sys
sys.path.append('/home/timi/openfast/reg_tests/lib')
from openfastDrivers import runOpenfastCase

# 运行 OpenFAST 案例
return_code = runOpenfastCase(
    inputFile='5MW_Land_ModeShapes.fst',
    executable='./glue-codes/openfast/openfast',
    verbose=True  # 显示详细输出
)

if return_code == 0:
    print("运行成功！")
else:
    print(f"运行失败，返回码: {return_code}")
```

#### 其他驱动函数

```python
# 运行 AeroMap 案例（稳态分析）
runAeromapCase('input.drv', executable)

# 运行 AeroDyn 驱动
runAerodynDriverCase('ad_driver.inp', executable)

# 运行 BeamDyn 驱动
runBeamdynDriverCase('bd_driver.inp', executable)

# 运行 HydroDyn 驱动
runHydrodynDriverCase('hd_driver.inp', executable)
```

---

## 自定义后处理脚本（高级）

### 1. 数据提取和分析

#### 提取特定通道数据

```python
#!/usr/bin/env python3
"""提取并分析特定通道数据"""

import sys
sys.path.append('/home/timi/openfast/reg_tests/lib')
from fast_io import load_output
import numpy as np

def extract_channel_data(outfile, channel_names):
    """
    从 OpenFAST 输出文件中提取指定通道的数据
    
    参数:
        outfile: 输出文件路径
        channel_names: 通道名称列表
    
    返回:
        dict: {通道名: 数据数组}
    """
    data, info, _ = load_output(outfile)
    time = data[:, 0]
    
    result = {'Time': time}
    
    for channel in channel_names:
        if channel in info['attribute_names']:
            idx = info['attribute_names'].index(channel)
            result[channel] = data[:, idx]
        else:
            print(f"警告: 通道 '{channel}' 不存在")
    
    return result

# 使用示例
channels = ['RotSpeed', 'GenPwr', 'BlPitch1', 'TTDspFA']
data_dict = extract_channel_data('5MW_Land_ModeShapes.outb', channels)

# 访问数据
time = data_dict['Time']
rot_speed = data_dict['RotSpeed']
```

#### 统计分析

```python
#!/usr/bin/env python3
"""计算统计量"""

import sys
sys.path.append('/home/timi/openfast/reg_tests/lib')
from fast_io import load_output
import numpy as np

def calculate_statistics(outfile, channel_name):
    """计算指定通道的统计量"""
    data, info, _ = load_output(outfile)
    
    if channel_name not in info['attribute_names']:
        raise ValueError(f"通道 '{channel_name}' 不存在")
    
    idx = info['attribute_names'].index(channel_name)
    values = data[:, idx]
    
    stats = {
        'mean': np.mean(values),
        'std': np.std(values),
        'min': np.min(values),
        'max': np.max(values),
        'rms': np.sqrt(np.mean(values**2)),
        'range': np.max(values) - np.min(values)
    }
    
    return stats

# 使用示例
stats = calculate_statistics('5MW_Land_ModeShapes.outb', 'RotSpeed')
print(f"转子转速统计:")
print(f"  均值: {stats['mean']:.2f} rpm")
print(f"  标准差: {stats['std']:.2f} rpm")
print(f"  最小值: {stats['min']:.2f} rpm")
print(f"  最大值: {stats['max']:.2f} rpm")
print(f"  RMS: {stats['rms']:.2f} rpm")
```

#### 频谱分析

```python
#!/usr/bin/env python3
"""频谱分析"""

import sys
sys.path.append('/home/timi/openfast/reg_tests/lib')
from fast_io import load_output
import numpy as np
from scipy import signal

def frequency_analysis(outfile, channel_name, dt=None):
    """
    对指定通道进行频谱分析
    
    参数:
        outfile: 输出文件路径
        channel_name: 通道名称
        dt: 时间步长（如果为 None，从数据中计算）
    
    返回:
        freqs: 频率数组 (Hz)
        psd: 功率谱密度
    """
    data, info, _ = load_output(outfile)
    
    if channel_name not in info['attribute_names']:
        raise ValueError(f"通道 '{channel_name}' 不存在")
    
    idx = info['attribute_names'].index(channel_name)
    values = data[:, idx]
    time = data[:, 0]
    
    if dt is None:
        dt = time[1] - time[0]  # 假设均匀时间步
    
    # 计算功率谱密度
    freqs, psd = signal.welch(values, 1/dt, nperseg=min(2048, len(values)))
    
    return freqs, psd

# 使用示例
freqs, psd = frequency_analysis('5MW_Land_ModeShapes.outb', 'TTDspFA')

# 找到主导频率
dominant_freq_idx = np.argmax(psd)
dominant_freq = freqs[dominant_freq_idx]
print(f"主导频率: {dominant_freq:.3f} Hz")
```

---

### 2. 可视化

#### 时间序列图

```python
#!/usr/bin/env python3
"""绘制时间序列图"""

import sys
sys.path.append('/home/timi/openfast/reg_tests/lib')
from fast_io import load_output
import matplotlib.pyplot as plt
import numpy as np

def plot_time_series(outfile, channel_names, save_path=None):
    """
    绘制多个通道的时间序列图
    
    参数:
        outfile: 输出文件路径
        channel_names: 通道名称列表
        save_path: 保存路径（可选）
    """
    data, info, _ = load_output(outfile)
    time = data[:, 0]
    
    n_plots = len(channel_names)
    fig, axes = plt.subplots(n_plots, 1, figsize=(10, 3*n_plots), sharex=True)
    
    if n_plots == 1:
        axes = [axes]
    
    for i, channel in enumerate(channel_names):
        if channel in info['attribute_names']:
            idx = info['attribute_names'].index(channel)
            unit = info['attribute_units'][idx]
            values = data[:, idx]
            
            axes[i].plot(time, values, linewidth=1.5)
            axes[i].set_ylabel(f'{channel} ({unit})')
            axes[i].grid(True, alpha=0.3)
            axes[i].set_title(channel)
        else:
            axes[i].text(0.5, 0.5, f"通道 '{channel}' 不存在", 
                        ha='center', va='center', transform=axes[i].transAxes)
    
    axes[-1].set_xlabel('Time (s)')
    plt.tight_layout()
    
    if save_path:
        plt.savefig(save_path, dpi=150)
    else:
        plt.show()

# 使用示例
plot_time_series(
    '5MW_Land_ModeShapes.outb',
    ['RotSpeed', 'GenPwr', 'BlPitch1', 'TTDspFA'],
    save_path='time_series.png'
)
```

#### 对比图

```python
#!/usr/bin/env python3
"""对比两个案例的结果"""

import sys
sys.path.append('/home/timi/openfast/reg_tests/lib')
from fast_io import load_output
import matplotlib.pyplot as plt
import numpy as np

def compare_cases(file1, file2, channel_names, labels=None):
    """
    对比两个案例的结果
    
    参数:
        file1, file2: 输出文件路径
        channel_names: 要对比的通道名称列表
        labels: 图例标签（可选）
    """
    data1, info1, _ = load_output(file1)
    data2, info2, _ = load_output(file2)
    
    if labels is None:
        labels = ['Case 1', 'Case 2']
    
    time1 = data1[:, 0]
    time2 = data2[:, 0]
    
    n_plots = len(channel_names)
    fig, axes = plt.subplots(n_plots, 1, figsize=(10, 3*n_plots), sharex=True)
    
    if n_plots == 1:
        axes = [axes]
    
    for i, channel in enumerate(channel_names):
        if channel in info1['attribute_names'] and channel in info2['attribute_names']:
            idx1 = info1['attribute_names'].index(channel)
            idx2 = info2['attribute_names'].index(channel)
            unit1 = info1['attribute_units'][idx1]
            
            axes[i].plot(time1, data1[:, idx1], label=labels[0], linewidth=2)
            axes[i].plot(time2, data2[:, idx2], label=labels[1], linewidth=2, linestyle='--')
            axes[i].set_ylabel(f'{channel} ({unit1})')
            axes[i].legend()
            axes[i].grid(True, alpha=0.3)
            axes[i].set_title(channel)
    
    axes[-1].set_xlabel('Time (s)')
    plt.tight_layout()
    plt.show()

# 使用示例
compare_cases(
    'case1.outb',
    'case2.outb',
    ['RotSpeed', 'GenPwr'],
    labels=['Baseline', 'Modified']
)
```

---

### 3. 批量分析

#### 参数化研究

```python
#!/usr/bin/env python3
"""参数化研究：批量运行不同参数的案例"""

import sys
sys.path.append('/home/timi/openfast/reg_tests/lib')
from openfastDrivers import runOpenfastCase
from fast_io import load_output
import numpy as np
import os

def parameter_study(base_fst_file, parameter_values, parameter_name, output_channel):
    """
    执行参数化研究
    
    参数:
        base_fst_file: 基础输入文件
        parameter_values: 参数值列表
        parameter_name: 参数名称（用于命名输出文件）
        output_channel: 要分析的输出通道
    
    返回:
        results: 字典，{参数值: 结果统计}
    """
    results = {}
    executable = './glue-codes/openfast/openfast'
    
    for value in parameter_values:
        print(f"\n运行参数 {parameter_name} = {value}")
        
        # 创建修改后的输入文件（这里简化处理，实际需要解析和修改 .fst 文件）
        modified_fst = f'temp_{parameter_name}_{value}.fst'
        # ... 修改输入文件的代码 ...
        
        # 运行案例
        return_code = runOpenfastCase(modified_fst, executable, verbose=False)
        
        if return_code == 0:
            # 读取结果
            outfile = modified_fst.replace('.fst', '.outb')
            if os.path.exists(outfile):
                data, info, _ = load_output(outfile)
                if output_channel in info['attribute_names']:
                    idx = info['attribute_names'].index(output_channel)
                    values = data[:, idx]
                    results[value] = {
                        'mean': np.mean(values),
                        'std': np.std(values),
                        'max': np.max(values),
                        'min': np.min(values)
                    }
                    print(f"  完成: {output_channel} 均值 = {results[value]['mean']:.2f}")
        
        # 清理临时文件
        if os.path.exists(modified_fst):
            os.remove(modified_fst)
    
    return results

# 使用示例
wind_speeds = [8, 10, 12, 14, 16]  # m/s
results = parameter_study(
    'base_case.fst',
    wind_speeds,
    'WindSpeed',
    'GenPwr'
)

# 绘制结果
import matplotlib.pyplot as plt
params = list(results.keys())
means = [results[p]['mean'] for p in params]
stds = [results[p]['std'] for p in params]

plt.errorbar(params, means, yerr=stds, marker='o', capsize=5)
plt.xlabel('Wind Speed (m/s)')
plt.ylabel('Mean Generator Power (kW)')
plt.grid(True)
plt.show()
```

#### 批量运行脚本

```python
#!/usr/bin/env python3
"""批量运行多个测试案例"""

import sys
sys.path.append('/home/timi/openfast/reg_tests/lib')
from openfastDrivers import runOpenfastCase
import os
import time

def batch_run_cases(case_list, executable, output_dir='results'):
    """
    批量运行多个案例
    
    参数:
        case_list: 案例文件路径列表
        executable: OpenFAST 可执行文件路径
        output_dir: 结果输出目录
    """
    os.makedirs(output_dir, exist_ok=True)
    
    results = []
    
    for i, case_file in enumerate(case_list, 1):
        case_name = os.path.splitext(os.path.basename(case_file))[0]
        print(f"\n[{i}/{len(case_list)}] 运行案例: {case_name}")
        
        start_time = time.time()
        return_code = runOpenfastCase(case_file, executable, verbose=False)
        elapsed_time = time.time() - start_time
        
        status = "成功" if return_code == 0 else "失败"
        results.append({
            'case': case_name,
            'status': status,
            'return_code': return_code,
            'time': elapsed_time
        })
        
        print(f"  状态: {status}, 耗时: {elapsed_time:.1f} 秒")
    
    # 生成汇总报告
    print("\n" + "="*60)
    print("批量运行汇总")
    print("="*60)
    total_time = sum(r['time'] for r in results)
    success_count = sum(1 for r in results if r['status'] == '成功')
    
    print(f"总案例数: {len(results)}")
    print(f"成功案例: {success_count}")
    print(f"失败案例: {len(results) - success_count}")
    print(f"总耗时: {total_time:.1f} 秒")
    print(f"平均耗时: {total_time/len(results):.1f} 秒/案例")
    
    # 保存结果到文件
    import json
    with open(os.path.join(output_dir, 'batch_results.json'), 'w') as f:
        json.dump(results, f, indent=2)
    
    return results

# 使用示例
case_files = [
    '../reg_tests/r-test/glue-codes/openfast/5MW_Land_ModeShapes/5MW_Land_ModeShapes.fst',
    '../reg_tests/r-test/glue-codes/openfast/AWT_YFix_WSt/AWT_YFix_WSt.fst',
    '../reg_tests/r-test/glue-codes/openfast/5MW_Land_DLL_WTurb/5MW_Land_DLL_WTurb.fst',
]

results = batch_run_cases(case_files, './glue-codes/openfast/openfast')
```

---

## 实际案例

### 案例 1：完整的后处理工作流

```python
#!/usr/bin/env python3
"""
完整的 OpenFAST 后处理工作流示例
1. 运行案例
2. 读取输出
3. 统计分析
4. 可视化
5. 生成报告
"""

import sys
sys.path.append('/home/timi/openfast/reg_tests/lib')
from openfastDrivers import runOpenfastCase
from fast_io import load_output
import numpy as np
import matplotlib.pyplot as plt
from datetime import datetime

def complete_postprocessing_workflow(fst_file, executable, output_dir='postprocessing'):
    """完整的后处理工作流"""
    
    import os
    os.makedirs(output_dir, exist_ok=True)
    
    # 1. 运行案例
    print("步骤 1: 运行 OpenFAST 案例...")
    case_name = os.path.splitext(os.path.basename(fst_file))[0]
    return_code = runOpenfastCase(fst_file, executable, verbose=False)
    
    if return_code != 0:
        print(f"错误: 案例运行失败，返回码: {return_code}")
        return
    
    # 2. 读取输出文件
    print("步骤 2: 读取输出文件...")
    outfile = fst_file.replace('.fst', '.outb')
    if not os.path.exists(outfile):
        outfile = fst_file.replace('.fst', '.out')
    
    data, info, _ = load_output(outfile)
    time = data[:, 0]
    
    # 3. 选择关键通道进行分析
    key_channels = ['RotSpeed', 'GenPwr', 'BlPitch1', 'TTDspFA', 'RootMyb1']
    available_channels = [ch for ch in key_channels if ch in info['attribute_names']]
    
    # 4. 统计分析
    print("步骤 3: 统计分析...")
    stats = {}
    for channel in available_channels:
        idx = info['attribute_names'].index(channel)
        values = data[:, idx]
        stats[channel] = {
            'mean': np.mean(values),
            'std': np.std(values),
            'min': np.min(values),
            'max': np.max(values),
            'rms': np.sqrt(np.mean(values**2))
        }
    
    # 5. 可视化
    print("步骤 4: 生成图表...")
    n_plots = len(available_channels)
    fig, axes = plt.subplots(n_plots, 1, figsize=(12, 3*n_plots), sharex=True)
    
    if n_plots == 1:
        axes = [axes]
    
    for i, channel in enumerate(available_channels):
        idx = info['attribute_names'].index(channel)
        unit = info['attribute_units'][idx]
        values = data[:, idx]
        
        axes[i].plot(time, values, linewidth=1.5)
        axes[i].set_ylabel(f'{channel}\n({unit})')
        axes[i].grid(True, alpha=0.3)
        
        # 添加统计信息
        mean_val = stats[channel]['mean']
        axes[i].axhline(mean_val, color='r', linestyle='--', alpha=0.5, label=f'均值: {mean_val:.2f}')
        axes[i].legend(loc='upper right')
    
    axes[-1].set_xlabel('Time (s)')
    plt.suptitle(f'OpenFAST 仿真结果: {case_name}', fontsize=14, y=0.995)
    plt.tight_layout()
    plt.savefig(os.path.join(output_dir, f'{case_name}_plots.png'), dpi=150)
    plt.close()
    
    # 6. 生成文本报告
    print("步骤 5: 生成报告...")
    report_file = os.path.join(output_dir, f'{case_name}_report.txt')
    with open(report_file, 'w', encoding='utf-8') as f:
        f.write("="*60 + "\n")
        f.write(f"OpenFAST 后处理报告\n")
        f.write("="*60 + "\n")
        f.write(f"案例名称: {case_name}\n")
        f.write(f"生成时间: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}\n")
        f.write(f"仿真时长: {time[-1]:.1f} 秒\n")
        f.write(f"时间步数: {len(time)}\n")
        f.write("\n" + "-"*60 + "\n")
        f.write("关键通道统计\n")
        f.write("-"*60 + "\n")
        
        for channel in available_channels:
            s = stats[channel]
            unit = info['attribute_units'][info['attribute_names'].index(channel)]
            f.write(f"\n{channel} ({unit}):\n")
            f.write(f"  均值: {s['mean']:12.4f}\n")
            f.write(f"  标准差: {s['std']:10.4f}\n")
            f.write(f"  最小值: {s['min']:12.4f}\n")
            f.write(f"  最大值: {s['max']:12.4f}\n")
            f.write(f"  RMS: {s['rms']:14.4f}\n")
    
    print(f"\n后处理完成！")
    print(f"  图表: {os.path.join(output_dir, f'{case_name}_plots.png')}")
    print(f"  报告: {report_file}")

# 使用示例
complete_postprocessing_workflow(
    '../reg_tests/r-test/glue-codes/openfast/5MW_Land_ModeShapes/5MW_Land_ModeShapes.fst',
    './glue-codes/openfast/openfast'
)
```

---

### 案例 2：对比多个案例

```python
#!/usr/bin/env python3
"""对比多个 OpenFAST 案例的结果"""

import sys
sys.path.append('/home/timi/openfast/reg_tests/lib')
from fast_io import load_output
import matplotlib.pyplot as plt
import numpy as np

def compare_multiple_cases(case_files, case_labels, channel_names, output_dir='comparison'):
    """
    对比多个案例的结果
    
    参数:
        case_files: 案例文件路径列表
        case_labels: 案例标签列表
        channel_names: 要对比的通道名称列表
        output_dir: 输出目录
    """
    import os
    os.makedirs(output_dir, exist_ok=True)
    
    # 读取所有案例的数据
    all_data = {}
    for case_file, label in zip(case_files, case_labels):
        outfile = case_file.replace('.fst', '.outb')
        if not os.path.exists(outfile):
            outfile = case_file.replace('.fst', '.out')
        
        data, info, _ = load_output(outfile)
        all_data[label] = {'data': data, 'info': info}
    
    # 为每个通道生成对比图
    for channel in channel_names:
        fig, ax = plt.subplots(1, 1, figsize=(10, 6))
        
        for label, case_data in all_data.items():
            info = case_data['info']
            data = case_data['data']
            
            if channel in info['attribute_names']:
                idx = info['attribute_names'].index(channel)
                unit = info['attribute_units'][idx]
                time = data[:, 0]
                values = data[:, idx]
                
                ax.plot(time, values, label=label, linewidth=2)
        
        ax.set_xlabel('Time (s)')
        ax.set_ylabel(f'{channel} ({unit})')
        ax.set_title(f'通道对比: {channel}')
        ax.legend()
        ax.grid(True, alpha=0.3)
        
        plt.tight_layout()
        plt.savefig(os.path.join(output_dir, f'compare_{channel}.png'), dpi=150)
        plt.close()
        
        print(f"已生成对比图: compare_{channel}.png")

# 使用示例
case_files = [
    '../reg_tests/r-test/glue-codes/openfast/5MW_Land_ModeShapes/5MW_Land_ModeShapes.fst',
    '../reg_tests/r-test/glue-codes/openfast/5MW_Land_DLL_WTurb/5MW_Land_DLL_WTurb.fst',
]

case_labels = ['ModeShapes', 'DLL_WTurb']
channels = ['RotSpeed', 'GenPwr', 'BlPitch1']

compare_multiple_cases(case_files, case_labels, channels)
```

---

### 案例 3：自动化回归测试

```python
#!/usr/bin/env python3
"""自动化回归测试脚本"""

import sys
sys.path.append('/home/timi/openfast/reg_tests/lib')
from openfastDrivers import runOpenfastCase
from pass_fail import readFASTOut, passing_channels, calculateNorms
import os
import json

def automated_regression_test(test_case, baseline_case, executable, rtol=2, atol=1.9):
    """
    自动化回归测试
    
    参数:
        test_case: 测试案例的 .fst 文件路径
        baseline_case: 基准案例的 .fst 文件路径（或输出文件路径）
        executable: OpenFAST 可执行文件路径
        rtol: 相对容差数量级
        atol: 绝对容差数量级
    
    返回:
        dict: 测试结果
    """
    case_name = os.path.splitext(os.path.basename(test_case))[0]
    
    # 1. 运行测试案例
    print(f"运行测试案例: {case_name}")
    return_code = runOpenfastCase(test_case, executable, verbose=False)
    
    if return_code != 0:
        return {
            'case': case_name,
            'status': 'FAILED_TO_RUN',
            'return_code': return_code
        }
    
    # 2. 读取输出文件
    test_outfile = test_case.replace('.fst', '.outb')
    if not os.path.exists(test_outfile):
        test_outfile = test_case.replace('.fst', '.out')
    
    baseline_outfile = baseline_case
    if baseline_case.endswith('.fst'):
        baseline_outfile = baseline_case.replace('.fst', '.outb')
        if not os.path.exists(baseline_outfile):
            baseline_outfile = baseline_case.replace('.fst', '.out')
    
    if not os.path.exists(test_outfile) or not os.path.exists(baseline_outfile):
        return {
            'case': case_name,
            'status': 'MISSING_OUTPUT',
            'test_file': test_outfile,
            'baseline_file': baseline_outfile
        }
    
    # 3. 对比结果
    testData, testInfo, _ = readFASTOut(test_outfile)
    baselineData, baselineInfo, _ = readFASTOut(baseline_outfile)
    
    passing = passing_channels(testData.T, baselineData.T, rtol, atol)
    norms = calculateNorms(testData, baselineData)
    
    # 4. 生成结果
    failed_channels = [testInfo['attribute_names'][i] 
                      for i in range(len(passing)) if not passing[i]]
    
    result = {
        'case': case_name,
        'status': 'PASS' if np.all(passing) else 'FAIL',
        'total_channels': len(passing),
        'passed_channels': np.sum(passing),
        'failed_channels': len(failed_channels),
        'failed_channel_names': failed_channels,
        'max_relative_norm': float(np.max(norms[:, 0])),
        'max_l2_norm': float(np.max(norms[:, 1])),
        'max_abs_norm': float(np.max(norms[:, 2]))
    }
    
    return result

# 批量回归测试
def batch_regression_test(case_list, baseline_dir, executable, rtol=2, atol=1.9):
    """批量回归测试"""
    results = []
    
    for test_case in case_list:
        case_name = os.path.splitext(os.path.basename(test_case))[0]
        baseline_case = os.path.join(baseline_dir, case_name, f'{case_name}.outb')
        
        result = automated_regression_test(test_case, baseline_case, executable, rtol, atol)
        results.append(result)
        
        status_icon = "✓" if result['status'] == 'PASS' else "✗"
        print(f"{status_icon} {case_name:40s} {result['status']}")
    
    # 生成汇总报告
    total = len(results)
    passed = sum(1 for r in results if r['status'] == 'PASS')
    failed = total - passed
    
    print("\n" + "="*60)
    print("回归测试汇总")
    print("="*60)
    print(f"总案例数: {total}")
    print(f"通过: {passed}")
    print(f"失败: {failed}")
    print(f"通过率: {passed/total*100:.1f}%")
    
    # 保存结果
    with open('regression_test_results.json', 'w') as f:
        json.dump(results, f, indent=2)
    
    return results

# 使用示例
test_cases = [
    '../reg_tests/r-test/glue-codes/openfast/5MW_Land_ModeShapes/5MW_Land_ModeShapes.fst',
    '../reg_tests/r-test/glue-codes/openfast/AWT_YFix_WSt/AWT_YFix_WSt.fst',
]

baseline_dir = '../reg_tests/r-test/glue-codes/openfast'
executable = './glue-codes/openfast/openfast'

results = batch_regression_test(test_cases, baseline_dir, executable)
```

---

## 总结

### 工具总结

| 工具 | 功能 | 适用场景 |
|------|------|----------|
| `fast_io.py` | 读取输出文件 | 数据提取和分析 |
| `errorPlotting.py` | 误差对比绘图 | 回归测试、结果对比 |
| `pass_fail.py` | 结果对比和判断 | 自动化测试 |
| `openfastDrivers.py` | 批量运行案例 | 参数化研究、批量分析 |

### 工作流建议

1. **单案例分析**：
   - 运行案例 → `fast_io` 读取 → 统计分析 → 可视化

2. **结果对比**：
   - 运行两个案例 → `pass_fail` 对比 → `errorPlotting` 绘图

3. **批量分析**：
   - `openfastDrivers` 批量运行 → 循环处理 → 汇总结果

4. **回归测试**：
   - `openfastDrivers` 运行 → `pass_fail` 对比 → 生成报告

### 最佳实践

1. **使用二进制文件**：`.outb` 文件更小、更快
2. **错误处理**：始终检查文件是否存在和通道是否可用
3. **保存中间结果**：保存处理后的数据，避免重复计算
4. **文档化**：在脚本中添加注释，说明用途和参数

---

**祝你使用顺利！** 🚀

