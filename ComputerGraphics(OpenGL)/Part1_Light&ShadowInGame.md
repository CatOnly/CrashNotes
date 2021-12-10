# 一、光

## 1. 颜色计算

常用的计算光照颜色的方法：

- 物体反射的颜色（我们感知到的颜色）：光源的颜色 * 物体的颜色
- 多光源的情况下，一般都是将各个光源的颜色累加起来，最后得出最终的颜色
- 同一个光源的光衰减系数是一样的



屏幕上显示的物体颜色可以通过以下公式得出，其中：

- $K_\gamma$：显示屏幕的 gamma 矫正
- $K_{Cam}$：相机的配置（曝光、白平衡）
- $K_i$：当前光源的衰减
- $I_i$：当前像素接受光源能量的比例
- $\Phi_i$：当前光源放射的总能量
- $L_i$：当前光源的颜色

$$
Color_{屏幕} = K_\gamma K_{Cam} \sum_{i=0}^{L_n} K_i I_i \Phi_i L_i
$$



## 2. 光的衰减（体积）

**衰减** Attenuation

随着光线传播距离的增长逐渐削减光的强度，光的衰减的模拟公式（其中 $K_c$、$K_l$、$K_q$ 的取值都是经验值）

- $K_c$ 通常保持为 1.0，它的主要作用是保证分母永远不会比1小，否则的话在某些距离上它反而会增加强度，这肯定不是我们想要的效果
- $K_l$ 与距离值相乘，以线性的方式减少强度
- $K_q$ 与距离的平方相乘，让光源以二次递减的方式减少强度。二次项在距离比较小的时候影响会比一次项小很多，但当距离值比较大的时候它就会比一次项更大了

**光体积** Light Volume

光源能够达到片段的范围（通过光的衰减计算出<u>光的半径</u>），一般通过渲染光的衰减半径的球体来确定光源的影响范围
根据衰减公式可知，衰减值只能无限接近于 0，因此需要通过选定一个衰减的最小值来限定光的范围
一般 $L_{att} = {x \over 256}$，除以 256 是因为默认的 8-bit 帧缓冲可以每个分
$$
\begin{align}
L_{att} &= {1.0 \over K_c + K_l * d + K_q * d^2}\\
K_q * d^2 + K_l * d + K_c &= {1.0 \over L_{att}}\\
K_q * d^2 + K_l * d + K_c - {1.0 \over L_{att}} &= 0\\
d &= {-K_l + \sqrt{K_l^2 - 4*K_q*(K_c - {1.0 \over L_{att}})} \over 2 * K_q}
\end{align}
$$

实际的光的衰减的计算

1. 通过一张 256 * 1 的纹理作为查找表
   通过的**点到光源距离的平方** 来（为了避免开方操作）查找衰减值（Unity 内的光衰减纹理）
2. 使用简化后的数学公式计算衰减（Unity 内使用的计算公式）

$$
L_{att} = {1.0 \over 光源到着色物体的距离 d}
$$



## 3. 光源类型

光源类型，即投光物(Light Caster)：将光**投射**(Cast)到物体的光源
在渲染方程中常常称一个具体的光源类型为 **精确光源 punctual lights**  



### 3.1 天光 Sky light

天光 (单位：$cd/m^2$)：环境光，模拟场景中带有太阳或阴天的光照环境，不仅可以用于户外环境，也可以给室内环境带来环境光效果

1. 使用固定的场景贴图：多用天空球网格贴图
2. 实时从场景中捕捉生成 Cubemap（立方体图）来生成环境光



### 3.2 自发光 Emissive light

自发光表面 (单位：$cd/m^2$)：直接由光源发射的光照，自发光一般为材质的颜色

- 实时渲染中自发光不会作为光源来照亮其他物体（不提供间接光照）
- 天光可以看做自发光的一种



### 3.3 平行光 Directional light

![](./images/irradiance.png)

平行光 (单位：$lux$)，又称定向光：光源处于无限远处，所有光线有相同的方向

- 不考虑光源位置，只考虑光的方向
- 表示用方向：平行光从光源发出的方向
- 计算用方向：平行光从片段发出到光源的方向（与平行光的表示相反）



**光照度**物理光照计算公式（其中，$\Phi _e$ 表示光通量，为已知条件）
$$
E_e = {\Phi_e \over A\cos \theta}
$$
实际简化后的计算方式：
使用 Phong 光照模型里的兰伯特漫反射模型<u>乘以固定的系数</u>来计算

```glsl
vec3 CalcDirLight(DirLight light, vec3 normal, vec3 viewDir)
{
    vec3 lightDir = normalize(-light.direction);
    
    // diffuse shading
    float diff = max(dot(normal, lightDir), 0.0);
    
    // specular shading
    vec3 reflectDir = reflect(-lightDir, normal);
    float spec = pow(max(dot(viewDir, reflectDir), 0.0), material.shininess);
    
    // combine results
    vec3 ambient = light.ambient * vec3(texture(material.diffuse, TexCoords));
    vec3 diffuse = light.diffuse * diff * vec3(texture(material.diffuse, TexCoords));
    vec3 specular = light.specular * spec * vec3(texture(material.specular, TexCoords));
    
    return (ambient + diffuse + specular);
}

```



### 3.4 点光源 Point light

![](./images/light_point.png)

点光源 (单位：$cd$)：光源处于世界中某一个位置的光源，它会朝着所有方向发光，但光线会随着距离逐渐衰减

- 计算用方向：平行光从片段发出到点光源的方向
- 衰减系数：点光源的最终结果需要乘以一个衰减系数



