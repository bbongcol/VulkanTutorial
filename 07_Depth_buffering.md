## Introduction

The geometry we've worked with so far is projected into 3D, but it's still
completely flat. In this chapter we're going to add a Z coordinate to the
position to prepare for 3D meshes. We'll use this third coordinate to place a
square over the current square to see a problem that arises when geometry is not
sorted by depth.

지금까지 작업 한 지오메트리는 3D로 투영되지만 여전히 완전히 평평합니다. 
이 장에서는 3D 메쉬를 준비하기 위해 위치에 Z 좌표를 추가합니다. 
이 세 번째 좌표를 사용하여 지형이 깊이로 정렬되지 않을 때 발생하는 문제를 확인하기 위해 현재 사각형 위에 정사각형을 배치합니다.

## 3D geometry

Change the `Vertex` struct to use a 3D vector for the position, and update the
`format` in the corresponding `VkVertexInputAttributeDescription`:

`Vertex` 구조체를 변경하여 위치에 3D 벡터를 사용하고 해당 
`VkVertexInputAttributeDescription` 에서 `format` 을 업데이트하십시오 :

```c++
struct Vertex {
    glm::vec3 pos;
    glm::vec3 color;
    glm::vec2 texCoord;

    ...

    static std::array<VkVertexInputAttributeDescription, 3> getAttributeDescriptions() {
        std::array<VkVertexInputAttributeDescription, 3> attributeDescriptions = {};

        attributeDescriptions[0].binding = 0;
        attributeDescriptions[0].location = 0;
        attributeDescriptions[0].format = VK_FORMAT_R32G32B32_SFLOAT;
        attributeDescriptions[0].offset = offsetof(Vertex, pos);

        ...
    }
};
```

Next, update the vertex shader to accept and transform 3D coordinates as input.
Don't forget to recompile it afterwards!

다음으로, 버텍스 쉐이더를 업데이트하여 3D 좌표를 입력으로 변환합니다. 나중에 다시 컴파일하는 것을 잊지 마십시오!

```c++
layout(location = 0) in vec3 inPosition;

...

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 1.0);
    fragColor = inColor;
    fragTexCoord = inTexCoord;
}
```

Lastly, update the `vertices` container to include Z coordinates:

마지막으로 `vertices` 컨테이너를 업데이트하여 Z 좌표를 포함시킵니다.

```c++
const std::vector<Vertex> vertices = {
    {{-0.5f, -0.5f, 0.0f}, {1.0f, 0.0f, 0.0f}, {0.0f, 0.0f}},
    {{0.5f, -0.5f, 0.0f}, {0.0f, 1.0f, 0.0f}, {1.0f, 0.0f}},
    {{0.5f, 0.5f, 0.0f}, {0.0f, 0.0f, 1.0f}, {1.0f, 1.0f}},
    {{-0.5f, 0.5f, 0.0f}, {1.0f, 1.0f, 1.0f}, {0.0f, 1.0f}}
};
```

If you run your application now, then you should see exactly the same result as
before. It's time to add some extra geometry to make the scene more interesting,
and to demonstrate the problem that we're going to tackle in this chapter.
Duplicate the vertices to define positions for a square right under the current
one like this:

응용 프로그램을 지금 실행하면 이전과 똑같은 결과가 나타납니다. 
이제 장면을보다 재미있게 만들고, 이 장에서 다루어야 할 문제를 보여주기 위해 여분의 지오메트리를 추가 할 차례입니다. 
정점을 복제하여 다음과 같이 현재 정사각형 바로 아래의 정사각형 위치를 정의합니다.

![](/images/extra_square.svg)

Use Z coordinates of `-0.5f` and add the appropriate indices for the extra
square:

`-0.5f` 의 Z 좌표를 사용하고 여분의 사각형에 적절한 인덱스를 추가하십시오 :

```c++
const std::vector<Vertex> vertices = {
    {{-0.5f, -0.5f, 0.0f}, {1.0f, 0.0f, 0.0f}, {0.0f, 0.0f}},
    {{0.5f, -0.5f, 0.0f}, {0.0f, 1.0f, 0.0f}, {1.0f, 0.0f}},
    {{0.5f, 0.5f, 0.0f}, {0.0f, 0.0f, 1.0f}, {1.0f, 1.0f}},
    {{-0.5f, 0.5f, 0.0f}, {1.0f, 1.0f, 1.0f}, {0.0f, 1.0f}},

    {{-0.5f, -0.5f, -0.5f}, {1.0f, 0.0f, 0.0f}, {0.0f, 0.0f}},
    {{0.5f, -0.5f, -0.5f}, {0.0f, 1.0f, 0.0f}, {1.0f, 0.0f}},
    {{0.5f, 0.5f, -0.5f}, {0.0f, 0.0f, 1.0f}, {1.0f, 1.0f}},
    {{-0.5f, 0.5f, -0.5f}, {1.0f, 1.0f, 1.0f}, {0.0f, 1.0f}}
};

const std::vector<uint16_t> indices = {
    0, 1, 2, 2, 3, 0,
    4, 5, 6, 6, 7, 4
};
```

