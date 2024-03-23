---
layout: post
title:  "Diamond-Square Generation"
categories: OpenGL
tags: [opengl, terrain]
---
Diamond-Square method는 말 그대로 마름모와 정사각형 모양을 기준으로 지형을 생성하는 방법이다. 

<p align="center">
  <img src="https://janert.me/blog/2022/the-diamond-square-algorithm-for-terrain-generation/grid.png" align="center" width="50%">
</p>

Terrain에서 정사각형 모양의 영역을 잡고, diamond step 에서는 영역의 정중앙의 height를 계산하고 square step에서는 영역이 만드는 정사각형의 네 모서리의 중앙의 height를 계산한다.
계산은 4개의 sampling point의 평균에 무작위 값을 더하는 것으로 이루어진다.
이후 영역의 크기를 절반으로 줄여서 동일한 과정을 진행하고, 영역의 크기가 1이 될 때까지 반복한다.<br/>
<br/>
원리는 단순하지만 구현에서 생각해야 할 점이 몇가지 있다.
1. 정사각형 영역이 절반으로 줄어들으므로, 초기 정사각형 영역(= Terrain의 크기)를 적절하게 설정해야 한다.
2. square step에서는 상하좌우 모든 방향의 점을 샘플링하므로 정사각형 영역 밖의 점을 샘플링 할 경우 heigtmap의 외부의 점을 샘플링 하는 경우가 생기게 된다.
3. 이전 과정이 다음 과정에 영향을 주므로 올바른 순서로 진행되어야 한다.

<br/>
### 1. Terrain의 크기 정하기
위의 예시 그림에서는 4x4 크기의 terrain에서 시작된다. 첫 번째 diamond step에서는 (0,0), (4,0), (4,4), (0,4)의 네 점을 평균내고 랜덤값을 더해서 (2,2)의 높이를 계산한다. 이후 square step에서 (2,0), (2,0), (2,4), (2,4)의 점이 계산된다. 이후 2x2 크기의 영역 4개에 대해서 동일한 작업이 진행된다.
조금 생각해 보면 2의 n제곱을 길이로 갖는 정사각형 terrain이여야 한다는 것을 알 수 있다.
따라서 heightmap은 $(2^n+1) \times (2^n+1)$의 크기를 가지는 배열이여야 한다.
`m_sizeX - 1`이 2의 제곱인지 확인하는 과정을 추가한다.
```cpp
void TerrainDiaSq::generate(float roughness, float initialDisplacement){
    if(!is_power_of_two(m_sizeX - 1)){
        std::cout << "Invalid size for diamond square step" << std::endl;
    }
    ...
}

bool is_power_of_two(int n){
    if(n <= 0)
        return false;
    else
        return (n & (n-1)) == 0;
}
```

2의 제곱인지 구하는 함수는 bitwise AND를 이용해서 구현해준다. 어떤 숫자가 2의 제곱일 경우 2진수로 표현할 때 첫자리만 `1`이고 나머지 자리는 `0`인 숫자임을 이용한다.

<br/>
### 2. heightmap 외부의 점을 샘플링해야 할 때
Square step에서 sampling point가 heightmap 외부에 있을 때(Terrain 밖에 있을 때)에는 적절히 처리해야 한다.

<p align="center">
  <img src="https://janert.me/blog/2022/the-diamond-square-algorithm-for-terrain-generation/grid.png" align="center" width="50%">
</p>

