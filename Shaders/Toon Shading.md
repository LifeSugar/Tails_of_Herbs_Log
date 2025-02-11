首先引入一些Custom lighting functions/sub-graphs
来自https://github.com/Cyanilux/URP_ShaderGraphCustomLighting
![[Pasted image 20250206232643.png]]

主要使用了CustomLighting.hlsl，太牛逼了只能说。

## 额外光源的支持
之前做的类似的Shader只能接受mainlight，这次加入了对额外光源的支持
主要使用了AdditionalToonLightAttenuation方法。
![[Pasted image 20250207005927.png]]
这个方法使用Blinn-Phong模型，忽略了高光，并为了实现TonnShader的色块效果，将diffuse color切割为不同分段。同时这个方法也将输出光源衰减系数的总和

输入
float3 WorldPosition
含义：物体表面或像素在世界空间（World Space）下的位置。
用处：用于计算点光、聚光等附加光源的距离衰减（光源距离顶点/像素有多远？越远衰减越大）；
还可以用来判断物体是否在聚光灯光锥内、或计算屏幕空间坐标（Forward+ 模式下）。
float3 WorldNormal
含义：物体表面在世界空间的法线向量。
用处：如果你需要对光照进行真实的角度计算（如漫反射方向）就会用到法线。但在这段 Toon 逻辑中，主要是做衰减时是否会用到也要看具体的 ToonAttenuation 算法。
注意：这里其实并没有在函数内直接使用到 WorldNormal 进行光照强度计算（当前函数以“衰减”为主），但它保留在参数里，可能给你做额外扩展（比如想加上法线与光线夹角衰减）也会用到。
half4 Shadowmask
含义：当场景使用 Shadowmask 模式（Lighting Settings 中可选）时，用来混合烘焙阴影与实时阴影的贴图采样结果，通常是一个 RGBA 四通道值，其中包含阴影或遮蔽信息。
用处：在函数内部，如果定义了相应的关键词（如 SHADOWS_SHADOWMASK, LIGHTMAP_SHADOW_MIXING），就会把这个值与光照衰减做混合。
注意：如果项目没用 Shadowmask，或者不需要它，可传入 half4(1,1,1,1)。
float PointLightBands & float SpotLightBands
含义：Toon 渲染中，为点光 (Point Light) 和聚光 (Spot Light) 分配了多段分层数。比如 PointLightBands = 3 就意味着把点光的光照从 0~1 拆分成 3 段，呈现卡通硬边效果。
用处：在内部调用 ToonAttenuation(...) 函数时，用这两个参数决定衰减区间如何被分段；数值越高，过渡越多。若 <= 1 表示不做多段，仅做一次简单判断（相当于要么有光要么没光）。

输出
float3 Diffuse
含义：累加后的漫反射颜色 (Diffuse)——也就是场景所有附加光源对这片像素最终贡献的颜色值。
实现：函数内会遍历每个附加光源，计算它的衰减与颜色，然后做 Toon 分段处理，最后把结果相加到 diffuseColor，再通过 out float3 Diffuse 输出。
用处：你可以将此 Diffuse 输出接到 Shader Graph 中合成最终颜色的部分，或再与主光源、纹理贴图等混合，得到完整的着色结果。
float Attenuation
含义：所有附加光源的衰减系数总和。
实现：在函数内部，每个光源会计算 (distanceAttenuation * shadowAttenuation)，然后将它累计到一个总值 attenSum；函数结束时把这结果赋给 Attenuation。
用处：这在 Toon 效果或其他自定义需求中可能非常有用——你可以拿到“光强衰减的合计”来驱动别的效果，比如：
在光源较亮时显示某些特效，较暗时隐藏；
或将其再与主光源的衰减做混合来调控整体亮度；
做自定义 Rim Light、轮廓线、溶解效果等等。
——————————————————