Run your program now and you'll see something resembling an Escher illustration:

이제 프로그램을 실행하면 Escher 그림과 비슷한 것을 볼 수 있습니다.

![](/images/depth_issues.png)

The problem is that the fragments of the lower square are drawn over the
fragments of the upper square, simply because it comes later in the index array.
There are two ways to solve this:

문제는 인덱스 배열의 뒷부분에 있기 때문에 아래 사각형의 단편이 위쪽 사각형의 단편에 그려지는 것입니다. 
이 문제를 해결하는 방법에는 두 가지가 있습니다.

* Sort all of the draw calls by depth from back to front
* Use depth testing with a depth buffer

* 뒤로부터 전방까지 모든 그리기 호출을 깊이별로 정렬
* 깊이 버퍼로 깊이 테스트 사용

The first approach is commonly used for drawing transparent objects, because
order-independent transparency is a difficult challenge to solve. However, the
problem of ordering fragments by depth is much more commonly solved using a
*depth buffer*. A depth buffer is an additional attachment that stores the depth
for every position, just like the color attachment stores the color of every
position. Every time the rasterizer produces a fragment, the depth test will
check if the new fragment is closer than the previous one. If it isn't, then the
new fragment is discarded. A fragment that passes the depth test writes its own
depth to the depth buffer. It is possible to manipulate this value from the
fragment shader, just like you can manipulate the color output.

첫 번째 방법은 투명 오브젝트 그리기에 일반적으로 사용됩니다. order-independent transparency은 해결하기가 어렵기 때문입니다. 
그러나 깊이별로 조각을 정렬하는 문제는 *depth buffer*를 사용하여 훨씬 더 일반적으로 해결됩니다.
깊이 버퍼는 색상 첨부가 모든 위치의 색상을 저장하는 것처럼 모든 위치의 깊이를 저장하는 추가 첨부 파일입니다.
래스터 라이저가 프래그먼트를 생성 할 때마다 깊이 테스트는 새 프래그먼트가 이전 프래그먼트보다 더 가까운 지 여부를 확인합니다. 
그렇지 않은 경우, 새로운 프래그먼트는 파기됩니다.
깊이 테스트를 통과 한 프레그먼트는 자체 깊이를 깊이 버퍼에 씁니다. 
색상 출력을 조작 할 수있는 것처럼 프래그먼트 셰이더에서 이 값을 조작 할 수 있습니다.

```c++
#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEPTH_ZERO_TO_ONE
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>
```

The perspective projection matrix generated by GLM will use the OpenGL depth
range of `-1.0` to `1.0` by default. We need to configure it to use the Vulkan
range of `0.0` to `1.0` using the `GLM_FORCE_DEPTH_ZERO_TO_ONE` definition.

GLM에 의해 생성 된 투시 투영 행렬은 기본적으로 OpenGL 깊이 범위가 `-1.0`에서 `1.0`까지 사용됩니다. 
GLM_FORCE_DEPTH_ZERO_TO_ONE 정의를 사용하여 Vulkan 범위를 `0.0`에서 `1.0`으로 사용하도록 구성해야합니다.

## Depth image and view

A depth attachment is based on an image, just like the color attachment. The
difference is that the swap chain will not automatically create depth images for us. We only need a single depth image, because only one draw operation is
running at once. The depth image will again require the trifecta of resources:
image, memory and image view.

Depth attachment은 색상 첨부와 마찬가지로 이미지를 기반으로 합니다.
차이점은 스왑 체인이 자동으로 깊이 이미지를 생성하지 않는다는 것입니다.
한 번에 하나의 그리기 작업만 실행되기 때문에 단일 깊이 이미지 만 필요합니다. 
심도 이미지를 위해 image, memory 및 image view 3가지 리소스가 필요합니다.

```c++
VkImage depthImage;
VkDeviceMemory depthImageMemory;
VkImageView depthImageView;
```

