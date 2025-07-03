# JPA系统函数参考手册

本文档详细描述了JPA自动工作点寻找与优化系统中所有函数的功能、参数和使用方法。

## 📚 目录

- [JPA器件调节原理](#JPA器件调节原理)
- [核心工具函数](#核心工具函数)
- [设备控制函数](#设备控制函数)
- [数据处理函数](#数据处理函数)
- [优化算法函数](#优化算法函数)
- [可视化函数](#可视化函数)
- [报告生成函数](#报告生成函数)

---

## JPA器件调节原理

一般情况下约瑟夫森参量放大器的可调节信号大概包括：
|参量名称|信号特征|信号行为|施加方式|
|:-:|:-:|:-:|:-:|
|MW_frequency|微波激励的频率|一般使用JPA频率的2倍频来进行泵浦，JPA一般设计在$7$ GHz附近，泵浦频率一般在$14$ GHz左右，且受到JPA偏置的影响|1. 微波源驱动，对应微波源的频率    2. DDS脉冲驱动，对应驱动脉冲的频率|
|MW_power|微波激励的功率|JPA的泵浦功率需要在合适的范围，才能让JPA工作，适合的功率可能有好几种状态|1. 微波源驱动，对应微波源输出的幅度    2. DDS脉冲驱动，对应驱动脉冲的振幅|
|DC_bias|直流偏置信号的电压|一般呈现出周期性的行为，会影响到JPA泵浦的频率|一般由直流源的偏移量（Offset）来给出|

因此，JPA的调节是一个典型的多参数联合调节的问题。对于多参数联合调节的问题，一般有两种通用思路：
- 多参数逐个扫描
  实际上是一定精度范围内枚举了可能的参数组合，通过JPA的性能评估函数或损失函数来选择较优范围。
- 多参数联合优化
  以迭代-反馈方式进行参数扫描，类似经典算法优化中的求极值优化问题，同样需要一个JPA性能评估函数或者损失函数。

那么如何来构建JPA性能评估函数或者是损失函数？构建方法是多样的，这里给出三个示例：
- JPA已经加入到含有比特的传输线中，且此时比特已经能做态的区分（`Scatter`）
  直接利用`Scatter`实验中比特态的iq信息，利用SVM算法中的多态的`visbility`作为性能评估。实际操作就是改变JPA可调节的三个参数，测量比特的`Scatter`实验，通过两团点甚至多团点分开的情况来选择合适的JPA工作参数。
- JPA没有加入到含有比特的传输线中
  此时没有比特信息作为参考，需要更广泛通用的评估方式。一般通过信号频率$\omega$在一定范围内的$S_{21}(\omega)$响应来判断。例如构建的损失函数是
$$f=\sum_\omega 20\log_{10}\left|\frac{{\rm Avg}[S_{21}(\omega)]}{{\rm Std}[S_{21}(\omega)]}\right|,$$
  此时平均（${\rm Avg}）$和标准差（${\rm Std}$）来自于一定次数的重复测量。这种方式构建的损失函数兼顾了信号的大小和信号的涨落。还可以减去一个额外的背景，例如采集一下在JPA不泵浦时的损失函数，通过JPA对损失函数的增益贡献来评价更好的JPA性能。
- JPA已经加入到含有比特的传输线中，但此时比特还不能做态的区分，但已经知道读取腔的频率
  此时损失函数可以考虑读取腔的特性，仅在读取腔频率附近来考虑评估函数。

## 核心工具函数

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
  
**注意事项**:
- 根据quark的版本不同，具体的实现方式可能略有差异。但这其实就是读设备的命令。

---

### `slowly_change(ed, st=None, per=0.01, channel='DC.CH1.Offset', write=False, slow=True, time_out=0.05, **kw)`

**功能**: 缓慢改变设备参数，避免突变对设备/JPA状态造成损害

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
- 避免设备损坏或者JPA敏感突变
- 参数扫描和优化
  
**注意事项**:
- 同样地，根据quark的版本不同，具体的实现方式可能略有差异。这就是写设备的命令。

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

#### 优化循环结构的伪代码

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
subprocess.run(['xelatex', f"{jpa_name}_report.tex"])
# 更建议使用Overleaf进行云编译，省去了配置tex文件编译环境的麻烦
# 使用Overleaf进行编译时，注意选择编译器为xeLatex
# 在打开的Project页面中，左上角的Menu菜单，点开可选择编译器（Setting-Compiler）
```

---

## 高级功能函数

### 损失函数设计实例

系统使用加权SNR作为优化目标：

```python
# 计算SNR
SNR = 20*np.log10(np.abs(s_pump_on.mean()/s_pump_on.std()))

# 应用频率权重，frequency_weight_function根据用户需求自定义
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

### 仪器状态检查

仪器的链接通过`quark`进行。需要在当前`quark`工作的`config`中的`dev`字段添加JPA测量使用到的设备。一个设备配置参考字典如下：

```python
{
    'dev': {
        'NA': {
            'addr': '192.168.103.220',
            'name': 'NetworkAnalyzer', # 请确认网分型号,
            'type': 'driver',
            'model': '3674C',
            'srate': 0,
            'inuse': True,
        },
        'MW': {
            'addr': '192.168.126.22',
            'name': 'PSG', # 请保证MW设备的driver是正常工作的
            'type': 'driver',
            'srate': 0,
            'inuse': True,
        },
        'DC': {
            'addr': '192.168.199.222',
            'name': 'RigolAWG', # 请保证DC设备的driver是正常工作的
            'type': 'driver',
            'srate': 0,
            'inuse': True,
        }
    }
}
```

---

## 性能优化建议

```python
# 并行测量多个参数，同时使用多台网分测量优化
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

*需要更多技术细节？查看源代码或联系开发团队。* 