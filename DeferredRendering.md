# Deferred Rendering 相关概念的总结

**2016-12-11**

Deferred Rendering 这个名词已经不是很陌生了，相对于 Forward Rendering，它的优势就是光照开销和场景复杂度无关，和光源数量不成线性比例关系，但与要处理的像素数量紧密相关。而 Forward Rendering 就不用多解释了，平时接触最多的就是这个，光照开销和场景复杂度和光源数量紧密相关，和处理像素数量成一定的比例关系，这也就是使用 Lightmaps 的原因了。这里主要讨论的是 Deferred Rendering。

我对 Deferred Rendering 一直以来有一个误解，认为它就是一种具体的渲染算法，其实并不是这样的，Deferred Rendering 只是这一类渲染算法的统称，而对其细分又有了 Deferred Shading 和 Deferred Lighting（当然还有其他进化版本）。这两个才是具体的渲染算法。我要弄清楚的就是 Deferred Shading 是什么，Deferred Lighting 又是什么，以及他们之间的一些关系及对比。以下内容部分来自 Unity 文档，部分为个人总结。

### Deferred Lighting

Deferred Lighting 的实现可分为三个步骤：

1. Base Pass:

	* 有一张屏幕空间大小的 RenderTexture（ARGB32），场景中的所有物体都会被渲染一遍，将摄像机空间中的法线数据存储到 rgb 通道，高光信息存储到 a 通道。如果硬件支持直接读取 Z-Buffer，那么后续步骤中就会直接读取 Z-Buffer，但是如果硬件不支持读取 Z-Buffer，那么整个场景必须再次被渲染一遍，将深度信息存储到自定义的 Z-Buffer 中。
			 	
2. Lighting Pass:

	* 这一步是为了生成一张屏幕空间大小的 Light Buffer，这张 Light-Buffer 包含了光源产生的光照以及阴影，这些都可以从上一步中的法线、高光、深度计算得来。
	
3. Final Pass:

	* 这是整个渲染过程的最后一步，整个场景将再被渲染一遍，将模型本身的 Diffuse、Emission 叠加上 Light-Buffer，如果有必要再与 Lightmaps 进行混合，最终产生的像素输出到屏幕像素缓冲区上。
	
### Deferred Shading

Deferred Shading 的实现有两个步骤：

1. G-buffer Pass:

	* 整个场景会被渲染一遍，Diffuse、Specular、Smoothness、世界空间法线、自发光会被写到 G-Buffer 中，外加一个可读取的 Z-Buffer。注意这里的 G-Buffer 并不是只有一张 RenderTexture，而是好几张，因为很显然这么多数据一张 RenderTexture 是无法容纳得下的。但是常规的 Pass 只能有一个输出，如果用多 Pass 来分别输出到多个 RenderTexture 那么消耗将会成倍增长。因此这里要用到 Multiple Render Targets（MRT）技术，可以在一个 Pass 内产生多个输出，分别输出到这几张 RenderTexture 上。
	
2. Lighting Pass:

	* 通过 G-Buffer 中的信息计算出光照色并叠加上自发光色输出到屏幕像素缓冲区上。
	
### Deferred Lighting VS Deferred Shading

Unity4 中默认的 Dererred Rendering 是 Deferred Lighting，而 Unity5 中默认的 Deferred Rendering 是 Deferred Shading。

Deferred Lighting 会对场景进行多次绘制，两次、也可能是三次，这取决于硬件是否支持读取 Z-Buffer。Deferred Shading 只会对场景进行一次绘制。

Deferred Lighting 需要用来存储 G-Buffer 以及 Light-Buffer 的空间相对较小。Deferred Shading 用来存储 G-Buffer 的空间会要求较高，可能还会有一些特殊的位格式。

Deferred Lighting 所用到的技术都比较普通，设备适配性较好。Deferred Shading 用到了 MRT，这是目前大部分移动设备所不支持的。

Deferred Lighting 只是光照的 Deferred，材质计算还是放在了 Final Pass 中，这样如果有一些特殊的材质属性需求，很容易就能实现。Deferred Shading 将所有信息都放到了 G-Buffer 中，如果有一些特殊的材质需求，就需要想办法将数据塞进本来已经很拥挤的 G-Buffer 中，这样增加了整体的难度和复杂度。

Deferred Lighting 和 Deferred Shading 都只能处理不透明物体，对于透明物体的渲染还是使用 Forward Rendering。都不支持硬件抗锯齿（比如 Multi-Sampling Anti-Aliasing），只能通过软件方式实现（比如 Post Processing Anti-Aliasing）。