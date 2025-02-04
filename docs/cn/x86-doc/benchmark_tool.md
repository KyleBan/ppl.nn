## X86 Benchmark 工具

x86架构性能测试使用pplnn工具。

本章仅介绍x86架构下使用pplnn测速的方法，cuda架构测速请参考：[Cuda Benchmark 工具](../cuda-doc/benchmark_tool.md)。

### 1. 编译

pplnn工具编译请参考[building-from-source.md](../../en/building-from-source.md)。

x86架构使用openmp作为线程池。若需要测试多线程性能，编译时请指定`-DHPCC_USE_OPENMP=ON`：

```bash
./build.sh -DHPCC_USE_OPENMP=ON
```

编译后pplnn工具的生成路径为：./pplnn-build/tools/pplnn

### 2. 准备测试数据

pplnn既可以生成随机数据，也可以从外部读入数据作为网络输入：

* 对于分类/语义分割等网络，网络的执行速度与输入数据的数值无关，测速时指定生成随机数据即可。
* 对于检测/实例分割等网络，网络的执行速度有可能会与输入数据的数值有关，因此建议从外部读入真实数据测试。

#### 2.1. 外部数据格式要求

当需要从外部读入测速数据时，推荐使用`--reshaped-inputs`选项来指定外部数据。

在此选项下，pplnn要求测试数据文件为二进制格式(可以用numpy的tofile函数将数据存为二进制格式)，网络中每个输入tensor需要单独存一个数据文件，文件命名方式为：

```
<tensor_name>-<input_shape>-<data_type>.dat
```

* \<tensor_name\>：对应onnx模型中输入tensor的名称，如：input
* \<input_shape\>：模型输入tensor的shape，以'\_'为分隔符，如：1_3_224_224
* \<data_type\>：文件的数据类型，目前支持fp64|fp32|fp16|int32|int64|bool

例如onnx模型中输入tensor的名称为input，shape为(1,3,224,224)，数据类型为float32，则数据文件命名应当为：

```
input-1_3_224_224-fp32.dat
```

### 3. 运行pplnn

#### 3.1. pplnn运行选项

pplnn中，与x86架构测速相关的运行选项有：

* `--onnx-model`：指定onnx模型文件
* `--reshaped-inputs`：指定外部数据，格式要求上文已阐述
* `--mm-policy`：内存管理策略，mem代表更少的内存使用，perf代表更激进的内存优化，默认为mem
* `--enable-profiling`：使能测速，默认为不使能
* `--min-profiling-time`：指定测速的最少持续时间，单位为秒，默认为1s
* `--warmuptimes`：指定warm up的次数，默认为0
* `--disable-avx512`：指定禁用avx512指令集，默认为启用
* `--core-binding`：启用绑核，默认不启用

#### 3.2. 环境变量设置

当编译指定了`-DHPCC_USE_OPENMP=ON`时，可使用环境变量`OMP_NUM_THREADS`来指定线程数：

```bash
export OMP_NUM_THREADS=8    # 指定8线程
```

#### 3.3. 使用随机数据测速

使用随机数据测速时，示例如下：

```bash
./pplnn --onnx-model <onnx_model> \   # 指定onnx模型
        --mm-policy mem           \   # 使用mem内存策略
        --enable-profiling        \   # 使能测速
        --min-profiling-time 10   \   # 测速时最少持续10s
        --warmuptimes 5           \   # warm up 5次
        --core-binding            \   # 启用绑核
        --disable-avx512              # 禁用avx512指令集
```

pplnn会自动根据模型输入的shape生成随机测试数据。

#### 3.4. 使用外部数据测速

外部数据格式要求见2.1节描述。

测速时，可使用如下命令测速：

```bash
./pplnn --onnx-model <onnx_model>                       \   # 指定onnx模型
        --reshaped-inputs input-1_3_224_224-fp32.dat    \   # 指定输入数据文件
        --mm-policy mem                                 \   # 使用mem内存策略
        --enable-profiling                              \   # 使能测速
        --min-profiling-time 10                         \   # 测速时最少持续10s
        --warmuptimes 5                                     # warm up 5次
```

