## 介绍

Vulkan的缓冲是可以存储任意数据的可以被显卡设备读取的内存区域。可以用来存储顶点数据，也是接下来我们将要
做的，也可以有其他用途，在接下里的章节里我们将会逐步介绍。和我们之前看到的其他Vulkan对象不同，我们需要手动
分配它的内存。在之前我们也看到了Vulkan API几乎把所有的控制权交给程序员，其中，内存管理也是其中之一。

## 创建缓冲

创建一个叫做 `createVertexBuffer` 的函数，然后在 `initVulkan` 函数中
`createCommandBuffers` 函数调用之后调用它：

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
    createImageViews();
    createRenderPass();
    createGraphicsPipeline();
    createFramebuffers();
    createCommandPool();
    createVertexBuffer();
    createCommandBuffers();
    createSyncObjects();
}

...

void createVertexBuffer() {

}
```

和创建Vulkan其他对象一样，创建一个顶点缓冲需要填充`VkBufferCreateInfo`结构体。

```c++
VkBufferCreateInfo bufferInfo{};
bufferInfo.sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO;
bufferInfo.size = sizeof(vertices[0]) * vertices.size();
```

成员变量`size`，用于指定所要创建缓冲所占字节的大小。可以直接通过`sizeof`函数来计算顶点数组所占的字节大小。

```c++
bufferInfo.usage = VK_BUFFER_USAGE_VERTEX_BUFFER_BIT;
```

成员变量`usage`，用于指定缓冲数据的使用目的。可以用位运算组合指定多种用途。当前我们将缓冲指定为存储
顶点数据，在接下来的章节我们将看到其他使用目的。

```c++
bufferInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;
```

和交换链中的图像一样，缓冲可以被特定的队列族所拥有或者同时在多个族之间共享。当前缓冲只需要使用图形队列，
我们指定为独享模式。

成员变量`flags`用于配置缓冲的内存稀疏程度，我们将其设置为 `0`使用默认值。
填写完结构体信息，我们就可以调用`vkCreateBuffer`函数来完成缓冲
创建。我们定义一个类成员变量`vertexBuffer`来存储创建的缓冲的句柄。


```c++
VkBuffer vertexBuffer;

...

void createVertexBuffer() {
    VkBufferCreateInfo bufferInfo{};
    bufferInfo.sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO;
    bufferInfo.size = sizeof(vertices[0]) * vertices.size();
    bufferInfo.usage = VK_BUFFER_USAGE_VERTEX_BUFFER_BIT;
    bufferInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;

    if (vkCreateBuffer(device, &bufferInfo, nullptr, &vertexBuffer) != VK_SUCCESS) {
        throw std::runtime_error("failed to create vertex buffer!");
    }
}
```

缓冲对象在整个渲染程序中都是可被渲染指令们使用。它并不依赖交换链，也就是说交换链重建时，我们不需要重新创建
缓冲。因此，我们需要在程序结束时，手动销毁创建的缓冲对象：


```c++
void cleanup() {
    cleanupSwapChain();

    vkDestroyBuffer(device, vertexBuffer, nullptr);

    ...
}
```

## 内存需求

缓冲创建好了，但是还没有分配内存。在实际分配内存之前，首先得通过`vkGetBufferMemoryRequirements`函数获取它的内存需求。

```c++
VkMemoryRequirements memRequirements;
vkGetBufferMemoryRequirements(device, vertexBuffer, &memRequirements);
```

返回值`VkMemoryRequirements` 结构体有三个成员变量：

* `size`: 所需内存字节大小，有可能和 `bufferInfo.size` 不同。
* `alignment`: 缓冲在分配内存当中起始位置，取决于 `bufferInfo.usage` 和 `bufferInfo.flags` 。
* `memoryTypeBits`: 适合该缓冲使用内存的位域。

显卡可以提供不同的内存类型。不同的类型的内存在使用限制和性能表现上都会有所不同。需要我们结合缓冲的需求和程序
需求，从而找到最适合的内存类型。所以我们先创建一个 `findMemoryType` 函数来统一做这件事。

```c++
uint32_t findMemoryType(uint32_t typeFilter, VkMemoryPropertyFlags properties) {

}
```

首先我们需要调用 `vkGetPhysicalDeviceMemoryProperties` Vulkan函数获取硬件可用的内存类型。

```c++
VkPhysicalDeviceMemoryProperties memProperties;
vkGetPhysicalDeviceMemoryProperties(physicalDevice, &memProperties);
```

返回值 `VkPhysicalDeviceMemoryProperties` 结构体含有 `memoryTypes` 和 `memoryHeaps`这两个数组成员变量。
Memory heaps 是一种特殊内存资源，有点像专用显存和显存用尽之后属于主内存的交换空间的那部分内存。关于两者之间的巨大差异暂且不表。
我们先不关心内存的分配来源，只需要知道两者会引起性能差异即可。

我们先获取适合缓冲的内存类型，操作如下：

```c++
for (uint32_t i = 0; i < memProperties.memoryTypeCount; i++) {
    if (typeFilter & (1 << i)) {
        return i;
    }
}

