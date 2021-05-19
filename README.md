# Vulkan 中文教程

该仓库是 [Overv](https://github.com/Overv) 的 [Vulkan 教程](https://github.com/Overv/VulkanTutorial) 的中文镜像仓库，旨在为广大 Vulkan 爱好者提供 Vulkan 教程的中文支持、改善学习体验、扩大 Vulkan 在国内的影响力。

# 翻译完成度

- [ ] Introduction
- [ ] Overview
- [ ] Development Environment
- [ ] Drawing A Triangle
  - [ ] Setup
    - [ ] Base Code
    - [ ] Instance
    - [ ] Validation Layers
    - [ ] Physical Devices And Queue Families
    - [ ] Logical Device And Queues
  - [ ] Presentation
    - [ ] Window Surface
    - [ ] Swap Chain
    - [ ] Image Views
  - [ ] Graphics Pipeline Basics
    - [ ] Introduction
    - [ ] Shader Modules
    - [ ] Fixed Function
    - [ ] Render Passes
    - [ ] Conclusion
  - [ ] Drawing
    - [ ] Framebuffers
    - [ ] Command Buffers
    - [ ] Rendering And Presentation
  - [ ] Swap Chain Recreation
- [ ] Vertex Buffers
  - [ ] Vertex Input Description
  - [ ] Vertex Buffer Creation
  - [ ] Staging Buffer
  - [ ] Index Buffer
- [ ] Uniform Buffers
  - [ ] Descriptor Layout And Buffer
  - [ ] Descriptor Pool And Sets
- [ ] Texture Mapping
  - [ ] Images
  - [ ] Image View And Sampler
  - [ ] Combined Image Sampler
- [ ] Depth Buffering
- [ ] Loading Models
- [ ] Generating Mipmaps
- [ ] Multisampling
- [ ] FAQ
- [ ] Privacy Policy
- [ ] Other

# 构建与部署

## 安装依赖

本文档基于 [MkDocs](https://www.mkdocs.org/) 搭建，而 MkDocs 基于 [Python3](https://www.python.org/)，如未安装 Python3 请先安装。

完成后请使用 pip 安装 MkDocs 和 MkDocs-Material 主题：

```shell
pip3 install mkdocs
pip3 install mkdocs-material
```

完成后可以使用如下指令检查是否安装成功：

```shell
mkdocs --version
```

如果未能找到 MkDocs 请自行找到可执行文件并将其添加到环境变量 `PATH` 中。

## 调试

进入项目根目录，使用如下指令启动调试服务器：

```shell
mkdocs serve
```

访问地址 `http://127.0.0.1:8000` 即可预览文档效果。

# 贡献指南

欢迎成为 VulkanTutorialCN 的贡献者，你可以参与的工作如下：

* 提交 [Issue](https://github.com/VulkanTutorialCN/VulkanTutorialCN/issues) 向我们提出问题
* 提交 [Pull Request](https://github.com/VulkanTutorialCN/VulkanTutorialCN/pulls) 来参与翻译 / 校对。

我们会将你的贡献记录在站点中。

# 许可声明

本仓库使用 [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) 协议进行许可。