위 그림에서 (2,0)의 점을 계산하기 위해서는 상하좌우 2만큼의 거리에 떨어진 점을 구해야 한다. 그러면 (0,0), (2,-2), (4,0), (2,2)의 높이를 구해야 하는데, (2,-2)의 높이는 정의되어 있지 않다. 따라서 이 점을 우리는 두가지 방법으로 처리할 수 있다.
<br/>
첫째, wrapping을 통해서 반대쪽 좌표를 이용한다. 높이를 생성하려는 좌표인 (2,0)를 마치 반대쪽 (2,4)라고 생각하고 (2,-2)의 좌표대신 (2,2)의 좌표를 이용하는 것이다. 다음과 같이 구현할 수 있다.
```cpp
unsigned int wrapCoord(int coord, unsigned int size){
    // size - 1 : length of the map
    if(coord < 0)
        return coord + size - 1;
    else if(coord > size)
        return coord - size + 1;
    else
        return coord;
}
```
그런데 이 방법에서는 문제가 발생하는데, wrapping이 일어나는 부분인 terrain의 모서리로 갈수록 반대편(wrapping된 좌표)의 높이를 샘플링 하기 때문에 주변(wrapping되지 않은 좌표)의 높이와는 다른 값을 같게 된다는 것이다.
![](/assets/img20.png)
그러면 이렇게 모서리 부분이 뾰족하게 된다.
대신에 wrapping을 하지 않고 아예 sampling 과정에서 빼버리는 방법을 사용할 수 있다.
```cpp
float topTop = 0.f;
int T = 0;
// top
if(centerZ - squareSize >= 0){
    topTop = getHeight(centerX, centerZ - squareSize);
    T++;
}

float top = (topTop + topRight + topLeft + center) / (3 + T) + randomDisplacement(randomGen);
```
(2,0)의 높이를 계산할 때의 sampling point인 (2,-2)의 z좌표를 계산한다. 만약 z좌표가 terrain 밖에 위치한다면 그 값을 샘플링하는 대신 0으로 두고서 평균을 구할 때 3으로 나눈다.

### 3. 구현하기 : 반복문? 재귀? 
점점 구역을 잡게 잡아가면서 계산을 진행하기 때문에 재귀를 이용하면 쉽게 구현할 수 있어 보인다.
```cpp
void TerrainDiaSqRecursive::stepRecursive(int topLeftX, int topLeftZ, int squareSize, float randomDisplacementInterval){
    /**
     * 
     *       X   ---   X     ---   topTop  ---      X
     *       |         |             |              |
     *       X   -  topLeft  ---    top    ---   topRight
     *       |         |             |              |
     *    leftLeft   left    ---   center  ---    right  - rightRight
     * ...
    */

    if(squareSize < 2)
        return;
    
    int halfSquareSize = squareSize / 2;
    int centerX = topLeftX + halfSquareSize;
    int centerZ = topLeftZ + halfSquareSize;
    // diamond
    float topLeft = getHeight(centerX - halfSquareSize, centerZ - halfSquareSize);
    float topRight = getHeight(centerX + halfSquareSize, centerZ - halfSquareSize);
    float bottomLeft = getHeight(centerX - halfSquareSize, centerZ + halfSquareSize);
    float bottomRight = getHeight(centerX + halfSquareSize, centerZ + halfSquareSize);

    std::uniform_real_distribution<float> randomDisplacement(-randomDisplacementInterval, +randomDisplacementInterval);
    
    float center = (topLeft + topRight + bottomLeft + bottomRight) / 4.0f + randomDisplacement(randomGen);
    setHeight(centerX, centerZ, center);

    // square
    // if sampling point is outside of the map, do not include in the average
    float topTop = 0.f, leftLeft = 0.f, rightRight = 0.f, bottomBottom = 0.f;
    int T = 0, L = 0, R = 0, B = 0; // flag whether sampled
    // top
    if(centerZ - squareSize >= 0){
        topTop = getHeight(centerX, centerZ - squareSize);
        T++;
    }
    // left
    if(centerX - squareSize >= 0){
        leftLeft = getHeight(centerX - squareSize, centerZ);
        L++;
    }
    // right
    if(centerX + squareSize < m_size){
        rightRight = getHeight(centerX + squareSize, centerZ);
        R++;
    }
    // bottom
    if(centerZ + squareSize < m_size){
        bottomBottom = getHeight(centerX, centerZ + squareSize);
        B++;
    }

    float top       = (topTop + topRight + topLeft + center)                / (3 + T) + randomDisplacement(randomGen);
    float left      = (topLeft + leftLeft + center + bottomLeft)            / (3 + L) + randomDisplacement(randomGen);
    float right     = (topRight + center + rightRight + bottomRight)        / (3 + R) + randomDisplacement(randomGen);
    float bottom    = (center + bottomLeft + bottomRight + bottomBottom)    / (3 + B) + randomDisplacement(randomGen);
    
    setHeight(centerX, centerZ - halfSquareSize, top);
    setHeight(centerX - halfSquareSize, centerZ, left);
    setHeight(centerX + halfSquareSize, centerZ, right);
    setHeight(centerX, centerZ + halfSquareSize, bottom);

    stepRecursive(topLeftX, topLeftZ, halfSquareSize, randomDisplacementInterval * m_displacementReduceRate);
    stepRecursive(topLeftX + halfSquareSize, topLeftZ, halfSquareSize, randomDisplacementInterval * m_displacementReduceRate);
    stepRecursive(topLeftX + halfSquareSize, topLeftZ + halfSquareSize, halfSquareSize, randomDisplacementInterval * m_displacementReduceRate);
    stepRecursive(topLeftX, topLeftZ + halfSquareSize,  halfSquareSize, randomDisplacementInterval * m_displacementReduceRate);
}
```

