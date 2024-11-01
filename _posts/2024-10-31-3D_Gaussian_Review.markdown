---
layout: post
title:  "3D Gaussian Splatting for Real-Time Radiance Field Rendering"
categories: Papers
tags: [paper]
---
<img src="/assets/img31.png" align="center" >
> Bernhard Kerbl, Georgios Kopanas, Thomas Leimkühler, and George Drettakis. 2023. 3D Gaussian Splatting for Real-Time Radiance Field Rendering. ACM Trans. Graph. 42, 4, Article 1 (August 2023), 14 pages. [https://doi.org/10.1145/3592433](https://doi.org/10.1145/3592433)

# Overview
Radiance Field methods have recently shown novel-view syntheses with high visual quality. However, the time for training and rendering makes these methods hard to result real-time (≥ 30 fps) synthesis with high resolution (1080p). The author overcomes this problem by rendering 3D Gaussians, instead of querying along the ray from the camera, which [NeRF](https://www.matthewtancik.com/nerf) method uses. The volumetric radiance fields avoid unnecessary computation or query for ray marching in empty space.  
<br/>
<img src="/assets/img23.png" align="center" >
*NeRF(left) queries all points along the ray from camera,
whereas 3D Gaussian splatting(right) renders a set of 3D Gaussians which is just a bundle of matrices.*
<br/>


This paper has three main components for the solution. First, the author introduces **3D Gaussians** for a scene representation. 3D Gaussians are differentiable volumetric representation which enables gradient descent. They can also be efficiently projected to 2D and applies 𝛼 blending.
Second, optimization of the properties of 3D Gaussians are interleaved with **adaptive density control** by adding or removing 3D Gaussians while optimization.
The third component is **tile-based rasterization** with GPU sorting algorithm. This method enables fast backward pass while rendering visible 3D Gaussians.  
<br/>

# Differentiable 3D Gaussian Splatting
<img src="/assets/img24.png" align="center" >
The author aims to optimize a scene representation starting from a spare point cloud from Structure-from-Motion (SfM) process. To represent the points, they choose 3D Gaussian which is differentiable and can be easily projected to 2D. The Gaussians are defined by covariance matrix $\Sigma$ defined in world space centered at point $\mu$:
\\[G(x) = e^{-\frac{1}{2}(x)^T\Sigma^{-1}(x)}\\]
Also, we get the projection of covariance matrix $\Sigma$ to camera space given a viewing transformation $W$:
\\[\Sigma^\prime = JW\Sigma W^TJ^T\\]
where $J$ is the Jacobian of the affine approximation of the projective transformation.  
<br/>
The covariance matrix $\Sigma$ has physical meaning only when it is positive semi-definite. However, gradient descent may produce invalid covariance matrices, which is not positive semi-definite. Instead, the author adopts scaling matrix $S$ and rotation matrix $R$, which is analogous to an ellipsoid. We now can find $\Sigma$, given $S$ and $R$:
\\[\Sigma = RSS^TR^T\\]
For independent optimization for both $S$ and $R$, they are stored separately: a 3D vector $s$ and a quaternion $q$ respectively.  
<br/>

# Optimization with Adaptive Density Control of 3D Gaussians

The optimization aims to get a dense set of 3D Gaussains that accurately represent the scene. In addition to positions $p$, $\alpha$, and covariance $\Sigma$, they also optimize spherical harmonic (SH) coefficients representing color $c$. The spherical harmonic can represent view-dependent appearance.  
The optimization is based on iterations of rendering the image and comparing with the dataset. The covariance matrix is initialized by the axes of an isotropic Gaussian which equal to mean of the distances to the closest three points. From the initial sparse set of Gaussians, the scene is represented by denser set of Gaussians.  
<img src="/assets/img25.png" align="right" width="50%">
The adaptive control densifies Gasussians with an average magnitutde of view-space position gradients above a threshold $\tau_{pos} = 0.0002$. There are two cases where the Gaussians should be denser for the accurate representation. When geometric features are missing compared to the dataset ("under-reconstruction"), the Gaussians are cloned and moved to the direction of positional gradient. On the other hand, when large Gaussians with high-covariance excessively cover the region ("over-reconstruction"), Gaussians are replaced by two new ones divided by a factor of $\phi = 1.6$. The position is also initialized using the original 3D Gaussian. The first case increases the total volume, starting from sparse points, while the second case obtain high frequency with more Gaussians.  
Similar to other volumetric representations, the optimization can be stuck by floaters near the input camera, resulting the increase in the number of Gaussians. To moderate this, the $\alpha$ values are set to near zero every $N = 3000$ iterations. This method increases the $\alpha$ values of valid Gaussians while decreasing the $\alpha$ values of invalid Gaussians that largely covers the view space. And these Gaussians are removed when $\alpha$ becomes less than $\epsilon_\alpha$.  
<br/>

# Fast Differentiation Rasterizer for Gaussians
For real-time rendering, approximation of $\alpha$-blending by choosing the appropriate number of splats is the key. It starts by splitting the screen into $16\times 16$ tiles and culls 3D Gaussians against the view frustum and each tile. It takes Gaussians with a 99% confidence interval intersecting the view frustum. Also it removes Gaussians with extreme positions since they may result unstable 2D covariance when projected. Then the Gaussians are sorted by depth in the viewspace per each tile. From the front Gaussians, we accumulate color and $\alpha$ values by traversing the sorted list to the back. When all pixels are staturated (i.e. $\alpha$ goes to 1), the processing stops.  
When updating the parameters, we have to find the opacity of each point which is an coefficient for gradient descent. The forward pass starts from the last point that affects any pixels in the tile. During the pass each point stores the final accumulated opacity $\alpha$. With this accumulation, the intermediate opacity can be easily obtained by dividing by the total opacity during the backward pass.  
These processes can be easily parallelized by loading/sharing the data within a tile. Moreover, the sorted list allows to utilize the results of the previous steps.
<br/>

# Results and Evaluations
<img src="/assets/img26.png" align="center">
Despite the little difference in image quality compared to Mip-NeRF360, the current SOTA, 3D Gaussian splatting shows significant performance on training time and FPS. However, we can see that the memory consumption is the main weakness of this method.

# Ablations
## Initialization from SfM
<img src="/assets/img27.png" align="center" width="50%">

- Uniform sample instead of SfM points avoids complete failure. Still, degradations are mainly on the background and some floaters are not removed during the optimization.  

## Densification
<img src="/assets/img28.png" align="center" width="50%">

- Splitting big gaussians reconstructs the background and cloning small gaussians allows better a convergence.

## Unlimited depth complexity of splats with gradients
<img src="/assets/img29.png" align="center" width="50%">

- The small number of depth makes optimization unstable and leads to a severe approximation. 

## Anisotropic covariance
<img src="/assets/img30.png" align="center" >

- Isotropic Gaussians fails to render in detail.

# Limitations
- Artifacts appear in region where the original image is not well observed. This applies same to previous methods.
- Artifacts of (large) Gaussians suddenly appears. This seems to be caused during culling or rasterization since these processes are view-dependent.
- Excessive memory consumption. The author expects that further optimization on low-level can solve this. 

# Conclusion
This method gives the first approach on real-time high-quality radiance feild rendering. Volumetric rendering of 3D Gaussians allows fast optimization and rasterization compared to other radiance field methods.
