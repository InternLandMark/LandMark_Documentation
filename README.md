<p align="center">
    <picture>
    <img src="./pictures/logo.png" width="600">
    </picture>
</p>

<p align="center"> <font size="4"> 🌏 NeRF the globe if you want </font> </p>

# Introduction

There are many one-of-a-kind highlights in the LandMark:

- Large-scale, high-quality novel view rendering:
    - For the first time, we realized efficient training of 3D neural scenes on over 100 square kilometers of city data; and the rendering resolution reached 4K. We used over 200 billion learnable parameters to model the scene.
- Multiple feature extensions:
    - Beyond rendering, we showcased layout adjustments such as removing or adding buildings, and scene stylization with alternative appearances such as changes of lighting and seasons.
- Training, rendering integrated system:
    - We delivered a system covering algorithms, operators, and computing systems, which serves as a solid foundation for the training, rendering, and application of real-world 3D large models.

To achieve better performance, we use several optimisation strategies, and this document provides an introduction to the following strategies:

- [Parallel Strategies](#parallel-strategies)
- [Dynamic Fetching](#dynamic-fetching)

# Parallel Strategies
The LandMark supports multiple types of parallel strategies. With the strategies, we achieve huge NeRF speedups and make the NeRF truly applicable to city-scale reconstruction. These parallel strategies mainly include the following types:

- [Branch Parallel](#branch-parallel)
- [Plane Parallel](#plane-parallel)
- [Channel Parallel](#channel-parallel)

Here we present how our Parallel methods work. Basic knowledge of NeRF are needed for better understanding.

## Branch Parallel

<p align="center">
<picture>
    <img src="./pictures/multi-branch%20model.png" width="600">
</picture>
</p>

The scene area is divided into multiple scene blocks. Correspondingly, the model changes from having a single large grid to multiple sub-grids. with each sub-grid representing a scene block.

All sub-grids share the same MLPs to decode features and output rgbσ. All the sub-grid and the shared MLPs constitute a complete multi-branch large model.

When a point sampled along the ray is fed into the multi-branch model, it is first assigned to the corresponding sub-grid based on its coordinate, and then inferred with this branch.

<p align="center">
<picture>
    <img src="./pictures/Branch.png" width="500">
</picture>
</p>

## Plane Parallel

Plane parallel divides the full plane into multiple sub-plane, and then scatters to different devices. Thus, each device holds a part of the full plane. Apart from the plane, the remaining modules are replicated across all devices, and their parameters are shared among all devices.

During training, when a point sampled along the ray is fed into the model, it is first assigned to the sub-plane and the corresponding device based on its x-y coordinate, and then inferred on the device to get its rgbσ value. After inferring a batch of points, the rgbσ values are gathered across all devices.

After training with plane parallel, the weights of all sub-planes can be merged into a full plane. In this way, the model trained with plane parallel can be rendered just like a single model.

Alternatively, the plane weights keep separate without merging, then the model is rendered just following the inferring way in training.

<p align="center">
<picture>
    <img src="./pictures/plane.png" width="600">
</picture>
</p>

## Channel Parallel

In channel parallel, both feature plane and feature line are shared along the channel dimension. The features computed by grid_sample are also shared along the channel dimension. Then we can get a full-channel feature if we do gather/all-gather on these shared features.

<p align="center">
<picture>
    <img src="./pictures/channel.png" width="300">
</picture>
</p>

# Dynamic Fetching
Dynamic Fetching is a memory swap-in-swap-out strategy designed to address the fact that a single GPU memory cannot hold the full set of model parameters as the total area increases. 

It takes advantage of spatial locality to load model parameters within a certain range around the coordinates of the current camera from the CPU to the GPU according to the coordinates of the current camera, and restricts the camera's field of view by limiting the height and tilt angle of the camera so that it can only see the rendered image within the range of the loaded model parameters.

When the camera moves, the model parameters in the next range are prefetched according to a certain heuristic strategy, so that when the camera moves to the next range, the prefetched model parameters can be seen, and the loading effect is the same as loading the full model parameters on the GPU.

<p align="center">
<picture>
    <img src="./pictures/dynamic fetching.png" width="350">
</picture>
</p>

The current dynamic fetching scheme is to use ping-pong buffers for parameter switching, defined as a computation buffer and a prefetch buffer, respectively, storing the adjacent N-block parameters as a complete tensor in the buffer and swapping the two buffers when computation is required and prefetching is complete.

<p align="center">
<picture>
    <img src="./pictures/fetch strategy.png" width=“300” height="300">
</picture>
</p>

If the viewpoint is moved, and the difference between the coordinates after the move and the centre coordinates of the last load exceeds the loading threshold, then dynamic fetching is triggered. The direction of fetching is determined based on its offset, and rank0 loads the block of the corresponding region from the CPU into the prefetch buffer. For the overlapping portion of the block, it is stored in the device memory. As for the new block, given the limitation of PCIe speed, it is broadcasted from rank0 to other ranks via collective communication to update the parameters.

<p align="center">
<picture>
    <img src="./pictures/distributed fetching.png" width="800">
</picture>
</p>


To attain rapid dynamic fetching and guarantee a smooth transition between regions in the real-time rendering system, we employ overlapped pipeline optimization. For the blocks requiring to remain, they must be copied to another buffer, and for the new blocks, they must be copied from the CPU to rank0 and sent to other ranks via aggregate communication. Performing this process sequentially results in significant time overhead (as shown in the Dynamic Fetching section), and communicating the entire buffer as a whole introduces redundant traffic, depending on the data continuance requirements of NCCL communication. 

<p align="center">
<picture>
    <img src="./pictures/overlap optimization.png" width="500" >
</picture>
</p>
To mitigate these issues, we propose the Overlapped pipeline optimisation (as depicted in the figure) which involves executing H2D (host to device) and D2D (device to device) copies in parallel as they do not have data dependency or resource competition. For H2D copies, we rearrange the operators and insert the corresponding fine-grained asynchronous communication immediately after each copy, while starting the next copy, so that the communication overhead is hidden and the D2D data does not need to be communicated as the small chunks of data are continuous. Please note that the single chunk operation depicted in the figure only represents a single operation, rather than an actual comparison of execution time. Therefore, optimization should result in a faster speedup than shown in the figure, with a total time ideally close to that of the H2D copy time, which is solely influenced by the amount of data and PCIe bandwidth speed.