当有多个输入时，`--reshaped-inputs`使用逗号','分割。

### 附录1. OpenPPL 在 10980XE 上的性能测试

平台信息：
 - 处理器：[Intel(R) Core(TM) i9-10980XE CPU @ 3.00GHz](https://ark.intel.com/content/www/us/en/ark/products/198017/intel-core-i9-10980xe-extreme-edition-processor-24-75m-cache-3-00-ghz.html "Intel(R) Core(TM) i9-10980XE CPU @ 3.00GHz")
 - 内存：4通道 DDR4 @ 3000MHz
 - 操作系统：Ubuntu 18.04 LTS
 - 工具链：GCC 7.5.0

#### 1. 单批/单线程 AVX512F 指令集

|  模型名称  |  OpenPPL  |  onnxruntime v1.8  |  OpenVINO r2021.2  |
| :------------ | :------------ | :------------ | :------------ |
|  deeplabv3plus  |  394.12  |  453.81  |  476.82  |
|  esrgan  |  1800.47  |  3216.80  |  3298.32  |
|  mobilenet_v2  |  4.45  |  5.41  |  5.04  |
|  resnet18  |  15.42  |  20.38  |  20.95  |
|  se_resnet50  |  45.68  |  56.87  |  53.08  |
|  densenet121  |  31.84  |  35.77  |  39.03  |
|  googlenet  |  12.64  |  16.32  |  16.91  |
|  inception_v3  |  28.84  |  36.31  |  36.01  |
|  mnasnet1_0  |  4.58  |  4.77  |  5.39  |
|  resnet34  |  30.70  |  40.98  |  41.80  |
|  resnet50  |  40.78  |  47.00  |  48.34  |
|  squeezenet1_1  |  3.83  |  4.31  |  4.27  |
|  vgg16  |  106.04  |  182.99  |  192.52  |
|  wide_resnet101_2  |  197.82  |  247.34  |  258.57  |

#### 2. 16批/16线程 AVX512F 指令集

|  模型名称  |  OpenPPL  |  onnxruntime v1.8  |  OpenVINO r2021.2  |
| :------------ | :------------ | :------------ | :------------ |
|  deeplabv3plus\*  |  423.09|  518.72  |  520.29  |
|  esrgan  |  无数据  |  无数据  |  无数据  |
|  mobilenet_v2  |  12.85  |  18.29  |  13.07  |
|  resnet18  |  20.85  |  28.14  |  26.05  |
|  se_resnet50  |  99.93  |  198.59  |  89.83  |
|  densenet121  |  89.49  |  168.49  |  93.11  |
|  googlenet  |  23.35  |  37.28  |  25.48  |
|  inception_v3  |  48.48  |  88.36  |  44.65  |
|  mnasnet1_0  |  12.12  |  11.13  |  11.24  |
|  resnet34  |  36.00  |  54.16  |  50.19  |
|  resnet50  |  65.37  |  72.69  |  67.27  |
|  squeezenet1_1  |  10.38  |  19.83  |  7.89  |
|  vgg16  |  107.52  |  236.27  |  233.77  |
|  wide_resnet101_2  |  259.18  |  344.04  |  321.00  |

\* deeplabv3plus 使用4批/4线程进行测试

#### 3. 单批/单线程 FMA3 指令集

|  模型名称  |  OpenPPL  |  onnxruntime v1.8  |  OpenVINO r2021.2  |  MindSpore Lite r1.3  |
| :------------ | :------------ | :------------ | :------------ | :------------ |
|  deeplabv3plus  |  630.45  |  919.51  |  836.47  |  938.73  |
|  esrgan  |  2428.81  |  5118.05  |  4686.33  |  3636.94  |
|  mobilenet_v2  |  6.84  |  7.56  |  7.06  |  7.54  |
|  resnet18  |  25.11  |  32.77  |  34.04  |  35.08  |
|  se_resnet50  |  72.38  |  87.49  |  82.97  |  91.65  |
|  densenet121  |  47.57  |  55.27  |  58.17  |  65.40  |
|  googlenet  |  19.99  |  26.95  |  28.01  |  25.58  |
|  inception_v3  |  48.89  |  55.19  |  56.08  |  不支持  |
|  mnasnet1_0  |  7.09  |  7.29  |  7.56  |  7.57  |
|  resnet34  |  46.33  |  65.74  |  69.38  |  67.63  |
|  resnet50  |  67.26  |  76.53  |  77.94  |  87.58  |
|  squeezenet1_1  |  5.91  |  6.64  |  6.76  |  6.60  |
|  vgg16  |  139.41  |  292.68  |  298.62  |  221.88  |
|  wide_resnet101_2  |  312.80  |  407.25  |  425.77  |  449.09  |

#### 4. 16批/16线程 FMA3 指令集

|  模型名称  |  OpenPPL  |  onnxruntime v1.8  |  OpenVINO r2021.2  |  MindSpore Lite r1.3  |
| :------------ | :------------ | :------------ | :------------ | :------------ |
|  deeplabv3plus\*  |  685.18  |  972.02  |  884.11  |  1129.83  |
|  esrgan  |  无数据  |  无数据  |  无数据  |  无数据  |
|  mobilenet_v2  |  14.87  |  18.33  |  9.29  |  24.57  |
|  resnet18  |  26.49  |  39.14  |  43.07  |  133.39  |
|  se_resnet50  |  118.81  |  230.49  |  142.70  |  323.55  |
|  densenet121  |  98.40  |  183.81  |  112.32  |  249.91  |
|  googlenet  |  27.82  |  45.97  |  37.03  |  73.40  |
|  inception_v3  |  61.70  |  104.37  |  69.58  |  不支持  |
|  mnasnet1_0  |  13.60  |  12.24  |  9.69  |  22.18  |
|  resnet34  |  44.43  |  77.27  |  83.74  |  273.40  |
|  resnet50  |  83.37  |  104.05  |  98.72  |  266.43  |
|  squeezenet1_1  |  12.44  |  20.02  |  10.30  |  27.87  |
|  vgg16  |  132.64  |  354.90  |  339.40  |  443.71  |
|  wide_resnet101_2  |  337.46  |  499.50  |  509.89  |  1527.74  |

\* deeplabv3plus 使用4批/4线程进行测试

### 附录2. OpenPPL 3700X 对比 10980XE

平台信息：
 - 处理器1：[AMD Ryzen 7 3700X 8-Core Processor](https://www.amd.com/zh-hans/products/cpu/amd-ryzen-7-3700x)
 - 内存1：2-channel DDR4 @ 3200MHz
 - 处理器2：[Intel(R) Core(TM) i9-10980XE CPU @ 3.00GHz](https://ark.intel.com/content/www/us/en/ark/products/198017/intel-core-i9-10980xe-extreme-edition-processor-24-75m-cache-3-00-ghz.html "Intel(R) Core(TM) i9-10980XE CPU @ 3.00GHz")
 - 内存2：4-channel DDR4 @ 3000MHz
 - 操作系统: Ubuntu 18.04 LTS
 - 工具链: GCC 7.5.0
 - 指令集: FMA3

#### 1. 单批1到8线程

|  模型名称  |  3700X 单线程  |  10980XE 单线程  |  3700X 2线程  |  10980XE 2线程  |  3700X 4线程  |  10980XE 4线程  |  3700X 8线程  |  10980XE 8线程  |
| :------------ | :------------ | :------------ | :------------ | :------------ | :------------ | :------------ | :------------ | :------------ |
|  deeplabv3plus  |  610.78  |  674.02  |  280.42  |  321.09  |  148.20  |  174.57  |  90.13  |  94.43  |
|  esrgan  |  1791.85  |  2459.53  |  982.46  |  1289.04  |  643.06  |  812.01  |  655.05  |  473.52  |
|  mobilenet_v2  |  6.14  |  6.77  |  3.79  |  4.26  |  2.25  |  2.51  |  2.16  |  2.30  |
|  resnet18  |  18.17  |  25.59  |  10.01  |  13.83  |  6.98  |  7.80  |  6.99  |  5.04  |
|  se_resnet50  |  57.75  |  72.36  |  30.53  |  38.20  |  18.39  |  21.39  |  15.84  |  14.56  |
|  densenet121  |  38.48  |  47.57  |  21.98  |  28.44  |  13.14  |  16.49  |  13.97  |  13.82  |
|  googlenet  |  16.54  |  19.85  |  9.43  |  11.72  |  5.64  |  6.71  |  5.67  |  4.92  |
|  inception_v3  |  38.78  |  48.32  |  22.55  |  28.53  |  13.02  |  16.59  |  11.69  |  12.18  |
|  mnasnet1_0  |  6.31  |  7.05  |  3.78  |  4.24  |  2.25  |  2.55  |  2.10  |  2.28  |
|  resnet34  |  31.33  |  47.61  |  17.41  |  25.74  |  11.89  |  14.37  |  12.44  |  8.98  |
|  resnet50  |  55.73  |  67.17  |  29.27  |  35.58  |  17.32  |  20.06  |  14.38  |  13.07  |
|  squeezenet1_1  |  5.29  |  5.96  |  2.87  |  3.41  |  1.55  |  2.01  |  1.60  |  1.62  |
|  vgg16  |  97.64  |  143.35  |  53.94  |  75.22  |  35.10  |  42.01  |  31.22  |  24.61  |
|  wide_resnet101_2  |  247.82  |  313.41  |  131.10  |  166.45  |  77.52  |  89.30  |  64.17  |  54.73  |

#### 2. 16批1到8线程

|  模型名称  |  3700X 单线程  |  10980XE 单线程  |  3700X 2线程  |  10980XE 2线程  |  3700X 4线程  |  10980XE 4线程  |  3700X 8线程  |  10980XE 8线程  |
| :------------ | :------------ | :------------ | :------------ | :------------ | :------------ | :------------ | :------------ | :------------ |
|  deeplabv3plus\*  |  2314.52  |  2646.51  |  1132.35  |  1299.84  |  604.27  |  683.62  |  337.72  |  357.54  |
|  esrgan  |  无数据  |  无数据  |  无数据  |  无数据  |  无数据  |  无数据  |  无数据  |  无数据  |
|  mobilenet_v2  |  99.76  |  115.57  |  53.04  |  61.05  |  34.24  |  33.71  |  30.91  |  20.53  |
|  resnet18  |  240.29  |  279.60  |  122.91  |  142.06  |  66.82  |  75.48  |  42.14  |  40.27  |
|  se_resnet50  |  938.73  |  1057.31  |  492.15  |  549.39  |  297.04  |  297.67  |  211.17  |  169.26  |
|  densenet121  |  690.09  |  789.15  |  388.93  |  415.81  |  258.43  |  232.00  |  188.48  |  136.29  |
|  googlenet  |  256.14  |  288.49  |  135.52  |  150.42  |  79.86  |  81.30  |  56.85  |  45.29  |
|  inception_v3  |  581.54  |  667.63  |  307.68  |  346.93  |  172.80  |  186.82  |  106.53  |  102.00  |
|  mnasnet1_0  |  101.10  |  118.79  |  54.10  |  62.30  |  33.40  |  34.60  |  28.15  |  20.84  |
|  resnet34  |  405.41  |  495.02  |  208.81  |  250.61  |  113.08  |  133.37  |  72.73  |  71.14  |
|  resnet50  |  839.27  |  946.68  |  539.82  |  482.61  |  289.67  |  255.34  |  137.18  |  136.48  |
|  squeezenet1_1  |  92.39  |  105.33  |  49.00  |  55.29  |  32.16  |  30.60  |  27.57  |  18.38  |
|  vgg16  |  1288.17  |  1654.04  |  665.49  |  845.69  |  362.35  |  421.77  |  231.20  |  229.89  |
|  wide_resnet101_2  |  3604.05  |  4171.32  |  1841.64  |  2107.49  |  966.50  |  1099.83  |  547.85  |  575.81  |

\* deeplabv3plus 使用4批进行测试
