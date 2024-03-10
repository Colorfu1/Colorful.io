---
layout: post
title:  "High-Resolution Image Synthesis with Latent Diffusion Models"
date:   2024-03-10 13:45:01 +0800
categories: deep learning
tag: stable diffusion
---


* content
{:toc}
[Stable diffusion](https://platform.stability.ai/)
### Core idea
diffusion porcess from piexl space -> latent space

### Method
![Structure]()
1. we utilize an autoencoding model which learns a space that is perceptually equivalent to the image space, but offers significantly reduced computational complexity.
2. Auto ecoder: trained with a perceptual loss and a patch-based adversarial objective, downsamples the image by a factor f = H/h = W/w, and we investigate different downsampling factors f = 2m, with m ∈ N
4. This model can be interpreted as a VQGAN but with the quantization layer absorbed by the decoder
5. This is in contrast to previous works, which relied on an arbitrary 1D ordering of the learned space z to model its distribution autoregressively(autoregressive mothod like pixelCNN or others, they generate pixels one by one) and thereby ignored much of the inherent structure of z.
6. Cross attention mechanism.
  - τθ (y) ∈ R^(M ×dτ), like 77 * 768.(CLIP在训练时设定的最大Token数量为77，故SD在前向推理时：如输入的Prompt的Token数量超过77，则会采取切片操作，只取前77个；如输入Token数量小于77，则采取Padding操作，得到77x768;)
  - Here, φi(zt) ∈ RN ×di  denotes a (flattened) intermediate representation of the UNet implementing. (for example UNet output 24 * 24 * 512，it would be flattened to 576 * 512, unlike the ViT form)