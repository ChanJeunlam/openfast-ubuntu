# OpenFAST 源码阅读指南（深入分析）

本指南深入分析 OpenFAST 项目源码，帮助开发者理解项目架构、核心模块实现细节和代码调用关系。

---

## 📋 目录

1. [项目整体架构](#项目整体架构)
2. [主程序入口分析](#主程序入口分析)
3. [核心模块深入分析](#核心模块深入分析)
4. [模块调用关系](#模块调用关系)
5. [数据流和接口机制](#数据流和接口机制)
6. [关键代码示例](#关键代码示例)

---

## 项目整体架构

### 目录结构

OpenFAST 采用模块化设计，主要目录结构如下：

```
openfast/
├── glue-codes/          # 胶合代码（主程序入口）
│   ├── openfast/       # OpenFAST 主程序
│   ├── fast-farm/      # FAST.Farm 多风机程序
│   ├── openfast-cpp/   # C++ API 接口
│   └── simulink/       # Simulink 接口
├── modules/            # 功能模块
│   ├── aerodyn/        # 气动力学模块
│   ├── elastodyn/      # 结构动力学模块
│   ├── servodyn/       # 控制系统模块
│   ├── hydrodyn/       # 水动力学模块
│   ├── inflowwind/     # 风场模块
│   ├── beamdyn/        # 高级梁模型
│   ├── subdyn/         # 子结构动力学
│   ├── moordyn/        # 系泊系统
│   └── ...             # 其他模块
├── modules/openfast-library/  # OpenFAST 库（胶合逻辑）
├── reg_tests/          # 回归测试
└── docs/               # 文档
```

### 模块化设计原则

OpenFAST 遵循 FAST Modularization Framework（FAST 模块化框架），每个模块都实现标准接口：

1. **初始化接口**：`ModuleName_Init()`
2. **状态更新接口**：`ModuleName_UpdateStates()`
3. **输出计算接口**：`ModuleName_CalcOutput()`
4. **结束接口**：`ModuleName_End()`

### 编译系统

- **CMake**：跨平台构建系统
- **Fortran 90/95**：主要编程语言
- **C/C++**：部分接口和工具
- **Registry 系统**：自动生成类型定义和接口代码

---

## 主程序入口分析

### FAST_Prog.f90 - 主程序

**位置**：`glue-codes/openfast/src/FAST_Prog.f90`

这是 OpenFAST 的主程序入口点，负责：

1. **命令行参数解析**
2. **初始化所有模块**
3. **时间步进循环**
4. **输出和清理**

#### 程序启动流程

```fortran
PROGRAM FAST
   USE FAST_Subs   ! 包含所有模块类型定义
   
   ! 1. 初始化 NWTC 库
   CALL NWTC_Init()
   
   ! 2. 解析命令行参数
   CALL CheckArgs( InputFile, Flag=FlagArg, Arg2=CheckpointRoot )
   
   ! 3. 处理特殊标志（重启、稳态分析等）
   IF ( TRIM(FlagArg) == 'RESTART' ) THEN
      ! 从检查点恢复
   ELSE IF ( TRIM(FlagArg) == 'STEADYSTATE' ) THEN
      ! 稳态分析
   ELSE
      ! 4. 初始化所有模块
      CALL FAST_InitializeAll_T( t_initial, i_turb, Turbine(i_turb), ErrStat, ErrMsg )
      
      ! 5. 初始解计算
      CALL FAST_Solution0_T( Turbine(i_turb), ErrStat, ErrMsg )
      
      ! 6. 时间步进循环
      DO n_t_global = Restart_step, Turbine(1)%p_FAST%n_TMax_m1
         CALL FAST_Solution_T( t_initial, n_t_global, Turbine(i_turb), ErrStat, ErrMsg )
      END DO
   END IF
END PROGRAM FAST
```

#### 关键数据结构

```fortran
TYPE(FAST_TurbineType) :: Turbine(NumTurbines)
```

`FAST_TurbineType` 包含：
- `p_FAST`：全局参数
- `y_FAST`：输出文件句柄
- `m_FAST`：杂项变量
- `ED`：ElastoDyn 模块数据
- `AD`：AeroDyn 模块数据
- `SrvD`：ServoDyn 模块数据
- `HD`：HydroDyn 模块数据
- `BD`：BeamDyn 模块数据
- ... 其他模块数据

---

## 核心模块深入分析

### 1. ElastoDyn 模块 - 结构动力学

**位置**：`modules/elastodyn/src/ElastoDyn.f90`

#### 功能

ElastoDyn 模拟风机的结构动力学，包括：
- 叶片、轮毂、机舱、塔架的刚体运动
- 叶片的模态表示（挥舞、摆振、扭转）
- 传动系统动力学

#### 关键子程序

**初始化**：
```fortran
SUBROUTINE ED_Init( InitInp, u, p, x, xd, z, OtherState, y, m, Interval, InitOut, ErrStat, ErrMsg )
```

**状态更新**：
```fortran
SUBROUTINE ED_UpdateStates( t, n, u, p, x, xd, z, OtherState, m, ErrStat, ErrMsg )
```

**输出计算**：
```fortran
SUBROUTINE ED_CalcOutput( t, u, p, x, xd, z, OtherState, y, m, ErrStat, ErrMsg )
```

#### 输入输出接口

**输入**（`ED_InputType`）：
- `HubPtMotion`：轮毂点运动（位置、速度、加速度）
- `NacelleMotion`：机舱运动
- `TowerPtLoads`：塔架点载荷
- `BladePtLoads`：叶片点载荷（每个叶片）

**输出**（`ED_OutputType`）：
- `HubPtMotion`：轮毂点运动（传递给 AeroDyn）
- `BladeRootMotion`：叶片根部运动（传递给 AeroDyn）
- `NacelleMotion`：机舱运动
- `TowerLn2Mesh`：塔架线网格运动

#### 关键代码片段

```fortran
! 读取输入文件
CALL ED_ReadInput( InitInp%InputFile, InputFileData, p%BD4Blades, Interval, p%RootName, ErrStat2, ErrMsg2 )

! 设置自由度
IF ( p%BD4Blades .or. p%RigidAero ) THEN
   InputFileData%FlapDOF1 = .FALSE.
   InputFileData%FlapDOF2 = .FALSE.
   InputFileData%EdgeDOF  = .FALSE.
ENDIF

! 计算初始状态
CALL ED_CalcInitialState( p, x, xd, z, OtherState, ErrStat, ErrMsg )
```

---

### 2. AeroDyn 模块 - 气动力学

**位置**：`modules/aerodyn/src/AeroDyn.f90`

#### 功能

AeroDyn 计算叶片的气动载荷，支持多种气动模型：
- **BEM（Blade Element Momentum）**：经典叶素动量理论
- **DBEMT（Dynamic BEM）**：动态 BEM
- **FVW（Free Vortex Wake）**：自由涡尾流模型
- **OLAF（OpenFAST Lifting Line Free Vortex Wake）**：升力线自由涡尾流

#### 关键子程序

**初始化**：
```fortran
SUBROUTINE AD_Init( InitInp, u, p, x, xd, z, OtherState, y, m, Interval, InitOut, ErrStat, ErrMsg )
```

**输出计算**（核心）：
```fortran
SUBROUTINE AD_CalcOutput( t, u, p, x, xd, z, OtherState, y, m, ErrStat, ErrMsg )
```

#### BEM 理论实现

AeroDyn 的核心是 BEM 理论，主要步骤：

1. **计算诱导速度**：
   ```fortran
   ! 轴向诱导因子
   a = 1.0 / (1.0 + 4.0*F*sin(phi)**2 / (sigma*Cl*cos(phi)))
   
   ! 切向诱导因子
   a_prime = 1.0 / (4.0*F*cos(phi) / (sigma*Cl*sin(phi)) - 1.0)
   ```

2. **计算攻角**：
   ```fortran
   alpha = phi - theta_pitch
   ```

3. **查表获取升阻力系数**：
   ```fortran
   CALL GetAirfoilCoefs( alpha, Re, Cl, Cd, Cm, AFInfo )
   ```

4. **计算气动力**：
   ```fortran
   ! 升力
   L = 0.5 * rho * Vrel**2 * chord * Cl
   
   ! 阻力
   D = 0.5 * rho * Vrel**2 * chord * Cd
   
   ! 法向力（垂直于旋转平面）
   Fn = L*cos(phi) + D*sin(phi)
   
   ! 切向力（平行于旋转平面）
   Ft = L*sin(phi) - D*cos(phi)
   ```

#### 输入输出接口

**输入**（`AD_InputType`）：
- `rotors(1)%BladeMotion`：叶片运动（来自 ElastoDyn）
- `rotors(1)%HubMotion`：轮毂运动（来自 ElastoDyn）
- `InflowOnBlade`：叶片上的入流速度（来自 InflowWind）

**输出**（`AD_OutputType`）：
- `rotors(1)%BladeLoad`：叶片载荷（传递给 ElastoDyn）
- `rotors(1)%HubLoad`：轮毂载荷（传递给 ElastoDyn）

---

### 3. ServoDyn 模块 - 控制系统

**位置**：`modules/servodyn/src/ServoDyn.f90`

#### 功能

ServoDyn 模拟风机的控制系统，包括：
- 变桨控制
- 发电机转矩控制
- 偏航控制
- DLL 接口（支持外部控制器）

#### 控制模式

```fortran
INTEGER(IntKi), PARAMETER :: ControlMode_NONE      = 0  ! 无控制
INTEGER(IntKi), PARAMETER :: ControlMode_SIMPLE    = 1  ! 简单内置控制器
INTEGER(IntKi), PARAMETER :: ControlMode_ADVANCED  = 2  ! 高级内置控制器
INTEGER(IntKi), PARAMETER :: ControlMode_USER      = 3  ! 用户定义控制器
INTEGER(IntKi), PARAMETER :: ControlMode_EXTERN    = 4  ! 外部控制器（Simulink/LabVIEW）
INTEGER(IntKi), PARAMETER :: ControlMode_DLL       = 5  ! DLL 控制器（Bladed 风格）
```

#### DLL 接口

ServoDyn 支持通过 DLL 加载外部控制器：

```fortran
! 加载 DLL
CALL LoadDynamicLib( DLL_FileName, DLL_Prog, ErrStat, ErrMsg )

! 调用控制器
CALL DLL_Controller( t, GenSpeed, GenTrq, BladePitch, YawAngle, ... )
```

#### 输入输出接口

**输入**（`SrvD_InputType`）：
- `GenTrq`：发电机转矩（来自 ElastoDyn）
- `GenSpeed`：发电机转速（来自 ElastoDyn）
- `WindSpeed`：风速（来自 InflowWind）

**输出**（`SrvD_OutputType`）：
- `BlPitchCom`：变桨指令（传递给 ElastoDyn）
- `GenTrq`：发电机转矩指令（传递给 ElastoDyn）
- `YawMom`：偏航力矩（传递给 ElastoDyn）

---

### 4. HydroDyn 模块 - 水动力学

**位置**：`modules/hydrodyn/src/HydroDyn.f90`

#### 功能

HydroDyn 计算海上结构物的水动力载荷，包括：
- **WAMIT**：势流理论（辐射、绕射）
- **Morison 方程**：粘性载荷
- **波浪模型**：规则波、不规则波

#### 子模块

- `WAMIT`：势流计算
- `WAMIT2`：二阶势流
- `Morison`：Morison 方程
- `SeaState`：海况模型

#### 关键计算

**Morison 方程**：
```fortran
! 拖曳力
F_drag = 0.5 * rho * Cd * D * |u| * u

! 惯性力
F_inertia = rho * Cm * A * du/dt

! 总力
F_total = F_drag + F_inertia
```

**WAMIT 辐射力**：
```fortran
! 辐射力 = 附加质量 * 加速度 + 辐射阻尼 * 速度
F_radiation = -A_added * a - B_radiation * v
```

#### 输入输出接口

**输入**（`HD_InputType`）：
- `Morison%LumpedMesh`：Morison 单元网格运动
- `WAMITMesh`：WAMIT 网格运动

**输出**（`HD_OutputType`）：
- `Morison%LumpedMesh`：Morison 载荷
- `WAMITMesh`：WAMIT 载荷

---

### 5. InflowWind 模块 - 风场模型

**位置**：`modules/inflowwind/src/InflowWind.f90`

#### 功能

InflowWind 提供风场数据，支持多种风场类型：
- **均匀风**：恒定风速
- **用户定义风**：自定义风场文件
- **TurbSim 风**：TurbSim 生成的风场
- **HAWC 风**：HAWC 格式风场
- **Bladed 风**：Bladed 格式风场

#### 关键子程序

```fortran
! 获取指定位置的风速
SUBROUTINE IfW_GetWindSpeed( t, Position, WindSpeed, ErrStat, ErrMsg )
```

---

## 模块调用关系

### 初始化序列

模块初始化有严格的顺序要求（在 `FAST_InitializeAll` 中定义）：

```
1. ElastoDyn (或 Simplified-ElastoDyn)
   └─ 提供叶片数量、轮毂位置等基础信息

2. BeamDyn (如果使用)
   └─ 需要 ElastoDyn 的轮毂位置

3. InflowWind
   └─ 独立初始化

4. AeroDyn
   └─ 需要 ElastoDyn 的叶片几何信息
   └─ 需要 InflowWind 的风场数据

5. ServoDyn
   └─ 需要 ElastoDyn 的发电机信息

6. HydroDyn (如果使用)
   └─ 需要 ElastoDyn 的平台位置

7. SubDyn (如果使用)
   └─ 需要 ElastoDyn 的平台信息

8. MoorDyn/MAP (如果使用)
   └─ 需要 ElastoDyn 的平台位置
```

### 时间步进调用序列

在每个时间步，模块按以下顺序调用：

```
1. FAST_Solution_T (主求解器)
   │
   ├─ 2. 计算模块输入
   │   ├─ ED_InputSolve (ElastoDyn 输入)
   │   ├─ AD_InputSolve (AeroDyn 输入)
   │   └─ ...
   │
   ├─ 3. 更新状态
   │   ├─ ED_UpdateStates
   │   ├─ AD_UpdateStates
   │   ├─ SrvD_UpdateStates
   │   └─ ...
   │
   ├─ 4. 计算输出
   │   ├─ ED_CalcOutput
   │   ├─ AD_CalcOutput
   │   ├─ SrvD_CalcOutput
   │   └─ ...
   │
   └─ 5. 写入输出文件
```

### 调用关系图（文本表示）

```
                    FAST_Prog.f90
                         │
                         ├─ FAST_InitializeAll_T
                         │       │
                         │       ├─ ED_Init (ElastoDyn)
                         │       ├─ AD_Init (AeroDyn)
                         │       ├─ SrvD_Init (ServoDyn)
                         │       ├─ HD_Init (HydroDyn)
                         │       └─ ...
                         │
                         └─ FAST_Solution_T (时间步进循环)
                                 │
                                 ├─ FAST_InputSolve_All
                                 │       │
                                 │       ├─ ED_InputSolve
                                 │       │   └─ 从其他模块获取载荷
                                 │       │
                                 │       ├─ AD_InputSolve
                                 │       │   └─ 从 ED 获取叶片运动
                                 │       │   └─ 从 IfW 获取风速
                                 │       │
                                 │       └─ ...
                                 │
                                 ├─ FAST_UpdateStates_All
                                 │       │
                                 │       ├─ ED_UpdateStates
                                 │       ├─ AD_UpdateStates
                                 │       ├─ SrvD_UpdateStates
                                 │       └─ ...
                                 │
                                 └─ FAST_CalcOutputs_All
                                         │
                                         ├─ ED_CalcOutput
                                         ├─ AD_CalcOutput
                                         ├─ SrvD_CalcOutput
                                         └─ ...
```

---

## 数据流和接口机制

### Registry 系统

OpenFAST 使用 Registry 系统自动生成模块的类型定义和接口代码。

**Registry 文件位置**：`modules/<module>/src/<Module>_Registry.txt`

**示例**（ElastoDyn_Registry.txt）：
```
ED_InputType
   - HubPtMotion: MeshType
   - NacelleMotion: MeshType
   - TowerPtLoads: MeshType
   - BladePtLoads: MeshType(3)
```

Registry 系统会生成：
- `ElastoDyn_Types.f90`：类型定义
- 输入/输出结构体
- 参数结构体

### Mesh 系统

OpenFAST 使用 Mesh 系统在不同模块间传递空间分布的数据（如载荷、运动）。

**Mesh 类型**：
- `Point`：点数据（如轮毂点）
- `Line2`：线数据（如叶片、塔架）
- `Line3`：三节点线数据

**Mesh 映射**：
```fortran
! 从 AeroDyn 叶片载荷映射到 ElastoDyn 叶片载荷
CALL Transfer_Line2_to_Line2( y_AD%rotors(1)%BladeLoad(k), &
                               ED%Input(1)%BladePtLoads(k), &
                               MeshMapData%AD_L_2_ED_L_B(k), &
                               ErrStat, ErrMsg, &
                               u_AD%rotors(1)%BladeMotion(k), &
                               ED%y%BladeLn2Mesh(k) )
```

### 数据传递示例

**ElastoDyn → AeroDyn**：
```fortran
! ElastoDyn 输出叶片运动
ED%y%BladeLn2Mesh(k)%Position(:,:)  ! 叶片位置
ED%y%BladeLn2Mesh(k)%TranslationVel(:,:)  ! 叶片速度

! AeroDyn 接收作为输入
AD%Input(1)%rotors(1)%BladeMotion(k) = ED%y%BladeLn2Mesh(k)
```

**AeroDyn → ElastoDyn**：
```fortran
! AeroDyn 输出叶片载荷
AD%y%rotors(1)%BladeLoad(k)%Force(:,:)   ! 叶片气动力
AD%y%rotors(1)%BladeLoad(k)%Moment(:,:)  ! 叶片气动力矩

! 映射到 ElastoDyn 输入
CALL Transfer_Line2_to_Line2( AD%y%rotors(1)%BladeLoad(k), &
                                ED%Input(1)%BladePtLoads(k), ... )
```

---

## 关键代码示例

### 示例 1：模块初始化

```fortran
! 在 FAST_InitializeAll 中初始化 ElastoDyn
Init%InData_ED%InputFile = p_FAST%EDFile
Init%InData_ED%RootName  = TRIM(p_FAST%OutFileRoot)//'.ED'
Init%InData_ED%Gravity   = p_FAST%Gravity

CALL ED_Init( Init%InData_ED, &
              ED%Input(1), &
              ED%p, &
              ED%x(STATE_CURR), &
              ED%xd(STATE_CURR), &
              ED%z(STATE_CURR), &
              ED%OtherSt(STATE_CURR), &
              ED%y, &
              ED%m, &
              p_FAST%dt_module(MODULE_ED), &
              Init%OutData_ED, &
              ErrStat2, ErrMsg2 )
```

### 示例 2：时间步进求解

```fortran
! 在 FAST_Solution_T 中
DO n_t_global = Restart_step, Turbine(1)%p_FAST%n_TMax_m1
   
   ! 1. 计算模块输入
   CALL FAST_InputSolve_All( t_global, Turbine(i_turb), ErrStat, ErrMsg )
   
   ! 2. 更新状态
   CALL FAST_UpdateStates_All( t_global, n_t_global, Turbine(i_turb), ErrStat, ErrMsg )
   
   ! 3. 计算输出
   CALL FAST_CalcOutputs_All( t_global, Turbine(i_turb), ErrStat, ErrMsg )
   
   ! 4. 写入输出文件
   CALL FAST_WriteOutput( Turbine(i_turb), ErrStat, ErrMsg )
   
END DO
```

### 示例 3：AeroDyn 输入求解

```fortran
! 在 AD_InputSolve 中
DO k = 1, p_FAST%nBeams  ! 遍历所有叶片
   
   ! 从 ElastoDyn 获取叶片运动
   AD%Input(1)%rotors(1)%BladeMotion(k) = ED%y%BladeLn2Mesh(k)
   
   ! 从 InflowWind 获取风速
   DO i = 1, AD%Input(1)%rotors(1)%BladeMotion(k)%NNodes
      Position = AD%Input(1)%rotors(1)%BladeMotion(k)%Position(:,i)
      CALL IfW_GetWindSpeed( t, Position, WindSpeed, ErrStat, ErrMsg )
      AD%Input(1)%rotors(1)%InflowOnBlade(k)%Velocity(:,i) = WindSpeed
   END DO
   
END DO
```

### 示例 4：载荷传递

```fortran
! 从 AeroDyn 传递载荷到 ElastoDyn
DO k = 1, p_FAST%nBeams
   
   CALL Transfer_Line2_to_Line2( &
      y_AD%rotors(1)%BladeLoad(k),           ! 源：AeroDyn 叶片载荷
      ED%Input(1)%BladePtLoads(k),          ! 目标：ElastoDyn 叶片载荷
      MeshMapData%AD_L_2_ED_L_B(k),         ! 映射数据
      ErrStat, ErrMsg,                      ! 错误状态
      u_AD%rotors(1)%BladeMotion(k),        ! 源网格（用于插值）
      ED%y%BladeLn2Mesh(k)                  ! 目标网格
   )
   
END DO
```

---

## 总结

### 关键设计模式

1. **模块化设计**：每个物理过程独立为模块
2. **标准接口**：所有模块实现相同的接口模式
3. **松耦合**：模块间通过 Mesh 系统传递数据
4. **Registry 系统**：自动生成类型定义和接口代码

### 学习建议

1. **从主程序开始**：理解 `FAST_Prog.f90` 的整体流程
2. **深入单个模块**：选择一个模块（如 ElastoDyn）深入理解
3. **理解数据流**：跟踪数据在不同模块间的传递
4. **阅读测试案例**：通过测试案例理解模块使用方式

### 参考资料

- **官方文档**：`docs/source/`
- **模块 README**：`modules/<module>/README.md`
- **测试案例**：`reg_tests/r-test/`
- **Registry 文件**：`modules/<module>/src/<Module>_Registry.txt`

---

**祝你阅读源码顺利！** 🚀

