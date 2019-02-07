In this chapter we're going to create two more resources that are needed for the
graphics pipeline to sample an image. The first resource is one that we've
already seen before while working with the swap chain images, but the second one
is new - it relates to how the shader will read texels from the image.

이 장에서 우리는 두 가지 리소스를 추가로 만들 계획입니다.
그래픽 파이프 라인을 사용하여 이미지를 샘플링합니다. 첫 번째 리소스는 우리가 가진 리소스입니다.
스왑 체인 이미지로 작업하는 동안 이미 본 적이 있지만 두 번째 것은
새로운 것입니다 - 셰이더가 이미지에서 텍셀을 읽는 방법과 관련이 있습니다.

## Texture image view

We've seen before, with the swap chain images and the framebuffer, that images
are accessed through image views rather than directly. We will also need to
create such an image view for the texture image.

스왑 체인 이미지와 프레임 버퍼를 사용하여 이전에 이미지를 보았습니다.
직접적으로보다는 이미지 뷰를 통해 액세스됩니다. 우리는 또한 텍스처 이미지에 대한 이미지 뷰를 생성합니다.

Add a class member to hold a `VkImageView` for the texture image and create a
new function `createTextureImageView` where we'll create it:

텍스처 이미지를 위한 `VkImageView`를 저장 할 클래스 멤버를 추가하고
새로운 함수 `createTextureImageView`를 생성합니다.

```c++
VkImageView textureImageView;

...

void initVulkan() {
    ...
    createTextureImage();
    createTextureImageView();
    createVertexBuffer();
    ...
}

...

void createTextureImageView() {

}
```

The code for this function can be based directly on `createImageViews`. The only
two changes you have to make are the `format` and the `image`:

이 함수의 코드는`createImageViews`에 직접 기반 할 수 있습니다. 유일한
두 가지 변경 사항은`format`과 `image`입니다 :

```c++
VkImageViewCreateInfo viewInfo = {};
viewInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
viewInfo.image = textureImage;
viewInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
viewInfo.format = VK_FORMAT_R8G8B8A8_UNORM;
viewInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
viewInfo.subresourceRange.baseMipLevel = 0;
viewInfo.subresourceRange.levelCount = 1;
viewInfo.subresourceRange.baseArrayLayer = 0;
viewInfo.subresourceRange.layerCount = 1;
```

I've left out the explicit `viewInfo.components` initialization, because
`VK_COMPONENT_SWIZZLE_IDENTITY` is defined as `0` anyway. Finish creating the
image view by calling `vkCreateImageView`:

나는 명시적인 `viewInfo.components` 초기화를 생략했습니다.
`VK_COMPONENT_SWIZZLE_IDENTITY`는 어쨌든`0`으로 정의됩니다.
`vkCreateImageView`를 호출하여 image view 생성으 완료 합니다. :

```c++
if (vkCreateImageView(device, &viewInfo, nullptr, &textureImageView) != VK_SUCCESS) {
    throw std::runtime_error("failed to create texture image view!");
}
```

Because so much of the logic is duplicated from `createImageViews`, you may wish
to abstract it into a new `createImageView` function:

논리의 상당 부분이 `createImageViews`와 중복되기 때문에,
이것을 새로운 `createImageView` 함수로 추상화합니다 :

```c++
VkImageView createImageView(VkImage image, VkFormat format) {
    VkImageViewCreateInfo viewInfo = {};
    viewInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
    viewInfo.image = image;
    viewInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
    viewInfo.format = format;
    viewInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
    viewInfo.subresourceRange.baseMipLevel = 0;
    viewInfo.subresourceRange.levelCount = 1;
    viewInfo.subresourceRange.baseArrayLayer = 0;
    viewInfo.subresourceRange.layerCount = 1;

    VkImageView imageView;
    if (vkCreateImageView(device, &viewInfo, nullptr, &imageView) != VK_SUCCESS) {
        throw std::runtime_error("failed to create texture image view!");
    }

    return imageView;
}
```

The `createTextureImageView` function can now be simplified to:

`createTextureImageView` 함수는 이제 다음과 같이 단순화 될 수 있습니다 :

```c++
void createTextureImageView() {
    textureImageView = createImageView(textureImage, VK_FORMAT_R8G8B8A8_UNORM);
}
```

And `createImageViews` can be simplified to:

그리고 `createImageViews`는 다음과 같이 단순화 될 수 있습니다 :

