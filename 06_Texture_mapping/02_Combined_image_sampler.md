## Introduction

We looked at descriptors for the first time in the uniform buffers part of the
tutorial. In this chapter we will look at a new type of descriptor: *combined
image sampler*. This descriptor makes it possible for shaders to access an image
resource through a sampler object like the one we created in the previous
chapter.

우리는 튜토리얼의 uniform buffers 파트에서 처음으로 descriptors를 살펴 보았습니다. 
이 장에서는 새로운 유형의 descriptors를 살펴볼 것입니다 : *combined image sampler*. 
이 기술자는 이전 장에서 생성 한 것과 같은 샘플러 객체를 통해 셰이더가 이미지 리소스에 액세스 할 수있게 합니다.

We'll start by modifying the descriptor layout, descriptor pool and descriptor
set to include such a combined image sampler descriptor. After that, we're going
to add texture coordinates to `Vertex` and modify the fragment shader to read
colors from the texture instead of just interpolating the vertex colors.

우리는 먼저 combined image sampler descriptor를 포함하도록 descriptor layout, descriptor pool 및 descriptor를 설정합니다. 
그 후, 텍스처 좌표를 `Vertex`에 추가하고 fragment shader를 수정하여 vertex 색상을 보간하는 대신 텍스처에서 색상을 읽도록 할 것입니다.

## Updating the descriptors

Browse to the `createDescriptorSetLayout` function and add a
`VkDescriptorSetLayoutBinding` for a combined image sampler descriptor. We'll
simply put it in the binding after the uniform buffer:

`createDescriptorSetLayout` 함수를 찾아서 결합 된 이미지 샘플러 기술자를 위한 `VkDescriptorSetLayoutBinding`을 추가하십시오. 
uniform buffer 다음에 바인딩에 넣을 것입니다.

```c++
VkDescriptorSetLayoutBinding samplerLayoutBinding = {};
samplerLayoutBinding.binding = 1;
samplerLayoutBinding.descriptorCount = 1;
samplerLayoutBinding.descriptorType = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
samplerLayoutBinding.pImmutableSamplers = nullptr;
samplerLayoutBinding.stageFlags = VK_SHADER_STAGE_FRAGMENT_BIT;

std::array<VkDescriptorSetLayoutBinding, 2> bindings = {uboLayoutBinding, samplerLayoutBinding};
VkDescriptorSetLayoutCreateInfo layoutInfo = {};
layoutInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO;
layoutInfo.bindingCount = static_cast<uint32_t>(bindings.size());
layoutInfo.pBindings = bindings.data();
```

