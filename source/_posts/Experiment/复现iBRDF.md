---
title: 复现iBRDF
layout: post
categories: [Experiment]
mathjax: false

---

Chen Z, Nobuhara S, Nishino K. Invertible neural BRDF for object inverse rendering[J]. IEEE Transactions on Pattern Analysis and Machine Intelligence, 2021.<!-- more -->

## 1 环境

**操作系统**

Distributor ID:	Ubuntu

Description:	Ubuntu 22.04.1 LTS

Release:	22.04

Codename:	jammy

**Docker**

Client: Docker Engine - Community

​	Version:           20.10.17

​	API version:       1.41

​	Go version:        go1.17.11

​	Git commit:        100c701

​	Built:             Mon Jun  6 23:02:46 2022

​	OS/Arch:           linux/amd64

​	Context:           default

​	Experimental:      true

Server: Docker Engine - Community

​	Engine:

​		Version:          20.10.17

​		API version:      1.41 (minimum version 1.12)

​		Go version:       go1.17.11

​		Git commit:       a89b842

​		Built:            Mon Jun  6 23:00:51 2022

​		OS/Arch:          linux/amd64

​		Experimental:     false

​	containerd:

​		Version:          1.6.7

​		GitCommit:        0197261a30bf81f1ee8e6a4dd2dea0ef95d67ccb

​	runc:

​		Version:          1.1.3

​		GitCommit:        v1.1.3-0-g6724737

​	docker-init:

​		Version:          0.19.0

​		GitCommit:        de40ad0

**依赖**：

1. A C++ 17-compatible compiler
2. CUDA 10.2
3. CMake
4. LibTorch 1.6.0
5. OptiX 7.0.0
6. TinyEXR (bundled)

## 2 实验过程

使用docker/下的脚本构建容器ranix。

