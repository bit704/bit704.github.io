---
title: 复现Extracting
layout: post
categories: [Experiment]
mathjax: false
---

Munkberg J, Hasselgren J, Shen T, et al. Extracting Triangular 3D Models, Materials, and Lighting From Images[C]//Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition. 2022: 8280-8290.

<!-- more -->

[代码仓库](https://github.com/NVlabs/nvdiffrec)

装环境的时候使用国内源不要使用-c pytorch

驱动要求：

https://github.com/NVlabs/nvdiffrec/issues/46

安装 CUDA Toolkit 11.7 [link](https://developer.nvidia.com/cuda-downloads?target_os=Windows&target_arch=x86_64&target_version=10&target_type=exe_network)

安装 Vulkan SDK (someone mentioned to do this in a linux post) NOTE! Make sure that the Vulkan SDK is in your PATH environment variables: [Vulkan SDK Install Instructions](https://vulkan.lunarg.com/doc/view/1.2.148.1/windows/getting_started.html)

