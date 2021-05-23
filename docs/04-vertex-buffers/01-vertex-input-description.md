## 介绍

在接下来的几个章节中，我们将用本章节介绍的vertex buffer替换在vertex shader中硬编码的顶点数据。
首先创建一个CPU的缓冲（利用 `memcpy` 直接拷贝顶点数据到该缓冲）。最后我们将会看到如何
拷贝顶点数据到高性能的内存 `staging buffer` 当中（以提高数据传输效率）。

## 顶点着色器

首先，在着色器代码去掉含有的顶点数据。顶点着色器使用 `in` 关键字从顶点缓冲中获取输入数据。


```glsl
#version 450
#extension GL_ARB_separate_shader_objects : enable

layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;

layout(location = 0) out vec3 fragColor;

void main() {
    gl_Position = vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```

 `inPosition` 和 `inColor` 变量代表*顶点属性*。它们代表了顶点缓冲中的每个顶点数据，
就跟我们使用数组定义的顶点数据是一样的。修改顶点着色器后重新编译顶点着色器，保证没有出现问题！

`layout(location = x)` 用于指定变量在顶点数据中的索引。
重要的是要知道某些特殊类型（例如 `dvec3` 64位向量）占用了多个索引位置。这意味着之后的索引并不是依次递增：


```glsl
layout(location = 0) in dvec3 inPosition;
layout(location = 2) in vec3 inColor;
```