如果运行容器时报错[找不到设备驱动](https://github.com/NVIDIA/nvidia-docker/issues/1243)：

```bash
docker: Error response from daemon: could not select device driver "" with capabilities: [[gpu]].
```

可以按照[教程](https://collabnix.com/introducing-new-docker-cli-api-support-for-nvidia-gpus-under-docker-engine-19-03-0-beta-release/)安装NVIDIA Container Runtime，使用`systemctl restart  docker`重启服务，解决这个问题。

下载[MERL BRDF](https://cdfg.csail.mit.edu/wojciech/brdfdatabase)，将二进制文件放到./datasets/merl目录下。挂vpn无法一次性下载全部，只能逐部下载。

![MERL BRDF数据库包含一百种不同材质的反射函数](https://cdfg.csail.mit.edu/assets/images/brdf.jpg)

下载[代码](https://github.com/chenzhekl/iBRDF)到iBRDF-main。

下载指定版本[libtorch](https://blog.csdn.net/weixin_43742643/article/details/114156298#cxx11_ABI_168)和[optix](https://developer.nvidia.com/designworks/optix/download)到./iBRDF-main/third_party/下。

运行容器ranix，将iBRDF-main和datasets复制到ranix:/root/workspace中。

进入iBRDF-main，使用cmake编译项目。

```bash
mkdir build
cd build
cmake ..
make
```

建立文件夹../datasets/merl_processed，进入iBRDF-main运行脚本转换BRDF数据格式适合训练：

```bash
python ./scripts/preprocess_merl.py ../datasets/merl ../datasets/merl_processed
```

训练iBRDF

```bash
Usage: ./build/bin/ibrdf_train [MERL root] [Number of BRDFs per batch] [Number of samples per BRDF] [Number of epochs] [Output]

Example: ./build/bin/ibrdf_train ../datasets/merl_processed 50 10000 10000 ./run/ibrdf
```

报错：

```bash
terminate called after throwing an instance of 'c10::Error'
  what():  CUDA error: no kernel image is available for execution on the device
Exception raised from distribution_nullary_kernel at /pytorch/aten/src/ATen/native/cuda/DistributionTemplates.h:169 (most recent call first):
frame #0: c10::Error::Error(c10::SourceLocation, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >) + 0x69 (0x7ff957bdeeb9 in /root/workspace/iBRDF-main/third_party/libtorch/lib/libc10.so)
frame #1: void at::native::(anonymous namespace)::distribution_nullary_kernel<float, float, 4, at::CUDAGeneratorImpl*, __nv_dl_wrapper_t<__nv_dl_tag<void (*)(at::TensorIterator&, at::CUDAGeneratorImpl*, __nv_dl_wrapper_t<__nv_dl_tag<void (*)(at::Tensor&, double, double, at::CUDAGeneratorImpl*), &(void at::native::templates::cuda::normal_kernel<at::CUDAGeneratorImpl*>(at::Tensor&, double, double, at::CUDAGeneratorImpl*)), 2u>, float, float>), &(void at::native::templates::cuda::normal_and_transform<float, float, 4ul, at::CUDAGeneratorImpl*, __nv_dl_wrapper_t<__nv_dl_tag<void (*)(at::Tensor&, double, double, at::CUDAGeneratorImpl*), &(void at::native::templates::cuda::normal_kernel<at::CUDAGeneratorImpl*>(at::Tensor&, double, double, at::CUDAGeneratorImpl*)), 2u>, float, float> >(at::TensorIterator&, at::CUDAGeneratorImpl*, __nv_dl_wrapper_t<__nv_dl_tag<void (*)(at::Tensor&, double, double, at::CUDAGeneratorImpl*), &(void at::native::templates::cuda::normal_kernel<at::CUDAGeneratorImpl*>(at::Tensor&, double, double, at::CUDAGeneratorImpl*)), 2u>, float, float>)), 2u>>, __nv_dl_wrapper_t<__nv_dl_tag<void (*)(at::Tensor&, double, double, at::CUDAGeneratorImpl*), &(void at::native::templates::cuda::normal_kernel<at::CUDAGeneratorImpl*>(at::Tensor&, double, double, at::CUDAGeneratorImpl*)), 2u>, float, float> >(at::TensorIterator&, at::CUDAGeneratorImpl*, __nv_dl_wrapper_t<__nv_dl_tag<void (*)(at::TensorIterator&, at::CUDAGeneratorImpl*, __nv_dl_wrapper_t<__nv_dl_tag<void (*)(at::Tensor&, double, double, at::CUDAGeneratorImpl*), &(void at::native::templates::cuda::normal_kernel<at::CUDAGeneratorImpl*>(at::Tensor&, double, double, at::CUDAGeneratorImpl*)), 2u>, float, float>), &(void at::native::templates::cuda::normal_and_transform<float, float, 4ul, at::CUDAGeneratorImpl*, __nv_dl_wrapper_t<__nv_dl_tag<void (*)(at::Tensor&, double, double, at::CUDAGeneratorImpl*), &(void at::native::templates::cuda::normal_kernel<at::CUDAGeneratorImpl*>(at::Tensor&, double, double, at::CUDAGeneratorImpl*)), 2u>, float, float> >(at::TensorIterator&, at::CUDAGeneratorImpl*, __nv_dl_wrapper_t<__nv_dl_tag<void (*)(at::Tensor&, double, double, at::CUDAGeneratorImpl*), &(void at::native::templates::cuda::normal_kernel<at::CUDAGeneratorImpl*>(at::Tensor&, double, double, at::CUDAGeneratorImpl*)), 2u>, float, float>)), 2u>> const&, __nv_dl_wrapper_t<__nv_dl_tag<void (*)(at::Tensor&, double, double, at::CUDAGeneratorImpl*), &(void at::native::templates::cuda::normal_kernel<at::CUDAGeneratorImpl*>(at::Tensor&, double, double, at::CUDAGeneratorImpl*)), 2u>, float, float>) + 0x2e6 (0x7ff959f7bc76 in /root/workspace/iBRDF-main/third_party/libtorch/lib/libtorch_cuda.so)
frame #2: void at::native::templates::cuda::normal_kernel<at::CUDAGeneratorImpl*>(at::Tensor&, double, double, at::CUDAGeneratorImpl*) + 0x3af (0x7ff959f7db9f in /root/workspace/iBRDF-main/third_party/libtorch/lib/libtorch_cuda.so)
frame #3: at::native::normal_kernel(at::Tensor&, double, double, c10::optional<at::Generator>) + 0x89 (0x7ff959f79d49 in /root/workspace/iBRDF-main/third_party/libtorch/lib/libtorch_cuda.so)
frame #4: <unknown function> + 0xcc0a84 (0x7ff997ac9a84 in /root/workspace/iBRDF-main/third_party/libtorch/lib/libtorch_cpu.so)
frame #5: <unknown function> + 0xcc11b0 (0x7ff997aca1b0 in /root/workspace/iBRDF-main/third_party/libtorch/lib/libtorch_cpu.so)
frame #6: at::native::normal_(at::Tensor&, double, double, c10::optional<at::Generator>) + 0x32 (0x7ff997aac6a2 in /root/workspace/iBRDF-main/third_party/libtorch/lib/libtorch_cpu.so)
frame #7: <unknown function> + 0x1383085 (0x7ff99818c085 in /root/workspace/iBRDF-main/third_party/libtorch/lib/libtorch_cpu.so)
frame #8: <unknown function> + 0x13a940b (0x7ff9981b240b in /root/workspace/iBRDF-main/third_party/libtorch/lib/libtorch_cpu.so)
frame #9: <unknown function> + 0x142c851 (0x7ff998235851 in /root/workspace/iBRDF-main/third_party/libtorch/lib/libtorch_cpu.so)
frame #10: at::Tensor::normal_(double, double, c10::optional<at::Generator>) const + 0x67 (0x7ff998208c07 in /root/workspace/iBRDF-main/third_party/libtorch/lib/libtorch_cpu.so)
frame #11: at::native::randn(c10::ArrayRef<long>, c10::optional<at::Generator>, c10::TensorOptions const&) + 0x6c (0x7ff997cf9a6c in /root/workspace/iBRDF-main/third_party/libtorch/lib/libtorch_cpu.so)
frame #12: at::native::randn(c10::ArrayRef<long>, c10::TensorOptions const&) + 0x30 (0x7ff997cf9b70 in /root/workspace/iBRDF-main/third_party/libtorch/lib/libtorch_cpu.so)
frame #13: <unknown function> + 0x138095f (0x7ff99818995f in /root/workspace/iBRDF-main/third_party/libtorch/lib/libtorch_cpu.so)
frame #14: <unknown function> + 0x11ab07d (0x7ff997fb407d in /root/workspace/iBRDF-main/third_party/libtorch/lib/libtorch_cpu.so)
frame #15: <unknown function> + 0x11a5a98 (0x7ff997faea98 in /root/workspace/iBRDF-main/third_party/libtorch/lib/libtorch_cpu.so)
frame #16: <unknown function> + 0x11ab07d (0x7ff997fb407d in /root/workspace/iBRDF-main/third_party/libtorch/lib/libtorch_cpu.so)
frame #17: at::randn(c10::ArrayRef<long>, c10::TensorOptions const&) + 0xe7 (0x7ff9980b7557 in /root/workspace/iBRDF-main/third_party/libtorch/lib/libtorch_cpu.so)
frame #18: <unknown function> + 0x18f1f (0x55a212218f1f in ./build/bin/ibrdf_train)
frame #19: <unknown function> + 0x18fbc (0x55a212218fbc in ./build/bin/ibrdf_train)
frame #20: <unknown function> + 0x14278 (0x55a212214278 in ./build/bin/ibrdf_train)
frame #21: __libc_start_main + 0xe7 (0x7ff9563aec87 in /lib/x86_64-linux-gnu/libc.so.6)
frame #22: <unknown function> + 0x1212a (0x55a21221212a in ./build/bin/ibrdf_train)

Aborted (core dumped)
```

可能是libtorch版本并不能用1.6.0













