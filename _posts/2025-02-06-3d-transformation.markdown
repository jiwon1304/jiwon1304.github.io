---
layout: post
title:  "3D Transformation과 행렬"
categories: DirectX
tags: [directx]
---
3D 모델(object space)이 화면(clip space)에 그려지기까지 세 번의 transform이 존재한다.
`[object space]` -> `(world transform)` -> `[world space]` -> `(view transform)` -> `[camera space]` -> `(projection transform)` -> `[clip space]`  
3D 공간에서 그리므로 4차원의 Homogeneous Coordinate를 이용해서 변환한다. Direct3D에서는 어떻게 진행되는지에 대해서 간단하게 정리해보았다. 

# Transform
## World Transform
scaling, rotation, translation에 대한 행렬은 손쉽게 `DirectXMath`의 함수들을 통해서 얻을 수 있다. 다만, world transform에 해당하는 행렬을 shader로 넘길 때, 모두 곱해서 하나의 행렬을 만들어야한다. 여기서 중요한 점은 의도한 대로 곱하려면 `XMMATRIX world = scaling * rotation * translation` 순서대로 해야한다. OpenGL이랑 반대 방향이니 주의한다. 

## View Transform
여기서 view transform의 새로운 좌표축 $u,v,w$에 대한 변환 행렬은 `eye(position)`, `at(focus)`, `up(0,1,0)` vector를 통해 얻을 수 있다. Direct3D는 왼손 좌표계이니까 `LH`를 쓴다. 
```cpp
XMVECTOR Eye = XMVectorSet( 0.0f, 1.0f, -5.0f, 0.0f );
XMVECTOR At = XMVectorSet( 0.0f, 1.0f, 0.0f, 0.0f );
XMVECTOR Up = XMVectorSet( 0.0f, 1.0f, 0.0f, 0.0f );
g_View = XMMatrixLookAtLH( Eye, At, Up );
```

## Projection Transform
projection transform에 필요한 행렬은 간단하게 `XMMatrixPerspectiveFovLH`를 이용해서 만들 수 있다. `FovAngleY`는 radian이다. `AspectRatio`는 `width/height`이다. 
```cpp
XMMATRIX XM_CALLCONV XMMatrixPerspectiveFovLH(
  [in] float FovAngleY,
  [in] float AspectRatio,
  [in] float NearZ,
  [in] float FarZ
) noexcept;
```

# 행렬 연산과 데이터 구조
## row-major order / column-major order
$4\times 4$ 의 2차원 데이터를 1차원 메모리에 저장하는 방법은 두 가지가 있다. 행들을 순서대로 저장하는 방법(row-major order)와 열들을 순서대로 저장하는 방법(column-major order)이다. **데이터를 저장하는 방법일 뿐이지, 실제 계산에는 상관이 없다.** 예를 들어 다음과 같은 행렬을 1차원 메모리 주소에 저장한다고 해보자
$$
\begin{equation}
M = \begin{bmatrix}
a\ b\ c\ d\\
e\ f\ g\ h\\
i\ j\ k\ l\\
m\ n\ o\ p
\end{bmatrix}
\end{equation}
$$

row-major order의 경우에는 `a b c d e f ...` 순서대로 메모리에 저장된다. 반대로 column-major order의 경우에는 `a e i m b f ...` 순서대로 저장된다. `DirectXMath`는 row-major이고, `HLSL`은 column-major이다.  

다음 행렬 A를 메모리에 저장해보자.
$$
\begin{equation}
A = \begin{bmatrix}
1.0\quad 0.0\quad 0.0\quad 0.0\\
0.0\quad 2.0\quad 0.0\quad 0.0\\
0.0\quad 0.0\quad 4.0\quad 0.0\\
1.0\quad 2.0\quad 3.0\quad 1.0
\end{bmatrix}
\end{equation}
$$

