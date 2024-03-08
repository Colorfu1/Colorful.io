---
layout: post
title:  "W ̈URSTCHEN: AN EFFICIENT ARCHITECTURE FOR LARGE-SCALETEXT-TO-IMAGE DIFFUSION MODELS"
date:   2024-03-08 14:31:01 +0800
categories: deep learning
tag: stable diffusion
---


* content
{:toc}
Stable cascade based model.\
[Stable Cascade](https://github.com/Stability-AI/StableCascade)

A key contribution of our work is to develop a latent diffusion technique in which we **learn a detailed but extremely compact semantic image representation** used to **guide the diffusion process**

The training requirements of our approach consists of **24,602 A100-GPU hours** – compared to Stable Diffusion 2.1's 200,000 GPU hours

### Problem to solve
[LDM](https://huggingface.co/stabilityai/stable-diffusion-2-1.) is limited by how much the encoder-decoder model can compress the image without degradation.

### Proposed method
A novel three-stage architecture named "W ̈urstchen", training a diffusion model on a very low dimensional latent space with a high compression ratio of 42:1(Stage C only).

A very low dimensional latent space with a high compression ratio of 42:1 (stage C), this very low dimensional latent-space data navigate a higher dimensional latent space of VQGAN (stage B)(compression ratio of 4:1)

**Inference time**, a text-conditional LDM is used to create a low dimensional latent representation of the image (Stage C)(c_c, 24, 24). This latent representation is used to condition another LDM (Stage B), producing a latent image in a latent space of higher dimensionality(c_b, 256, 256). Finally, the latent image is decoded by a VQGAN-decoder to yield the full-resolution output image (Stage A)(3, 1024, 1024).
### Inference Stage
![Inference](https://github.com/Colorfu1/Colorful.io/raw/master/_posts/resources/2024-03-08-144906.png)

**Training time** 
Stage A: Employs a VQGAN to create a latent space
Stage B: A latent diffusion process, conditioned on the outputs of a Semantic Compressor (an encoder operating at a very high spatial compression rate) and on text embeddings. to reconstruct the latent space of Stage A.
Stage C: The strongly compressed latents of the Semantic Compressor from Stage B are used to project images into the condensed latent space where a text-conditional LDM is trained.

### Train Stage
![Train](https://github.com/Colorfu1/Colorful.io/raw/master/_posts/resources/2024-03-08-145442.png)

### Details
1. makes use of Stages A & B to achieve a notably higher compression than usual, Stage A consists of a f 4 VQGAN -> encodes images X ∈ R3×1024×1024 into 256 × 256 discrete tokens. 
2. the quantization is dropped from Stage A, and Stage B is trained in the unquantized latent space of the Stage A-encoder as a conditioned LDM, text and Semantic Compressor results as the condition. Concat the text embeding and Semantic Compressor results and cross-attention.
3. The embeddings of stage C will have a shape of R1280×24×24 obtained by encoding images with shape X ∈ R3×786×786. We use simple bicubic interpolation for the resizing of the images from 1024×1024 to 786×786.
4. Hence we updated the weights of the Semantic Compressor during training(only stage B), establishing a latent space with high-precision semantic information
5. By conditioning Stage B on low-dimensional latent representations, we can effectively decode images from a R16×24×24 latent space to a resolution of X ∈ R3×1024×1024, resulting in a total spatial compression of 42:1.
6. We use a cosine schedule to generate  ̄αt and use continuous timesteps
7. We decided to formulate the stage C objective as such, since it made the training more stable ![objective](https://github.com/Colorfu1/Colorful.io/raw/master/_posts/resources/2024-03-08-153032.png)
8. Stage C consists of 16 ConvNeXt-block (Liu et al., 2022b) without downsampling
9. We use the standard meansquared-error loss between the predicted noise and the ground truth noise. Additionally, we employ the p2 loss weighting:p2(t) · $$\|\| ε −  ̄ε \|\|^2$$,where p2(t) is defined as $$(1−  ̄αt) / ( 1+  ̄αt)$$ , making higher noise levels contribute more to the loss
10. initial noise -> stage C -> regard it as semantic compressor of (16 * 24 * 24) -> (16 * 576), concat with text embeding -> stage B (with 4 * 256  *256 initialized with random tokens from VQGAN codebook) -> stage A decode
11. we trained an 18M parameter Stage A, a 1B parameter Stage B and a 1B parameter Stage C, All stages were trained on subsets of the improved-aesthetic LAION-5B dataset.
12. Both stages also make use of classifier-free-guidance with guidance scale w. We fix the hyperparameters for Stage B sampling to τB = 12 and w = 4, Stage C uses τC = 60 for sampling
13. We provide the percentage of images, where PickScore preferred the image of W ̈urstchen over the image of the other model. For example the line 1, Baseline LDM(ours) and 96.5% express than PickScore perfer W ̈urstchen images over Baseline LDM(ours) 96.5% ![Metric](https://github.com/Colorfu1/Colorful.io/raw/master/_posts/resources/2024-03-08-162015.png)


### Results
With this approach we are able to train a 1B parameter Stage C textconditional diffusion model within approximately 24,602 GPU hours, resembling a 8x reduction in computation compared to the amount SD 2.1 used for training (200,000 GPU hours), while showing similar fidelity both visually and numerically

![other score](https://github.com/Colorfu1/Colorful.io/raw/master/_posts/resources/2024-03-08-162624.png)

a: However, for MS-COCO this statistic is inconclusive. We hypothesize that this is due to the vague prompts generating a more diverse set of images, making the preference more subject to personal taste, biasing this statistics towards users that completed more comparisons

b:By doing so, we include only individuals with at least 30 (MS-COCO) and 51 (Parti-prompts) comparisons in the statistic -> In summary, the human preference experiments confirm the observation made in the PickScore experiments.
![human choice](https://github.com/Colorfu1/Colorful.io/raw/master/_posts/resources/2024-03-08-163710.png)
![speed_1](https://github.com/Colorfu1/Colorful.io/raw/master/_posts/resources/2024-03-08-162807.png)