Create a new function `createDepthResources` to set up these resources:

이 리소스를 설정하기 위해 `createreaDepthResources` 라는 새로운 함수를 만듭니다 :

```c++
void initVulkan() {
    ...
    createCommandPool();
    createDepthResources();
    createTextureImage();
    ...
}

...

void createDepthResources() {

}
```

Creating a depth image is fairly straightforward. It should have the same
resolution as the color attachment, defined by the swap chain extent, an image
usage appropriate for a depth attachment, optimal tiling and device local
memory. The only question is: what is the right format for a depth image? The
format must contain a depth component, indicated by `_D??_` in the `VK_FORMAT_`.

깊이 이미지를 만드는 것은 매우 간단합니다. 스왑 체인 범위, 깊이 첨부에 적합한 이미지 사용법, 
최적의 타일링 및 장치 로컬 메모리로 정의되는 색상 첨부와 동일한 해상도를 가져야합니다.
유일한 질문은 깊이 이미지의 올바른 형식은 무엇인지 입니다.
포맷은 `VK_FORMAT_` 에 `_D??_` 로 표시되는 깊이 컴포넌트를 포함해야합니다.

Unlike the texture image, we don't necessarily need a specific format, because
we won't be directly accessing the texels from the program. It just needs to
have a reasonable accuracy, at least 24 bits is common in real-world
applications. There are several formats that fit this requirement:

텍스쳐 이미지와 달리 우리는 프로그램에서 텍셀에 직접 접근하지 않기 때문에 반드시 특정 포맷을 필요로 하지는 않습니다.
합리적인 정확성만 있으면 됩니다. 실제 응용 프로그램에서는 적어도 24 비트가 일반적 입니다. 이 요구 사항에 적합한 여러 형식이 있습니다.

* `VK_FORMAT_D32_SFLOAT`: 32-bit float for depth
* `VK_FORMAT_D32_SFLOAT_S8_UINT`: 32-bit signed float for depth and 8 bit
stencil component
* `VK_FORMAT_D24_UNORM_S8_UINT`: 24-bit float for depth and 8 bit stencil
component

* `VK_FORMAT_D32_SFLOAT` : 깊이를 위한 32 비트 float
* `VK_FORMAT_D32_SFLOAT_S8_UINT` : 깊이를 위한 32 비트 부호 있는 float 와 8 비트 스텐실 구성 요소
* `VK_FORMAT_D24_UNORM_S8_UINT` : 깊이를 위한 24 비트 float 및 8 비트 스텐실 구성 요소

