---
layout: post
title:  "Tile-based Light Culling"
categories: DirectX
tags: [directx]
---
# idea

- naive한 라이트 계산은 모든 광원을 iterate하면서 라이트 계산을 했음.
    - pointlight와 spotlight의 경우에는 attenuation radius가 있어서 영향을 받지 않는 pixel이 생기지만, 일단 iterate 하면서 자신이 영향을 받는지(=거리가 먼지) 확인해야 함
    - 월드 상에 광원이 별로 없다면 상관이 없지만, 많아지게 된다면 iterate하는 cost가 큼
    - → pixel이 영향을 받는 광원만 iterate하게 하자
- Tile로 pixel을 쪼개서, per-tile로 영향을 주는 광원만을 미리 모아놓자
    - 광원에 index를 부여하고, tile에 영향을 주는 광원의 index를 저장
    - global한 광원의 배열을 만들고(`structedbuffer`), index만 추려낸다
- compute shader로 계산
    1. Tile을 world-space의 frustum으로 변환한다.
    2. 광원은 구형이므로, frustum-sphere intersection을 수행
    3. 교차하는 광원의 index를 저장
    4. 이후 라이팅을 할 때 pixel shader에서 속하는 tile을 찾아가서 적용받는 light만 사용

# 문제점

## Frustum-sphere intersection의 false positive

<p align="center">
  <img src="/assets/img33.png" align="center" width="90%">
</p>


- frustum의 4개의 평면(near / far를 제외)를 기준으로, sphere는 내부에 존재
    - 평면은 무한하므로 frustum 밖에서 교차할 수 있다.
    - 평면의 방정식에 대입했을 때, 원의 중점을 $c_0$, 평면 위의 한 점을 $p_0$라고 하면, 평면과 $c_0(x_0,y_0,z_0)$ 사이의 signed distance인 $\rho$는,
        
        $$
        \rho  = \frac{(c_0 - p_0)\cdot \mathbf{n}}{\|\mathbf{n}\|}= \frac{Ax_0+By_0+Cz_0+D}{\sqrt{A^2+B^2+C^2}}
        $$
        
        - note that $p_0\cdot\mathbf{n}=Ap_x+Bp_y+Cp_z=-D$
            
<p align="center">
  <img src="/assets/img34.png" align="center" width="90%">
</p>
            
        - 여기서 $0<\rho<R$ 일 때, plane을 기준으로 positive한 방향에 있고, 거리가 $R$ 보다 작다.
            - Frustum의 평면의 방정식을 만들 때, normal이 어떤 방향인지에 따라서 부등호가 달라짐

### 해결책

- corner와 center에 ray를 쏴서 ray-sphere intersection을 확인
    - plane intersection보다 느림
    - plane intersection으로 negative를 모두 걸러내고, true/false positive에 대해서만 적용
    - plane only(17ms) / ray only(24ms) / plane + ray(17ms)
        
<p align="center">
  <img src="/assets/img35.png" align="center" width="90%">
</p>
        
    - 하지만 ray와 intersection하지 않는 것은 false negative로 바뀌어버림
        - 가장 확실한건 AABB를 사용하는 것
