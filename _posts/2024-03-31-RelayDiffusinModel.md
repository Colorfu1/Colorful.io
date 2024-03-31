---
layout: post
title:  "Relay Diffusion: Unifying diffusion rpocess across resolutions for image synthesis"
date:   2024-03-24 16:52:01 +0800
categories: deep_learning
tag: stable_diffusion
---


* content
{:toc}
### RDM
#### Core idea
- diffusion models face challenges in high-resolution generation, we found that the same noise level on a higher resolution results in a higher Signal-to-Noise Ratio in the frequency domain.
- transfers a low-resolution image or noise into an equivalent high-resolution one for diffusion model via blurring diffusion and block noise. the diffusion process can continue seamlessly in any new resolution or model without restarting from pure noise or low resolution conditioning

#### background
- diffusion models still require a large amount of resources to train on high-resolution images, The cascaded method (Cascaded diffusion models for high fidelity image generation; Blurring diffusion models) trains a series of varying-size super-resolution diffusion models, which is effective but **needs a complete sampling for each stage separately**.
- an ideal noise schedule should be resolution-dependent, resulting in suboptimal performance to train high-resolution models directly with common schedules designed for resolutions of 32×32 or 64×64 pixels.
- the cascaded method contributes in both training efficiency and noise schedule
  - It provides flexibility to adjust the model size and architecture for each stage to find the most efficient combination
  - The existence of low-resolution condition makes the early sampling steps easy, so that the common noise schedules (optimized for low-resolution models) can be applied as a feasible baseline to the super-resolution models
  - high-resolution images are more difficult to obtain on the Internet than low-resolution images. The cascaded method can leverage the knowledge from low-resolution samples, meanwhile keep the capability to generate high-resolution images.
- The disadvantages of the cascaded method
  - Although the low-resolution part is determined, a complete diffusion model starting from pure noise is still trained and sampled for super-resolution, which is time-consuming
  - The distribution mismatch between ground-truth and generated low-resolution condition will hurt the performance, so that tricks like conditioning augmentation (Ho et al., 2022) become vitally important to mitigate the gap. Besides, the noise schedule of high-resolution stages are still not well studied
- our model RDM
  - In each stage, the model starts diffusion from the result of the last stage, instead of conditioning on
  - shows different perceptual effects on different resolutions, and introduce the block noise to bridge the gap
  - gets rid of the low-resolution conditioning and its distribution mismatch problem(CDM), low-resolution result instead of pure noise, the training and sampling step
  - In this paper, we generally follow the EDM formulation and implementation. The training objective of EDM is defined as $L_{2}$ error terms:
    $$
    \mathbb{E}_{\mathbf{x}\sim p_{data},\sigma\sim p(\sigma)}\mathbb{E}_{\epsilon\sim\mathcal{N}(\mathbf{0},\mathbf{I})}\|D(\mathbf{x}+\sigma\epsilon,\sigma)-\mathbf{x}\|^2,
    $$
  - where $p(\sigma)$ represents the distribution of a continuous noise schedule. $D( \mathbf{x} + \epsilon, \sigma) $ represents the denoiser function depending on the noise scale. We also follow the EDM precondition for $D(\mathbf{x}+\epsilon,\sigma)$ with $\sigma$-dependent skip connection
  - Blurring diffusion is further derived by augmenting the Gaussian noise with heat dissipation for image corruption. simulating the heat equation up to time t is equivalent to a convolution with a Gaussian kernel with variance $σ^2 = 2t$ in an infinite plane. If Neumann boundary conditions are assumed, blurring diffusion in discrete 2D pixel space can be transformed to the frequency space by Discrete Cosine Transformation (DCT) conveniently.
    $$
    q(\boldsymbol{u}_t|\boldsymbol{u}_0)=\mathcal{N}(\boldsymbol{u}_t|D_t\boldsymbol{u}_0,\sigma_t^2\boldsymbol{I}),
    $$
  - where $u_t= DCT(x_t) $ ,and $D_t=e^{\boldsymbol{\Lambda}t}$ is a diagonal matrix with $\Lambda_{i\times W+j}=-\pi^2(\frac{i^2}{H^2}+\frac{j^2}{W^2})$ for coordinate $(i,j)$. Here Gaussian noise with variance $\sigma_t^2$ is mixed into the blurring diffusion process