`XMMATRIX A`를 다음과 같이 지정하면,
```cpp
XMMATRIX A(1.0f, 0.0f, 0.0f, 0.0f,
            0.0f, 2.0f, 0.0f, 0.0f,
            0.0f, 0.0f, 4.0f, 0.0f,
            1.0f, 2.0f, 3.0f, 1.0f);
```
이 행렬은 메모리 상에 다음과 같이 저장된다.
```
0x0115F7A0  00 00 80 3f 00 00 00 00 00 00 00 00 00 00 00 00  
0x0115F7B0  00 00 00 00 00 00 00 40 00 00 00 00 00 00 00 00  
0x0115F7C0  00 00 00 00 00 00 00 00 00 00 80 40 00 00 00 00  
0x0115F7D0  00 00 80 3f 00 00 00 40 00 00 40 40 00 00 80 3f  
```
32바이트 부동소수점으로 보면 다음과 같다.
```
0x0115F7A0  1.00000000    0.00000000    0.00000000    0.00000000  
0x0115F7B0  0.00000000    2.00000000    0.00000000    0.00000000  
0x0115F7C0  0.00000000    0.00000000    4.00000000    0.00000000  
0x0115F7D0  1.00000000    2.00000000    3.00000000    1.00000000  
```

## 쉐이더로 constant buffer 넘기기
`DirectXMath`와 `HLSL`은 같은 행렬을 서로 다른 순서로 저장한다. constant buffer를 통해서 전달될 때, 이러한 차이에 대한 암시적인 변환은 없기 때문에 명시적으로 transpose를 하여 데이터의 순서를 바꿔 주어야 한다. 이는 실제 수학적 계산에서의 transpose를 하는 것이 아니고, 데이터 구조만 변경되었을 뿐 실제 계산은 동일하게 이루어진다.