你可以通过[OpenGL wiki](https://www.khronos.org/opengl/wiki/Layout_Qualifier_(GLSL)) 查看更多有关布局修饰符的信息。

```glsl
struct OutData
{
  vec3 data1;
  dvec4 data2;
  float val[3];
};

layout(location = 0) out vec3 vals[4];    // 消耗 4 个位置
layout(location = 4) out OutData myOut;   // 消耗 6 个位置. dvec4 uses 2, and `val[3]` uses 3 total
layout(location = 10) out vec2 texCoord;  // 消耗 1 个位置
```
## 顶点数据

我们将顶点数据从着色器代码移动到C++代码。首先介绍GLM库，该库为我们提供了线性代数库，
例如向量和矩阵。我们将使用这些类型来指定位置和颜色向量。

```c++
#include <glm/glm.hpp>
```

创建一个 `Vertex` 具有两个属性的新结构体类型，我们将在其中的顶点着色器中使用它们：

```c++
struct Vertex {
    glm::vec2 pos;
    glm::vec3 color;
};
```

GLM很方便地为我们提供了与着色器语言中使用的向量类型完全兼容的C++类型。


```c++
const std::vector<Vertex> vertices = {
    {{0.0f, -0.5f}, {1.0f, 0.0f, 0.0f}},
    {{0.5f, 0.5f}, {0.0f, 1.0f, 0.0f}},
    {{-0.5f, 0.5f}, {0.0f, 0.0f, 1.0f}}
};
```
现在，我们使用 `Vertex` 结构体数组来定义我们的顶点数据。这次不同于之前章节在顶点着色器定义的那样
将顶点位置和顶点颜色数据定义在不同的数组中，我们将它们定义在了同一个结构体数组中。这称为交叉顶点属性( interleaving vertex attributes )。


## 绑定描述

下一步是告诉Vulkan，一旦 `Vertex` 结构体（CPU缓冲）传递给GPU显存中，如何将其传递给顶点着色器使用。传达此信息需要两种类型的结构。

第一个结构体是[VkVertexInputBindingDescription](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkVertexInputBindingDescription.html) ，
我们将向 `Vertex` 添加一个静态成员函数，在里面返回 `Vertex` 结构体的顶点数据存放方式。


```c++
struct Vertex {
    glm::vec2 pos;
    glm::vec3 color;

    static VkVertexInputBindingDescription getBindingDescription() {
        VkVertexInputBindingDescription bindingDescription{};

        return bindingDescription;
    }
};
```

`顶点绑定`描述了整个顶点从内存中加载数据的速率。它指定数据块之间的字节数，以及是否在每个顶点之后或在
每个实例之后移至下一个数据块。

```c++
VkVertexInputBindingDescription bindingDescription{};
bindingDescription.binding = 0;
bindingDescription.stride = sizeof(Vertex);
bindingDescription.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;
```

我们所有的每个顶点数据都打包在一个数组中，因此我们只会使用一个 `binding` 。 `binding` 参数指定绑定数组中绑定的索引。
`stride` 参数指定顶点结构所占的字节大小（`sizeof(Vertex)`）， `inputRate` 参数可以具有以下值之一：
* `VK_VERTEX_INPUT_RATE_VERTEX`: 逐顶点处理
* `VK_VERTEX_INPUT_RATE_INSTANCE`: 逐实例处理

此处我们不使用`instanced`渲染。

## 属性描述

第二个结构体属性描述 [VkVertexInputAttributeDescription](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkVertexInputAttributeDescription.html) 。
我们将添加另一个辅助函数 `getAttributeDescriptions` 来填充 VkVertexInputAttributeDescription 结构体。


```c++
#include <array>

...

static std::array<VkVertexInputAttributeDescription, 2> getAttributeDescriptions() {
    std::array<VkVertexInputAttributeDescription, 2> attributeDescriptions{};

    return attributeDescriptions;
}
```

如函数原型所示，将有两个这样的结构。`属性描述结构`描述了如何从源自`绑定描述`的大量顶点数据中提取顶点属性。
此处我们有两个顶点属性，位置和颜色，因此我们需要两个`属性描述结构`。


```c++
attributeDescriptions[0].binding = 0;
attributeDescriptions[0].location = 0;
attributeDescriptions[0].format = VK_FORMAT_R32G32_SFLOAT;
attributeDescriptions[0].offset = offsetof(Vertex, pos);
```

`binding` 参数告诉Vulkan每个顶点数据来自哪个绑定。 `location` 参数指定顶点着色器从哪里读。
位置顶点属性对应顶点着色器中的输入`0`，`layout(location = 0) in dvec3 inPosition;`该位置属性具有两个32位浮点分量（`dvec3`）。

`format` 参数描述该属性的数据类型。这些格式是使用与颜色格式相同的枚举指定的。以下着色器类型和格式通常一起使用：

* `float`: `VK_FORMAT_R32_SFLOAT`
* `vec2`: `VK_FORMAT_R32G32_SFLOAT`
* `vec3`: `VK_FORMAT_R32G32B32_SFLOAT`
* `vec4`: `VK_FORMAT_R32G32B32A32_SFLOAT`

颜色类型（`SFLOAT`，`UINT`，`SINT`）和比特宽度也应与着色器输入的类型匹配。请参阅以下示例：

* `ivec2`: `VK_FORMAT_R32G32_SINT`, 由32位有符号整数组成的2维向量
* `uvec4`: `VK_FORMAT_R32G32B32A32_UINT`, 一个由32位有符号整数组成的4维向量
* `double`: `VK_FORMAT_R64_SFLOAT`, 双精度 (64位) 浮点数

`format` 参数隐式定义属性数据的字节大小，`offset` 参数指定偏移量。
当前`offset`值为位置属性（`pos`）与该结构的开头之间的字节偏移量，使用`offsetof`宏自动计算的。

```c++
attributeDescriptions[1].binding = 0;
attributeDescriptions[1].location = 1;
attributeDescriptions[1].format = VK_FORMAT_R32G32B32_SFLOAT;
attributeDescriptions[1].offset = offsetof(Vertex, color);
```
颜色属性的描述方式几乎相同。

## 管线顶点输入

现在，我们需要建立图形管线来接收顶点数据，并在`createGraphicsPipeline`中通过引用该结构体。
用上面介绍的两种描述结构体对 `vertexInputInfo` 结构体的成员变量进行赋值：
* bindingDescription 绑定描述
* attributeDescriptions 属性描述

```c++
auto bindingDescription = Vertex::getBindingDescription();
auto attributeDescriptions = Vertex::getAttributeDescriptions();

vertexInputInfo.vertexBindingDescriptionCount = 1;
vertexInputInfo.vertexAttributeDescriptionCount = static_cast<uint32_t>(attributeDescriptions.size());
vertexInputInfo.pVertexBindingDescriptions = &bindingDescription;
vertexInputInfo.pVertexAttributeDescriptions = attributeDescriptions.data();
```

现在，管线已准备好接收`vertices`结构体的顶点数据，并将其传递给我们的顶点着色器。
如果现在在启用验证层的情况下运行该程序，您会看到它报出没有绑定到该绑定的顶点缓冲区的错误。
下一步是创建一个顶点缓冲区并将顶点数据传到该缓冲区，以便GPU能够访问它。

[C++ 代码](/code/17_vertex_input.cpp) /
[顶点着色器](/code/17_shader_vertexbuffer.vert) /
[片段着色器](/code/17_shader_vertexbuffer.frag)
