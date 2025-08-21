---
layout: post
title:  "Deferred Rendering"
categories: Graphics
tags: [directx]
---
`IA-VS-RS-PS-OM`으로 이어지는 렌더링 파이프라인에서 Rasterizer에서 나온 픽셀은 Pixel Shader를 거치면서 라이팅과 같은 여러 연산을 진행하게 되고, 다음 단계인 Output Merger에서 테스트를 통해 최종적으로 그려질지 결정된다.

여기서 문제가 발생하는데, OM단계에서 픽셀이 그려질지 결정하기 때문에 RS에서 나온 모든 픽셀이 Pixel Shader를 거치게 된다는 것이다.
이는 depth/stencil test를 fail할 픽셀도 PS단계의 연산을 진행한다는 것으로, 결과물과는 상관 없는 픽셀에 대해서 연산을 진행하는 꼴이 된다.

![](/assets/img52.png)

위의 파이프라인을 한 번 진행(패스)하는 것을 forward rendering이라고 하고, 위의 문제를 해결하기 위해서 패스를 두 번 진행하는 deferred rendering(shading)이 나왔다.

# Deferred Rendering
Deferred rendering의 주요 골자는 "화면에 그려질 픽셀만 연산하자"이다.
첫 번째 패스(Geometry pass)에서 화면에 그려질 픽셀만 빠르게 분류하고, 두 번째 패스(Lighting pass)에서 해당 픽셀에 대해서 shading을 진행한다.
Geometry pass는 복잡한 shading 과정 없이 albedo, normal, depth 등과 같이 shading에 필요한 정보를 texture에 기록한다.
첫 단계에서 모든 draw call의 데이터는 파이프라인을 거치게 되지만, pixel shader에서의 단계가 생략되었기 때문에 빠르게 계산을 끝낼 수 있다.
이후 Output merger를 통과한 픽셀들에 대해서만 두 번째 패스를 진행하게 되는데, 이 때에는 실제 화면에 그려질 픽셀에 대해서만 연산을 하기 때문에 forward rendering에서 있었던 불필요한 과정을 생략할 수 있다.

이렇듯 deferred rendering은 불필요한 연산을 줄인다는 장점이 있지만, 동시에 geometry pass에서 통과한 픽셀들을 저장하기 위해서 texture가 많이 필요하다는 단점이 있다. (이를 처리하기 위해서 vram에서의 큰 bandwidth를 필요로 한다.)

![](/assets/img53.png)

Geometry pass를 통과하면 다음과 같은 texture가 나오게 된다.

![](/assets/img54.png)

위에서부터 순서대로 diffuse, UUID, WorldPos, Normal, albedo, specular이다. 이렇게 나온 texture를 lighting pass에서 잘 조합해서 사용한다. (albedo랑 diffuse는 특별한 목적이 있어서 나눠놨지, 기본적으로는 동일하게 diffuse texture를 사용하였다.)

```c
Texture2D g_renderTex : register(t10);
Texture2D g_depthOnlyTex : register(t11);
Texture2D g_worldPosTex : register(t12);
Texture2D g_worldNormalTex : register(t13);
Texture2D g_albedoTex : register(t14);
Texture2D g_specularTex : register(t15);

float4 mainPS(SamplingPixelShaderInput input) : SV_TARGET
{
    float2 EncodedNormal = g_worldNormalTex.Sample(g_Sampler, input.texcoord).rg; // normal
    float3 normal = DecodeNormalOctahedral(EncodedNormal); // texture에 압축되어있던 정보를 decode

    float3 DiffuseColor = g_renderTex.Sample(g_Sampler, input.texcoord).rgb; // diffuse
    
    float3 materialDiffuseColor = g_albedoTex.Sample(g_Sampler, input.texcoord).rgb; // albedo 사용
    float3 materialSpecularColor = g_specularTex.Sample(g_Sampler, input.texcoord).rgb; // specular texture 사용
    float3 worldPos = g_worldPosTex.Sample(g_Sampler, input.texcoord).rgb; // world position 사용
    float materialSpecularScalar = g_specularTex.Sample(g_Sampler, input.texcoord).a; // specular의 exponent 사용
    float3 viewDirection = normalize(float3(eyeWorld - worldPos));

    float3 color = float3(0.f, 0.f, 0.f);
    int i;
    // constant buffer로 넘어온 light에 대해서 계산
    for (i = 0; i < NumDirLights; i++)
    {
        color += CalculateDirectionLight(DirLights[i], worldPos, normal, viewDirection, materialDiffuseColor, materialSpecularColor, materialSpecularScalar);
    }
    
    for (i = 0; i < NumPointLights; i++)
    {
        color += CalculatePointLight(PointLights[i], worldPos, normal, viewDirection, materialDiffuseColor, materialSpecularColor, materialSpecularScalar);
    }
    
    DiffuseColor = color;
    ...
    return float4(DiffuseColor, 1.0f);
}
```

이러한 과정을 거치면 라이팅에 대한 계산은 오직 lighting pass에서의 픽셀, 즉 최종적으로 모니터 상에 그려질 픽셀에 대해서만 진행되게 된다. 

## texture 용량 줄이기
여기서 하나 눈여겨 볼 점은 normal과 specular texture이다. Deferred rendering의 최대 단점은 렌더링에 필요한 texture의 개수가 많아진다는 것이다. 이를 해결하기 위해서 최대한 texture의 개수를 줄이고, texture의 크기를 줄여야 한다. 예를 들어, [KILLZONE 2](https://www.guerrilla-games.com/media/News/Files/Develop07_Valient_DeferredRenderingInKillzone2.pdf)에서는 다음과 같이 5개의 texture에 필요한 정보를 모두 넣었다.

![](/assets/img55.png)

이번 구현 과정에서는 기본적으로 R32G32B32A32의 포맷을 따랐지만, 정밀도가 높게 요구되지 않는 normal의 경우에는 R16G16을 이용해서 낮은 정밀도를 이용했고, 동시에 normal을 구성하는 $N=(x,y,z)$에서 $|N| = \sqrt{x^2+y^2+z^2} = 1$라는 성질을 이용해서 texture에 normal을 기록할 때 z값은 기록하지 않고 대신 이후 패스에서 [octahedral mapping](https://gamedev.stackexchange.com/questions/169508/octahedral-impostors-octahedral-mapping)을 통해서 z값을 복원하였다.

이렇게 texture에 저장된 정보를 잘 사용하면, 다음과 같이 lighting이 적용된 결과물을 얻을 수 있다.
![](/assets/img56.png)