- AABB를 이용해서 확인(미구현)
    - perspective view일 경우에는 화면 중심에서 멀어질수록 frustum이 cube에서 변형된 정도가 커짐
    - 따라서 frustum을 z축을 기준으로 잘라서, 작은 AABB를 만들고, 각각의 AABB와 intersection을 확인한다
        - [https://www.slideshare.net/slideshow/a-25d-culling-for-forward-siggraph-asia-2012/34909590](https://www.slideshare.net/slideshow/a-25d-culling-for-forward-siggraph-asia-2012/34909590)
        - [https://wickedengine.net/2018/01/optimizing-tile-based-light-culling/](https://wickedengine.net/2018/01/optimizing-tile-based-light-culling/)
    - 각 타일에서의 object에 대한 min max z를 구하고, 이를 기준으로 32등분
    - 32등분된 volume에 속하는 z값이 있는지 확인해서 bit mask를 생성
    - light에 대해서도 volume에 대한 intersection확인
    - AND operation으로 영향을 주는 light만 선택
- Hardware Rasterizer를 이용(미구현)
    - 먼저 depth buffer를 render
    - 이후 Proxy Geometry를 통해서 광원을 render
        - depth test를 통해서 광원을 그림 → 타일 내에서의 min을 구하고 depth 비교
    - [https://ubm-twvideo01.s3.amazonaws.com/o1/vault/gdc2016/Presentations/Zhdan_Sjoholm_Light_culling_MGPU.pdf](https://ubm-twvideo01.s3.amazonaws.com/o1/vault/gdc2016/Presentations/Zhdan_Sjoholm_Light_culling_MGPU.pdf)

# 구현

## structuedbuffer

- 기존의 방식에서는 constant buffer를 이용해서 광원의 정보를 GPU로 넘겼음
    - 빠르지만, 많은 용량을 제공하지 않음 ( ~ 4096 * 16 byte = 64KB)
        - pointlight 하나당 48 byte → 1365개까지만 가능
    - 따라서 많은 용량을 제공해주는 `structuredbuffer` 를 이용
        - 또한 GPU에서 write가 가능함

## Unordered Access View

- 버퍼를 GPU에서 이용하는 SRV는 read-only임
- Compute Shader에서 write를 하려면 UAV로 접근해야함
- UAV로 이용하는 과정
    - `DeviceContext::CreateBuffer()`로 버퍼를 생성
        
        ```cpp
            D3D11_BUFFER_DESC bufDesc = {};
            bufDesc.Usage = D3D11_USAGE_DEFAULT;
            bufDesc.ByteWidth = sizeof(FTileLightIndex) * tileCount;
            bufDesc.BindFlags = D3D11_BIND_UNORDERED_ACCESS | D3D11_BIND_SHADER_RESOURCE;
            bufDesc.CPUAccessFlags = 0;
            bufDesc.StructureByteStride = sizeof(FTileLightIndex);
            bufDesc.MiscFlags = D3D11_RESOURCE_MISC_BUFFER_STRUCTURED;
        ```
        
    - `DeviceContext::CreateUnorderedAccessView()` 로 UAV를 생성
        
        ```cpp
            D3D11_UNORDERED_ACCESS_VIEW_DESC uavDesc = {};
            uavDesc.Format = DXGI_FORMAT_UNKNOWN;
            uavDesc.ViewDimension = D3D11_UAV_DIMENSION_BUFFER;
            uavDesc.Buffer.FirstElement = 0;
            uavDesc.Buffer.NumElements = tileCount;
        
            hr = Device->CreateUnorderedAccessView(TileLightListBuffer, &uavDesc, &TileLightListUAV); 
        ```
        
    - Shader에서 접근
        
        ```cpp
        // c++
        DeviceContext->CSSetShaderResources(0, 1, srvs);
        DeviceContext->CSSetUnorderedAccessViews(1, 1, &TileLightListUAV, nullptr);
        
        // hlsl
        StructuredBuffer<FPointLight> PointLightBufferList : register(t0);
        RWStructuredBuffer<FTileLightIndex> TileLightIndicesListCS : register(u1);
        
        // c++
        DeviceContext->CSSetUnorderedAccessViews(1, 1, nullUAV, nullptr);
        ```
        

<aside>
💡

Compute Shader에 바인딩되어있는 resource view는 바인딩이 풀리기 전까지 다른 shader stage에서 사용할 수 없다. (write를 수반하므로) 따라서 compute shader에서 사용이 끝나면 꼭 바인딩을 풀어줘야한다.  

`D3D11 WARNING: ID3D11DeviceContext::PSSetShaderResources: Resource being set to PS shader resource slot 17 is still bound on output! Forcing to NULL. [ STATE_SETTING WARNING #7: DEVICE_PSSETSHADERRESOURCES_HAZARD]`

</aside>

## heatmap

- 일단 buffer에 제대로 기록되고 있는지 확인해야함
    - heatmap을 기존의 RTV에 alpha값을 준채로 그려서 해당 타일에 들어가는 광원의 갯수를 시각화
    - → 한 타일에 들어가는 광원의 수가 제한보다 많을 경우(=빨간색) 최대 값을 조절. scene에 맞는 값을 찾는데 이용
    
<p align="center">
  <img src="/assets/img36.png" align="center" width="90%">
</p>

## Compute Shader

- CUDA의 DirectX 버전
- `DeviceContext::Dispatch()` 로 ThreadGroup의 개수를 설정할 수 있고 셰이더 코드에서 `[numthreads(NumThread, 1, 1)]` 를 통해서 group내의 thread 개수를 설정할 수 있음

<p align="center">
  <img src="/assets/img37.png" align="center" width="90%">
</p>

- 실행되는 thread의 위치를 알려주는 키워드 제공
    - `SV_GroupThreadID`, `SV_GroupID`, `SV_DispatchThreadID`, `SV_GroupIndex`
- Tile 내에서의 Per-Pixel 계산은 수행하지 않음
    - 이번 경우에는 `numthreads(64, 1, 1)` 를 통해서 두 개의 warp에서 라이트와의 교차 확인을 할 수 있도록 함
    - 교차 확인에 필요한 Tile의 plane과 ray는 굳이 공유하지 않고 각각의 thread가 구해서 사용
        - WARP 두개만 쓰고, 별로 큰 연산이 아니기 때문에 굳이 하나의 thread 또는 warp가 맡아서 구할 필요 없음
    - `numthreads(32, 1, 1)` 보단 빠르지만, 그 이상의 thread는 효과가 없었음
        - 32 / 64 / 128 thread의 dispatch 시간 : 5.33ms / 3.96ms/ 4.00ms
- per-thread의 계산이 끝나면, `tid==0` 인 thread가 기록
    - 어디까지 모든 thread가 실행하도록 하고, 어디부터 하나의 thread만 실행하게 해야하는지가 중요

### Thread별 work 분배 (& groupshared)

- 기존구조는 thread각각이 `SharedLightIndices[TileIndex][LocalIndex]` 에 접근해서 하나씩 기록하였음 (3.96ms)
    - race condition을 방지하기 위해서 `InterlockedAdd` 를 통해서 thread가 서로 다른 index를 접근하고 기록하도록 하였음
    - InterlockedAdd가 필요한가? 차라리 thread당 배열을 만들고, thread는 자신의 배열에만 기록한 이후 `tid == 0`인 thread가 한번에 기록하면 어떨까?

```cpp
groupshared uint SharedLightIndices[MAX_NUM_INDICES_PER_TILE];
groupshared uint SharedLightCount;
...

    int NumIntersection = 0;
    for (uint LightIndex = localThreadID; LightIndex < MaxNumPointLight; LightIndex += NumThread)
    {
        // 쓰레기값은 버림 (heap에서 넘어와서 다 음수값)
        if (PointLightBufferList[LightIndex].AttenuationRadius <= 0)
        {
            continue;
        }
        // frustum 방식은 false positive 발생
        // true negative만 reject
        if (!SphereFrustumPlaneIntersection(PointLightBufferList[LightIndex].Position, PointLightBufferList[LightIndex].AttenuationRadius, Planes))
        {
            continue;
        }
        
        // if문에 넣지말고 밖에서 값 넣기
        bool intersects = false;
        // corner4개에서 나오는 ray와 intersection + 중심
        for (int j = 0; j < 5; ++j)
        {
            if (IntersectRaySphere(RayOrigin[j], RayDir[j], PointLightBufferList[LightIndex].Position, PointLightBufferList[LightIndex].AttenuationRadius))
            {
                intersects = true;
                break;
            }
        }
        
        if (intersects)
        {
            uint index;
            **InterlockedAdd(SharedLightCount, 1, index); // 각 thread한테 index 부여**
            if (index < MAX_NUM_INDICES_PER_TILE)
            {
                **SharedLightIndices[index] = LightIndex; // 각 thread가 자신의 index에 기록**
            }
        }
    }
    
    // thread0가 기록
    GroupMemoryBarrierWithGroupSync();
    if (localThreadID == 0)
    {
        TileLightIndicesListCS[tileIndex].LightCount = min(SharedLightCount, MAX_NUM_INDICES_PER_TILE);
        for (uint i = 0; i < min(SharedLightCount, MAX_NUM_INDICES_PER_TILE); ++i)
        {
            TileLightIndicesListCS[tileIndex].LightIndices[i] = SharedLightIndices[i];
        }
    }
}
```

- thread별로 리스트를 저장하고 나중에 하나의 thread가 그 결과물을 접근해서 모으도록 변경했는데, 오히려 dispatch 시간이 증가 (4.29ms)
    - `InterlockedAdd`가 생각보다 느리지 않은 것 같다.
    - `하나의 uint에 대해서 atomic 계산 + 병렬 write` vs `하나의 thread만 write`
        - 어차피 thread 하나당 4B의 bandwidth를 보장받음 (multi-bank architecture)
        - bank conflict도 없음
        - `numthreads`가 변경되면 다시 확인 필요할것으로 보임

```cpp
groupshared uint SharedLightIndices[NumThread][MAX_NUM_INDICES_PER_TILE];
groupshared uint SharedLightNumIndices[NumThread];
...

    
    SharedLightNumIndices[localThreadID] = 0;
    uint LocalLightNumIndex = 0;

    int NumIntersection = 0;
    for (uint LightIndex = localThreadID; LightIndex < MaxNumPointLight; LightIndex += NumThread)
    {
        // 쓰레기값은 버림 (heap에서 넘어와서 다 음수값)
        if (PointLightBufferList[LightIndex].AttenuationRadius <= 0)
        {
            continue;
        }
        // frustum 방식은 false positive 발생
        // true negative만 reject
        if (!SphereFrustumPlaneIntersection(PointLightBufferList[LightIndex].Position, PointLightBufferList[LightIndex].AttenuationRadius, Planes))
        {
            continue;
        }
      
        bool intersects = false;
        // corner4개에서 나오는 ray와 intersection + 중심
        for (int j = 0; j < 5; ++j)
        {
            if (IntersectRaySphere(RayOrigin[j], RayDir[j], PointLightBufferList[LightIndex].Position, PointLightBufferList[LightIndex].AttenuationRadius))
            {
                intersects = true;
                break;
            }
        }
        
        if (intersects)
        {
            if (LocalLightNumIndex < MAX_NUM_INDICES_PER_TILE)
            {
                      **// thread가 groupshared된 2차원 배열에서 자신의 index에 해당하는 배열저장**
                **SharedLightIndices[localThreadID][LocalLightNumIndex] = LightIndex;**
                LocalLightNumIndex++;
            }
        }

    }
    SharedLightNumIndices[localThreadID] = LocalLightNumIndex;
    
    // thread0가 기록
    GroupMemoryBarrierWithGroupSync();
    
    // 가장 앞의 쓰레드가 값을 컨트롤함
    
    if (localThreadID == 0)
    {
        uint SharedLightCount = 0;

     ****   for (uint tid = 0; tid < NumThread; ++tid)
        {
            for (uint LocalIndex = 0; LocalIndex < SharedLightNumIndices[tid]; ++LocalIndex)
            {
                if (SharedLightCount >= MAX_NUM_INDICES_PER_TILE - 1)
                {
                    TileLightIndicesListCS[tileIndex].LightCount = SharedLightCount;
                    return;
                }
                **// tid==0가 모든 thread가 기록한 배열을 돌면서 하나씩 기록
                TileLightIndicesListCS[tileIndex].LightIndices[SharedLightCount++]
                = SharedLightIndices[tid][LocalIndex];**
            }
        }
        TileLightIndicesListCS[tileIndex].LightCount = SharedLightCount;
    }
```