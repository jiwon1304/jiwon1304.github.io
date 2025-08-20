---
layout: post
title:  "Skeletal Mesh"
categories: Graphics
tags: [directx]
---
# FBX Parsing

`FFbxLoader` 에서 FBX Parsing 수행. `FFbxLoader::LoadFBXObject(...)` 함수에서 다음 작업을 차례대로 실행한다.

1. `Traverse(...)`: FBX 노드 트리를 순회하며 `eSkeleton` , `eMesh` 타입 노드를 리스트로 수집
2. `LoadFbxSkeleton(...)`: `eSkeleton` 노드들에 대해 Skeleton 정보 파싱
    - 각 Bone(Joint)마다 `localBindPose`와 `inverseBindPose` 행렬을 추출
3. `LoadSkinWeights(...)`: `eSkeleton` 노드들에 대해 Skin 정보 파싱
    - skin - cluster - linkedBone을 통해 `controlIndex` 별로 연결된 bone을 Map으로 추출
2. `LoadFBXMesh(...); LoadFBXMaterials(...)`: `eMesh` 노드들에 대해 Mesh, Material 정보 파싱
    1. 각 요소들의 LayerElement가 있는지 확인하고 파싱 시도
    2. vertex, normal은 MappingMode에 따라 다르게 처리
    3. 기존의 `UMaterial`을 활용하여 추출한 머터리얼 정보를 저장

## 다중 메시

보통 FBX 파일에 메시 노드는 하나지만 드물게 여러 메시 노드가 있을 수 있다. 때문에 모든 노드를 순회하며 `eMesh` 타입 노드를 리스트로 수집하고, 이 리스트에 대해 Mesh, Material 정보를 파싱하게 된다.

각 메시마다 대응되는 `UMaterial`과 `FMaterialSubset` 로 머터리얼, 서브셋을 구현한다.

## Control point / Polygon vertex

FBX에는 정점(vertex)을 두 가지 형태로 관리한다. 정점의 위치 정보만을 가진 `control point`와, 위치 정보를 제외한 다른 정보 (uv, normal 등)을 가진 `polygon vertex`로 나눠져 있다.

우리가 DirectX에 정점 정보를 넘길 때는 control point에서 위치 정보를 얻고, polygon vertex에서 다른 정보를 얻어서 이를 잘 조합해 넘겨야 한다. 따라서 파싱할 때에는 control point에 대해서 한 번 파싱하고, 이후 polygon vertex에 적혀 있는 index를 통해 control point를 접근한 후 polygon vertex의 위치를 지정해야 한다.

만약 이를 간과하고 control point와 polygon vertex가 일대일 대응한다고 여긴다면 다음과 같은 문제가 발생하게 된다.

![alt text](/assets/img40.png)

정점 `a1`과 `b1`은 동일한 위치를 공유한다. 두 정점은 동일한 control point를 가지면서 다른 uv coordinate를 가져야 한다. 처음 파싱을 구현했을 때에는 하나의 control point에 대해서 첫 번째 polygon vertex만 취했기 때문에, `a1` 정점을 누락한 채로 `b1-a2-a3`의 삼각형과 `b1-b2-b3`의 삼각형을 만들게 되었다.

그 결과 빨간색 삼각형에서 `a1`의 texcoord를 사용한게 아니라 `b1`의 texcoord를 사용하게 되었고, 가운데 사진처럼 texture에서 잘못된 매핑을 하게 되었다.

![alt text](/assets/img41.png)

이를 해결하기 위해서, 동일한 control point을 가지는 여러 polygon vertex에 대해서 정점을 각각 만들었다. 그 결과 의도했던 대로 `a1-a2-a3`와 `b1-b2-b3` 삼각형을 만들 수 있었고, 문제를 해결할 수 있었다.

# Skinning

- 3D Mesh를 posed skeleton에 붙이는 작업

## Per-Vertex Skining

- vertex는 한 개 이상의 joint에 연결될(bound) 수 있음
    - 다수의 joint에 연결 될 경우 weight을 부여
- vertex는 joint에 대해서 index와 weighting factor를 갖고 있음
    - 주로 index는 8bit를 사용. 마지막 weight는 생략될 수 있음 (합이 1이여야 하기 때문에)
    - **하지만 일부 프로그램은 합을 1로 강제하지 않음 → sum을 따로 계산해야함**