```c++
void createImageViews() {
    swapChainImageViews.resize(swapChainImages.size());

    for (uint32_t i = 0; i < swapChainImages.size(); i++) {
        swapChainImageViews[i] = createImageView(swapChainImages[i], swapChainImageFormat);
    }
}
```

Make sure to destroy the image view at the end of the program, right before
destroying the image itself:

프로그램이 끝나면 바로 전 image 자체를 destroy하 전에 image view를 destroy 해야합니다:

```c++
void cleanup() {
    cleanupSwapChain();

    vkDestroyImageView(device, textureImageView, nullptr);

    vkDestroyImage(device, textureImage, nullptr);
    vkFreeMemory(device, textureImageMemory, nullptr);
```

## Samplers

It is possible for shaders to read texels directly from images, but that is not
very common when they are used as textures. Textures are usually accessed
through samplers, which will apply filtering and transformations to compute the
final color that is retrieved.

셰이더가 이미지에서 직접 텍셀을 읽을 수는 있지만, 텍스처로 사용할 때 그다지 일반적이지 않습니다.
텍스처는 일반적으로 샘플러를 통해 액세스됩니다. 샘플러는 필터링 및 변환을 적용하여 검색된 최종 색상을 계산합니다.

These filters are helpful to deal with problems like oversampling. Consider a
texture that is mapped to geometry with more fragments than texels. If you
simply took the closest texel for the texture coordinate in each fragment, then
you would get a result like the first image:

이러한 필터는 오버 샘플링과 같은 문제를 처리하는 데 유용합니다. 텍셀보다 더 많은 조각으로 도형에 매핑되는 텍스처를 고려해보십시오.
각 fragment에서 텍스쳐 좌표에 가장 근접한 텍셀을 단순히 가져왔다면, 첫 번째 이미지와 같은 결과를 얻을 것입니다 :

![](/images/texture_filtering.png)

If you combined the 4 closest texels through linear interpolation, then you
would get a smoother result like the one on the right. Of course your
application may have art style requirements that fit the left style more (think
Minecraft), but the right is preferred in conventional graphics applications. A
sampler object automatically applies this filtering for you when reading a color
from the texture.

직선 보간법을 통해 네 개의 가장 가까운 텍셀을 결합하면 오른쪽과 같은 부드러운 결과를 얻을 수 있습니다. 
물론 응용 프로그램에는 왼쪽 스타일에 더 맞는 아트 스타일 요구 사항이있을 수 있지만 (Minecraft 같은), 
그럴 권리는 기존 그래픽 응용 프로그램에서 선호됩니다. 
sampler 객체는 텍스처에서 색상을 읽을 때 자동으로이 필터링을 적용합니다.

Undersampling is the opposite problem, where you have more texels than
fragments. This will lead to artifacts when sampling high frequency patterns
like a checkerboard texture at a sharp angle:

Undersampling은 반대의 문제인데, fragments보다 텍셀이 많은 경우 입니다.
예리한 각도에서 바둑판 무늬 텍스처와 같은 고주파수 패턴을 샘플링 할 때 아티팩트가 생깁니다.

![](/images/anisotropic_filtering.png)

