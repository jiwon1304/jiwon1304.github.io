---
layout: post
title:  "DirectX11 Deferred Context"
categories: DirectX
tags: [directx]
---
DirectX11에서는 멀티쓰레딩을 Deferred Context를 통해서 지원한다. 기본적으로 많이 쓰이는 `D3D11CreateDeviceAndSwapChain()`을 통해서 생성되는 `ID3D11DeviceContext`의 Immediate Context는 main thread에서만 호출할 수 있다. 따라서, 다른 쓰레드에서 호출하려면 추가적으로 `ID3D11Device::CreateDeferredContext()`를 통해서 Immediate Context가 아닌 Deferred Context를 생성해야한다.

전체적인 흐름은 다음과 같다.

```
Immediate Context 생성 -> main thread에서 Deferred Context 생성 -> worker thread에서 Deferred Context를 통한 DrawCall -> worker thread에서 DrawCall이 끝난 Deferred Context의 FinishCommandList() 호출 -> 모든 context의 FinishCommandList()가 호출되면 main thread에서 ExecuteCommandList() 호출
```

Immediate Context가 아닌 Deferred Context는 `Draw()`를 호출해도 바로 GPU로 가지 않는다. 대신 ComamndList에 쌓이고, `FinishCommandList()`를 통해서 CommandList를 완성하고, immediate context의 `ExecuteCommandList()`로 GPU에 전송된다.

다만, CommandList는 `ExecuteCommandList()`에서 전송되기 전까지 한번의 `FinishCommandList()`만 받는 것 같다. 여러번 하니까 터진다. (확인 필요)

이번 과제가 Instancing 없이 low poly(vertex 2000개) 오브젝트 5만개를 그리는 과제였는데, Batching을 안하기로 가닥을 잡으니까 최대한 많은 Drawcall을 빠르게 호출해야 했다.

다음은 직접 작성한 코드이다.

```cpp
// main thread에서 실행
void FRenderer::RenderStaticMeshes()
{
    PrepareShader();
    int NumThreadsRedering = 0;

    ID3D11CommandList* CommandList[NUM_DEFERRED_CONTEXT];
    for (auto& [Material, DataMap] : MaterialMeshMap)
    {
        // 이번에 사용하는 머티리얼을 GPU로 전달
        UpdateMaterial(Material->GetMaterialInfo());
        for (const auto& [StaticMesh, DataArray] : DataMap)
        {
            // Create a vector to store threads
            TArray<std::thread> threads;

            // Split the DataArray into chunks and process each chunk in a separate thread
            size_t chunk_size = DataArray.size() / (NUM_DEFERRED_CONTEXT-1);  // Divide by number of hardware threads
            
            if (chunk_size < 512)
            {
                chunk_size = DataArray.size();
            }
            for (size_t i = 0; i < DataArray.size(); i += chunk_size)
            {
                // Calculate the end index of the chunk
                size_t end = std::min(i + chunk_size, DataArray.size());

                // Create a lambda that processes each chunk of FMeshData
                size_t tid = i / chunk_size;
                threads.Add(std::thread([this, &DataArray, i, end, tid, Material, StaticMesh, &CommandList]()
                    {
                    RenderStaticMeshesThread(DataArray, i, end, tid, Material, StaticMesh, std::ref(CommandList[tid]));
                    }));
                NumThreadsRedering++;
            }
            // Wait for all threads to finish for the current StaticMesh
            for (auto& t : threads)
            {
                t.join();
            }
            // Command list execute
            for (int i = 0; i < threads.Num(); i++)
            {
                Graphics->DeviceContext->ExecuteCommandList(CommandList[i], true);
                CommandList[i]->Release();
                CommandList[i] = nullptr;
            }
        }
    }
}

// worker thread에서 실행
void FRenderer::RenderStaticMeshesThread(std::vector<FMeshData> DataArray, size_t i, size_t end, size_t tid,
    UMaterial* Material, const UStaticMesh* StaticMesh, 
    ID3D11CommandList* &CommandList)
{
    ID3D11DeviceContext* Context = Graphics->DeferredContexts[tid];
    PrepareShaderDeferred(Context);
    OBJ::FStaticMeshRenderData* RenderData = StaticMesh->GetRenderData();
    UINT offset = 0;
    // 버텍스 버퍼 업데이트
    Context->IASetVertexBuffers(0, 1, &RenderData->VertexBuffer, &Stride, &offset);
    if (RenderData->IndexBuffer)
    {
        Context->IASetIndexBuffer(RenderData->IndexBuffer, DXGI_FORMAT_R32_UINT, 0);
    }

    Context->VSSetConstantBuffers(0, 1, &ConstantBuffer);
    Context->VSSetConstantBuffers(5, 1, &ConstantBufferView);
    Context->VSSetConstantBuffers(6, 1, &ConstantBufferProjection);

    Context->PSSetConstantBuffers(0, 1, &ConstantBuffer);
    Context->PSSetConstantBuffers(1, 1, &MaterialConstantBuffer);

    UpdateMaterialDeferred(Context, Material->GetMaterialInfo());
    Context->RSSetViewports(1, &Graphics->Viewport);
    Context->OMSetRenderTargets(1, &QuadRTV, Graphics->DepthStencilView);

    // Process each FMeshData in the chunk
    for (size_t j = i; j < end; ++j)
    {
        const FMeshData& Data = DataArray[j];
        FMatrix MVP = Data.WorldMatrix;
        FVector4 UUIDColor = Data.EncodeUUID / 255.0f;
        UpdateConstantDeferred(Context, MVP, UUIDColor, Data.bIsSelected);

        // Draw
        Context->DrawIndexed(Data.IndexCount, Data.IndexStart, 0);

    }
    Context->FinishCommandList(FALSE, &CommandList);
}
```

현재는 Mesh별로 CommandList를 작성하고 GPU로 날리고 있지만, thread pool을 이용한다면 commandlist에 최대한 많이 넣을 수 있을 것 같다. 다만 한번에 thread에 얼마나 넘겨줘야하는지가 중요할 것이다. state를 변경하는것이 큰 overhead이므로 최대한 동일한 state를 유지하면서 thread에 일을 넘겨줘야 할것이다. 


---
출처
<https://d29g4g2dyqv443.cloudfront.net/sites/default/files/akamai/gamedev%2Fdocs%2FGDC_2013_DUDASH_DeferredContexts.pdf>