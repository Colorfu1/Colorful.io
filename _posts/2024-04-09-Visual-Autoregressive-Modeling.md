---
layout: post
title:  "Visual Autoregressive Modeling"
date:   2024-04-09 10:13:01 +0800
categories: deep_learning
tag: stable_diffusion
---


* content
{:toc}
### VAR
- VAR redefines the autoregressive learning on images as coarse-to-fine "next-scale prediction" or "next-resolution prediction", diverging from the standard raster-scan "next-token prediction"
- VAR makes GPT-style AR models surpass diffusion transformers in image generation. better (FID 1.80, IS 356.4) and faster (with 20× faster inference speed)
- Scaling up VAR models exhibits clear power-law scaling laws similar to those observed in LLMs, VAR has initially emulated the two important properties of LLMs: Scaling Lawsand zero-shot generalization
- Our approach begins by encoding an image into multi-scale token maps. The autoregressive process is then started from the 1×1 token map, and progressively expands in resolution: at each step, the transformer predicts the next higher-resolution token map conditioned on all previous ones
![OverView]()
#### Related work
- Scaling laws describe the relationship between the growth of model parameters, dataset sizes, computational resources
- Image tokenizer and autoregressive models. VQVAE, VQGAN, VQVAE-2, RQ Transformer, ViT-VQGAN
- Masked-prediction model. MaskGIT, MagVit, MagViT-2, MUSE
- Diffusion models. LDM, DiT, SD3.0, SORA
#### Method
##### Preliminary: autoregressive modeling via next-token prediction
- predict a sequence of indices of vocabulary of size V.
- tokenize an image into several discrete tokens, and define a 1D order of tokens for unidirectional modeling. encode the image and do quantize.
$$
\begin{aligned}&
f=\mathcal{E}(im),\quad q=\mathcal{Q}(f),
\quad\hat{f}=\operatorname{lookup}(Z,q),\quad i\hat{m}=\mathcal{D}(\hat{f}),\\&\mathcal{L}=\|im-i\hat{m}\|_2+\|f-\hat{f}\|_2+\lambda_\mathrm{P}\mathcal{L}_\mathrm{P}(i\hat{m})+\lambda_\mathrm{G}\mathcal{L}_\mathrm{G}(i\hat{m}),\end{aligned}
$$
- $im$ denotes the raw image, $\mathcal{E}(\cdot)$ a encoder, and $Q(\cdot)$ a quantizer, lookup$(Z,v)$ means taking the $v$-th vector in codebook $Z$, the  $q^{(i,j)}$ is the index of codebook, a new image $i\hat{m}$ is reconstructed using the decoder, $\mathcal{L}_{\mathbb{P}}(\cdot)$ is a perceptual loss such as LPIPS, $\mathcal{L}_{\mathbb{G}}(\cdot)$ a discriminative loss like StyleGAN’s discriminator loss, and $\lambda_\mathrm{P},\lambda_\mathrm{G}$ are loss weights.
- Once the autoencoder $\{\mathcal{E},\mathcal{Q},\mathcal{D}\}$ is fully trained it will be used to tokenize images for subsequent training of a unidirectional autoregressive model.
- Disscussion
  - Mathematical premise violation. Image encoders use all messages, the sequence of tokens exhibits bidirectional correlations, conflits to unidirectional dependency assumption of autoregressive models.
  - Structural degradation. This spatial relationship(一个像素的上下左右像素相关性更大) is compromised in the linear sequencex, where unidirectional constraints diminish these correlations.
  - Inefficiency. Generating with a conventional self-attention transformer incurs $O(n^2)$ autoregressive steps and $O(n^6)$ computational cost.
##### Visual autoregressive modeling via next-scale prediction
- shifting from "nexttoken prediction" to "next-scale prediction" strategy
- We start by quantizing a feature map $f\in\mathbb{R}^{h\times w\times C}$ into $K$ multi-scale token maps $(r_1,r_2,\ldots,r_K)$, each at a increasingly higher resolution $h_k\times w_k$, culminating in $r_K$ matches the original feature map's resolution $h\times w.$ The autoregressive likelihood is formulated as:
$$
p(r_1,r_2,\ldots,r_K)=\prod_{k=1}^Kp(r_k\mid r_1,r_2,\ldots,r_{k-1}),
$$
- where each autoregressive unit $r_k\in[V]^{h_k\times w_k}$ is the token map at scale $k$, and the sequence $(r_1,r_2,\ldots,r_{k-1})$ serves as the the “prefix” for $r_k$. During the $k$-th autoregressive step, all distributions over the $h_k\times w_k$ tokens in $r_k$ are inter-dependent and will be generated in parallel, conditioned on $r_k$ 's prefix and associated $k$-th position embedding map. This “next-scale prediction’ methodology is what we define as visual autoregressive modeling (VAR).
![process]()
- Discussion
  - The mathematical premise is satisfied if we constrain each $r_k$ to depend only on its prefix
  - there is no flattening operation in VAR, and tokens in each $r_k$ are fully correlated
  - The complexity for generating an image with n × n latent is significantly reduced to $O(n^4)$
- Tokenization.
  - We employ the same architecture as VQGAN but with a modified multi-scale quantization layer
  - Note that a shared codebook $Z$ is utilized across all scales, ensuring that each $r_k$ 's tokens belong to the same vocabulary $[V].$ To address the information loss in upscaling $z_k$ to $h_K\times w_K$, we use $K$ extra convolution layers $\{\phi_k\}_{k=1}^K.$ No convolution is used after downsampling $f$ to $h_k\times w_k.$
![Algorithm]()
##### Implementation
- VAR tokenizer. K extra convolutions, codebook for all scales with V = 4096 and a latent dim of 32. trained on OpenImages with the compound loss
- VAR transformer. 
  - We adopt the architecture of standard decoder-only transformers akin to GPT-2 and VQGAN, with the only modification of substituting traditional layer normalization for adaptive normalization (AdaLN)
  - we use the class embedding as the start token [s] and also the condition of AdaLN
  - Our model shape hyperparameter follows a simple rule that the width $w$, head counts $h$, and drop rate $dr$ are linearly scaled with the depth $d$ as follows:
$$
\begin{aligned}w=64d,\quad&h=d,\quad&dr=0.1\cdot d/24.\end{aligned}
$$
- 但是inference是如何做的没有仔细说，猜测是。从code book sample一个vector index, upsample 到下一个stage的分辨率，然后过transformer得到预测的indexes，过embeding再过Decoder得到inference的结果。