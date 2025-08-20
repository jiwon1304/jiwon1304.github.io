`UChunkManager`에서 청크 풀링을 하면서 `ATerrainChunk`를 생성하고 데이터를 담아주는 코드를 작성하는데, 생각보다 성능이 잘 나오지 않았다. 그래서 Unreal engine의 트레이스를 사용해서 프로파일링 해보았다. 

# Unreal Engine Trace

코드의 실행 시간을 추적해주는 유용한 프로파일링 도구이다. 매크로를 이용해서 구간을 정할 수 있다. 프레임타임과 비교해서 부하가 많이 걸리는 위치를 찾을 수 있다.

## 코드

트레이스는 다음과 같이 사용할 수 있다. 매크로의 파라미터는 문자열을 받을 때도 있고, 내부에서 문자열로 바꿔줄 때도 있다.

```cpp
#include "ProfilingDebugging/CpuProfilerTrace.h" // 헤더 포함
{
  ...
    TRACE_CPUPROFILER_EVENT_MANUAL_START("ATerrainChunk::GenerateMeshCPU") // 트레이스 시작.
    ProceduralMesh->CreateMeshSection_LinearColor(0, MeshData.Vertices, MeshData.Triangles, MeshData.Normals,
        MeshData.UVs, MeshData.Colors, TArray<FProcMeshTangent>(), true);
    TRACE_CPUPROFILER_EVENT_MANUAL_END() // 가장 최근에 시작된 트레이스 종료
}
// 또는
{
    TRACE_CPUPROFILER_EVENT_SCOPE(ATerrainChunk::GenerateMeshCPU);
    ProceduralMesh->CreateMeshSection_LinearColor(0, MeshData.Vertices, MeshData.Triangles, MeshData.Normals,
        MeshData.UVs, MeshData.Colors, TArray<FProcMeshTangent>(), true);
} // 스코프가 끝나면서 자동으로 종료

```

## 트레이스 시작

트레이스는 에디터 우특 하단의 버튼을 눌러서 트레이스를 시작/저장할 수 있고 저장된 데이터를 볼 수 있다.

![image.png](/assets/img44.png)

## Insights

![image.png](/assets/img45.png)

**사용법**

1. 프레임타임. 빨갛고 높으면 해당 tick이 오래걸린 것
2. 코드의 실행시간. 쓰레드별 / 스택 별로 표시되고 확대할 수 있다. (1에서 선택한 프레임은 파란색 구간으로 표시된다.)
3. 함수별 실행시간

## 오류

![image.png](/assets/img46.png)

가끔 어쩌다가 `FEngineLoop::Tick`이 계속 나오는 경우가 있다. `TRACE_CPUPROFILER_EVENT_MANUAL_END()`가 제대로 호출되지 않을 경우에 이런 현상이 나온다. if문에서 트레이스를 제대로 종료하지 않은 채 return하는지 찾아보자. 아니면 안전하게 `TRACE_CPUPROFILER_EVENT_MANUAL_START()`를 쓰자

# 프로파일링

청크 생성 과정에서 끊김이 있어서 부하가 많은 작업을 확인해야 했다. 특히 `TTask`를 사용하고 있기 때문에 멀티쓰레드에서의 문제인지도 확인해야 했다. 

청크 생성 과정은 크게 두 가지로 나누어져 있는데, 데이터 생성과 메시 생성이다. 데이터 생성은 내가 작성한 코드에서 실행되지만, 메시 생성은 `UProceduralMeshComponent`에서 진행되었기 때문에 트레이스 관련 매크로를 넣기 불편했다. 이를 위해서 동일한 내용을 가진 클래스를 하나 더 만들어서 생성 코드 사이에 매크로를 넣고 확인해보았다. 

매크로는 다음과 같이 들어갔다.

