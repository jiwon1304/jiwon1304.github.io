---
layout: post
title:  "Smoothing Terrain"
categories: OpenGL
tags: [opengl, terrain]
---
무작위로 지형을 만들다보면 높이가 너무 튀는 경우가 생긴다. 
![](/assets/img12.png)

이를 해결하기 위해서 한 좌표를 기준으로 주변 높이를 평균내서 다시 높이를 설정한다.

```cpp
void Terrain::smooth(unsigned int iterTimes){
    // copy original one
    float originalHeightMap[m_numX*m_numZ];
    for(unsigned int x=0;x<m_numX;x++)
        for(unsigned int z=0;z<m_numZ;z++)
            originalHeightMap[x+z*m_numX] = getHeight(x,z);

    for(unsigned i = 0; i < iterTimes; i++){
        for(int x = 0; x < m_numX; x++){
            for(int z = 0; z < m_numZ; z++){
                // sample adjacent points
                float samplingSum = 0;
                unsigned int numOfSamplePoints = 0;
                for(int _x = x-1; _x < x+2; _x++){
                    for(int _z = z-1; _z < z+2; _z++){
                        if(0 <= _x && _x < m_numX && 0 <= _z && _z < m_numZ){
                            samplingSum += originalHeightMap[_x+_z*m_numX];
                            numOfSamplePoints++;
                        }
                    }
                }
                setHeight(x, z, samplingSum / numOfSamplePoints);
            }
        }
    }
}
```
cache를 신경쓰면서 만든다면 더 빠른 코드를 만들 수 있겠지만, 별로 호출될 것 같지 않기에 간단하게만 만들어보았다.

<p align="center">
  <img src="/assets/img13.png" align="center" width="32%">
  <img src="/assets/img14.png" align="center" width="32%">
  <img src="/assets/img15.png" align="center" width="32%">
  <figcaption align="center">iteration = 1, 2, 5</figcaption>
</p>

---
[출처](http://nic-gamedev.blogspot.com/2013/02/simple-terrain-smoothing.html)