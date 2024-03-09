---
layout: post
title:  "Trinagle Strip으로 지형 만들기"
categories: OpenGL
tags: [opengl, terrain]
---

게임과 같은 3D 공간에서 지형을 그릴 때 Y축 높이의 값인 heightmap을 통해서 지형을 생성할 수 있다. heightmap에 저장된 값을 x,z좌표에 적절히 대응시키면 지형의 각 좌표를 얻을 수 있다. 이를 화면에 띄우기 위해서는 각 좌표가 삼각형의 꼭짓점이 되도록 그려야 한다. 예를 들어서 좌표 (0,0), (1,0), (0,1), (1,1)에는 다음과 같이 삼각형을 그려서 사각형 지형을 그릴 수 있다.
```
A(0,0) --- B(0,1)
  |   \      |
  |     \    |
  |       \  |
C(1,0) --- D(1,1)
```
이 사각형을 그리기 위해서는 삼각형 ADC, ABD를 그려야 한다. index를 이용해서 그린다면 { A, D, C, A, B, D }가 된다. 하지만 [triangle strip](https://en.wikipedia.org/wiki/Triangle_strip)을 쓰면 4개의 index = { A, D, C, B }를 이용해서 표현할 수 있다.\
각 꼭짓점의 (x,z)좌표를 기준으로 2차원 배열을 만들고, y값을 value로 한다고 생각해보자. 우리는 row방향(x 방향)으로 하나의 긴 사각형을 그려 나갈 수 있다.
```
A(0,0) --- B(0,1) --- E(0,2) --- G(0,3)
   |  \       |  \       |  \       |
   |    \     |    \     |    \     |   ...
   |      \   |      \   |      \   |
C(1,0) --- D(1,1) --- F(1,2) --- H(1,3)
```
이런 방법으로 연속해서 나아가면 하나의 row를 완성시킬 수 있다. 하지만 다음 column으로 넘어갈 때 문제가 발생한다. 바로 다음 column으로 넘어가자니 전체 배열을 가로지르는 삼각형이 생기기 때문이다. 이를 해결하기 위해서는 다음 row로 넘어갈때 때 삼각형을 그리지 않으면서 넘어가야 한다.\
꼭짓점은 있는데 삼각형을 그리지 않으려면 삼각형이 나오지 않는 꼭짓점을 선택하면 된다. 고등학교 지식으로 돌아가자면, 세 점이 같은 직선 위에 존재한다면 삼각형이 존재할 수 없다. 따라서, 삼각형을 구성할 꼭짓점을 선택할 때 동일한 꼭짓점을 중복으로 선택하면 삼각형 대신 선이 그려지게 된다. 선의 경우에는 rasterization과정에서 sampling 되지 않을 것이므로 사라진 꼴이 된 것이다. \
\
![img](https://www.learnopengles.com/wordpress/wp-content/uploads/2012/05/ibo_with_degenerate_triangles.png)\
[이 링크](https://www.learnopengles.com/tag/triangle-strips/)에 나와있는 대로 row 끝에서 index를 {..., 9, 5, 10, 10, 6, 6, 11, 7, ...}로 하게 된다면 {5, 10, 10}, {10, 10, 6}, {10, 6, 6}, {6, 6, 11}에 해당하는 삼각형은 사라지게(degenerate) 된다. 따라서 이러한 방법으로 하나씩 (x,z)좌표를 선택하게 된다면 `glDrawElements(GL_TRIANGLE_STRIP, ...)`으로 한번에 지형을 그릴 수 있다.


```cpp
m_indices = new unsigned int[(2*m_numX+2)*(m_numZ-1)-3];
unsigned int j = 0;
for(unsigned int z = 0; z < m_numZ - 2; z++){ // except last row triangles
    for(unsigned int x = 0; x < m_numX; x++){
        m_indices[j++] = z*m_numX + x;
        m_indices[j++] = (z+1)*m_numX + x;
    }
    // degenerated triangles
    m_indices[j++] = z*m_numX + (m_numX-1) + m_numX;
    m_indices[j++] = (z+1)*m_numX;
}
// last row; no degeneration
unsigned int z = m_numZ - 2;
for(unsigned int x = 0; x < m_numX; x++){
    m_indices[j++] = z*m_numX + x;
    m_indices[j++] = (z+1)*m_numX + x;
}
```

이를 `glPolygonMode(GL_FRONT_AND_BACK,GL_LINE)`를 통해서 보면 다음과 같다.
![img1](/assets/img1.png)

heightmap에 무작위 값을 넣고 다시 확인해본다.
![img2](/assets/img2.png)

row를 가로지르는 선과 모서리 부분에 의도하지 않은 선이 몇개 보인다. 이 선은 index가 겹치게 되면서 삼각형이 아니라 선분으로 그려진 도형에 해당한다. 

이후 heightmap을 잘 이용해서 각 vertex에 정보를 넣으면 사각형 모양의 지형을 그릴 수 있다.

---
참고: [LearnOpenGLES](https://www.learnopengles.com/tag/triangle-strips/)