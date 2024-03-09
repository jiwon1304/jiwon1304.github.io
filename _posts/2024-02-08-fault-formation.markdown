---
layout: post
title:  "Fault Formation"
categories: OpenGL
tags: [opengl, terrain]
---
Fault Formation는 [단층](https://ko.wikipedia.org/wiki/단층) 형성이라는 뜻으로, 지진 등으로 인해 지각이 끊어지며 수직 운동하게 되는 현상을 본따서 지형을 생성하는 방법이다.
<p align="center">
  <img src="http://wiki.hash.kr/images/1/1a/단층.png" align="center" width="50%">
  <figcaption align="center">단층의 모습. 단면을 기준으로 한쪽은 올라가고 한쪽은 내려가있다.</figcaption>
</p>
<br/>
단층을 형성하는 방법은 간단하다. 지형 위에 직선을 긋고, 직선 기준으로 한쪽은 올리고 한쪽은 내리면 된다. 이를 반복하다 보면 그럴듯한 지형이 만들어지게 된다.
<br/>
무작위 직선이야 아무 두 점을 이으면 얻을 수 있다. 구현을 하기 위해 고민해야 할 것은 어떻게 직선을 기준으로 한쪽은 올리고 한쪽은 내리는가다. 이를 해결하기 위해서 벡터의 외적(cross product)를 사용할 수 있다.
무작위 직선을 구성하는 두 점 p0, p1 그리고 height를 결정할 임의의 점 P에 대해서, (p1 - p0)와 (P - p0) 벡터를 만든다. 두 벡터를 외적하면 나오는 벡터는 y축 방향으로 +또는 -방향을 가리킬 것이다. 이를 통해서 한쪽은 내리고 한쪽은 올릴 수 있다.

```cpp
void TerrainFaultFormation::generateInternal(float _scale, int _seed){
    ...
    int cutLineDirX = p1[0] - p0[0];
    int cutLineDirZ = p1[1] - p0[1];

    for(unsigned int z = 0; z < m_numZ; z++){
        for(unsigned int x = 0; x < m_numX; x++){
            int dirX = x - p0[0];
            int dirZ = z - p0[1];

            int crossProduct = cutLineDirX * dirZ - cutLineDirZ * dirX;

            float deltaHeight;
            if(crossProduct > 0){
                deltaHeight = _scale;
            }
            else{
                deltaHeight = (-1)*_scale;
            }

            m_heightMap[x + z*m_numX] += deltaHeight;
        }
    }
}
```
모든 좌표에 대해서 계산하고 나면 다음과 같은 결과를 얻을 수 있다.
![](/assets/img6.png)
이와 같은 작업을 여러번 반복한다. 이때, 자연스러운 결과물을 얻기 위해서 반복이 진행될수록 높이를 변화시키는 정도를 줄인다.
```cpp
void TerrainFaultFormation::generateFaultFormation(unsigned int iterations, float scale, int seed){
    std::minstd_rand0 gen(seed);
    for(unsigned int i = 1; i <= iterations; i++){
        generateInternal(scale * (1 - i / (float)iterations), gen());
    }
}
```
한번 반복할 때 마다 단층이 생기는 것을 확인할 수 있다.

<p align="center">
  <img src="/assets/img7.png" align="center" width="32%">
  <img src="/assets/img8.png" align="center" width="32%">
  <img src="/assets/img9.png" align="center" width="32%">
  <figcaption align="center">iteration = 1, 2, 3</figcaption>
</p>
<p align="center">
  <img src="/assets/img10.png" align="center" width="100%">
  <figcaption align="center">iteration = 10</figcaption>
</p>
충분히 많이 반복하면 다음과 같은 지형이 나온다.
![](/assets/img11.png)


---
참고 : [OGLDEV](https://www.youtube.com/watch?v=z9YML6j5yDg)