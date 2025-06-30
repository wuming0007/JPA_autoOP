# JPA自动工作点寻找系统 - 快速开始指南

## 🚀 5分钟快速开始

本指南将帮助您在5分钟内完成JPA测试系统的基本设置并运行第一次测试。

## 📋 前置条件检查

### 1. 硬件准备
- [ ] 稀释制冷机已降温至基温（~10mK）
- [ ] JPA器件已正确安装在MXC平台
- [ ] 测试线路连接完成（输入线、输出线、泵浦线）
- [ ] 所有测试设备已开机并预热30分钟以上

### 2. 软件环境
- [ ] Python 3.8+ 环境
- [ ] Jupyter Notebook 或 JupyterLab
- [ ] 网络连接到实验室内网（访问Quark服务器）

## ⚡ 快速安装

### 步骤1: 环境设置
```bash
# 1. 进入工作目录
cd /path/to/JPA_autoOP

# 2. 检查Python版本
python --version  # 应该 >= 3.8

# 3. 安装依赖包
pip install numpy matplotlib scipy tqdm nevergrad
```

### 步骤2: 验证设备连接
```python
# 在Python中快速测试
from quark.app import login
s = login('baqis')
s.start()

# 测试设备连接
print("VNA连接:", s.read('na_103_220.CH1.Power'))
print("MW源连接:", s.read('mw_126_22.CH1.Power'))
print("DC源连接:", s.read('Rigol_199_222.CH1.Offset'))
```

## 🎯 首次运行

### 步骤1: 打开Notebook
```bash
# 启动Jupyter
jupyter notebook

# 或使用JupyterLab
jupyter lab
```

在浏览器中打开 `JPA_NA_OPT_TEX_annotated.ipynb`

### 步骤2: 配置您的JPA
**重要**: 在运行测试前，必须修改配置以匹配您的JPA器件！

找到第25个cell中的`JPA_var_config`，修改以下关键参数：

```python
JPA_var_config = {
    # 修改为您的JPA名称
    'jpa_name': 'JPA20241201_YourSample_01',
    
    # 根据您的测试线路修改衰减器配置
    'wiring': {
        'att': {
            'Input line': [40, 0, 0, 20, 0, None],    # 根据实际衰减器修改
            'Output line': [None, None, None, 0, 0, None],
            'Pump line': [None, 0, 0, 20, 0, None],
        },
        # 其他配置...
    },
    
    # 根据您的JPA特性调整优化参数
    'opt_params': {
        'mw_frequency': {
            'init': 14.0e9,      # 调整为您的JPA工作频率的2倍
            # 其他参数...
        },
        'dc_bias': {
            'init': 0.0,         # 调整为合理的初始偏置
            # 其他参数...
        },
        # 其他参数...
    },
}
```

### 步骤3: 执行测试
**按顺序执行以下关键步骤：**

1. **初始化** (Cells 1-3)
   ```python
   # Cell 1: 导入库和设置
   # Cell 2: 连接Quark服务器
   # Cell 3: 启动服务器
   ```

2. **设备配置** (Cells 6-23)
   ```python
   # 设置网络分析仪、直流源、微波源参数
   # 这些参数已经在配置中定义，通常不需要修改
   ```

3. **执行偏置扫描** (Cells 42-46)
   ```python
   # Cell 43: 执行偏置扫描（大约5-10分钟）
   # Cell 44: 生成SNR vs 偏置图
   # Cell 45: 选择最优偏置点（需要手动点击图中的点）
   ```

4. **工作点优化** (Cells 62-68)
   ```python
   # Cell 63: 多参数优化（大约15-30分钟）
   # Cell 66: 设置最优参数
   # Cell 68: 验证优化结果
   ```

## 📊 理解输出结果

### 关键指标解读

运行完成后，在`JPA_var_config['report']`中查看结果：

```python
# 查看测试结果
print("测试结果:")
print(f"增益: {JPA_var_config['report']['gain']:.1f} dB")
print(f"带宽: {JPA_var_config['report']['bandwith']/1e6:.0f} MHz")
print(f"工作频率: {JPA_var_config['report']['range'][0]/1e9:.2f} - {JPA_var_config['report']['range'][1]/1e9:.2f} GHz")
```

### 图表文件
测试完成后会生成以下文件：
- `{JPA_name}_Scan_SNR_vs_Bias.pdf` - 偏置扫描结果
- `{JPA_name}_Opt_Gain.pdf` - 增益优化结果
- `JPA_test_circuit.pdf` - 电路配置图

## ⚠️ 常见问题快速解决

### Q1: 连接Quark服务器失败
```python
# 检查网络连接
ping baqis-server  # 替换为实际服务器地址

# 检查登录信息
s = login('baqis')  # 确认登录名正确
```

### Q2: 设备通道错误
```python
# 查看可用设备
s.list_devices()

# 检查设备状态
s.read('na_103_220.CH1.Status')
```

### Q3: 优化结果不理想
1. **检查初始参数设置**
   - 确认泵浦频率接近信号频率的2倍
   - 确认偏置范围包含最优点

2. **调整优化设置**
   ```python
   # 增加优化预算
   'opt_config': {'budget': 40},  # 从20增加到40
   
   # 增加测量重复次数
   'lost_setting': {'repeat': 60},  # 从40增加到60
   ```

### Q4: 测量噪声过大
```python
# 降低网分带宽
'lost_setting': {
    'params': {
        'Bandwidth': 1000,  # 从5000降到1000Hz
    }
}

# 增加等待时间
'time_out': 1.0,  # 从0.5增加到1.0秒
```

## 🔄 典型工作流程

### 每日测试例程
1. **开机检查** (5分钟)
   - 检查制冷机温度
   - 验证设备连接
   - 运行设备自检

2. **快速验证** (10分钟)
   - 执行单点测量
   - 确认JPA基本响应
   - 检查信号质量

3. **完整测试** (60分钟)
   - 偏置扫描 (10分钟)
   - 参数优化 (30分钟)
   - 性能评估 (20分钟)

### 测试数据管理
```python
# 保存配置
import json
with open(f"{JPA_var_config['jpa_name']}_config.json", 'w') as f:
    json.dump(JPA_var_config, f, indent=2, default=str)

# 备份数据
import shutil
shutil.copy('JPA_NA_OPT_TEX_annotated.ipynb', 
           f"{JPA_var_config['jpa_name']}_backup.ipynb")
```

## 📞 获取帮助

### 技术支持
- **实验室内部**: 联系设备负责人
- **软件问题**: 查看[故障排除指南](README.md#故障排除)
- **硬件问题**: 检查设备手册

### 进阶功能
- [自定义优化目标](README.md#自定义优化目标)
- [批量测试脚本](docs/batch_testing.md)
- [数据分析工具](docs/data_analysis.md)

---

## 🎉 成功标志

如果您看到以下结果，说明系统运行正常：
- ✅ 增益 > 10 dB
- ✅ 带宽 > 100 MHz  
- ✅ 优化过程收敛
- ✅ 生成完整的测试报告

恭喜！您已经成功掌握了JPA自动测试系统的基本使用。

---

*需要更多帮助？查看完整的[用户手册](README.md)或联系技术支持。* 