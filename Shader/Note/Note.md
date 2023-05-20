# Unity Shader 笔记

## GPU流水线

### GPU渲染流程图

<img src="./images/GPU渲染流程.png"><br>*▲ 颜色表示了不同阶段的可配置性或可编程性：绿色表示该流水线阶段是完全可编程控制的，黄色表示该流水线阶段可以配置但不是可编程的，蓝色表示该流水线阶段是由 GPU 固定实现的，开发者没有任何控制权。实线表示该 Shader 必须由开发者编程实现，虚线表示该 Shader 是可选的*</img>

### 顶点着色器（Vertex Shader）

顶点着色器需要完成的工作主要有： **[坐标变换](#坐标变换)** 和 **[逐顶点光照](#逐顶点光照)** 。

#### 坐标变换

一个最基本的顶点着色器必须完成的一个工作是： **把顶点坐标从模型空间转换到齐次裁剪空间** 。

##### 这是 Unity 内置渲染管线中的部分代码

```hlsl
#include "UnityCG.cginc"

// 标准写法
o.pos = mul(UNITY_MVP，v.position);

// 更高效的写法
o.pos = mul(UNITY_MATRIX_VP, mul(unity_ObjectToWorld, float4(v.position, 1.0)));

// 直接调用 Unity 函数
o.pos = UnityObjectToClipPos(v.position)
```

##### 这是 Unity URP 中的部分代码

```hlsl
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

o.pos = TransformObjectToHClip(v.position);
```

#### 逐顶点光照

***Todo***: 待学习

### 裁剪（Clipping）

<img src="./images/裁剪示意图.png"><br>*▲ 只有在单位立方体的图元才需要被继续处理。因此，完全在单位立方体外部的图元（红色三角形）被舍弃，完全在单位立方体内部的图（绿色三角形）将被保留。和单位立方体相交的图元（黄色三角形）会被裁剪，新的顶点会被生成，原来在外部的顶点会被舍弃。*</img>

**裁剪** 是硬件上的固定操作，但我们可以自定义一个裁剪操作来对这一步进行配置。

### 屏幕映射（Screen Mapping）

屏幕映射（Screen Mapping）的任务是把每个图元的 x 和 y 坐标转换到屏幕坐标系（Screen Coordinates）下。屏幕坐标系是一个二维坐标系，它和我们用于显示画面的分辨率有很大关系。\
*注意：* OpenGL 把屏幕的左下角当成最小的窗口坐标值，而 DirectX 则定义了屏幕的左上角为最小的窗口坐标值。

### 三角形设置（Triangle Setup）

这个阶段会计算光栅化一个三角网格所需的信息。它的输出是为了给下一个阶段做准备。

### 三角形遍历（Triangle Traversal）

这个阶段将会检查每个 **像素** 是否被一个三角网格所覆盖。如果被覆盖的话，就会生成一个 **片元**（fragment）。
而这样一个找到哪些像素被三角网格覆盖的过程就是三角形遍历，这个阶段也被称为扫描变换（Scan Conversion）

<img src="./images/三角形遍历.png"><br>*▲ 根据几何阶段输出的顶点信息，最终得到该三角网格覆盖的像素位置。对应像素会生成一个片元，而片元中的状态是对 3 个顶点的信息进行插值得到的。例如，上图中 3 个顶点的深度进行插值得到其重心位置对应的片元的深度值为 -10.0*</img>

这一步的输出就是得到一个片元序列。需要注意的是，一个片元并不是真正意义上的像素，而是包含了很多状态的集合，这些状态用于计算每个像素的最终颜色。这些状态包括了（但不限于）它的屏幕坐标、深度信息，以及其他从几何阶段输出的顶点信息，例如法线、纹理坐标等。

### 片元着色器（Fragment Shader）

在DirectX中，片元着色器被称为像素着色器（Pixel Shader）。\
片元着色器的输入是上一个阶段对顶点信息插值得到的结果，更具体来说，是根据那些从顶点着色器中输出的数据插值得到的。而它的输出是一个或者多个颜色值。\
这一阶段可以完成很多重要的渲染技术，其中最重要的技术之一就是纹理采样。为了在片元着色器中进行[纹理采样](#纹理采样texture-sampling)，我们通常会在顶点着色器阶段输出每个顶点对应的纹理坐标，然后经过光栅化阶段对三角网格的3个顶点对应的纹理坐标进行插值后，就可以得到其覆盖的片元的纹理坐标了。\
它的局限在于，它仅可以影响单个片元。也就是说，当执行片元着色器时，它不可以将自己的任何结果直接发送给它的邻居们。\

### 逐片元操作

逐片元操作（Per-Fragment Operations）是 OpenGL 中的说法，在 DirectX 中，这一阶段被称为输出合并阶段（Output-Merger）。\
这一阶段有几个主要任务。

1. 决定每个片元的可见性。这涉及了很多测试工作，例如[模板测试](#模板测试stencil-test流程)、[深度测试](#深度测试depth-test流程)等。
2. 如果一个片元通过了所有的测试，就需要把这个片元的颜色值和已经存储在颜色缓冲区中的颜色进行合并，或者说是[混合](#混合blend流程)。

<img src="./images/逐片元操作流程.png"><br>*▲ 逐片元操作阶段所做的操作。只有通过了所有的测试后，新生成的片元才能和颜色缓冲区中已经存在的像素颜色进行混合，最后再写入颜色缓冲区中。*</img>

#### 模板测试和深度测试

<img src="./images/片元模板测试和深度测试.png"><br>*▲ 模板测试和深度测试的简化流程图。注意他们在测试失败后的不同流程。*</img>

##### 模板测试（Stencil Test）流程

与之相关的是[Stencil 命令](#stencil-命令详解)。

如果开启了模板测试， GPU 会首先读取（使用读取掩码）模板缓冲区中该片元位置的模板值，然后将该值和读取（使用读取掩码）到的参考值（reference value）进行比较，这个比较函数可以是由开发者指定的，例如小于时舍弃该片元，或者大于等于时舍弃该片元。如果这个片元没有通过这个测试，该片元就会被舍弃。不管一个片元有没有通过模板测试，我们都可以根据模板测试和下面的深度测试结果来修改模板缓冲区，这个修改操作也是由开发者指定的。开发者可以设置不同结果下的修改操作，例如，在失败时模板缓冲区保持不变，通过时将模板缓冲区中对应位置的值加1等。模板测试通常用于限制渲染的区域。另外，模板测试还有一些更高级的用法，如渲染阴影、轮廓渲染等。

如果一个片元通过了模板测试，那么它会进行下一个测试——深度测试（Depth Test）。

##### 深度测试（Depth Test）流程

如果开启了深度测试， GPU 会把该片元的深度值和已经存在于深度缓冲区中的深度值进行比较。这个比较函数也是可由开发者设置的，例如小于时舍弃该片元，或者大于等于时舍弃该片元。通常这个比较函数是小于等于的关系，即如果这个片元的深度值大于等于当前深度缓冲区中的值，那么就会舍弃它。这是因为，我们总想只显示出离摄像机最近的物体，而那些被其他物体遮挡的就不需要出现在屏幕上。如果这个片元没有通过这个测试，该片元就会被舍弃。

和模板测试有些不同的是，如果一个片元没有通过深度测试，它就没有权利更改深度缓冲区中的值。而如果它通过了测试，开发者还可以指定是否要用这个片元的深度值覆盖掉原有的深度值，这是通过开启/关闭深度写入来做到的。

透明效果和深度测试以及深度写入的关系非常密切。

如果一个片元通过了上面的所有测试，它就可以来到混合功能的面前。

#### 混合（Blend）流程

<img src="./images/片元混合操作.png"><br>*▲ 混合操作的简化流程图*</img>

混合操作也是可以高度配置的：开发者可以选择开启/关闭混合功能。\
如果没有开启混合功能，就会直接使用片元的颜色覆盖掉颜色缓冲区中的颜色。那么就无法得到透明效果。\
如果开启了混合， GPU 会取出源颜色和目标颜色，将两种颜色进行混合。源颜色指的是片元着色器得到的颜色值，而目标颜色则是已经存在于颜色缓冲区中的颜色值。之后，就会使用一个混合函数来进行混合操作。

更详细的解释参考[混合命令详解](#blend-命令详解)

## 数学基础

### 笛卡尔坐标系

#### 二维笛卡儿坐标系

一个二维的笛卡儿坐标系包含了两个部分的信息：

- 一个特殊的位置，即原点，它是整个坐标系的中心；
- 两条过原点的互相垂直的矢量，即x轴和y轴。这些坐标轴也被称为是该坐标系的 [基矢量](#基矢量basis-vector) 。

二维笛卡尔坐标系的坐标轴方向不是固定的。

<img src="./images/OpenGL 和 DirectX 屏幕映射时使用的笛卡尔坐标系.png"><br>*▲ 在屏幕映射时，OpenGL和DirectX使用了不同方向的二维笛卡儿坐标系*</img>

#### 三维笛卡儿坐标系

和二维笛卡儿坐标系类似，三维笛卡儿坐标系中的坐标轴方向也不是固定的，这种不同导致了两种不同种类的坐标系：左手坐标系（left-handed coordinate space）和右手坐标系（right-handed coordinate space）。除了坐标轴朝向不同之外，左手坐标系和右手坐标系对于正向旋转的定义也不同，左手坐标系对应左手法则（left-hand rule）；右手坐标系对应右手法则（right-hand rule）。

<img src="./images/左手坐标系示意图.png">  <img src="./images/右手坐标系示意图.png"><br>*▲ 左手坐标系和右手坐标系*</img>

<img src="./images/左手法则示意图.png">  <img src="./images/右手法则示意图.png"><br>*▲ 左手法则和右手法则*</img>

**个人总结：** 无论哪种坐标系，旋转正方向的顺序都是 X轴正方向 -> Y轴正方向 -> Z轴正方向 -> X轴正方向 ....

#### Unity 使用的坐标系

对于模型空间和世界空间，Unity使用的是左手坐标系。一个物体的右侧（right）、上侧（up）和前侧（forward）分别对应了x轴、y轴和z轴的正方向。

但对于观察空间来说，Unity使用的是右手坐标系。在这个坐标系中，摄像机的前向是z轴的负方向，这与在模型空间和世界空间中的定义相反。也就是说，z轴坐标的减少意味着场景深度的增加。

<img src="./images/Unity的观察空间使用了右手坐标系.png"><br>*▲ 在Unity中，观察空间使用的是右手坐标系，摄像机的前向是z轴的负方向，z轴越小，物体的深度越大，离摄像机越远*</img>

***ToDo***：加入模型空间、世界空间、观察空间的名词解释和链接

### 点和矢量

#### 点和矢量的定义

参考 [点的定义](#点point) 和 [矢量的定义](#矢量vector) 。

*注意：* 虽然程序上使用同样的数据结构来表示点和矢量，但是他们的数学意义并不相同。

#### 矢量和标量乘法/除法

矢量和标量的乘法运算只需要把矢量的每个分量和标量相乘即可

$$
k \vec v = (kx_v,\ ky_v,\ kz_v)
$$

一个矢量除以一个非零的标量等同于和这个标量的倒数相乘

$$
\frac{\vec v}{k} \
= \frac{(x_v,\ y_v,\ z_v)}{k} \
= \left( \frac{x_v}{k},\ \frac{y_v}{k},\ \frac{z_v}{k} \right) \quad {k \neq 0}
$$

矢量和标量的乘法/除法具有以下数学性质：

- 对于乘法，矢量和标量的位置可以互换
- 对于除法，只能是矢量除以标量，而不能是标量除以矢量

#### 矢量的加法和减法

一个矢量只能和相同维度的矢量相加减，运算时只需要把两个矢量的对应分量进行相加或相减即可，其结果是一个相同维度的新矢量。\
公式如下

$$
\begin{aligned}
\vec a + \vec b &= (x_a + x_b,\ y_a + y_b,\ z_a + z_b) \\
\vec a - \vec b &= (x_a - x_b,\ y_a - y_b,\ z_a - z_b)
\end{aligned}
$$

**矢量的加法** 参数顺序可以互换。\
当矢量用于表示位置信息时，如果第一个矢量被视为空间中的一个点，那么第二个矢量可以解释为从该位置偏移或“跳跃”。他们的和就是（偏移之后）到达的新位置\
如果矢量表示力，那么根据它们的方向和大小来考虑它们会更直观（大小表示力的大小）。两个力矢量相加会产生一个新矢量，该矢量等效于这些力的合力。

<img src="./images/矢量加法示意图.png"><br>*▲ 从 a 点，经过 b 偏移之后到达 c 点。*</img>

**矢量的减法** 参数顺序不能互换。\
矢量减法最常用于获取一个位置到另一个位置的方向和距离，即 $\vec c = \vec {ab} = \vec b - \vec a$ 。

<img src="./images/矢量减法示意图.png"><br>*▲ 从 a 点到 b 点的距离和方向（注意参数顺序）。*</img>

#### 矢量的点积

矢量点积（dot product）也被称为内积（inner product），数学公式如下

$$
\begin{aligned}
\vec a \cdot \vec b \
& = (x_a,\ y_a,\ z_a) \cdot (x_b,\ y_b,\ z_b) \\
& = x_a x_b + y_a y_b + z_a z_b \\
\end{aligned}
$$

更广泛的点积公式为 对于 $n$ 维矢量 $\vec A$ 和 $\vec B$

$$
\vec A \cdot \vec B = \sum_{i=1}^{n} {A_i B_i}
$$

点积的名称来源于这个运算的符号： $\vec a \cdot \vec b$ 。 *注意：* 中间的这个圆点符号是不可以省略的。\
点积是把两个矢量对应分量相乘后再取和，最后的结果是一个 **标量** 。

矢量点积具有以下数学性质：

- 点积满足交换律，即 $\vec a \cdot \vec b = \vec b \cdot \vec a$
- 点积满足分配率，即 $\vec a \cdot (\vec b + \vec c) = {\vec a \cdot \vec b} + {\vec a \cdot \vec c}, \quad {\vec a + \vec b} \cdot \vec c = {\vec a \cdot \vec c + \vec b \cdot \vec c}$
- 点积可结合标量乘法，即 $(k\vec a) \cdot \vec b = \vec a \cdot (k\vec b) = k (\vec a \cdot \vec b)$

矢量点积的几何公式如下

$$
\vec a \cdot \vec b = \lVert \vec a \rVert \ \lVert \vec b \rVert \ \cos\theta
$$

点积在数学上比计算余弦更简单，因此在某些情况下可以用它代替 Mathf.Cos 函数

当计算一个矢量在另一个矢量方向上的大小，点积很有用。例如：计算力或者速度在某个方向上的分量大小时，可以使用点积运算：\
分量大小 = 力或者速度矢量 $\cdot$ 那个方向的单位矢量。

<img src="./images/矢量点积用于投影.png"><br>*▲ 矢量点积用于投影*</img>

点积的结果可能是负数。结果的正负号与两个矢量的方向有关：当它们的方向相反（夹角大于90°）时，结果小于0；当它们的方向互相垂直（夹角为90°）时，结果等于0；当它们的方向相同（夹角小于90°）时，结果大于0 。

<img src="./images/矢量点积结果的符号.png"><br>*▲ 矢量点积结果的符号用于判断两个矢量的方向关系*</img>

#### 矢量的叉积

矢量叉积（cross product），也被称为外积（outer product），数学公式如下

$$
\begin{aligned}
\vec a \times \vec b \
& = (x_a,\ y_a,\ z_a) \times (x_b,\ y_b,\ z_b) \\
& = (ya z_b - z_a y_b,\ z_a x_b - x_a z_b, x_a y_b - y_a x_b) \\
& = \
\begin{bmatrix}
0 & -z_a & y_a \\
z_a & 0 & -x_a \\
-y_a & x_a & 0
\end{bmatrix}
\begin{bmatrix}
x_b \\
y_b \\
z_b
\end{bmatrix}
\end{aligned}
$$

和点积类似，叉积的名称来源于它的符号： $\vec a \times \vec b$ 。 *注意：* 中间的这个叉号是不可以省略的。\
与点积不同的是，矢量叉积的结果仍是一个 **矢量** 。\
矢量叉积只是用于三维矢量。

矢量叉积具有以下数学性质：

- 叉积不满足交换律，但是满足反交换律，即 ${\vec a \times \vec b} = {-(\vec b \times \vec a)}$
- 叉积不满足结合律，即 ${(\vec a \times \vec b) \times \vec c} \neq {\vec a \times (\vec b \times \vec c)}$ ，其实 $\vec a \times (\vec b \times \vec c) = (\vec a \cdot \vec c) \vec b - (\vec a \cdot \vec b) \vec c$
- 叉积满足分配率，即 ${\vec a \times (\vec b + \vec c)} = {\vec a \times \vec b + \vec a \times \vec c}, \quad {(\vec a + \vec b) \times \vec c} = {\vec a \times \vec c + \vec b \times \vec c}$
- 方向相同或相反的（任意大小的）矢量，叉积为 $\vec 0$ 矢量。 *注意：* 不是 0 标量。

矢量叉积的模公式如下

$$
\lVert \vec a \times \vec b \rVert = \lVert \vec a \rVert \ \lVert \vec b \rVert \ \sin\theta
$$

叉积在数学上比计算正弦更简单，因此在某些情况下可以用它代替 Mathf.Sin 函数

矢量叉积的方向同时垂直与两个参数矢量，因为空间上同时垂直于参数矢量的方向有两种可能，所以参数的顺序很重要，在 Unity 中可以通过左手定则来判断。

<img src="./images/矢量叉积在两种坐标系下的方向.png"><br>*▲ 矢量叉积在两种坐标系下有不同的方向*</img>

<img src="./images/矢量叉积在Unity中的方向.png"><br>*▲ 矢量叉积在Unity中的方向*</img>

矢量叉积常用于获取三角形面的法线方向，可以通过下面的方法来获取。\
*注意：* 叉积参数的顺序必须和三角形的顶点顺序一致。

<img src="./images/矢量叉积用于计算三角形面的法线方向.png"><br>*▲ 矢量叉积用于获取法线方向*</img>

代码如下

```csharp
// 假定三角形顶点的顺序就是 a, b, c
Vector3 a, b, c;

// 参数顺序必须和三角形顶点顺序一致（先 b 后 c ）。
Vector3 side1 = b - a;
Vector3 side2 = c - a;

Vector3 normal = Vector3.Cross(side1, side2);
```

### 矩阵

#### 矩阵的定义

矩阵的格式用 $行数 \times 列数$ 来表示。例如一个 $3 \times 4$ 的矩阵，它有三行四列。

下面的矩阵 $M$ 是一个 $r \times c$ 的矩阵，元素 $m_{ij}$ 表明了这个元素在矩阵 $M$ 的第 $i$ 行、第 $j$ 列。

$$
M_{rc} = \
\begin{bmatrix}
m_{11} & m_{12} & \cdots & m_{1c} \\
m_{21} & m_{22} & \cdots & m_{2c} \\
\vdots  & \vdots  & \ddots & \vdots  \\
m_{r1} & m_{r2} & \cdots & m_{rc}
\end{bmatrix}
$$

#### 矩阵和标量的乘法

和矢量类似，矩阵也可以和标量相乘，它的结果仍然是一个相同维度的矩阵。它们之间的乘法非常简单，就是矩阵的每个元素和该标量相乘。

$$
k M_{rc} = \
\begin{bmatrix}
k m_{11} & k m_{12} & \cdots & k m_{1c} \\
k m_{21} & k m_{22} & \cdots & k m_{2c} \\
\vdots  & \vdots  & \ddots & \vdots  \\
k m_{r1} & k m_{r2} & \cdots & k m_{rc}
\end{bmatrix}
$$

#### 矩阵乘法

一个 $r \times n$ 的矩阵 $A$ 和一个 $n \times c$ 的矩阵 $B$ 相乘，它们的结果 $C = AB$ 将会是一个 $r \times c$ 大小的矩阵。\
第一个矩阵的列数 **必须** 和第二个矩阵的行数相同，它们相乘得到的矩阵的行数是第一个矩阵的行数，而列数是第二个矩阵的列数。\
例如，如果矩阵 $A$ 的维度是 $4 \times 3$ ，矩阵 $B$ 的维度是 $3 \times 6$ ，那么 $AB$ 的维度就是 $4 \times 6$ 。

$C$ 中的每一个元素 $c_{ij}$ 等于 $A$ 的第 $i$ 行所对应的矢量和 $B$ 的第 $j$ 列所对应的矢量进行矢量点乘的结果

数学公式如下

$$
\begin{aligned}
c_{ij} \
& = a_{i1} b_{1j} + a_{i2}b_{2j} + \cdots + a_{in} b_{nj} \\
& = \sum_{k=1}^n {a_{ik} b_{kj}} \\
\end{aligned}
$$

矩阵乘法具有以下数学性质：

- 矩阵乘法不满足交换律，即 $AB \neq BA$
- 矩阵乘法满足结合律，即 $(AB)C = A(BC)$
- 矩阵乘法满足分配律，即 $A(B+C) = AB + AC, \quad (A+B)C = AC + BC$
- 对于 [行矩阵](#行矩阵row-matrix) $R$ 和 [列矩阵](#列矩阵column-matrix) $C$ 相乘： $RC$ = 标量

#### 转置矩阵（transposed matrix）

转置矩阵（transposed matrix）实际是对原矩阵的一种运算，即转置运算。给定一个 $r \times c$ 的矩阵 $M$ ，它的转置可以表示成 $M^T$ ，这是一个 $c \times r$ 的矩阵。原矩阵的第 $i$ 行变成了第 $i$ 列，而第 $j$ 列变成了第 $j$ 行。

转置矩阵具有以下数学性质：

- 矩阵转置的转置等于原矩阵，即 $(M^T)^T = M$
- 矩阵串接的转置，等于反向串接各个矩阵的转置，即 $(AB)^T = B^T A^T$

#### 逆矩阵（inverse matrix）

给定一个矩阵 $M$ ，它的逆矩阵用 $M^{-1}$ 来表示。逆矩阵最重要的性质就是，如果我们把 $M$ 和 $M^{-1}$ 相乘，那么它们的结果将会是一个 [单位矩阵](#单位矩阵identity-matrix) $I$ 。即 $M M^{-1} = M^{-1} M = I$

如果一个矩阵有对应的逆矩阵，我们就说这个矩阵是可逆的（invertible）或者说是非奇异的（nonsingular）；相反的，如果一个矩阵没有对应的逆矩阵，我们就说它是不可逆的（noninvertible）或者说是奇异的（singular）。

逆矩阵具有以下数学性质：

- 逆矩阵的逆矩阵是原矩阵本身，即 $(M^{-1})^{-1} = M$
- [单位矩阵](#单位矩阵identity-matrix) 的逆矩阵是它本身，即 $I^{-1} = I$
- [转置矩阵](#转置矩阵transposed-matrix) 的逆矩阵是逆矩阵的转置，即 $(M^T)^{-1} = (M^{-1})^T$
- 矩阵串接相乘后的逆矩阵等于反向串接各个矩阵的逆矩阵，即 $(AB)^{-1} = B^{-1} A^{-1}$

#### 正交矩阵（orthogonal matrix）

如果一个 [方阵](#方块矩阵square-matrix) $M$ 和它的 [转置矩阵](#转置矩阵transposed-matrix) $M^{T}$ 的乘积是 [单位矩阵](#单位矩阵identity-matrix) $I$ 的话，那么这个矩阵是正交的（orthogonal）。反过来也是成立的。也就是说，矩阵M是正交的等价于： $M M^T = M^T M = I$\
即如果一个矩阵是正交的，那么它的转置矩阵和逆矩阵是一样的。也就是说，矩阵 $M$ 是正交的等价于： $M^T = M^{-1}$

[简单推导](#正交矩阵-就是由一组-标准正交基-组成的推导过程) 后，可以发现正交矩阵就是由一组标准正交基组成的，反之亦然。

#### 用矩阵表示矢量

在 Unity 中，常规做法是把矢量放在矩阵的右侧，即把矢量转换成 [列矩阵](#列矩阵column-matrix) 来进行运算。这意味着，我们的矩阵乘法通常都是右乘，例如： $C B A \vec v = (C (B (A \vec v)))$

使用列矩阵的结果是，我们的阅读顺序是从右到左，即对矢量 $\vec v$ 先使用 $A$ 进行变换，再使用 $B$ 进行变换，最后使用 $C$ 进行变换。

任何变换得到的结果是一个新的列矩阵，也就是变换之后的新矢量。

### 变换（transform）

#### 线性变换（linear transform）

线性变换指的是那些可以保留矢量加和标量乘的变换。用数学公式来表示这两个条件就是：

$$
\begin{aligned}
 f(x) + f(y) & = f(x + y) \\
 kf(x) & = f(kx)
\end{aligned}
$$

对于三维矢量，仅用 $3 \times 3$ 矩阵就可以表示线性变化。

线性变换可以表示旋转、缩放、错切（shear）、镜像（mirroring，也被称为 reflection）、正交投影（orthographic projection）。

线性变换 **不能表示平移** 。

#### 仿射变换（affine transform）

对于三维矢量，仿射变换就是合并线性变换和平移变换的变换类型。仿射变换可以使用一个 $4 \times 4$ 的矩阵来表示，为此，我们需要把矢量扩展到四维空间下，这就是 [齐次坐标](#齐次坐标homogeneous-coordinate) 空间（homogeneous space）。

变换矩阵由 4 个基础部分组成，如下所示

$$
\begin{bmatrix}
M_{3 \times 3} & T_{3 \times 1} \\
0_{1 \times 3} & 1
\end{bmatrix}
$$

其中

- $M_{3 \times 3}$ 用于表示线性变换，例如旋转和缩放
- $T_{3 \times 1}$ 用于表示平移
- $0_{1 \times 3}$ 是零矩阵，即 $0_{1 \times 3}=[0 \ 0 \ 0]$
- 右下角的 1 就是标量 1

可以参考 [齐次坐标系入门级思考](https://oncemore2020.github.io/blog/homogeneous/)

#### 齐次坐标（homogeneous coordinate）

事实上齐次坐标的维度可以超过四维，但这里中所说的齐次坐标将泛指四维齐次坐标。

将三维坐标增加一个维度 $w$ ，就可以转为齐次坐标。但是三维坐标既可以表示位置信息，也可以表示方向信息，所以针对不同的类型， $w$ 的值并不一样。

- 对于位置坐标 $p = (x, y, z)$ ， 增加维度 $w = 1$ ， 变为齐次坐标 $\bar p = (x, y, z, 1)$
- 对于方向矢量 $\vec v = (x, y, z)$ ， 增加维度 $w = 0$ ， 变为齐次坐标 $\bar v = (x, y, z, 0)$

齐次坐标非常适合用于区分位置坐标和方向矢量。因为对于方向矢量转化而来的齐次坐标，如果进行平移变化，不会有任何影响。而且对于位置和矢量的加减运算，依然满足基本定义的概念，如下所示：

- 位置 $\bar p_1$ $\pm$ 矢量 $\bar v$ = 新的位置 $\bar p_2$
- 矢量 $\bar v_1$ $\pm$ 矢量 $\bar v_2$ = 新的矢量 $\bar v_3$
- 位置 $\bar p_1$ - 位置 $\bar p_2$ = 从 $\bar p_2$ 指向 $\bar p_1$ 的矢量 $\bar v$
- 位置 $\bar p_1$ + 位置 $\bar p_2$ = 两个位置的中点 $\bar p_3$

为了统一处理位置坐标（例如上面的两个位置相加），通常对于一个齐次坐标

$${
\bar p = (x, y, z, w) = \
\begin{cases}\begin{aligned}
Point: \quad &(\frac{x}{w}, \frac{y}{w}, \frac{z}{w}), \quad &w \neq 0 \\
Vector: \quad &(x, y, z), \quad &w = 0
\end{aligned}\end{cases}
}$$

#### 常见的变换矩阵

常见的变换矩阵性质如下

| 变换名称 | 线性变换 | 仿射变换 | 可逆矩阵 | 正交矩阵 |
| :--- | :---: | :---: | :---: | :---: |
| [平移矩阵](#平移矩阵) | ✘ | ✔ | ✔ | ✘ |
| [旋转矩阵](#旋转矩阵) | ✔ | ✔ | ✔ | ✔ |
| [缩放矩阵](#缩放矩阵) | ✔ | ✔ | ✔ | ✘ |
| [错切矩阵](#错切矩阵) | ✔ | ✔ | ✔ | ✘ |
| [镜像矩阵](#镜像矩阵) | ✔ | ✔ | ✔ | ✔ |
| [正交投影矩阵](#正交投影矩阵) | ✔ | ✔ | ✘ | ✘ |
| [透视投影矩阵](#透视投影矩阵) | ✘ | ✘ | ✘ | ✘ |

#### 平移矩阵

假设沿 $x, y, z$ 轴上的平移分别为 $t_x, t_y, t_z$ ，那么平移矩阵为

$$
M_{translation}
=\begin{bmatrix}
1 & 0 & 0 & t_x \\
0 & 1 & 0 & t_y \\
0 & 0 & 1 & t_z \\
0 & 0 & 0 & 1
\end{bmatrix}
$$

可以看出，对于位置坐标 $\bar p = {x, y, z, 1}$

$$
\begin{bmatrix}
1 & 0 & 0 & t_x \\
0 & 1 & 0 & t_y \\
0 & 0 & 1 & t_z \\
0 & 0 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
x \\
y \\
z \\
1
\end{bmatrix}
= \
\begin{bmatrix}
x + t_x \\
y + t_y \\
z + t_z \\
1
\end{bmatrix}
$$

而对于方向矢量 $\bar v = {x, y, z, 0}$

$$
\begin{bmatrix}
1 & 0 & 0 & t_x \\
0 & 1 & 0 & t_y \\
0 & 0 & 1 & t_z \\
0 & 0 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
x \\
y \\
z \\
0
\end{bmatrix}
= \
\begin{bmatrix}
x \\
y \\
z \\
0
\end{bmatrix}
$$

如此就保证了

- 对 *位置坐标* 进行平移操作，得到平移之后新的位置坐标。
- 对 *方向矢量* 进行平移操作，得到的结果还是方向矢量本身。

平移矩阵 **不是** [正交矩阵](#正交矩阵orthogonal-matrix) ，平移矩阵的逆矩阵是反向平移的矩阵，即

$$
M_{translation}^{-1}
=\begin{bmatrix}
1 & 0 & 0 & t_x \\
0 & 1 & 0 & t_y \\
0 & 0 & 1 & t_z \\
0 & 0 & 0 & 1
\end{bmatrix}^{-1}
=\begin{bmatrix}
1 & 0 & 0 & -t_x \\
0 & 1 & 0 & -t_y \\
0 & 0 & 1 & -t_z \\
0 & 0 & 0 & 1
\end{bmatrix}
$$

#### 旋转矩阵

绕 X 轴旋转的旋转矩阵为

$$
R_x(\theta) =
\begin{bmatrix}
1 & 0 & 0 & 0 \\
0 & \cos\theta & -\sin\theta & 0 \\
0 & \sin\theta & \cos\theta & 0 \\
0 & 0 & 0 & 1
\end{bmatrix}
$$

绕 Y 轴旋转的旋转矩阵为

$$
R_y(\theta) =
\begin{bmatrix}
\cos\theta & 0 & \sin\theta & 0 \\
0 & 1 & 0 & 0 \\
-\sin\theta & 0 & \cos\theta & 0 \\
0 & 0 & 0 & 1
\end{bmatrix}
$$

绕 Z 轴旋转的旋转矩阵为

$$
R_z(\theta) =
\begin{bmatrix}
\cos\theta & -\sin\theta & 0 & 0 \\
\sin\theta & \cos\theta & 0 & 0 \\
0 & 0 & 1 & 0 \\
0 & 0 & 0 & 1
\end{bmatrix}
$$

对于绕 X、 Y、 Z 轴进行的复合旋转，见 [复合旋转](#复合旋转)

绕任意周旋转的旋转矩阵见 [四元数部分](#四元数旋转对应的矩阵)

旋转矩阵 **是** [正交矩阵](#正交矩阵orthogonal-matrix) ，多个旋转的复合矩阵同样 **是** [正交矩阵](#正交矩阵orthogonal-matrix) 。即
$$
M_{rotation}^{-1}\ =\ M_{rotation}^{T}
$$

#### 复合旋转

Unity 复合旋转的顺序为 Z轴 -> X轴 -> Y轴 ，虽然在 Inspector 窗口中显示的是用绕坐标轴旋转的欧拉角，但是实际上 Unity 是用 [Quaternion](https://docs.unity3d.com/ScriptReference/Quaternion.html) 来存储旋转的。

**特别注意：** Unity 在使用欧拉角行旋转时， **不旋转坐标系本身** 。例如：在 Inspector 窗口中输入 X 轴旋转 90 度，Y 轴旋转 90 度。得到的结果是：先绕坐标系的 X 轴旋转 90 度，然后绕 **同样坐标系** 的 Y 轴旋转 90 度。

**经验证：** Unity 实际使用的旋转矩阵是

$$\begin{aligned}
M_{rotation} = R_y \ R_x \ R_z
=\begin{bmatrix}
\cos\theta_y & 0 & \sin\theta_y & 0 \\
0 & 1 & 0 & 0 \\
-\sin\theta_y & 0 & \cos\theta_y & 0 \\
0 & 0 & 0 & 1
\end{bmatrix} \
\begin{bmatrix}
1 & 0 & 0 & 0 \\
0 & \cos\theta_x & -\sin\theta_x & 0 \\
0 & \sin\theta_x & \cos\theta_x & 0 \\
0 & 0 & 0 & 1
\end{bmatrix} \
\begin{bmatrix}
\cos\theta_z & -\sin\theta_z & 0 & 0 \\
\sin\theta_z & \cos\theta_z & 0 & 0 \\
0 & 0 & 1 & 0 \\
0 & 0 & 0 & 1
\end{bmatrix}
\end{aligned}$$

参考 [Unity 文档](https://docs.unity3d.com/ScriptReference/Transform-eulerAngles.html)

#### 缩放矩阵

沿坐标轴进行缩放，分为 [统一缩放](#统一缩放uniform-scale) 和 [非统一缩放](#非统一缩放nonuniform-scale)\
假设沿 $x, y, z$ 轴上的缩放系数分别为 $k_x, k_y, k_z$ ，那么缩放矩阵为

$$
M_{scale}
=\begin{bmatrix}
k_x & 0 & 0 & 0 \\
0 & k_y & 0 & 0 \\
0 & 0 & k_z & 0 \\
0 & 0 & 0 & 1
\end{bmatrix}
$$

缩放矩阵 **不是** [正交矩阵](#正交矩阵orthogonal-matrix) ，缩放矩阵的逆矩阵是使用原缩放系数的倒数来求得，即

$$
M_{scale}^{-1}
=\begin{bmatrix}
k_x & 0 & 0 & 0 \\
0 & k_y & 0 & 0 \\
0 & 0 & k_z & 0 \\
0 & 0 & 0 & 1
\end{bmatrix}^{-1}
=\begin{bmatrix}
\dfrac{1}{k_x} & 0 & 0 & 0 \\
0 & \dfrac{1}{k_y} & 0 & 0 \\
0 & 0 & \dfrac{1}{k_z} & 0 \\
0 & 0 & 0 & 1
\end{bmatrix}
$$

#### 复合变换

由 平移、（复合）旋转、缩放 所组成的变换，被称为复合变换。在 Unity 中，变换的顺序为：先缩放，再旋转，最后平移。即

$$
M_{transform} = M_{translation} \ M_{rotation} \ M_{scale}
$$

**注意：** 这个变换的顺序其实是由变换矩阵本身的特点（即所有变换都是相对同一个坐标系而言）所决定的，符合一般人的基本概念。所以这个顺序并不是随便定义的。

#### 复合变换矩阵各列的意义

假设 [复合变换](#复合变换) 矩阵为如下形式

$$
M =
\begin{bmatrix}
m_0 & m_3 & m_6 & m_9 \\
m_1 & m_4 & m_7 & m_{10} \\
m_2 & m_5 & m_8 & m_{11} \\
0 & 0 & 0 & 1
\end{bmatrix}
=\begin{bmatrix}
\ | & | & | & | \ \\
\ \vec v_1 & \vec v_2 & \vec v_3 & \vec v_4\ \\
\ | & | & | & | \ \\
\ 0 & 0 & 0 & 1 \
\end{bmatrix}
$$

那么

- 矢量 $\vec v_1$ 的方向 $\hat v_1$ 就是（原坐标系的） $x$ 轴变换之后（在原坐标系下）的方向，
模长 $\lVert \vec v_1 \rVert$ 为 $x$ 轴方向的缩放系数。
- 矢量 $\vec v_2$ 的方向 $\hat v_2$ 就是（原坐标系的） $y$ 轴变换之后（在原坐标系下）的方向，
模长 $\lVert \vec v_2 \rVert$ 为 $y$ 轴方向的缩放系数。
- 矢量 $\vec v_3$ 的方向 $\hat v_3$ 就是（原坐标系的） $z$ 轴变换之后（在原坐标系下）的方向，
模长 $\lVert \vec v_3 \rVert$ 为 $z$ 轴方向的缩放系数。
- 矢量 $\vec v_4$ 就是（原坐标系的）原点变换之后（在原坐标系下）的位置。

那么就可以算出来 $M$ 的逆矩阵为
$$\begin{aligned}
M^{-1} &=
\begin{bmatrix}
\dfrac{m_0}{{\lVert \vec v_1 \rVert}^2} & \dfrac{m_1}{{\lVert \vec v_1 \rVert}^2} & \dfrac{m_2}{{\lVert \vec v_1 \rVert}^2} & 0 \\
\ \\
\dfrac{m_3}{{\lVert \vec v_2 \rVert}^2} & \dfrac{m_4}{{\lVert \vec v_2 \rVert}^2} & \dfrac{m_5}{{\lVert \vec v_2 \rVert}^2} & 0 \\
\ \\
\dfrac{m_6}{{\lVert \vec v_3 \rVert}^2} & \dfrac{m_7}{{\lVert \vec v_3 \rVert}^2} & \dfrac{m_8}{{\lVert \vec v_3 \rVert}^2} & 0 \\
\ \\
\ 0 & 0 & 0 & 1
\end{bmatrix} \
\begin{bmatrix}
1 & 0 & 0 & -m_9 \\
\ \\
0 & 1 & 0 & -m_{10} \\
\ \\
0 & 0 & 1 & -m_{11} \\
\ \\
0 & 0 & 0 & 1
\end{bmatrix} \\
\ \\ &=\begin{bmatrix}
\dfrac{m_0}{{\lVert \vec v_1 \rVert}^2} & \dfrac{m_1}{{\lVert \vec v_1 \rVert}^2} & \dfrac{m_2}{{\lVert \vec v_1 \rVert}^2} & -\ \dfrac{\vec v_1 \cdot \vec v_4}{{\lVert \vec v_1 \rVert}^2} \\
\ \\
\dfrac{m_3}{{\lVert \vec v_2 \rVert}^2} & \dfrac{m_4}{{\lVert \vec v_2 \rVert}^2} & \dfrac{m_5}{{\lVert \vec v_2 \rVert}^2} & -\ \dfrac{\vec v_2 \cdot \vec v_4}{{\lVert \vec v_2 \rVert}^2} \\
\ \\
\dfrac{m_6}{{\lVert \vec v_3 \rVert}^2} & \dfrac{m_7}{{\lVert \vec v_3 \rVert}^2} & \dfrac{m_8}{{\lVert \vec v_3 \rVert}^2} & -\ \dfrac{\vec v_3 \cdot \vec v_4}{{\lVert \vec v_3 \rVert}^2} \\
\ \\
\ 0 & 0 & 0 & 1
\end{bmatrix}
\end{aligned}$$

#### 错切矩阵

***ToDo***: 待完善

#### 镜像矩阵

***ToDo***: 待完善

#### 正交投影矩阵

假设，近平面到摄像机的距离为 $N$ ,远平面到摄像机的距离为 $F$ ，近平面中心点到近平面的顶边的距离为 $size$ ，近平面的 $width : height = aspect$ 。那么正交投影矩阵为

$$
M_{ortho} = \
\begin{bmatrix}
\dfrac{1}{aspect\ *\ size} &0 &0 &0  \\
\ \\
0 & \dfrac{1}{size} & 0 &0  \\
\ \\
0 & 0 & -\ \dfrac{2}{F - N} & -\ \dfrac{F + N}{F - N}  \\
\ \\
0 & 0 & 0 & 1
\end{bmatrix}
$$

具体推导见 [正交投影矩阵的推导](#正交orthographic投影矩阵推导)

#### 透视投影矩阵

假设，近平面到摄像机的距离为 $N$ ,远平面到摄像机的距离为 $F$ ，摄像机的垂直张角为 $FOV$ 。近平面的 $\dfrac{width}{height} = aspect$ 。那么透视投影矩阵为

$$
M_{frustum} = \
\begin{bmatrix}
\dfrac{\cot(\dfrac{FOV}{2})}{aspect} & 0 & 0 & 0  \\
\ \\
0 & \cot(\dfrac{FOV}{2}) & 0 & 0  \\
\ \\
0 & 0 & -\ \dfrac{F + N}{F - N} & -\ \dfrac{2\ *\ F\ *\ N}{F - N}  \\
\ \\
0 & 0 & -1 & 0
\end{bmatrix}
$$

具体推导见 [透视投影矩阵的推导](#透视perspective投影矩阵推导)

### 坐标空间






## Shader 概述

### 材质（Material）和Unity Shader

总体来说，在Unity中我们需要配合使用材质（Material）和 Unity Shader 才能达到需要的效果。一个最常见的流程是：

1. 创建一个材质；
2. 创建一个 Unity Shader ，并把它赋给上一步中创建的材质；
3. 把材质赋给要渲染的对象；
4. 在材质面板中调整 Unity Shader 的属性，以得到满意的效果。

### Unity Shader 模板

#### Standard Surface Shader

一个包含了标准光照模型的表面着色器模板

#### Unlit Shader

一个不包含光照（但包含雾效）的基本的顶点/片元着色器

#### Image Effect Shader

为实现各种屏幕后处理效果提供了一个基本模板。

#### Compute Shader

这类 Shader 旨在利用 GPU 的并行性来进行一些与常规渲染流水线无关的计算，暂时不管。

#### Ray Tracing Shader

光线追踪，暂时不管。

### Unity Shader的结构

一个 Unity Shader 的基础结构如下所示：

```hlsl
Shader "ShaderName" {
　　Properties {
　　　　// 属性
　　}
　　SubShader {
　　　　// 显卡A使用的子着色器
　　}
　　SubShader {
　　　　// 显卡B使用的子着色器
　　}　　
　　Fallback "VertexLit"
}
```

### Shader 名称

每个 Unity Shader 文件的第一行都需要通过 Shader 语义来指定该 Unity Shader 的名字。这个名字由一个字符串来定义，例如 “MyShader” 。\
当为材质选择使用的 Unity Shader 时，这些名称就会出现在材质 Inspector 的下拉列表里。\
通过在字符串中添加斜杠（“/”），可以控制 Unity Shader 在材质 Inspector 中出现的位置。例如：Shader "Custom/MyShader" {　　 }

### Shader Properties

Properties 是材质和 Unity Shader 的桥梁

Properties 语义块的定义通常如下：

```hlsl
Properties {
　　[optional: attribute] name("display text in Inspector", type name) = default value
　　[optional: attribute] name("display text in Inspector", type name) = default value
　　// 更多属性
}
```

开发者们声明这些属性是为了在材质 Inspector 中能够方便地调整各种材质属性。\
如果我们需要在 Shader 中访问它们，就需要使用每个属性的名称（name）。**在 Unity 中，这些属性的名称通常由一个下划线开始。**\
显示的名称（display text in Inspector）则是出现在材质 Inspector 上的名称。\
我们需要为每个属性指定它的类型（type name）\
除此之外，我们还需要为每个属性指定一个默认值（default value），在我们第一次把该 Unity Shader 赋给某个材质时，材质 Inspector 上显示的就是这些默认值。

***注意：***

- 即使不在 Properties 语义块中声明这些属性，也可以直接在 Shader 代码片中定义变量。无论属性是否存在于 Properties 语义块中，都可以通过 C# 脚本（使用同样的方法）向 Shader 传递这些属性。
- 如果设置了属性， Shader Pass 中的变量名必须和属性名相同。同样的，通过 C# 脚本向 Shader 传递属性时，使用的属性名也必须和 Pass 中的变量名相同。

#### 属性类型

常见的属性类型如下表所示

| 属性类型 | 例子 | 备注 |
| :--- | :--- | :--- |
| Integer | _MyInteger ("My Integer", Integer) = 1 | 这是真正的整型，而不是像 Int 那样实际使用了浮点数。 |
| Int(legacy) | _MyInt("My Integer", Int) = 1 | 实际使用了浮点数。如果需要整型推荐使用 Integer 类型。 |
| Float | _MyFloat("My Float", Float) = 1.0 | 在 Shader 代码中，对应  float, half 或 fixed 类型。 |
| Range | _MyRange("My Range", Range(0.0, 1.0)) = 0.5 | <li>在 Shader 代码中，对应  float, half 或 fixed 类型。<li>范围的最大最小值均包含在内。 |
| Color | _MyColor("My Color", Color) = (.25, .5, .5, 1) | <li>在 Shader 代码中，对应 float4, half4 或 fixed4 类型。<li>通常 fixed4 类型的精度就足够了。<li>在Inspector中默认使用颜色工具，如果需要手动输入数值，使用 Vector 类型。 |
| Vector | _MyVector("My Vector", Vector) = (.25, .5, .5, 1) | 在 Shader 代码中，对应 float4, half4 或 fixed4 类型。 |
| Texture2D | _MyTexture2D("My Texture2D", 2D) = "" {}<br>_MyTexture2D("My Texture2D", 2D) = "red" {} | <li>在 Shader 代码中，对应 sampler2D 类型。<li>Unity 支持以下字符串来使用内置材质:<br>“white” (RGBA: 1,1,1,1)<br>“black” (RGBA: 0,0,0,1)<br>“gray” (RGBA: 0.5,0.5,0.5,1)<br>“bump” (RGBA: 0.5,0.5,1,0.5)<br>“red” (RGBA: 1,0,0,1)<li>如果字符串留空或不正确，则默认使用"gray"<li>这些默认材质在Inspector中是不可见的。 |
| Texture3D | _MyTexture3D("My Texture3D", 3D) = "" {}<br>_MyTexture3D("My Texture3D", 3D) = "white" {} | <li>在 Shader 代码中，对应 sampler3D 类型。<li>字符串内容同 Texture2D |
| Cubemap | _MyCubemap ("My Cubemap", Cube) = "" {}<br>_MyCubemap ("My Cubemap", Cube) = "black" {} | <li>在 Shader 代码中，对应 samplerCUBE 类型。<li>字符串内容同 Texture2D |
| Texture2DArray | _MyTexture2DArray ("My Texture2D Array", 2DArray) = "" {} | 参考[文档](https://docs.unity3d.com/Manual/class-Texture2DArray.html) [Unity 文档](https://docs.unity3d.com/Manual/class-Texture2DArray.html) |
| CubemapArray | _MyCubemapArray ("My Cubemap Array", CubeArray) = "" {} | 参考 [Unity 文档](https://docs.unity3d.com/Manual/class-CubemapArray.html) |

#### 在 Shader 代码中获取属性

想要在 Shader 代码中获取属性，必须满足以下条件

- Shader Pass 中的变量名必须和属性名相同。通过 C# 脚本向 Shader 传递属性时，使用的属性名也必须和 Pass 中的变量名相同。
- 属性类型对应的 Shader Pass 中的变量类型，可以查看 [Shader属性类型](#属性类型) 中的表格。

Unity 为 Shader 传递属性值的顺序如下：

- 每个实例数据（per-instance data）覆盖一切；
- 然后使用材质数据（Material data）；
- 最后，如果这两个地方不存在 Shader 属性，则使用全局属性值（例如：Shader.SetGlobalXXX）。
- 最后，如果在任何地方都没有定义 Shader 属性，那么将提供“默认”值（浮点数为零，颜色为黑色，纹理为空白色纹理）。

#### Shader 代码中 Texture 的特殊属性

材质（Material）中通常可以为纹理设置[缩放和偏移](#纹理缩放tilling和纹理偏移offset)，这些属性在 Shader 代码中可以通过一个类型为 float4 ，名称为 *{TextureName}`_ST`* 的变量来获取，其中，ST是缩放（scale）和平移（translation）的缩写，这个变量包含的数据为

- `x` = Tilling.x
- `y` = Tilling.y
- `z` = Offset.x
- `w` = Offset.y

Shader Pass 中的代码例子如下

```hlsl
sampler2D _MainTex;
float4 _MainTex_ST;

struct a2v {
    float2 uv：TEXCOORD0;
    // 其他属性
};

struct v2f {
　　float2 uv：TEXCOORD0;
    // 其他属性
};

v2f vert(a2v in) {
    v2f out;
    out.uv = in.uv.xy * _MainTex_ST.xy + _MainTex_ST.zw;

    // 或者直接使用内置宏
    // #define TRANSFORM_TEX(tex，name) (tex.xy * name##_ST.xy + name##_ST.zw)
    out.uv = TRANSFORM_TEX(in.uv，_MainTex);
}
```

在 Shader 代码中纹理本身的尺寸属性可以通过类型为 float4 ，名称为 *{TextureName}`_TexelSize`* 的变量来获取，这个变量包含的数据为

- `x` = 1.0 / width
- `y` = 1.0 / height
- `z` = width
- `w` = height

#### 常见的属性装饰（Attribute）

可选的装饰（Attribute）用来控制属性在Inspector中的显示方式。

常见的 Attribute 如下表所示

| Attribute | 说明 |
| :--- | :--- |
| \[Gamma] | <li>指示浮点或矢量属性使用 sRGB 值，这意味着如果项目中的颜色空间需要，它必须与其他 sRGB 值一起转换。<li> *没有找到具体用法。* |
| \[HDR] | <li>指示纹理或颜色属性使用 [HDR](https://docs.unity3d.com/Manual/HDR.html) 值。<li>对于纹理属性，如果分配了 LDR 纹理，Unity 编辑器会显示警告。<li>对于颜色属性，Unity Editor 使用 HDR 颜色选择器来编辑此值。 |
| \[HideInInspector] | 在 Inspector 窗口中隐藏这个属性。 |
| \[MainTexture] | <li>设置材质的主纹理，可以在 C# 脚本中使用 [Material.mainTexture](https://docs.unity3d.com/ScriptReference/Material-mainTexture.html) 访问它。<li>默认情况下，Unity 将属性名称为 `_MainTex` 的纹理视为主纹理。如果主纹理具有不同的属性名称，而且希望 Unity 将其视为主纹理，必须使用这个 Attribute 。<li>如果多次使用这个 Attribute ，Unity 将使用第一个属性并忽略后续属性。<li> *注意：* 如果使用此 Attribute 设置主纹理，在使用纹理流调试视图模式（texture streaming debugging view mode）或自定义调试工具时，纹理在 Game view 中不可见。 |
| \[MainColor] | <li>设置材质的主要颜色，可以在 C# 脚本中使用 [Material.color](https://docs.unity3d.com/ScriptReference/Material-color.html) 访问它。<li>默认情况下，Unity 将属性名称为 `_Color` 的颜色视为主颜色。如果主颜色具有不同的属性名称，而且希望 Unity 将其视为主颜色，必须使用这个 Attribute 。如果多次使用这个 Attribute ，Unity 将使用第一个属性并忽略后续属性。 |
| \[NoScaleOffset] | <li>在 Inspector 窗口中藏此纹理的[缩放和偏移](#纹理缩放tilling和纹理偏移offset)。<li>依然可以在 Shader 和 C# 代码中访问及修改纹理的缩放和偏移
| \[Normal] | <li>指示纹理属性需要法线贴图。<li>如果分配不兼容的纹理，Unity 编辑器会显示警告。 |
| \[PerRendererData] | <li>指示纹理属性将来自每个渲染器数据，形式为 MaterialPropertyBlock 。<li>在 Inspector 窗口中这些属性为只读。<li>*暂时不管*

#### 其他的属性装饰（Attribute）

其他的 Attribute 如下：

##### Toggle

启用或禁用单个 [着色器关键字（Shader Keyword）](#着色器关键字shader-keyword) 。它使用一个浮点数变量来存储着色器关键字的状态，并将其显示为一个开关。启用开关时，Unity 启用着色器关键字，对应的变量为 True ；禁用开关时，Unity 会禁用着色器关键字，对应的变量为 False 。

如果指定关键字名称，则切换会影响具有该名称的 Shader 关键字。如果未指定着色器关键字名称，则切换会影响名称为 `(uppercase property name)_ON` 的着色器关键字。

```hlsl
// 这个例子指定了关键字名称。
// 启用开关时， Unity 启用名称为 "ENABLE_EXAMPLE_FEATURE" 的着色器关键字，对应 _ExampleFeatureEnabled 变量值为 True 。
// 禁用开关时， Unity 禁用名称为 "ENABLE_EXAMPLE_FEATURE" 的着色器关键字，对应 _ExampleFeatureEnabled 变量值为 False 。
[Toggle(ENABLE_EXAMPLE_FEATURE)] _ExampleFeatureEnabled ("Enable example feature", Float) = 0

// 这个例子没有指定关键字名称。
// 启用开关时， Unity 启用名称为 "_ANOTHER_FEATURE_ON" 的着色器关键字，对应 _Another_Feature 变量值为 True 。
// 禁用开关时， Unity 禁用名称为 "_ANOTHER_FEATURE_ON" 的着色器关键字，对应 _Another_Feature 变量值为 False 。
[Toggle] _Another_Feature ("Enable another feature", Float) = 0

// 使用宏：
// 对应的宏定义，推荐使用 #pragma shader_feature
#pragma multi_compile __ ENABLE_EXAMPLE_FEATURE
#pragma shader_feature _ANOTHER_FEATURE_ON
// 宏的用法
#if _ANOTHER_FEATURE_ON
    ApplyEffect();
#endif

// 使用变量：
// 定义变量，变量名必须和属性名相同，类型可以不用 float4
uniform bool _ExampleFeatureEnabled;
// 使用变量
if (_ExampleFeatureEnabled) 
{
    ApplyEffect();
}
```

##### ToggleOff

类似于 Toggle，但是当启用开关时， Unity 会禁用着色器关键字，但是对应的变量为 True ；当禁用开关时， Unity 启用着色器关键字，但是对应的变量为 False 。如果未指定着色器关键字名称，默认名称为 `(uppercase property name)_OFF` 。

ToggleOff 可用于向现有着色器添加功能和切换，同时保持向后兼容性。

```hlsl
// 这个例子指定了关键字名称。
// 启用开关时， Unity 禁用名称为 "DISABLE_EXAMPLE_FEATURE" 的着色器关键字，对应 _ExampleFeatureEnabled 变量值为 True 。
// 禁用开关时， Unity 启用名称为 "DISABLE_EXAMPLE_FEATURE" 的着色器关键字，对应 _ExampleFeatureEnabled 变量值为 False 。
// 注意：这里的宏的名称、变量名、描述三者之间的区别。
[ToggleOff(DISABLE_EXAMPLE_FEATURE)] _ExampleFeatureEnabled ("Enable example feature", Float) = 0

// 这个例子没有指定关键字名称。
// 启用开关时， Unity 禁用名称为 "_ANOTHER_FEATURE_OFF" 的着色器关键字，对应 _Another_Feature 变量值为 True 。
// 禁用开关时， Unity 启用名称为 "_ANOTHER_FEATURE_OFF" 的着色器关键字，对应 _Another_Feature 变量值为 False 。
// 注意：这里的宏的名称、变量名、描述三者之间的区别。
[ToggleOff] _Another_Feature ("Enable another feature", Float) = 0

// 使用宏：
// 对应的宏定义，推荐使用 #pragma shader_feature
#pragma multi_compile __ DISABLE_EXAMPLE_FEATURE
#pragma shader_feature _ANOTHER_FEATURE_OFF
// 宏的用法，注意用的是 #ifndef
#ifndef _ANOTHER_FEATURE_OFF
    ApplyEffect();
#endif

// 使用变量：
// 定义变量，变量名必须和属性名相同，类型可以不用 float4
uniform bool _ExampleFeatureEnabled;
// 使用变量
if (_ExampleFeatureEnabled) 
{
    ApplyEffect();
}
```

##### KeywordEnum

选择启用一组着色器关键字中的哪一个。为一个浮点数变量弹出下拉菜单，浮点数的值决定了 Unity 启用哪个着色器关键字。 Unity 启用名称为 `(uppercase property name)_(uppercase enum value name)` 的着色器关键字，对应的变量值为从 0 开始依次加 1 的整数。最多可以提供 9 个名称。

```hlsl
// 显示下拉菜单选择 None, Add, Multiply 。
// 每个菜单项对应设置着色器关键字 _OVERLAY_NONE, _OVERLAY_ADD, _OVERLAY_MULTIPLY ，这些关键字，只会存在一个。
// 每个菜单项对应 _Overlay 变量值为 0, 1, 2 。
[KeywordEnum(None, Add, Multiply)] _Overlay ("Overlay mode", Float) = 0

// 使用宏：
// 对应的宏定义，推荐使用 #pragma multi_compile
#pragma multi_compile _OVERLAY_NONE _OVERLAY_ADD _OVERLAY_MULTIPLY
// 宏的用法
#if _OVERLAY_MULTIPLY
    ApplyEffect();
#endif

// 使用变量：
// 定义变量，变量名必须和属性名相同，类型可以不用 float4
uniform int _Overlay;
if (_Overlay == 2)
{
    ApplyEffect();
}
```

##### Enum

为一个浮点数变量弹出下拉菜单。可以提供一个枚举类型名称（最好使用名称空间完全限定，以防有多种类型），或者要显示的显式名称/值对。最多可以指定 7 个名称/值对。

```hlsl
// 使用枚举类型名称，注意：（即使是自定义的枚举也）不需要 using C# 文件。
[Enum(UnityEngine.Rendering.BlendMode)] _Blend ("Blend mode", Float) = 1

// 使用名称/值对，其中 "One" 对应变量值为 1 ，"SrcAlpha" 对应变量值为 5 。
[Enum(One,1, SrcAlpha,5)] _Blend2 ("Blend mode subset", Float) = 1

// 使用变量：
// 定义变量，变量名必须和属性名相同，类型可以不用 float4
uniform int _Blend2;
switch (_Blend2)
{
case 1:
    ApplyEffectOne();
    break;
case 5:
    ApplyEffectSrcAlpha();
    break;
}
```

##### PowerSlider

指数级的缩放滑动条。

```hlsl
// 这个例子是一个指数等级为 3.0 ，取值范围在 [0.01, 1] 区间的滑动条。
// 指数值必须大于 0 ，当这个值为 1 时，就是线性滑动条。这个值越大，那么越能精确调整滑动条左侧的值；这个值越小，那么能精确调整滑动条右侧的值。
// 对应的变量必须设定取值范围（属性类型必须是 Range ），但是取值范围的极值可以是 0 或者负数。
[PowerSlider(3.0)] _Shininess ("Shininess", Range (0.01, 1)) = 0.08
```

##### IntRange

限定为整数的滑动条。

```hlsl
// 这个例子是一个整数滑动条，取值范围在 [0, 255] 区间。
// 对应的变量必须设定取值范围（属性类型必须是 Range ），但是取值范围的极值可以是 0 或者负数。
// 即使在 Inspector 输入框中输入小数，也会自动变成整数。
[IntRange] _Alpha ("Alpha", Range (0, 255)) = 100
```

##### Space

在 Inspector 窗口这个属性 **上面** 添加空行。

```hlsl
// 默认的小段空行，比 Space(1) 要大
[Space] _Prop1 ("Prop1", Float) = 0

// 一大段空行
[Space(50)] _Prop2 ("Prop2", Float) = 0
```

##### Header

在 Inspector 窗口这个属性 **上面** 添加一个标题

```hlsl
// 同时添加空行用来隔开标题和属性，稍微好看点。
[Header(A group of things)][Space] _Prop1 ("Prop1", Float) = 0
```

以上内容参考

- [Material Property Attribute 文档](https://docs.unity3d.com/Manual/SL-Properties.html#material-property-attributes)
- [Material Property Drawer 文档](https://docs.unity3d.com/ScriptReference/MaterialPropertyDrawer.html)

#### SubShader

每一个Unity Shader文件可以包含多个SubShader语义块，但最少要有一个。当Unity需要加载这个Unity Shader时，Unity会扫描所有的SubShader语义块，然后选择第一个能够在目标平台上运行的SubShader。如果都不支持的话，Unity就会使用Fallback语义指定的Unity Shader。

Unity 采用以下标准来判断SubShader是否可用

- 硬件是否支持。
- Shader的LOD
- 当前激活的渲染管线。

SubShader语义块中包含的定义如下

```hlsl
SubShader
{
    <optional: LOD>
    <optional: tags>
    <optional: commands>
    <One or more Pass definitions>
}
```

##### SubShader中的LOD

SubShader 中的 LOD(level of detail) 采用数字表示。用于在运行时调整 Unity 使用该 Shader 中的哪个 SubShader 。\
Unity 总是选择第一个 LOD 小于等于 Shader.maximumLOD 的 SubShader 。所以在编写 Shader 的时候，应该把 LOD 较大的 SubShader 放在代码前面。\
在 C# 代码中可以通过 Shader.maximumLOD 来改变某个 Shader 对象的 LOD ，也可以通过 Shader.globalMaximumLOD 来改变所有 Shader 对象的 LOD 。

##### SubShader中的Tags

SubShader 标签是键值对数据。 Unity 使用预定义的键和值来确定如何以及何时使用给定的 SubShader ，也可以使用自定义值创建自己的自定义 SubShader 标签。可以从 C# 代码访问 SubShader 标签，例如 [Material.GetTag](https://docs.unity3d.com/ScriptReference/Material.GetTag.html), [Material.SetOverrideTag](https://docs.unity3d.com/ScriptReference/Material.SetOverrideTag.html) 。

*注意：* SubShader 的 Tags 和 Pass 的 Tags 是不同的，两者不能互换。

格式如下

```hlsl
// 注意：双引号必不可少
Tags { “[name1]” = “[value1]” “[name2]” = “[value2]”}
```

常见的Tags如下表所示
| Tag | 作用 | 可选项及说明 | 备注 |
| :--- | :--- | :--- | :--- |
| RenderPipeline | 指定 SubShade r是否支持 [URP](#urp) 或者 [HDRP](#hdrp) | <li>UniversalRenderPipeline<li>HighDefinitionRenderPipeline |
| Queue | 指定 SubShader 属于哪个渲染队列。<br>数字越小则越先渲染。 | <li>常见可选项参考 [SubShader Tags 中的 Queue](#subshader-tags-中的-queue)<li>[offset] 例如 "Geometry+1" | 在 C# 脚本中，可以通过设置 [Material.renderQueue](https://docs.unity3d.com/ScriptReference/Material-renderQueue.html) 来修改这个值。
| RenderType | 对 SubShader 进行分类 | 用于 [shader replacement](https://docs.unity3d.com/Manual/SL-ShaderReplacement.html) | 对于可编程渲染管线，还没有找到怎么用。***Todo***: 有待学习
| ForceNoShadowCasting | 阻止此物体产生阴影。<br>某些情况下也阻止接受阴影 | <li>True<li>False 默认值 | 如果设置为True，那么：<li>在内置渲染管线中，如果使用 Forward、Legacy Vertex Lit 或 Legacy Deferred 渲染路径，Unity 还会阻止物体接收阴影。<li>在 HDRP 中，这不会阻止物体产生接触阴影。
| DisableBatching | 阻止 Unity 将动态批处理 [Dynamic Batching](#batching) 应用到此 SubShader 上。 | <li>True<li>False 默认值<li>LODFading: Unity 会阻止属于 LODGroup 且 Fade Mode 值不是 None 的所有动态合批。否则，Unity 不会阻止动态合批。| 这对于执行对象空间操作的着色器程序很有用。动态批处理将所有几何体转换为世界空间，这意味着着色器程序无法再访问对象空间。因此，依赖对象空间的着色器程序无法正确渲染。为避免此问题，请使用此 SubShader 标记来阻止 Unity 应用动态合批。
| IgnoreProjector | 阻止此物体接受阴影。 | <li>True<li>False 默认值 | **仅在内置渲染管线中有效**，这主要用于排除和阴影不兼容的半透明物体。
| PreviewType | 设置 SubShader 对应的材质在 Inspector 窗口中的预览模式 | <li>Sphere<li>Plane<li>Skybox
| CanUseSpriteAtlas | 是否允许 Sprite 打包进 ***Legacy Sprite Packer*** | <li>True 默认值<li>False | 如果设置为 False ，那么，当这个 SubShader 应用于 ***Legacy Sprite Packer*** 中的 Sprite 时， Unity 会在 Inspector 窗口中显示错误信息。

***ToDo***：完善 URP 中的 Tags ，参考 [Unity URP 文档](https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@12.1/manual/urp-shaders/urp-shaderlab-pass-tags.html)

##### SubShader Tags 中的 Queue

SubShader Tags 中 Queue 的常见可选项如下表所示
| Queue | Value | 说明 |
| :--- | :--- | :--- |
| Background | 1000 | 这个队列通常会最先渲染。 |
| Geometry | 2000 | 这是默认的渲染队列，它被用于绝大多数对象。不透明的几何体使用该队列。 |
| AlphaTest | 2450 | 需要开启透明度测试的队列。Unity5 之后从 Geometry 队列中拆分出来，因为在所有不透明的物体渲染完之后再渲染会比较高效。 |
| Transparent | 3000 | 所有 Geometry 和 AlphaTest 渲染完后，再（按照从后往前的顺序）进行渲染。任何使用了透明度混合的物体都应该使用该队列（例如玻璃和粒子效果）。 |
| Overlay | 4000 | 该队列用于实现一些叠加效果，适合最后渲染的效果（如镜头光晕）。 |

#### SubShader 中的 Commands

SubShader 中的命令用于设置渲染状态，很多都对应于渲染流程中的 “不可编程但可配置” 阶段。

SubShader 中的命令可以设置它所包含的所有 Pass 的状态。

命令格式如下

```hlsl
Cull Off
ZWrite Off
```

常见的 Commands 如下表所示，下面这些命令既可以用于 SubShader 也可以用于 Pass。

| Command | 作用 | 可选项及说明 | 备注 |
| :--- | :--- | :--- | :--- |
| AlphaToMask | 开关 GPU 的 alpha-to-coverage 功能，此功能用于减少 MSAA 中的过度锯齿化。 | <li>ON<li>Off | 必须同时开启 MSAA 才能正确生效。***Todo***: 待学习
| Blend | 设置片段着色器的输出与渲染目标之间的混合模式。 | 见 [Blend 命令详解](#blend-命令详解) | 启用混合会禁用 GPU 上的某些优化（主要是 hidden surface removal / [Early-Z](#early-z-技术) ），这会导致 GPU 帧时间增加。
| BlendOp | 设置混合的操作。 | 见 [BlendOp 命令详解](#blendop-命令详解) | <li>如果没有设置 BlendOp，则默认为 Add<li>如果设置了 BlendOp，则同一个块中必须设置 Blend |
| ColorMask | 设置颜色通道的掩码 | <li>0 禁止写入 RGBA 通道<li>任何 R、G、B、A 的组合代表允许写入对应的通道 | <li>设置 RenderTarget 的格式为：`ColorMask RGB 2`<li>使用 [Stencil 命令](#stencil-命令详解) 用来单纯设置模板缓冲区时，可能会配合使用 ColorMask 命令来禁止写入任何颜色。<li>***ToDo***：关联深度缓冲相关内容<li>参考 [Unity 文档](https://docs.unity3d.com/Manual/SL-ColorMask.html) |
| Stencil| 配置与模板缓冲区相关的设置 | 见 [Stencil 命令详解](#stencil-命令详解) |
| Cull | 设置剔除模式 | <li>Back 剔除背面<li>Front 剔除正面<li>Off 关闭剔除 | 关于正面背面的判断，参考 [Unity 文档](https://docs.unity3d.com/Manual/AnatomyofaMesh.html) 中 Winding order 小节 |
| ZClip | ***ToDo***：待学习 | <li>True<li>False |
| ZTest | 设置几何体通过或未通过深度测试的条件。 | 见 [ZTest 命令详解](#ztest-命令详解) |
| ZWrite | 设置是否允许写入[深度缓冲区](#深度缓冲区depth-buffer)。 | <li>On<li>Off | <li>只有通过 [深度测试](#ztest-命令详解) 的片元才有资格写入。见 [深度测试流程](#深度测试depth-test流程) 。<li>一般对不透明物体启用深度写入，对半透明物体禁用深度写入。<li>禁用 ZWrite 会导致不正确的深度排序。在这种情况下，需要在 CPU 上对几何图形进行排序。 |
| Conservative | *用于计算？* | <li>True<li>False | <li>*暂时不管*<li>参考 [Unity 文档](https://docs.unity3d.com/Manual/SL-Conservative.html) |
| Offset | ***ToDo***：待学习 |

下面的命令只能用于 SubShader 中

| Command | 作用 | 可选项及说明 | 备注 |
| :--- | :--- | :--- | :--- |
| UsePass | 从另一个 Shader 中，将某个 Pass 添加过来 | 见 [UsePass 命令详解](#usepass-命令详解) |
| GrabPass | 创建一个特殊类型的 Pass 用来将帧缓冲区抓取到一个纹理中 | 见 [GrabPass 命令详解](#grabpass-命令详解) |

### SubShader 中的 Pass

Pass 是着色器对象的基本元素。它包含设置 GPU 状态的说明，以及在 GPU 上运行的着色器程序。每个 Pass 都定义了一次完整的渲染流程。

简单的着色器对象可能只包含一个 Pass ，复杂的着色器可以包含多个 Pass 。可以使用单独的 Pass 来定义工作方式不同的着色器对象部分；例如，需要更改渲染状态的 Pass 、不同 Program 的 Pass 或 不同 LightMode 标签的 Pass 。

如果 Pass 的数目过多，往往会造成渲染性能的下降。因此，我们应尽量使用最小数目的 Pass 。

Pass 的结构如下

```hlsl
Pass
{
    <optional: name>
    <optional: tags>
    <optional: commands>
    <optional: shader code>
}
```

例子如下

```hlsl
Shader "Examples/SinglePass"
{
    SubShader
    {
        Pass
        {                
              Name "ExamplePassName"
              Tags { "ExampleTagKey" = "ExampleTagValue" }

              // ShaderLab commands go here.

              // HLSL code goes here.
        }
    }
}
```

#### Pass 的名称

每个 Pass 都可以有自己的名称，也可以没有名称。

*注意：* 在内部，Unity 将 Pass 名称转换为大写。所以在 ShaderLab 代码中使用 [UsePass 命令](#usepass-命令详解) 引用其他 Shader 中的某个 Pass 时，**必须** 使用大写变体。例如，如果某个 Pass 的名称是 "example" ，则引用时，应该使用的名称为 "EXAMPLE"。

如果同一个 SubShader 中有多个 Pass 具有相同的名称，Unity 将使用代码中的第一个 Pass。

C# 脚本中：

- [Material.FindPass(string passName)](https://docs.unity3d.com/ScriptReference/Material.FindPass.html) 函数中 passName 参数不考虑大小写（ String.EqualsIgnoreCase ）
- [Material.GetPassName(int pass)](https://docs.unity3d.com/ScriptReference/Material.GetPassName.html) 函数的返回值是 Shader 代码中 Pass 的名字，没有转为大写。

#### Pass 中的 Tags

Pass 的标签是键值对数据。 Unity 使用预定义的标签和值来确定如何以及何时渲染给定的 Pass 。还可以使用自定义值创建自己的自定义 Pass 标记，并从 C# 代码访问它们。

格式如下

```hlsl
// 注意：双引号必不可少
Tags {"<name1>" = "<value1>" "<name2>" = "<value2>"}
```

***ToDo***：有待学习 [Unity 文档](https://docs.unity3d.com/Manual/SL-PassTags.html)

#### Pass 中的 Commands

Pass 中的命令和 [SubShader 中的 Commands](#subshader-中的-commands) 基本一致。

#### Pass 中的 ShaderCode

***ToDo***：有待学习 [Unity 文档](https://docs.unity3d.com/Manual/shader-shaderlab-code-blocks.html)

### Shader Fallback

紧跟在各个 SubShader 语义块后面的，可以是一个 Fallback 指令。它用于告诉 Unity ，“如果上面所有的 SubShader 在这块显卡上都不能运行，那么就使用这个最低级的 Shader 吧！”

它的语义如下

```hlsl
Fallback "name"
// 或者
Fallback Off
```

事实上，Fallback还会影响阴影的投射。在渲染阴影纹理时，Unity会在每个Unity Shader中寻找一个阴影投射的Pass。通常情况下，我们不需要自己专门实现一个Pass，这是因为Fallback使用的内置Shader中包含了这样一个通用的Pass。因此，为每个Unity Shader正确设置Fallback是非常重要的。

## SubShader的命令详解

### Blend 命令详解

Blend 决定了GPU如何将片段着色器的输出与渲染目标进行混合。\
启用混合会禁用 GPU 上的某些优化（主要是hidden surface removal/Early-Z），这会导致 GPU 帧时间增加。

Blend 可以在 Pass 或 SubShader 中设置。

*注意：* 绝大多数情况下开启混合模式的时候会设置 ZWrite Off 。***ToDo***：完善ZWrite相关内容。

#### Blend 命令的计算公式

此命令的功能取决于[BlendOp](#blendop-命令详解)，可以使用 BlendOp 命令进行设置。如果开启了混合，则执行以下操作：

- 如果使用 BlendOp 命令，则 *operation* 设置为该值。否则，BlendOp默认为Add。
- 如果 BlendOp 是 Add、Sub、RevSub、Min 或 Max，GPU 会将片段着色器的输出值乘以srcFactor。
- 如果 BlendOp 是 Add、Sub、RevSub、Min 或 Max，GPU 会将片已经存在的目标值乘以dstFactor。
- 按照 *operation* 执行运算，得到最终值。

*注意：* 虽然官网上说 Min、Max 操作会计算 Factor 因子，但是实际测试发现这两个操作完全忽略了 Factor 因子。

公式如下：
>finalValue = (srcFactor \* srcValue) *operation* (dstFactor \* dstValue)

在这个公式中：

- finalValue 表示写入目标缓冲区的值。
- srcFactor 由 Blend 命令指定。
- srcValue 表示片段着色器的输出值。
- *operation* 表示 BlendOp 指定的混合操作。
- dstFactor 由 Blend 命令指定。
- dstValue 表示当前已经存在于目标缓冲区中的值。

#### 常见的 Blend 命令格式

常见的Blend命令格式如下表所示

| Blend格式 | 说明 | 备注
| :--- | :--- | :--- |
| Blend Off | 关闭混合模式。 | Blend 默认就是关闭。
| Blend srcFactor dstFactor | 开启混合，统一为 RGBA 设置 Factor 因子 | 等同于 Blend srcFactor dstFactor, srcFactor dstFactor |
| Blend srcFactorRGB dstFactorRGB, srcFactorAlpha, dstFactorAlpha | 开启混合，分别为 RGB 和 Alpha 设置不同的 Factor 因子 |

*注意：* 以上格式都是在默认的 RenderTarget （也就是屏幕）上操作，可以通过格式 Blend 0 ~ 7 XXX 来指定特定的 render target，例如：`Blend 1 One Zero`\
*关于URP：* 在 URP 管线中设置 RenderTarget 参考 [Unity URP 文档](https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@12.1/manual/rendering-to-a-render-texture.html)

#### 常见的 Factor 格式

常见的Factor因子格式如下表所示

| Factor | 说明 |
| :--- | :--- |
| One | 因子为 1 。用来直接使用 srcValue 或 dstValue 。 |
| Zero | 因子为 0 。用来去除 srcValue 或 dstValue 。 |
| SrcColor | 因子为源颜色值。将源颜色值的 RGBA 分量作为对应分量的混合因子。 |
| SrcAlpha | 因子为源颜色的 Alpha 分量。 |
| SrcAlphaSaturate | 因子为 srcAlpha 和 (1 - dstAlpha) 中的较小值。 |
| DstColor | 因子为目标颜色值。将目标颜色值的 RGBA分 量作为对应分量的混合因子。 |
| DstAlpha | 因子为目标颜色的 Alpha 分量。 |
| OneMinusSrcColor | 因子为 (1 - 源颜色值) 。将结果的 RGBA 分量作为对应分量的混合因子。 |
| OneMinusSrcAlpha | 因子为 (1 - 源颜色的 Alpha 分量)。 |
| OneMinusDstColor | 因子为 (1 - 目标颜色值) 。将结果的 RGBA 分量作为对应分量的混合因子。 |
| OneMinusDscAlpha | 因子为 (1 - 目标颜色的 Alpha 分量)。 |

#### 常见的混合效果及示意图

常见的混合类型如下表所示

| Blend | BlendOp | 效果 | 图片演示 |
| :--- | :---| :--- | :--- |
| Blend SrcAlpha OneMinusSrcAlpha | Add | 传统透明度<br>Traditional transparency | <img src="./images/Blend-DstColor-SrcColor.png" width="50%" height="50%"> |
| Blend OneMinusDstColor One | Add | 柔和相加<br>Soft additive | <img src="./images/Blend-OneMinusDstColor-One.png" width="50%" height="50%"> |
| Blend One OneMinusSrcColor | Add | 同上 | 同上 |
| Blend DstColor Zero | Add | 正片叠底<br>Multiplicative | <img src="./images/Blend-DstColor-Zero.png" width="50%" height="50%"> |
| Blend DstColor SrcColor | Add | 两倍相乘<br>2x multiplicative | <img src="./images/Blend-DstColor-SrcColor.png" width="50%" height="50%"> |
| Blend One One | Add | 叠加<br>Additive | <img src="./images/Blend-One-One.png" width="50%" height="50%"> |
| Blend One One | Min | 变暗<br>Darken | <img src="./images/BlendOp-Min.png" width="50%" height="50%"> |
| Blend One One | Max | 变亮<br>Lighten | <img src="./images/BlendOp-Max.png" width="50%" height="50%"> |

更多细节可以参考 [Unity 文档](https://docs.unity3d.com/Manual/SL-Blend.html)

### BlendOp 命令详解

如果设置了BlendOp，那么在同一个块中**必须**设置[Blend](#blend-命令详解)。“同一个块”的意思是：如果BlendOp在Pass块中，则对应的Blend也要在Pass块中；如果BlendOp在SubShader块中，则对应的Blend也要在SubShader块中。

BlendOp 可以在 Pass 或 SubShader 中设置。

常见的BlendOp如下表所示

| BlendOp | 操作方式 | 备注
| :--- | :--- | :--- |
| Add | Src + Dst | 默认的操作。 |
| Sub | Src - Dst |
| RevSub | Dst - Src |
| Min | min(Src, Dst) | 逐分量比较。测试发现好像在此操作中 Factor 因子失效了。 |
| Max | max(Src, Dst) | 逐分量比较。问题同上。 |
| 其他操作 | 参考 [Unity 文档](https://docs.unity3d.com/Manual/SL-BlendOp.html)

### Stencil 命令详解

Stencil 命令用来配置与模板缓冲区相关的设置。

Stencil 可以在 Pass 或 SubShader 中设置。

#### 模板缓冲区是什么

模板缓冲区实际上是[深度缓冲区](#深度缓冲区depth-buffer)的一部分，模板缓冲区为每个像素存储一个8位整数值。
在执行片段着色器之前对于给定的像素，GPU 可以将模板缓冲区中的当前值与给定的参考值进行比较。这称为[模板测试](#模板测试stencil-test流程)。
如果模板测试通过，GPU 将执行深度测试。如果模板测试失败，GPU 将跳过该像素的其余处理。
这意味着可以使用模板缓冲区作为掩码来告诉 GPU 绘制哪些像素以及丢弃哪些像素。

#### Stencil 命令的用法

使用 Ref、ReadMask 和 Comp 参数来配置模板测试。使用 Ref、WriteMask、Pass、Fail 和 ZFail 参数来配置模板写入。

可以使用 Stencil 做两件不同的事情：配置模板测试，以及配置 GPU 写入模板缓冲区的内容。
可以在同一个命令中执行这两项操作，最常见的用例是创建一个 Shader 对象屏蔽其他着色器对象无法绘制的屏幕区域。
为此，需要将第一个 Shader 对象配置为始终通过模板测试并写入模板缓冲区，并将其他对象配置为执行模板测试而不写入模板缓冲区。

#### 模板测试的运算式

```hlsl
(ref & readMask) comparisonFunction (stencilBufferValue & readMask)
```

#### Stencil 表达式

```hlsl
Stencil
{
    Ref <ref>
    ReadMask <readMask>
    WriteMask <writeMask>
    Comp <comparisonOperation>
    Pass <passOperation>
    Fail <failOperation>
    ZFail <zFailOperation>
    CompBack <comparisonOperationBack>
    PassBack <passOperationBack>
    FailBack <failOperationBack>
    ZFailBack <zFailOperationBack>
    CompFront <comparisonOperationFront>
    PassFront <passOperationFront>
    FailFront <failOperationFront>
    ZFailFront <zFailOperationFront>
}
```

*注意：* 上面的参数 **全部** 都是可选的，例子如下：

```hlsl
Stencil
{
    Ref 2
    Comp equal
    Pass keep
    ZFail decrWrap
}
```

#### Stencil 参数说明

| 参数 | 取值范围 | 说明 |
| :--- | :--- | :--- |
| Ref | [0, 255]区间内的整数。<br>默认为 0 | 参考值。<br>使用 Comp 中定义的操作将模板缓冲区的当前内容与该值进行比较。<br>该值会和 ReadMask 或 WriteMask 进行按位与运算，具体取决于发生的是读操作还是写操作。<br>如果 Pass、Fail 或 ZFail 的值为 Replace，会将此值写入模板缓冲区。 |
| ReadMask | [0, 255]区间内的整数。<br>默认为 255 | 执行模板测试时的掩码。<br>具体参考上面的[运算式](#模板测试的运算式)。 |
| WriteMask | [0, 255]区间内的整数。<br>默认为 255 | 写入模板缓冲时的掩码。<br>*注意：* 它指定是操作中包含哪些位。例如，WriteMask 0 表示写入操作中不包含任何位，而不是模板缓冲区接收到数值 0 |
| Comp | 见[Comparison 操作可选项](#stencil-comparison-操作可选项)<br>默认为 Always | 为所有像素的模板测试执行此操作。<br>这定义了所有像素的操作，无论是正面还是背面。如果除 CompBack 和 CompFront 之外还定义了 Comp ，则 Comp 将覆盖它们。 |
| CompBack | 同上。 | 仅对背面像素生效。如果同时定义了 Comp ，那么 Comp 将覆盖 CompBack 。 |
| CompFront | 同上。 | 仅对正面像素生效。如果同时定义了 Comp ，那么 Comp 将覆盖 CompFront 。 |
| Pass | 见[写入操作可选项](#stencil-写入操作可选项)<br>默认为 Keep | 当像素同时通过模板测试和深度测试时，在模板缓冲区上执行的操作。<br>这定义了所有像素的操作，无论是正面还是背面。如果除 PassBack 和 PassFront 之外还定义了 Pass ，则 Pass 会覆盖它们。 |
| PassBack | 同上。 | 仅对背面像素生效。如果同时定义了 Pass ，那么 Pass 会覆盖 PassBack 。 |
| PassFront | 同上。 | 仅对正面像素生效。如果同时定义了 Pass ，那么 Pass 会覆盖 PassFront 。 |
| Fail | 见[写入操作可选项](#stencil-写入操作可选项)<br>默认为 Keep | 当像素未通过模板测试时，在模板缓冲区上执行的操作。<br>这定义了所有像素的操作，无论是正面还是背面。如果除 FailBack 和 FailFront 之外还定义了 Fail ，则 Fail 会覆盖它们。 |
| FailBack | 同上。 | 仅对背面像素生效。如果同时定义了 Fail ，那么 Fail 会覆盖 FailBack 。 |
| FailFront | 同上。 | 仅对正面像素生效。如果同时定义了 Fail ，那么 Fail 会覆盖 FailFront 。 |
| ZFail | 见[写入操作可选项](#stencil-写入操作可选项)<br>默认为 Keep | 当像素通过模板测试但未通过深度测试时，在模板缓冲区上执行的操作。<br>这定义了所有像素的操作，无论是正面还是背面。如果除了 ZFailBack 和 ZFailFront 之外还定义了 ZFail ，则 ZFail 将覆盖它们。 |
| ZFailBack | 同上。 | 仅对背面像素生效。如果同时定义了 ZFail ，那么 ZFail 会覆盖 ZFailBack 。 |
| ZFailFront | 同上。 | 仅对背面像素生效。如果同时定义了 ZFail ，那么 ZFail 会覆盖 ZFailFront 。 |

#### Stencil Comparison 操作可选项

在 C# 脚本中，可以通过枚举变量 [Rendering.CompareFunction](https://docs.unity3d.com/ScriptReference/Rendering.CompareFunction.html) 访问这些值。

| Shader 中的值 | C# 枚举对应的整数值 | 功能 |
| :--- | :--- | :--- |
| Never | 1 | [运算式](#模板测试的运算式) 直接返回 False |
| Less | 2 | [运算式](#模板测试的运算式) 为 左边 \< 右边 |
| Equal | 3 | [运算式](#模板测试的运算式) 为 左边 == 右边 |
| LEqual | 4 | [运算式](#模板测试的运算式) 为 左边 \<= 右边 |
| Greater | 5 | [运算式](#模板测试的运算式) 为 左边 \> 右边 |
| NotEqual | 6 | [运算式](#模板测试的运算式) 为 左边 \!= 右边 |
| GEqual | 7 | [运算式](#模板测试的运算式) 为 左边 \>= 右边 |
| Always| 8 | [运算式](#模板测试的运算式) 直接返回 True |

#### Stencil 写入操作可选项

在 C# 脚本中，可以通过枚举变量 [Rendering.StencilOp](https://docs.unity3d.com/ScriptReference/Rendering.StencilOp.html) 访问这些值。

| Shader 中的值 | C# 枚举对应的整数值 | 功能 |
| :--- | :--- | :--- |
| Keep | 0 | 保持模板缓冲中的值不变。 |
| Zero | 1 | 将模板缓冲中的值设置为0。 |
| Replace | 2 | 将 Ref 值写入模板缓冲。<br>*注意：* 不是将 (Ref & WriteMask) 的值写入模板缓冲，而是 WriteMask 控制写入哪些位。 |
| IncrSat | 3 | 模板缓冲中的值 +1 。如果模板缓冲中的值是 255 ，则保持为 255 不变。 |
| DecrSat | 4 | 模板缓冲中的值 -1 。如果模板缓冲中的值是 0 ，则保持为 0 不变。 |
| Invert | 5 | 模板缓冲中的值按位取反。 |
| IncrWrap | 6 | 模板缓冲中的值 +1 。如果模板缓冲中的值是 255 ，则变成 0 。 |
| DecrWrap | 7 | 模板缓冲中的值 -1 。如果模板缓冲中的值是 0 ，则变为 255 。 |

#### Stencil 命令的例子

```hlsl
Shader "Examples/CommandExample"
{
    SubShader
    {
        Pass
        {    
            // All pixels in this Pass will pass the stencil test and write a value of 2 to the stencil buffer
            // You would typically do this if you wanted to prevent subsequent shaders from drawing to this area of the render target or restrict them to render to this area only
            Stencil
            {
                Ref 2
                Comp Always
                Pass Replace
            }
        }
    }
}

Shader "Examples/CommandExample"
{
    SubShader
    {
        // All pixels in this SubShader pass the stencil test only if the current value of the stencil buffer is less than 2
        // You would typically do this if you wanted to only draw to areas of the render target that were not "masked out"
        Stencil
        {
            Ref 2
            Comp Less
        }
    }
}
```

### ZTest 命令详解

ZTest 用来设置几何体通过深度测试的条件。见[深度测试流程](#深度测试depth-test流程)

深度测试允许具有 [Early-Z](#early-z-技术) 功能的 GPU 在渲染流程的早期拒绝几何体，并确保几何体的正确排序。可以更改深度测试的条件以实现对象遮挡等视觉效果。

ZTest 可以在 Pass 或 SubShader 中设置。

常见的 ZTest 如下表所示

| ZTest | 功能 |
| :--- | :--- |
| Less | 当前片元片元位于深度缓冲区中片元的前面时，才绘制当前片元。 |
| LEqual | 当前片元位于深度缓冲区中片元的前面，或深度值相等时，才绘制当前片元。<br>这是默认值。 |
| Equal | 当前片元与深度缓冲区中片元深度值相同时，才绘制当前片元。 |
| GEqual | 当前片元位于深度缓冲区中片元的后面，或深度值相等时，才绘制当前片元。 |
| Greater | 当前片元位于深度缓冲区中片元的后面时，才绘制当前片元。 |
| NotEqual | 当前片元与深度缓冲区中片元深度值不同时，才绘制当前片元。 |
| Always | 不进行深度测试。无论深度值如何，绘制所有片元。 |

### UsePass 命令详解

UsePass 用来将另外一个 Shader 中的 Pass 添加过来，由此减少代码重复量。

UsePass 的命令格式为

```hlsl
UsePass "Shader名称/大写的Pass名称"
```

如果对应名称的 Shader 包含多个 SubShader，Unity 将遍历 SubShader，直到它找到第一个包含具有给定名称的 Pass 的受支持的 SubShader。

如果 SubShader 包含多个具有相同名称的 Pass ， Unity 将返回它找到的 **最后一个** Pass。

如果 Unity 没有找到匹配的 Pass，它会显示错误着色器（紫色）。

例子如下

```hlsl
// 这个 Shader 创建了一个有名称的 Pass
Shader "Examples/ContainsNamedPass"
{
    SubShader
    {
        Pass
        {    
              Name "Pass1"
            
              // The rest of the Pass contents go here.
        }
    }
}

// 这个 Shader 引用了上面的 Pass
Shader "Examples/UsesNamedPass"
{
    SubShader
    {
        UsePass "Examples/ContainsNamedPass/PASS1"
    }
}
```

### GrabPass 命令详解

***ToDo***：待学习 [Unity 文档](https://docs.unity3d.com/Manual/SL-GrabPass.html)

## 其他内容

### 着色器关键字（Shader Keyword）

***ToDo***：关于 Shader Keyword 可以从下面的文档继续学习

- [Shader 变体和关键字 文档](https://docs.unity3d.com/Manual/shader-variants-and-keywords.html)
- [Pragma 指令 文档](https://docs.unity3d.com/Manual/SL-PragmaDirectives.html)

## 名词解释

### ShaderLab

在 Unity 中，所有的 Unity Shader 都是使用 ShaderLab 来编写的。 ShaderLab 是 Unity 提供的编写 Unity Shader 的一种说明性语言。\
它使用了一些嵌套在花括号内部的语义（syntax）来描述一个 Unity Shader 文件的结构。
这些结构包含了许多渲染所需的数据，例如 Properties 语句块中定义了着色器所需的各种属性，这些属性将会出现在材质面板中。\
从设计上来说， ShaderLab 类似于 CgFX 和 Direct3D Effects （.FX）语言，它们都定义了要显示一个材质所需的所有东西，而**不仅仅是着色器代码**。\
基本结构可以参考[Unity Shader的结构](#unity-shader的结构)

### Unity Shader 或 ShaderLab 文件

在 Unity 里， Unity Shader 实际上指的就是一个 ShaderLab 文件——硬盘上以 .shader 作为文件后缀的一种文件。
在Unity Shader（或者说是ShaderLab文件）里，我们可以做的事情远多于一个传统意义上的 Shader 。

- 在传统的 Shader 中，我们仅可以编写特定类型的 Shader ，例如顶点着色器、片元着色器等。而在 Unity Shader 中，我们可以在同一个文件里同时包含需要的顶点着色器和片元着色器代码。
- 在传统的 Shader 中，我们无法设置一些渲染设置，例如是否开启混合、深度测试等，这些是开发者在另外的代码中自行设置的。而在 Unity Shader 中，我们通过一行特定的指令就可以完成这些设置。
- 在传统的 Shader 中，我们需要编写冗长的代码来设置着色器的输入和输出，要小心地处理这些输入输出的位置对应关系等。
而在 Unity Shader 中，我们只需要在特定语句块中声明一些属性，就可以依靠材质来方便地改变这些属性。
而且对于模型自带的数据（如顶点位置、纹理坐标、法线等）， Unity Shader 也提供了直接访问的方法，不需要开发者自行编码来传给着色器。

### NDC

归一化的设备坐标（Normalized Device Coordinates）

### Early-Z 技术

作为一个想充分提高性能的 GPU ，它会希望尽可能早地知道哪些片元是会被舍弃的，对于这些片元就不需要再使用片元着色器来计算它们的颜色。\
在Unity给出的渲染流水线中，我们也可以发现它给出的深度测试是在片元着色器之前。这种将深度测试提前执行的技术通常也被称为 Early-Z 技术。

### HLSL

DirectX 的 HLSL （High Level Shading Language）\
由微软控制着色器的编译，就算使用了不同的硬件，同一个着色器的编译结果也是一样的（前提是版本相同）。但也因此支持 HLSL 的平台相对比较有限，几乎完全是微软自已的产品。这是因为在其他平台上没有可以编译 HLSL 的编译器。

### GLSL

OpenGL 的 GLSL （OpenGL Shading Language）\
GLSL 的优点在于它的跨平台性，，但这种跨平台性是由于 OpenGL 没有提供着色器编译器，而是由显卡驱动来完成着色器的编译工作。换句话说， GLSL 是依赖硬件，而非操作系统层级的。但这也意味着 GLSL 的编译结果将取决于硬件供应商，他们对 GLSL 的编译实现不尽相同，这可能会造成编译结果不一致的情况，因为这完全取决于供应商的做法。

### CG

NVIDIA 的 CG （C for Graphic）\
CG 是真正意义上的跨平台。它会根据平台的不同，编译成相应的中间语言。 CG 语言的跨平台性很大原因取决于与微软的合作，这也导致 CG 语言的语法和 HLSL 非常相像， CG 语言可以无缝移植成 HLSL 代码。但缺点是可能无法完全发挥出 OpenGL 的最新特性。

### Batching

批处理（Batching）就是把很多小的 DrawCall 合并成一个大的 Draw Call 。参考 [Unity 文档](https://docs.unity3d.com/Manual/DrawCallBatching.html)

### URP

通用渲染管线 Universal Render Pipeline

### HDRP

高清渲染管线 High Definition Render Pipeline

### 纹理采样（Texture Sampling）

纹理采样是通过 GPU 读取纹理的过程。图形硬件（显卡）中嵌入了一组纹理单元，这些硬件单元能够直接读取纹理像素或使用不同的算法对这些纹理进行采样。

纹理采样选项包括[环绕模式](#纹理环绕模式wrap-modes)和[过滤模式](#纹理过滤模式filtering-mode)。

纹理采样选项本身并不是纹理选项，而是关于着色器如何使用其纹理采样器读取纹理的更多配置。\
但是在 Unity 中，这个属性可以在纹理的 Inspector 窗口中进行设置。

参考 [LearnOpenGL教程](https://learnopengl-cn.github.io/01%20Getting%20started/06%20Textures/#_2) 、 [Unity 文档](https://docs.unity3d.com/Manual/SL-SamplerStates.html)

### 纹理环绕模式（Wrap Modes）

纹理的环绕模式是处理 [0, 1] UV 范围外行为的采样器的配置。可以为每个(UV)轴设置不同的包裹模式：Repeat (Wrap) , Clamp 或者 Mirror。

<img src="./images/纹理WrapMode示意图.png"><br>*▲ 纹理WrapMode示意图。注：Unity 的 Clamp 就是 GL_CLAMP_TO_EDGE*</img>

### 纹理过滤模式（Filtering Mode）

纹理的过滤模式是像素在屏幕上绘制时的行为方式。例如，旧设备（PlayStation 1）无法执行纹理过滤以将一种像素颜色混合到另一种颜色，缺少过滤会导致所有像素都可见。现代引擎可以让你完全禁用过滤，这就是所谓的点过滤 Point Filtering（也叫邻近过滤，Nearest Neighbor Filtering）。除了 Point Filtering，常见的有 Bilinear 和 Trilinear 。

### 多级渐远纹理（Mipmap）

想象一下，假设我们有一个包含着上千物体的大房间，每个物体上都有纹理。有些物体会很远，但其纹理会拥有与近处物体同样高的分辨率。由于远处的物体可能只产生很少的片段，OpenGL从高分辨率纹理中为这些片段获取正确的颜色值就很困难，因为它需要对一个跨过纹理很大部分的片段只拾取一个纹理颜色。在小物体上这会产生不真实的感觉，更不用说对它们使用高分辨率纹理浪费内存的问题了。\
多级渐远纹理(Mipmap)就是用来解决这个问题的。它简单来说就是一系列的纹理图像，后一个纹理图像是前一个的二分之一。多级渐远纹理背后的理念很简单：距观察者的距离超过一定的阈值，引擎会使用不同的多级渐远纹理，即最适合物体的距离的那个。由于距离远，解析度不高也不会被用户注意到。同时，多级渐远纹理另一加分之处是它的性能非常好。多级渐远纹理例子如下图：

<img src="./images/多级渐远纹理示意图.png"><br>*▲ 多级渐远纹理示意图*</img>

### 纹理缩放（Tilling）和纹理偏移（Offset）

在[纹理采样](#纹理采样texture-sampling)时，通过 Tilling 对输入的UV进行缩放，通过 Offset 对缩放后的 UV 进行偏移。\
使用函数表示如下

```hlsl
void Unity_TilingAndOffset_float(float2 UV, float2 Tiling, float2 Offset, out float2 Out)
{
    Out = UV * Tiling + Offset;
}
```

Tilling 和 Offset 的最终结果和纹理的[环绕模式](#纹理环绕模式wrap-modes)以及[过滤模式](#纹理过滤模式filtering-mode)有关。

在 Unity 中，纹理的缩放和偏移可以在材质的 Inspector 窗口中进行设置。

C# 脚本中，可以通过 [Material.SetTextureScale](https://docs.unity3d.com/ScriptReference/Material.SetTextureScale.html) 来设置 Tilling ，通过 [Material.SetTextureOffset](https://docs.unity3d.com/ScriptReference/Material.SetTextureOffset.html) 来设置 Offset 。

### 深度缓冲区（Depth Buffer）

是 [Render Texture](#运行时纹理render-texture) 的一部分。用于保存深度信息和[模板信息](#stencil-命令详解)

### 运行时纹理(Render Texture)

运行时纹理（渲染纹理）是 Unity 在运行时创建和更新的纹理。具体参考 [Unity 文档](https://docs.unity3d.com/Manual/class-RenderTexture.html)

### 基矢量（basis vector）

笛卡尔坐标系中的坐标轴也被成为基矢量。

### 正交基（orthonormal basis）

如果坐标系中的坐标轴之间互相垂直，那么这样的 [基矢量](#基矢量basis-vector) 被称为正交基。 *注意：* 正交基的单位长度不一定相同。

### 标准正交基（orthonormal basis）

如果坐标系中的坐标轴之间互相垂直，而且单位长度都为 1 。那么这样的 [基矢量](#基矢量basis-vector) 被称为标准正交基。

### 点（point）

点（point）是n维空间（游戏中主要使用二维和三维空间）中的一个位置，它没有大小、宽度这类概念。在笛卡儿坐标系中，我们可以使用2个或3个实数来表示一个点的坐标，如： $p = (x_p,\ y_p)$ ，表示二维空间的点， $p = (x_p,\ y_p,\ z_p)$ 表示三维空间中的点。

### 矢量（vector）

矢量（vector，也被称为向量）是指n维空间中一种包含了[模（magnitude）](#矢量的模magnitude)和[方向（direction）](#矢量的方向direction)的有向线段。

矢量的表示方法和点类似。我们可以使用 $\vec v = (x_v,\ y_v)$ 来表示二维矢量，用 $\vec v = (x_v,\ y_v,\ z_v)$ 来表示三维矢量，用 $\vec v = (x_v,\ y_v,\ z_v,\ w_v)$ 来表示四维矢量。

矢量只有模和方向两个属性，并没有位置信息。只要矢量的模和方向保持不变，无论放在哪里，都是同一个矢量。

### 矢量的模（magnitude）

矢量的模是一个标量，可以理解为是矢量在空间中的长度。矢量 $\vec v$ 的模，通常用符号 $\lVert \vec v \rVert$ 来表示。\
三维矢量的模的计算公式如下

$$
 \lVert \vec v \rVert = \sqrt{x_v^2 + y_v^2 + z_v^2}
$$

$n$ 维矢量 $\vec V=(V_1, V_2, \cdots, V_n)$ 的模的计算公式如下

$$
\lVert \vec V \rVert = \sqrt{\sum_{i=1}^n V_i^2}
$$

*注意：* 当我们只需要比较距离的大小时，应该避免使用 magnitude 属性，而是使用 sqrMagnitude 属性，从而避免（消耗大量性能的）开方运算。

```csharp
if (heading.sqrMagnitude < maxRange * maxRange) {
    // Target is within range.
}
```

### 矢量的方向（direction）

矢量[归一化(normalize)](#矢量归一化normalize)之后得到的[单位矢量](#单位矢量unit-vector)就是它的方向。\
通常用符号 $\hat{v}$ 来表示矢量 $\vec v$ 的方向

归一化是计算动作，单位矢量是这个计算的结果。在 Unity 中 Normalize 函数表示这个计算动作， normalized 属性表示这个计算的结果。

### 矢量归一化（normalize）

对任何给定的非零矢量，把它转换成[单位矢量](#单位矢量unit-vector)的过程就被称为归一化（normalization）。\
可以通过将矢量除以其自身的[模](#矢量的模magnitude)来进行归一化。

*注意：* 零矢量（即矢量的每个分量值都为 0 ，如 $\vec v = (0,\ 0,\ 0)$ ）是不可以被归一化的。

单位矢量的计算方法如下

```csharp
var distance = heading.magnitude;
var direction = heading / distance; // This is now the normalized direction.
```

*注意：* 当需要同时使用 magnitude 和 normalized 属性时，应该使用上面的代码来自行计算，而不是分别访问这两个属性，因为每次访问这些属性都会进行开方运算（上面的代码节省了1次开放运算）。

### 单位矢量（unit vector）

单位矢量指的是那些[模](#矢量的模magnitude)为 1 的矢量。单位矢量也被称为被归一化的矢量（normalized vector）。

从几何意义上看，对二维空间来说，我们可以画一个单位圆，那么单位矢量就可以是从圆心出发、到圆边界的矢量。在三维空间中，单位矢量就是从一个单位球的球心出发、到达球面的矢量。

<img src="./images/二维单位矢量示意图.png"><br>*▲  二维空间的单位矢量都会落在单位圆上*</img>

### 行矩阵（row matrix）

只有一行（ $1 \times n$ ）的矩阵，被称为行矩阵。

### 列矩阵（column matrix）

只有一列（ $n \times 1$ ）的矩阵，被称为列矩阵。

### 方块矩阵（square matrix）

方块矩阵（square matrix），简称方阵，是指那些行和列数目相等的矩阵。在三维渲染里，最常使用的就是 $3 \times 3$ 和 $4 \times 4$ 的方阵。

### 方阵的对角元素（diagonal elements）

[方阵](#方块矩阵square-matrix) 的对角元素指的是行号和列号相等的元素。

### 对角矩阵（diagonal matrix）

如果一个 [方阵](#方块矩阵square-matrix) 除了 [对角元素](#方阵的对角元素diagonal-elements) 外的所有元素都为0，那么这个矩阵就叫做 对角矩阵(diagonal matrix) 。

### 单位矩阵（identity matrix）

[对角元素](#方阵的对角元素diagonal-elements) 都是 1 的 [单位矩阵](#对角矩阵diagonal-matrix) 被称为单位矩阵，用 $I_n$ 来表示。

任何矩阵和单位矩阵相乘的结果都还是原来的矩阵。也就是说，\
$MI = IM = M$\
这就跟标量中的数字 1 一样

### 统一缩放（uniform scale）

对于沿坐标轴的缩放，如果各坐标轴的缩放系数相同，则称为统一缩放（uniform scale）。

### 非统一缩放（nonuniform scale）

对于沿坐标轴的缩放，如果各坐标轴的缩放系数不相同，则称为非统一缩放（nonuniform scale）。

## 推导过程

### [正交矩阵](#正交矩阵orthogonal-matrix) 就是由一组 [标准正交基](#标准正交基orthonormal-basis) 组成的推导过程

对于 $3 \times 3$ 的正交矩阵 $M$ ，我们有：

$$
\begin{aligned}
M^T M \
& = \
\begin{bmatrix}
\ - & \vec v_1 & - \ \\
\ - & \vec v_2 & - \ \\
\ - & \vec v_3 & - \
\end{bmatrix} \
\begin{bmatrix}
\ | & | & | \ \\
\ \vec v_1 & \vec v_2 & \vec v_3 \ \\
\ | & | & | \
\end{bmatrix} \\
\ \\
& = \
\begin{bmatrix}
\vec v_1 \cdot \vec v_1 & \vec v_1 \cdot \vec v_2 & \vec v_1 \cdot \vec v_3 \\
\ \\
\vec v_2 \cdot \vec v_1 & \vec v_2 \cdot \vec v_2 & \vec v_2 \cdot \vec v_3 \\
\ \\
\vec v_3 \cdot \vec v_1 & \vec v_3 \cdot \vec v_2 & \vec v_3 \cdot \vec v_3
\end{bmatrix} \\
\ \\
& = \
\begin{bmatrix}
1 & 0 & 0 \\
0 & 1 & 0 \\
0 & 0 & 1
\end{bmatrix} \\
\ \\
& = \
I
\end{aligned}
$$

那么

$$
\begin{aligned}
\vec v_1 \cdot \vec v_1 = 1 \quad , \quad \vec v_1 \cdot \vec v_2 = 0 \quad , \quad \vec v_1 \cdot \vec v_3 = 0 \\
\ \\
\vec v_2 \cdot \vec v_1 = 0 \quad , \quad \vec v_2 \cdot \vec v_2 = 1 \quad , \quad \vec v_2 \cdot \vec v_3 = 0 \\
\ \\
\vec v_3 \cdot \vec v_1 = 0 \quad , \quad \vec v_3 \cdot \vec v_2 = 0 \quad , \quad \vec v_3 \cdot \vec v_3 = 1
\end{aligned}
$$

由此可以得到以下结论：

- 矩阵的每一行，即 $\vec v_1,\ \vec v_2,\ \vec v_3$ 是[单位矢量](#单位矢量unit-vector)，因为只有这样它们与自己的点积才能是 1
- 矩阵的每一行，即 $\vec v_1,\ \vec v_2,\ \vec v_3$ 之间互相垂直，因为只有这样它们互相之间的点积才能是 0
- 上述两条结论对矩阵的每一列同样适用，因为如果 $M$ 是正交矩阵的话， $M^T$ 也会是正交矩阵。

我们会发现 [标准正交基](#标准正交基orthonormal-basis) 正好符合上述条件。所以正交举证就是由一组标准正交基组成的。

### 正交（Orthographic）投影矩阵推导

参考 [OpenGL的推导过程](http://www.songho.ca/opengl/gl_projectionmatrix.html)

前置数据如下

近平面到摄像机的距离为 $N$ ,远平面到摄像机的距离为 $F$ ，近平面中心点到近平面的顶边的距离为 $size$ ，近平面的 $width : height = aspect$ 。\
设近平面顶边的 $y$ 坐标为 $t$ ，近平面右边 $x$ 坐标为 $r$ 。那么

$$
\begin{aligned}
t &= size\\
\dfrac{r}{t} &= aspect \ \Rightarrow\ r = aspect\ *\ size
\end{aligned}
$$

对于齐次坐标为 $\bar p=(x, y, z, 1)^T$ 的点进行矩阵运算\
*注意：* 因为摄像机空间的 $z$ 轴反向了，所以近平面和远平面的 $z$ 坐标分别为 $-N$ 和 $-F$ 。

$$
\begin{bmatrix}
S_x & 0 & 0 & 0 \\
\ \\
0 & S_y & 0 & 0 \\
\ \\
0 & 0 & A & B \\
\ \\
0 & 0 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
x \\
\ \\
y  \\
\ \\
z  \\
\ \\
1
\end{bmatrix}
= \
\begin{bmatrix}
S_x * x  \\
\ \\
S_y * y  \\
\ \\
A * z + B  \\
\ \\
1
\end{bmatrix}
$$

那么

$$
\begin{aligned}
&\begin{cases}
\begin{aligned}
\text{当点位于近平面右边，即 }x = r\text{ 的时候， } &{S_x * (aspect\ *\ size) = 1}  \\
\ \\
\text{当点位于近平面顶边，即 }y = t\text{ 的时候， } &{S_y * (size) = 1}
\end{aligned}
\end{cases}
\Rightarrow
\begin{cases}
S_x = \dfrac{1}{aspect\ *\ size}  \\
\ \\
S_y = \dfrac{1}{size}
\end{cases}  \\
\ \\
&\begin{cases}
\begin{aligned}
\text{当点位于近平面，即 }z = -N\text{ 的时候， } &{A * (-N) + B = -1}  \\
\ \\
\text{当点位于远平面，即 }z = -F\text{ 的时候， } &{A * (-F) + B = 1}
\end{aligned}
\end{cases}
\Rightarrow
\begin{cases}
A = -\ \dfrac{2}{F - N}  \\
\ \\
B = -\ \dfrac{F + N}{F - N}
\end{cases}
\end{aligned}
$$

由此得到正交投影的变换矩阵为

$$
M_{ortho} = \
\begin{bmatrix}
\dfrac{1}{aspect\ *\ size} &0 &0 &0  \\
\ \\
0 & \dfrac{1}{size} & 0 &0  \\
\ \\
0 & 0 & -\ \dfrac{2}{F - N} & -\ \dfrac{F + N}{F - N}  \\
\ \\
0 & 0 & 0 & 1
\end{bmatrix}
$$

### 透视（Perspective）投影矩阵推导

参考 [OpenGL的推导过程](http://www.songho.ca/opengl/gl_projectionmatrix.html)

前置数据如下

近平面到摄像机的距离为 $N$ ,远平面到摄像机的距离为 $F$ ，摄像机的垂直张角为 $FOV$ 。近平面的 $\dfrac{width}{height} = aspect$ 。\
设近平面顶边的 $y$ 坐标为 $t$ ，近平面右边 $x$ 坐标为 $r$ 。那么

$$
\begin{aligned}
\tan({\dfrac{FOV}{2}}) = \dfrac{t}{N} \ &\Rightarrow\ t = N\ *\ \tan({\dfrac{FOV}{2}}) \\
\ \\
aspect = \dfrac{r}{t}\ &\Rightarrow\ r = N\ *\ aspect\ *\ \tan({\dfrac{FOV}{2}})
\end{aligned}
$$

对于齐次坐标为 $\bar p=(x, y, z, 1)^T$ 的点进行矩阵运算\
*注意：* 因为摄像机空间的 $z$ 轴反向了，所以近平面和远平面的 $z$ 坐标分别为 $-N$ 和 $-F$ 。

$$
\begin{bmatrix}
S_x & 0 & 0 & 0  \\
\ \\
0 & S_y & 0 & 0  \\
\ \\
0 & 0 & A & B  \\
\ \\
0 & 0 & -1 & 0
\end{bmatrix}
\begin{bmatrix}
x  \\
\ \\
y  \\
\ \\
z  \\
\ \\
1
\end{bmatrix}
= \
\begin{bmatrix}
S_x * x  \\
\ \\
S_y * y  \\
\ \\
A * z + B  \\
\ \\
-z
\end{bmatrix}
= \
\begin{bmatrix}
\dfrac{S_x\ *\ x}{-z}  \\
\ \\
\dfrac{S_y\ *\ y}{-z}  \\
\ \\
\dfrac{A * z + B }{-z}  \\
\ \\
1
\end{bmatrix}
$$

*注意：* 因为摄像机空间的 $z$ 轴反向了，所以 $p$ 点到摄像机的 $z$ 轴距离为 $-z$ 。\
**尤其注意：** 矩阵的最后一行为 -1 ，就是这个原因。

$$
\begin{aligned}
&\text{当点位于视锥体的右锥面上}
\begin{cases}
\begin{aligned}
\text{根据三角形相似性， } &\dfrac{x}{-z} = \dfrac{r}{N} = aspect\ *\ \tan(\dfrac{FOV}{2})  \\
\ \\
\text{最终落在 }x = 1\text{ 的位置， } &\dfrac{S_x\ *\ x}{-z} = 1
\end{aligned}
\end{cases}
\Rightarrow\begin{aligned}
S_x &= \dfrac{-z}{x} = \dfrac{1}{aspect\ *\ \tan(\dfrac{FOV}{2})}  \\
\ \\
&= \dfrac{\cot(\dfrac{FOV}{2})}{aspect}
\end{aligned}  \\
\ \\
&\text{当点位于视锥体的右锥面上}
\begin{cases}
\begin{aligned}
\text{根据三角形相似性， } &\dfrac{y}{-z} = \tan(\dfrac{FOV}{2})  \\
\ \\
\text{最终落在 }y = 1\text{ 的位置， } &\dfrac{S_y\ *\ y}{-z} = 1
\end{aligned}
\end{cases}
\Rightarrow\begin{aligned}
S_y &= \dfrac{-z}{y} = \dfrac{1}{\tan(\dfrac{FOV}{2})}  \\
\ \\
&= \cot(\dfrac{FOV}{2})
\end{aligned}  \\
\ \\
&\begin{cases}
\begin{aligned}
\text{当点位于近平面上，即 }z = -N\text{ 的时候， } &\dfrac{A * (-N) + B }{-\ (-N)} = -1  \\
\ \\
\text{对于远平面上的点，即 }z = -F\text{ 的时候， } &\dfrac{A * (-F) + B }{-\ (-F)} = 1
\end{aligned}
\end{cases}
\Rightarrow
\begin{cases}
A = -\ \dfrac{F + N}{F - N}  \\
\ \\
B = -\ \dfrac{2\ *\ F\ *\ N}{F - N}
\end{cases}
\end{aligned}
$$

由此得到透视投影的变换矩阵

$$
M_{frustum} = \
\begin{bmatrix}
\dfrac{\cot(\dfrac{FOV}{2})}{aspect} & 0 & 0 & 0  \\
\ \\
0 & \cot(\dfrac{FOV}{2}) & 0 & 0  \\
\ \\
0 & 0 & -\ \dfrac{F + N}{F - N} & -\ \dfrac{2\ *\ F\ *\ N}{F - N}  \\
\ \\
0 & 0 & -1 & 0
\end{bmatrix}
$$

## 旋转和四元数

参考 [四元数教程](https://krasjet.github.io/quaternion/quaternion.pdf)

### 2D 旋转

让 2D 空间中任意一个矢量 $\vec v$ 旋转 $\theta$ 度，得到的新矢量为 $\vec{v\prime}$ 。

- 2D 旋转公式（矩阵形式）如下

$$
\vec{v\prime} \ = \
\begin{bmatrix}
\cos{(\theta)} & - \sin{(\theta)} \\
\sin{(\theta)} & \cos{(\theta)}
\end{bmatrix} \
\vec v
$$

- 2D 旋转公式（复数形式）如下

$$
\vec{v\prime} \ = \
(\cos{(\theta)} + i \sin{(\theta)}) \
\vec v
$$

- 2D 旋转公式（指数形式）如下

$$
\vec{v\prime} \ = \
e^{i \theta} \
\vec v
$$

### 3D 旋转

让 3D 空间中任意一个矢量 $\vec v$ 沿着单位矢量 $\hat u$ 旋转 $\theta$ 角度之后，得到的新矢量为 $\vec {v\prime}$ 。

$$
v′ = \cos{(\theta)} \vec v + (1 - \cos{(\theta)})(\hat u \cdot \vec v)\hat u + \sin{(\theta)}(\hat u \times \vec v)
$$

### 四元数的基本概念

#### 四元数的定义

所有的四元数 $q \in \mathbb{H}$（ $\mathbb{H}$ 代表四元数的发现者 William Rowan Hamilton）都可以写成下面这种形式\
*备注：* $\mathbb{R}$ 表示实数域

$$
q = a + bi + cj + dk, \quad (a, b, c, d \in \mathbb{R})
$$

其中

$$
i^2 = j^2 = k^2 = ijk = -1
$$

写成列矩阵形式为

$$
q \ = \
\begin{bmatrix}
a  \\ b \\ c \\ d
\end{bmatrix}
$$

写成标量和矢量的有序对形式为

$$
q \ = \ [s, \ \vec v], \quad \
(\vec v = \begin{bmatrix}x \\ y \\ z\end{bmatrix}, \ s, x, y, z \in \mathbb{R})
$$

#### 四元数乘法

对任意四元数 $q_1 = [s, \ \vec v], q_2 = [𝑡, \ \vec u]$ ， $q_1 q_2$ 的结果是

$$
q_1 q_2 = [ s t - \vec v \cdot u, \ s \vec u + t \vec v + \vec v \times \vec u ]
$$

四元数的乘法具有以下数学性质：

- 四元数的乘法不满足交换律，即 $p q \neq q p$
- 四元数的乘法满足结合律，

#### 纯四元数

如果一个四元数能写成这样的形式：

$$v = [0, \ \vec v]$$

那我们则称 $v$ 为一个纯四元数 (Pure Quaternion)，即仅有虚部的四元数。
因为纯四元数仅由虚部的 3D 矢量决定，我们可以将任意的 3D 矢量转换为纯四元数。

纯四元数有一个很重要的特性：如果有两个纯四元数 $v = [0, \ \vec v], \ u = [0, \ \vec u]$ ，那么

$$\begin{aligned}
v u &= [0 - \vec v \cdot \vec u, \ 0 + \vec v \times \vec u] \\
&= [- \vec v \cdot \vec u, \ \vec v \times \vec u]
\end{aligned}$$

#### 四元数的逆

对于四元数 $q$ ，它的逆四元数 $q^{-1}$ 满足

$$
q q^{-1} = q^{-1} q = 1, \quad (q \neq 0)
$$

也就是说

$$
(p q) q^{-1} = p (q q^{-1}) = p \\
\ \\
q^{-1} (q p) = (q^{-1} q) p = p
$$

通过 [共轭四元数](#共轭四元数) 可以很方便的求出

$$q^{-1} = \dfrac{q^{\ast}}{{\lVert q \rVert}^{2}}$$

对于模长为 1 的单位四元数 $q^{-1} = q^{\ast}$

#### 共轭四元数

一个四元数 $q = a + bi + cj + dk$ 的共轭为 $q^{\ast} = a - bi - cj - dk$ \
如果用标量矢量有序对的形式来定义的话， $q = [s, \ \vec v]$ 的共轭为 $q^{\ast} = [s, \ -\vec v]$ 。

将矢量形式代入乘法运算可以得到，共轭四元数的一个非常有用的性质就是

$$
q q^{\ast} = {\lVert q \rVert}^{2}
$$

### 四元数旋转

任意矢量 $\vec v$ 绕着以单位矢量定义的旋转轴 $\hat u$ 旋转 $\theta$ 度之后，得到的新矢量为 $\vec {v\prime}$ 。

$$\begin{aligned}
\text{设如下两个四元数： } &v = [0, \ \vec v], \quad q = [\cos{(\dfrac{\theta}{2})}, \ \sin{(\dfrac{\theta}{2})} \hat u] \\
\ \\
\text{那么： } &\vec {v\prime} = q v q^{\ast} = q v q^{-1}
\end{aligned}$$

同样的

$$\begin{aligned}
\text{对于单位四元数： } &q = [a, \ \vec b] \\
\ \\
\text{表示的旋转角：} &\theta = 2 \cos^{-1}(a) \\
\ \\
\text{表示的轴：} &\hat u = \dfrac{\vec b}{\sin{(\cos^{-1}(a))}} \\
\ \\
&(cos^{-1} \text{ 表示 } \arccos)
\end{aligned}$$

### 四元数旋转对应的矩阵

任意矢量 $\vec v$ 绕着以单位矢量定义的旋转轴 $\hat u = (x_u, y_u, z_u)$ 旋转 $\theta$ 度之后，得到的新矢量为 $\vec {v\prime}$ 。

$$\begin{aligned}
\text{设： } &a \ = \ \cos{(\dfrac{\theta}{2})}, \ b = \sin{(\dfrac{\theta}{2})}x_u, \
c = \sin{(\dfrac{\theta}{2})}y_u, \ d = \sin{(\dfrac{\theta}{2})}z_u \\
\ \\
\text{则： } &\vec {v\prime} \ = \
\begin{bmatrix}
1 - 2c^2 - 2d^2 & 2bc - 2ad & 2ac + 2bd \\
\ \\
2bc + 2ad & 1 - 2b^2 - 2d^2 & 2cd - 2ab \\
\ \\
2bd - 2ac & 2ab + 2cd & 1 - 2b^2 - 2c^2
\ \\
\end{bmatrix} \ \vec v
\end{aligned}$$

### 四元数旋转的复合

对于矢量 $\vec v$ 依次进行 $q_1, q_2, \cdots, q_n$ 变换之后等价与进行 $q = q_n \cdots q_2 q_1$ 变换。\
*注意：* 变换顺序和四元数乘法顺序正好是相反的，非常类似矩阵运算。

### 四元数的插值算法

#### N 维矢量的 Lerp 算法

对于 N 维矢量 $\vec V$ ，它的 Lerp 算法为

$$\begin{aligned}
\vec V_t = Lerp(\vec V_0, \vec V_1, t) &= \vec V_0 + t (V_1 - V_0) \\
\ \\
&=(1 - t) V_0 + t V_1
\end{aligned}$$

因为 Lerp 的结果不是单位矢量，所以四元数插值不能直接使用 Lerp 来运算。

#### N 维矢量的 Nlerp 算法

对于 N 为矢量 $\vec V$ ，它的 Nlerp 算法就是将 [Lerp](#n-维矢量的-lerp-算法) 的结果归一化。

$$\begin{aligned}
\vec V_t = Nlerp(\vec V_0, \vec V_1, t) &= \dfrac{Lerp(\vec V_0, \vec V_1, t)}{\lVert Lerp(\vec V_0, \vec V_1, t) \rVert} \\
\ \\
&= \dfrac{(1 - t) \vec V_0 + t \vec V_1}{\lVert (1 - t) \vec V_0 + t \vec V_1 \rVert}
\end{aligned}$$

如果两个旋转角度之间的差别非常小，那么就非常适合使用 Nlerp

#### N 维矢量的 Slerp 算法

对于 N 维矢量 $\vec V$ ，它的 Slerp 算法就是按照角度的线性插值来计算新的矢量。

$$\begin{aligned}
&\vec V_t = Slerp(\vec V_0, \vec V_1, t) = \dfrac{\sin{((1 - t)\theta)}V_0\ +\ \sin{(t\theta)}V_1}{\sin(\theta)} \\
\ \\
&\text{其中： } \theta = \cos^{-1}(V_0 \cdot V_1), \quad  cos^{-1} \text{ 表示 } \arccos
\end{aligned}$$

**注意：** 当两个矢量的夹角 $\theta$ 非常小时，作为分母的 $\sin{(\theta)}$ 因为精度问题，可能会变成 $0.0$ ，从而导致被 $0$ 除的错误。所以在使用 Slerp 的时候必须对这种情况做出判断，在夹角 $\theta$ 非常小的时候，应该使用 [Nlerp](#n-维矢量的-nlerp-算法) 算法。

### 四元数的双倍覆盖

对于单位四元数 $q=[\cos{(\dfrac{\theta}{2})}, \ \sin{(\dfrac{\theta}{2})} \hat u]$ ，它表示绕单位矢量 $\hat u$ 旋转 $\theta$ 角\
同样的，对于 $-q=[-\cos{(\dfrac{\theta}{2})}, \ -\sin{(\dfrac{\theta}{2})} \hat u]$ ，因为

$$\begin{aligned}
-q&=[-\cos{(\dfrac{\theta}{2})}, \ -\sin{(\dfrac{\theta}{2})} \hat u] \\
\ \\
&=[\cos{(\pi - \dfrac{\theta}{2})}, \ \sin{(\pi - \dfrac{\theta}{2})} (-\hat u)]
\end{aligned}$$

所以 $-q$ 表示的是沿 $\hat u$ 的反方向，旋转 $2\pi - \theta$ 角，和 $q$ 表示的旋转实际上是完全相同的。

所以 $q$ $-q$ 实际上表示的是同一个旋转。

我们可以说，**每个单位四元数对应一个 3D 旋转；而每个特定的 3D 旋转，有两个互为正负的单位四元数和它对应；这两个互为正负的单位四元数对应的旋转矩阵是完全相同的。**

所以，当对四元数 $q_0$ 和 $q_1$ 进行插值的时候，如果这两个四元数之间的夹角为钝角（即它们的 [点积](#矢量的点积) 小于 0），说明直接对他们插值会导致沿着球面的大弧度进行。\
所以为了正确的进行插值，应该对 $q_0$ 和 $-q_1$ 进行插值。公式表述如下

$$\begin{aligned}
\text{如果： } &q_0 \cdot q_1 < 0 \\
\text{那么直接使用 }&Slerp(q_0, q_1, t)\text{ ， 从效果上来说是错误的} \\
\text{应该改为使用 }&Slerp(q_0, -q_1, t)\text{ ， 才是正确的效果}
\end{aligned}$$

## 参考资料

[关于图形和 OpenGL 以及相关数学推导的个人网站](http://www.songho.ca/opengl/gl_overview.html)

[Nvidia Cg stdlib](https://developer.download.nvidia.com/cg/index_stdlib.html)

[Nvidia Cg Reference Manual](https://developer.download.nvidia.com/cg/Cg_3.1/Cg-3.1_April2012_ReferenceManual.pdf)

## 待学习的参考资料

[Unity URP PBR源码拆解](https://www.bilibili.com/read/cv16791975)

[ShaderLab语法汇总](https://www.bilibili.com/read/cv16654922)

<link rel="stylesheet" type="text/css" href="./auto_number_title.css"/>
