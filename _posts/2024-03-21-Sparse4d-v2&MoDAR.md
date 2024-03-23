---
layout: post
title:  "Sparse4D & MoDar"
date:   2024-03-21 13:04:01 +0800
categories: deep_learning
tag: lidar_deep_learning
---


* content
{:toc}
### Sparse 4D Series
[Link](https://zhuanlan.zhihu.com/p/637096473)
#### V1
- 3d query 和 3d feature initialize -> get sample points, bbox panel center(6) + 7 learnable other points -> project 3d query to history frames and sample feature, get (key_points, timestamp, view, scale) features -> aggregate to current frame's feature.
- depth reweight: 3d feature get depth ditribution and put it in feature.
#### V2
- rnn-like pipeline, put history feature to current frame -> current init query(feature) + history query(feature), cross attention -> current queries(feature)
#### V3
- add noise to GT, get more negtive/positive data
- decoupled atttention: add 2 mlp in before and after attention

### MoDAR
![process](https://github.com/Colorfu1/Colorful.io/raw/master/_posts/resources/2024-03-21-145314.png)
- occluded and long-range objects, can be alleviated by long-term sequence data but unefficiency
- MoDAR, using motion forecasting outputs as a type of virtual modality, to augment LiDAR point clouds; The MoDAR modality propagates object information from temporal contexts to a target frame, represented as a set of virtual points.
- MoDAR, which represents object information propagated from past (or/and from future, in an offline setting) to the current frame
- MoDAR points at T = 0 are generated from motion forecasting on a set of history subsequences (in the online setting) or/and a set of future subsequences (in the offline setting)
- Specifically, given a history subsequence of frames T = −m − K : −m (K + 1 context frames), forward motion forecasting predicts future trajectories of all detected objects at frame T = −m
- In an offline setting, we can access future sensor data, which allows us to take a future subsequence and run reverse motion forecasting to predict object locations backwards.
  - We firstly run a pre-trained LiDAR based 3D object detector to localize and classify objects at every frame in the subsequence
  - we run multiobject tracking using a Kalman-filter based tracker
  - a trajectory prediction model takes the tracked object boxes and predicts future object locations and headings
- Extensions beyond a single prediction
  - First, to fully leverage the point cloud sequence data, we can combine motion forecasting from separate history (or/and future) subsequences by taking a union of the MoDAR points generated from each subsequence. To distinguish their sources, we add an extra channel of the closet frame timestamp (in the subsequence) to the current frame.
  - MultiPath++ predicts 6 possible trajectories with different confidence scores. MoDAR can include all these predictions. To distinguish them, the trajectory confidence can be added as an addition field of the a MoDAR point.
- Compared to the number of LiDAR points in a single frame (around 200K in a frame from the Waymo Open Dataset), the number of MoDAR points is marginal. There are (N × J) MoDAR points from one motion forecasting prediction, where N is the number of objects (usually less than 100), and J is the number of trajectories for each object (e.g. 6).
- we use up to 180 frames (18 seconds: 9 seconds in history and 9 seconds in future) in our experiments.
- run inference MoDAR points to get detection results on all frames at both train set and validation set frames