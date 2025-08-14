---
layout: post
title:  "Depth of Field"
categories: Graphics
tags: [directx]
---

# Photography
Depth of field란 카메라의 성질 때문에 거리에 따라서 초점이 맞는 영역을 의미한다. 카메라의 조리개가 커지면 초점이 맞는 영역이 더 줄어들고, 조리개가 작아지면 초점이 맞는 영역이 더 길어진다.

정확하게 말하면, 초점이 맞는다는 것은 착란원이 카메라 센서의 한 pixel보다 작아졌을 때를 의미한다. 피사체의 한 지점이 카메라 렌즈를 통과하면서 센서에 맺히는 착란원은, 그 지점과 카메라 렌즈 사이의 거리에 따라서 결정이 된다. 

이걸 물리적으로 완벽하게 구현하려면 화면에 비치는 모든 지점에 대해서 렌즈와의 거리를 계산하고, 착란원의 크기를 계산한 다음 화면에 그려내야 할 것이다. 하지만 이렇게 그려내는 대신, 그래픽스에서는 gaussian blur를 통해서 이를 흉내낸다.

# Graphics
Depth of field를 흉내내기 위해서, 그래픽스에서는 gaussian blur를 통해서 생성된 완전히 out-of-focus인 화면을 기존의 in-focus 화면과 섞는다. 다만, 지금은 compute shader를 사용할 것이 아니기 때문에 out-of-focus인 텍스쳐를 따로 생성하는 것이 아니라 post-processing과정에서의 pixel shader에서 주변 픽셀을 샘플링하여 블러 처리를 할 것이다.

```c
Texture2D SceneDepthTexture : register(t99); // scene depth
Texture2D SceneTexture : register(t100); // post-process 되지않은 원래의 scene rgb 

float4 mainPS(VS_OUTPUT input) : SV_Target
{
    int k = int(BlurRadius);
    float3 FinalColor = float3(0, 0, 0);
    float TotalWeight = 0.0;
    float2 uv = input.UV;

    float centerDepth = GetSceneDepth(uv); // 현재 픽셀의 depth
    float FocusFactor = GetFocusFactor(centerDepth, FocalDistance, FocalRegion, BlurAmount); // 현재 

    for (int i = -k; i <= k; ++i)
    {
        for (int j = -k; j <= k; ++j)
        {
            // 주변 픽셀
            float2 UVAdj = uv + float2(i, j) / 1080.0;
            
            // 주변 픽셀의 depth
            float DepthAdj = GetSceneDepth(UVAdj);
            // 주변 픽셀이 초점에서 얼마나 벗어나 있는지
            // in-focus일 때 0 out-of-focus일 때 1
            float FocusFactorAdj = GetFocusFactor(DepthAdj, FocalDistance, FocalRegion, BlurAmount);
            
            // 가우시안을 기준으로 weight를 적용
            float SamplingWeight = Gaussian(length(float2(i, j)), BlurRadius);

            // 초점에서 벗어날수록, 가우시안 값이 클수록 weight가 증가
            float Weight = FocusFactorAdj * SamplingWeight;
            
            // 주변 픽셀을 샘플링하고 weight와 함께 누적
            float3 ColorAdj = SceneTexture.Sample(SceneSampler, UVAdj).rgb;
            FinalColor += ColorAdj * Weight;
            TotalWeight += Weight;
        }
    }

    if (TotalWeight > 0.0)
        FinalColor /= TotalWeight;
    else
        FinalColor = SceneTexture.Sample(SceneSampler, uv).rgb;


    return float4(FinalColor, FocusFactor);
}
```

![alt text](/assets/img43.png)