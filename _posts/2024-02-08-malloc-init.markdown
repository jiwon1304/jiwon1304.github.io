---
layout: post
title:  "new 이용 시 초기화"
categories: C++
tags: [cpp, trivial]
---
지형을 만들다보면 가끔 이상한 현상이 발생한다.

![img3](/assets/img3.png)

그런데 다시 실행해보면 일부만 이상하게 나오거나

![img4](/assets/img4.png)

아니면 정상적으로 나오기도 한다.

![img5](/assets/img5.png)

처음에는 난수 생성에서 문제가 생긴 줄 알았는데, heightmap의 값을 확인해 보면 이상한 값이 들어간다는 것을 확인할 수 있었다.

**보통 매 실행마다 다른 결과가 나온다는 것은 동적 메모리 할당에서 문제가 발생한다는 것이다.**

따라서 heightmap을 할당 이후 초기화시켜주면 잘 작동하는 것을 확인할 수 있다.