throw std::runtime_error("failed to find suitable memory type!");
```

`typeFilter` 参数用于指定我们需要的内存类型的位域。我们只需要遍历可用内存类型数组，检测每个内存类型是否满足我们需要即可 (相应位域为
1)。

但是，我们不仅仅关心内存类型是否满足顶点缓冲的需要，还需要把顶点数据写入该内存中。 `memoryTypes` 数组
由 `VkMemoryType` 结构体组成位域。比如 `VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT` 使用此标志类型分配的内存可供主机访问。
主机可以使用映射函数（`vkMapMemory()`）来访问其内容。我们还需要使用 `VK_MEMORY_PROPERTY_HOST_COHERENT_BIT`
使能主机写入对设备可见或设备写入对主机可见。我们待会解释为什么映射内存的时候需要两者。

修改代码，检测是否满足我们需要的内存类型：

```c++
for (uint32_t i = 0; i < memProperties.memoryTypeCount; i++) {
    if ((typeFilter & (1 << i)) && (memProperties.memoryTypes[i].propertyFlags & properties) == properties) {
        return i;
    }
}
```

由于我们不只一个需要的内存属性，所以仅仅检测位与运算的结果是否非 0 是不够的，
还需要检测它是否与我们需要的属性的位域完全相同，否则我们抛出异常。

## 内存分配

We now have a way to determine the right memory type, so we can actually
allocate the memory by filling in the `VkMemoryAllocateInfo` structure.

确定好内存需求，接下来我们可以填写`VkMemoryAllocateInfo`结构体。

```c++
VkMemoryAllocateInfo allocInfo{};
allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
allocInfo.allocationSize = memRequirements.size;
allocInfo.memoryTypeIndex = findMemoryType(memRequirements.memoryTypeBits, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT);
```

内存分配只需要填写好需要的内存大小和内存类型，然后调用 `vkAllocateMemory` 函数分配内存即可：

```c++
VkBuffer vertexBuffer;
VkDeviceMemory vertexBufferMemory;

...