Make sure to set the `stageFlags` to indicate that we intend to use the combined
image sampler descriptor in the fragment shader. That's where the color of the
fragment is going to be determined. It is possible to use texture sampling in
the vertex shader, for example to dynamically deform a grid of vertices by a
[heightmap](https://en.wikipedia.org/wiki/Heightmap).

프레그먼트 쉐이더에서 combined image sampler descriptor를 사용하고자 함을 나타 내기 위해 `stageFlags`를 설정하십시오. 
그곳이 프레그먼트의 색상이 결정될 곳입니다. 버텍스 쉐이더에서 텍스처 샘플링을 사용할 수도 있습니다.
예를 들어, [heightmap](https://en.wikipedia.org/wiki/Heightmap)을 통해 정점 그리드를 동적으로 변형할 수 있습니다.

If you would run the application with validation layers now, then you'll see
that it complains that the descriptor pool cannot allocate descriptor sets with
this layout, because it doesn't have any combined image sampler descriptors. Go
to the `createDescriptorPool` function and modify it to include a
`VkDescriptorPoolSize` for this descriptor:

이제 유효성 검사 레이어로 응용 프로그램을 실행하면 결합 된 이미지 샘플러 설명자가 없으므로 
descriptor pool에서 이 레이아웃으로 descriptor sets를 할당 할 수 없다는 메시지가 표시됩니다. 
`createDescriptorPool` 함수로 가서이 descriptor를 위한 `VkDescriptorPoolSize`를 포함하도록 수정하십시오 :

```c++
std::array<VkDescriptorPoolSize, 2> poolSizes = {};
poolSizes[0].type = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
poolSizes[0].descriptorCount = static_cast<uint32_t>(swapChainImages.size());
poolSizes[1].type = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
poolSizes[1].descriptorCount = static_cast<uint32_t>(swapChainImages.size());

VkDescriptorPoolCreateInfo poolInfo = {};
poolInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO;
poolInfo.poolSizeCount = static_cast<uint32_t>(poolSizes.size());
poolInfo.pPoolSizes = poolSizes.data();
poolInfo.maxSets = static_cast<uint32_t>(swapChainImages.size());
```

The final step is to bind the actual image and sampler resources to the
descriptors in the descriptor set. Go to the `createDescriptorSets` function.

마지막 단계는 실제 이미지와 샘플러 자원을 디스크립터 세트의 디스크립터에 바인드하는 것입니다.
`createDescriptorSets` 함수로 갑니다.

```c++
for (size_t i = 0; i < swapChainImages.size(); i++) {
    VkDescriptorBufferInfo bufferInfo = {};
    bufferInfo.buffer = uniformBuffers[i];
    bufferInfo.offset = 0;
    bufferInfo.range = sizeof(UniformBufferObject);

    VkDescriptorImageInfo imageInfo = {};
    imageInfo.imageLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
    imageInfo.imageView = textureImageView;
    imageInfo.sampler = textureSampler;

    ...
}
```

The resources for a combined image sampler structure must be specified in a
`VkDescriptorImageInfo` struct, just like the buffer resource for a uniform
buffer descriptor is specified in a `VkDescriptorBufferInfo` struct. This is
where the objects from the previous chapter come together.

`VkDescriptorBufferInfo` 구조체에 지정된 uniformbuffer 디스크립터의 버퍼 자원 처럼 
`VkDescriptorImageInfo` 구조체에 결합된 이미지 샘플러 구조체를 위한 자원을 지정합니다. 
이것은 이전 장에서 구현한 객체들이 모이는 곳입니다.

```c++
std::array<VkWriteDescriptorSet, 2> descriptorWrites = {};

descriptorWrites[0].sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
descriptorWrites[0].dstSet = descriptorSets[i];
descriptorWrites[0].dstBinding = 0;
descriptorWrites[0].dstArrayElement = 0;
descriptorWrites[0].descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
descriptorWrites[0].descriptorCount = 1;
descriptorWrites[0].pBufferInfo = &bufferInfo;

descriptorWrites[1].sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
descriptorWrites[1].dstSet = descriptorSets[i];
descriptorWrites[1].dstBinding = 1;
descriptorWrites[1].dstArrayElement = 0;
descriptorWrites[1].descriptorType = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
descriptorWrites[1].descriptorCount = 1;
descriptorWrites[1].pImageInfo = &imageInfo;

vkUpdateDescriptorSets(device, static_cast<uint32_t>(descriptorWrites.size()), descriptorWrites.data(), 0, nullptr);
```

The descriptors must be updated with this image info, just like the buffer. This
time we're using the `pImageInfo` array instead of `pBufferInfo`. The descriptors
are now ready to be used by the shaders!

디스크립터는 버퍼와 마찬가지로이 이미지 정보로 업데이트되어야 합니다. 
이번에는`pBufferInfo` 대신 `pImageInfo` 배열을 사용하고 있습니다. 
이제 셰이더에서 설명자를 사용할 준비가되었습니다!

## Texture coordinates (텍스처 좌표)

There is one important ingredient for texture mapping that is still missing, and
that's the actual coordinates for each vertex. The coordinates determine how the
image is actually mapped to the geometry.

아직 누락 된 텍스처 매핑을위한 중요한 요소가 있으며, 그것은 각 정점의 실제 좌표입니다. 
좌표는 이미지가 실제로 기하학에 매핑되는 방법을 결정합니다.

```c++
struct Vertex {
    glm::vec2 pos;
    glm::vec3 color;
    glm::vec2 texCoord;

    static VkVertexInputBindingDescription getBindingDescription() {
        VkVertexInputBindingDescription bindingDescription = {};
        bindingDescription.binding = 0;
        bindingDescription.stride = sizeof(Vertex);
        bindingDescription.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;

        return bindingDescription;
    }

    static std::array<VkVertexInputAttributeDescription, 3> getAttributeDescriptions() {
        std::array<VkVertexInputAttributeDescription, 3> attributeDescriptions = {};

        attributeDescriptions[0].binding = 0;
        attributeDescriptions[0].location = 0;
        attributeDescriptions[0].format = VK_FORMAT_R32G32_SFLOAT;
        attributeDescriptions[0].offset = offsetof(Vertex, pos);

        attributeDescriptions[1].binding = 0;
        attributeDescriptions[1].location = 1;
        attributeDescriptions[1].format = VK_FORMAT_R32G32B32_SFLOAT;
        attributeDescriptions[1].offset = offsetof(Vertex, color);

        attributeDescriptions[2].binding = 0;
        attributeDescriptions[2].location = 2;
        attributeDescriptions[2].format = VK_FORMAT_R32G32_SFLOAT;
        attributeDescriptions[2].offset = offsetof(Vertex, texCoord);

        return attributeDescriptions;
    }
};
```

Modify the `Vertex` struct to include a `vec2` for texture coordinates. Make
sure to also add a `VkVertexInputAttributeDescription` so that we can use access
texture coordinates as input in the vertex shader. That is necessary to be able
to pass them to the fragment shader for interpolation across the surface of the
square.

`Vertex` 구조체를 수정하여 텍스쳐 좌표를위한 `vec2` 를 포함 시키십시오.
버텍스 쉐이더에서 입력 텍스쳐 좌표를 사용할 수 있도록 `VkVertexInputAttributeDescription`도 추가해야 합니다. 
사각형의 표면을 보간하기 위해 프레그먼트 쉐이더로 전달할 수 있어야 합니다.

```c++
const std::vector<Vertex> vertices = {
    {{-0.5f, -0.5f}, {1.0f, 0.0f, 0.0f}, {1.0f, 0.0f}},
    {{0.5f, -0.5f}, {0.0f, 1.0f, 0.0f}, {0.0f, 0.0f}},
    {{0.5f, 0.5f}, {0.0f, 0.0f, 1.0f}, {0.0f, 1.0f}},
    {{-0.5f, 0.5f}, {1.0f, 1.0f, 1.0f}, {1.0f, 1.0f}}
};
```

In this tutorial, I will simply fill the square with the texture by using
coordinates from `0, 0` in the top-left corner to `1, 1` in the bottom-right
corner. Feel free to experiment with different coordinates. Try using
coordinates below `0` or above `1` to see the addressing modes in action!

이 튜토리얼에서는 오른쪽 상단 모서리의 `0, 0` 좌표를, 오른쪽 하단 모서리의 `1, 1`좌표로 사용하여 텍스처를 사각형으로 채웁니다.
다른 좌표로 자유롭게 실험 해보십시오. 실행중인 주소 지정 모드를 보려면 `0` 또는 `1` 보다 아래의 좌표를 사용해 보십시오!

## Shaders

The final step is modifying the shaders to sample colors from the texture. We
first need to modify the vertex shader to pass through the texture coordinates
to the fragment shader:

마지막 단계는 셰이더를 텍스처에서 샘플 색상으로 수정하는 것입니다.
먼저 텍스쳐 좌표를 프레그먼트 셰이더에 전달하도록 버텍스 쉐이더를 수정해야합니다 :

```glsl
layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;
layout(location = 2) in vec2 inTexCoord;

layout(location = 0) out vec3 fragColor;
layout(location = 1) out vec2 fragTexCoord;

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
    fragTexCoord = inTexCoord;
}
```

Just like the per vertex colors, the `fragTexCoord` values will be smoothly
interpolated across the area of the square by the rasterizer. We can visualize
this by having the fragment shader output the texture coordinates as colors:

버텍스당 색상 처럼, `fragTexCoord` 값은 래스터 라이저에 의해 정사각형 영역 전체에 부드럽게 보간됩니다.
프래그먼트 셰이더가 텍스처 좌표를 색상으로 출력하도록 하면 시각화 할 수 있습니다.

```glsl
#version 450
#extension GL_ARB_separate_shader_objects : enable

layout(location = 0) in vec3 fragColor;
layout(location = 1) in vec2 fragTexCoord;

layout(location = 0) out vec4 outColor;

void main() {
    outColor = vec4(fragTexCoord, 0.0, 1.0);
}
```

You should see something like the image below. Don't forget to recompile the
shaders!

아래 그림과 같은 것이 보일 것입니다. 셰이더를 다시 컴파일하는 것을 잊지 마십시오!

![](/images/texcoord_visualization.png)

The green channel represents the horizontal coordinates and the red channel the
vertical coordinates. The black and yellow corners confirm that the texture
coordinates are correctly interpolated from `0, 0` to `1, 1` across the square.
Visualizing data using colors is the shader programming equivalent of `printf`
debugging, for lack of a better option!

녹색 채널은 수평 좌표를 나타내고 빨간색 채널은 수직 좌표를 나타냅니다. 
검은 색과 노란색 모서리는 텍스쳐 좌표가 사각형을 가로 질러 `0, 0`에서 `1, 1`까지 정확하게 보간되었음을 확인합니다. 
colors을 사용하여 데이터를 시각화하는 것은 `printf` 디버깅과 동일한 쉐이더 프로그래밍입니다.

A combined image sampler descriptor is represented in GLSL by a sampler uniform.
Add a reference to it in the fragment shader:

결합 된 이미지 샘플러 기술자는 샘플러 유니폼에 의해 GLSL로 표현됩니다. 
프레그먼트 쉐이더에서 참조를 추가합니다.

```glsl
layout(binding = 1) uniform sampler2D texSampler;
```

There are equivalent `sampler1D` and `sampler3D` types for other types of
images. Make sure to use the correct binding here.

다른 유형의 이미지에는 `sampler1D` 와 `sampler3D` 유형이 있습니다. 
올바른 바인딩을 여기에서 사용하십시오.

```glsl
void main() {
    outColor = texture(texSampler, fragTexCoord);
}
```

Textures are sampled using the built-in `texture` function. It takes a `sampler`
and coordinate as arguments. The sampler automatically takes care of the
filtering and transformations in the background. You should now see the texture
on the square when you run the application:

텍스처는 빌트인 `texture` 함수를 사용하여 샘플링됩니다. 인수로 `sampler`와 좌표를 취합니다. 
샘플러는 백그라운드에서 자동으로 필터링 및 변환을 처리합니다. 이제 응용 프로그램을 실행할 때 사각형에 텍스처가 표시됩니다.

![](/images/texture_on_square.png)

Try experimenting with the addressing modes by scaling the texture coordinates
to values higher than `1`. For example, the following fragment shader produces
the result in the image below when using `VK_SAMPLER_ADDRESS_MODE_REPEAT`:

`1`보다 높은 값으로 텍스처 좌표를 스케일링하여 어드레싱 모드를 실험 해보십시오.
예를 들어, 다음의 쉐이더 셰이더는 `VK_SAMPLER_ADDRESS_MODE_REPEAT`를 사용할 때 아래와 같은 결과를 생성합니다 :

```glsl
void main() {
    outColor = texture(texSampler, fragTexCoord * 2.0);
}
```

![](/images/texture_on_square_repeated.png)

You can also manipulate the texture colors using the vertex colors:

정점 색상을 사용하여 텍스처 색상을 조작 할 수도 있습니다.

```glsl
void main() {
    outColor = vec4(fragColor * texture(texSampler, fragTexCoord).rgb, 1.0);
}
```

I've separated the RGB and alpha channels here to not scale the alpha channel.

알파 채널의 크기를 조정하지 않기 위해 RGB 및 알파 채널을 분리했습니다.

![](/images/texture_on_square_colorized.png)

You now know how to access images in shaders! This is a very powerful technique
when combined with images that are also written to in framebuffers. You can use
these images as inputs to implement cool effects like post-processing and camera
displays within the 3D world.

셰이더에서 이미지에 액세스하는 방법을 알았습니다!
이것은 프레임 버퍼에서 작성된 이미지와 결합 할 때 매우 강력한 기술입니다.
이러한 이미지를 입력으로 사용하여 3D 세계에서의 사후 처리 및 카메라 디스플레이와 같은 멋진 효과를 구현할 수 있습니다.

[C++ code](/code/25_texture_mapping.cpp) /
[Vertex shader](/code/25_shader_textures.vert) /
[Fragment shader](/code/25_shader_textures.frag)