The stencil component is used for [stencil tests](https://en.wikipedia.org/wiki/Stencil_buffer),
which is an additional test that can be combined with depth testing. We'll look
at this in a future chapter.

스텐실 구성 요소는 [stencil tests](https://en.wikipedia.org/wiki/Stencil_buffer)에 사용됩니다. 
이 테스트는 깊이 테스트와 결합 할  수있는 추가 테스트입니다. 우리는 장래의 장에서 이것을 볼 것입니다.

We could simply go for the `VK_FORMAT_D32_SFLOAT` format, because support for it
is extremely common (see the hardware database), but it's nice to add some extra
flexibility to our application where possible. We're going to write a function
`findSupportedFormat` that takes a list of candidate formats in order from most
desirable to least desirable, and checks which is the first one that is
supported:

`VK_FORMAT_D32_SFLOAT` 포맷은 지원이 매우 일반적이기 때문에 (하드웨어 데이터베이스 참조), 
가능한 경우 우리의 어플리케이션에 유연성을 추가하는 것이 좋습니다.
우리는 가장 바람직한 것에서 부터 가장 바람직한 것에 이르기까지 후보 형식의 목록을 취하는 
함수 `findSupportedFormat` 을 작성하고 지원되는 첫 번째 검사를 검사 할 것입니다.

```c++
VkFormat findSupportedFormat(const std::vector<VkFormat>& candidates, VkImageTiling tiling, VkFormatFeatureFlags features) {

}
```

The support of a format depends on the tiling mode and usage, so we must also
include these as parameters. The support of a format can be queried using
the `vkGetPhysicalDeviceFormatProperties` function:

format의 지원은 타일링 모드 및 사용법에 따라 다르므로 이러한 매개 변수도 포함해야합니다.
format의 지원은 `vkGetPhysicalDeviceFormatProperties` 함수를 사용하여 질의 할 수 있습니다 :

```c++
for (VkFormat format : candidates) {
    VkFormatProperties props;
    vkGetPhysicalDeviceFormatProperties(physicalDevice, format, &props);
}
```

The `VkFormatProperties` struct contains three fields:

`VkFormatProperties` 구조체는 세 개의 필드를 포함합니다 :

* `linearTilingFeatures`: Use cases that are supported with linear tiling
* `optimalTilingFeatures`: Use cases that are supported with optimal tiling
* `bufferFeatures`: Use cases that are supported for buffers

* `linearTilingFeatures` : 선형 타일링으로 지원되는 경우 사용
* `optimalTilingFeatures` : 최적의 타일링으로 지원되는 경우 사용
* `bufferFeatures` : 버퍼에 대해 지원되는 경우 사용

Only the first two are relevant here, and the one we check depends on the
`tiling` parameter of the function:

여기서는 처음 두 개만 관련이 있으며, 우리가 확인하는 것은 함수의 `tiling` 매개 변수에 달려 있습니다.

```c++
if (tiling == VK_IMAGE_TILING_LINEAR && (props.linearTilingFeatures & features) == features) {
    return format;
} else if (tiling == VK_IMAGE_TILING_OPTIMAL && (props.optimalTilingFeatures & features) == features) {
    return format;
}
```

If none of the candidate formats support the desired usage, then we can either
return a special value or simply throw an exception:

후보 형식 중 어느 것도 원하는 사용법을 지원하지 않으면 특수 값을 반환하거나 단순히 예외를 throw 할 수 있습니다.

```c++
VkFormat findSupportedFormat(const std::vector<VkFormat>& candidates, VkImageTiling tiling, VkFormatFeatureFlags features) {
    for (VkFormat format : candidates) {
        VkFormatProperties props;
        vkGetPhysicalDeviceFormatProperties(physicalDevice, format, &props);

        if (tiling == VK_IMAGE_TILING_LINEAR && (props.linearTilingFeatures & features) == features) {
            return format;
        } else if (tiling == VK_IMAGE_TILING_OPTIMAL && (props.optimalTilingFeatures & features) == features) {
            return format;
        }
    }

    throw std::runtime_error("failed to find supported format!");
}
```

We'll use this function now to create a `findDepthFormat` helper function to
select a format with a depth component that supports usage as depth attachment:

이 함수를 사용하여 depth attachment 사용을 지원하는 depth component가 있는 
형식을 선택하기 위해 `findDepthFormat` 헬퍼 함수를 만듭니다.

```c++
VkFormat findDepthFormat() {
    return findSupportedFormat(
        {VK_FORMAT_D32_SFLOAT, VK_FORMAT_D32_SFLOAT_S8_UINT, VK_FORMAT_D24_UNORM_S8_UINT},
        VK_IMAGE_TILING_OPTIMAL,
        VK_FORMAT_FEATURE_DEPTH_STENCIL_ATTACHMENT_BIT
    );
}
```

Make sure to use the `VK_FORMAT_FEATURE_` flag instead of `VK_IMAGE_USAGE_` in
this case. All of these candidate formats contain a depth component, but the
latter two also contain a stencil component. We won't be using that yet, but we
do need to take that into account when performing layout transitions on images
with these formats. Add a simple helper function that tells us if the chosen
depth format contains a stencil component:

이 경우`VK_IMAGE_USAGE_` 대신`VK_FORMAT_FEATURE_` 플래그를 사용해야 합니다. 
이러한 모든 후보 형식에는 깊이 구성 요소가 포함되어 있지만 후자의 두 형식에는 스텐실 구성 요소가 포함되어 있습니다. 
아직 사용하지는 않겠지만 이러한 형식의 이미지에서 레이아웃 전환을 수행 할 때는 이를 고려해야 합니다. 
선택한 깊이 형식에 스텐실 구성 요소가 포함되어 있는지 알려주는 간단한 도우미 함수를 추가하십시오.

```c++
bool hasStencilComponent(VkFormat format) {
    return format == VK_FORMAT_D32_SFLOAT_S8_UINT || format == VK_FORMAT_D24_UNORM_S8_UINT;
}
```

Call the function to find a depth format from `createDepthResources`:

함수를 호출하여 `createDepthResources`에서 깊이 형식을 찾습니다.

```c++
VkFormat depthFormat = findDepthFormat();
```

We now have all the required information to invoke our `createImage` and
`createImageView` helper functions:

우리는 이제 `createImage` 와 `createImageView` 헬퍼 함수를 호출하는데 필요한 모든 정보를 가지고 있습니다 :

```c++
createImage(swapChainExtent.width, swapChainExtent.height, depthFormat, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, depthImage, depthImageMemory);
depthImageView = createImageView(depthImage, depthFormat);
```

However, the `createImageView` function currently assumes that the subresource
is always the `VK_IMAGE_ASPECT_COLOR_BIT`, so we will need to turn that field
into a parameter:

그러나,`createImageView` 함수는 현재 서브 소스가 항상
`VK_IMAGE_ASPECT_COLOR_BIT`라고 가정 하므로, 이 필드를 매개 변수로 변환해야합니다 :

```c++
VkImageView createImageView(VkImage image, VkFormat format, VkImageAspectFlags aspectFlags) {
    ...
    viewInfo.subresourceRange.aspectMask = aspectFlags;
    ...
}
```

Update all calls to this function to use the right aspect:

올바른 기능을 사용하려면이 함수에 대한 모든 호출을 업데이트하십시오.

```c++
swapChainImageViews[i] = createImageView(swapChainImages[i], swapChainImageFormat, VK_IMAGE_ASPECT_COLOR_BIT);
...
depthImageView = createImageView(depthImage, depthFormat, VK_IMAGE_ASPECT_DEPTH_BIT);
...
textureImageView = createImageView(textureImage, VK_FORMAT_R8G8B8A8_UNORM, VK_IMAGE_ASPECT_COLOR_BIT);
```

That's it for creating the depth image. We don't need to map it or copy another
image to it, because we're going to clear it at the start of the render pass
like the color attachment. However, it still needs to be transitioned to a
layout that is suitable for depth attachment usage. We could do this in the
render pass like the color attachment, but here I've chosen to use a pipeline
barrier because the transition only needs to happen once:

그것은 depth image를 만드는 것입니다. 컬러 첨부 처럼 렌더 패스가 시작될 때 이미지를 지우므로 
이미지를 매핑 하거나 다른 이미지를 복사 할 필요가 없습니다. 
그러나 depth attachment 사용에 적합한 레이아웃으로 전환해야 합니다. 
색상 첨부 처럼 렌더 패스에서 이 작업을 수행 할 수 있지만 
여기서는 전환이 한 번만 수행되어야 하기 때문에 파이프 라인 장벽을 사용하기로 했습니다.

```c++
transitionImageLayout(depthImage, depthFormat, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL);
```

The undefined layout can be used as initial layout, because there are no
existing depth image contents that matter. We need to update some of the logic
in `transitionImageLayout` to use the right subresource aspect:

중요한 depth image contents가 없기 때문에 정의되지 않은 레이아웃을 초기 레이아웃으로 사용할 수 있습니다. 
올바른 subresource aspect를 사용하기 위해 `transitionImageLayout`의 일부 로직을 업데이트해야합니다 :

```c++
if (newLayout == VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL) {
    barrier.subresourceRange.aspectMask = VK_IMAGE_ASPECT_DEPTH_BIT;

    if (hasStencilComponent(format)) {
        barrier.subresourceRange.aspectMask |= VK_IMAGE_ASPECT_STENCIL_BIT;
    }
} else {
    barrier.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
}
```

Although we're not using the stencil component, we do need to include it in the
layout transitions of the depth image.

스텐실 구성 요소를 사용하지는 않지만 깊이 이미지의 레이아웃 전환에 스텐실 구성 요소를 포함시켜야합니다.

Finally, add the correct access masks and pipeline stages:

마지막으로 올바른 액세스 마스크와 파이프 라인 단계를 추가하십시오.

```c++
if (oldLayout == VK_IMAGE_LAYOUT_UNDEFINED && newLayout == VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL) {
    barrier.srcAccessMask = 0;
    barrier.dstAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;

    sourceStage = VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT;
    destinationStage = VK_PIPELINE_STAGE_TRANSFER_BIT;
} else if (oldLayout == VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL && newLayout == VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL) {
    barrier.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
    barrier.dstAccessMask = VK_ACCESS_SHADER_READ_BIT;

    sourceStage = VK_PIPELINE_STAGE_TRANSFER_BIT;
    destinationStage = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
} else if (oldLayout == VK_IMAGE_LAYOUT_UNDEFINED && newLayout == VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL) {
    barrier.srcAccessMask = 0;
    barrier.dstAccessMask = VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_READ_BIT | VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT;

    sourceStage = VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT;
    destinationStage = VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT;
} else {
    throw std::invalid_argument("unsupported layout transition!");
}
```

The depth buffer will be read from to perform depth tests to see if a fragment
is visible, and will be written to when a new fragment is drawn. The reading
happens in the `VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT` stage and the
writing in the `VK_PIPELINE_STAGE_LATE_FRAGMENT_TESTS_BIT`. You should pick the
earliest pipeline stage that matches the specified operations, so that it is
ready for usage as depth attachment when it needs to be.

depth buffer는 depth tests를 수행하여 프래그먼트가 표시 되는지 확인하고 새 프래그먼트가 그려지는 시점에 기록됩니다.
읽기는 `VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT` 단계와 `VK_PIPELINE_STAGE_LATE_FRAGMENT_TESTS_BIT`에 작성됩니다.
지정된 작업과 일치하는 가장 빠른 파이프 라인 스테이지를 선택해야 하므로 필요할 때 depth attachment로 사용할 준비가 됩니다.

## Render pass

We're now going to modify `createRenderPass` to include a depth attachment.
First specify the `VkAttachmentDescription`:

우리는 이제 `createRenderPass`를 수정하여 깊이 첨부를 포함하려고 합니다. 
먼저 `VkAttachmentDescription`을 지정하십시오 :

```c++
VkAttachmentDescription depthAttachment = {};
depthAttachment.format = findDepthFormat();
depthAttachment.samples = VK_SAMPLE_COUNT_1_BIT;
depthAttachment.loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
depthAttachment.storeOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
depthAttachment.stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
depthAttachment.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
depthAttachment.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
depthAttachment.finalLayout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;
```

The `format` should be the same as the depth image itself. This time we don't
care about storing the depth data (`storeOp`), because it will not be used after
drawing has finished. This may allow the hardware to perform additional
optimizations. Just like the color buffer, we don't care about the previous
depth contents, so we can use `VK_IMAGE_LAYOUT_UNDEFINED` as `initialLayout`.

`format`은 깊이 이미지 자체와 동일해야 합니다. 이번에는 드로잉이 끝난 후에 사용되지 않기 때문에 
depth data (`storeOp`)를 저장하는 것에 신경 쓰지 않습니다.
이로 인해 하드웨어가 추가 최적화를 수행 할 수 있습니다.
색상 버퍼와 마찬가지로 이전의 깊이 내용을 신경 쓰지 않으므로 
`VK_IMAGE_LAYOUT_UNDEFINED` 을 `initialLayout` 으로 사용할 수 있습니다.

```c++
VkAttachmentReference depthAttachmentRef = {};
depthAttachmentRef.attachment = 1;
depthAttachmentRef.layout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;
```

Add a reference to the attachment for the first (and only) subpass:

첫 번째 (및 유일한) 서브 패스의 첨부 파일에 대한 참조를 추가하십시오.

```c++
VkSubpassDescription subpass = {};
subpass.pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS;
subpass.colorAttachmentCount = 1;
subpass.pColorAttachments = &colorAttachmentRef;
subpass.pDepthStencilAttachment = &depthAttachmentRef;
```

Unlike color attachments, a subpass can only use a single depth (+stencil)
attachment. It wouldn't really make any sense to do depth tests on multiple
buffers.

색상 첨부와 달리 하위 패스는 하나의 깊이 (+스텐실) 첨부 파일만 사용할 수 있습니다. 
여러 버퍼에서 깊이 테스트를 수행하는 것은 실제로 의미가 없습니다.

```c++
std::array<VkAttachmentDescription, 2> attachments = {colorAttachment, depthAttachment};
VkRenderPassCreateInfo renderPassInfo = {};
renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO;
renderPassInfo.attachmentCount = static_cast<uint32_t>(attachments.size());
renderPassInfo.pAttachments = attachments.data();
renderPassInfo.subpassCount = 1;
renderPassInfo.pSubpasses = &subpass;
renderPassInfo.dependencyCount = 1;
renderPassInfo.pDependencies = &dependency;
```

Finally, update the `VkRenderPassCreateInfo` struct to refer to both
attachments.

마지막으로, `VkRenderPassCreateInfo` 구조체를 업데이트하여 두 첨부 파일 모두를 참조하십시오.

## Framebuffer

The next step is to modify the framebuffer creation to bind the depth image to
the depth attachment. Go to `createFramebuffers` and specify the depth image
view as second attachment:

다음 단계는 프레임 버퍼 생성을 수정하여 depth image를 depth attachment에 바인딩하는 것입니다. 
`createFramebuffers`로 가서 깊이 이미지 뷰를 두 번째 첨부 파일로 지정하십시오 :

```c++
std::array<VkImageView, 2> attachments = {
    swapChainImageViews[i],
    depthImageView
};

VkFramebufferCreateInfo framebufferInfo = {};
framebufferInfo.sType = VK_STRUCTURE_TYPE_FRAMEBUFFER_CREATE_INFO;
framebufferInfo.renderPass = renderPass;
framebufferInfo.attachmentCount = static_cast<uint32_t>(attachments.size());
framebufferInfo.pAttachments = attachments.data();
framebufferInfo.width = swapChainExtent.width;
framebufferInfo.height = swapChainExtent.height;
framebufferInfo.layers = 1;
```

The color attachment differs for every swap chain image, but the same depth
image can be used by all of them because only a single subpass is running at the
same time due to our semaphores.

색상 첨부는 모든 스왑 체인 이미지마다 다르지만, 
우리의 세마포어 때문에 하나의 서브 패스만 동시에 실행되기 때문에 동일한 깊이 이미지를 사용할 수 있습니다.

You'll also need to move the call to `createFramebuffers` to make sure that it
is called after the depth image view has actually been created:

depth image view가 실제로 생성 된 후에 호출되도록 하려면 
`createFramebuffers`로 호출을 이동해야합니다.

```c++
void initVulkan() {
    ...
    createDepthResources();
    createFramebuffers();
    ...
}
```

## Clear values

Because we now have multiple attachments with `VK_ATTACHMENT_LOAD_OP_CLEAR`, we
also need to specify multiple clear values. Go to `createCommandBuffers` and
create an array of `VkClearValue` structs:

우리는 이제 `VK_ATTACHMENT_LOAD_OP_CLEAR`가 있는 첨부 파일을 
여러 개 가지고 있기 때문에 여러 개의 명확한 값을 지정해야합니다. 
`createCommandBuffers`로 가서 `VkClearValue` 구조체의 배열을 만듭니다 :

```c++
std::array<VkClearValue, 2> clearValues = {};
clearValues[0].color = {0.0f, 0.0f, 0.0f, 1.0f};
clearValues[1].depthStencil = {1.0f, 0};

renderPassInfo.clearValueCount = static_cast<uint32_t>(clearValues.size());
renderPassInfo.pClearValues = clearValues.data();
```

The range of depths in the depth buffer is `0.0` to `1.0` in Vulkan, where `1.0`
lies at the far view plane and `0.0` at the near view plane. The initial value
at each point in the depth buffer should be the furthest possible depth, which
is `1.0`.

깊이 버퍼의 깊이 범위는 Vulkan에서 `0.0`에서 `1.0`까지이며, 
여기서 1.0은 원거리 뷰 평면에 있고 0.0은 근거리 뷰 평면에 있습니다. 
깊이 버퍼의 각 지점에서의 초기 값은 가능한 가장 먼 깊이 인 `1.0`이어야 합니다.

## Depth and stencil state

The depth attachment is ready to be used now, but depth testing still needs to
be enabled in the graphics pipeline. It is configured through the
`VkPipelineDepthStencilStateCreateInfo` struct:

depth attachment는 이제 사용할 준비가되었지만 여전히 그래픽 파이프 라인에서 깊이 테스트를 활성화해야 합니다. 
그것은 `VkPipelineDepthStencilStateCreateInfo` 구조체를 통해 설정됩니다 :

```c++
VkPipelineDepthStencilStateCreateInfo depthStencil = {};
depthStencil.sType = VK_STRUCTURE_TYPE_PIPELINE_DEPTH_STENCIL_STATE_CREATE_INFO;
depthStencil.depthTestEnable = VK_TRUE;
depthStencil.depthWriteEnable = VK_TRUE;
```

The `depthTestEnable` field specifies if the depth of new fragments should be
compared to the depth buffer to see if they should be discarded. The
`depthWriteEnable` field specifies if the new depth of fragments that pass the
depth test should actually be written to the depth buffer. This is useful for
drawing transparent objects. They should be compared to the previously rendered
opaque objects, but not cause further away transparent objects to not be drawn.

`depthTestEnable` 필드는 새 프래그먼트의 depth를 depth buffer와 비교하여 버려야 하는지 확인하기 위해 지정합니다. 
`depthWriteEnable` 필드는 깊이 테스트를 통과 한 새로운 프래그먼트 depth가 depth buffer에 실제로 기록되어야 하는지를 지정합니다. 
투명 오브젝트를 그릴때 유용합니다. 그것들은 이전에 렌더링된 불투명 한 객체와 비교되어야 하지만 멀리 떨어진 투명 객체는 그려지지 않습니다.

```c++
depthStencil.depthCompareOp = VK_COMPARE_OP_LESS;
```

The `depthCompareOp` field specifies the comparison that is performed to keep or
discard fragments. We're sticking to the convention of lower depth = closer, so
the depth of new fragments should be *less*.

`depthCompareOp` 필드는 프래그먼트를 유지하거나 버리기 위해 수행되는 비교를 지정합니다. 
우리는 더 낮은 깊이의 관습을 고수하고 있습니다. 따라서 새로운 조각의 깊이는 더 *작아* 져야합니다.

```c++
depthStencil.depthBoundsTestEnable = VK_FALSE;
depthStencil.minDepthBounds = 0.0f; // Optional
depthStencil.maxDepthBounds = 1.0f; // Optional
```

The `depthBoundsTestEnable`, `minDepthBounds` and `maxDepthBounds` fields are
used for the optional depth bound test. Basically, this allows you to only keep
fragments that fall within the specified depth range. We won't be using this
functionality.

`depthBoundsTestEnable`, `minDepthBounds` 및 `maxDepthBounds` 필드는 선택적 깊이 바인딩 테스트에 사용됩니다. 
기본적으로 이것은 지정된 깊이 범위 내에있는 프래그먼트만 유지할 수 있습니다. 우리는 이 기능을 사용하지 않을 것입니다.

```c++
depthStencil.stencilTestEnable = VK_FALSE;
depthStencil.front = {}; // Optional
depthStencil.back = {}; // Optional
```

The last three fields configure stencil buffer operations, which we also won't
be using in this tutorial. If you want to use these operations, then you will
have to make sure that the format of the depth/stencil image contains a stencil
component.

마지막 세 필드는 스텐실 버퍼 작업을 구성합니다. 이 튜토리얼에서는 사용하지 않을 것입니다. 
이러한 작업을 사용하려면 깊이/스텐실 이미지의 형식에 스텐실 구성 요소가 포함되어 있는지 확인해야 합니다.

```c++
pipelineInfo.pDepthStencilState = &depthStencil;
```

Update the `VkGraphicsPipelineCreateInfo` struct to reference the depth stencil
state we just filled in. A depth stencil state must always be specified if the
render pass contains a depth stencil attachment.

방금 채운 깊이 스텐실 상태를 참조하도록 `VkGraphicsPipelineCreateInfo` 구조체를 업데이트하십시오. 
렌더패스에 깊이 스텐실 첨부가 포함되어 있는 경우에는 반드시 스텐실 상태를 지정해야합니다.

If you run your program now, then you should see that the fragments of the
geometry are now correctly ordered:

프로그램을 지금 실행하면 지오메트리의 조각이 올바르게 정렬되어 있어야합니다.

![](/images/depth_correct.png)

## Handling window resize

The resolution of the depth buffer should change when the window is resized to
match the new color attachment resolution. Extend the `recreateSwapChain`
function to recreate the depth resources in that case:

깊이 버퍼의 해상도는 창을 새 색상 부착 해상도와 일치 하도록 크기를 조정할 때 변경해야 합니다. 
`recreateSwapChain` 함수를 확장하여 깊이 자원을 다시 생성하십시오 :

```c++
void recreateSwapChain() {
    vkDeviceWaitIdle(device);

    cleanupSwapChain();

    createSwapChain();
    createImageViews();
    createRenderPass();
    createGraphicsPipeline();
    createDepthResources();
    createFramebuffers();
    createCommandBuffers();
}
```

The cleanup operations should happen in the swap chain cleanup function:

정리 작업은 스왑 체인 정리 기능에서 수행되어야합니다.

```c++
void cleanupSwapChain() {
    vkDestroyImageView(device, depthImageView, nullptr);
    vkDestroyImage(device, depthImage, nullptr);
    vkFreeMemory(device, depthImageMemory, nullptr);

    ...
}
```

Congratulations, your application is now finally ready to render arbitrary 3D
geometry and have it look right. We're going to try this out in the next chapter
by drawing a textured model!

축하합니다. 이제 응용 프로그램이 마침내 임의의 3D 지오메트리를 렌더링하고 
올바른 모양으로 렌더링 할 준비가되었습니다. 
텍스처 모델을 그리는 것으로 다음 장에서 이것을 시도 할 것입니다!

[C++ code](/code/26_depth_buffering.cpp) /
[Vertex shader](/code/26_shader_depth.vert) /
[Fragment shader](/code/26_shader_depth.frag)
