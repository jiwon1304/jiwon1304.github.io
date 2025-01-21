---
layout: post
title:  "아이폰 사파리에서 폰트 사이즈가 이상할 때"
categories: web
tags: [trivial, web]
---
PC에서 볼 때는 문제가 없는데 mobile에서 볼 때 code block의 폰트 사이즈가 비정상적으로 컸다. 같은 폰트 사이즈(px)인데도 다르게 표시되었다. 원인은 모바일은 화면이 회전할 수 있기 때문에 폰트 크기가 가변인데, 여기서 문제가 발생하여 회전했을 때의 폰트 크기가 적용된다. `root` 속성에 한 줄 추가하면 해결 가능하다.
```css
  -webkit-text-size-adjust: 100%;
```

---
출처 : <https://velog.io/@jongk91/root-속성-webkit-text-size-adjust-100>