```cpp
void UChunkManager::UpdateChunksTask(const FVector& ReferenceLocation, UNoiseGenerator* NoiseGenerator)
{
    TRACE_CPUPROFILER_EVENT_MANUAL_START("UChunkManager::UpdateChunksTask::1")
    // 완료된 Task 처리
    uint32 NumTaskDone = 0;
    while (!PrepareMeshTasks.IsEmpty() && (NumMeshGeneratePerTick == 0 || NumTaskDone < NumMeshGeneratePerTick))
    {
        UE::Tasks::TTask<ATerrainChunk*> TaskDone;
        if (PrepareMeshTasks.Peek(TaskDone) && TaskDone.IsCompleted())
        {
            PrepareMeshTasks.Pop();
            ATerrainChunk* ChunkDone = TaskDone.GetResult();
            ChunkDone->GenerateMesh();
            ChunkDone->ChunkState = EChunkState::Loaded;
            ChunkDone->SetActorRelativeScale3D(FVector(VoxelSize,VoxelSize,VoxelSize));
            NumTaskDone++;
        }
        else
        {
            break;
        }
    }
    TRACE_CPUPROFILER_EVENT_MANUAL_END()

    TRACE_CPUPROFILER_EVENT_MANUAL_START("UChunkManager::UpdateChunksTask::2")
    // 1. 기준 좌표
    // 2. 필요 청크 집합 계산 (중앙부터 처리)
    TRACE_CPUPROFILER_EVENT_MANUAL_END()

    TRACE_CPUPROFILER_EVENT_MANUAL_START("UChunkManager::UpdateChunksTask::3")
    // 3. 불필요 청크 제거
    TRACE_CPUPROFILER_EVENT_MANUAL_END()

    TRACE_CPUPROFILER_EVENT_MANUAL_START("UChunkManager::UpdateChunksTask::4")

    // 4. 필요한데 없는 청크는 생성
    for (const FIntVector& Coord : RequiredChunks)
    {
        if (!ActiveChunks.Contains(Coord))
        {
            if (ATerrainChunk* NewChunk = CreateChunk(Coord, NoiseGenerator))
            {
                NewChunk->ChunkState = EChunkState::LoadingData;
                ChunksToGenerateMesh.Add(NewChunk);
            }
            else // Pool이 부족
                break;
        }
    }
    TRACE_CPUPROFILER_EVENT_MANUAL_END()

    TRACE_CPUPROFILER_EVENT_MANUAL_START("UChunkManager::UpdateChunksTask::Task")
    // TTask 기반 비동기 메시 준비
    TArray<TUniquePtr<UE::Tasks::TTask<void>>> Tasks;
    Tasks.Reserve(NumChunksToGenerateMesh);

    for (ATerrainChunk* Chunk : ChunksToGenerateMesh)
    {
        UE::Tasks::TTask<ATerrainChunk*> Task =
            UE::Tasks::Launch(UE_SOURCE_LOCATION,
                [Chunk]
                {
                    Chunk->PrepareMesh(0);
                    Chunk->ChunkState = EChunkState::LoadingMesh;
                    return Chunk;
                });
        PrepareMeshTasks.Enqueue(MoveTemp(Task));
    }
    TRACE_CPUPROFILER_EVENT_MANUAL_END()

}
```

![image.png](/assets/img47.png)

일단 프레임타임이 높게 나온 지점을 확대해서 확인해본다. 빨간색(~50ms)은 노이즈를 통해서 새로운 복셀 데이터를 생성하는 구간이다.

![image.png](/assets/img48.png)

해당 구간을 살펴보면 `NewObject`를 호출하는데 많은 시간이 걸리는 것을 볼 수 있다. 새로운 오브젝트를 생성하는 것은 game thread의 iteration에 영향을 주는 액션이므로, thread-safe하지 않다. 따라서 이는 불가피한 호출이기 때문에 queue에 넣어서 여러 프레임에 나눠서 호출해야 할 것 같다. 

다음으로 23ms 정도 나오는 구간이다.

![image.png](/assets/img49.png)

청크 업데이트가 있는 프레임에 `TTask`로 다른 쓰레드에 일을 넘기고, 뒤 프레임부터는 끝난 태스크를 게임 쓰레드에서 처리하게 된다. 현재는 프레임 당 4개의 태스크를 나누어서 계산하고 있는데, 이 값은 더 조절해야할 것으로 보인다. 일단 업데이트가 있는 프레임을 조사해보자.

가장 많이 차지하고 있는 것은 `UWorld_SendAllEndOfFrameUdpates`이다. 여기는 월드 틱이 종료하고 GC 전후로 필요한 작업들이 실행되는 구간이다. Delegate로 등록된 함수들이 여기서 broadcast된다. `FScene_AddPrimitive`가 가장 큰 문제지만 일단 내가 핸들할 수 있는 코드부터 확인해본다.