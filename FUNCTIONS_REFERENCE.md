# JPA系统函数参考手册

本文档详细描述了JPA自动工作点寻找与优化系统中所有函数的功能、参数和使用方法。

## 📚 目录

- [核心工具函数](#核心工具函数)
- [设备控制函数](#设备控制函数)
- [数据处理函数](#数据处理函数)
- [优化算法函数](#优化算法函数)
- [可视化函数](#可视化函数)
- [报告生成函数](#报告生成函数)

---

## 核心工具函数

### `extent(A, B)`

**功能**: 计算二维数据的边界范围，用于matplotlib的imshow等函数

**参数**:
- `A`: numpy数组，x轴数据范围
- `B`: numpy数组，y轴数据范围

**返回**: 
- tuple: `(xmin, xmax, ymin, ymax)` 用于设置图像边界

**使用示例**:
```python
x = np.linspace(0, 10, 11)
y = np.linspace(0, 5, 6)
ext = extent(x, y)
plt.imshow(data, extent=ext)
```

**应用场景**: 
- 生成二维扫描图
- 设置colormap的坐标范围

---

### `server_read(channel)`

**功能**: 从Quark服务器读取指定通道的数据

**参数**:
- `channel`: 字符串，设备通道名称（格式：设备名.通道名.参数名）

**返回**: 
- 对应通道的数值（类型取决于参数）

**使用示例**:
```python
# 读取网络分析仪功率
power = server_read('na_103_220.CH1.Power')

# 读取直流源偏置
bias = server_read('Rigol_199_222.CH1.Offset')

# 读取S参数数据
s_data = server_read('na_103_220.CH1.S')
```

**应用场景**:
- 获取设备当前状态
- 读取测量数据
- 监控设备参数

---

### `slowly_change(ed, st=None, per=0.01, channel='DC.CH1.Offset', write=False, slow=True, time_out=0.05, **kw)`

**功能**: 缓慢改变设备参数，避免突变对设备造成损害

**参数**:
- `ed`: 目标值
- `st`: 起始值（None时自动读取当前值）
- `per`: 每步变化量
- `channel`: 设备通道
- `write`: 是否实际写入设备（False时仅打印）
- `slow`: 是否使用缓慢变化模式
- `time_out`: 每步之间的等待时间（秒）

**使用示例**:
```python
# 缓慢调整直流偏置从当前值到0.1V
slowly_change(0.1, channel='Rigol_199_222.CH1.Offset', write=True)

# 快速设置微波功率
slowly_change(-10, slow=False, channel='mw_126_22.CH1.Power', write=True)

# 测试模式（不实际写入）
slowly_change(5e9, channel='mw_126_22.CH1.Frequency', write=False)
```

**应用场景**:
- 安全调整敏感参数
- 避免设备损坏
- 参数扫描和优化

---

## 数据处理函数

### `get_norm(x)`

**功能**: 将数据归一化到[0.05, 0.95]范围，用于颜色映射

**参数**:
- `x`: numpy数组，输入数据

**返回**: 
- numpy数组，归一化后的数据

**使用示例**:
```python
data = np.random.randn(100)
normalized = get_norm(data)
colors = plt.cm.viridis(normalized)
```

**应用场景**:
- 数据可视化
- 颜色映射
- 图表美化

---

### `get_repeat_s(channel, repeat, _time_out=0.01, bar=True)`

**功能**: 重复测量S参数数据，用于统计分析

**参数**:
- `channel`: 测量通道
- `repeat`: 重复次数
- `_time_out`: 每次测量间隔（秒）
- `bar`: 是否显示进度条

**返回**: 
- numpy数组，形状为`(repeat, 频率点数)`

**使用示例**:
```python
# 重复测量40次，显示进度条
s_data = get_repeat_s('na_103_220.CH1', repeat=40)

# 快速测量10次，不显示进度条
s_data = get_repeat_s('na_103_220.CH1', repeat=10, bar=False)

# 计算平均值和标准差
s_mean = s_data.mean(axis=0)
s_std = s_data.std(axis=0)
```

**应用场景**:
- 噪声评估
- 统计分析
- 信噪比计算

---

### `from_s_get_logabs_db(s)`

**功能**: 将S参数转换为dB单位的幅度

**参数**:
- `s`: 复数S参数数组

**返回**: 
- numpy数组，以dB为单位的幅度

**使用示例**:
```python
# 复数S21数据
s21_complex = s_data.mean(axis=0)

# 转换为dB
s21_db = from_s_get_logabs_db(s21_complex)

# 绘制传输曲线
plt.plot(freqs/1e9, s21_db)
plt.ylabel('|S21| (dB)')
```

**应用场景**:
- 增益分析
- 传输特性评估
- 频率响应绘制

---

## 可视化函数

### `draw_JPA_text_circuit()`

**功能**: 绘制JPA测试电路的示意图

**参数**: 无（从全局变量`JPA_var_config`读取配置）

**返回**: 
- `fig, ax`: matplotlib图形对象

**功能特性**:
- 自动读取配置中的衰减器设置
- 显示稀释制冷机的温度平台
- 标注设备型号和连接关系
- 生成用于报告的专业图表

**使用示例**:
```python
# 绘制电路图
fig, ax = draw_JPA_text_circuit()

# 保存为PDF
fig.savefig('JPA_circuit_diagram.pdf')

# 调整图片大小
fig.set_size_inches(12, 4)
```

**应用场景**:
- 报告文档
- 实验记录
- 配置验证

---

## 优化算法相关

### 优化配置结构

系统使用`nevergrad`库进行多参数优化，主要类和函数：

#### `NgOpt(variables, config)`

**功能**: 创建优化器实例

**参数**:
- `variables`: 优化变量列表，每个变量为`VarData`对象
- `config`: 优化配置字典

**使用示例**:
```python
from nevergrad import VarData

# 定义优化变量
variables = [
    VarData('add', init_freq, low_freq, high_freq),
    VarData('add', init_power, low_power, high_power),
    VarData('add', init_bias, low_bias, high_bias)
]

# 创建优化器
optimizer = NgOpt(variables, {'method': 'TwoPointsDE', 'config': {'budget': 20}})
```

#### 优化循环结构

```python
# 标准优化循环
for iteration in range(budget):
    # 询问下一个测试点
    x = optimizer.ask()
    
    # 设置参数并测量
    set_parameters(x)
    y = measure_performance()
    
    # 告知结果（注意：最小化目标，所以传入-y）
    optimizer.tell(x, -y)
```

---

## 报告生成函数

### `get_report(tester_name='李四')`

**功能**: 生成LaTeX格式的测试报告

**参数**:
- `tester_name`: 测试人员姓名

**返回**: 
- 字符串，完整的LaTeX报告代码

**报告内容**:
- 器件基本信息
- 测试环境描述
- 性能指标汇总
- 测试结果图表
- 参数对比表格

**使用示例**:
```python
# 生成报告
report_latex = get_report(tester_name='张三')

# 保存为tex文件
with open(f"{jpa_name}_report.tex", 'w') as f:
    f.write(report_latex)

# 编译PDF（需要LaTeX环境）
import subprocess
subprocess.run(['pdflatex', f"{jpa_name}_report.tex"])
```

---

## 高级功能函数

### 损失函数设计

系统使用加权SNR作为优化目标：

```python
# 计算SNR
SNR = 20*np.log10(np.abs(s_pump_on.mean()/s_pump_on.std()))

# 应用频率权重
weighted_SNR = SNR @ frequency_weight_function(frequencies)

# 最终损失（注意：优化器最小化，所以用负值）
loss = -weighted_SNR
```

### 自定义权重函数示例

```python
# 单频带权重
weight_func = lambda f: np.piecewise(f, [(6.5e9<=f)*(f<=7.5e9)], [1, 0])

# 多频带权重
def multi_band_weight(f):
    return (np.piecewise(f, [(6.0e9<=f)*(f<=7.0e9)], [0.5, 0]) + 
            np.piecewise(f, [(7.0e9<=f)*(f<=8.0e9)], [1.0, 0]))

# 高斯权重
def gaussian_weight(f, center=7e9, sigma=0.5e9):
    return np.exp(-((f-center)/sigma)**2)
```

---

## 错误处理和调试

### 常用调试函数

```python
def check_device_status():
    """检查所有设备的连接状态"""
    devices = ['na_103_220.CH1', 'mw_126_22.CH1', 'Rigol_199_222.CH1']
    for device in devices:
        try:
            status = server_read(f"{device}.Status")
            print(f"{device}: OK ({status})")
        except Exception as e:
            print(f"{device}: ERROR - {e}")

def validate_measurement(s_data):
    """验证测量数据的有效性"""
    if np.any(np.isnan(s_data)):
        raise ValueError("测量数据包含NaN值")
    if np.any(np.isinf(s_data)):
        raise ValueError("测量数据包含无穷大值")
    if s_data.shape[0] == 0:
        raise ValueError("测量数据为空")
    print(f"数据验证通过：{s_data.shape[0]}次测量，{s_data.shape[1]}个频率点")
```

---

## 性能优化建议

### 1. 测量速度优化
```python
# 并行测量多个参数
def quick_measurement(channels, repeats=10):
    results = {}
    for channel in channels:
        results[channel] = get_repeat_s(channel, repeats, bar=False)
    return results

# 自适应重复次数
def adaptive_measurement(channel, target_snr=30):
    repeats = 10
    while True:
        data = get_repeat_s(channel, repeats, bar=False)
        snr = 20*np.log10(np.abs(data.mean()/data.std()))
        if np.mean(snr) > target_snr or repeats > 100:
            break
        repeats += 10
    return data
```

### 2. 内存优化
```python
# 流式数据处理
def streaming_measurement(channel, total_repeats, batch_size=20):
    results = []
    for i in range(0, total_repeats, batch_size):
        batch_data = get_repeat_s(channel, 
                                min(batch_size, total_repeats-i), 
                                bar=False)
        # 处理批次数据
        batch_mean = batch_data.mean(axis=0)
        results.append(batch_mean)
        # 清理内存
        del batch_data
    return np.array(results)
```

---

*需要更多技术细节？查看源代码或联系开发团队。* 