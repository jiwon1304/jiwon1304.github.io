---
layout: post
title:  "DirectX11 Occlusion Culling Query"
categories: DirectX
tags: [directx]
---
Occlusion Culling이란, 앞의 물체로부터 가려지는(occluded)물체를 제외하는(culling)하는 기법이다. 물체가 가려지는지 가려지지 않는지 확인하기 위해서는 실제로 draw를 해봐야하는데, 이 때 모든 vertices를 그릴 수는 없으니 단순한 물체(구, 박스)로 대체해서 그린다. 

Culling은 최대한 보수적으로 작동해야 한다. 따라서 그려지지 않아도 되는 것을 그리는 것은 괜찮지만, 그려야 하는 것을 그리지 않는 것은 방지해야한다. 이를 위해서 occlusion에 사용되는 mesh(박스, 구, 원기둥 등 실제의 복잡한 mesh를 대신해서 그려지는 간단한 mesh)에 대해서, 뒤 물체를 가리는 물체(occluder)는 실제 mesh보다 작아야 하고, 앞 물체로부터 가려지는 물체(occludee)는 실제 mesh보다 커야한다.

사실 occlusion culling의 occluder는 보통 맵에 고정된, 큰 물체를 주로 이용한다. 이번 과제에서는 그런 물체는 없고 5만개의 사과 object였기 때문에, occluder와 occludee를 따로 정하지 않고, 모두가 가리고 가려질 수 있도록 하였다. 사과 mesh가 원형에 가까웠기 때문에 실제 사과의 vertices를 이용하지 않고 쉐이더에 직접 10개의 삼각형을 집어넣었고, 10개의 삼각형을 사과의 위치에 배치하고 billboard 방식으로 항상 카메라를 가리키도록 하였다.