As shown in the left image, the texture turns into a blurry mess in the
distance. The solution to this is [anisotropic filtering](https://en.wikipedia.org/wiki/Anisotropic_filtering),
which can also be applied automatically by a sampler.

왼쪽 이미지에서 보듯이, 텍스처는 먼 거리에서 흐릿한 엉망으로 변합니다.
이에 대한 해결책은 [anisotropic filtering](https://en.wikipedia.org/wiki/Anisotropic_filtering)입니다.
샘플러에서 자동으로 적용 할 수도 있습니다.

Aside from these filters, a sampler can also take care of transformations. It
determines what happens when you try to read texels outside the image through
its *addressing mode*. The image below displays some of the possibilities:

이 필터들 외에도 샘플러는 변환을 처리 할 수 있습니다.
그것은 당신이 *addressing mode*를 통해 이미지 외부의 텍셀을 읽을 때 어떤 일이 일어나는지를 결정합니다.
아래 이미지는 몇 가지 가능성을 보여줍니다.

![](/images/texture_addressing.png)

We will now create a function `createTextureSampler` to set up such a sampler
object. We'll be using that sampler to read colors from the texture in the
shader later on.

그런 샘플러 객체를 설정하는 함수 `createTextureSampler`를 생성 할 것입니다.
나중에 샘플러를 사용하여 셰이더의 텍스처에서 색상을 읽을 것입니다.

```c++
void initVulkan() {
    ...
    createTextureImage();
    createTextureImageView();
    createTextureSampler();
    ...
}

...

void createTextureSampler() {

}
```

Samplers are configured through a `VkSamplerCreateInfo` structure, which
specifies all filters and transformations that it should apply.

샘플러는 `VkSamplerCreateInfo` 구조체를 통해 구성됩니다.
구조체는 적용해야 하는 모든 필터와 변환을 지정합니다.

```c++
VkSamplerCreateInfo samplerInfo = {};
samplerInfo.sType = VK_STRUCTURE_TYPE_SAMPLER_CREATE_INFO;
samplerInfo.magFilter = VK_FILTER_LINEAR;
samplerInfo.minFilter = VK_FILTER_LINEAR;
```

The `magFilter` and `minFilter` fields specify how to interpolate texels that
are magnified or minified. Magnification concerns the oversampling problem
describes above, and minification concerns undersampling. The choices are
`VK_FILTER_NEAREST` and `VK_FILTER_LINEAR`, corresponding to the modes
demonstrated in the images above.

`magFilter`와 `minFilter` 필드는 확대되거나 축소 된 텍셀을 보간하는 법을 지정합니다.
배율은 위의 oversampling 문제와 관련이 있으며, minification은 언더 샘플링과 관련이 있습니다.
선택 사항은 위의 이미지에서 설명한 `VK_FILTER_NEAREST`와 `VK_FILTER_LINEAR` 모드가 있습니다.

```c++
samplerInfo.addressModeU = VK_SAMPLER_ADDRESS_MODE_REPEAT;
samplerInfo.addressModeV = VK_SAMPLER_ADDRESS_MODE_REPEAT;
samplerInfo.addressModeW = VK_SAMPLER_ADDRESS_MODE_REPEAT;
```

The addressing mode can be specified per axis using the `addressMode` fields.
The available values are listed below. Most of these are demonstrated in the
image above. Note that the axes are called U, V and W instead of X, Y and Z.
This is a convention for texture space coordinates.

Addressing mode는 `addressMode` 필드를 사용하여 축마다 지정할 수 있습니다.
사용 가능한 값은 다음과 같습니다. 이들 대부분은 위의 이미지에서 보여줍니다.
축은 X, Y 및 Z 대신 U, V 및 W 라고 합니다. 이는 텍스처 공간 좌표에 대한 규칙입니다.

* `VK_SAMPLER_ADDRESS_MODE_REPEAT`: Repeat the texture when going beyond the
image dimensions.
* `VK_SAMPLER_ADDRESS_MODE_MIRRORED_REPEAT`: Like repeat, but inverts the
coordinates to mirror the image when going beyond the dimensions.
* `VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE`: Take the color of the edge closest to
the coordinate beyond the image dimensions.
* `VK_SAMPLER_ADDRESS_MODE_MIRROR_CLAMP_TO_EDGE`: Like clamp to edge, but
instead uses the edge opposite to the closest edge.
* `VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_BORDER`: Return a solid color when sampling
beyond the dimensions of the image.

* `VK_SAMPLER_ADDRESS_MODE_REPEAT` : 이미지 크기를 넘어서 갈 때 텍스처 반복
* `VK_SAMPLER_ADDRESS_MODE_MIRRORED_REPEAT` : 반복과 같지만 크기를 벗어나면 좌표를 반전시켜 반복합니다.
* `VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE` : 크기를 벗어나면 좌표에 가장 가까운 가장자리의 색을 취합니다.
* `VK_SAMPLER_ADDRESS_MODE_MIRROR_CLAMP_TO_EDGE` : CLAMP_TO_EDGE와 비슷하지만 대신에 가장 가까운 에지 반대편의 에지를 사용합니다.
* `VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_BORDER` : 이미지의 크기를 넘어 샘플링 할 때 단색을 반환합니다.

It doesn't really matter which addressing mode we use here, because we're not
going to sample outside of the image in this tutorial. However, the repeat mode
is probably the most common mode, because it can be used to tile textures like
floors and walls.

이 튜토리얼에서는 이미지 외부에서 샘플링하지 않기 때문에 여기서 어떤 어드레싱 모드를 사용하는지는 중요하지 않습니다.
그러나 반복 모드는 바닥과 벽과 같은 텍스처를 타일링하는 데 사용할 수 있기 때문에 가장 일반적인 모드입니다.

```c++
samplerInfo.anisotropyEnable = VK_TRUE;
samplerInfo.maxAnisotropy = 16;
```

These two fields specify if anisotropic filtering should be used. There is no
reason not to use this unless performance is a concern. The `maxAnisotropy`
field limits the amount of texel samples that can be used to calculate the final
color. A lower value results in better performance, but lower quality results.
There is no graphics hardware available today that will use more than 16
samples, because the difference is negligible beyond that point.

이 두 필드는 이방성 필터링을 사용해야하는지 여부를 지정합니다. 성능이 중요하지 않으면 사용하지 않을 이유가 없습니다.
`maxAnisotropy` 필드는 최종 색을 계산하는 데 사용할 수있는 텍셀 샘플의 양을 제한합니다.
값을 낮추면 성능은 향상되지만 품질은 낮아집니다.
현재 16 개 이상의 샘플을 사용할 그래픽 하드웨어는 없습니다. 왜냐하면 그 이상의 값은 무시할만한 차이기 때문입니다. 

```c++
samplerInfo.borderColor = VK_BORDER_COLOR_INT_OPAQUE_BLACK;
```

The `borderColor` field specifies which color is returned when sampling beyond
the image with clamp to border addressing mode. It is possible to return black,
white or transparent in either float or int formats. You cannot specify an
arbitrary color.

`borderColor` 필드는 클램프에서 테두리 어드레싱 모드로 이미지를 넘어 샘플링 할 때 어떤 색을 반환 할지를 지정합니다.
float 또는 int 형식으로 검정, 흰색 또는 투명을 반환 할 수 있습니다. 임의의 색상을 지정할 수 없습니다.

```c++
samplerInfo.unnormalizedCoordinates = VK_FALSE;
```

The `unnormalizedCoordinates` field specifies which coordinate system you want
to use to address texels in an image. If this field is `VK_TRUE`, then you can
simply use coordinates within the `[0, texWidth)` and `[0, texHeight)` range. If
it is `VK_FALSE`, then the texels are addressed using the `[0, 1)` range on all
axes. Real-world applications almost always use normalized coordinates, because
then it's possible to use textures of varying resolutions with the exact same
coordinates.

`unnormalizedCoordinates` 필드는 이미지의 텍셀을 주소 지정하는 데 사용할 좌표계를 지정합니다. 
이 필드가`VK_TRUE`이면 `[0, texWidth]` 와 `[0, texHeight]` 범위 내의 좌표를 사용하면됩니다. 
`VK_FALSE`의 경우, 텍셀은 모든 축의 `[0, 1]` 범위를 사용해 주소 지정됩니다. 
실제 응용 프로그램은 거의 항상 정규화 된 좌표를 사용합니다. 정확한 좌표와 함께 다양한 해상도의 텍스처를 사용할 수 있기 때문입니다.

```c++
samplerInfo.compareEnable = VK_FALSE;
samplerInfo.compareOp = VK_COMPARE_OP_ALWAYS;
```

If a comparison function is enabled, then texels will first be compared to a
value, and the result of that comparison is used in filtering operations. This
is mainly used for [percentage-closer filtering](https://developer.nvidia.com/gpugems/GPUGems/gpugems_ch11.html)
on shadow maps. We'll look at this in a future chapter.

비교 함수가 유효하게되어있는 경우, 텍셀은 우선 값과 비교되어 그 비교 결과가 필터링 조작에 사용됩니다. 
이것은 주로 쉐도우 맵에서 [percentage-closer filtering](https://developer.nvidia.com/gpugems/GPUGems/gpugems_ch11.html)에 사용됩니다. 
우리는 나중에 이것에 대해 살펴볼 것입니다.

```c++
samplerInfo.mipmapMode = VK_SAMPLER_MIPMAP_MODE_LINEAR;
samplerInfo.mipLodBias = 0.0f;
samplerInfo.minLod = 0.0f;
samplerInfo.maxLod = 0.0f;
```

All of these fields apply to mipmapping. We will look at mipmapping in a [later
chapter](/Generating_Mipmaps), but basically it's another type of filter that can be applied.

이 모든 필드는 밉 매핑에 적용됩니다. 우리는 나중에 [챕터](/Generating_Mipmaps)에서 밉 매핑을 살펴볼 것이지만, 
기본적으로 이것은 적용 할 수있는 필터의 또 다른 유형입니다.

The functioning of the sampler is now fully defined. Add a class member to
hold the handle of the sampler object and create the sampler with
`vkCreateSampler`:

샘플러의 기능이 이제 완전히 정의되었습니다. 샘플러 객체의 핸들을 보관할 클래스 멤버를 추가하고 샘플러 객체를 만듭니다.
`vkCreateSampler`:

```c++
VkImageView textureImageView;
VkSampler textureSampler;

...

void createTextureSampler() {
    ...

    if (vkCreateSampler(device, &samplerInfo, nullptr, &textureSampler) != VK_SUCCESS) {
        throw std::runtime_error("failed to create texture sampler!");
    }
}
```

Note the sampler does not reference a `VkImage` anywhere. The sampler is a
distinct object that provides an interface to extract colors from a texture. It
can be applied to any image you want, whether it is 1D, 2D or 3D. This is
different from many older APIs, which combined texture images and filtering into
a single state.

샘플러는 어디에서나 `VkImage`를 참조하지 않습니다. 
샘플러는 텍스처에서 색상을 추출하는 인터페이스를 제공하는 고유한 객체입니다. 
그것이 1D, 2D 또는 3D 이든 원하는 모든 이미지에 적용 할 수 있습니다. 
이는 텍스처 이미지와 필터링을 결합하여 단일 상태로 만드는 많은 이전 API와는 다릅니다.

Destroy the sampler at the end of the program when we'll no longer be accessing
the image:

더 이상 이미지에 액세스하지 않을 때 프로그램 끝에서 샘플러를 destory 합니다.

```c++
void cleanup() {
    cleanupSwapChain();

    vkDestroySampler(device, textureSampler, nullptr);
    vkDestroyImageView(device, textureImageView, nullptr);

    ...
}
```

## Anisotropy device feature (이방성 장치 특징)

If you run your program right now, you'll see a validation layer message like
this:

프로그램을 지금 실행하면 다음과 같은 유효성 검사 레이어 메시지가 표시됩니다:

![](/images/validation_layer_anisotropy.png)

That's because anisotropic filtering is actually an optional device feature. We
need to update the `createLogicalDevice` function to request it:

이방성 필터링은 실제로 선택적인 장치 기능이기 때문입니다. 
이 기능을 요청하기 위해`createLogicalDevice` 함수를 업데이트해야합니다 :

```c++
VkPhysicalDeviceFeatures deviceFeatures = {};
deviceFeatures.samplerAnisotropy = VK_TRUE;
```

And even though it is very unlikely that a modern graphics card will not support
it, we should update `isDeviceSuitable` to check if it is available:

최신 그래픽 카드가 지원하지 않을 가능성은 매우 낮지 만 isDeviceSuitable을 업데이트하여 사용 가능한지 확인해야합니다.

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    ...

    VkPhysicalDeviceFeatures supportedFeatures;
    vkGetPhysicalDeviceFeatures(device, &supportedFeatures);

    return indices.isComplete() && extensionsSupported && swapChainAdequate && supportedFeatures.samplerAnisotropy;
}
```

The `vkGetPhysicalDeviceFeatures` repurposes the `VkPhysicalDeviceFeatures`
struct to indicate which features are supported rather than requested by setting
the boolean values.

`vkGetPhysicalDeviceFeatures` 는 `VkPhysicalDeviceFeatures` 구조체를 다시 작성하여 
boolean 값을 설정하여 요청한 것이 아니라 지원되는 기능을 나타냅니다.

Instead of enforcing the availability of anisotropic filtering, it's also
possible to simply not use it by conditionally setting:

이방성 필터링의 가용성을 강화하는 대신 다음과 같이 조건부로 설정하여 간단히 사용할 수도 있습니다.

```c++
samplerInfo.anisotropyEnable = VK_FALSE;
samplerInfo.maxAnisotropy = 1;
```

In the next chapter we will expose the image and sampler objects to the shaders
to draw the texture onto the square.

다음 장에서는 이미지와 샘플러 객체를 쉐이더에 노출시켜 텍스처를 사각형에 그리게됩니다.

[C++ code](/code/24_sampler.cpp) /
[Vertex shader](/code/21_shader_ubo.vert) /
[Fragment shader](/code/21_shader_ubo.frag)
