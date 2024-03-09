---
layout: post
title:  "static 변수의 초기화"
categories: C++
tags: [trivial]
---

Diamond-Square method를 구현하였는데, 생각했던 대로 결과가 나오지 않았다. 원했던 것은 어느정도 dynamic함이 있어야 하는데, 그냥 slope하나에 random displacement가 추가된 느낌이었다.

원래 코드에는 random 라이브러리를 이용해서 `std::uniform_real_distribution`를 이용했다. 한 step씩 하면서 square사이즈가 줄어들 때마다 random value의 범위를 줄이기 위해서 다음과 같이 했다.
```cpp
std::uniform_real_distribution<float> randomDisplacement(-initialDisplacement, +initialDisplacement);
```
매 iteration마다 `initialDisplacement`가 줄어들면서 값을 뽑아내도록 할 의도였다.

그런데 매 iteration마다 `uniform_real_distribution`의 오브젝트를 만들고 지우기 보다는 계속해서 메모리에 상주시키면서 실행시키고 싶었다. 그래서 static 키워드를 붙였다. 이러면 불필요하게 메모리에 생겼다 사라졌다 할 필요가 없다고 생각하기 때문이다.

```cpp
static std::uniform_real_distribution<float> randomDisplacement(-initialDisplacement, +initialDisplacement);
```

이렇게 하고 코드를 실행해 보니 다음과 같은 결과물이 나오게 된 것이다.

![](/assets/img16.png)

원래대로라면 한 step마다 square 사이즈가 작아지면서 그와 동시에 random value또한 작아져야 했지만 그렇지 못한 상황이다. 이를 해결하기 위해서는 static 키워드를 지우면 된다.

![](/assets/img17.png)

이런 상황이 발생한 이유는 static이 붙어있을 경우에는 constructor를 최초 한번만 부르기 때문이다. 즉, 반복문을 돌면서 constructor에 다른 값이 들어간다 해도 이는 무시된다는 뜻이다. 다음의 예제를 보면 확인할 수 있다.

```cpp
#include <iostream>

class Myclass{
public:
    Myclass(int a){
        std::cout << a << std::endl;
    }
    ~Myclass(){
        std::cout << "destructor" << std::endl;
    }
};

int main(){
    std::cout << "Loop start" << std::endl;
    for(int i = 0; i<3; i++){
        static Myclass myvar(i);
    }
    std::cout << "Loop end" << std::endl;
    return 0;
}
```

```
% ./test
Loop start
0
Loop end
destructor
```

실행 결과대로, static이 붙었을 경우에는 다른 값이 들어오더라도 객체를 새로 만들지 않고 무시하게 된다. static의 역할을 충실히 하고 있다. 또한 static variable의 constructor는 처음 실행된 라인에서 실행되지만, [destuctor는 main함수가 끝날 때 실행됨을 알 수 있다.](https://stackoverflow.com/questions/2204608/does-c-call-destructors-for-global-and-class-static-variables)