## Math

- 모든 vertex (skinned vertex를 포함)는 model space를 기준으로 특정된다.
- vertex를 bind pose(reference pose, rest pose, T-pose)에서 current pose로 변경하는 행렬을 skinning matrix라고 한다.
- **skinning matrix를 곱한다고 다른 좌표계로 이동하는 것이 아님**
    - 내부에서는 joint space와 model space를 넘나들지만, 곱해지는 대상과 결과물(vertex)은 model space를 기준으로 한다.

### Example : One-Jointed Skeleton

![img38.png](/assets/img38.png)

- 두 개의 좌표계를 이용 : Model Space($M$), Joint Space($J$)
- bind pose($B$), current pose($C$) : animation(transform) 전과 후의 포즈 (bind pose로 T-pose를 사용할 때가 많다.)

- bind pose($B$)일 때 model space($M$)에 존재하는 vertex $\mathbf v_M^B$ 에 대해서, current pose일 때 모델 좌표계에서의 새로운 위치 $\mathbf v_M^C$를 구하고 싶다.
- **joint space에서의 vertex는 joint가 어떻게 변형되든지 항상 유지된다.**
#### 이에 따라서
1. model space($M$)의 bind pose position에 위치한 vertex($\mathbf v_M^B$)를 joint space($J$)로 변환시킨다.
2. joint를 current pose로 이동시킨다. (Joint space($J$)를 이동시킨다.)
3. 다시 vertex를 model space($M$)로 변환시킨다.

#### 이를 숫자로 표현하면
1. $\mathbf v_M^B(4,6)$을 joint space에서의 위치인 $\mathbf v_j(1,3)$으로 변환한다.
    - $\mathbf B_{M \rightarrow j}$는 $T_{(-3,-3)}$이다.
    -  current pose = bind pose이므로 bind pose matrix의 inverse는 current pose matrix이다. $\mathbf C_{j \rightarrow M} = {\mathbf B_{j \rightarrow M}}^{-1} = T_{(+3,+3)}$
    - joint가 얼마나 움직이든 $\mathbf v_j=(1,3)$은 변하지 않는다. space가 움직이는 것이지, space 내부는 그대로이기 때문.
2. joint space의 좌표계를 model space의 좌표계에 대해서 $T_{(+2, +2)}$만큼 움직인다.
    - 이 때 $\mathbf v_M^C$는 $(6,8)$로 이동한다.
    - 동시에 새로운 current pose matrix $\mathbf C_{j \rightarrow M} = T_{(+3,+3)}T_{(+2,+2)} = T_{(+5,+5)}$
    - 반면 그대로 $\mathbf v_j=(1,3)$이다.
3. 여기서 $(4,6)$을 $(6,8)$로 옮기는 것이 skinning transformation이다

![img39.png](/assets/img39.png)

#### 수학으로 표현하면
- joint $j$에 대해서 bind pose matrix를 $\mathbf B_{j\rightarrow M}$으로 표현할 수 있다.
    - 이 행렬은 joint $j$의 공간($J$)에서의 벡터나 점을 model space($M$)의 벡터와 점으로 변환시키는 행렬이다.
1. $\mathbf v_j = \mathbf v_M^B \mathbf B_{M\rightarrow j} = \mathbf v_M^B (\mathbf B_{j\rightarrow M})^{-1}$ 
2. joint의 current pose matrix $\mathbf C_{j\rightarrow M}$에 대해서, $\mathbf v_M^C = \mathbf v_j \mathbf C_{j\rightarrow M}$
3.  $
    \mathbf v_M^C = \mathbf v_j \mathbf C_{j\rightarrow M} = \mathbf v_M^B (\mathbf B_{j\rightarrow M})^{-1} \mathbf C_{j\rightarrow M} = \mathbf v_M^B \mathbf K_j 
    $
    - 여기서 skinning matrix $\mathbf K_j = (\mathbf B_{j\rightarrow M})^{-1} \mathbf C_{j\rightarrow M}$ 
        

### Multijointed Skeletons