#### Method
![Illustration](https://github.com/Colorfu1/Colorful.io/raw/master/_posts/resources/2024-03-31-154319.png)
- Frequency spectrum analysis of the diffusion process
  - upsample the 64 × 64one to 256 × 256, perform DCT and compare them in the 256-point DCT spectrum. the high-freqency period is from interpolation and meaningless 意思是这一段的高频是因为上采样导致的，我们看图的时候可以不用关心这一部分。
  - 可以看到，高分辨率的图像更少被噪声影响。经过block noise 之后基本使得高分辨率受噪声的影响和低分辨率受到的影响一致。
  - a higher SNR means that during training the neural network presumes the input image more accurate, but the early steps may not be able to generate such accurate images after the increase in SNR。意思是因为噪声对高分辨率的图片影响更小，所以导致在denoise的最初时刻，需要产出（比低分辨率）更清晰的结果才行，因而会影响效果。
  - Block Noise
  $$
  Block[s](\epsilon)_{x,y}=\frac{1}{s}\sum_{i=0}^{s-1}\sum_{j=0}^{s-1}\epsilon_{x-i,y-j}
  $$
  - where $Block[s](\epsilon)_{x,y}$ is the block noise at the position (x,y), and $\epsilon_{-x}=\epsilon_{x_{max}-x}$. 对于点(x,y),以其为右上点，画一个s*s的范围，用这个范围内的均值代替(x,y)位置的值。
- Why does the cascaded models alleviate this issue?
  - The reason is that in the super-resolution stages, the low-resolution condition greatly ease the difficulty of the early steps, so that even the higher SNR requires a more accurate input, the accuracy is within the capability of the model.
- RDM
![RDM](https://github.com/Colorfu1/Colorful.io/raw/master/_posts/resources/2024-03-31-160333.png)
  - a cascaded pipeline connecting the stages with block noise and (patch-level) blurring diffusion.
  - Suppose that the generated $64\times64$ low-resolution image $\mathbf{x}_0^L=\mathbf{x}^L+\epsilon_L$ can be decomposed into a sample in real distribution $\mathrm{x}^L$ and a remaining noise $\epsilon_L\sim\mathcal{N}(0,\beta_0^2\mathbf{I})$, $\epsilon_H$ is equivalent to $\epsilon_L$, which is Block[4] noise with variance $\beta_0^2$. After (nearest) upsampling, x$^{L^1}$ becomes $\mathrm{x}^H$, where each $4\times4$ grid share the same pixel values. We can define it as the starting state of a $patch{-wise}\:blurring\:diffusion.$
  - we propose to implement the heat dissipation on each 4 × 4 patch independently.define a series of patch-wise blurring matrix $\{D_t^p\}$, forward process would have a similar representation with equation 
  $$
  q(x_t|x_0)=\mathcal{N}(x_t|VD_t^pV^\mathrm{T}x_0,{\sigma_t}^2\boldsymbol{I}),\quad t\in\{0,..,T\},
  $$
  - where $V^\mathrm{T}$ is the projection matrix of DCT and $\sigma_t$ is the variance of noise. Here the $D_T^p$ is chosen to guarantee $VD_T^pV^\mathrm{T}x_0$ in the same distribution as $x^H$ , meaning that the blurring process ultimately makes the pixel value in each $4\times4$ patch the same.
  - The training objective of the high-resolution stage of RDM generally follows EDM framework in our implementation. The loss function is defined on the prediction of denoiser function $D$ to fit with true data $x$, which is written as:
  $$
  \begin{aligned}&\mathbb{E}_{\boldsymbol{x}\sim p_{data},t\sim\mathcal{U}(0,1),\epsilon\sim\mathcal{N}(\mathbf{0},\mathbf{I})}\|D(x_t,\sigma_t)-\boldsymbol{x}\|^2,\\&\mathrm{where}\quad x_t=\underbrace{\color{green}{VD_t^pV^T}}_{\color{green}{bluer}rring}x+\frac\sigma{\sqrt{1+\alpha^2}}\big(\epsilon+\alpha\cdot\underbrace{\color{red}{\mathrm{Block}[s](\epsilon^{\prime})}}_{\color{red}{block~noise}}\big),\end{aligned}
  $$
  - where $\epsilon$ and $\epsilon^{\prime}$ are two independent Gaussian noise. The main difference in training between RDM and EDM is that the corrupted sample $x_t$ is not simply $x_t=x+\epsilon$ but a mixture of the blurred image, block noise and independent Gaussian noise.
- Stochastic sampler
![Sample algorithm](https://github.com/Colorfu1/Colorful.io/raw/master/_posts/resources/2024-03-31-163350.png)

#### Results
![result_1](https://github.com/Colorfu1/Colorful.io/raw/master/_posts/resources/2024-03-31-163722.png)
![result_2](https://github.com/Colorfu1/Colorful.io/raw/master/_posts/resources/2024-03-31-163729.png)
![result_3](https://github.com/Colorfu1/Colorful.io/raw/master/_posts/resources/2024-03-31-163747.png)
![result_4](https://github.com/Colorfu1/Colorful.io/raw/master/_posts/resources/2024-03-31-163823.png)
