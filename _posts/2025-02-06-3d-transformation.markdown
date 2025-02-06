---
layout: post
title:  "3D Transformation"
categories: DirectX
tags: [directx]
---
3D 모델(object space)이 화면(clip space)에 그려지기까지 세 번의 transform이 존재한다.
`[object space]` -> `(world transform)` -> `[world space]` -> `(view transform)` -> `[camera space]` -> `(projection transform)` -> `[clip space]`

3D 공간에서 그리므로 4차원의 Homogeneous Coordinate를 이용해서 변환한다.

# View Transform
여기서 view transform의 새로운 좌표축 $u,v,w$에 대한 변환 행렬은 `eye(position)`, `at(focus)`, `up(0,1,0)` vector를 통해 얻을 수 있다. Direct3D는 왼손 좌표계이니까 `LH`를 쓴다. 
```cpp
XMVECTOR Eye = XMVectorSet( 0.0f, 1.0f, -5.0f, 0.0f );
XMVECTOR At = XMVectorSet( 0.0f, 1.0f, 0.0f, 0.0f );
XMVECTOR Up = XMVectorSet( 0.0f, 1.0f, 0.0f, 0.0f );
g_View = XMMatrixLookAtLH( Eye, At, Up );
```

# Projection Transform
projection transform에 필요한 행렬은 간단하게 `XMMatrixPerspectiveFovLH`를 이용해서 만들 수 있다. `FovAngleY`는 radian이다. `AspectRatio`는 `width/height`이다. 
```cpp
XMMATRIX XM_CALLCONV XMMatrixPerspectiveFovLH(
  [in] float FovAngleY,
  [in] float AspectRatio,
  [in] float NearZ,
  [in] float FarZ
) noexcept;
```

# 쉐이더로 constant buffer 넘기기
DirectXMath는 행렬을 row-major order로 저장한다. 즉, 같은 열의 요소는 메모리상에서 연속해서 존재한다. 반면 쉐이더 언어인 HLSL에서는 행렬을 column-major order로 저장한다. constant buffer를 통해서 전달될 때, 이러한 고려는 없기 때문에 transpose를 통해서 데이터의 위치를 바꿔 주어야 한다. 실제 수학적 계산에서의 transpose가 발생하는 것은 아니다.

# 쉐이더에서의 계산
```hlsl
cbuffer ConstantBuffer : register( b0 )
{
	matrix World;
	matrix View;
	matrix Projection;
}

VS_OUTPUT VS( float4 Pos : POSITION, float4 Color : COLOR )
{
    VS_OUTPUT output = (VS_OUTPUT)0;
    output.Pos = mul( Pos, World );
    output.Pos = mul( output.Pos, View );
    output.Pos = mul( output.Pos, Projection );
    output.Color = Color;
    return output;
}
```
쉐이더로 입력된 `float4 Pos`는 row vector로 처리된다. `mul(A,B)`는 수학적으로 $AB$의 연산을 제공하는데, `Pos`가 $1 \times 4$ 행렬(=벡터)이고, `World`가 $4 \times 4$ 행렬이니 `mul(Pos, World)`는 $1 \times 4$ 행렬을 반환하게 된다.  
벡터를 column vector로 인식하는 OpenGL에서의 `World * vec4(pos, 1.f)`와 대비된다. 물론, 결과값은 동일하다.

---
출처
<https://nickchooshin.tistory.com/7>