- single jointed에서의 $\mathbf B_{j\rightarrow M}$과 $\mathbf C_{j \rightarrow M}$이 하나의 joint만으로 계산이 되었다면, multi-jointed에선 joint의 heirarchy를 따라서 계산되어야함.
- joint가 여러개가 되었으니, skinning matrix $\mathbf K_j$는 각 joint $j$에 대해서 각각 구해져야 한다.
    - 이때 $\mathbf K_j$의 배열을 **matrix pallete**라고 한다.
    - 렌더할 때, 각각의 vertex는 자신이 영향받는 joint의 $\mathbf K_j$가 곱해진다.
- child joint $j$에서 그의 parent joint $p(j)$으로의 local pose의 변환을 $\mathbf P_{j \rightarrow p(j)}$라고 할 수 있다.
    - $\mathbf P_{j \rightarrow M} = \displaystyle\prod^0_{i=j} \mathbf P_{i \rightarrow p(i)}$이고, $p(0) \equiv M$이다.
- multijointed이므로, current pose는 상위 여러 개의 joint의 곱으로 이루어져 있다. 즉, local pose들의 곱으로 이루어진 global pose이다.
    - $\mathbf C_{j \rightarrow M} = \mathbf P_{j \rightarrow p(j)}\mathbf C_{p(j) \rightarrow M} = \mathbf P_{j \rightarrow p(j)}\mathbf P_{p(j) \rightarrow p(p(j))} \cdots \mathbf P_{0 \rightarrow p(0)} = \mathbf P_{j \rightarrow p(j)}\mathbf P_{p(j) \rightarrow p(p(j))} \cdots \mathbf P_{0 \rightarrow M}$
    - child joint와 parent joint사이의 transform들의 곱이다.
    - 예를 들어, right shoulder→right arm으로 변환하는 transform이 변경되면, right arm아래의 right hand도 영향을 받는다.
- 보통은 각각의 $(j, p(j))$마다 local pose $\mathbf P_{j \rightarrow p(j)}$를 먼저 계산하고, 이후 이를 이용해서 global pose $\mathbf C_{j\rightarrow M}$을 계산한다.
    - hierarchy를 잘 이용하면 중복되는 부모의 변환을 한번만 계산할 수 있다.
- current pose matrix $\mathbf C_{j \rightarrow M}$는 매 프레임 변경된다.
- 하지만 inverse bind pose matrix $(\mathbf B_{j\rightarrow M})^{-1}$는 constant하다.
    - bind pose 상태일 때 model space에서 joint space로 좌표값(벡터, 점)을 변환시키는 행렬
    - bind pose는 애니메이션과 상관없이 고정되어있는 포즈이다. 따라서 행렬도 고정이다.
    - **따라서 생성시에 cache한다.**

- 애니메이션 과정에서 $\mathbf C_{j\rightarrow M}$를 매 프레임 계산하면서, $\mathbf K_j = (\mathbf B_{j\rightarrow M})^{-1} \mathbf C_{j\rightarrow M}$를 매 프레임 vertex에 곱해준다.

## 이번 코드와의 비교

| 항목 | 기호 | 저장 위치 |
| --- | --- | --- |
| Local bind pose | $\mathbf C_{j\rightarrow p(j)}$  | `USkeletalMesh::RefSkeleton::RawRefBonePose` (파싱 시)
`USkeletalMeshComponent::OverrideSkinningTransform` (포즈 변경 시) |
| Inverse bind pose | $(\mathbf B_{j\rightarrow M})^{-1}$ | `USkeletalMesh::InverseBindPoseMatrix` |
| Global bind pose | $\mathbf C_{j\rightarrow M}$  | 매 프레임 직접 계산 |
| Vertex | $\mathbf v_M^B$ | `USkeletalMesh::RenderData::RenderSections`  |
| Skinning matrix | $\mathbf K_j$  | `USkeletalMeshComponent::GetSkinningMatrices()` 에서 매 프레임 계산 |

# Code

