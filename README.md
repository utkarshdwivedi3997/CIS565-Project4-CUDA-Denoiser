CUDA Denoiser For CUDA Path Tracer
==================================

**University of Pennsylvania, CIS 565: GPU Programming and Architecture, Project 4**

* Utkarsh Dwivedi
  * [LinkedIn](https://www.linkedin.com/in/udwivedi/), [personal website](https://utkarshdwivedi.com/)
* Tested on: Windows 11 Home, AMD Ryzen 7 5800H @ 3.2GHz 16 GB, Nvidia GeForce RTX 3060 Laptop GPU 6 GB

## Introduction

|<img src="img/mainNoisy.png" width="400">|<img src="img/mainDenoised.png" width="400">|
|:-:|:-:|
|Original 100 samples per pixel|Denoised|

This is a CUDA implementation of the paper ["Edge-Avoiding A-Trous Wavelet Transform for fast Global Illumination Filtering"](https://jo.dreggn.org/home/2010_atrous.pdf), which describes a fast-approximation for a Gaussian-blur based denoiser for noisy path traced images coupled with an edge-avouding solution. This denoiser helps significantly reduce render times by reducing the overall samples per pixel required to get a denoised render that looks comparable to a render that would require many more samples per pixel to look similar.

This project is an extension of my [CUDA Path Tracer](https://github.com/utkarshdwivedi3997/CIS-565-Project3-CUDA-Path-Tracer).

## Process

### 1. Generate noisy path-traced result

This can be something generated with very few samples per pixel (in my experience as low as 40 spp work well with lambertian diffuse surfaces).

|<img src="img/noisyImg.png" width="300">|
|:-:|
|100 samples per pixel|

### 2. Generate G-Buffer

The paper explains using a G-buffer with world-space positions and world-space normals as a texture. This project's CUDA implementation simply makes a buffer of these values in the first path tracing sample and sends them over to the GPU for use later in the denoising process.

|<img src="img/pos.png" width="300">|<img src="img/nor.png" width="300">|<img src="img/depth.png" width="300">|
|:-:|:-:|:-:|
|World Space Positions|World Space Normals|Depth (only visualised, not needed in G-buffer)|

### 3. Blur noisy pathtraced image using A trous wavelet transform

I'm not a fan of fancy names. What the A trous wavelet transformation described in this paper actually does is a **fast approximation of the Gaussian blur applied over iteratively increasing filter sizes**.

This works as follows: in Gaussian blur, we look at the neighbouring pixels of the current pixel and average out its colours based on a weight kernel that describes how much each pixel's colour contributes to the final average colour of the current pixel. "Denoising" here works by increasing this kernel size over multiple iterations, so we first look at a kernel of 5x5, then 7x7, then 9x9 and so on. This can become expensive, as we start to include more and more pixels for computation with increasing kernel sizes.

This is where "a-trous" comes in. In very simple terms, this is still a Gaussian blur, but *with holes*. When kernel size is increased in successive iterations, the number of pixels sampled for averaging colour is kept constistent: every iteration, every 2<sup>i *th* </sup> pixel is sampled, where *i* is the iteration number. This keeps the kernel size constant, and the iterative blur algorithm fast.

|<img src="img/atrous.png">|
|:-:|
|3 iterations of the fast approximated Gaussian blur kernel (image source: see [paper](https://jo.dreggn.org/home/2010_atrous.pdf))|

This blur looks like this:
|<img src="img/blurIter3.png" width="300">|<img src="img/blurIter5.png" width="300">|<img src="img/blurPS.png" width="300">|
|:-:|:-:|:-:|
|Fast Approximated Blur 3 Iterations|Fast Approximated Blur 5 Iterations|Photoshop's Gaussian Blur|

### 4. Apply edge-stopping function

Finally, we apply an edge stopping function that uses the position, normal from the G-buffer and the colour values from the noisy pathtraced result to maintain sharp edges in the blurred output, based on user-defined weightage given to each of the parameters.

|<img src="img/noisyImg.png" width="400">|<img src="img/denoisedImg.png" width="400">|
|:-:|:-:|
|Original 100 samples per pixel|Denoised|

## Performance and Quality Analysis

### Denoise time compared to path-tracing time

This comparison was done using a filter size of 65x65 (5 denoiser iterations) on the cornell box scene with a specular sphere, by varying the samples per pixel taken during path tracing.

<img src="img/denoisePTCompaison.png">

Since the denoising happens only once at the end of the path-tracing, it does not have a significant cost if used. Since at least 5 samples are required to get a good result anyway, we can use the denoiser without much worry about its performance cost, as that is definitely not the bottleneck.

What is interesting, however, is that the samples per pixel required to produce an "acceptable" result can be significantly lowered by using this denoiser.

|<img src="img/PT5000Iters.png" width="400">|<img src="img/denoisedImg.png" width="400">|
|:-:|:-:|
|Path-traced (5000 samples per pixel)<br>Render time: ~1 minute|Path-traced (100 samples per pixel) + Denoised<br>Render time: 1.03 seconds|

The above denoised image does not look significantly different from the path-traced image, but does cut the render cost by almost 60x!

### Denoise time at different resolutions

This comparison was done using a filter size of 65x65 (5 denoiser iterations) on the cornell box scene with a specular sphere, with increasing image resolution.

<img src="img/imageResDenoiseTime.png">

The time to denoise images with exponentially growing scales also scales exponentially. With that said, when compared to the time taken to path-trace an acceptable image, the denoise time is insignificant, as the path-tracing time is what scales up exponentially as well.

### Denoise time with varying filter sizes

This comparison was done using varying filter sizes on the cornell box scene with a specular sphere, with a constant image resolution of 800x800 pixels and 100 samples per pixel.

|<img src="img/denoiseIter1.png" width="300">|<img src="img/denoiseIter2.png" width="300">|<img src="img/denoiseIter3.png" width="300">|
|:-:|:-:|:-:|
|5x5 fitler (1 denoise iteration)|9x9 filter (2 denoiser iterations)|17x17 filter (3 denoiser iterations)|
|<img src="img/denoiseIter4.png" width="300">|<img src="img/denoiseIter5.png" width="300">|<img src="img/denoiseIter6.png" width="300">|
|33x33 fitler (4 denoise iteration)|65x65 filter (5 denoiser iterations)|129x129 filter (6 denoiser iterations)|

About 5 denoiser iterations are enough to get an acceptable result. The "gain" in visual quality does not seem to scale linearly with increasing filter size, as the gains seem to become lesser and lesser. The difference between the denoised results from 5 and 6 denoiser iterations are not significant enough to justify using more denoiser iterations.

<img src="img/filterSizeDenoiseTime.png">

The time taken to denoise with exponentially increasing filter sizes scales linearly only, since exponentially increasing filter sizes only scale the denoiser iterations linearly (by 1 with each power of 2 filter size). This means that the filter size is not too much of a concern and will not be the primary bottleneck in path-tracing.

### Denoiser vs material types

<ins>**Lambertian Diffuse**</ins>

|<img src="img/diffuse5000.png" width="400">|<img src="img/diffuseDenoised.png" width="400">|<img src="img/diff2.png" width="400">|
|:-:|:-:|:-:|
|Path-traced (5000 samples per pixel)<br>Render time: ~1 minute|Path-traced (100 samples per pixel) + Denoised<br>Render time: <1 second|Diff|

In the diffuse case, the denoiser works best. The denoised results look acceptable, though the only problematic areas are around the edges of the light source. This is an unfortunate side effect of the way this denoiser works.

<ins>**Perfect Specular**</ins>

|<img src="img/PT5000Iters.png" width="400">|<img src="img/denoisedImg.png" width="400">|<img src="img/diff1.png" width="400">|
|:-:|:-:|:-:|
|Path-traced (5000 samples per pixel)<br>Render time: ~1 minute|Path-traced (100 samples per pixel) + Denoised<br>Render time: 1.03 seconds|Diff|

While the result does not look that bad on the specular sphere, it is smoothed out a bit too much, as the normals on the sphere are changing constantly and its difficult to find any edge here for the edge stopping.

<ins>**Refractive Materials**</ins>

|<img src="img/glass.png" width="400">|<img src="img/glassDenoised.png" width="400">|<img src="img/diff3.png" width="400">|
|:-:|:-:|:-:|
|Path-traced (10000 samples per pixel)<br>Render time: ~3.2 minutes|Path-traced (100 samples per pixel) + Denoised<br>Render time: 5 seconds|Diff|

This is where this algorithm starts to fail. Most of the detail that is refracted through glass is blurred out too significantly to be able to be interpreted as refraction. In this example image, even the specular highlights on the icosphere behind the glass monkey are blurred out in the denoiser to the point where it looks like a simple diffuse icosphere.

### Denoiser vs differently sized light sources

|<img src="img/denoiseIter5.png" width="400">|<img src="img/denoiseBig.png" width="400">|
|:-:|:-:|
|Path-traced (100 samples per pixel) + Denoised<br>Render time: 1.03 seconds|Path-traced (100 samples per pixel) + Denoised<br>Render time: 0.82 seconds|

This result is expected. With a larger light source, there would be less noise in the path-traced result simply because a lot more rays would reach the light source. As a result, denoising a less noisy image would result in a smoother looking render.

## References

- Adam Mally's CIS 561 Advanced Computer Graphics course at University of Pennsylvania
- TinyGLTF
- PBRT
- [Edge-Avoiding A-Trous Wavelet Transform for fast Global Illumination Filtering](https://jo.dreggn.org/home/2010_atrous.pdf)