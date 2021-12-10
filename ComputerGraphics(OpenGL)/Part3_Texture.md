# 一、颜色插值

根据三角形三个顶点的颜色来求整个三角面的插值颜色

## 1. 普通的插值方法

方法一：根据**高度比**来插值

![](./images/linear_interpolate_triangle.png)
$$
\begin{align}
f_i &= d_i / h_i \\
Color &= f_i * Color_i + f_j * Color_j + f_k * Color_k
\end{align}
$$


方法二：根据所占**三角形面积**来插值
$$
\begin{align}
f_i &= {area(x, x_j, x_k) \over area(x_i, x_j, x_k)} \\
Color &= f_i * Color_i + f_j * Color_j + f_k * Color_k
\end{align}
$$



## 2. 在三角形的重心坐标系下做插值

在三个顶点组成的平面内，方法如下：

1. 将普通笛卡尔坐标系转化为重心坐标系
2. 根据重心坐标计算**不受透视投影影响**的三个插值系数
   这三个插值系数可以对当前三个顶点的任意属性进行插值



### 2.1 计算重心坐标系

![](./images/barycentric.jpg)

设 P 为 2D 空间内三角形 ABC 内任意一点，求三角形 ABC 以 AB，AC 为坐标轴的重心坐标 u, v
$$
\begin{align}
\vec {AP} &= u \vec {AB} + v \vec {AC} \\
A-P &= u(A-B) + v(A-C) \\
P &= (1-u-v)A + uB + vC \\\\

\vec {AP} &= u \vec {AB} + v \vec {AC} \\
u \vec {AB} + v \vec {AC} + \vec {PA} &= 0 \\
u (\vec {AB})_x + v (\vec {AC})_x + (\vec {PA})_x &= 0 \\
u (\vec {AB})_y + v (\vec {AC})_y + (\vec {PA})_y &= 0 \\
\begin{bmatrix}u & v & 1 \end{bmatrix} 
\begin{bmatrix}(\vec {AB})_x \\ (\vec {AC})_x \\ (\vec {PA})_x \end{bmatrix} &= 0 \\
\begin{bmatrix}u & v & 1 \end{bmatrix} 
\begin{bmatrix}(\vec {AB})_y \\ (\vec {AC})_y \\ (\vec {PA})_y \end{bmatrix} &= 0 \\
\begin{bmatrix}(\vec {AB})_x \\ (\vec {AC})_x \\ (\vec {PA})_x \end{bmatrix} \times 
\begin{bmatrix}(\vec {AB})_y \\ (\vec {AC})_y \\ (\vec {PA})_y \end{bmatrix} &= \begin{bmatrix}u \\ v \\ 1 \end{bmatrix} \\\\

u \vec {AB} + v \vec {AC} + \vec {PA} &= 0 \\
a \vec {AB} + b \vec {AC} + c \vec {PA} &= 0 &u = {a \over c}, v ={b \over c} \\
P &= (1-u-v)A + uB + vC \\
P &= (1-{a \over c}-{b \over c})A + {a \over c}B + {b \over c}C
\end{align}
$$
编码为

```c
// 将屏幕上的笛卡尔坐标系转换为 ABC 三角形内的重心坐标系
vec3 barycentric(vec2 A, vec2 B, vec2 C, vec2 P) {
    vec3 s[2];
    for (int i=2; i--; ) {
        s[i][0] = B[i]-A[i];
        s[i][1] = C[i]-A[i];
        s[i][2] = A[i]-P[i];
    }
    vec3 u = cross(s[0], s[1]);
    if (0 == std::abs(u.z)) return vec3(-1,1,1);
        
    return vec3(1.f-(u.x+u.y)/u.z, u.x/u.z, u.y/u.z);
}
```



### 2.2 根据重心坐标系计算插值系数

**注意：**

- 以下使用的深度都是<u>线性深度</u>，实际存储的是非线性深度，需要转换一下
- 对于非线性深度的插值，必须是<u>非透视矫正</u>的插值系数



已知