点光源环境下，[球坐标系](https://baike.baidu.com/item/球坐标系) $(r,\theta,\phi)$：
实际输出的光通量 $\Phi _e$ 与实际接收的光强度 $I_e$ 的转换比例为
$$
\begin{align}
\Phi_e &=
\int _0^{2\pi} \int_0^{\pi} sin\theta \space d\theta d\phi \space I_e
= 4\pi \space I_e \\\\
1 \space lm &=4\pi \space cd \\
&\approx  12.6 \space cd
\end{align}
$$

```glsl
vec3 CalcPointLight(PointLight light, vec3 normal, vec3 fragPos, vec3 viewDir)
{
    vec3 lightDir = normalize(light.position - fragPos);
    
    // diffuse shading
    float diff = max(dot(normal, lightDir), 0.0);
    
    // specular shading, func reflect need inverse direction of input light
    vec3 reflectDir = reflect(-lightDir, normal);
    float spec = pow(max(dot(viewDir, reflectDir), 0.0), material.shininess);
    
    // combine results
    vec3 ambient = light.ambient * vec3(texture(material.diffuse, TexCoords));
    vec3 diffuse = light.diffuse * diff * vec3(texture(material.diffuse, TexCoords));
    vec3 specular = light.specular * spec * vec3(texture(material.specular, TexCoords));
    
    // attenuation
    float distance = length(light.position - fragPos);
    float attenuation = 1.0 / (light.constant + light.linear * distance + light.quadratic * (distance * distance));   

    return (ambient + diffuse + specular) * attenuation;
}
```



### 3.5 聚光灯 Spot light

![](./images/light_spotlight.png)

聚光灯 (单位：$cd$)：只朝一个特定方向而不是所有方向照射光线，只有在聚光方向的特定半径内的物体才会被照亮，其它的物体都会保持黑暗

实际简化后的计算方式：

- LightDir：聚光照射到片元的方向

- SpotDir：聚光的方向

- $\phi$ 切光角：聚光的照在物体上光圈的半径大小

- $\theta$ LightDir 和 SpotDir 之间的夹角



**聚光的边缘软化**

- 聚光边缘强度变化：需要一个内切光角和一个外切光角，通过从内到外切光角的过渡来表示聚光强度的变化

- 强度计算公式，其中
  $I$ 为聚光强度，范围是 [0, 1] 
  $\theta$ 为 LightDir 和 SpotDir 之间的夹角
  $\phi$ 为外切光角，$\gamma$ 为内切光角（$\phi$、$\gamma$ 一般作为聚光的属性，都是常数）

  $I = {\theta - \phi \over cos\gamma - cos\phi}$



聚光灯环境下，[球坐标系](https://baike.baidu.com/item/球坐标系) $(r,\theta,\phi)$：
实际输出的光通量 $\Phi _e$ 与实际接收的光强度 $I_e$ 的转换比例为
$$
\begin{align}
\Phi_e &=
\int _0^{2\pi} \int_0^x sin\theta \space d\theta d\phi \space I_e
= 2\pi(1-cosx) \space I_e , \space x\in[0,{\pi \over 2}]\\\\
1 \space lm &=2\pi(1-cosx) \space cd , \space x\in[0\degree,90\degree]\\
&\approx  6.3(1-cosx) \space cd
\end{align}
$$

```glsl
vec3 CalcSpotLight(SpotLight light, vec3 normal, vec3 fragPos, vec3 viewDir)
{
    vec3 lightDir = normalize(light.position - fragPos);
    
    // diffuse shading
    float diff = max(dot(normal, lightDir), 0.0);
    
    // specular shading
    vec3 reflectDir = reflect(-lightDir, normal);
    float spec = pow(max(dot(viewDir, reflectDir), 0.0), material.shininess);
    
    // combine results
    vec3 ambient = light.ambient * vec3(texture(material.diffuse, TexCoords));
    vec3 diffuse = light.diffuse * diff * vec3(texture(material.diffuse, TexCoords));
    vec3 specular = light.specular * spec * vec3(texture(material.specular, TexCoords));
    
    // attenuation
    float distance = length(light.position - fragPos);
    float attenuation = 1.0 / (light.constant + light.linear * distance + light.quadratic * (distance * distance));  

    // spotlight intensity
    float theta = dot(lightDir, normalize(-light.direction)); 
    float epsilon = light.cutOff - light.outerCutOff;
    float intensity = clamp((theta - light.outerCutOff) / epsilon, 0.0, 1.0);
    
    return (ambient + diffuse + specular) * attenuation * intensity;
}
```



### 3.6 面光源 Area light

![](./images/light_area.png)

面光源 (单位：$cd$)：由空间中的矩形限定，在所有方向上均匀地在其表面区域上发出光，但仅从矩形的一侧发出，类似于方正的聚光灯

> 由于面光源会同时从几个不同的方向照亮对象，因此阴影比其他类型的光更柔和细微
> 可用于创建逼真的路灯或靠近播放器的一排灯
>
> 小面积的光源可以模拟较小的光源（例如室内照明），比点光源具有更逼真的效果



面光源环境下，[球坐标系](https://baike.baidu.com/item/球坐标系) $(r,\theta,\phi)$：
实际输出的光通量 $\Phi _e$ 与实际接收的光强度 $I_e$ 的转换比例为
$$
\begin{align}
\Phi_e &=
\int _0^{2\pi} \int_0^{\pi \over 2} sin\theta \space d\theta d\phi \space I_e
= \pi \space I_e\\\\
1 \space lm &=\pi \space cd \\
&\approx 3.14 \space cd
\end{align}
$$



### 3.7 光域光源 Photometric light

![](./images/light_IES.png)

光域光源：是一种用来描述光源辐射范围和强度的说明文件，可以模拟任何形状和强度的光源
用来模拟一些由于发光物体形状迥异和自身遮挡原因的光源

Illuminating Engineering Society (IES)：一种被广泛应用的用来描述光域光源的标准格式 `.ies`

**实现方法**：一般会通过将描述光源辐射范围和强度的 [球坐标系](https://baike.baidu.com/item/球坐标系) $(r,\theta,\phi)$ 转化为笛卡尔坐标系，将 IES 文件转化为 (U, V) 分别为 $(\theta, \space \cos \phi)$ 的纹理

```c
float getIESProfileAttenuation ( float3 L, ShadowLightInfo light )
{
    // Sample direction into light space
    float3 iesSampleDirection = mul ( light . worldToLight , -L);

    // Cartesian to spherical
    // Texture encoded with cos( phi ), scale from -1->1 to 0->1
    float phiCoord = iesSampleDirection.z * 0.5f + 0.5f;
    float theta = atan2(iesSampleDirection.y, iesSampleDirection.x);
    float thetaCoord = theta * FB_INV_TWO_PI ;
    float3 texCoord = float3(thetaCoord, phiCoord, light.lightIndex);
    float iesProfileScale = iesTexture.SampleLevel(sampler, texCoord, 0).r;

    return iesProfileScale;
}

// ...

att *= getAngleAtt (L, lightForward , lightAngleScale , lightAngleOffset );
att *= getIESProfileAttenuation (L, light );
```





# 二、光的反射模型

> 材质反射模型是对 BRDF 光照模型进行简化和理想化后的经验模型

**着色（shading）**：计算某个观察方向出射度的过程，期间需要材质属性、光源信息 和 一个等式（这个等式也称为光照模型）
$$
基础材质模型 = 环境光 f_a + 漫反射 f_d + 镜面反射 f_s
$$


![](./images/material.png)

![](./images/material.jpg)



## 1. 环境光 Ambient

是一个全局光照，同一个场景中的所有物体都使用同样的环境光（一般为常量）

**泛光模型**
即只考虑环境光，这是最简单的**经验**模型，只会去考虑环境光的影响

- $K_a$ 代表物体表面对环境光的反射率
- $I_a$ 代表入射环境光的亮度

$$
I_{Env} = K_a I_a
$$



## 2. 漫反射 Diffuse

物体表面随机散射后反射的光照

### 2.1 Lambert 模型

![](./images/light_diffuse.png)

反射的光线强度 与 表面法线和光源方向夹角 的余弦值 成正比（物体背面的光照不会参与着色计算）

- 计算方法
  两个单位向量的点积  $\hat n \cdot I = |\hat n||I| \cos \theta = \cos \theta$
  反射的光线强度 和 **单位**表面法线和**单位**光源方向 的点积 成正比
- 实际计算公式
  max 函数防止出现表面法线 $n$ 和 光源方向 $I$ 夹角 $\theta$ 大于 90 度的情况（即，光源被物体遮挡的情况）

$$
Color_{diff} = Color_{light} \cdot Color_{材质强度} \max(0, \hat n \cdot I)
$$

### 2.2 Half Lambert 模型

基于 Lambert 模型（物体背面的光照会参与着色计算）

- 计算方法
  不通过限制余弦值的大小而是将余弦值的范围从 [-1, 1] 映射到 [0, 1]
- 实际计算公式

$$
Color_{diff} = Color_{light} \cdot Color_{材质强度} (0.5 + 0.5* \hat n \cdot I)
$$



## 3. 镜面反射 Specular

### 3.1 Phong 模型

> Phong 着色：使用 Phong 光照模型，在**片元着色器**逐像素的计算（使用的顶点法线在当前片面的插值）
> Gouraud 着色：使用 Phong 光照模型，在**顶点着色器**逐顶点的计算（计算量相对较小）

Phong 照模型只关心由光源发射，经过物体表面一次反射后**进入摄像机的光线**

**镜面反射 Specular**：

![](./images/light_reflect.png)



**计算反射方向** $r$，已知法线 $\hat n$ 是单位向量，$L$ 是入射光线 $I$ 到 $\hat n$ 的投影
$$
\begin{align}
|\hat n| &= 1 \\ \\
r + I &= 2 L\\
&= 2(|I|cos \theta) \\
&= 2(|\hat n||I|cos \theta) \\
&= 2(\hat n\cdot I) \\
r &= 2 (\hat n \cdot I) \hat n - I
\end{align}
$$

**高光反射**，已知 观察方向 $\hat v$ 是单位向量

![](./images/light_specular.png)
$$
Color_{spec} = Color_{light} \cdot Color_{高光强度} \max(0, \hat v \cdot r)^{Gloss}
$$
Gloss：光泽度，控制高光区域的亮点（光泽度越大，亮点越小）
max 函数防止出现 $v$ 和 $r$ 夹角 $\theta$ 大于 90 度的情况（即，光源在摄像头后侧的情况）



### 3.2 Blinn-Phong 模型

**Phong 光照的缺点**：
当物体的反光度非常小时，它产生的镜面高光半径足以让相反方向的光线对亮度产生足够大的影响。在这种情况下就不能忽略它们对镜面光分量的贡献了

![](./images/light_over_90.png)



Blinn-Phong 光照模型为了解决 Phong 光照的上述缺点，**计算高光强度的方法改为**计算 **半程向量** 与 法线向量 的夹角的方式

![](./images/light_blinn.png)

$$
\hat h = {I + \hat v \over |I + \hat v|}
$$
Blinn-Phong 的高光强度
$$
Color_{spec} = Color_{light} \cdot Color_{高光强度} \max(0, \hat h \cdot \hat n)^{Gloss}
$$


Blinn-Phong 较 Phong 具有更真实的光照效果

![](./images/light_comparrison.png)

![](./images/light_comparrison2.png)



**代码实现**


```glsl
// VS
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec3 aNormal;
layout (location = 2) in vec2 aTexCoords;

out vec3 FragPos;
out vec3 Normal;
out vec2 TexCoords;

uniform mat4 model;
uniform mat4 view;
uniform mat4 projection;

void main() {
    // Word coordinate
    FragPos = vec3(model * vec4(aPos, 1.0));
    
    // mode 4D to 3D for remove translate
    Normal = mat3(transpose(inverse(model))) * aNormal;  
    TexCoords = aTexCoords;
    
    gl_Position = projection * view * vec4(FragPos, 1.0);
}

// FS
vec3 CalcPointLight(PointLight light, vec3 normal, vec3 fragPos, vec3 viewDir)
{
    // lightDir from frag to light
    vec3 lightDir = normalize(light.position - fragPos);
    
    // diffuse shading
    float diff = max(dot(normal, lightDir), 0.0);
    
    // specular shading
    vec3 halfwayDir = normalize(viewDir + lightDir);
    float spec = pow(max(dot(viewDir, halfwayDir), 0.0), material.shininess);
    
    // combine results
    vec3 ambient = light.ambient * vec3(texture(material.diffuse, TexCoords));
    vec3 diffuse = light.diffuse * diff * vec3(texture(material.diffuse, TexCoords));
    vec3 specular = light.specular * spec * vec3(texture(material.specular, TexCoords));
    
    // attenuation
    float distance = length(light.position - fragPos);
    float attenuation = 1.0 / (light.constant + light.linear * distance + light.quadratic * (distance * distance)); 

    return (ambient + diffuse + specular) * attenuation;
}
```



## 4. 微平面模型 Microfacet

现实当中大多数物体的表面都会有非常微小的缺陷：微小的凹槽，裂缝，几乎肉眼不可见的凸起，以及在正常情况下过于细小以至于难以使用 Normal map 去表现的细节。尽管这些微观的细节几乎是肉眼观察不到的，但是他们仍然影响着光的扩散和反射

- 平面越粗糙，这个平面上的微平面的排列就越混乱。当我们特指镜面光/镜面反射时，入射光线更趋向于向完全不同的方向发散 (Scatter) 开来
- 平面越光滑，光线大体上会更趋向于向同一个方向反射，造成更小更锐利的反射

![](./images/microfacets2.png)

### 4.1 Oren Nayarh 模型

Lambert 模型由于是理想环境下的光照模拟，不能正确体现物体微表面（特别是粗糙物体）的光照效果
Oren Nayarh 模型考虑到微小平面之间的 相互遮挡 和 互相反射照明，主要对粗糙表面的物体建模，比如石膏、沙石、陶瓷等

![](./images/oren_nayar.jpg)



### 4.2 GGX 模型

GGX 模型所解决的问题是，如何将微平面反射模型推广到表面粗糙的**半透明**材质，从而能够模拟类似于毛玻璃的粗糙表面的**透射**效果，它也提出了一种新的描述微平面**法线**方向分布的函数

![](./images/GGX.png)



## 5. 菲涅尔反射 Fresnel

**菲涅耳反射** Fresnel reflection（反射的光线所占折射和反射的比率）
观察方向和物体表面法线的夹角越大，反射效果越明显

模拟物理效果的近似公式，其中 $v$ 表示视角方向的**单位向量**，$n$ 表示物体表面**单位法线**

![](./images/light_fresenel_reflection.png)

### 5.1 Schlick 模型

Schlick 模型简化了 Phong 模型的镜面反射中的指数运算，它模拟的高光反射效果跟 pow 运算基本一致，且效率比 pow 运算高，采用以下公式替代
其中，$n_1$ 表示入射光线介质的折射率，$n_2$ 表示折射光线介质的折射率
$$
\begin{align}
F_0 &= ({n_1 - n_2 \over n_1 + n_2})^2\\
F   &= F_0+(1−F_o)(1−v \cdot n)^5 \\
\end{align}
$$

实时渲染中，常会用一些 经验公式 Empircial Formular 来代替，其中 bias，scale，power 是控制项
$$
F(v,n) = max(0,min(1, bias + scale * (1- v \cdot n)^{power}))
$$






# 三、 阴影

## 1. 阴影效果分析

**阴影具有近实（边缘锐利清晰），远虚（边缘模糊）的效果**

根据被遮挡程度，阴影的类型可分为：

1. lit 照亮：没有被遮挡
2. umbra 本影区：完全被遮挡
3. penumbra 半影区：部分被遮挡

![](./images/shadow_map.png)



## 2. 阴影映射 Shadow Mapping

注意：

- 由于阴影数据的精度问题，光源距离物体越远效果越好
- 点光源的阴影（透视投影）需要更高的精度和更小的竖直方向的视角
- 法线最好采用法线贴图，顶点法线生成的阴影在一些特殊视角会有阴影形变问题



整体思路

![](./images/shadow_map2.png)



方法：

1. **渲染深度贴图（阴影贴图）**
   以光的位置为视角进行渲染，我们能看到的东西都将被点亮，看不见的是阴影
   以光源的类型选择 正交投影 或者 透视投影
   
   ```c
   // 存储的是实际 Z 的深度值，没有标准化（这个时候的 Z 无法确定输入范围）
   GLuint depthMap;
   glGenTextures(1, &depthMap);
   glBindTexture(GL_TEXTURE_2D, depthMap);
   glTexImage2D(GL_TEXTURE_2D, 0, GL_DEPTH_COMPONENT, 
                SHADOW_WIDTH, SHADOW_HEIGHT, 0, GL_DEPTH_COMPONENT, GL_FLOAT, NULL);
   glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
   glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
   glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT); 
   glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);
   ```
   
2. **深度贴图纹理坐标计算**
   世界空间坐标 -> 光源空间坐标 -> 裁切空间的标准化设备坐标-> 根据深度贴图和坐标求出阴影深度值

3. 计算片段是否在阴影之中：若当前坐标的 Z 值比深度贴图的值大，则物体在阴影后面，物体有阴影

   ```c
   // shadow 只能为 0 或 1
   // 阴影中只有环境光，没有高光反射和漫反射
   vec3 lighting = (ambient + (1.0 - shadow) * (diffuse + specular)) * color;
	```

   

重点：

1. 不使用颜色缓冲，不包含颜色缓冲的帧缓冲是不完整的，因此只能禁止颜色缓冲
   并且在片源着色器里什么都不干
   
   ```c
   glBindFramebuffer(GL_FRAMEBUFFER, depthMapFBO);
   glFramebufferTexture2D(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_TEXTURE_2D, depthMap, 0);
   glDrawBuffer(GL_NONE);
   glReadBuffer(GL_NONE);
   glBindFramebuffer(GL_FRAMEBUFFER, 0);
   ```
   
2. 获取阴影贴图的值为透视投影下的非线性深度值

   **解决方案**：将非线性深度值通过透视投影的逆变换转换为线性深度，[投影矩阵](../LinearAlgebra/Part1_Matrix.md)
   $$
   \begin{align}
   Z_n &= {{far + near} \over {far - near}} +{2 \cdot far \cdot near \over {far - near}}{1 \over Z_{linear}} \\
   (far - near)Z_n &= (far + near) + 2 \cdot far \cdot near {1 \over Z_{linear}} \\
   {(far - near)Z_n - (far + near) \over 2 \cdot far \cdot near} &= {1 \over Z_{linear}} \\
   Z_{linear} &= {2 \cdot far \cdot near \over (far - near)Z_n - (far + near)}
   \end{align}
   $$
   
3. 在**距离光源比较远**时，多个片段会从深度贴图的同一个值中采样
   当光以一定角度朝向物体表面时，物体表面会产生明显的线条样式

   ![](./images/shadow_line.png)

   **解决方案**：阴影偏移（shadow bias）+ 深度纹理的线性采样 + 精度修正
   根据对阴影贴图应用一个**随 物体表面朝向和光线的角度**变化的偏移量

   ```c
   // 1. 阴影偏移
   float bias = max(0.05 * (1.0 - dot(normal, lightDir)), 0.005);
   float shadow = currentDepth - bias > closestDepth  ? 1.0 : 0.0;
   
   // 2. 精度问题 
   // 2.1 精度打包
   vec4 bitShift = vec4(1.0, 256.0, 256.0 * 256.0, 256.0 * 256.0 * 256.0);
   const vec4 bitMask = vec4(1.0/256.0, 1.0/256.0, 1.0/256.0, 0.0);
   vec4 rgbaDepth = fract(gl_FragCoord.z * bitShift);
   rgbaDepth -= rgbaDepth.gbaa * bitMask;
   
   // 2.2 精度解包
   
   ```
   
   ![](./images/shadow_acne_bias.png)
   
   这样会带来一个问题 —— 悬浮
   
   ![](./images/shadow_peter_panning.png)
   
   解决悬浮的一种方法：通过在生成阴影深度贴图时采用正面剔除的方式，只保留实体物体背面阴影深度，这样阴影的深度更真实，由于偏移出现的部分多余的阴影也会由于阴影深度的更精确而消失，但是地板的深度会去掉
   
   
   
4. 阴影贴图有一定的范围，无法覆盖所有场景

   ![](./images/shadow_texture_scope.png)

   **解决方案**：让阴影贴图范围外的没有阴影

   1. <u>采样位置超出深度贴图边缘</u>
      将阴影贴图的纹理环绕选项设置为 `GL_CLAMP_TO_BORDER`，给边框设一个较亮的白色（最大深度 1）

   2. <u>深度 Z 的范围超过远平面的裁剪范围 -1.0 ～ 1.0</u>
      首先在片源着色器里判断深度值是否超出 1.0，如果超出，强制设置为无阴影

      

5. 阴影贴图受限于分辨率，画出的阴影有锯齿感

   ![](./images/shadow_soft.png)

   **解决方案**：PCF（percentage-closer filtering）
   计算阴影时，多次进行深度图的采样计算，给做一次 BoxBlur 均值滤波，来模糊阴影边缘的锯齿



## 3. 阴影类型

### 3.1 效率最高的阴影（PPS）

**平面投影阴影** Planar Projected Shadows

TODO: https://zhuanlan.zhihu.com/p/31504088



### 3.2 近实远虚的阴影（PCSS）

#### 3.2.1 Percentage-Closer Soft Shadows

PCF 由于采样区域是固定大小的，因此会在所有地方展示同样形状的软阴影。
为了做到**近实远虚**的效果，我们需要一个系数来控制 PCF 的步长，让近处 PCF步长短（清晰），远处 PCF 步长长（模糊）

![](./images/shadow_PCSS.png)


$$
\begin{align}
{W_{Penumbra} \over W_{Light}} &= {{(d_{Receiver} - d_{Blocker})} \over W_{Blocker}} \\
W_{Penumbra} &= {{(d_{Receiver} - d_{Blocker})}  W_{Light} \over W_{Blocker}}
\end{align}
$$

在计算平均深度时，可以使用 mipmap 来加速平均深度的计算，通过减少采样次数的方式来提高效率

```c++
#define BIAS 		5e-5
#define nSamples 	8

float findAVGBlocker(const vec3& coords, const float& bias)
{
    int blockerCount = 0;
    float totalDepth = 0;
    for (int i = 0; i < nSamples - 2; ++i) {
        vec2 uv = vec2(coords.x, coords.y) + u_offsets[i]；
        float shadowMapDepth = sample2D(texDepth, uv);
        if (coord.z > (bias + shadowMapDepth)) {
            totalDepth += shadowMapDepth;
            blockCount += 1;
        }
    }
    
    if (0 == blockCount) {
        return -1.0f;
    } else if (nSamples - 2 == blockCount) {
        return 2.0f;
    } else {
        return totalDepth / float(blockCount);
    }
}

float PercentageCloserSoftShadows(
    const vec3& coords, 
    const vec3& normal, 
    const vec3& lightDir
)
{
    float bias = MAX(BIAS, BIAS * (1.0f - nomral.dot(lightDir)));
    
    // 1. avg blocker depth
    float zBlocker = findAVGBlocker(coords, bias);
    if (zBlocker > EPS) {
        return 1.0f;
    } else if (zBlocker > 1.0f + EPS) {
        return 0.0f;
    }
    
    // 2. penumbra size
    float penumbraScale = (coord.z - zBlocker) / zBlocker;
    
    // 3. filtering
    float sum = 0.0f;
    for (int i = 0; i < nSamples; ++i) {
        vec2 uv = vec2(coord.x, coord.y) + u_offsets[i] * penumbraScale;
        sum += (coord.z > sample2D(texDepth, uv) ? 0.0f : 1.0f);
    }
    
    return sum / nSamples;
}
```

另外还有通过影子都是水平的这个假设 + 概率方差的方式来给 PCSS 计算加速的 Variance Shadow Maps（VSM），以及修正 VSM 漏光问题的 Moment Shadow Mapping（MSM）方法，由于使用场景特定且实现方式复杂等问题，这里不再详述，具体可以看 [实时渲染｜Shadow Map：PCF、PCSS、VSM、MSM - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/369710758)



### 3.3 大场景的阴影（CSM）

**阴影贴图**方法对于**大型场景**渲染显得力不从心，很容易出现**阴影抖动**和**锯齿边缘**现象

**级联式纹理映射** Cascaded Shadow Maps(CSM) 方法根据**对象**到**观察者**的距离提供**不同分辨率**的**深度纹理**来解决上述问题

1. 将**相机**的**视锥体**分割成若干部分，然后为分割的每一部分生成**独立**的**深度贴图**
2. 根据物体在场景中的位置对位置附近的两张深度贴图进行采样，根据 深度 距离来对两个采样进行线性插值



### 3.4 点光源阴影 Point Shadows

点光阴影，过去的名字是万向阴影贴图（omnidirectional shadow maps）技术

方法：

1. 渲染深度**立方体**贴图
   将立方体贴图 GL_TEXTURE_CUBE_MAP 绑定到 FBO 上，通过几何着色器，一次绘制 6 个面的贴图
   顶点着色器：将顶点变换到世界空间
   几何着色器：将所有世界空间的顶点变换到 6 个不同的光空间（输入：一个三角形的 3 个顶点）

   ```c
   // 几何着色器
   #version 330 core
   layout (triangles) in;
   layout (triangle_strip, max_vertices=18) out;
   
   uniform mat4 shadowMatrices[6];
   out vec4 FragPos; // FragPos from GS (output per emitvertex)
   
   void main() {
       for(int face = 0; face < 6; ++face) {
           gl_Layer = face; // built-in variable that specifies to which face we render.
           for(int i = 0; i < 3; ++i) { // for each triangle's vertices
               FragPos = gl_in[i].gl_Position;
               gl_Position = shadowMatrices[face] * FragPos;
               EmitVertex();
           }    
           EndPrimitive();
       }
   }
   
   // 片源着色器
   #version 330 core
   in vec4 FragPos;
   
   uniform vec3 lightPos;
   uniform float far_plane;
   
   void main() {
       // get distance between fragment and light source
       float lightDistance = length(FragPos.xyz - lightPos);
   
       // map to [0;1] range by dividing by far_plane
       lightDistance = lightDistance / far_plane;
   
       // write this as modified depth
       gl_FragDepth = lightDistance;
   }
   ```

   

2. 渲染场景
   为了确保 6 个面的深度贴图边缘都对齐，设置透视投影的视角为 90 度

   ```c
   float ShadowCalculation(vec3 fragPos) {
       // Get vector between fragment position and light position
       vec3 fragToLight = fragPos - lightPos;
       // Use the fragment to light vector to sample from the depth map    
       float closestDepth = texture(depthMap, fragToLight).r;
       // It is currently in linear range between [0,1]. 
       // Let's re-transform it back to original depth value
       closestDepth *= far_plane;
       // Now get current linear depth as the length between the fragment and light position
       float currentDepth = length(fragToLight);
       // Now test for shadows
       float bias = 0.05; 
       // We use a much larger bias since depth is now in [near_plane, far_plane] range
       float shadow = currentDepth -  bias > closestDepth ? 1.0 : 0.0;
   
       return shadow;
   }
   ```


![](./images/shadow_point.png)



### 3.5 透明物体的阴影





### 3.6 环境光遮蔽 SSAO

屏幕空间的环境光遮挡 （Screen Space Ambient Occlusion，SSAO）通过将褶皱、孔洞和非常靠近的墙面变暗的方法近似模拟出间接光照（常用来模拟大面积的光源对整个场景的光照 如，下图）

![](./images/light_ssao.png)

方法：在三维物体已经生成二维图片之后计算遮蔽因子

1. 几何阶段：准备输入数据
   **1.1 渲染当前相机范围的 顶点、法线、线性深度 到观察空间下的 G-Buffer（Geometry Buffer）**
   注意：纹理采样使用 `GL_CLAMP_TO_EDGE` 方法，防止采样到在屏幕空间中纹理默认坐标区域之外的深度值
   
   ```c
   // 几何着色器 VS
   #version 330 core
   layout (location = 0) in vec3 position;
   layout (location = 1) in vec3 normal;
   layout (location = 2) in vec2 texCoords;
   
   out vec3 FragPos;
   out vec2 TexCoords;
   out vec3 Normal;
   
   uniform mat4 model;
   uniform mat4 view;
   uniform mat4 projection;
   
   void main() {
       vec4 viewPos = view * model * vec4(position, 1.0f);
       FragPos = viewPos.xyz; // 观察空间
       gl_Position = projection * viewPos;
       TexCoords = texCoords;
       
       mat3 normalMatrix = transpose(inverse(mat3(view * model)));
       Normal = normalMatrix * normal; // 观察空间 -> 切线空间
   }
   
   // 几何着色器 FS
   #version 330 core
   layout (location = 0) out vec4 gPositionDepth;
   layout (location = 1) out vec3 gNormal;
   layout (location = 2) out vec4 gAlbedoSpec;
   
   in vec2 TexCoords;
   in vec3 FragPos;
   in vec3 Normal;
   
   const float NEAR = 0.1; // 投影矩阵的近平面
   const float FAR = 50.0f; // 投影矩阵的远平面
   float LinearizeDepth(float depth) {
       float z = depth * 2.0 - 1.0; // 回到NDC
       return (2.0 * NEAR * FAR) / (FAR + NEAR - z * (FAR - NEAR));    
   }
   
   void main() {    
       // 1. 储存片段的位置矢量到第一个 G 缓冲纹理
       gPositionDepth.xyz = FragPos;
       // 2. 储存线性深度到 gPositionDepth 的 alpha 分量
       gPositionDepth.a = LinearizeDepth(gl_FragCoord.z); 
       // 3. 储存法线信息到 G 缓冲
       gNormal = normalize(Normal);
       // 4. 储存漫反射颜色
       gAlbedoSpec.rgb = vec3(0.95);
   }
   ```
   
   **1.2 计算法向半球采样在切线空间的位置**
   在**切线空间**内，距离**每个片源**半球形范围内随机取固定数量的采样坐标，一般会将采样点靠近分布
   
      ```c
   // 在应用程序初始化中调用
   // 随机浮点数，范围0.0 - 1.0
   std::uniform_real_distribution<GLfloat> randomFloats(0.0, 1.0);
   std::default_random_engine generator;
   std::vector<glm::vec3> ssaoKernel;
   
   GLfloat lerp(GLfloat a, GLfloat b, GLfloat f) {
     return a + f * (b - a);
   }
   
   for (GLuint i = 0; i < 64; ++i) {
     // 半球采样点 x，y ~ [-1, 1], z ~ [0, 1]
     glm::vec3 sample(
       randomFloats(generator) * 2.0 - 1.0, 
       randomFloats(generator) * 2.0 - 1.0, 
       randomFloats(generator)
     );
     sample = glm::normalize(sample);
     sample *= randomFloats(generator);
     GLfloat scale = GLfloat(i) / 64.0;
     // 将更多的注意放在靠近真正片段的遮蔽上，也就是将核心样本靠近原点分布
     scale = lerp(0.1f, 1.0f, scale * scale);
     ssaoKernel.push_back(sample * scale);  
   }
      ```
   
   **1.3 创建随机核心旋转噪声纹理**
   半球内采样位置会被所有片源共享使用，需要通过随机转动来确保在较低采样数量的情况下有较好的采样效果
   由于，对场景中每一个片段创建一个随机旋转向量，会占用大量内存
   因此，创建一个小的随机旋转向量纹理（4X4）<u>像瓷砖一样反复平铺</u>在屏幕上
   
      ```c
   // 纹理生成
   std::vector<glm::vec3> ssaoNoise;
   for (GLuint i = 0; i < 16; i++) {
     glm::vec3 noise(
       randomFloats(generator) * 2.0 - 1.0, 
       randomFloats(generator) * 2.0 - 1.0, 
       0.0f); // 围绕 Z 轴偏移旋转，因此 Z 轴不需要有任何变化
     ssaoNoise.push_back(noise);
   }
      ```
   
2. 光照处理阶段：计算遮蔽因子

   **2.1 SSAO 阶段**
   SSAO  着色器在 2D 的铺屏四边形上运行，它对于每一个生成的片段计算遮蔽值（为了在最终的光照着色器中使用）。由于环境遮蔽的结果是一个灰度值，只需要纹理的红色分量，所以将颜色缓冲的内部格式设置为 `GL_RED`

   ```c
   #version 330 core
   out float FragColor;
   in vec2 TexCoords;
   
   uniform sampler2D gPositionDepth;
   uniform sampler2D gNormal;
   uniform sampler2D texNoise;
   
   uniform vec3 samples[64];
   uniform mat4 projection;
   
   // 最好设置为 uniform
   int kernelSize = 64;
   float radius = 1.0;
   
   void main() {
       // Get input for SSAO algorithm
       vec3 fragPos = texture(gPositionDepth, TexCoords).xyz;
       vec3 normal = texture(gNormal, TexCoords).rgb;
     
     	// 为了将 [0,1] 的屏幕纹理坐标转化为平铺的噪声纹理坐标 [0,1]
   		// 1. 获取随机旋转向量这里需要一个缩放值
   		const vec2 noiseScale = vec2(800.0f/4.0f, 600.0f/4.0f); // 屏幕 = 800x600
       vec3 randomVec = texture(texNoise, TexCoords * noiseScale).xyz;
     
       // 2. 根据随机旋转向量创建正交坐标
       vec3 tangent = normalize(randomVec - normal * dot(randomVec, normal));
       vec3 bitangent = cross(normal, tangent);
       mat3 TBN = mat3(tangent, bitangent, normal);
     
       // 3. 根据每个片源共用的半球体内采样数量计算遮蔽因子
       float occlusion = 0.0;
       for(int i = 0; i < kernelSize; ++i)
       {
           // 3.1 获取半球体内每个采样点位置（切线空间内）
           vec3 sample = TBN * samples[i];     // 切线 -> 观察空间
           sample = fragPos + sample * radius; // 根据偏移步长和方向，计算偏移后的采样点
           
           // 3.2 将观察空间的采样点投影到屏幕上
           vec4 offset = projection * vec4(sample, 1.0); // from view to clip-space
           offset.xyz /= offset.w; 											// perspective divide
           offset.xyz = offset.xyz * 0.5 + 0.5;          // transform to range 0.0 - 1.0
           
           // 3.3 获取采样点对应周围采样的深度值
           float sampleDepth = -texture(gPositionDepth, offset.xy).w;
           
           // 3.4 检测周围采样的深度如果在法向半球采样半径内，则被保留
           // 从而避免：当检测一个靠近表面边缘的片段时，它将会考虑测试表面之下的表面的深度值
           float rangeCheck = smoothstep(0.0, 1.0, radius / abs(fragPos.z - sampleDepth ));
         
           // 3.5 若周围采样的深度比当前观察深度大，则累加遮蔽因子的值
           occlusion += (sampleDepth >= sample.z ? 1.0 : 0.0) * rangeCheck;           
       }
     
       // 4. 均值化遮蔽因子
       occlusion = 1.0 - (occlusion / kernelSize);
       
       FragColor = occlusion;
   }
   ```

   **2.2 模糊环境遮蔽结果**
   重复的噪声纹理再上一步的图中清晰可见，为了创建一个光滑的环境遮蔽结果，需要用 box bluer 来模糊环境遮蔽纹理

   ```c
   // 需要额外创建 FBO 在存储这种后处理效果
   #version 330 core
   in vec2 TexCoords;
   
   out float fragColor;
   
   uniform sampler2D ssaoInput;
   const int blurSize = 4; // use size of noise texture (4x4)
   
   void main() {
      vec2 texelSize = 1.0 / vec2(textureSize(ssaoInput, 0));
      float result = 0.0;
      for (int x = 0; x < blurSize; ++x) {
         for (int y = 0; y < blurSize; ++y) {
            vec2 offset = (vec2(-2.0) + vec2(float(x), float(y))) * texelSize;
            result += texture(ssaoInput, TexCoords + offset).r;
         }
      }
    
      fragColor = result / float(blurSize * blurSize);
   }
   ```

   **2.3 应用遮蔽因子在光照计算中**
   光照模型中的环境光 = 原来的环境光常量 * 遮蔽因子（环境遮蔽纹理中）

   ```c
   #version 330 core
   out vec4 FragColor;
   in vec2 TexCoords;
   
   uniform sampler2D gPositionDepth;
   uniform sampler2D gNormal;
   uniform sampler2D gAlbedo;
   uniform sampler2D ssao;
   
   struct Light {
       vec3 Position;
       vec3 Color;
   
       float Linear;
       float Quadratic;
       float Radius;
   };
   uniform Light light;
   
   void main() {             
       // 从 G 缓冲中提取数据
       vec3 FragPos = texture(gPositionDepth, TexCoords).rgb;
       vec3 Normal = texture(gNormal, TexCoords).rgb;
       vec3 Diffuse = texture(gAlbedo, TexCoords).rgb;
   	  // BoxBlur 后的遮蔽因子
       float AmbientOcclusion = texture(ssao, TexCoords).r;
   
       // Blinn-Phong (观察空间中)
       vec3 ambient = vec3(0.3 * AmbientOcclusion); // 这里我们加上遮蔽因子
       vec3 lighting = ambient; 
       vec3 viewDir  = normalize(-FragPos); // Viewpos 为 (0.0.0)，在观察空间中
       // 漫反射
       vec3 lightDir = normalize(light.Position - FragPos);
       vec3 diffuse = max(dot(Normal, lightDir), 0.0) * Diffuse * light.Color;
       // 镜面
       vec3 halfwayDir = normalize(lightDir + viewDir);  
       float spec = pow(max(dot(Normal, halfwayDir), 0.0), 8.0);
       vec3 specular = light.Color * spec;
       // 衰减
       float dist = length(light.Position - FragPos);
       float attenuation = 1.0 / (1.0 + light.Linear * dist + light.Quadratic * dist * dist);
       diffuse  *= attenuation;
       specular *= attenuation;
       lighting += diffuse + specular;
   
       FragColor = vec4(lighting, 1.0);
   }
   ```





# 四、游戏中的可移动性

通过对游戏中可移动性的分类，可以对游戏部分数据进行预先计算以提高游戏 Runtime 效率
从游戏 Runtime 的可移动性上，光源/阴影可以分为：

1. 静态 **Static**：不可移动，属性值（颜色、强度） **不变**
   通过离线烘焙（预计算）的方式来实现，直接使用已经计算好的贴图，运行时无需额外的计算
   例：Lightmap 的使用
2. 静止 **Stationary**：不可移动，属性值（颜色、强度） **可变**
   固定光源使用有范围的区域阴影
3. 可移动 **Movable**：可移动，属性值（颜色、强度） **可变**
   需要配置当前场景对应的阴影偏差，以便于实时计算





# 五、光照渲染路径 Rendering Path

渲染路径：**决定光照**如何应用到 shader 中，是当前渲染目标使用光照的流程

|                  | 前向渲染 Forward                     | 延迟渲染 Deferred                                       |
| ---------------- | ------------------------------------ | ------------------------------------------------------- |
| **场景复杂度**   | 简单                                 | 复杂                                                    |
| **光源支持数量** | 少量光源                             | 多光源                                                  |
| **时间复杂度**   | O(g⋅f⋅l)                             | O(f⋅l)                                                  |
| **显存和带宽**   | 较低                                 | 较高                                                    |
| **硬件要求**     | 低，几乎覆盖100%的硬件设备           | 较高，需 MRT 的支持，需要Shader Model 3.0+              |
| **后处理**       | 无法支持需要法线和深度等信息的后处理 | 支持需要法线和深度等信息的后处理，如 SSAO、SSR、SSGI 等 |
| **画质**         | 清晰，抗锯齿效果好                   | 模糊，抗锯齿效果打折扣                                  |
| **屏幕分辨率**   | 低                                   | 高                                                      |

<u>延迟渲染 比 前向渲染 模糊</u>：因为经历了两次光栅化（几何通道和光照通道），相当于执行了两次离散化，造成了两次信号的丢失，对后续的重建造成影响，以致最终画面的方差扩大



## 1. 前向渲染 Forward

前向渲染路径（Unity 默认渲染路径）

- 优点：实现简单；带宽消耗低；MSAA 和透明渲染支持较好
- 缺点：对大规模的光源支持不好；对需要深度和法线的后期处理算法支持不好

方法：对场景中的每个物体分别进行着色，在 VS 或 FS 对 每个光源逐个进行计算（世界坐标系）并累加到 frame buffer
优化：有些作用程度特别小的光源可以不进行考虑（Unity 中只考虑重要程度最大的前 4 个光源）

```c
// 复杂度 O(Objects * Pixels * lights)
void RenderForword() {
    // 1. 遍历场景中的每个物体
    // Unity3D 4.X 版本中，根据光照对物体的距离采用不同程度的计算
    // 根据距离由近到远，采用的光照计算方式在每个光源中均进行：逐像素计、顶点、求调和函数计算 Spherical Harmonic
    for each(object in ObjectsInScene)
	{
        // 2. 遍历像素(在一个 RT 中的像素) [PS]
        for each(pixel in RenderTargetPixels) {
            color = 0;    
            // 3. 遍历所有灯光，将每个灯光的光照计算结果累加到颜色中
            for each(light in Lights) {
                color += CalculateLightColor(light, pixelData);
            }
        }
        WriteColorToRenderTarget(color);
    } // ObjectsInScene
}
```



### 1.1 Forward+

前向渲染增强，又称 *Tiled Forward Rendering*

- 优点：带宽消耗低；MSAA 和透明渲染支持较好；**支持大规模光源**
- 缺点：需要一个 Pre-Z Pass，一个场景要渲染两遍；对需要法线、深度的后处理支持不好

![](./images/RenderingDeferredTiledBased.png)

方法：
1. <u>PrePass (Depth Only Pass / Early-Z Pass)</u>，用来渲染不透明物体的深度，只写入深度不写入颜色
2. <u>Light Culling Pass</u>，把 Z-Buffer 划分成许多四边形（大小一般是 2 的 N 次方，且长宽不一定相等，称为 Tile）
   根据屏幕划分的范围和深度信息构成 Tile 内场景的包围盒，根据 Tile 包围和光的包围盒求交来剔除无用的光源
3. <u>Shading Pass</u>，每个 Tile 根据各自拥有的 Light list 来计算光照



### 1.2 Cluster Forward

分簇前向渲染 *Cluster Forward Rendering*

方法：和 Tiled Forward 一致，只是在划分 Tile 的场景包围盒上更加精确（屏幕空间 + 深度），进而更精准地裁剪光源，避免深度不连续时的光源裁剪效率降低
主要的细分 tile 场景包围盒子的方法有隐式 Implicit 分簇法（绿色），显式 Explicit 分簇法（蓝色）
以下为 Tiled Forward 的划分方式（红色）与 Clustered 的对比图

![](./images/RenderingClusteredDeferred.png)

优化：*Volume Tiled Forward Rendering*，不使用深度（少了一个 Pass），根据相机投影信息和高度 H，计算出 Tile 的深度从而构成 Volume Tile 和光源求交。相对 Cluster Forward 更粗略但是能快速计算出当前 Tile 的光源影响列表





## 2. 延迟渲染 Deferred

延迟渲染 *Deferred Rendering*

- 优点：支持大规模的光源；针对需要深度、法线的后处理算法友好
- 缺点：带宽、显存消耗大；MSAA 和透明渲染支持不好；需要 MRT 硬件支持

![](./images/RenderingDeferred.png)

方法：将光照处理这一步放在三维物体已经生成二维图片之后进行处理（屏幕坐标系）

1. <u>几何 Pass</u>：渲染所有 几何/颜色 到 G-Buffer（Geometry Buffer）
   G-Buffer：用来存储每个像素对应的 Position，Normal，Diffuse Color 和其他 Material parameters（所有变量都在**世界坐标系**下，同一个场景会渲染多次产生多个 Render Target）

   ```c
   // 复杂度 O(Objects * Pixels)
   void RenderGeometryPass() {
       SetupGBuffer(); // 设置几何数据缓冲区
       
       // 1. 遍历场景(非半透明物体)
       for each(Object in OpaqueAndMaskedObjectsInScene) {
           SetUnlitMaterial(Object);    // 设置无光照的材质
           // 2. 渲染 Object 的几何信息到 GBuffer[VS + PS]
           DrawObjectToGBuffer(Object);
       }
   }
   ```
   
2. <u>光照 Pass</u>：使用 G-buffer 计算场景的光照渲染
   每个 light 需要画一个 light volume，以决定它会影响到哪些 pixel
   
   ```c
   // 复杂度 O(Pixels * lights)
   void RenderLightingPass() {
       BindGBuffer();       // 绑定几何数据缓冲区
       SetupRenderTarget(); // 设置渲染纹理
       
       // 1. 遍历像素(在一个 RT 中的像素) [PS]
       for each(pixel in RenderTargetPixels) {
           // 获取 GBuffer 数据
           pixelData = GetPixelDataFromGBuffer(pixel.uv);
           // 清空累计颜色
           color = 0;    
           // 2. 遍历所有灯光，将每个灯光的光照计算结果累加到颜色中
           for each(light in Lights) {
               color += CalculateLightColor(light, pixelData);
           }
           // 写入颜色到 RT
           WriteColorToRenderTarget(color);
       }
   }
   ```



**多渲染目标** *Multiple Render Targets*, MRT 技术：可以一次渲染完成对像素 位置、颜色、法线等对象信息到多个帧缓冲里

```c
// 1. 一个 FBO 绑定多个 buffer
GLuint gBuffer;
glGenFramebuffers(1, &gBuffer);
glBindFramebuffer(GL_FRAMEBUFFER, gBuffer);
GLuint gPosition, gNormal, gColorSpec;

// - 位置颜色缓冲
glGenTextures(1, &gPosition);
glBindTexture(GL_TEXTURE_2D, gPosition);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB16F, SCR_WIDTH, SCR_HEIGHT, 0, GL_RGB, GL_FLOAT, NULL);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, gPosition, 0);

// - 法线颜色缓冲
glGenTextures(1, &gNormal);
glBindTexture(GL_TEXTURE_2D, gNormal);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB16F, SCR_WIDTH, SCR_HEIGHT, 0, GL_RGB, GL_FLOAT, NULL);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT1, GL_TEXTURE_2D, gNormal, 0);

// - 颜色 + 镜面颜色缓冲
glGenTextures(1, &gAlbedoSpec);
glBindTexture(GL_TEXTURE_2D, gAlbedoSpec);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, SCR_WIDTH, SCR_HEIGHT, 0, GL_RGBA, GL_FLOAT, NULL);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT2, GL_TEXTURE_2D, gAlbedoSpec, 0);

// - 告诉OpenGL我们将要使用(帧缓冲的)哪种颜色附件来进行渲染
GLuint attachments[3] = { GL_COLOR_ATTACHMENT0,     GL_COLOR_ATTACHMENT1, GL_COLOR_ATTACHMENT2 };
glDrawBuffers(3, attachments);

// 2. 片源着色器绘制 buffer
#version 330 core
layout (location = 0) out vec3 gPosition;  // location = 0 和 frame buffer 的 GL_COLOR_ATTACHMENT0 对应
layout (location = 1) out vec3 gNormal;
layout (location = 2) out vec4 gAlbedoSpec;

in vec2 TexCoords;
in vec3 FragPos;
in vec3 Normal;

uniform sampler2D texture_diffuse1;
uniform sampler2D texture_specular1;

void main() {    
    // 存储第一个G缓冲纹理中的片段位置向量
    gPosition = FragPos;
    // 同样存储对每个逐片段法线到G缓冲中
    gNormal = normalize(Normal);
    // 和漫反射对每个逐片段颜色
    gAlbedoSpec.rgb = texture(texture_diffuse1, TexCoords).rgb;
    // 存储镜面强度到gAlbedoSpec的alpha分量
    gAlbedoSpec.a = texture(texture_specular1, TexCoords).r;
}
```



### 2.1 Deferred Lighting

延迟光照，又称 *Light Pre-Pass*

- 优点：相对于 Deferred 可以不需要 MRT 硬件支持；支持 MSAA
- 缺点：带宽、显存消耗大（相对于 Deferred，在几何 Pass 时减少，但 drawcall 多了一次）；透明渲染支持不好

方法：

1. <u>几何 Pass</u>：只在 G-Buffer 中存储 深度信息、法线值 RGB 和 粗糙度 Alpha（只需要绘制到 1 个 RT，不需要 MRT）

2. <u>光照 Pass</u>：在 FS 阶段利用 G-Buffer 计算出所必须的 light properties
   Channel RGB 存储：根据 Normal，LightDir, LightColor, 光的衰减系数 得到的漫反射值
   Channel A 存储：根据 Normal，LightDir, LightColor, 光的衰减系数，以及高光强度，观察角度 得到的高光反射值
   高光反射的值本是一个三维数据，通过将颜色转换为亮度的公式转化为一维数据存储
   $$
   lum(x) = 0.2126 x_r + 0.7152x_g + 0.0722x_b
   $$

3.  <u>第二次几何 Pass</u>：将结果送到 forward rendering 渲染方式在 FS 里计算最后的光照效果



### 2.2 Tile-Based Deferred（TBDR）

基于瓦片的延迟渲染

- 优点：相对于 Deferred，在光照 Pass 时带宽消耗减少
- 缺点：显存消耗大；MSAA 和透明渲染支持不好；需要 MRT 硬件支持

![](./images/RenderingDeferredTiledBased.png)

方法：

1. <u>几何 Pass</u>：生成 G-Buffer，这一步和传统 deferred shading 一样
2. <u>光照 Pass</u>：把 G-Buffer 划分成许多四边形（大小一般是 2 的 N 次方，且长宽不一定相等，称为 Tile）
   根据屏幕划分 Tile，构成 Tile 的包围盒，并将其与光的包围盒求交来剔除无用的光源
   对于 G-Buffer 的每个 pixel，用它所在 Tile 的 light list 累加光照计算 shading



### 2.3 Clustered Deferred

分簇延迟渲染 *Clustered Deferred Rendering*

方法：和 TBDR 一致，只是在划分 Tile 的场景包围盒上更加精确（屏幕空间 + 深度），进而更精准地裁剪光源，避免深度不连续时的光源裁剪效率降低
主要的细分 tile 场景包围盒子的方法有隐式 Implicit 分簇法（绿色），显式 Explicit 分簇法（蓝色）
以下为 TBDR 的划分方式（红色）与 Clustered 的对比图

![](./images/RenderingClusteredDeferred.png)





# Reference

- [learnopengl-Lighting Advanced](https://learnopengl-cn.github.io/05 Advanced Lighting/01 Advanced Lighting/)
- [Schlick's approximation](https://en.wikipedia.org/wiki/Schlick's_approximation)
- [Everything has Fresnel](http://filmicworlds.com/blog/everything-has-fresnel/)
- [SIGGRAPH 2012 Course: Practical Physically Based Shading in Film and Game Production](https://blog.selfshadow.com/publications/s2012-shading-course/#course_content)
- [Unity_Shaders_Book](https://github.com/candycat1992/Unity_Shaders_Book)
- [MSDN-Cascaded Shadow Maps](https://docs.microsoft.com/zh-cn/windows/win32/dxtecharts/cascaded-shadow-maps?redirectedfrom=MSDN)
- [GDC: Deferred Shading](http://developer.amd.com/wordpress/media/2012/10/D3DTutorial_DeferredShading.pdf)
- [Deferred lighting approaches](http://www.realtimerendering.com/blog/deferred-lighting-approaches/)
- [Deferred Shading VS Deferred Lighting](https://blog.csdn.net/BugRunner/article/details/7436600)
- [Clustered Deferred and Forward Shading](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.592.3067&rep=rep1&type=pdf)
- [Decoupled deferred shading for hardware rasterization](https://cg.ivd.kit.edu/publications/p2012/shadingreuse/shadingreuse_preprint.pdf)
- [Microfacet Models for Refraction through Rough Surfaces](https://www.cs.cornell.edu/~srm/publications/EGSR07-btdf.pdf)
- [基于物理的渲染—更精确的微表面分布函数 GGX](https://blog.uwa4d.com/archives/1582.html)
- [UE4 BRDF 公式解析](https://zhuanlan.zhihu.com/p/35878843)
- [阴影渲染](https://zhuanlan.zhihu.com/p/102135703)
- [使用顶点投射的方法制作实时阴影](https://zhuanlan.zhihu.com/p/31504088)
- [弧长和曲面面积](https://blog.csdn.net/sunbobosun56801/article/details/78657455)
- [实时阴影技术总结 - xiaOp的博客 (xiaoiver.github.io)](https://xiaoiver.github.io/coding/2018/09/27/实时阴影技术总结.html)
- [Cascaded Shadow Maps(CSM)实时阴影的原理与实现](https://zhuanlan.zhihu.com/p/53689987)

