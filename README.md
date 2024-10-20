# MCGPULite_v1.5使⽤⼿册

项目地址：https://github.com/SEU-CT-Recon/MCGPULite_v1.5?tab=readme-ov-file

- [MCGPULite_v1.5使⽤⼿册](#mcgpulite-v15----)
  * [简介](#--)
  * [代码的一些主要特点](#---------)
  * [MCGPULite_v1.5 vs MCGPULite](#mcgpulite-v15-vs-mcgpulite)
  * [MCGPULite_v1.5 vs VICTRE_MCGPU](#mcgpulite-v15-vs-victre-mcgpu)
  * [MCGPULite 环境配置](#mcgpulite-----)
  * [输⼊⽂件与参数](#-------)
    + [输入文件准备](#------)
      - [x-ray能谱⽂件](#x-ray----)
      - [体素化几何模型文件](#---------)
        * [材料文件](#----)
    + [输入参数说明](#------)
      - [1. 模拟配置 SIMULATION CONFIG](#1------simulation-config)
      - [2. x射线源 SOURCE](#2-x----source)
      - [3. 图像探测器 IMAGE DETECTOR](#3-------image-detector)
      - [4. 扫描轨迹 SCAN TRAJECTORY](#4------scan-trajectory)
      - [5. 模型文件 VOXELIZED GEOMETRY FILE](#5------voxelized-geometry-file)
      - [6. 材料文件列表 MATERIAL FILE LIST](#6--------material-file-list)
  * [运行](#--)
  * [输出文件](#----)
  * [已知的问题](#-----)
    + [亮场模拟bug](#----bug)

## 简介

MCGPU主要⽤途是利⽤GPU多线程计算能⼒，模拟在相关模型中传输X射线，⽣成（⼈体解剖学）真实模型的合成放射图像。

MCGPULite_v1.5是https://github.com/DIDSR/VICTRE_MCGPU/的轻量版本，**只允许在Linux系统运行**。

其他版本入口:

- [MCGPULite](https://github.com/SEU-CT-Recon/MCGPULite):MCGPU_v1.3的轻量版本，只允许在Linux系统运行。

- [MCGPU-for-MATLAB](https://github.com/SEU-CT-Recon/MCGPU-for-MATLAB):MCGPU_v1.3的MATLAB接口版本，可以直接在Windows上运行

此使⽤⼿册为作者学习使⽤该程序时整理，对于其中内容的理解还较为粗浅，若有错误的地⽅欢迎批评指正。

## 代码的一些主要特点

- 代码的基本操作包括以下几项任务：调整仿真输入`.in`文件以描述X射线源的位置和特征，定义CT扫描轨迹（如有），列出仿真中使用的材料，定义X射线探测器的几何结构，以及指定体素化对象`vox`文件。
- 仿真系统的坐标系由输入的体素化几何结构确定。坐标原点假设位于体素化体积的后下角，并且坐标轴位于体素化体积的顶点上。这意味着第一个体素的后下角位于原点，后续的体素沿正X、Y和Z轴方向排列（第一象限）。
- MC-GPU可以模拟单个投影图像或完整的CT扫描。CT扫描是通过围绕静态体素化几何形状生成许多投影图像来模拟的。
- 输出的文件不再包含ASCII格式文件，但会保存每张投影的模拟报告，需要用imageJ打开。

## MCGPULite_v1.5 vs MCGPULite

相比于https://github.com/SEU-CT-Recon/MCGPULite，MCGPULite_v1.5新增了一些改进：

- 可以自行决定旋转轴是x轴、y轴还是z轴，而不再仅限于执行围绕Z轴简单的CT轨迹旋转  
- 可以通过简单地填写角度的方式决定第一张投影图的初始旋转角度，而不是计算初始入射源的位置（方向余弦）或是修改模体文件
- 可以修改X射线束相对于探测器中心的偏移
- 可选择的滤线珊模型
- 可选择的保护壳/盖板模型
- 可以选择在内存中使用二叉树结构存储的vox模体以节省内存（如果vox模体非常大（体素数量大于**2^31**））【**目前暂不适用，后续会进行修复**】

## MCGPULite_v1.5 vs VICTRE_MCGPU

相比于https://github.com/DIDSR/VICTRE_MCGPU/，MCGPULite_v1.5做了一下删减和修改：

- [Makefile](./Makefile) 已被重写，以支持独立的 zlib、CUDA、nvcc 和 openmpi，因此不再需要 sudo apt-get。并与https://github.com/SEU-CT-Recon/MCGPULite同步。

- 删除了focal spot 模型，`.in`输入文件不再需要focal spot的相关参数：

  ```diff
  #[SECTION SOURCE v.2016-12-02]
  ...
  - 0.0300                 # SOURCE GAUSSIAN FOCAL SPOT FWHM [cm]
  - 0.18                   # 0.18 for DBT, 0 for FFDM [Mackenzie2017]  # ANGULAR BLUR DUE TO MOVEMENT ([exposure_time]*[angular_speed]) [degrees]
  - YES                     # COLLIMATE BEAM TOWARDS POSITIVE AZIMUTHAL (X) ANGLES ONLY? (ie, cone-beam center aligned with chest wall in mammography) [YES/NO]
  ```

- 删除了探测器模型中关于荧光逃逸、荷电生成、斯旺克因子、电子噪声的参数：

  ```diff
   #[SECTION IMAGE DETECTOR v.2017-06-20]
   ...
  - 12658.0 11223.0 0.596 0.00593  # DETECTOR K-EDGE ENERGY [eV], K-FLUORESCENCE ENERGY [eV], K-FLUORESCENCE YIELD, MFP AT FLUORESCENCE ENERGY [cm]
  - 50.0    0.99                   # EFECTIVE DETECTOR GAIN, W_+- [eV/ehp], AND SWANK FACTOR (input 0 to report ideal energy fluence)
  - 5200.0                         # ADDITIVE ELECTRONIC NOISE LEVEL (electrons/pixel)
  ```

- 默认探测器不固定（多角度投影时）以及默认不同时模拟0度投影和断层扫描，以适配CBCT的情况：

  ```diff
  #[SECTION TOMOGRAPHIC TRAJECTORY v.2016-12-02]
  ...
  - YES                             # KEEP DETECTOR FIXED AT 0 DEGREES FOR DBT? [YES/NO]
  - YES                             # SIMULATE BOTH 0 deg PROJECTION AND TOMOGRAPHIC SCAN (WITHOUT GRID) WITH 2/3 TOTAL NUM HIST IN 1st PROJ (eg, DBT+mammo)? [YES/NO]
  ```

- 剂量信息不再报告，这与https://github.com/SEU-CT-Recon/MCGPULite保持相同：

  ```diff
  #[SECTION TOMOGRAPHIC TRAJECTORY v.2016-12-02]
  ...
  
  - #[SECTION DOSE DEPOSITION v.2012-12-12]
  ...
  
  #[SECTION VOXELIZED GEOMETRY FILE v.2017-07-26]
  ```

- 模体文件目前只支持文档中带header的格式，且暂时禁用了二叉树存储格式，

  ```diff
  #[SECTION VOXELIZED GEOMETRY FILE v.2017-07-26]
  ...
  - 1280   1950   940              # NUMBER OF VOXELS: INPUT A 0 TO READ ASCII FORMAT WITH HEADER SECTION, RAW VOXELS WILL BE READ OTHERWISE
  - 0.0050 0.0050 0.0050           # VOXEL SIZES [cm]
  - 1 1 1                          # SIZE OF LOW RESOLUTION VOXELS THAT WILL BE DESCRIBED BY A BINARY TREE, GIVEN AS POWERS OF TWO (eg, 2 2 3 = 2^2x2^2x2^3 = 128 input voxels per low res voxel; 0 0 0 disables tree)
  ```

  MCGPU_v1.5b将材料的密度固定成某一个值，这样的话如果vox文件中同个材料不同密度的情况会变得无效，因此MCGPULite_v1.5取消了这个限制，但是这似乎会影响到二叉树存储的功能，因此目前暂时禁用了二叉树存储的功能。

- 材料文件不再允许用户自定义密度和编号，因为发现不适用于带header的vox文件 

  ```diff
  #[SECTION MATERIAL FILE LIST v.2020-03-03]   
  -#  -- Input material file names first, then material density after keyword 'density=' (optional if using nominal density), then comma-separated list of voxel ID numbers after keyword 'voxelID=' (empty if material not used).
  - air__5-120keV.mcgpu                  density=0.0012   voxelId=0        
  adipose__5-120keV.mcgpu                      
  ```

  

- 输出的文件包含三个切片：total信号、primary信号以及散射信号，而不是只包含total信号和primary信号，这与https://github.com/SEU-CT-Recon/MCGPULite保持相同。

## MCGPULite 环境配置

**注：以下的环境配置都是基于linux系统**。

- 安装cuda

- 安装[zlib](https://www.zlib.net/zlib-1.2.12.tar.gz)和[openmpi](https://download.open-mpi.org/release/open-mpi/v4.1/openmpi-4.1.1.tar.gz)，分别解压并进⾏编译

  ```makefile
  # 进⼊下载的zlib压缩包所在⽬录
  tar -zxvf zlib-1.2.12.tar.gz # 解压缩
  cd zlib-1.2.12
  mkdir install # 创建安装路径
  ./configure --prefix=xxx/zlib-1.2.12/install/ # 配置：指定安装绝对路径
  make # 编译
  make install # 安装
  # 进⼊下载的openmpi压缩包所在⽬录
  tar -zxvf openmpi-4.1.1.tar.gz
  cd openmpi-4.1.1
  mkdir install
  ./configure --prefix=xxx/openmpi-4.1.1/install/
  make
  make install
  # Makefile⽂件已重写，⽀持独⽴的zlib、CUDA、nvcc和openmpi，因此不需要使⽤sudo apt-get进⾏安装。
  ```

- **根据⾃⼰的计算机**修改 Makefile 中的路径

  ```makefile
  NVCC = /usr/local/cuda-10.0/bin/nvcc
  CUDA_INCLUDE = /usr/local/cuda-10.0/include/
  CUDA_LIB = /usr/local/cuda-10.0/lib64/
  CUDA_SDK_INCLUDE = /usr/local/cuda-10.0/samples/common/inc/
  CUDA_SDK_LIB = /usr/local/cuda-10.0/samples/common/lib/linux/x86_64/
  OPENMPI_INCLUDE = /home/zhuoxu/app/openmpi-4.1.1/install/include/
  OPENMPI_LIB = /home/zhuoxu/app/openmpi-4.1.1/install/lib/
  ZLIB_INCLUDE = /home/zhuoxu/app/zlib-1.2.12/install/include/
  ZLIB_LIB = /home/zhuoxu/app/zlib-1.2.12/install/lib/
  ```

  

- 编译MCGPULite

  ```makefile
  make clean
  make
  ```

- 如果需要，可以创建符号链接并将其添加到 PATH 中，**注意链接命名不要与MCGPULite重复**

  ```makefile
  ln -s MCGPULite_v1.5.x MCGPULite1.5
  vim ~/.bashrc
  # 在编辑器中插⼊下⾯这条语句
  export PATH="/path/to/MCGPULite/folder/:$PATH"
  # 保存并退出编辑器
  source ~/.bashrc
  ```

## 输⼊⽂件与参数

本程序的输⼊⽂件路径和输⼊参数集中在了`.in`⽂件中，相关模板为MCGPULite的`MCGPULite_v1.3.sample.in`或MCGPU-for-MATLAB的`MCGPU_MATLAB_sample.in`，后者相较于前者多了一小部分内容，本节将会全部提及。

### 输入文件准备

#### x-ray能谱⽂件

x 射线源定义为发射 x 射线的点源，其能量从⽤户提供的能谱中随机采样。

关于能谱⽂件的产⽣，可以使⽤程序https://github.com/I-STAR/SPEKTR，在matlab中全部添加路径后运⾏spektr.m⽂件即可。

![0]([.\assets](https://github.com/IIIIIsak/MCGPULite_v1.5.io/tree/main/assets/0.png)

在区域1更改参数可以形成不同的能谱（其他参数可⾃⾏探索），点击按钮Generate Spectrum产⽣能谱，点击按钮Save保存能谱⽂件为txt：

![1](https://github.com/IIIIIsak/MCGPULite_v1.5.io/tree/main/assets/1.png)

**根据MCGPULite的输⼊要求，需要做以下更改：**

1. 能量范围需要根据材料的要求进⾏更改，本项⽬中设置为5 ~120keV，且单位改为eV

2. 去除单位光⼦数（Photons / mm^2 / mAs）为0的⾏，并在最后添加⼀⾏单位光⼦数为负值的项标志能谱⽂件的结束。

经过修改后数据如下（仅展⽰头尾）:

![2](https://github.com/IIIIIsak/MCGPULite_v1.5.io/blob/main/assets/2.png)



![3](https://github.com/IIIIIsak/MCGPULite_v1.5.io/tree/main/assets/3.png)

#### 体素化几何模型文件

体素化几何模型是将每层CT图的各个部位填入相对应的编号和密度，需要按照如下⽰例的格式进⾏创建：

```c
[SECTION VOXELS HEADER v.2008-04-13]
411  190  113      No. OF VOXELS IN X,Y,Z
5.000e-02  5.000e-02  5.000e-02    VOXEL SIZE (cm) ALONG X,Y,Z
1                  COLUMN NUMBER WHERE MATERIAL ID IS LOCATED
2                  COLUMN NUMBER WHERE THE MASS DENSITY IS LOCATED
1                  BLANK LINES AT END OF X,Y-CYCLES (1=YES,0=NO)
[END OF VXH SECTION]
1 0.00120479
1 0.00120479
...
```

文件头信息中，有五个参数，依次为体素（体积元素）在X、Y、Z轴上的数量，单个体素在X、Y、Z轴上的长度，材料ID所在列，材料质量密度所在列，X、Y循环结束时是否有空行

头信息后为两列数据，每一行分别为材料ID和该材料的密度，材料ID为输入的材料文件在`.in`中的顺序1,2,3···

**注意，ndarray与模体的关系为：\[nSlice=VoxZ\]\[height=VoxY\]\[width=VoxX\]**

下面展示使用python模拟生成water plate，其他示例代码可通过https://github.com/IIIIIsak/vox_vol_interconvert自取。

```python
# Generate water plate phantom .vox

import numpy as np

# The whole volume.

VoxX = 256
VoxY = 256
VoxZ = 256
VoxelSize = 0.125 # cm
WaterMaterialId = 2 # Material index in .in file

for WaterThickness in range(11, 21):  # cm
    # The water plate.
    ModelX = 128  # 4cm
    ModelY = int(WaterThickness / VoxelSize)
    ModelZ = 128
vol = np.ones((VoxX, VoxY, VoxZ), dtype=int)
xl = (VoxX - ModelX) // 2
yl = (VoxY - ModelY) // 2
zl = (VoxZ - ModelZ) // 2

vol[xl:xl + ModelX, yl:yl + ModelY, zl:zl + ModelZ] = WaterMaterialId

voxFile = ""
voxFile += "[SECTION VOXELS phantomER]\n{} {} {} No. OF VOXELS IN X,Y,Z\n{} {} {} VOXEL SIZE (cm) ALONG X,Y,Z\n".format(
    VoxX, VoxY, VoxZ, VoxelSize, VoxelSize, VoxelSize)
voxFile += "1 COLUMN NUMBER WHERE MATERIAL ID IS LOCATED\n"
voxFile += "2 COLUMN NUMBER WHERE THE MASS DENSITY IS LOCATED\n"
voxFile += "0 BLANK LINES AT END OF X,Y-CYCLES (1=YES,0=NO)\n"
voxFile += "[END OF VXH SECTION]\n"

Rhos = [None, '0.00120479', '1'] # Density

vol = vol.flatten()
# Save corresponding .raw volume.
# Use ImageJ Volume Viewer to display. 
# Ray points to +y in ImageJ coordinate when "0.0 1.0 0.0  # SOURCE DIRECTION COSINES: U V W"
with open('WaterPhantom.raw', 'wb') as fp:
    fp.write(vol.astype(np.uint8).tobytes())

# Generate .vox file.
for i in range(vol.shape[0]):
    voxFile += "{} {}\n".format(str(vol[i]), Rhos[vol[i]])

SavePath = '/somewhere/' + str(WaterThickness) + 'cm.vox'
with open(SavePath, 'w') as fp:
    fp.write(voxFile)

print('Done {} cm.'.format(WaterThickness))
```

##### 材料文件

MC-GPU 使用基于 PENELOPE 数据库的材料属性数据库，已经提供了一组通常用于医学成像模拟的材料的预定义材料文件“[material](https://github.com/z0gSh1u/MCGPULite/tree/master/material)”

目前已有的材料文件：

金属：铝、铯、钨、铅、钢、钛

空气、水、草酸钙、PMMA、硒、聚碳酸酯PC

人体器官：脂肪、血液90_碘10、血液、血_脾、骨头、脑袋、乳房、软骨、眼睛、腺、肾脏、肝、肺、肌肉、红骨髓、皮肤、软组织、骨松质、胃肠

### 输入参数说明

#### 1. 模拟配置 SIMULATION CONFIG

```makefile
#[SECTION SIMULATION CONFIG v.2009-05-12]
1.0e10                           # TOTAL NUMBER OF HISTORIES, OR SIMULATION TIME IN SECONDS IF VALUE < 100000
20220216                      # RANDOM SEED (ranecu PRNG)
2                               # GPU NUMBER TO USE WHEN MPI IS NOT USED, OR TO BE AVOIDED IN MPI RUNS
512                             # GPU THREADS PER CUDA BLOCK (multiple of 32)
1000                             # SIMULATED HISTORIES PER GPU THREAD
```

- TOTAL NUMBER OF HISTORIES：模拟的x射线总光子数，一般探测器上**每个像素**的接收**光子数**应大于1.0e5，Nx*Nz\*每个像素的接收光子数
- GPU NUMBER TO USE WHEN MPI IS NOT USED, OR TO BE AVOIDED IN MPI RUNS：当使用单个GPU时，设置为GPU号；当使用多GPU时，设置为-1

#### 2. x射线源 SOURCE

```makefile
#[SECTION SOURCE v.2011-07-12]
./80kVp_bowtie.spec    # X-RAY ENERGY SPECTRUM FILE
8.0   -20.0   8.0              # SOURCE POSITION: X Y Z [cm]
0.0   1.0    0.0                # SOURCE DIRECTION COSINES: U V W
-1 -1                          # TOTAL AZIMUTHAL (WIDTH, X) AND POLAR (HEIGHT, Z) APERTURES OF THE FAN BEAM [degrees] (input negative to automatically cover the whole detector)
0  -1   0             # EULER ANGLES (RxRyRz) TO ROTATE RECTANGULAR BEAM FROM DEFAULT POSITION AT Y=0, NORMAL=(0,-1,0)
```

- X-RAY ENERGY SPECTRUM FILE：指定x射线能谱文件路径
- SOURCE POSITION：x射线源初始位置 X Y Z
- SOURCE DIRECTION COSINES：x射线源初始照射方向，这里是用方向余弦进行表示的，U V W分别代表与X Y Z轴的夹角的余弦，**若设为0则为pencil beam**。
- EULER ANGLES (RxRyRz): 用于旋转矩形光束的欧拉角（RzRyRz）,定义了如何将矩形X射线束（光束）从默认位置旋转到指定位置，默认设NORMAL=(0,-1,0)。

#### 3. 图像探测器 IMAGE DETECTOR

```makefile
./out/somewhat                # OUTPUT IMAGE FILE NAME
512    512                      # NUMBER OF PIXELS IN THE IMAGE: Nx Nz
25     25                     # IMAGE SIZE (width, height): Dx Dz [cm]
50.0                           # SOURCE-TO-DETECTOR DISTANCE (detector set in front of the source, perpendicular to the initial direction)
0.0    0.0                     # IMAGE OFFSET ON DETECTOR PLANE IN WIDTH AND HEIGHT DIRECTIONS (BY DEFAULT BEAM CENTERED AT IMAGE CENTER) [cm]
 0.0200                         # DETECTOR THICKNESS [cm] (DEBUG!!)
 0.004027  # ==> MFP(Se,19.0keV)   # DETECTOR MATERIAL MEAN FREE PATH AT AVERAGE ENERGY [cm] (DEBUG!!)
0.05  3.51795           # PROTECTIVE COVER THICKNESS (detector+grid) [cm], MEAN FREE PATH AT AVERAGE ENERGY [cm] (input 0 or neative to disable)
10.0   78.74   0.0017            # ANTISCATTER GRID RATIO, FREQUENCY, STRIP THICKNESS [X:1, lp/cm, cm] (enter 0 to disable the grid)
0.0157   1.2521   # ==> MFP(lead&polystyrene,19keV)  # ANTISCATTER STRIPS AND INTERSPACE MEAN FREE PATHS AT AVERAGE ENERGY [cm]
1                              # ORIENTATION 1D FOCUSED ANTISCATTER GRID LINES: 0==STRIPS PERPENDICULAR LATERAL DIRECTION (mammo style); 1==STRIPS PARALLEL LATERAL DIRECTION (CBCT style)
```

- OUTPUT IMAGE FILE NAME：输出文件路径

- NUMBER OF PIXELS IN THE IMAGE：图像的宽和高（像素）

- IMAGE SIZE：图像的宽和高（厘米）

- SOURCE-TO-DETECTOR DISTANCE：x射线源到探测器之间的距离，探测器平面自动位于源焦点正前方的指定距离处，准直锥形光束指向探测器的几何中心。**注意要保证x射线源和探测器旋转一周均不与物体模型重合**

- IMAGE OFFSET ON DETECTOR PLANE IN WIDTH AND HEIGHT DIRECTIONS ： 图像探测器平面在宽度方向（X轴）和高度方向（Z轴）上的偏移量，也就是说，它用于定义X射线束相对于探测器中心的偏移。如果不进行偏移，X射线束会被默认对准探测器的中心

- DETECTOR THICKNESS ： 探测器的厚度（厘米） （**仍在DEBUG**！，目前保持默认值不变即可）

- DETECTOR MATERIAL MEAN FREE PATH AT AVERAGE ENERGY  ： 探测器材料在平均能量下的平均自由程（MFP）（**仍在DEBUG**！，目前保持默认值不变即可）

- PROTECTIVE COVER THICKNESS (detector+grid) [cm], MEAN FREE PATH AT AVERAGE ENERGY [cm] ： 保护盖厚度和给定能谱的**加权平均能量**下的平均自由程， 输入0或负数可以禁用。

- ANTISCATTER GRID RATIO, FREQUENCY, STRIP THICKNESS [X:1, lp/cm, cm] ：抗散射栅格比、频率和栅格条厚度， 输入0或负数可以禁用。

- ORIENTATION 1D FOCUSED ANTISCATTER GRID LINES： 滤线珊的方向，控制栅格的条带（strips）是**如何排列**的，默认设置为**1**，即**平行于横向方向**排列。

#### 4. 扫描轨迹 SCAN TRAJECTORY

```makefile
3                               # NUMBER OF PROJECTIONS (beam must be perpendicular to Z axis, set to 1 for a single projection)
20.0                            # SOURCE-TO-ROTATION AXIS DISTANCE (rotation radius, axis parallel to Z)
120.0                            # ANGLE BETWEEN PROJECTIONS [degrees] (360/num_projections for full CT)
0                           # ANGULAR ROTATION TO FIRST PROJECTION (USEFUL FOR DBT, INPUT SOURCE DIRECTION CONSIDERED AS 0 DEGREES) [degrees]
 0.0  0.0  1.0                  # AXIS OF ROTATION (Vx,Vy,Vz)
 0.0                            # VERTICAL TRANSLATION BETWEEN PROJECTIONS (HELICAL SCAN)
```

- NUMBER OF PROJECTIONS：投影的数量，**射线的光束是垂直于Z轴的**
- SOURCE-TO-ROTATION AXIS DISTANCE：源到旋转轴之间的距离，旋转轴是平行于Z轴的。
- ANGLE BETWEEN PROJECTIONS：不同投影之间的间隔角度，如果需要360度全投影，其值为**360/NUMBER OF PROJECTIONS**。
- ANGULAR ROTATION TO FIRST PROJECTION ： 第一张投影的角度。
- AXIS OF ROTATION (Vx,Vy,Vz) ： 旋转轴的方向，决定旋转轴是x轴、y轴还是z轴。

#### 5. 模型文件 VOXELIZED GEOMETRY FILE

```makefile
/home/zhuoxu/workspace/scatter-mc/WaterPlate3cm.vox          # VOXEL GEOMETRY FILE (penEasy 2008 format; .gz accepted)
0.0    0.0    0.0              # OFFSET OF THE VOXEL GEOMETRY (DEFAULT ORIGIN AT LOWER BACK CORNER) [cm]
```

- VOXEL GEOMETRY FILE：物体的体素化集合模型

**在本程序中，坐标系原点并不是物体的中心，而是物体“包装盒”的后下角，即整体只处于第一象限内，如图所示:**

![5](https://github.com/IIIIIsak/MCGPULite_v1.5.io/tree/main/assets/5.png)

**在设置其他参数时，需要以此坐标系特征为前提。**

假设有一个大小为16\*16\*16cm的圆柱形水模，源初始方向为(0,1,0)，我们在已知SOURCE-TO-ROTATION AXIS DISTANCE（SOD）的情况下，如何设置射线源的初始位置才能使得旋转轴穿过模型的中心呢？

![图片1](https://github.com/IIIIIsak/MCGPULite_v1.5.io/tree/main/assets/图片1.png)

如图，射线源的位置为($\frac x2,\frac y2 - SOD,\frac z2$)

间隔120°得到的三张投影结果如下：

![7](https://github.com/IIIIIsak/MCGPULite_v1.5.io/tree/main/assets/7.png)

#### 6. 材料文件列表 MATERIAL FILE LIST

```makefile
/home/kanshengqi/software/MCGPULite/material/air__5-120keV.mcgpu.gz                  density=0.0012   voxelId=0         #  1st MATERIAL FILE (.gz accepted)
/home/kanshengqi/software/MCGPULite/material/water__5-120keV.mcgpu.gz              density=1    voxelId=1                #  2nd MATERIAL FILE
/home/kanshengqi/software/MCGPULite/material/adipose__5-120keV.mcgpu               density=0.920    voxelId=5,10    #  3rd MATERIAL FILE
/home/kanshengqi/software/MCGPULite/material/blood__5-120keV.mcgpu                                                                         #  4th MATERIAL FILE
```

- ！！注意：***目前如果用带header的`.vox`*文件，也就是老方法的vox文件，自定义编号和密度是不适用的**
- 旧版本代码使用了硬编码的转换表，将材料分配给特定的体素ID号，并为每种材料设定了固定的密度。该代码已升级，允许用户为输入文件中列出的每种材料选择密度和体素ID。
- 在每个材料文件名后添加关键字`density=`和所需的材料密度（如果未给出密度，则使用材料文件中默认材料密度）。
- 可以使用关键字`voxelID=`提供以逗号分隔的体素ID号列表。输入几何体中具有给定ID的体素将被分配给相应的材料。请参见输入文件“MC-GPULite_v1.5_sample.in”以获取示例。如果未输入密度和体素ID，软件将自动使用默认密度和编号。新代码与旧输入文件100%兼容，且更改对模拟结果没有影响。

## 运行

- 单GPU运行

```makefile
# .in file. Specify which CUDA GPU to run on here.

3              # GPU NUMBER TO USE WHEN MPI IS NOT USED, OR TO BE AVOIDED IN MPI RUNS

# Command line. Just as it should be.

MCGPULite1.5 ./MCGPULite_v1.5.sample.in
```

- 多GPU运行

```makefile
# .in file. Use -1 always here.

-1              # GPU NUMBER TO USE WHEN MPI IS NOT USED, OR TO BE AVOIDED IN MPI RUNS

# Command line. Use CUDA_VISIBLE_DEVICES to specify GPUs, and pass GPU counts to -np.

mpirun -x CUDA_VISIBLE_DEVICES=0,1,2,3 -np 4 MCGPULite1.5 ./MCGPULite_v1.5.sample.in
```

经过实验，如果 TOTAL NUMBER OF HISTOIES 小于 1e10，因为分布开销，不建议使用多个 GPU。如果光子数达到 1e11 ，时间会缩短了66%。

- 可能出现的BUG

如果出现类似**error while loading shared libraries: libcudart.so.10.0: cannot open shared object file，**可以参考此篇博客[(23条消息) error while loading shared libraries libcudart.so.8.0: No such file or directory问题解决_坎幽黑尔弥？的博客-CSDN博客](https://blog.csdn.net/qq_38469553/article/details/80765504?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-80765504-blog-102997059.pc_relevant_multi_platform_whitelistv3&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-80765504-blog-102997059.pc_relevant_multi_platform_whitelistv3&utm_relevant_index=1)

参考修改：

```makefile
vim .bashrc
在这个文件中添加上述依赖项的位置目录：
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda-10.0/
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda-10.0/lib64/
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/wuxian/app/openmpi-4.1.1/install/lib/
保存并退出之后执行：
source .bashrc
```

## 输出文件

MCGPULite_v1.5输出的每个raw文件里包含三张图，依次为：**Total(Primary+ Scatter), Primary, Scatter，即含散射的投影，不含散射的投影，散射图，** 单个文件导入imageJ中时注意设置Number of images为3。 MCGPU-for-MATLAB 可直接通过代码自行选择保存。

![8](https://github.com/IIIIIsak/MCGPULite_v1.5.io/tree/main/assets/8.png)

除此外，MCGPULite_v1.5还会输出每张投影的模拟报告的二进制文件，只能通过**imageJ**打开，该报告包含了该张投影图的几何参数、光子到达探测器的平均概率、模拟的光子数、模拟时间等参数。

![image-20241018144708511](https://github.com/IIIIIsak/MCGPULite_v1.5.io/tree/main/assets/9)

## 已知的问题

### 亮场模拟bug

在模拟亮场时，一般是将模体做成一个全是空气密度的模体，然后其他参数不变就可以了；但是在MCGPU_v1.5b版本中，如果射线源有光子穿过了空体，似乎会发生一些“奇怪”的反应，导致生成的结果出现**环形的噪声**，目前暂时不知道应该怎么修复，但是可以通过设置射线源的位置，比如“拉高”射线源位置使得光子可以不穿过模体直接打到探测器上，这样模拟出来的亮场是没有问题的。