if (vkAllocateMemory(device, &allocInfo, nullptr, &vertexBufferMemory) != VK_SUCCESS) {
    throw std::runtime_error("failed to allocate vertex buffer memory!");
}
```

如果内存分配成功，我们需要调用 `vkBindBufferMemory` 函数绑定顶点缓冲到该内存：

```c++
vkBindBufferMemory(device, vertexBuffer, vertexBufferMemory, 0);
```

函数的前三个参数非常直白，第四个参数是偏移值。这里我们将内存用作顶点缓冲，可以将其设置为 0。
偏移值需要满足能够被 `memRequirements.alignment` 整除。

当然，和C++的动态内存分配一样，我们需要自行进行内存释放。当缓冲不再被使用时，需要将绑定的内存释放：

```c++
void cleanup() {
    cleanupSwapChain();

    vkDestroyBuffer(device, vertexBuffer, nullptr);
    vkFreeMemory(device, vertexBufferMemory, nullptr);
```

## 填充顶点缓冲

现在我们可以将顶点数据拷贝到缓冲中。我们需要使用 `vkMapMemory` 函数将缓冲关联的
内存映射到CPU可以访问的内存。

```c++
void* data;
vkMapMemory(device, vertexBufferMemory, 0, bufferInfo.size, 0, &data);
```

`vkMapMemory` 函数允许我们通过给定的内存偏移和内存大小访问特定的内存资源。偏移值和内存值我们设为 0 和
`bufferInfo.size` 。我们还可以使用 `VK_WHOLE_SIZE` 来映射整个申请的内存。函数的倒数第二个参数可以用来
指定一个特殊的 flag 参数，但是目前的版本还没有可以使用的 flag 参数，必须设为 0 。最后的参数用于返回内存映射
后的地址。

```c++
void* data;
vkMapMemory(device, vertexBufferMemory, 0, bufferInfo.size, 0, &data);
    memcpy(data, vertices.data(), (size_t) bufferInfo.size);
vkUnmapMemory(device, vertexBufferMemory);
```

现在可以用 `memcpy` 函数将顶点数据拷贝到已映射的内存，或者使用 `vkUnmapMemory` 函数取消映射。
然而，驱动程序可能并不会及时将数据拷贝到缓冲关联的内存中，这是因为处理器缓存机制的存在。
写入缓冲的数据对于映射的内存可能并不可见。目前有两个方法解决这个问题：

* 属性内存类型，保证内存可见的一致性 `VK_MEMORY_PROPERTY_HOST_COHERENT_BIT` 。
* 在写入数据到映射的内存后，调用 `vkFlushMappedMemoryRanges` 函数。在读取映射的内存数据前
  调用 `vkInvalidateMappedMemoryRanges` 函数。

We went for the first approach, which ensures that the mapped memory always
matches the contents of the allocated memory. Do keep in mind that this may lead
to slightly worse performance than explicit flushing, but we'll see why that
doesn't matter in the next chapter.

我们使用第一种方法，以保证映射的内存匹配分配的内存。记住此方法比第二种方法对性能稍稍有些影响，但是在接下来
的章节我们会看到这不重要。

## 绑定顶点缓冲

拓展 `createCommandBuffers` 函数，使用顶点缓冲进行渲染操作：

```c++
vkCmdBindPipeline(commandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, graphicsPipeline);

VkBuffer vertexBuffers[] = {vertexBuffer};
VkDeviceSize offsets[] = {0};
vkCmdBindVertexBuffers(commandBuffers[i], 0, 1, vertexBuffers, offsets);

vkCmdDraw(commandBuffers[i], static_cast<uint32_t>(vertices.size()), 1, 0, 0);
```

我们使用 `vkCmdBindVertexBuffers` 函数绑定顶点缓冲。第二个参数和第三个参数指定偏移值和我们要绑定的顶点
缓冲的数量。最后两个参数指定需要绑定的顶点缓冲数组和顶点数在顶点缓冲中的偏移值数组。我们还需要修改
 `vkCmdDraw` 函数，将顶点缓冲中的顶点数量替换之前硬编码的数字 3 。

现在运行程序，我们可以再次看到三角形：

![](/images/triangle.png)

通过改变顶点数组，修改颜色数据：

```c++
const std::vector<Vertex> vertices = {
    {{0.0f, -0.5f}, {1.0f, 1.0f, 1.0f}},
    {{0.5f, 0.5f}, {0.0f, 1.0f, 0.0f}},
    {{-0.5f, 0.5f}, {0.0f, 0.0f, 1.0f}}
};
```

再次运行程序，可以看到：

![](/images/triangle_white.png)

In the next chapter we'll look at a different way to copy vertex data to a
vertex buffer that results in better performance, but takes some more work.

下一章节，我们将会介绍另一种高效传输顶点数据的方法。

[C++ code](/code/18_vertex_buffer.cpp) /
[Vertex shader](/code/17_shader_vertexbuffer.vert) /
[Fragment shader](/code/17_shader_vertexbuffer.frag)