```cpp
void FSkeletalMeshRenderPass::RenderAllSkeletalMeshes(const std::shared_ptr<FEditorViewportClient>& Viewport)
{
    for (USkeletalMeshComponent* SkeletalMeshComponent : SkeletalMeshComponents)
    {
        if (!SkeletalMeshComponent) continue;
        if (!SkeletalMeshComponent->GetSkeletalMesh()) continue;

        USkeletalMesh* SkeletalMesh = SkeletalMeshComponent->GetSkeletalMesh();
        if (!SkeletalMesh) continue;
        const FSkeletalMeshRenderData& Renderdata = SkeletalMesh->GetRenderData();

        // Bone Matrix는 CPU에서 처리
        // Model -> j -> transform -> model space로 변환하는 행렬
        // 즉, transform을 적용해주는 행렬
        TArray<FMatrix> SkinningMatrices;
        SkeletalMeshComponent->GetSkinningMatrices(SkinningMatrices);

        // 공통으로 사용하는 constant buffer를 업데이트
        UpdateObjectConstant(
            SkeletalMeshComponent->GetWorldMatrix(),
            SkeletalMeshComponent->EncodeUUID() / 255.0f,
            SkeletalMeshComponent->IsActive(),
            bIsCPUSkinning
        );

        // 각각의 section에 대해서 렌더
        for (int SectionIndex = 0; SectionIndex < Renderdata.RenderSections.Num(); ++SectionIndex)
        {
            const FSkelMeshRenderSection& RenderSection = Renderdata.RenderSections[SectionIndex];
            FVertexInfo VertexInfo;
            // CPU skinning일 경우
            if (SkeletalMesh->bCPUSkinned)
            {
                // Update vertex buffer
                TArray<FSkeletalVertex> Vertices;
                GetSkinnedVertices(SkeletalMesh, SectionIndex, SkinningMatrices, Vertices);

                // 매 프레임 vertex buffer를 생성
                BufferManager->CreateDynamicVertexBuffer(RenderSection.Name, Vertices, VertexInfo);
                BufferManager->UpdateDynamicVertexBuffer(RenderSection.Name, Vertices);
            }
            else
            {
                // constant buffer로 skinning matrices를 넘김
                UpdateBoneMatrices(SkinningMatrices);
                
                // vertex buffer를 로드
                BufferManager->CreateVertexBuffer(RenderSection.Name,
                    RenderSection.Vertices, VertexInfo);
            }

            ...
            // DrawIndexed 호출
            Graphics->DeviceContext->DrawIndexed(IndexCount, StartIndex, 0);
            ...
        }
    }
}

```
## CPU Skinning

```cpp
    ...
    if (SkeletalMesh->bCPUSkinned)
    {
        // Update vertex buffer
        TArray<FSkeletalVertex> Vertices;
        GetSkinnedVertices(SkeletalMesh, SectionIndex, SkinningMatrices, Vertices);

        BufferManager->CreateDynamicVertexBuffer(RenderSection.Name, Vertices, VertexInfo);
        BufferManager->UpdateDynamicVertexBuffer(RenderSection.Name, Vertices);
    }
    ...
```

```cpp
void FSkeletalMeshRenderPass::GetSkinnedVertices(USkeletalMesh* SkeletalMesh, uint32 Section, const TArray<FMatrix>& BoneMatrices, TArray<FSkeletalVertex>& OutVertices) const
{
    const FSkelMeshRenderSection& RenderSection = SkeletalMesh->GetRenderData().RenderSections[Section];
    const TArray<FSkeletalVertex>& Vertices = RenderSection.Vertices;
    OutVertices.SetNum(Vertices.Num());
    for (int i = 0; i < Vertices.Num(); ++i)
    {
        const FSkeletalVertex& Vertex = Vertices[i];
        FSkeletalVertex& SkinnedVertex = OutVertices[i];
        FVector4 SkinnedPosition(0, 0, 0, 0);
        FVector4 SkinnedNormal(0, 0, 0, 0);
        FVector4 SkinnedTangent(0, 0, 0, 0);
        for (int j = 0; j < 8; ++j)
        {
            if (Vertex.BoneWeights[j] > 0.0f)
            {
                const FMatrix& BoneMatrix = BoneMatrices[Vertex.BoneIndices[j]];
                SkinnedPosition += BoneMatrix.TransformPosition(Vertex.Position) * Vertex.BoneWeights[j];
                SkinnedNormal += BoneMatrix.TransformFVector4(FVector4(Vertex.Normal, 0)) * Vertex.BoneWeights[j];
                SkinnedTangent += BoneMatrix.TransformFVector4(Vertex.Tangent) * Vertex.BoneWeights[j];
            }
        }
        SkinnedVertex.Position = SkinnedPosition.Pos();
        SkinnedVertex.Normal = SkinnedNormal.Pos();
        SkinnedVertex.Tangent = SkinnedTangent.Pos();
        SkinnedVertex.Color = Vertex.Color;
        SkinnedVertex.UV = Vertex.UV;
    }
}
```