그러면 이런 결과가 나온다.
![](/assets/img21.png)
격자 모양이 나오게 되는데, 이는 재귀를 이용하면 depth-first로 계산되기 떄문이다.
square step을 하려면 영역 밖의 높이도 sampling하게 되는데, 만약 재귀를 이용한다면 정사각형 영역 밖의 높이가 계산이 되어있지 않은 상태일수도 있다. 이로인해 격자 모양이 나타나게 된다. 
따라서 재귀를 이용하기보다는 반복문을 이용해서 breadth-first로 계산해야 한다.

```cpp
void TerrainDiaSq::generate(float roughness, float initialDisplacement){
    ...
    {
        while(squareSize > 1){
            diamondStep(squareSize, randomDisplacementInterval);
            squareStep(squareSize, randomDisplacementInterval);

            randomDisplacementInterval *= displacementReduceRate;
            // std::cout << randomDisplacementInterval << std::endl;
            squareSize /= 2;
        }
    }
}
void TerrainDiaSq::diamondStep(unsigned int squareSize, float randomDisplacementInterval){
    unsigned int halfSquareSize = squareSize / 2;
    std::uniform_real_distribution<float> randomDisplacement(-randomDisplacementInterval, +randomDisplacementInterval);
    for(unsigned int centerPosZ = halfSquareSize; centerPosZ < m_sizeZ; centerPosZ += squareSize){
        for(unsigned int centerPosX = halfSquareSize; centerPosX < m_sizeX; centerPosX += squareSize){
            diamondStepSampling(squareSize, centerPosX, centerPosZ, randomDisplacementInterval);
        }
    }
}
void TerrainDiaSq::squareStep(unsigned int squareSize, float randomDisplacementInterval){
    unsigned int halfSquareSize = squareSize / 2;
    // odd row
    for(int centerPosZ = 0; centerPosZ < m_sizeZ; centerPosZ += squareSize){
        for(int centerPosX = halfSquareSize; centerPosX < m_sizeX; centerPosX += squareSize){
            squareStepSampling(squareSize, centerPosX, centerPosZ, randomDisplacementInterval);
        }
    }
    // even row
    for(int centerPosZ = halfSquareSize; centerPosZ < m_sizeZ; centerPosZ += squareSize){
        for(int centerPosX = 0; centerPosX < m_sizeX; centerPosX += squareSize){
            squareStepSampling(squareSize, centerPosX, centerPosZ, randomDisplacementInterval);
        }
    }
}
```

![](/assets/img22.png)


---
출처 : [Janert](https://janert.me/blog/2022/the-diamond-square-algorithm-for-terrain-generation/), 
[StackExchange](https://gamedev.stackexchange.com/questions/37389/diamond-square-terrain-generation-problem), 
[OGLDEV](https://www.youtube.com/watch?v=qNoSliX_6Gc&t=393s)