---
layout: post
title:  "3D Transformation"
categories: DirectX
tags: [directx]
---
3D 모델(object space)이 화면(clip space)에 그려지기까지 세 번의 transform이 존재한다.
`[object space]` -> `(world transform)` -> `[world space]` -> `(view transform)` -> `[camera space]` -> `(projection transform)` -> `[clip space]`  
3D 공간에서 그리므로 4차원의 Homogeneous Coordinate를 이용해서 변환한다. Direct3D에서는 어떻게 진행되는지에 대해서 간단하게 정리해보았다. 

# Transform
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
$4\times 4$ 의 2차원 데이터를 1차원 메모리에 저장하는 방법은 두 가지가 있다. 행들을 순서대로 저장하는 방법(row-major order)와 열을 순서대로 저장하는 방법(column-major order)이다. 예를 들어 다음과 같은 행렬을 1차원 메모리 주소에 저장한다고 해보자
$$
M = \begin{bmatrix}
a\ b\ c\ d\\
e\ f\ g\ h\\
i\ j\ k\ l\\
m\ n\ o\ p
\end{bmatrix}
\\
$$

row-major order의 경우에는 `a b c d e f ...` 순서대로 메모리에 저장된다. 반대로 column-major order의 경우에는 `a e i m b f ...` 순서대로 저장된다. `DirectXMath`는 row-major이고, `HLSL`은 column-major이다.  

다음 행렬 A를 메모리에 저장해보자.
$$
A = \begin{bmatrix}
1.0\ \ 0.0\ \ 0.0\ \ 0.0\\
0.0\ \ 2.0\ \ 0.0\ \ 0.0\\
0.0\ \ 0.0\ \ 4.0\ \ 0.0\\
1.0\ \ 2.0\ \ 3.0\ \ 1.0
\end{bmatrix}
\\
$$

실제 `XMMATRIX`의 구조체의 메모리 구조이다.
```cpp
XMMATRIX A(1.0f, 0.0f, 0.0f, 0.0f,
            0.0f, 2.0f, 0.0f, 0.0f,
            0.0f, 0.0f, 4.0f, 0.0f,
            1.0f, 2.0f, 3.0f, 1.0f);
```
이 행렬은 메모리 상에 다음과 같이 저장되어 있다.
```
0x0115F7A0  00 00 80 3f 00 00 00 00 00 00 00 00 00 00 00 00  
0x0115F7B0  00 00 00 00 00 00 00 40 00 00 00 00 00 00 00 00  
0x0115F7C0  00 00 00 00 00 00 00 00 00 00 80 40 00 00 00 00  
0x0115F7D0  00 00 80 3f 00 00 00 40 00 00 40 40 00 00 80 3f  
```
32바이트 부동소수점으로 보면 다음과 같다.
```
0x0115F7A0      1.00000000    0.00000000    0.00000000    0.00000000  
0x0115F7B0      0.00000000    2.00000000    0.00000000    0.00000000  
0x0115F7C0      0.00000000    0.00000000    4.00000000    0.00000000  
0x0115F7D0      1.00000000    2.00000000    3.00000000    1.00000000  
```

## 쉐이더로 constant buffer 넘기기
`DirectXMath`와 `HLSL`은 같은 행렬을 서로 다르게 저장한다. constant buffer를 통해서 전달될 때, 이러한 차이에 대한 암시적인 변환은 없기 때문에 명시적으로 transpose를 하여 데이터의 순서를 바꿔 주어야 한다. 하지만, 이는 실제 수학적 계산에서의 transpose가 발생하는 것은 아니고, 데이터 구조만 변경되었을 뿐 실제 계산은 동일하게 이루어진다.

[그래픽 디버깅](https://develop-dream.tistory.com/149)을 통해서 `HLSL` 내에서 어떻게 저장되는지 확인해보면 다음과 같다.
<p align="center">
  <img src="/assets/img32.png" align="center" width="90%">
</p>

메모리 공간에 transpose된 채로 저장되어 있다. column-major order인 것을 감안하면 `A[0]`는 원래의 행렬 $A$의 첫번째 열과 동일한 것을 볼 수 있다.

## post-multiplied / pre-multiplied
`XMMATRIX` [문서](https://learn.microsoft.com/en-us/windows/win32/api/directxmath/ns-directxmath-xmmatrix)를 살펴보면 *pre-multiplication*이라는 말이 나온다. pre-multiplication은 벡터($\mathbf{v}$)와 행렬($M$)을 곱할 때 벡터가 왼쪽에 나온다는($\mathbf{v}\cdot M$) 것이다. 그리고 이 계산이 성립하려면 벡터($\mathbf{v}$)는 행 벡터여야 한다. 반대로 post-multiplication은 벡터를 열 벡터로 보고 행렬과의 곱은 반대 순서($M \cdot \mathbf{v}$)로 이루어진다.  
`DirectXMath`와 `HLSL`에서 벡터는 행벡터이다. row-major order든 column-major order든 수학적 계산을 할 때에는 신경쓰지 않아도 된다는 것이다.
행벡터 $\mathbf{v}$와 행렬 $M$ 사이의 곱셈은 다음과 같이 이루어진다.

$$
\mathbf v = \begin{bmatrix}1.0\ \ 2.0\ \ 3.0\ \ 4.0\end{bmatrix},\ 
M = \begin{bmatrix}
1.0\ \ 0.0\ \ 0.0\ \ 0.0\\
0.0\ \ 2.0\ \ 0.0\ \ 0.0\\
0.0\ \ 0.0\ \ 4.0\ \ 0.0\\
1.0\ \ 2.0\ \ 3.0\ \ 1.0
\end{bmatrix}
\\
\\
\mathbf v \cdot M = \begin{bmatrix}5.0\ \ 12.0\ \ 24.0\ \ 4.0\end{bmatrix}
$$

## 쉐이더에서의 계산
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

DirectXMath를 이용한 행렬 $M$이 만약 HLSL로 넘어간다면 $M^T$의 상태로 넘어갈 것이다.
$$
v_{row} = \begin{bmatrix}x\ y\ z\ w\end{bmatrix}, v_{col} = \begin{bmatrix}x\\y\\z\\w\end{bmatrix}
M = \begin{bmatrix}
m_{11}\ m_{12}\ m_{13}\ m_{14}\ \\
m_{21}\ m_{22}\ m_{23}\ m_{24}\ \\
m_{31}\ m_{32}\ m_{33}\ m_{34}\ \\
m_{41}\ m_{42}\ m_{43}\ m_{44}\ \\
\end{bmatrix}
\\
\\
v_{row} \cdot M = \begin{bmatrix}(xm_{11} + ym_{21} + zm_{31} + wm_{41})\ (xm_{12}+...)\ (...)\ (...)\end{bmatrix}
\\
M \cdot v_{col} = \begin{bmatrix}xm_{11} + ym_{12} + zm_{13} + wm_{14}\\xm_{21}+...\\...\\...\end{bmatrix}
$$

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
<https://m.blog.naver.com/jungwan82/221473518603>
<https://nickchooshin.tistory.com/7>
<https://nsgg.tistory.com/353>
<https://stackoverflow.com/questions/16578765/hlsl-mul-variables-clarification>
<https://github.com/microsoft/DirectX-SDK-Samples/tree/main/C%2B%2B/Direct3D11/Tutorials/Tutorial04>