[그래픽 디버깅](https://develop-dream.tistory.com/149)을 통해서 GPU 메모리 내에서 어떻게 저장되는지 확인해보면 다음과 같다.
<p align="center">
  <img src="/assets/img32.png" align="center" width="90%">
</p>

메모리 공간에 transpose된 채로 저장되어 있다. column-major order인 것을 감안하면 `A[0]`는 원래의 행렬 $A$의 첫번째 열과 동일한 것을 볼 수 있다.  

## post-multiplied / pre-multiplied
데이터 구조에 대한 이야기는 잠시 접어두고, 이번에는 계산하는 방법에 대해서 알아보자.
`XMMATRIX` [문서](https://learn.microsoft.com/en-us/windows/win32/api/directxmath/ns-directxmath-xmmatrix)를 살펴보면 *pre-multiplication*이라는 말이 나온다. pre-multiplication은 벡터($\mathbf{v}$)와 행렬($M$)을 곱할 때 벡터가 왼쪽에 나온다는($\mathbf{v}\cdot M$) 것이다. 그리고 이 계산이 성립하려면 벡터($\mathbf{v}$)는 행 벡터여야 한다. 반대로 post-multiplication은 벡터를 열 벡터로 보고 행렬과의 곱은 반대 순서($M \cdot \mathbf{v}$)로 한다.  
`DirectXMath`에서 벡터는 행벡터이다. 행벡터 $\mathbf{v}$와 행렬 $M$ 사이의 곱셈은 다음과 같이 이루어진다.
$$
\begin{equation}
\mathbf v = \begin{bmatrix}1.0\quad 2.0\quad 3.0\quad 4.0\end{bmatrix},\ 
M = \begin{bmatrix}
1.0\quad 0.0\quad 0.0\quad 0.0\\
0.0\quad 2.0\quad 0.0\quad 0.0\\
0.0\quad 0.0\quad 4.0\quad 0.0\\
1.0\quad 2.0\quad 3.0\quad 1.0
\end{bmatrix}
\end{equation}
$$
$$
M = \begin{equation}
\mathbf v \cdot M = \begin{bmatrix}5.0\quad 12.0\quad 24.0\quad 4.0\end{bmatrix}
\end{equation}
$$
$\mathbf v$를 `XMVECTOR`로 메모리 상에 저장하면 다음과 같다.
```
0x0115F620  00 00 80 3f 00 00 00 40 00 00 40 40 00 00 80 40
또는
0x0115F620  1.00000000   2.00000000   3.00000000   4.00000000
```
그리고 곱셈의 결과를 메모리상에서 보면 다음과 같다.
```
0x0115F5B0  00 00 a0 40 00 00 40 41 00 00 c0 41 00 00 80 40
또는
0x0115F5B0  5.00000000   12.0000000   24.0000000    4.00000000
```
당연한 결과이다. 사실 post-multiplication와 pre-multiplication은 결국 벡터를 column vector로 볼건지, row vector로 볼건지 선택에 대한 결과물이다.   다만 여기서 주의해야 할 점은, 여러 개의 행렬을 적용해야 할 때이다. 

## 행렬곱 순서
object space에서 world space로 변환을 할 때, object space에서의 한 지점을 $v$, scaling을 $M_s$, rotation을 $M_r$, translation을 $M_t$라고 하자. 만약 $v$를 column vector라고 해석한다면 올바른(의도한) 곱 순서는 $M_t\cdot M_r\cdot M_s\cdot v$일 것이다. 하지만 $v$를 row vector라고 해석한다면 반대로 $v\cdot M_s\cdot M_r\cdot M_t$이 의도한 순서일 것이다. 따라서 world transform에서는 이러한 순서를 주의하면서 scaling, rotation, translation을 곱해야한다. 적는 순서가 어찌되었든 $v$에는 $M_s$ 다음 $M_r$ 다음 $M_t$가 적용되어야 한다.

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
보통 world transform에서는 scaling, rotation, translation을 모두 적용한채로 쉐이더에 넘기기 때문에 이를 다 적용한 행렬 $M$이 필요하다. DirectX는 $v_{row}\cdot M=v_{row}\cdot M_s\cdot M_r\cdot M_t$이고, OpenGL은 $M \cdot v_{col}=M_t\cdot M_r\cdot M_s\cdot v_{col}$이다. 따라서 이 코드에서 쉐이더에 넘기는 행렬은 각각 `g_World = rot * trans`와 `model = trans * rot`가 된다. 그리고 이는 각각 쉐이더에서 `output.Pos = mul( Pos, World )`와 `gl_Position = Model * pos`로 계산된다.  
참고로 `HLSL`에서는 벡터를 특별히 행벡터나 열벡터로 정의하진 않는다. 대신, 벡터의 위치에 따라서 적절하게 알아듣고 계산한다. 예를 들어 행렬과 행렬, 또는 행렬과 벡터 사이의 곱은 `mul()`을 이용하는데, 행렬 `M`과 벡터 `V`에 대해서 `mul(M, V)`는 `V`를 열벡터로, `mul(V, M)`는 `V`를 행벡터로 보고서 계산한다. 즉 $M\cdot V$의 계산과 $V \cdot M$의 계산을 한다. 참고로 `*`은 요소간(element-wise) 곱셈이다. 

# 왜 row-major?
왜 데이터를 row-major order로 저장할까? 이는 병렬 처리(SIMD) 시에 계산을 최적화하기 위해서이다. 벡터와 행렬의 곱셈을 당담하는 함수 `XMVector4Transform()`를 확인해보자.
```cpp
inline XMVECTOR XM_CALLCONV XMVector4Transform
(
    FXMVECTOR V,
    FXMMATRIX M
) noexcept
{
#if defined(_XM_NO_INTRINSICS_)

    float fX = (M.m[0][0] * V.vector4_f32[0]) + (M.m[1][0] * V.vector4_f32[1]) + (M.m[2][0] * V.vector4_f32[2]) + (M.m[3][0] * V.vector4_f32[3]);
    float fY = (M.m[0][1] * V.vector4_f32[0]) + (M.m[1][1] * V.vector4_f32[1]) + (M.m[2][1] * V.vector4_f32[2]) + (M.m[3][1] * V.vector4_f32[3]);
    float fZ = (M.m[0][2] * V.vector4_f32[0]) + (M.m[1][2] * V.vector4_f32[1]) + (M.m[2][2] * V.vector4_f32[2]) + (M.m[3][2] * V.vector4_f32[3]);
    float fW = (M.m[0][3] * V.vector4_f32[0]) + (M.m[1][3] * V.vector4_f32[1]) + (M.m[2][3] * V.vector4_f32[2]) + (M.m[3][3] * V.vector4_f32[3]);
    XMVECTORF32 vResult = { { { fX, fY, fZ, fW } } };
    return vResult.v;

#elif defined(_XM_ARM_NEON_INTRINSICS_)
    float32x2_t VL = vget_low_f32(V);
    XMVECTOR vResult = vmulq_lane_f32(M.r[0], VL, 0); // X
    vResult = vmlaq_lane_f32(vResult, M.r[1], VL, 1); // Y
    float32x2_t VH = vget_high_f32(V);
    vResult = vmlaq_lane_f32(vResult, M.r[2], VH, 0); // Z
    return vmlaq_lane_f32(vResult, M.r[3], VH, 1); // W
#elif defined(_XM_SSE_INTRINSICS_)
    XMVECTOR vResult = XM_PERMUTE_PS(V, _MM_SHUFFLE(3, 3, 3, 3)); // W
    vResult = _mm_mul_ps(vResult, M.r[3]);
    XMVECTOR vTemp = XM_PERMUTE_PS(V, _MM_SHUFFLE(2, 2, 2, 2)); // Z
    vResult = XM_FMADD_PS(vTemp, M.r[2], vResult);
    vTemp = XM_PERMUTE_PS(V, _MM_SHUFFLE(1, 1, 1, 1)); // Y
    vResult = XM_FMADD_PS(vTemp, M.r[1], vResult);
    vTemp = XM_PERMUTE_PS(V, _MM_SHUFFLE(0, 0, 0, 0)); // X
    vResult = XM_FMADD_PS(vTemp, M.r[0], vResult);
    return vResult;
#endif
}
```
`_XM_NO_INTRINSICS_`으로 SIMD를 사용하지 않는 계산은 직관적이다. 대신 `_XM_SSE_INTRINSICS_`으로 SIMD를 활용한 코드를 보자. `V`는 `{1.0, 2.0, 3.0, 4.0}`이고 `M`은 위의 행렬과 동일하다. 계산 과정은 다음과 같다.

1. 먼저 `XM_PERMUTE_PS`를 통해서 `V.w = 4.0`을 가져온다. `vResult = {4.0, 4.0, 4.0, 4.0}` 이다.
    ```
    XMVECTOR vResult = XM_PERMUTE_PS(V, _MM_SHUFFLE(3, 3, 3, 3)); // W
    ```

2. 다음 `_mm_mul_ps`를 통해서 네 번째 행인 `M.r[3]`과 element-wise로 곱해준다.
    ```
    vResult = _mm_mul_ps(vResult, M.r[3]);
    ```
3. 다시 `XM_PERMUTE_PS`를 통해서 `V.z = 3.0`을 가져온다. `vTemp = {3.0, 3.0, 3.0, 3.0}` 이다.
    ```
    XMVECTOR vTemp = XM_PERMUTE_PS(V, _MM_SHUFFLE(2, 2, 2, 2)); // Z
    ```

4. 이번엔 `XM_FMADD_PS`를 통해서 세 번째 행인 `M.r[2]`과 element-wise로 곱해주고, 이전의 값에 누적한다.
    ```
    vResult = XM_FMADD_PS(vTemp, M.r[2], vResult);
    ```
5. 두 번째, 첫 번째 행과도 동일한 연산을 누적하면서 진행한다.

이를 다시 표현하면 다음과 같다. ($\odot$은 element-wise multiplication)
$$
\begin{equation}
\mathbf v = \begin{bmatrix}x\quad y\quad z\quad w\end{bmatrix},\quad 
M = \begin{bmatrix}
m_{11}\quad m_{12}\quad m_{13}\quad m_{14}\\
m_{21}\quad m_{22}\quad m_{23}\quad m_{24}\\
m_{31}\quad m_{32}\quad m_{33}\quad m_{34}\\
m_{41}\quad m_{42}\quad m_{43}\quad m_{44}\\
\end{bmatrix}
\end{equation}
$$
1. $\textrm{vResult} \leftarrow (w,\  w ,\  w ,\  w)$
2. $\textrm{vResult} \leftarrow (w,\  w ,\  w ,\  w) \odot (m_{41},\  m_{42},\  m_{43},\  m_{44}) = (m_{41}w,\  m_{42}w,\  m_{43}w,\  m_{44}w)$
3. $\textrm{vTemp} \leftarrow (z,\  z ,\  z ,\  z)$
4. $\textrm{vTemp} \leftarrow (z,\  z ,\  z ,\  z) \odot (m_{31},\  m_{32},\  m_{33},\  m_{34}) = (m_{31}w,\  m_{32}w,\  m_{33}w,\  m_{34}w)$
   $\textrm{vResult} \leftarrow \textrm{vResult} + \textrm{vTemp} = (m_{31}z + m_{41}w,\  m_{32}z + m_{42}w,\  m_{33}z + m_{43}w,\  m_{34}z + m_{44}w)$

결국 예상했던 값이 값이 나온다.
$\textrm{vResult} = (m_{11}x + m_{21}y + m_{31}z + m_{41}w,\  m_{12}x + m_{22}y + m_{32}z + m_{42}w,\  m_{13}x + m_{23}y + m_{33}z + m_{43}w,\  m_{14}x + m_{24}y + m_{34}z + m_{44}w)$

다시 코드로 돌아오면, SIMD를 활용하면 행렬을 행 단위로 load해야하는데 이때 여러 곳에서 가져온다면 느려질 수 있다. 하지만 row-major order라면 4개의 요소가 연속적으로 존재하므로 메모리에서 바로 가져올 수 있다. cache와는 별 상관없어보이는게, `4byte * 16 = 64byte`이므로 $4\times 4$ 행렬이 딱 하나의 cache line에 들어간다. 

# 왜 column-major?
왜 CPU에서는 row-major를 SIMD로 잘 써먹고, GPU에서는 column-major로 바꿔서 쓸까? 인터넷을 뒤져봐도 column-major를 쓰면 result의 성분 하나하나를 내적 계산으로 효율적으로 할 수 있다라는 말밖에 없다. 그래서 `HLSL` 코드를 Nsight를 통해서 disassembly 해보았다.

다음은 `HLSL` 코드와 그의 어셈블리 코드인 `DXBC`(DirectX ByteCode)이다.
```hlsl
cbuffer ConstantBuffer : register( b0 )
{
	matrix World;
	matrix View;
	matrix Projection;
}
struct VS_OUTPUT
{
    float4 Pos : SV_POSITION;
    float4 Color : COLOR0;
};

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
<details>
<summary> DXBC 코드 접기/펼치기 </summary>

<div markdown="1">

```
//
// Generated by Microsoft (R) HLSL Shader Compiler 10.1
//
//
// Buffer Definitions: 
//
// cbuffer ConstantBuffer
// {
//
//   float4x4 World;                    // Offset:    0 Size:    64
//   float4x4 View;                     // Offset:   64 Size:    64
//   float4x4 Projection;               // Offset:  128 Size:    64
//
// }
//
//
// Resource Bindings:
//
// Name                                 Type  Format         Dim      HLSL Bind  Count
// ------------------------------ ---------- ------- ----------- -------------- ------
// ConstantBuffer                    cbuffer      NA          NA            cb0      1 
//
//
//
// Input signature:
//
// Name                 Index   Mask Register SysValue  Format   Used
// -------------------- ----- ------ -------- -------- ------- ------
// POSITION                 0   xyzw        0     NONE   float   xyzw
// COLOR                    0   xyzw        1     NONE   float   xyzw
//
//
// Output signature:
//
// Name                 Index   Mask Register SysValue  Format   Used
// -------------------- ----- ------ -------- -------- ------- ------
// SV_POSITION              0   xyzw        0      POS   float   xyzw
// COLOR                    0   xyzw        1     NONE   float   xyzw
//
0x00000000: vs_4_0
0x00000008: dcl_constantbuffer CB0[12], immediateIndexed
0x00000018: dcl_input v0.xyzw
0x00000024: dcl_input v1.xyzw
0x00000030: dcl_output_siv o0.xyzw, position
0x00000040: dcl_output o1.xyzw
0x0000004C: dcl_temps 2
0x00000054: dp4 r0.x, v0.xyzw, cb0[0].xyzw
0x00000074: dp4 r0.y, v0.xyzw, cb0[1].xyzw
0x00000094: dp4 r0.z, v0.xyzw, cb0[2].xyzw
0x000000B4: dp4 r0.w, v0.xyzw, cb0[3].xyzw
0x000000D4: dp4 r1.x, r0.xyzw, cb0[4].xyzw
0x000000F4: dp4 r1.y, r0.xyzw, cb0[5].xyzw
0x00000114: dp4 r1.z, r0.xyzw, cb0[6].xyzw
0x00000134: dp4 r1.w, r0.xyzw, cb0[7].xyzw
0x00000154: dp4 r0.x, r1.xyzw, cb0[8].xyzw
0x00000174: dp4 r0.y, r1.xyzw, cb0[9].xyzw
0x00000194: dp4 r0.z, r1.xyzw, cb0[10].xyzw
0x000001B4: dp4 r0.w, r1.xyzw, cb0[11].xyzw
0x000001D4: mov r1.xyzw, v1.xyzw
0x000001E8: mov o0.xyzw, r0.xyzw
0x000001FC: mov o1.xyzw, r1.xyzw
0x00000210: ret 
// Approximately 16 instruction slots used
```
</div>
</details>

많이 기니까 서로 해당하는 코드만 가져오면 다음과 같다.
```hlsl
output.Pos = mul( Pos, World );
```
```
0x00000054: dp4 r0.x, v0.xyzw, cb0[0].xyzw
0x00000074: dp4 r0.y, v0.xyzw, cb0[1].xyzw
0x00000094: dp4 r0.z, v0.xyzw, cb0[2].xyzw
0x000000B4: dp4 r0.w, v0.xyzw, cb0[3].xyzw
```
constant buffer인 `cb0`는 [한 element당 4개의 32비트 값을 가진다](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-constants). 따라서 `cb[0]`는 `World` 행렬의 첫번째 열이다. `dp4`는 [벡터 내적을 한다](https://learn.microsoft.com/ko-kr/windows/win32/direct3dhlsl/dp4--sm4---asm-). 따라서 GPU와 CPU는 서로 다른 방식으로 곱셈을 한다는 것을 알 수 있다. 

~~왜 다른걸까? 추측해보건대, GPU core 4개가 한번에 연산하는 것을 CPU의 SIMD 연산이랑 비교해보면, GPU core가~~

~~왜 그런걸까? 아마 CPU는 GPU에 비해 SIMD계산에 소수의 unit만 쓰기 때문에 그럴 것 같다. 다음은 행렬과 벡터를 곱할 때의 어셈블리 코드 중 일부이다.~~
```
    vResult = XM_FMADD_PS(vTemp, M.r[1], vResult);
00BF443B  mov         eax,10h  
00BF4440  shl         eax,0  
00BF4443  movaps      xmm0,xmmword ptr [vTemp]  
00BF444A  mulps       xmm0,xmmword ptr M[eax]  
00BF444F  movaps      xmmword ptr [ebp-330h],xmm0  
00BF4456  movaps      xmm0,xmmword ptr [ebp-330h]  
00BF445D  addps       xmm0,xmmword ptr [vResult]  
00BF4464  movaps      xmmword ptr [ebp-310h],xmm0  
00BF446B  movaps      xmm0,xmmword ptr [ebp-310h]  
00BF4472  movaps      xmmword ptr [vResult],xmm0  
```

~~보면 128비트=32바이트=4*`float` 의 크기를 가진 `xmm0` 레지스터 하나만 쓰고있다. 그리고 메모리 주소를 차례대로 읽고 있다. [아마 `addps`, `mulps`의 latency가 별로 안되서(=3) 하나만 쓰는 것 같다.](https://www.agner.org/optimize/instruction_tables.pdf)~~

~~왜 다르게 계산할까? 추측해보자면, CPU는 결국 하나의 processor에서만 진행하기 때문에 메모리 대역폭에 제한되지 않는다. 반면 GPU는 여러개의 processor(unit)이 있기 때문에 한 unit당 한번에 여러개의 메모리를 접근하는것은 전체적으로 보면 꽤 큰 overhead이기 때문에 메모리 대역폭이 부족해질 수 있기 때문이지 않을까... 한다.~~

의문점 : CPU도 GPU도 각각 FMADD instruction이 있는데([`VFMADD`](https://www.intel.com/content/www/us/en/docs/cpp-compiler/developer-guide-reference/2021-8/intrinsics-for-fp-fused-mult-add-fma-ops.html) / [`mad`](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/mad)) 왜 쓰지 않는걸까? 특히, GPU는 FMADD를 쓰면 CPU와 동일하게 계산할 수 있는데 왜 굳이 누적해서 계산하지 않고 하나씩 하는걸까?

정리하자면, row-major order는 SIMD 연산을 위해서이고, 행렬곱의 순서는 DirectXMath이 벡터를 행벡터로 사용하기 때문이다.(HLSL은 행이든 열이든 둘다 가능하다.) row/column-major order는 constant buffer로 넘길때 transpose한다는 것만 기억하자. row/column vector는 행렬과 곱할 때의 순서에 영향을 끼치는 것만 기억하자.


---
출처
<https://m.blog.naver.com/jungwan82/221473518603>
<https://nickchooshin.tistory.com/7>
<https://nsgg.tistory.com/353>
<https://stackoverflow.com/questions/16578765/hlsl-mul-variables-clarification>
<https://github.com/microsoft/DirectX-SDK-Samples/tree/main/C%2B%2B/Direct3D11/Tutorials/Tutorial04>
<https://develop-dream.tistory.com/149>
<https://lifeisforu.tistory.com/392>
<https://fgiesen.wordpress.com/2012/02/12/row-major-vs-column-major-row-vectors-vs-column-vectors/>