```cpp
const static float2 PolygonPos[30] =
{
    // 9개의 삼각형을 만드는 버텍스들
    // 각 삼각형의 3개 정점 정의 (중앙 점 (0, 0) + 원 둘레의 점들)

    // 삼각형 1
    float2(0.0000, 0.0000), float2(1.0000, 0.0000), float2(0.7660, 0.6428),
    // 삼각형 2
    float2(0.0000, 0.0000), float2(0.7660, 0.6428), float2(0.3420, 0.9397),
    // 삼각형 3
    float2(0.0000, 0.0000), float2(0.3420, 0.9397), float2(-0.3420, 0.9397),
    // 삼각형 4
    float2(0.0000, 0.0000), float2(-0.3420, 0.9397), float2(-0.7660, 0.6428),
    // 삼각형 5
    float2(0.0000, 0.0000), float2(-0.7660, 0.6428), float2(-1.0000, 0.0000),
    // 삼각형 6
    float2(0.0000, 0.0000), float2(-1.0000, 0.0000), float2(-0.7660, -0.6428),
    // 삼각형 7
    float2(0.0000, 0.0000), float2(-0.7660, -0.6428), float2(-0.3420, -0.9397),
    // 삼각형 8
    float2(0.0000, 0.0000), float2(-0.3420, -0.9397), float2(0.3420, -0.9397),
    // 삼각형 9
    float2(0.0000, 0.0000), float2(0.3420, -0.9397), float2(0.7660, -0.6428),
    // 삼각형 10
    float2(0.0000, 0.0000), float2(0.7660, -0.6428), float2(1.0000, 0.0000)
};
```
해당 메쉬가 그려지는지 안그려지는지(=depth test를 통과하는지) 확인하기 위해서는 [DirectX11의 `Query()`](https://learn.microsoft.com/ko-kr/windows/win32/api/d3d11/nn-d3d11-id3d11query)를 이용하면 된다. 다양한 Query가 있지만, occlusion을 위해서 사용하는 것은 해당 drawcall에서 depth test를 통과한 픽셀이 몇개인지 확인하는 쿼리이다. Occluder를 모두 그린 후, occludee를 그리면서 drawcall에서 depth test를 통과한 픽셀이 없다는 것은 모든 픽셀이 앞의 물체에 의해서 가려진다는 것이다. 따라서 다음과 같이 `Begin() -> Draw() -> End() -> Query() -> GetData()`의 순서로 진행된다.

```cpp
// draw 준비
...
    for (auto& StaticMeshComp : Meshes)
    {
        ID3D11Query* Query;
        if (!QueryPool.empty()) // query pool에서 꺼내온다. ID3D11Query는 재활용 가능하다.
        {
            Query = QueryPool.front();
            QueryPool.pop();
        }
        Graphics->DeviceContext->Begin(Query);
        RenderOccludee(StaticMeshComp); // 간소화된 occludee를 그림
        Graphics->DeviceContext->End(Query);
        Queries.push(Query);
        QueryMeshes.push(StaticMeshComp);
    }
// 모든 drawcall당마다 하나의 query를 붙임
...
// 모든 query가 끝나면
    while (!Queries.empty())
    {
        ID3D11Query* Query = Queries.front();
        UStaticMeshComponent* StaticMeshComp = QueryMeshes.front();
        UINT64 pixelCount = 0;
        while (Graphics->DeviceContext->GetData(Query, &pixelCount, sizeof(pixelCount), 0) == S_FALSE) {} // GetData가 성공할때까지 계속 기다린다.
        if (pixelCount > 0) { // 픽셀 하나라도 그려지면,
            DisOccludedMeshes.Add(StaticMeshComp); // 렌더할 목록에 추가한다.
        }
        Queries.pop();
        QueryMeshes.pop();
        QueryPool.push(Query);
    }
```

하지만 이 결과 하나의 사과에 대해서 occluder용 drawcall 한번, occludee용 drawcall 한번 그리게 되어서, 결국 사과를 하나 그리기 위해서 drawcall을 세번 그려야 했다. 이를 해결하기 위해서, occluder를 따로 그리지 않고 이전 프레임의 depth map을 이용하기로 했다. 이러면 occluder를 위해서 한번, 실제 mesh 한번만 그리면 되었다. 또한 이전 프레임의 `Present()` 부터 현재 프레임의 `Draw()`까지만 쿼리가 완료되면 되므로 thread를 이용해서 `Present()`가 끝나는 직후 depth texture를 가져와 `Query()`를 진행하고, `Draw()`를 실행하기 직전에 `GetData()`를 호출하였다. 

`Query()`는 main thread가 아닌 별개의 worker thread에서 진행되었는데, 이 과정에서 immediate context는 여러 thread에서 호출될 수 없다는 것을 알았다. 이를 위해서 deferred context를 생성하고 worker thread에서 query를 하였고, [getdata는 main thread의 immediate context에서 호출하였다.](https://learn.microsoft.com/en-us/windows/win32/api/d3d11/nf-d3d11-id3d11devicecontext-getdata)

```cpp
// 비동기 오클루전 컬링 실행 (결과는 나중에 GetResult()로 받음)
// thread는 init때만 정함.
void OcclusionAsync::QueryAsync(
    ID3D11DepthStencilView* InDepthStencilView,
    TArray<UStaticMeshComponent*> InComponents)
{
    Futures.clear();
    
    int TotalComponents = InComponents.Len();
    int ChunkSize = (TotalComponents + NumThreads - 1) / NumThreads;

    assert(NumThreads == QueryContexts.Num());

    FViewportCameraTransform CameraTransform = GEngineLoop.GetLevelEditor()->GetActiveViewportClient()->ViewTransformPerspective;

    FVector CameraLocation = CameraTransform.GetLocation();
    FMatrix ViewProjection = GEngineLoop.GetLevelEditor()->GetActiveViewportClient()->GetViewMatrix()
        * GEngineLoop.GetLevelEditor()->GetActiveViewportClient()->GetProjectionMatrix();


    for (int i = 0; i < NumThreads; ++i)
    {
        int Start = i * ChunkSize;
        int End = FMath::Min((i + 1) * ChunkSize, TotalComponents);
        if (Start >= End) break;

        // read only니깐 상관없음
        Futures.push_back(std::async(std::launch::async, &OcclusionAsync::ExecuteQuery, this, 
            InDepthStencilView, InComponents, Start, End , ViewProjection, CameraLocation, std::ref(QueryContexts[i])));
    }

    // 모든 쓰레드가 drawcall을 끝낼때까지 기다림.
    // 실제로는 command queue에 넣고있긴함.
    // TODO : futre 하나 끝나면 하나 다 execute하기
    for (auto& Future : Futures)
    {
        Future.wait();
    }


    for (auto& QC : QueryContexts)
    {
        // command list에 모두 push.
        ID3D11CommandList* CommandList = nullptr;
        HRESULT hr = QC.Context->FinishCommandList(FALSE, &CommandList);
        if (FAILED(hr)) {
            hr = Graphics->Device->GetDeviceRemovedReason();
            std::cerr << "Failed to create Command List!" << std::endl;
        }
        QC.CommandList = CommandList;

    }

    // Command list execute
    for (auto& QC : QueryContexts)
    {
        Graphics->DeviceContext->ExecuteCommandList(QC.CommandList, true);
        QC.CommandList->Release();
        QC.CommandList = nullptr;
    }
}

void OcclusionAsync::GetResultOcclusionQuery(struct QueryContext& QC)
{
    while (!QC.Queries.empty())
    {
        ID3D11Query* Query = QC.Queries.front();
        UStaticMeshComponent* StaticMeshComp = QC.Meshes.front();
        UINT64 pixelCount = 0;
        while (Graphics->DeviceContext->GetData(Query, &pixelCount, sizeof(pixelCount), 0) == S_FALSE) {}
        // 통과할경우
        if (pixelCount > 0) {
            // 이미 들어가있는 항목일경우(전에 그린거) 중복을 안하고 안넣음
            if (!StaticMeshComp->bAlreadyQueued)
            {
                QC.DisOccludedMeshes.Add(StaticMeshComp);
                StaticMeshComp->bWasDrawnBefore = true;
            }
        }
        else
        {
            StaticMeshComp->bWasDrawnBefore = false;
        }
        // 초기화
        StaticMeshComp->bAlreadyQueued = false;

        QC.Queries.pop();
        QC.Meshes.pop();

        QueryPool.push(Query);
    }
}

// Draw 직전
void FRenderer::PrepareRender()
{
    // Frustum Culling
    Octree->FrustumCull(Frustum, Components);

    DisoccludedComponents = OcclusionSystem.GetResult();
    ...
}
...
// Draw
void FRenderer::Render()
{
    ...
    PrepareShader();
    ...
    RenderStaticMeshes();
    OcclusionSystem.QueryAsync(Graphics->DepthStencilView, ComponentsProxy);
    ...
}
```

하지만 아직까지 `drawcall`을 두번 해야한다는 단점이 있었다. 이를 해결하기위해서, mesh를 draw할때 새로운 texture에 UUID를 기록하고, 다음 프레임에서 UUID가 기록된 것만 draw하기로 하였다. 대략적인 흐름은 다음과 같다.

`depth stencil buffer와 UUID용 32bit buffer에 기록 -> GPU에서 UAV로 접근해서 적혀있는 픽셀 확인 -> 기록된 UUID만 structured buffer에 담아서 CPU에 전달 -> 해당 UUID만 CPU에서 drawcall`

결과적으론 실패했다. Pixel shader에서 UUID용 텍스쳐 버퍼에 그릴려고 했는데, 생각해보니 depth test를 진행하기 전에 UUID를 그리는 꼴이 되어서 모든 픽셀이 다 그려졌다. 해결하려면 
(제대로 작동하지 않았지만) 속도 또한 느렸는데, Pixel 모두를 iterate하면서 중복된 항목 없이 UUID를 하나씩 추가해야하는데, UUID를 확인하고 추가하는 과정이서 race condition이 많이 발생하는 것으로 보인다. 
대신 CPU에서 해보려고 했지만, 동일하게 많은 시간이 걸렸다. single thread를 이용했기 때문에, race condition은 발생하지 않았지만, 중복을 제거하기 위해서 hashmap에 넣는 과정에서 많은 시간이 걸렸다. 아마도 hash function을 쓰지 않고 Array를 쓴다면 더 좋은 결과가 나올 것 같지만 시간이 부족해서 해보지는 못했다.

참고용으로 Compute shader를 사용하는 과정의 코드를 남긴다.

<details>
<summary> C++ 코드 </summary>

<div markdown="1">

```cpp
HRESULT FRenderer::CreateUAV()
{
    constexpr uint32 MAX_UUID_COUNT = 2 << 10; // 2^16개의 uuid만 허용. 필요하면 2^32-2까지 늘릴 수 있음.
    HRESULT hr = S_OK;
    //////////////////////////////////////////
    // UAV에 사용할 Texture 생성
    D3D11_TEXTURE2D_DESC textureDesc = {};
    textureDesc.Width = Graphics->screenWidth;
    textureDesc.Height = Graphics->screenHeight;
    textureDesc.MipLevels = 1;
    textureDesc.ArraySize = 1;
    textureDesc.Format = DXGI_FORMAT_R32_UINT;
    textureDesc.SampleDesc.Count = 1;
    textureDesc.Usage = D3D11_USAGE_DEFAULT;
    textureDesc.BindFlags = D3D11_BIND_SHADER_RESOURCE | D3D11_BIND_UNORDERED_ACCESS;
    Graphics->Device->CreateTexture2D(&textureDesc, nullptr, &UUIDMapTexture);

    // Shader Resource View (SRV) 생성
    D3D11_SHADER_RESOURCE_VIEW_DESC srvDesc = {};
    srvDesc.Format = DXGI_FORMAT_R32_UINT;
    srvDesc.ViewDimension = D3D11_SRV_DIMENSION_TEXTURE2D;
    srvDesc.Texture2D.MipLevels = 1;
    hr = Graphics->Device->CreateShaderResourceView(UUIDMapTexture, &srvDesc, &UUIDTextureSRV);
    if (FAILED(hr))
    {
        return hr;
    }
    // UAV 생성
    D3D11_UNORDERED_ACCESS_VIEW_DESC uavDesc = {};
    uavDesc.Format = DXGI_FORMAT_R32_UINT;
    uavDesc.ViewDimension = D3D11_UAV_DIMENSION_TEXTURE2D;
    hr = Graphics->Device->CreateUnorderedAccessView(UUIDMapTexture, &uavDesc, &UUIDTextureUAV);
    if (FAILED(hr))
    {
        return hr;
    }

    //////////////////////////////////////////
    // UUID 결과 저장용 Structured Buffer 생성
    D3D11_BUFFER_DESC bufferDesc = {};
    bufferDesc.BindFlags = D3D11_BIND_UNORDERED_ACCESS | D3D11_BIND_SHADER_RESOURCE;
    bufferDesc.ByteWidth = sizeof(uint32) * (MAX_UUID_COUNT + 1);
    bufferDesc.StructureByteStride = sizeof(uint32);
    bufferDesc.MiscFlags = D3D11_RESOURCE_MISC_BUFFER_STRUCTURED;
    bufferDesc.Usage = D3D11_USAGE_DEFAULT;
    bufferDesc.CPUAccessFlags = D3D11_CPU_ACCESS_READ;
    hr = Graphics->Device->CreateBuffer(&bufferDesc, nullptr, &UUIDListBuffer);
    if (FAILED(hr))
    {
        return hr;
    }

    // UAV 생성
    D3D11_UNORDERED_ACCESS_VIEW_DESC uavBufferDesc = {};
    uavBufferDesc.ViewDimension = D3D11_UAV_DIMENSION_BUFFER;
    uavBufferDesc.Format = DXGI_FORMAT_UNKNOWN;
    uavBufferDesc.Buffer.NumElements = MAX_UUID_COUNT + 1;
    hr = Graphics->Device->CreateUnorderedAccessView(UUIDListBuffer, &uavBufferDesc, &UUIDListUAV);
    if (FAILED(hr))
    {
        return hr;
    }

    //////////////////////////////////////////
    // shader 설정

    ID3DBlob* VertexShaderCSO;
    ID3DBlob* PixelShaderCSO;
    ID3DBlob* ComputeShaderCSO;
    ID3DBlob* ErrorBlob = nullptr;

    hr = D3DCompileFromFile(L"Shaders/UUIDRenderShader.hlsl", nullptr, D3D_COMPILE_STANDARD_FILE_INCLUDE, "mainVS", "vs_5_0", 0, 0, &VertexShaderCSO, &ErrorBlob);
    if (FAILED(hr))
    {
        if (ErrorBlob) {
            OutputDebugStringA((char*)ErrorBlob->GetBufferPointer());
            ErrorBlob->Release();
        }
    }
    Graphics->Device->CreateVertexShader(VertexShaderCSO->GetBufferPointer(), VertexShaderCSO->GetBufferSize(), nullptr, &UUIDVertexShader);

    hr = D3DCompileFromFile(L"Shaders/UUIDRenderShader.hlsl", nullptr, D3D_COMPILE_STANDARD_FILE_INCLUDE, "mainPS", "ps_5_0", 0, 0, &PixelShaderCSO, &ErrorBlob);
    if (FAILED(hr))
    {
        if (ErrorBlob) {
            OutputDebugStringA((char*)ErrorBlob->GetBufferPointer());
            ErrorBlob->Release();
        }
    }
    Graphics->Device->CreatePixelShader(PixelShaderCSO->GetBufferPointer(), PixelShaderCSO->GetBufferSize(), nullptr, &UUIDPixelShader);

    hr = D3DCompileFromFile(L"Shaders/UUIDRenderShader.hlsl", nullptr, D3D_COMPILE_STANDARD_FILE_INCLUDE, "mainCS", "cs_5_0", 0, 0, &ComputeShaderCSO, &ErrorBlob);
    if (FAILED(hr))
    {
        if (ErrorBlob) {
            OutputDebugStringA((char*)ErrorBlob->GetBufferPointer());
            ErrorBlob->Release();
        }
    }
    Graphics->Device->CreateComputeShader(ComputeShaderCSO->GetBufferPointer(), ComputeShaderCSO->GetBufferSize(), nullptr, &UUIDComputeShader);


    D3D11_INPUT_ELEMENT_DESC layout[] = {
        {"POSITION", 0, DXGI_FORMAT_R32G32B32_FLOAT, 0, 0, D3D11_INPUT_PER_VERTEX_DATA, 0},
    };

    Graphics->Device->CreateInputLayout(
        layout, ARRAYSIZE(layout), VertexShaderCSO->GetBufferPointer(), VertexShaderCSO->GetBufferSize(), &UUIDInputLayout
    );

    VertexShaderCSO->Release();
    PixelShaderCSO->Release();
    ComputeShaderCSO->Release();
    if(ErrorBlob) ErrorBlob->Release();

    return hr;
}

// Vertex / Pixel Shader 용
void FRenderer::PrepareRenderUUID(ID3D11DeviceContext* Context)
{
    Context->VSSetShader(UUIDVertexShader, nullptr, 0);
    Context->PSSetShader(UUIDPixelShader, nullptr, 0);
    Context->CSSetShader(UUIDComputeShader, nullptr, 0);
    Context->IASetInputLayout(UUIDInputLayout);
    Context->IASetPrimitiveTopology(D3D11_PRIMITIVE_TOPOLOGY_TRIANGLELIST);

    Context->RSSetViewports(1, &Graphics->Viewport);

    // UUID를 기록할 Texture를 UAV로 바인딩
    Graphics->DeviceContext->OMSetRenderTargetsAndUnorderedAccessViews(
        1, &Graphics->RTVs[0], Graphics->DepthStencilView, 1, 1, &UUIDTextureUAV, nullptr);
}

// Compute Shader 용
TArray<UINT> FRenderer::ReadValidUUID()
{
    // Compute Shader에서 사용할 리소스 바인딩
    Graphics->DeviceContext->CSSetShaderResources(0, 1, &UUIDTextureSRV);  // t0: UUIDTextureRead (읽기)
    Graphics->DeviceContext->CSSetUnorderedAccessViews(2, 1, &UUIDListUAV, nullptr);  // u2: UUIDList (쓰기)

    // Compute Shader 실행
    Graphics->DeviceContext->CSSetShader(UUIDComputeShader, nullptr, 0);
    Graphics->DeviceContext->Dispatch(Graphics->screenWidth / 16, Graphics->screenHeight / 16, 1);  // Thread groups 크기 설정

    // UUID 담을곳
    TArray<UINT> UUIDs;
    
    // UUIDList의 내용을 읽기 위한 Map 작업
    D3D11_MAPPED_SUBRESOURCE mappedResource;
    HRESULT hr = Graphics->DeviceContext->Map(UUIDListBuffer, 0, D3D11_MAP_READ, 0, &mappedResource);
    if (SUCCEEDED(hr))
    {
        uint32* pUUIDList = (uint32*)mappedResource.pData;

        uint32 uuidCount = pUUIDList[0];

        for (uint32 i = 1; i <= uuidCount; ++i)
        {
            uint32 uuid = pUUIDList[i];
            UUIDs.Add(uuid);
        }

        Graphics->DeviceContext->Unmap(UUIDListBuffer, 0);
    }

    // 해제하주기
    ID3D11ShaderResourceView* nullSRV[1] = { nullptr };
    ID3D11UnorderedAccessView* nullUAV[1] = { nullptr };

    Graphics->DeviceContext->CSSetShaderResources(0, 1, &nullSRV[0]);  // t0: UUIDTextureRead (읽기)
    Graphics->DeviceContext->CSSetUnorderedAccessViews(2, 1, &nullUAV[0], nullptr);  // u2: UUIDList (쓰기)
    return UUIDs;
}

```
</div>
</details>

<details>
<summary> Shader 코드 </summary>

<div markdown="1">

```c
#include "ShaderBuffers.hlsl"

// UAV 선언
RWTexture2D<uint> UUIDTextureWrite : register(u1);

struct VS_INPUT
{
    float3 position : POSITION; // 버텍스 위치
};

struct PS_INPUT
{
    float4 position : SV_Position;
};


PS_INPUT mainVS(VS_INPUT input)
{
    PS_INPUT output;
    output.position = float4(input.position, 1.0f);
    output.position = mul(output.position, WorldMatrix);
    output.position = mul(output.position, ViewMatrix);
    output.position = mul(output.position, ProjectionMatrix);
    
    return output;
}

// output color 없음. depth만 기록하고, UUID는 UUIDBuffer에 기록(이후 compute shader에서 사용)
void mainPS(PS_INPUT input)
{
    int2 coord = int2(input.position.xy);
    //UUIDTextureWrite[float2(0, 0)] = uint(123); // UAV에 UUID 저장
    //UUIDTextureWrite[int(coord.x), int(coord.y)] = uint(1);
    UUIDTextureWrite[int2(input.position.x, input.position.y)] = UUIDuint;
}

////////////////////////////////////////////////
// ComputeShader
// UUID가 저장된 UAV를 읽고 리스트를 생성
Texture2D<uint> UUIDTextureRead : register(t0);
RWStructuredBuffer<uint> UUIDList : register(u2);

[numthreads(16, 16, 1)]
void mainCS(uint3 DTid : SV_DispatchThreadID)
{
    return;
    int2 pixelPos = DTid.xy;
    uint uuid = UUIDTextureRead.Load(int3(pixelPos, 0));
    UUIDList[0] = uuid;

    if (uuid == 0)
        return; // 배경이면 무시

    for (uint i = 1; i <= UUIDList[0]; i++)
    {
        if (UUIDList[i] == uuid)
        {
            return; // 이미 존재하면 추가하지 않음
        }
    }

    uint index;
    InterlockedAdd(UUIDList[0], 1, index);
    UUIDList[index + 1] = uuid;
}

```

주의할 점은, UAV를 Pixel shader에서 접근할 수 있게 하려면 `OMSetRenderTargetsAndUnorderedAccessViews`를 써야 한다는 것이다. 이 함수를 통해서 UAV에 연결된 buffer를 원하는 슬롯에 연결한다.
</div>
</details>

---
출처
<https://d29g4g2dyqv443.cloudfront.net/sites/default/files/akamai/gamedev%2Fdocs%2FGDC_2013_DUDASH_DeferredContexts.pdf>