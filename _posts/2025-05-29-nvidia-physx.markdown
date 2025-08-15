---
layout: post
title:  "NVIDIA PhysX 적용기"
categories: Graphics
tags: [directx]
---
NVIDIA GPU에서 물리 연산을 처리해주는 NVIDIA PhysX sdk를 적용해본다. [Documentation](https://nvidia-omniverse.github.io/PhysX/physx/5.6.1/index.html)

# 1. 라이브러리 등록
PhysX sdk는 binary를 직접 빌드할 수 있게 하였다. 적용하려는 프로젝트의 런타임 라이브러리 설정(`/MD`, `/MT`)에 맞춰서 빌드한다.
```
PhysX_64.lib
PhysXFoundation_64.lib
PhysXExtensions_static_64.lib
PhysXPvdSDK_static_64.lib // 디버거용
```

# 2. PhysX 초기화
PhysX 라이브러리를 초기화하고, 필요한 경우엔 [PVD(PhysX Visual Debugger)](https://developer.nvidia.com/physx-visual-debugger)를 연결해준다.
```cpp
class FPhysxSolversModule
{
    ...
    // Physx
    PxDefaultAllocator      Allocator;
    PxDefaultErrorCallback  ErrorCallback;
    PxFoundation* Foundation = nullptr;
    PxPhysics* Physics = nullptr;
    PxMaterial* DefaultMaterial = nullptr;
    PxDefaultCpuDispatcher* Dispatcher = nullptr;
    PxPvd* Pvd = nullptr;
    PxPvdTransport* PvdTransport = nullptr;
};

void FPhysxSolversModule::Initialize()
{
    Foundation = PxCreateFoundation(PX_PHYSICS_VERSION, Allocator, ErrorCallback);

    // PVD (PhysX Visual Debugger) 설정
#ifdef _DEBUG
    Pvd = PxCreatePvd(*Foundation);
    PvdTransport = PxDefaultPvdSocketTransportCreate(PVD_HOST, 5425, 10);
    Pvd->connect(*PvdTransport, PxPvdInstrumentationFlag::eALL);

    Physics = PxCreatePhysics(PX_PHYSICS_VERSION, *Foundation, PxTolerancesScale(), true, Pvd);
#else
    Physics = PxCreatePhysics(PX_PHYSICS_VERSION, *Foundation, PxTolerancesScale());
#endif

    PxInitExtensions(*Physics, Pvd);
    DefaultMaterial = Physics->createMaterial(0.5f, 0.5f, 0.02f);
}
```

# 2. Physics Scene 생성
Physics scene은 물리 연산을 효율적으로 하기 위해서 원래의 메시 대신 proxy 도형을 이용해서 오브젝트를 표현한다. 월드 상에 존재할 메시들을 간단한 도형으로 대체해서 위치시킬 물리 씬을 생성한다. PVD를 이용하기 위해선 scene에 추가적인 함수를 호출해야한다.
```cpp
PxScene* FPhysxSolversModule::CreateScene()
{
    PxSceneDesc sceneDesc(Physics->getTolerancesScale());
    sceneDesc.gravity = PxVec3(0, 0, -10);
    Dispatcher = PxDefaultCpuDispatcherCreate(PX_NUM_DISPATCHER);
    sceneDesc.cpuDispatcher = Dispatcher;
    sceneDesc.filterShader = PxDefaultSimulationFilterShader;

    sceneDesc.flags |= PxSceneFlag::eENABLE_ACTIVE_ACTORS;
    sceneDesc.flags |= PxSceneFlag::eENABLE_CCD;
    sceneDesc.flags |= PxSceneFlag::eENABLE_PCM;
#if _DEBUG
    sceneDesc.flags |= PxSceneFlag::eENABLE_ENHANCED_DETERMINISM;
#endif // _DEBUG

    PxScene* Scene = Physics->createScene(sceneDesc);


#ifdef _DEBUG
    Scene->setVisualizationParameter(PxVisualizationParameter::eSCALE, 1.0f); // PVD용
    Scene->setVisualizationParameter(PxVisualizationParameter::eACTOR_AXES, 1.0f); // PVD용
    Scene->setVisualizationParameter(PxVisualizationParameter::eCOLLISION_SHAPES, 1.0f); // PVD용
    Scene->setVisualizationParameter(PxVisualizationParameter::eJOINT_LOCAL_FRAMES, 1.0f);
    Scene->setVisualizationParameter(PxVisualizationParameter::eJOINT_LIMITS, 1.0f);
    Scene->getScenePvdClient()->setScenePvdFlags(PxPvdSceneFlag::eTRANSMIT_CONSTRAINTS | PxPvdSceneFlag::eTRANSMIT_SCENEQUERIES | PxPvdSceneFlag::eTRANSMIT_CONTACTS);
#endif
    return Scene;
}
```

# 3. 강체를 Scene에 등록
물리 씬 위에 올라가는 오브젝트는 `PxActor`를 상속받는다. 우리가 처리할 강체의 경우에는 그 자식 클래스인 `PxRigidBody`를 상속받는다. 

강체의 경우에는 크게 두 가지로 나뉘는데, 씬 내부에서 움직이지 않는 것으로 처리되는(implicit infinite mass/inertia) `PxRigidStatic`와 씬 내부에서 움직이는 `PxRigidDynamic`이 존재한다. 자세한 내용은 [다음 문서](https://nvidia-omniverse.github.io/PhysX/physx/5.6.1/docs/RigidBodyOverview.html)를 참고.


```cpp
physx::PxActor* FPhysicsSolver::RegisterObject(FPhysScene* InScene, const FBodyInstance* NewInstance, const FMatrix& InitialMatrix)
{

    FVector InitialPosition = InitialMatrix.GetTranslationVector();
    FQuat InitialRotation = InitialMatrix.ToQuat();
    
    PxVec3 Position(InitialPosition.X, InitialPosition.Y, InitialPosition.Z);
    PxQuat Rotation(InitialRotation.X, InitialRotation.Y, InitialRotation.Z, InitialRotation.W);

    PxTransform InitialTransform(Position, Rotation);

    PxPhysics* Physics = FPhysxSolversModule::GetModule()->Physics;

    PxRigidActor* NewRigidActor = nullptr;

    // 게임 내에서 movable한지 여부에 따라서 생성 방법이 다름.
    switch (NewInstance->ObjectType)
    {
    case ECollisionChannel::ECC_WorldStatic:
    {
        PxRigidStatic* NewRigidStatic = Physics->createRigidStatic(InitialTransform);
        NewRigidActor = NewRigidStatic; // Static Actor로 설정
        break;
    }
    case ECollisionChannel::ECC_WorldDynamic:
    {
        PxRigidDynamic* NewDynamicActor = Physics->createRigidDynamic(InitialTransform);
        if (NewInstance->bSimulatePhysics)
        {
            // 시뮬레이션 결과를 따르게 설정
            NewDynamicActor->setRigidBodyFlag(PxRigidBodyFlag::eKINEMATIC, false);
        }
        else
        {
            // 시뮬레이션 결과를 무시하게 설정
            NewDynamicActor->setRigidBodyFlag(PxRigidBodyFlag::eKINEMATIC, true);
        }
        NewRigidActor = NewDynamicActor; // Dynamic Actor로 설정
        break;
    }
    default:
    {
        NewRigidActor = nullptr;
        assert(0);
        break;
    }
    }

    // 중력에 영향을 받는지 구분
    switch (NewInstance->bEnableGravity)
    {
    case true:
        NewRigidActor->setActorFlag(PxActorFlag::eDISABLE_GRAVITY, false);
        break;
    case false:
        NewRigidActor->setActorFlag(PxActorFlag::eDISABLE_GRAVITY, true);
        break;
    }

    NewRigidActor->setActorFlag(PxActorFlag::eVISUALIZATION, true); // PVD에서 렌더링
    NewRigidActor->setName(*NewInstance->OwnerComponent->GetName()); // 이름 설정

    NewRigidActor->userData = (void*)NewInstance; // 사용자 정의 데이터로 FBodyInstance를 설정

    const FKAggregateGeom& AggGeom = NewInstance->ExternalCollisionProfileBodySetup->AggGeom;
    float Volume = 0.f;

    for (const FKBoxElem& BoxElem : AggGeom.BoxElems)
    {
        PxBoxGeometry BoxGeometry(PxVec3(BoxElem.X, BoxElem.Y, BoxElem.Z)); // 크기
        PxShape* NewShape = Physics->createShape(BoxGeometry, *FPhysxSolversModule::GetModule()->DefaultMaterial); // 
        
        FVector Center = BoxElem.Center;
        FQuat Quat = BoxElem.Rotation.Quaternion();
        NewShape->setLocalPose(PxTransform(PxVec3(Center.X, Center.Y, Center.Z), PxQuat(Quat.X, Quat.Y, Quat.Z, Quat.W)));
        NewRigidActor->attachShape(*NewShape);
        
        Volume = Volume + BoxElem.X * BoxElem.Y * BoxElem.Z;
    }

    for (const FKSphereElem& SphereElem : AggGeom.SphereElems)
    {
        PxSphereGeometry SphereGeometry(SphereElem.Radius);
        // 등록
    }

    for (const FKSphylElem& SphylElem : AggGeom.SphylElems)
    {
        PxCapsuleGeometry CapsuleGeometry(SphylElem.Radius, SphylElem.Length / 2.f);
        // 등록
    }

    // Physics material (density 등)이 있으면 지정
    if (PxRigidDynamic* RigidDynamic = NewRigidActor->is<PxRigidDynamic>())
    {
        // 예외 처리 및 등록
        physx::PxRigidBodyExt::updateMassAndInertia(*RigidDynamic, 1);
    }

    // 물리 씬에 추가
    Scene->addActor(*NewRigidActor);

    return NewRigidActor;
}
```

# 4. Constraint를 Scene에 등록
강체와 강체 사이의 이동을 제한하는 constraint를 추가한다. Constraint의 종류는 Unreal Engine을 따라 `Linear`, `Cone`, `Twist`로 하였다. `Linear`는 x축 방향(부모-자식 joint의 방향)으로 translation을 제한한다. `Cone`과 `Twist`는 rotation을 제한하는데, `Twist`는 x축을 기준으로 한 회전을 제한하고, `Cone`은 xy평면과 xz평면에서의 회전을 제한한다. Skeletal mesh에서 하나의 physics body에 대해서 하나의 `PxActor`를 할당하였다. 
```cpp
physx::PxJoint* FPhysicsSolver::CreateJoint(FPhysScene* InScene, PxActor* Child, PxActor* Parent, const FConstraintInstance* NewInstance)
{
    PxRigidDynamic* ParentDynamic = Parent->is<PxRigidDynamic>();
    PxRigidDynamic* ChildDynamic = Child->is<PxRigidDynamic>();

   // joint space -> world space로의 trnasformation
    PxTransform parentWorld = ParentDynamic->getGlobalPose();
    PxTransform childWorld = ChildDynamic->getGlobalPose();

    // FBX파일을 파싱할 때 부모<->자식 사이의 방향이 z축으로 지정되어서
    // x축을 가정하는 PhysX에 맞게 보정해야 작동하는 것으로 보임
    PxQuat q_AxisCorrection = PxQuat(PxMat33(
        PxVec3(0.0f, 0.0f, 1.0f),
        PxVec3(1.0f, 0.0f, 0.0f), 
        PxVec3(0.0f, 1.0f, 0.0f)  
    ));
    q_AxisCorrection.normalize(); // 정규화

    PxTransform localFrameChild(PxVec3(0.0f), q_AxisCorrection);
    PxTransform childJointFrameInWorld = childWorld * localFrameChild;

    // 자식 joint space -> 부모 joint space로의 transform
    PxTransform localFrameParent = parentWorld.getInverse() * childJointFrameInWorld;

    PxD6Joint* Joint = PxD6JointCreate(*FPhysxSolversModule::GetModule()->Physics, ParentDynamic, localFrameParent, ChildDynamic, localFrameChild);

    // 3. 프로파일 속성 적용
    // 선형 제한
    if (/* 선형 이동에 대해서 lock이 없을 때 */)
    {
        // 값을 설정
        physx::PxReal extentVal = LinearLimitProps.Limit;

        physx::PxSpring spring(stiffness, damping);
        physx::PxJointLinearLimit limitParams(extentVal, spring);
    }

    // 각도 제한
    Joint->setMotion(physx::PxD6Axis::eSWING1, FPhysicsAssetUtils::MapAngularMotionToPx(ConeLimitProps.Swing1Motion));
    Joint->setMotion(physx::PxD6Axis::eSWING2, FPhysicsAssetUtils::MapAngularMotionToPx(ConeLimitProps.Swing2Motion));

    if (ConeLimitProps.Swing1Motion != EAngularConstraintMotion::ACM_Free ||
        ConeLimitProps.Swing2Motion != EAngularConstraintMotion::ACM_Free)
    {
        physx::PxReal coneStiffness = 0.0f;
        physx::PxReal coneDamping = 0.0f;

        if (ConeLimitProps.bSoftConstraint) 
        {
            coneStiffness = ConeLimitProps.Stiffness;
            coneDamping = ConeLimitProps.Damping;
        }
        physx::PxSpring spring(coneStiffness, coneDamping);

        //Swing2는 xz, Swing1은 xy평면에서의 제한임.
        physx::PxReal swing2Angle, swing1Angle;

        if (ConeLimitProps.Swing2Motion == EAngularConstraintMotion::ACM_Locked) 
        {
            swing2Angle = 0.0f;
        }
        else 
        { // ACM_Limited 또는 ACM_Free (이 경우 LimitDegrees 값 사용)
            swing2Angle = FMath::DegreesToRadians(ConeLimitProps.Swing2LimitDegrees);
        }

        // swing1도 동일하게 적용

        physx::PxJointLimitCone coneLimitParams
        (
            swing2Angle,
            swing1Angle,
            spring
        );
        Joint->setSwingLimit(coneLimitParams);
    }

    // 트위스트 제한
    Joint->setMotion(physx::PxD6Axis::eTWIST, FPhysicsAssetUtils::MapAngularMotionToPx(TwistLimitProps.TwistMotion));

    if (TwistLimitProps.TwistMotion != EAngularConstraintMotion::ACM_Free)
    {
        physx::PxReal twistStiffness = 0.0f;
        physx::PxReal twistDamping = 0.0f;
        if (TwistLimitProps.bSoftConstraint)
        {
            twistStiffness = TwistLimitProps.Stiffness;
            twistDamping = TwistLimitProps.Damping;
        }
        physx::PxSpring spring(twistStiffness, twistDamping);

        physx::PxReal halfAngleRad;
        // 값을 적당히 처리 (rad / deg)

        physx::PxJointAngularLimitPair twistLimitParams
        (
            -halfAngleRad, // 최소 각도
            halfAngleRad,  // 최대 각도
            spring          // 스프링 설정
        );

        Joint->setTwistLimit(twistLimitParams);
    }

    Joint->setConstraintFlag(physx::PxConstraintFlag::eCOLLISION_ENABLED, !Profile.bDisableCollision); 
#if _DEBUG
    Joint->setConstraintFlag(physx::PxConstraintFlag::eVISUALIZATION, true);
#endif
    return Joint;
}
```

# 5. Dispatch and Fetch
`FPBDPhysicsSolver`와 `FPBDRigidsEvolutionGBF`를 참고하였다. PhysX에서는 simulate할 때 지정한 timestep만큼 진행시킬 수 있다. 이후 Fetch할 때에는 bone의 transform을 유의하면서 새롭게 갱신한다. 참고로 Fetch에는 barrier가 걸려있어서 simulate이후 시간이 좀 걸릴 수 있다.
```cpp
class FPhysicsSolver
{
public:
    void InitScene(FPhysScene* InScene) const;

    PxActor* RegisterObject(FPhysScene* InScene, const FBodyInstance* NewInstance, const FMatrix& InitialMatrix);
    
    PxJoint* CreateJoint(FPhysScene* InScene, PxActor* Child, PxActor* Parent, const FConstraintInstance* NewInstance);

    // 물리 시뮬레이션을 특정 시간에 대해서 진행
    void AdvanceOneTimeStep(FPhysScene* InScene, float Dt);

    void FetchData(FPhysScene* InScene);

    ...
};
```

```cpp
void FPhysicsSolver::AdvanceOneTimeStep(FPhysScene* InScene, float Dt)
{
    for (int VehicleNum = 0; VehicleNum < Vehicles.Num(); VehicleNum++) 
    {
        Vehicles[VehicleNum]->StepPhysics(Dt, InScene->PhysxScene);
    }
    
    InScene->PhysxScene->simulate(Dt);
}
```

```cpp
void FPhysicsSolver::FetchData(FPhysScene* InScene)
{
    InScene->PhysxScene->fetchResults(true);

    PxScene* Scene = InScene->PhysxScene;
    // 움직일 수 있는 dynamic rigid body에 대해서만 값을 확인한다.
    PxU32 TotalActors = Scene->getNbActors(PxActorTypeFlag::eRIGID_DYNAMIC);
    PxActor** Actors = new PxActor* [TotalActors];

    PxU32 NumReturnActors = Scene->getActors(PxActorTypeFlag::eRIGID_DYNAMIC, Actors, TotalActors);

    for (PxU32 i = 0; i < NumReturnActors; ++i)
    {
        PxRigidDynamic* DynamicActor = Actors[i]->is<PxRigidDynamic>();
        // bone의 world space에서의 transform
        PxTransform Transform = DynamicActor->getGlobalPose();

        FBodyInstance* BodyInstance = static_cast<FBodyInstance*>(DynamicActor->userData);

        if (BodyInstance->OwnerComponent)
        {
            // static mesh의 경우에는 액터의 translation과 rotation만 고려하면 된다.
            if (UStaticMeshComponent* StaticMeshComp = Cast<UStaticMeshComponent>(BodyInstance->OwnerComponent))
            {
                FQuat PhysicQuat(Transform.q.x, Transform.q.y, Transform.q.z, Transform.q.w);

                BodyInstance->OwnerComponent->SetWorldTransform(
                    FTransform(
                        PhysicQuat,
                        FVector(Transform.p.x, Transform.p.y, Transform.p.z),
                        BodyInstance->OwnerComponent->GetComponentScale3D()
                    )
                );
            }
            // skeletal mesh의 경우에는 bone에 대해서 모든 정보(SQT)를 갱신해줘야 한다.
            // PxActor하나가 bone에 해당한다는 것을 유의
            else if (USkeletalMeshComponent* SkeletalMeshComp = Cast<USkeletalMeshComponent>(BodyInstance->OwnerComponent))
            {
                USkeletalMesh* SkeletalMesh = SkeletalMeshComp->GetSkeletalMeshAsset();
                int16 BoneIndex = BodyInstance->InstanceBoneIndex;
                const FReferenceSkeleton& RefSkeleton = SkeletalMesh->GetSkeleton()->GetReferenceSkeleton();
                int16 ParentIndex = RefSkeleton.GetRawRefBoneInfo()[BoneIndex].ParentIndex;

                // model space에서 부모 bone transform
                FTransform ParentComponentSpaceTransform =
                    (ParentIndex != INDEX_NONE)
                    ? SkeletalMeshComp->GetBoneComponentSpaceTransform(ParentIndex)
                    : FTransform::Identity;

                // 시뮬레이션 결과(월드 좌표계)
                FQuat PhysicQuat(Transform.q.x, Transform.q.y, Transform.q.z, Transform.q.w);
                FVector PhysicPos(Transform.p.x, Transform.p.y, Transform.p.z);

                // 스케일 유지용
                FTransform CachedBoneWorldTransform =
                    SkeletalMeshComp->GetComponentTransform() *
                    SkeletalMeshComp->GetBoneComponentSpaceTransform(BoneIndex);
                FVector OriginScale = CachedBoneWorldTransform.GetScale3D();

                // world space에서 현재 bone transform
                FTransform SimulatedWorldTransform(PhysicQuat, PhysicPos, OriginScale);

                // model space에서 현재 bone transform
                FTransform SimulatedComponentSpace =
                    SimulatedWorldTransform.GetRelativeTransform(SkeletalMeshComp->GetComponentTransform());

                // model space에서 현재 -> 부모 bone transform
                FTransform NewBoneLocal =                     SimulatedComponentSpace.GetRelativeTransform(ParentComponentSpaceTransform);

                // current pose matrix를 저장
                SkeletalMeshComp->GetBonePoseContext().Pose[BoneIndex] = NewBoneLocal;
            }
        }
    }
}
```
PhysX에서 fetch한 데이터는 world space기준 bone의 global pose transform이 나온다. 따라서 이걸 부모 bone을 기준으로 하는 local pose transform으로 변경해야 한다. 이러한 과정이 꽤나 골머리를 썩였는데, 일단은 위의 코드가 잘 작동하는 것으로 보인다. 다음은 `NewBoneLocal`을 수식으로 정리한 것이다.

$\mathbf P_{p(i)} = \mathbf P_{i-1}$라고 한다면,

$$\begin{equation}
NewBoneLocal = SimulatedComponentSpace \cdot ParentComponentSpace^{-1}
\end{equation}$$

$$\begin{equation}
\begin{split}
\mathbf P_{i \rightarrow i-1}
&=
\mathbf C_{i \rightarrow M}
{\mathbf C_{i-1 \rightarrow M}}^{-1} \\
&=
(\mathbf P_{i \rightarrow i-1} \cdots \mathbf P_{0 \rightarrow M})
(\mathbf P_{i-1 \rightarrow i-2} \cdots \mathbf P_{0 \rightarrow M})^{-1} \\
&=
(\mathbf P_{i \rightarrow i-1} \cdots \mathbf P_{0 \rightarrow M})
(\mathbf P_{M \rightarrow 0} \cdots \mathbf P_{i-2 \rightarrow i-1})
\end{split}
\end{equation}$$

`TTransform::GetRelativeTransform()`는 A * B(-1)를 계산한다. 근데 행렬 계산 기준으로는 $AB^{-1}$이지만, VQS(Vector-Quaternion-Scale) 기준으로는 $B^{-1}A$인 것으로 보인다. FTransform과 FMatrix는 [일대일 대응이 되는 체계가 아니기 때문](https://www.gradientspace.com/tutorials/2024/11/25/the-tragic-tale-of-ftransform)에 많이 헷갈렸다. 마지막에 `DEBUG_INVERSE_TRANSFORM` 매크로를 사용하는 부분을 보면 행렬 계산의 순서를 확인할 수 있다.

다음은 `TTransform::GetRelativeTransform()`의 코드 일부이다. ([github](https://github.com/EpicGames/UnrealEngine/blob/6978b63c8951e57d97048d8424a0bebd637dde1d/Engine/Source/Runtime/Core/Private/Math/Transform.cpp#L168))
<details>
<summary> TTransform::GetRelativeTransform의 코드 일부 </summary>

```cpp
template<typename T>
TTransform<T> TTransform<T>::GetRelativeTransform(const TTransform<T>& Other) const
{
	// A * B(-1) = VQS(B)(-1) (VQS (A))
	// 
	// Scale = S(A)/S(B)
	// Rotation = Q(B)(-1) * Q(A)
	// Translation = 1/S(B) *[Q(B)(-1)*(T(A)-T(B))*Q(B)]
	// where A = this, B = Other
	TTransform<T> Result;
		
	if (Other.IsRotationNormalized() == false)
	{
		return TTransform<T>::Identity;
	}

	if (Private_AnyHasNegativeScale(this->Scale3D, Other.Scale3D))
	{
		// @note, if you have 0 scale with negative, you're going to lose rotation as it can't convert back to quat
		GetRelativeTransformUsingMatrixWithScale(&Result, this, &Other);
	}
	else
	{
		// Scale = S(A)/S(B)
		
		//VQTranslation = (  ( T(A).X - T(B).X ),  ( T(A).Y - T(B).Y ), ( T(A).Z - T(B).Z), 0.f );

		// Inverse RotatedTranslation

		//Translation = 1/S(B)
	
		// Rotation = Q(B)(-1) * Q(A)	

		Result.Scale3D = VScale3D;
		Result.Translation = VTranslation;
		Result.Rotation = VRotation;

		Result.DiagnosticCheckNaN_All();
#if DEBUG_INVERSE_TRANSFORM
		TMatrix<T> AM = ToMatrixWithScale();
		TMatrix<T> BM = Other.ToMatrixWithScale();

		Result.DebugEqualMatrix(AM *  BM.InverseFast());
#endif
	}

	return Result;
}
```
</details>

# 6. 결과물
PVD와 제대로 연결했다면, 생성된 `PxScene`를 시각화하고 녹화할 수 있다.
![alt text](/assets/vid1.gif)
좌 : PhysX Visual Debugger / 우 : 결과