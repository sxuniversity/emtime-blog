---
title: Docker容器访问N系显卡
categories: docker
date: 2024-06-13 20:32:26
updated: 2024-06-13 20:32:26
---

参考文章：https://www.cnblogs.com/linhaifeng/p/16108285.html

在进行深度学习、图像处理或视频编解码等计算密集型任务时，GPU 的加速能力几乎是刚需；而如果我们选择使用 Docker 部署这些应用，让容器能够直接访问宿主机的 GPU 就变得非常关键；

相比直接在物理机或虚拟环境中运行程序，通过 Docker 使用宿主机 GPU 有以下几大优势：
- 环境隔离：每个容器都可以拥有独立的依赖、驱动版本和运行环境，不怕“装坏”系统；
- 更灵活的部署：GPU 资源可以按需挂载到不同容器，便于多项目并行运行；
- 启动快、迁移轻：容器镜像打包后，可快速迁移到其他支持 GPU 的机器上部署；
- 配合 nvidia-docker 使用更丝滑：NVIDIA 官方提供的 nvidia-container-toolkit 工具链，支持零配置挂载 GPU 驱动和 CUDA 库；
- 可通过 Docker 的参数控制容器能使用哪一块 GPU，避免资源争用；

本篇文章将介绍如何让 Docker 容器访问宿主机的 GPU，并实际运行一个简单的测试案例验证 GPU 是否成功启用；

## 安装 NVIDIA Container Toolkit

如果你希望 Docker 容器也能像宿主机一样使用 GPU 进行加速计算，那么安装 NVIDIA 官方提供的 [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html) 是关键的一步；

这个工具链可以帮助我们将宿主机的 GPU 驱动和 CUDA 库映射进容器内部，使得容器可以像“原生运行”一样调用 GPU 资源；

首先，根据你的 Linux 系统版本添加 NVIDIA 的软件源并安装工具包：

```bash
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list

sudo apt update && sudo apt install -y nvidia-container-toolkit
sudo systemctl restart docker
```

安装完成后，nvidia-container-toolkit 会自动配置 Docker，使其支持 --gpus 参数，用于将 GPU 挂载到容器中；

## 验证安装是否成功

要验证安装是否成功，可以通过以下命令检查 Docker 是否已识别 --gpus 参数支持：

```bash
docker run --help | grep -i gpus
```

如果能看到类似 --gpus 的输出，说明 Toolkit 已成功生效；

你还可以运行 NVIDIA 提供的 CUDA 基准容器来确认 GPU 是否真的可用：

```bash
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

输出应该类似于宿主机运行 nvidia-smi 的结果，显示当前 GPU 的使用状态、驱动版本、CUDA 版本等信息；

## 启动一个带 GPU 的容器

```bash
docker run --gpus all -itd --name 容器名 镜像名 bash --privileged=true
```

- --gpus all：将所有可用的 GPU 设备挂载到容器中；
- -itd：交互式后台运行容器；
- --name my-gpu-container：指定容器名称；
- --privileged=true：容器具有更高权限（某些情况下处理硬件设备更稳定）；

如果你只想绑定特定编号的 GPU（比如第 0 和第 2 块），也可以这样写：

```bash
docker run --gpus '"device=0,2"' ...
```

## 总结

总的来说，通过 NVIDIA Container Toolkit 和 --gpus 参数，Docker 容器可以在保持环境隔离的同时，毫无障碍地访问宿主机的 GPU，这为部署 AI 模型训练、视频转码、科学计算等任务提供了极大的便利；

接下来，你可以尝试基于 GPU 的 Python 脚本、TensorFlow、PyTorch 或自定义镜像，在 Docker 容器中真正跑起来；
