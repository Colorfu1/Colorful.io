---
layout: post
title:  "UniVision: A Unified Framework for Vision-Centric 3D Perception"
date:   2024-03-19 15:45:01 +0800
categories: deep_learning
tag: stable_diffusion
---


* content
{:toc}
[UniVision](https://arxiv.org/abs/2401.06994)

### Core idea
- occ + 3d-detection
- we propose an explicit-implicit view transform module for complementary 2D-3D feature transformation
- propose a local-global feature extraction and fusion module for efficient and adaptive voxel and BEV feature extraction, enhancement, and interaction
- propose a joint occupancy-detection data augmentation strategy and a progressive loss weight adjustment strategy which enables the efficiency and stability of the multi-task framework training

### Method
- input: surrounding images
![model pipeline](https://github.com/Colorfu1/Colorful.io/raw/master/_posts/resources/2024-03-19-155701.png)
- input -> backbone -> image feature --(Ex-Im)--> 3d voxel feature -> voxel feature & bev feature -> infor-exchange on the voxel features and bev features -> output
- LLS generate the voxel features
- 3D voxel feature is highly related to the accuracy of the estimated depth and the points generated is unevenly distributed, Points are dense close to the camera and are sparse in distance ($F_{voxel}^{ex}$)
  - use the query-guided feature sampling to compensate
  - define the learnable voxel queries $q_{voxel} \in R^{C*X*Y*Z}$, use 3D transformer to sample the features from the images.
  - project the voxel center to image plane, use N transformer blocks(deformable cross-attention, 3D Conv and FFN), get $F^{im}_{voxel}$
  - concate the $F^{im}_{voxel}$ and $F_{voxel}^{ex}$ -> $F_{voxel}$
![3D query](https://github.com/Colorfu1/Colorful.io/raw/master/_posts/resources/2024-03-19-172009.png)
![Ex-Im process](https://github.com/Colorfu1/Colorful.io/raw/master/_posts/resources/2024-03-19-172413.png)
- generate bev feature from voxel, $F_{bev} = Conv(Stack(F_{voxel}, dim = Z))$ (b, h, w, d, c) -> (b, h, w, d*c)
- local feature extraction(voxel) & global feature extraction(bev)
  - voxel: 3d conv, ResNet 3D, FPN of SECOND -> $F^{local}_{voxel}$
  - bev: DCNv3, FPN of SECOND -> $$F^{global}_{bev}$$
- cross representation feature interaction
  - lift bev faeture to voxel feature -> $F^{global}_{voxel} = repeat(f_{bev}^{global}, dim = Z)$
  - map the voxel feature to bev feature -> $F^{local}_{bev} = add(F^{local}_{voxel}, dim = Z)$
  - $F^{local}_{voxel}$ as query, $F^{global}_{voxel}$ as key and value -> $F_{voxel}^{fusion}$
  - $F^{global}_{bev}$ as query, $F^{local}_{bev}$ as key and value -> $F_{bev}^{fusion}$.
- OCC loss: the crossentropy loss, Lovasz softmax loss, geometry affinity loss, and semantic affinity loss $$ L_{\text{occ}} = \lambda_1 L_{\text{ce}} + \lambda_2 L_{\text{lovasz}} + \lambda_3 L_{\text{geo}} + \lambda_4 L_{\text{sem}} $$
- Detection loss: the classification loss and the regression loss. $$ L_{\text{det}} = \lambda_5 L_{\text{cls}} + \lambda_6 L_{\text{reg}} $$
- add the depth loss Ldepth used in BEVDepth as the image level supervision $$ L_{\text{img}} = \lambda_7 L_{\text{depth}} $$
- propose the progressive loss weight adjustment strategy to dynamically adjust the loss weights (hard to converge)
$$L = L_{\text{img}} + \delta \cdot L_{\text{det}} + \delta \cdot L_{\text{occ}}$$
$$\delta = \max(V_{\text{min}}, \min(V_{\text{max}}, i - V_{\text{max}} / N))$$
- spatial-level data augmentation
  - propose a joint Occ-Det spatial data augmentation to allow simultaneous augmentation in both the 3D detection task and the occupancy task
  - we propose to transform the voxel features instead of directly operating on the occupancy labels for the data augmentation
$$
 C_{org} = P_{i-c} * I_{org} \\
 I_{aug} = P_{c-i} * M_{aug} * C_{org} \\
 F_{org} = S(F_{aug}, I_{org}) \\
 L_{occ} = f(G_{occ}, F_{org}) * M_{occ}
$$
  - $P_{i-c}$ or $P_{c-i}$ is the transformation matrices between voxel indices and spatial coordinates (世界坐标系和车身坐标系的转换矩阵)
  - $M_{aug}$ is 3D transformation for data augmentation
  - $F_{aug}$ is the augmented features, $F_{ori}$ is the sampled $F_{aug}$ feature according to $I_{ori}$
  - 整体意思就是数据增强的时候进行了旋转然后获取了旋转后的feature_aug， 那么需要记下旋转后的I_aug, 用I_aug采样feature_aug就得到了没有旋转的结果。（品，你细品）
  - 作者的公式写错了。
![Results_1](https://github.com/Colorfu1/Colorful.io/raw/master/_posts/resources/2024-03-19-222908.png)
![Results_2](https://github.com/Colorfu1/Colorful.io/raw/master/_posts/resources/2024-03-19-222931.png)
![Results_3](https://github.com/Colorfu1/Colorful.io/raw/master/_posts/resources/2024-03-19-222939.png)
![Results_4](https://github.com/Colorfu1/Colorful.io/raw/master/_posts/resources/2024-03-19-223002.png)
![Results_5](https://github.com/Colorfu1/Colorful.io/raw/master/_posts/resources/2024-03-19-223014.png)
![Results_6](https://github.com/Colorfu1/Colorful.io/raw/master/_posts/resources/2024-03-19-223023.png)



