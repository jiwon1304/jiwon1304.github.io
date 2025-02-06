---
layout: post
title:  "3D Transformation"
categories: DirectX
tags: [directx]
---
3D 모델(object space)이 화면(clip space)에 그려지기까지 세 번의 transform이 존재한다.
`[object space]` -> `(world transform)` -> `[world space]` -> `(view transform)` -> `[camera space]` -> `(projection transform)` -> `[clip space]`  
3D 공간에서 그리므로 4차원의 Homogeneous Coordinate를 이용해서 변환한다. Direct3D에서는 어떻게 진행되는지에 대해서 간단하게 정리해보았다. 

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

# 행렬곱 순서
object space에서 world space로 변환을 할 때, object space에서의 한 지점을 $v$, scaling을 $M_s$, rotation을 $M_r$, translation을 $M_t$라고 하자.  
만약 $v$를 column vector라고 해석한다면 올바른(의도한) 곱 순서는 $M_t\cdot M_r\cdot M_s\cdot v$일 것이다. 하지만 $v$를 row vector라고 해석한다면 반대로 $v\cdot M_s\cdot M_r\cdot M_t$이 의도한 순서일 것이다. 따라서 world transform에서는 이러한 순서를 주의하면서 scaling, rotation, translation을 곱해야한다.

다음은 모델을 회전시키고 옮기고 싶을때,  DirectX와 OpenGL에서의 코드이다.
```cpp
// DirectX
XMMATRIX rot = XMMatrixRotationY(0.4);
XMMATRIX trans = XMMatrixTranslation(2, 0, 0);
g_World = rot * trans;

// 이후 쉐이더(HLSL)에서
output.Pos = mul( Pos, World );
```
```cpp
// OpenGL
glm::mat4 trans = glm::translate(glm::mat4(1.f), {2, 0, 0});
glm::quat q(glm::vec3(2, 0, 0));
glm::mat4 rot = glm::toMat4(q);
model = trans * rot;

// 이후 쉐이더(GLSL)에서
gl_Position = Model * pos;
```

DirectX는 계산식을 종합하면 왼쪽에 벡터가 오기 때문에 왼쪽부터 행렬을 곱해나간다. 반대로, OpenGL은 오른쪽에 벡터가 오기 때문에 오른쪽부터 행렬을 곱해나간다.

---
출처
<https://nickchooshin.tistory.com/7>