매 프레임 각 `D3D11_USAGE_DYNAMIC`형태의 vertex buffer를 생성한다. CPU에서 skinning이 되어있는 vertex가 매 프레임 업데이트되어서 GPU에 올라간다. 

## GPU Skinning
```c
cbuffer BoneMatrices : register(b11)
{
    row_major matrix BoneMatrixArray[64]; // Maximum of 128 bones, adjust as needed
}

struct VS_INPUT_SkeletalMesh
{
    float3 Position : POSITION;
    float4 Color : COLOR;
    float3 Normal : NORMAL;
    float4 Tangent : TANGENT;
    float2 UV : TEXCOORD;
    uint MaterialIndex : MATERIAL_INDEX;
    int4 BoneIndices0 : BONE_INDICES0;
    int4 BoneIndices1 : BONE_INDICES1;
    float4 BoneWeights0 : BONE_WEIGHTS0;
    float4 BoneWeights1 : BONE_WEIGHTS1;
};


PS_INPUT_SkeletalMesh mainVS(VS_INPUT_SkeletalMesh Input, uint id: SV_VertexID)
{
    PS_INPUT_SkeletalMesh Output;

    // Skinning: Transform vertex position and normal with bone matrices
    float4 SkinnedPosition = float4(0.0, 0.0, 0.0, 0.0);
    float3 SkinnedNormal = float3(0.0, 0.0, 0.0);
    float4 SkinnedTangent = float4(0.0, 0.0, 0.0, 0.0);
    if (!bCPUSkinning)
    {
        for (int i = 0; i < 4; i++)
        {
            if (Input.BoneWeights0[i] > 0.0f)
            {
                matrix BoneMatrix = BoneMatrixArray[Input.BoneIndices0[i]];
                SkinnedPosition += mul(float4(Input.Position, 1.0), BoneMatrix) * Input.BoneWeights0[i];
                SkinnedNormal += mul(Input.Normal, (float3x3) BoneMatrix) * Input.BoneWeights0[i];
                SkinnedTangent.xyz += mul(Input.Tangent.xyz, (float3x3) BoneMatrix) * Input.BoneWeights0[i];
            }
        }
        for (int i = 0; i < 4; i++)
        {
            if (Input.BoneWeights1[i] > 0.0f)
            {
                matrix BoneMatrix = BoneMatrixArray[Input.BoneIndices1[i]];
                SkinnedPosition += mul(float4(Input.Position, 1.0), BoneMatrix) * Input.BoneWeights1[i];
                SkinnedNormal += mul(Input.Normal, (float3x3) BoneMatrix) * Input.BoneWeights1[i];
                SkinnedTangent.xyz += mul(Input.Tangent.xyz, (float3x3) BoneMatrix) * Input.BoneWeights1[i];
            }
        }
        //SkinnedPosition = float4(Input.Position, 1) + float4(0, 0, 0, 0);

        SkinnedNormal = normalize(SkinnedNormal);
        SkinnedTangent.xyz = normalize(SkinnedTangent.xyz);
        SkinnedTangent.w = Input.Tangent.w;

    }
    ...
    // MVP 계산
}
```
constant buffer로 bone matrices를 넘긴다. bone의 개수와 무관하게 constant buffer의 크기는 런타임에 변경을 하지 못하기 때문에 bone matrices의 개수를 적절한 수로 제한을 한다. 또한 메시 중에서 bone weight가 4개 이상인 경우가 있었기에, 이를 고려해서 bone weight를 8개까지 받도록 한다.

