---
layout: post
title:  "랜덤 terrain만들기"
categories: OpenGL
---

게임과 같은 3D 공간에서 지형을 그릴 때 Y축 높이의 값인 heightmap을 통해서 지형을 생성할 수 있다. 이를 화면에 띄우기 위해서는 각 좌표가 삼각형의 꼭짓점이 되도록 그려야 한다. 예를 들어서 좌표 (0,0), (1,0), (0,1), (1,1)에는 다음과 같이 삼각형이 그려진다.
```
A(0,0) --- B(0,1)
  |   \      |
  |     \    |
  |       \  |
C(1,0) --- D(1,1)
```
이 사각형을 그리기 위해서는 삼각형 ADC, ABD를 그려야 한다. index를 이용해서 그린다면 { A, D, C, A, B, D }가 된다. 하지만 [triangle strip](https://en.wikipedia.org/wiki/Triangle_strip)을 쓰면 4개의 index = { A, D, C, B }를 이용해서 표현할 수 있다.\
꼭짓점을 2차원 배열이라고 생각하면, 이런 방법으로 하나의 row를 완성시킬 수 있다. 하지만 다음 column으로 넘어갈 때 문제가 발생한다. 바로 다음 column으로 넘어가자니 전체 배열을 가로지르는 삼각형이 생기기 때문이다. 이를 해결하기 위해서는 다음 row를 그리러 넘어갈 때 삼각형을 그리지 않으면서 다음 row로 넘어가야 한다.\
꼭짓점은 있는데 삼각형을 그리지 않으려면 삼각형이 나오지 않는 꼭짓점을 선택하면 된다. 면을 구성하는 가장 간단한 방법이 삼각형이니, 차원을 하나 낮춰야 한다. 삼각형을 구성할 꼭짓점을 선택할 때, 동일한 꼭짓점을 중복으로 선택하면 삼각형 대신 선이 그려지게 된다. 선의 경우에는 rasterization과정에서 sampling 되지 않을 것이므로 사라진 꼴이 된 것이다. \
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

