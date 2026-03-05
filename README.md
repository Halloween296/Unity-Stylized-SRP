# Unity-Stylized-SRP
基于 Unity C# 的学习向自定义渲染管线包含pbr和npr后处理栈等

模型：miHoYo/神帝宇

<img width="514" height="333" alt="屏幕截图 2026-03-05 183401" src="https://github.com/user-attachments/assets/217287f1-f435-471b-ae51-3b606f919869" />

从零开始实现的自定义渲染管线，支持 PBR 物理渲染和卡通风格化渲染的无缝切换(支持多光源toon用的smooth光照影响更加平滑)，包含体积光、IBL、Bloom 等。

在同一个 Shader 中实现 PBR 和 Toon 的切换
使用 #ifdef _TOON_SHADING 在 Shader 中分支避免重复进行光照计算纹理采样适合一个场景中同时支持 PBR 和 Toon的情况

待解决/优化：
```text
1、问题：
因为为了spot体积光的叠加问题我们单独写了一个shader专门存放阴影为了early-z比如多个面片叠加的情况可以直接抛弃被遮挡的像素没必要进入ps阶段
但是问题是drawcall增加了多了pass
1、解决方法：
如果是移动端或者对gpu要求低的环境我可能会在一个pass中去copy一次深度写入rt去算遮挡关系，或者直接用圆锥形面片代替就不用光线步进体积光了同时
利用深度图调节圆锥的透明度

2、问题：
高光溢出导致ibl反射有死白的地方
2、解决办法：
加aces因为之前软光栅实现过了就不重新写了

3、问题：
还是可能有锯齿因为我用的msaa是多点采样对边缘进行优化而已
3、解决办法：
再加一个fxaa

4、Lut、len有bug待解决以及我这个体积光采样为了偷懒全部像素都发射射线反正会计算聚光灯夹角所以写死了1
4、解决办法：
通过先计算射线发射也就是能被我们看到/聚光灯圆锥体内的范围（求交）剔除不需要的像素再计算降低ALU计算量

5、如果需要节约性能级联阴影可以降低层数用较低分辨率的shadowmap以及ibl漫反射部分可以换成球谐函数环境光并且取消勾选brdf lut查表用近似算、brdf的计算换成f和d项近似算法降低光线步进体积光步数勾选SRP Batcher具体见注释
```
特性：
```text
##光照系统##
PBR 光照（基于物理）：
Metallic-Roughness 工作流
Image-Based Lighting (IBL)
漫反射环境光
镜面反射
BRDF LUT（Split-Sum 近似）

Toon 光照（风格化）：
Ramp 渐变（自定义明暗分界以及分界软硬程度）
SDF 面部阴影
MatCap 高光
Rim Light（边缘光）
多光源 Toon（主光源用 Ramp，辅助光用叠加）

体积光（Ray Marching）
效果： 聚光灯在雾气中形成可见的"光柱"
算法： 从相机发射射线，沿射线步进采样光照
优化：
降采样渲染（性能提升）
随机抖动（消除分层）ue里面用的光线步进云是噪点图做的随机这里用的公式
----------------------------------------------------------------------------
多光源：
方向光（最多 4 个）
点光源 & 聚光灯（最多 64 个）

后处理栈
Bloom（高斯模糊金字塔）
LUT 查找表支持
MSAA 抗锯齿

描边渲染
支持多种描边方法，可切换：
法线外扩法
视图空间法
笔触法（受光面细，背光面粗）同理我ue实现的的sobel算子深度差描边是利用1-dot结果去乘导致背光像素更多多采样邻居实现的详情见作品集
```
```text
Custom RP/
   ├── Runtime/
   │   ├── CameraRenderer.cs
   │   ├── CustomRenderPipeline.cs
   │   ├── CustomRenderPipelineAsset.cs
   │   └── Lighting.cs
   ├── ShaderLib/
   │   ├── Common.hlsl
   │   ├── Lighting.hlsl
   │   ├── BRDF.hlsl
   │   ├── IBL.hlsl
   │   ├── ToonLighting.hlsl
   │   ├── VolumetricLight.hlsl
   │   └── PostFX/
   ├── Shaders/
   │   ├── Lit.shader
   │   └── PostFX.shader
   └── Editor/
       └── LitShaderGUI.cs