### 잘못된 Bone weight 적용
만약 weight의 합이 1이 아닐 경우에는 $1-\sum w_i$만큼 원점에 weight가 적용된 것으로 계산이 된다. 따라서 몇몇 vertex가 원점으로 늘어나게 된다. 내 경우에는 bone weight가 5개 이상인데 4개만 처리해버렸을 때 다음과 같은 현상이 발생했다. 다리 사이로 몸통의 vertex가 길게 늘어나있다.
![](/assets/img50.png) 

## Performance
![alt text](/assets/img42.png)
CPU Skinnning에 비해 GPU Skinning는 두 배 정도의 성능 향상을 보여줬는데, 이는 병렬 계산의 이점이라기 보다는 CPU skinned된 vertex buffer를 매번 생성하는 과정에서 생긴 오버헤드라고 볼 수 있다.

# 번외 - Async Loading
애니메이션을 파싱할 때, 구현상의 문제로 실제 transform 정보를 그대로 가져오는 것이 아니라 매 프레임 transform을 evaluate해서 정보를 가져왔다. 이로 인해서 초기 로드 속도가 1분이 넘어가기 시작했고, iteration cost를 줄이기 위해서 메시를 로드하는 것은 백그라운드로 넘기고 엔진을 시작하게 만들었다.

FBX SDK는 기본적으로 thread safety를 보장하지 않는다. 따라서 하나의 worker 쓰레드를 생성한 후 그 쓰레드에서만 FBX SDK의 함수를 실행하도록 했다. 또한 데이터를 가공하는 쓰레드와, 바이너리로 저장하는 쓰레드를 생성하여 총 3개의 쓰레드가 FBX 파싱을 작업을 진행하도록 했다. 

쓰레드는 프로그램 시작시 생성되고, 프로그램이 끝날 때 까지 while문을 돌면서 작업을 하게 된다. busy-wait를 피하기 위해서 conditional variable을 이용해서 새로운 메시가 추가되기 전까지는 block을 하도록 했다. 또한 spinlock을 사용해서 map에 대한 thread-safety를 지켰는데, 아마 접근하는 thread가 제한되어 있어서 의미가 없었을 듯 하다.

```cpp
// 프로그램 시작 시에 쓰레드를 생성
void FFbxManager::Init()
{
    FFbxLoader::Init();
    LoadThread = std::thread(&FFbxManager::LoadFunc);
    SaveThread = std::thread(&FFbxManager::SaveFunc);
    ConvertThread = std::thread(&FFbxManager::ConvertFunc);

    LoadThread.detach();
    SaveThread.detach();
    ConvertThread.detach();
}


// 파싱이 끝난 FBX 파일을 사용할 수 있게 가공하는 함수
void FFbxManager::ConvertFunc()
{
    while (!bStopThread)
    {
        // 이전 단계(파싱)의 쓰레드에서 notify하기 전까지 conditional variable을 기다림 
        std::unique_lock<std::mutex> Lock(ConvertMutex);
        ConvertCondition.wait(Lock, [&] { 
            return !PriorityConvertQueue.IsEmpty() || !ConvertQueue.IsEmpty() || bStopThread;
            });

        while (!PriorityConvertQueue.IsEmpty() || !ConvertQueue.IsEmpty())
        {
            if (bStopThread)
                break;
            bool IsPriority = !PriorityConvertQueue.IsEmpty();
            TQueue<FString>& Queue = IsPriority ? PriorityConvertQueue : ConvertQueue;
            FString FileName;
            if (Queue.Dequeue(FileName))
            {
                // Dequeue된 FileName에 대해서 파싱을 진행 

                // Spinlock을 이용해서 Map에 애니메이션(과 메시)을 추가.
    
                // 로드가 끝났다고 delegate를 통해 broadcast
                OnLoadFBXCompleted.Execute(FileName);
                UE_LOG(ELogLevel::Display, TEXT("Converted FBX file: %s"), *FileName);

                // 다음 단계에 전달하고 알림
                SaveQueue.Enqueue(FileName);
                SaveCondition.notify_one();
            }
        }
    }
    // 만약 프로그램 종료를 위해서 while문을 탈출하게 된다면, main thread에 반복문이 정상적으로 종료되었음을 notify하고 쓰레드를 종료함
    {
        std::lock_guard<std::mutex> Lock(ConvertTerminatedMutex);
        bConvertThreadTerminated = true;
        ConvertTerminatedCondition.notify_one();
    }
}
```