- 透视投影后 2D 屏幕空间的 三角形 ABC 的深度为 $Z_{P'} = \alpha' Z_{A'} + \beta' Z_{B'} +  \gamma' Z_{C'}$
- $\alpha' + \beta' + \gamma' = 1$

求：透视投影前 3D 裁剪空间的 三角形 ABC 的深度为 $Z_P = \alpha Z_A + \beta Z_B +  \gamma Z_C$
$$
\begin{align}
1 &= \alpha' + \beta' + \gamma' \\
{Z_P \over Z_P} &= {Z_A \over Z_A}\alpha' + {Z_B \over Z_B}\beta' + {Z_C \over Z_C}\gamma' \\
{Z_P \over Z_P} &=
\begin{bmatrix}Z_A & Z_B & Z_C\end{bmatrix}
\begin{bmatrix}{1 \over Z_A}\alpha' \\ {1 \over Z_B}\beta' \\ {1 \over Z_C}\gamma' \end{bmatrix} \\
Z_P &=
\begin{bmatrix}Z_A & Z_B & Z_C\end{bmatrix}
\begin{bmatrix}{1 \over Z_A}\alpha' \\ {1 \over Z_B}\beta' \\ {1 \over Z_C}\gamma' \end{bmatrix} Z_P \\
Z_P &=
\begin{bmatrix}Z_A & Z_B & Z_C\end{bmatrix}
\begin{bmatrix}{Z_P \over Z_A}\alpha' \\ {Z_P \over Z_B}\beta' \\ {Z_P \over Z_C}\gamma' \end{bmatrix}\\
Z_P &=
\begin{bmatrix}Z_A & Z_B & Z_C\end{bmatrix}
\begin{bmatrix}\alpha \\ \beta \\ \gamma \end{bmatrix}\\
\\
\alpha + \beta + \gamma &= 1\\
{Z_P \over Z_A}\alpha' + {Z_P \over Z_B}\beta' + {Z_P \over Z_C}\gamma' &= 1\\
Z_P &= {1 \over {{\alpha' \over Z_A} + {\beta' \over Z_B} + {\gamma' \over Z_C}}}
\\

\end{align}
$$
要通过这些插值其他属性的值 I，则
$$
\begin{align}
I_P &= \begin{bmatrix} I_A & I_B & I_C \end{bmatrix}
\begin{bmatrix} \alpha \\ \beta \\ \gamma \end{bmatrix}\\
&= \begin{bmatrix} I_A & I_B & I_C \end{bmatrix}
\begin{bmatrix} {Z_P \over Z_A}\alpha' \\ {Z_P \over Z_B}\beta' \\ {Z_P \over Z_C}\gamma'\end{bmatrix}\\
&= \begin{bmatrix} {Z_P \over Z_A}I_A & {Z_P \over Z_B}I_B & {Z_P \over Z_C}I_C \end{bmatrix}
\begin{bmatrix} \alpha' \\ \beta' \\ \gamma' \end{bmatrix}\\
&= ({\alpha' \over Z_A}I_A + {\beta' \over Z_B}I_B + {\gamma' \over Z_C}I_C)Z_P \\
&= ({\alpha' \over Z_A}I_A + {\beta' \over Z_B}I_B + {\gamma' \over Z_C}I_C) / {1 \over Z_P} \\
&= { {\alpha' \over Z_A}I_A + {\beta' \over Z_B}I_B + {\gamma' \over Z_C}I_C \over {{\alpha' \over Z_A} + {\beta' \over Z_B} + {\gamma' \over Z_C}} }
\end{align}
$$



# 二、纹理基础

纹理材质的反光性质

- 各项异性：固定视角和光源方向旋转表面时，反射会发生任何改变
- 各项同性：固定视角和光源方向旋转表面时，反射不会发生任何改变



纹理映射坐标

- 所有的纹理尺寸都会映射在 [0, 1] 的范围
  顶点着色器使用的是 uv 纹理坐标（坐标值为原始图片大小的值）
  片段着色器使用的是 st 纹理坐标（坐标值为归一化以后的值）
- OpenGL、Unity 纹理坐标原点在 左下角
- DirectX 纹理坐标原点在 左上角



纹理的尺寸

- 长宽大小应该是 2 的幂
  非 2 的幂的纹理会占用更多的内存空间和读取时间，有些平台会不支持非 2 的幂尺寸的纹理
- 纹理可以是非正方形的



## 1. 纹理环绕（坐标包装）

> 当**纹理坐标超出默认范围**时，每种纹理环绕方式都有不同的视觉效果输出

OpenGL 设置纹理不同坐标轴的环绕方式

```c
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_NEAREST); //纹理坐标 s/u/x 轴的包装格式
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_LINEAR);  //纹理坐标 t/v/y 轴的包装格式
```

![](images/texture_wrapping.png)

| 环绕方式           | 描述                                                         |
| ------------------ | ------------------------------------------------------------ |
| GL_REPEAT          | 对纹理的默认行为，重复纹理图像                               |
| GL_MIRRORED_REPEAT | 和 GL_REPEAT 一样，但每次重复图片是镜像放置的                |
| GL_CLAMP_TO_EDGE   | 纹理坐标会被约束在 0 ～ 1之间，超出的部分会重复纹理坐标的边缘，产生一种边缘被拉伸的效果 |
| GL_CLAMP_TO_BORDER | 超出的坐标处的纹理为用户指定的边缘颜色                       |



## 2. 纹理过滤（采样）

> 纹素，纹理单元：在**纹理坐标系**中一个像素占用的纹理数据
>
> 当三维空间里面的多边形，变成二维屏幕上的一组像素的时候，对每个像素需要到相应纹理图像中进行采样一个像素 Pixel 对应 N 个纹素 Texel 的映射过程就称为纹理过滤 

**纹理过滤的两种情况**

- 纹理被缩小 `GL_TEXTURE_MIN_FILTER`：**一个像素对应多个纹素**
  现象：走样问题，摩尔纹（远） + 锯齿（近）
  解决方法：超采样、Mipmap、各向异性过滤 Anisotropic
  例，一个 8 X 8 的纹理贴到远处正方形上，最后在屏幕上占了 2 X 2 个像素矩阵
- 纹理被放大 `GL_TEXTURE_MAG_FILTER`：**一个纹素对应多个像素**
  现象：模糊 / 锯齿
  解决方法：插值，Nearest、Bilinear
  例，一个 2 X 2 的纹理贴到近处正方形上，最后在屏幕上占了 8 X 8 个像素矩阵


OpenGL 中针对放大和缩小的情况的设置

```c
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST); //缩小
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);  //放大
```



### 2.1 邻近点采样 Nearest neighbor

优点：效率最高
缺点：效果最差

方法：选择最接近中心点纹理坐标的 **1 个纹理单元**采样

![](images/texture_nearest.png)



### 2.2 双线性过滤 Bilinear

优点：适于处理有一定精深的静态影像
缺点：不适用于绘制动态的物体，当三维物体很小时会产生深度赝样锯齿 (Depth Aliasing artifacts)

方法：选择最接近中心点纹理坐标的 2 X 2 纹理单元矩阵进行采样，取 **4 个纹理单元**采样的插值
另外还有种类似的方法 **Bicubic**，取附近 **16** 个纹理单元采样的插值

![](images/texture_linear.png)



### 2.3 多级渐远纹理过滤 Mipmap 

Migmap 用来对同一纹理生成多个不同尺寸的纹理，用 *Level of Detail* (**LOD**) 来规定纹理缩放的大小
LOD 0 为原始尺寸，从 LOD 1 开始，LOD n 的纹理宽高为 LOD n-1 的一半，直到纹理的大小缩放为 1 X 1 为止

距观察者的距离超过一定的阈值，OpenGL会使用不同的多级渐远纹理，即最适合物体的距离的那个。由于距离远，解析度不高也不会被用户注意到

优点：效果最好，适用于动态物体或景深很大的场景
缺点：效率低，会占用一定的空间，只能用于纹理被缩小的情况

![](./images/texture_mipmapping.png)

开启 Mipmap 下的纹理采样

![](./images/texture_mipmap.png)

例，三线性过滤 Trilinear 方法：

1. 取 Mipmap 纹理中距离与当前屏幕上尺寸相近的两个纹理

2. 将 1 中选取的纹理 选择最接近中心点纹理坐标的 2 X 2 纹理单元矩阵进行采样（线性过滤）

3. 将 2 中两次采样的结果进行加权平均（**8 个纹理单元**采样），得到最后的采样数据



**Mipmap Level 计算**
取 X 轴和 Y 轴的最大变化长度 L 作为查询的输入参数，通过 $d = log_2L$​​ 得到 Mipmap 的查询层级 Level d

**Mipmap 采样**
得到的层级 d 是个小数，分别向上和向下取整，得到两个不同 level 的 Mipmap 像素值，然后在根据 d 的小数部分做插值

<img src="./images/conpute_mipmaplevel.png" style="zoom:150%;" />



### 2.4 各向异性过滤 Anisotropic

> 之前提到的三种过滤方式，默认纹理在 x，y 轴方向上的缩放程度是一致的（纹理表面刚好正对着摄像机）
> 当纹理在 3D 场景中，纹理表面刚倾斜于虚拟屏幕平面时，出现一个轴的方向纹理放大，一个轴的方向纹理缩小的情况（**OpenGL 判定为纹理缩小**）需要使用各向异性过滤配合以上三种过滤方式来达到最佳的效果

优点：效果最好，使画面更加逼真
缺点：效率最低，由硬件实现

各向异性过滤包含会生成一张包含 Mipmap 的图 Ripmap，如下图（对角线的集合是 Mipmap）
各向异性过滤也只是覆盖了大部分情况，另一种方法 EWA filtering 多重椭圆形采样效果更好 

![](./images/ripmap.png)

方法：根据视角对梯形范围内的纹理采样

1. 确定 X、Y 方向的**采样比例（Ripmap 的采样范围 N x N）**
   ScaleX = 纹理的宽 / 屏幕上显示的纹理的宽
   ScaleY = 纹理的高 / 屏幕上显示的纹理的高
   异向程度 N = max(ScaleX, ScaleY) / min(ScaleX, ScaleY);

   例，64 X 64的纹理最后投影到屏幕上占了128 X 32 的像素矩阵
   ScaleX = 64.0 / 128.0 = 0.5;
   ScaleY = 64.0 / 32.0 = 2.0;
   异向程度 N = 2.0 / 0.5 = 4;

2. 根据采样比例分别在 X、Y 方向上采用 *三线性过滤* 或 *双线性过滤* 获得采样数据，**采样的范围由异向程度决定，不是原来的 2 X 2 像素矩阵**

   例，64 X 64 的纹理最后投影到屏幕上占了 128 X 32 的像素矩阵
   异向程度为 4，且在 缩放方面 X 轴 > Y 轴，所以 X 轴采样 2 个像素，Y 轴采样 2 * 异向程度 = 8 个像素
   采样范围为最接近中心点纹理坐标的 2 X 8 的像素矩阵

![](./images/texture_anisotropic.png)

OpenGL 中设置各向异性过滤

```c
glTexParameterf(GL_TEXTURE_2D, GL_TEXTURE_MAX_ANISOTROPY_EXT, 异向程度);
```

各向异性对比三线性

![](images/texture_anisotropic.jpg)



### 2.4 多级渐远纹理过滤 Mipmap

![](./images/texture_mipmap.jpg)

Migmap 用来对同一纹理生成多个不同尺寸的纹理，用 *Level of Detail* (**LOD**) 来规定纹理缩放的大小
LOD 0 为原始尺寸，从 LOD 1 开始，LOD n 的纹理宽高为 LOD n-1 的一半，直到纹理的大小缩放为 1 X 1 为止

距观察者的距离超过一定的阈值，OpenGL会使用不同的多级渐远纹理，即最适合物体的距离的那个。由于距离远，解析度不高也不会被用户注意到

优点：效果最好，适用于动态物体或景深很大的场景
缺点：效率低，会占用一定的空间，只能用于纹理被缩小的情况

![](./images/texture_mipmapping.png)

开启 Mipmap 下的纹理采样

![](./images/texture_mipmap.png)

例，三线性过滤 Trilinear 方法：

1. 取 Mipmap 纹理中距离与当前屏幕上尺寸相近的两个纹理

2. 将 1 中选取的纹理 选择最接近中心点纹理坐标的 2 X 2 纹理单元矩阵进行采样（线性过滤）

3. 将 2 中两次采样的结果进行加权平均（**8 个纹理单元**采样），得到最后的采样数据



MipMap Level 计算

<img src="./images/conpute_mipmaplevel.png" style="zoom:150%;" />



### 2.5 圆内均匀随机采样

![](./images/sample.png)

1. 一般圆内随机采样的方式如图 2，其中 a，b 为均匀的随机数
   这种变换的局部性保持很差，如果两点在笛卡尔坐标系下连续，那么投影到圆后的两个点同样是连续的。但是，反过来就不一定成立了。

$$
\begin{align}
r &= \sqrt a \\
\theta &= 2 \pi \space b \\\\
(u, v) &= (r\cos \theta, r\sin \theta) \\
\end{align}
$$

2. 一种较好的改进方式如图 3，其中 a，b 为均匀的随机数
   具体论证见论文 [ A Low Distortion Map Between Disk and Square | Semantic Scholar](https://www.semanticscholar.org/paper/A-Low-Distortion-Map-Between-Disk-and-Square-Shirley-Chiu/43226a3916a85025acbb3a58c17f6dc0756b35ac)

$$
\begin{align}
r &= \sqrt a \\
\theta &= {\pi \space b  \over 4 \space a}
\end{align}
$$

3. [泊松圆盘采样 Poisson Disk](https://bost.ocks.org/mike/algorithms/#sampling)
   由于采样计算方法复杂，大家多用查表法来提前存储采样结果，然后再用旋转提前存储采样点的方式得到伪随机采样点的坐标![](./images/sample_poisson_disk.png)

   ```c++
   glm::vec2 sample(int numCandidates, const std::vector<glm::vec2>& samples) 
   {
     float bestDistance = 0;
     glm::vec2 bestCandidate;
     for (int i = 0; i < numCandidates; ++i) {
       glm::vec2 c(Math.random() * width, Math.random() * height);
       float d = glm::distance(findClosest(samples, c), c);
       if (d > bestDistance) {
         bestDistance = d;
         bestCandidate = c;
       }
     }
   
     return bestCandidate;
   }
   ```



## 3. 反走样 Anti-Aliasing

> 香农定理告诉我们，即便我们有无限的频率，无论对原信号采样多少次，总会在重建信号时有一些误差

通过增加采样质量来反走样，是适用于各个渲染场景的唯一通用方案


### 3.1 SSAA

超采样抗锯齿 Super Sample Anti-aliasing, SSAA

- 步骤：渲染一张比显示的纹理更高分辨率的帧缓冲，分辨率下采样到正常的分辨率
- 缺点：性能开销很大



### 3.2 MSAA

多重采样抗锯齿 Multisample Anti-aliasing, MSAA

- 步骤：将单一的采样点变为多个采样点（采样点的数量可以是任意的）
  不再使用像素中心的单一采样点，而是以特定图案排列的 4 个子采样点(Subsample)
  (4 个以上采样点的效果差别不大) 
  由顶点插值得到的像素颜色，会存储在**被图形遮盖住**的**每个**子采样点中，最终的像素颜色是子采样点的平均值
  如果不想以平均计算子采样颜色的方式，OpenGL 允许我们在 FS 阶段获取到每个子采样点的颜色并计算最终采样结果
- 缺点：颜色缓冲的大小会随着子采样点的增加而增加

![](./images/trick_anti_aliasing_sample.png)

### 3.3 FXAA





### 3.4 TAA







#  三、遮罩贴图

- 保护纹理的某些区域，使它们免于修改
- 主要用与控制光照，使同一个纹理的模型不同的角度拥有了不同的高光强度
- 一般为单通道纹理，不过有时候一张 RGBA 四通道的遮罩纹理可以控制 四种 表面属性的强度
- 使用方式为：物体的颜色 = 当前纹理坐标对应的遮罩纹理强度 * 光照计算后的颜色



## 1. Opacity 透明贴图

贴图的不透明度：黑色是透明的部分，白色为不透明的部分，灰色为半透明的部分



## 2. Ambient Occlusion 环境遮挡贴图

环境光遮蔽贴图属于**预计算的贴图类型**（预先计算好纹理效果，降低实时计算成本）
模拟物体之间所产生的阴影，在不打光的时候增加体积感
完全不考虑光线，单纯基于物体与其他物体越接近的区域，受到反射光线的照明越弱这一现象来模拟现实照明（的一部分）效果

白色表示应接受完全间接光照的区域，以黑色表示没有间接光照





# 四、微观几何形态存储



## 1. 凹凸贴图 Bump Map

又称高度贴图：使用一张高度纹理来模拟表面上下高度的位移，存储相对于顶点位置的偏移量（白色区域是高区域，黑色区域是低区域）

- 优点：非常直观，可以从高度纹理中明确的知道一个模型表面的凹凸情况
- 缺点：**不改变几何信息**，计算较复杂，不能直接得到表面法线，需要通过像素的灰度值计算得到
  如图，先通过凹凸贴图和顶点位置计算高度值，根据高度值计算表面切线，根据表面切线的垂线得到法线
  ![](./images/texture_bump_map.png)



## 2. 位移贴图 Displacement Map

![](./images/texture_displacement.png)

**和凹凸贴图一样**存储相对于顶点位置的偏移量值

- 优点：可直接使用凹凸贴图作为位移贴图，改变几何信息
- 缺点：相比上更逼真，要求模型足够细致，运算量更高
  DirectX 有 Dynamic 的插值法，对模型做插值，使得初始不用过于细致



## 3. 法线贴图 Normal map

法线贴图

- 直接存储表面法线
- 根据法线所在的坐标空间类型可分为
  模型空间的法线纹理 (object-space normal map)：将修改后的**模型**空间表面的法线存储在一张纹理中
  切线空间的法线纹理 (tangent-space normal map)：将修改后的**纹理**切线空间表面的法线存储在一张纹理中
  **一般使用切线空间的法线纹理**



法线的模型变换矩阵

- 在顶点坐标的模型变换中，当我们使用一个不等比缩放时，法线不会再垂直于对应的表面
  ![](./images/normal_transformation.png)

- 法线需要一个基于顶点坐标的模型变换的专门的 模型矩阵
  如果模型变换 $M_t$ 不是正交变换，则法线变换矩阵为：$M_{n} = (M_t^T)^{-1}$
  如果模型变换 $M_t$ 是正交变换，则法线变换矩阵为：$M_{n} = M_t$
  正交变换：旋转变换，[公式的推导过程](../LinearAlgebra/Part1_Matrix.md)
  由于位移对于法线方向没有影响，而逆变换计算量较大，因此一般采用没有位移的 3 X 3 矩阵来计算法线变换



### 3.1 使用流程

**I. 顶点信息补充**

   1. 由 模型变换 得到 法线的模型变换矩阵（逆矩阵耗时大，尽量放在 CPU 上算一次或者放在顶点着色器）
   2. 根据顶点位置和纹理坐标信息，计算**模型空间下的** 切线 和 副切线
   3. 每三个顶点构成一个平面，他们共享一组 切线 和 副切线



**II. 顶点着色器**

1. 将顶点数据中的 切线、副切线、法线坐标系位置经过 法线的模型变换矩阵 转换为
   **世界空间下的** 切线空间坐标，Gram-Schmidt 正交化后构建 切线空间矩阵
2. 计算世界空间下的光源在 切线空间 的坐标



**III. 片元着色器**

   1. 根据法线纹理对应的普通纹理的纹理坐标，从法线纹理读取切线空间下的法线数据（像素值）
   2. 将范围是 [0, 1] 的像素值，转换为范围是 [-1, 1] 的表面法线值：$normal = pixel*2.0 - 1.0$
   3. 将在**切线空间**下的光源和物体片元的坐标与法线计算得到片元颜色



### 3.2 切线空间

切线空间的坐标系，原点：模型的顶点

- Z 轴：N（Normal）法线方向（和 Z 轴的正方向始终保持一致）
- X 轴：T（Tagent）切线方向，和纹理坐标的 X 轴（U）一致
- Y 轴：B（Bitangent）副切线方向 ，和纹理坐标的 Y 轴（V）一致

![](./images/normal_mapping_tbn.png)



计算额外的顶点信息：纹理法线 **切线空间** 到 **模型空间** 的矩阵

- 已知切线空间法线纹理的切线 T 和 副切线 B 分别对应与法线纹理对应普通纹理的 U 和 V 坐标轴
  （此时 T、B 在模型空间下）且点 $P_1$、$P_2$、$P_3$ 与纹理坐标的对应关系如下图，
  求切线方向 T 和副切线方向 B

  ![](./images/normal_mapping_surface_edges.png)

- 则：
  $$
  \begin{align}
  E_1 &= \Delta U_1 T + \Delta V_1 B\\
  E_2 &= \Delta U_2 T + \Delta V_2 B\\
  \begin{bmatrix} E_1\\ E_2 \end{bmatrix}
  &= 
  \begin{bmatrix}
  \Delta U_1 & \Delta V_1\\
  \Delta U_2 & \Delta V_2
  \end{bmatrix}
  \begin{bmatrix} T\\ B \end{bmatrix} \\
  \begin{bmatrix}
  \Delta U_1 & \Delta V_1\\
  \Delta U_2 & \Delta V_2
  \end{bmatrix}^{-1}
  \begin{bmatrix} E_1\\ E_2 \end{bmatrix}
  &= 
  \begin{bmatrix} T\\ B \end{bmatrix} \\
  {1 \over \Delta U_1 \Delta V_2 - \Delta U_2 \Delta V_1}
  \begin{bmatrix}
  \Delta V_2 & -\Delta V_1\\
  -\Delta U_2 & \Delta U_1
  \end{bmatrix}
  \begin{bmatrix} E_1\\ E_2 \end{bmatrix}
  &= 
  \begin{bmatrix} T\\ B \end{bmatrix} \\
  {1 \over \Delta U_1 \Delta V_2 - \Delta U_2 \Delta V_1}
  \begin{bmatrix}
  \Delta V_2 E_1 -\Delta V_1 E_2\\
  -\Delta U_2 E_1 + \Delta U_1 E_2
  \end{bmatrix}
  &= 
  \begin{bmatrix} T\\ B \end{bmatrix}
  \end{align}
  $$



**Gram-Schmidt 正交化**

当在更大的网格上计算切线向量的时候，它们往往有很大数量的共享顶点，当法向贴图应用到这些表面时将切线向量平均化（一个三角面平均三个顶点的切向量）通常能获得更好更平滑的结果。但是这样做有个问题，就是TBN向量可能会不能互相垂直，这意味着 TBN 矩阵不再是正交矩阵了

这时需要在**顶点着色器**做正交化操作，让 TBN 回归到正交矩阵
$$
\begin{align}
N &= normalize(N) \\
T &= normalize(U - dot(U, N) * N) \\
B &= normalize(cross(N, T))
\end{align}
$$



### 3.3 不同坐标空间的比较

**模型空间**法线纹理的优点：

- 实现简单，更加直观
- 在纹理坐标的缝合处和尖锐的边角部分，可见的突变（缝隙）较少，边界过渡平滑
  模型空间的法线纹理存储的是同一坐标系下的法线信息，在边界可将法线通过插值，来实现平滑过渡
  切线空间的法线依靠纹理坐标的方向得到的结果，会在边缘处或尖锐的地方出现缝合现象



**切线空间**法线纹理的优点：

- 自由度高，可做 UV 纹理动画
  可映射到不同的网格上，而模型空间法线纹理只能用于创建他的网格
- 可以复用法线纹理
  一个砖块的 6 个面可以共用一张切线空间法线纹理
- 对于纹理使用的额外数据是 可压缩的
  可只存储额外的 切线 和 副切线 2 个方向，而模型空间的法线必须存储 3 个方向的值



## 4. Curvature 曲率贴图

存储凹凸信息：黑色的值代表了凹区域，白色的值代表了凸区域，灰度值代表中性/平地



## 5. Thickness 厚度贴图

辅助制作表面散射 SSS：黑色代表薄的地方、白色代表厚的地方



# 五、环境光贴图

环境光贴图 Environment Light Map，存储所有外部方向的环境光照信息（环境光来自无限远处，强度一致，只记录方向）

在游戏引擎里做**场景**地图的时候会用到
用于静态模型上的间接光照：将场景的光照结果烘培到模型贴图上，从而实现模拟现实光照效果，可以节省硬件资源

## 1. 球形环境贴图 Spherical Environment Map

存储的一个**镜子球反射**的环境光色彩
球形贴图扭曲拉伸问题严重，贴图存储的环境光照信息不均匀



## 2. 立方体贴图 Cube Map

贴图扭曲拉伸问题相较于球形环境贴图较轻，但计算量较大
立方体贴图 GL_TEXTURE_CUBE_MAP 

```c
// shader
uniform samplerCube skybox;

// CPU code
glBindTexture(GL_TEXTURE_CUBE_MAP, _texID);
glTexImage2D(GL_TEXTURE_CUBE_MAP_POSITIVE_X + i, ...);
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
```

- 包含了 6 个 2D 纹理的纹理，每个 2D 纹理都组成了立方体的一个面，它通过一个方向向量来进行采样
  **方向向量的大小并不重要，只要提供了方向**

- 纹理坐标：一般为世界坐标系下的顶点坐标（范围 -1，1）
  处于世界坐标系下，是一个由立方体中心出发指向立方体面的三维向量
  贴图的顺序一般为：<u>右、左、上、下、前、后</u>

  ![](./images/texture_cube_map.png)

### 2.1 天空盒 Skybox

天空盒：包含了整个场景的（大）立方体，它包含周围环境的 6 个图像，让玩家以为他处在一个比实际大得多的环境当中

- 天空盒会跟随相机移动，从而让人认为天空盒的图像在无法到达的远方
- 使用提前深度测试将天空盒最后渲染以节省带宽
- 纹理环绕方式：超出采样部分取边界
- 通过将输出位置的 z 分量等于它的 w 分量，让 z 分量永远等于 1.0，使 z 在透视除法时，深度始终是最大的 1
  `gl_Position = pos.xyww;`

```glsl
// VS
#version 330 core
layout (location = 0) in vec3 aPos;

out vec3 TexCoords;

uniform mat4 projection;
uniform mat4 view;
uniform mat4 model;

void main() {
    TexCoords = aPos;
    vec4 pos = projection * view * model * vec4(aPos, 1.0);
	  // 用 w 替换 z，让天空盒的深度值在进行深度除法后始终保持 1.0 的最大值，确保天空盒在最后绘制
    gl_Position = pos.xyww;
}

// FS
#version 330 core
out vec4 FragColor;

in vec3 TexCoords;

uniform samplerCube skybox;

void main() {
    FragColor = texture(skybox, TexCoords);
}
```



### 2.2 环境映射 Environment Mapping

环境映射：通过使用环境的立方体贴图（不仅仅是天空盒）我们可以给物体反射和折射的属性

动态环境贴图：通过帧缓冲，为物体的 6 个角度创建出场景纹理，并在每个渲染迭代中将它们储存到一个立方体贴图中。之后可以使用这个（动态生成的）立方体贴图来创建出更真实的，包含其它物体的，反射和折射表面了

**反射**：通过物体表面单位法线，观察方向来计算反射方向作为立方体贴图的纹理坐标

![](./images/texture_reflection.png)



**折射**：通过物体表面单位法线，观察方向，以及两个材质之间的折射率（Refractive Index）来求出折射方向，其中 OpenGL 输入的折射率 = $出发材质（空气）的折射率 \over 进入材质（水）的折射率$

折射法则通过 [斯涅尔定律 Snell's Law](https://en.wikipedia.org/wiki/Snell%27s_law) 来描述，其中 $\eta$ 为材质的折射率

$$
\eta_{入射角} \sin \theta_{入射角} = \eta_{出射角}  \sin \theta_{折射角}
$$

![](./images/texture_refraction.png)

```glsl
// VS
#version 330 core
layout (location = 0) in vec3 position;
layout (location = 1) in vec3 normal;

out vec3 Normal;
out vec3 Position;

uniform mat4 projection;
uniform mat4 view;
uniform mat4 model;

void main() {
    Normal = mat3(transpose(inverse(model))) * normal;
    Position = vec3(model * vec4(position, 1.0));
    gl_Position = projection * view * model * vec4(position, 1.0);
}

// FS
#version 330 core
out vec4 FragColor;

in vec3 Normal;
in vec3 Position;

uniform vec3 cameraPos;
uniform samplerCube skybox;

void main() {
    vec3 I = normalize(Position - cameraPos);
    vec3 R = reflect(I, normalize(Normal)); // 反射

    // 折射
    float ratio = 1.00 / 1.52;
    R = refract(I, normalize(Normal), ratio);
    FragColor = vec4(texture(skybox, R).rgb, 1.0);
}

```





# 六、高级纹理

## 1. 渲染目标纹理 RTT

渲染目标纹理（Render Target Texture）把整个三维场景渲染到中间缓冲中，而不是传统的帧缓冲或者后备缓冲（back buffer），与之相关的是多重渲染目标（Multiple Render Target，MRT）

应用：

- 场景中的镜子
- 场景中的透明玻璃



## 2. 程序纹理

程序纹理：由计算机生成的纹理，可以使用各种颜色以外参数来控制纹理的外观

### 2.1 Perlin 噪声纹理

常用于模拟水波纹，火和地形，生成二维 Perlin 噪声纹理的过程如下：

1. 晶格划分
   将二维空间划分为多个大小相等的晶格（矩形）例：1024px * 1024px 的噪声图，可以选择 64px 为晶格尺寸
   
   ```glsl
   p0 = floor(pos / size) * size;
   p1 = p0 + float2(1, 0) * size;
   p2 = p0 + float2(0, 1) * size;
   p3 = p0 + float2(1, 1) * size;
   
   posInGrid = (pos - p0) / size;
   ```
   
   ![](./images/texture_noise_lattice.png)
   
2. 伪随机梯度生成
   根据晶格的位置 P 与随机种子，对晶格的每个顶点生成一个伪随机梯度，表示为一个二维向量
   经过**随机函数 gold_noise** 生成的随机的 x, y 后再归一化，最后用 grad 表

   ```glsl
   #define PHI (1.61803398874989484820459 * 00000.1)
   #define PI (3.14159265358979323846264 * 00000.1)
   #define SQ2 (1.41421356237309504880169 * 10000.0)
   
   float gold_noise(float2 pos, float seed) {
     return frac(tan(distance(pos * (PHI + seed), float2(PHI, PI))) * SQ2) * 2 - 1;
   }
   ```

![](./images/texture_noise_lattice1.png)

3. 晶格内插值
   计算当前点 P 相对于晶格四个顶点的偏移量 delta

   ![](./images/texture_noise_lattice2.png)

   对 delta 和 伪随机梯度得到的 grad 进行点积得到 v，最后将四个顶点的 v 值插值为一个数值
   得到 Perlin 噪声的**最终值（范围 -1, 1）**

   插值系数的计算一般为：$k = 6t^5 - 15t^4 + 10t^3$ 或 $k = 3t^2 - 2t^3$

   ```glsl
   float smoothLerp(float a, float b, float t) {
       float k = pow(t, 5) * 6 - pow(t, 4) * 15 + pow(t, 3) * 10;
       return (1 - k) * a + k * b
   }
   
   v0 = dot(delta0, grad0);
   v1 = dot(delta1, grad1);
   v2 = dot(delta2, grad2);
   v3 = dot(delta3, grad3);
   
   // Lerp with x
   a = smoothLerp(v0, v1, posInGrid.x);
   b = smoothLerp(v2, v3, posInGrid.x);
   
   // Lerp with y
   return smoothLerp(a, b, posInGrid.y);
   ```

4. 分型噪声图

   仅通过晶格化随机梯度生成的二维噪声图难以模拟自然界中的噪声现象
   即便是缩小晶格尺寸，也只能徒增噪声图的 "颗粒感"
   需要通过：将多种不同晶格尺寸的噪声图**叠加**得到自相似的分形噪声图

   ```glsl
   // 噪声图的叠加过程也可以描述成一个 分形布朗运动（FBM）函数
   inline float fbm(float2 pos) {
       float value = 0;
       float amplitude = 0.5;
   
       for(int i = 0; i < _Iteration; i++) {
           // 由于 noise_function 返回值的范围在 -1 ~ 1，取绝对值后，可用于地形的生成
           // value += amplitude * abs(noise_function(pos));
           value += amplitude * noise_function(pos);
           pos *= 2;
           amplitude *= .5;
       }
     
       return value;
   }
   ```



### 2.2 Worley 噪声纹理

常用于模拟多孔噪声，如：石头、水、纸张



## 3. 虚拟纹理 Virtual Texture











# 八、纹理压缩







# Reference

1. [Render To Texture](http://www.paulsprojects.net/opengl/rtotex/rtotex.html)
2. [Implementing an anisotropic texture filter](https://www.sciencedirect.com/science/article/abs/pii/S0097849399001594)
3. [Lesson 2: Triangle rasterization and back face culling · ssloy/tinyrenderer Wiki (github.com)](https://github.com/ssloy/tinyrenderer/wiki/Lesson-2:-Triangle-rasterization-and-back-face-culling)
4. [(PDF) Accelerated Half-Space Triangle Rasterization (researchgate.net)](https://www.researchgate.net/publication/286441992_Accelerated_Half-Space_Triangle_Rasterization)
6. [learnopengl-法线贴图](https://learnopengl-cn.github.io/05%20Advanced%20Lighting/04%20Normal%20Mapping/)
7. [learnopengl-立方体贴图](https://learnopengl-cn.github.io/04 Advanced OpenGL/06 Cubemaps/#_7)
8. [Understanding Perlin Noise](https://flafla2.github.io/2014/08/09/perlinnoise.html)
9. [基于 ComputeShader 生成 Perlin Noise 噪声图](https://zhuanlan.zhihu.com/p/88518193)
10. [Unity_Shaders_Book : https://github.com/candycat1992/Unity_Shaders_Book](https://link.zhihu.com/?target=https%3A//github.com/candycat1992/Unity_Shaders_Book)
11. [Unity Manual: https://docs.unity3d.com/Manual/TextureTypes.html](https://link.zhihu.com/?target=https%3A//docs.unity3d.com/Manual/TextureTypes.html)
14. [Learning DirectX 12 – Lesson 4 – Textures](https://www.3dgep.com/learning-directx-12-4/)
15. [Unity GPU优化(Occlusion Culling 遮挡剔除，LOD 多细节层次，GI 全局光照)](https://gameinstitute.qq.com/community/detail/120912)
16. [《我所理解的 Cocos2d-x》秦春林](https://book.douban.com/subject/26214576/)
17. [《Unity Shader 入门精要》冯乐乐](https://book.douban.com/subject/26821639/)
18. [深入探索透视纹理映射（下）](https://blog.csdn.net/popy007/article/details/5570803)
17. [FXAA Whitepaper](http://developer.download.nvidia.com/assets/gamedev/files/sdk/11/FXAA_WhitePaper.pdf)

