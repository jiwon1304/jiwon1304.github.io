---
layout: post
title:  "DirectXMath"
categories: DirectX
tags: [directx]
---
> 출처 : <https://learn.microsoft.com/en-us/windows/win32/dxmath/directxmath-portal>

# DirectXMath
DirectXMath는 이전의 XNA Math Library를 대체하는 새로운 수학 라이브러리이다. VS2012이후 기본적으로 Windows SDK에 포함되어있어서 `#include <DirectXMath.h>`만을 통해서 사용가능하다. `XMVECTOR`는 SIMD 하드웨어 register(`__m128`)의 proxy라고 보면 된다. 데이터를 저장할 땐 16비트에 맞춰서 메모리에 정렬되어야하는데, 64비트 프로그램에서는 자동으로 정렬된다. 반면, 32비트 프로그램의 경우에는 heap에서 8비트 단위로 정렬되기 때문에 직접 메모리를 정렬해야한다. 

## 사용법
1. 상수 값으로 사용하려면 `XMVECTORF32`를 이용한다. 변수로 이용하려면 `XMVectorSet`를 이용한다. 
```cpp
static const XMVECTORF32 vFactors = { 1.0f, 2.0f, 3.0f, 4.0f };
XMVECTOR v = XMVectorSet( fx, fy, fz, fw );
```

2. `XMVECTOR`의 일부 값만 변경해서 다른 `XMVECTOR`에 대입할 땐 `XMVectorSet_`를 이용한다.
```cpp
// v1의 w값만 fw로 변경해서 v2에 대입
XMVECTOR v2 = XMVectorSetW( v1, fw );
```

3. `XMVECTOR`를 다른 값으로 초기화할 때에는 주어진 함수를 잘 이용하자
```cpp
XMVectorReplicate
XMVectorSetW
XMVectorSplatZ
XMVectorSwizzle
XMVectorPermute
XMVectorReplicatePtr
```

4. SIMD register에서 데이터 전송은 매우 느리므로, 가급적 최소화하는게 좋다. 계산 값이 스칼라(내적, 길이 등)로 나오더라도, 함수는 모든 요소가 같은 벡터를 반환한다. 웬만하면 모든 계산을 벡터 형식으로 진행하고, 처음과 마지막의 load/store만 float를 이용하는 것이 좋다.
```cpp
// X값만 가져올때
float f = XMVectorGetX( v );
// 모든 값을 가져올때
XMFLOAT4A t;
XMStoreFloat4A( &t, v );
```

5. align되어야 SIMD를 잘 활용할 수 있기에, 가능하면 함수든 struct이든 뒤에 접미사로 `A`가 붙은 타입을 써야한다. 클래스 멤버에서는 가급적 `XMVECTOR`보다 `XMFLOATn`을 쓰는 것이 좋다. 

6. [컴파일 옵션은 `/fp:fast`가 추천된다.](https://learn.microsoft.com/en-us/windows/win32/dxmath/pg-xnamath-optimizing#use-correct-compilation-settings) Visual Studio에서 프로젝트의 속성에 들어가 `구성 속성 > C/C++ > 코드 생성 > 부동 소수점 모델`에서 변경 가능하다. 


7. calling convention으로 constructor를 제외한 함수에는 `XM_CALLCONV`를 사용한다. 또한 파라미터로 `XMVECTOR`나 `XMMATRIX`를 넘길 땐 이의 alias를 통해서 [넘겨야 한다](https://learn.microsoft.com/en-us/windows/win32/dxmath/pg-xnamath-internals#calling-conventions). return value는 alias 없이 `XMVECTOR`나 `XMMATRIX`를 사용한다.
- `XMVECTOR`를 여러개 넘길때는 다음과 같은 규칙을 갖는다.
   - 첫 3개는 `FXMVECTOR`를 사용한다.
   - 4번째는 `GXMVECTOR`를 사용한다.
   - 5, 6번째는 `HXMVECTOR`를 사용한다.
   - 그 이상은 `CXMVECTOR`를 사용한다.
   - 단, constructor의 경우에서는 4번째 이상의 파라미터부터 `CXMVECTOR`를 사용한다.
- `XMMATRIX`는 첫 파라미터는 `FXMMATRIX`로, 두번째 파라미터부턴 `CXMMATRIX`로 한다.
  - 단, constructor의 경우에는 모두 `CXMMATRIX`를 쓴다.  

예시는 Library 내 함수를 참고하자.
```cpp
XMMATRIX XM_CALLCONV XMMatrixLookAtLH(FXMVECTOR EyePosition, FXMVECTOR FocusPosition, FXMVECTOR UpDirection);
XMMATRIX XM_CALLCONV XMMatrixTransformation2D(FXMVECTOR ScalingOrigin,  float ScalingOrientation, FXMVECTOR Scaling, FXMVECTOR RotationOrigin, float Rotation, GXMVECTOR Translation);
void XM_CALLCONV XMVectorSinCos(XMVECTOR* pSin, XMVECTOR* pCos, FXMVECTOR V);
XMVECTOR XM_CALLCONV XMVectorHermiteV(FXMVECTOR Position0, FXMVECTOR Tangent0, FXMVECTOR Position1, GXMVECTOR Tangent1, HXMVECTOR T);
XMMATRIX(FXMVECTOR R0, FXMVECTOR R1, FXMVECTOR R2, CXMVECTOR R3)
XMVECTOR XM_CALLCONV XMVector2Transform(FXMVECTOR V, FXMMATRIX M);
XMMATRIX XM_CALLCONV XMMatrixMultiplyTranspose(FXMMATRIX M1, CXMMATRIX M2);
```

그 외에도 코드 최적화는 [MS 문서](https://learn.microsoft.com/en-us/windows/win32/dxmath/ovw-xnamath-progguide)를 참고하면 좋다.