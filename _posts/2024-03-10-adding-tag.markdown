---
layout: post
title:  "페이지에 tag 추가하기"
categories: web
tags: [trivial, web]
---

페이지에 카테고리 기능만 있어서 글을 하나의 카테고리에만 넣을 수 밖에 없었다. 여러 개의 태그를 저장하고서 태그별로 볼 수 있는 기능이 필요했다.
문제는 웹 프로그래밍이란 것을 하나도 모른다는 것이다.
5시간 가까이 삽질한 끝에 추가할 수 있었고, 들였던 노력이 아까워서 과정이라도 적어놓으려고 한다. 다 완료하고서 쓴 것이라서 순서를 잘못 썼을 수도 있고, 잘못된 것이 있을 수도 있다.<br/>
<br/>
맨 먼저 tag기능을 추가할 수 있는지 찾아보았다.
많은 글을 찾아보았지만, 결국 직접 다 만들어야 한다는 것이었고, 그나마 [이 글](https://wormwlrm.github.io/2019/09/22/How-to-add-tags-on-Jekyll.html)을 많이 참고하였다.
일단 jekyll에는 tag기능이 있다.
markdown 글 맨 앞에 categories를 적어놓는 것 처럼 tags를 적을 수 있다.
특히, 여러 개일 경우 `tags: cpp,opengl`이 아니라 배열 형식으로  `tags: [cpp,opengl]`로 써야 한다.
그리하여 일단 글에 적당한 태그를 달 수 있었다.<br/>
<br/>
태그를 달았으니, 포스트에 태그가 보이도록 해야겠다.
`_includes/post.html`에서 날짜 다음에 다음과 같은 코드를 집어넣는다.
```html
{%raw%}<div class="tags">
    {% for tag in page.tags %}
    <span class="tag">{{tag}}</span>
    {% endfor %}
</div>{%endraw%}
```
참고한 글에서 코드를 가져왔는데, 이 글에선 좀 복잡한 방법으로 하길래 대신 간단하게 하려고 했다.
현재 페이지에 저장된 태그는 `page.tags`라는 배열에 들어있는 것 같아서 적절히 수정해보았다.
python의 for문과 닮았기에 쉽게 쓸 수 있었다.
<p align="center">
  <img src="/assets/img18.png" align="center" width="50%">
</p>
또한 `class`가 `style.css`에 정의되어 있는대로 포맷시키는 것 같기에 `style.css`에 적당히 기존 코드를 봐가면서 추가했다.
```css
.tags{
    font-size: 100%;
    color: #888888;
    padding-right: 0.1em;
}
```
이렇게 해서 사이드바에 태그 추가하기를 끝냈다.
<br/>
<br/>
그러면 이제 태그를 고를 수 있게 해야한다.
카테고리와 동일한 방식으로 선택할 수 있도록 할 것이다.
일단 `_includes/sidebar.html`을 보고 categories에 해당하는 부분을 똑같이 이름만 바꿔서 복사해준다.
```html
{% raw %}<div class="sidebar-item sidebar-nav">
    <ul class="nav">
          <li class="nav-title">Tags</li>
      {% for tag in site.tags %}
          <a class="nav-item" href="{{ root_url }}/{{ site.tag_dir }}#{{ tag | first }}">
              <span class="tags">{{ tag | first }}</span>
          </a>
      {% endfor %}
    </nav>
  </ul>
</div>{% endraw %}
```
그리고 `_config.yml`에도 
```
tag_dir: tag/ 
```
을 한줄 추가해주고, 루트 디렉토리에 `tag.md`라는 파일을 하나 추가해준다.
내용은 똑같이 바꿔주기만 한다.
```md
---
layout: tag
title: Tag
permalink: /tag/
---
```
이제 사이드바에 추가가 되었다.<br/>
<br/>
문제는 이제부터인데, 태그를 선택했을 때 나오는 페이지를 만들어야 한다는 것이다.
일단 카테고리를 선택했을 때 나오는 페이지가 어떻게 만들어지는지 찾아보았다.<br/>
`_includes/category.html`의 문자열을 바꿔보니 페이지도 바뀌는 것을 보아하니 이 파일로 만들어지는 것 같다.
또한 `_layouts/category.html`에도 `include`라는 문자열이 있는것을 보아 `_layout` -> `_include` 순으로 호출되는 것 같다.
두 파일 모두 tag로 적당하게 이름을 변경하고 복사해준다.
<br/>
<br/>
이제부터가 문제인데, 카테고리의 경우에는 하나밖에 없었지만 태그는 여러개이기 때문에 그대로 적용할 수 없었다.
또한 jQuery도 있었기에 많이 복잡하였다.
일단 맨 아래의 jQuery의 역할은 해당하는 포스트는 보여주고 아닌 포스트는 숨기는 역할을 한다.
그러니까 html도 수정하고 jQuery도 수정해야한다는 것이다.
원래 코드는 각각의 포스트마다 카테고리를 자식 요소로 갖고서 원하는 카테고리를 갖고 있을 때만 표시한다.
이를 이를 약간 수정하여 각각의 포스트가 자신이 가진 모든 태그를 자식 요소로 갖게 하려고 한다.
```html
{% raw %}<!--category.html -->
<li class="post-list-item post-category-{{post.categories}}">
    <!-- {%- assign date_format = site.date_format | default: "%b %-d, %Y" -%} -->
    {%- assign date_format = "%Y/%m/%d" -%}
    <span class="post-date">{{ post.date | date: date_format }}</span>
    <a class="post-link" href="{{ post.url | relative_url }}">
        {{ post.title | escape }}
    {%- if site.show_excerpts -%}{{ post.excerpt }}{%- endif -%}
</li>{% endraw%}
```
<br/>
```html
{% raw %}<!--tag.html -->
<li class="post-list-item">
    {%- for tag in post.tags -%}
        <a class="post-tag-{{ tag }}">
            <!-- {%- assign date_format = site.date_format | default: "%b %-d, %Y" -%} -->
            {%- assign date_format = "%Y/%m/%d" -%}
        </a>
    {%- endfor -%}
    <span class="post-date">{{ post.date | date: date_format }}</span>
    <a class="post-link" href="{{ post.url | relative_url }}">
        {{ post.title | escape }}
    </a>
    {%- if site.show_excerpts -%}{{ post.excerpt }}{%- endif -%}
</li>{% endraw%}
```
`<span>, <li>, <a>`가 뭔진 모르겠지만 그냥 자식 요소에 넣어버렸다.
이제 jQuery를 통해서 원하는 태그를 찾게 만들어본다.

```html
<script>
    function removeTag() {
        if (window.location.hash) {
            let hash = window.location.hash.substring(1);
            $('#tag-name').text(hash);
            let tag_class = '.post-tag-' + hash;
            let items = $('.post-list-item').each((i, e) => {
                if (!$(e).children().hasClass('post-tag-' + hash)) {
                    $(e).hide();
                } else {
                    $(e).show();
                }
            });
        }
    }
    $(window).on('hashchange', () => {
        removeCategory();
    });
    $(document).ready(() => {
        removeCategory();    
    })
    
</script>
```
`$('.post-list-item').each((i, e) => {`를 통해서 post-list-item 속성을 가진 것들을 [iterate하는 것](https://jjeongil.tistory.com/1168)으로 보인다.
즉, 여기서의 `e`가 `{% raw %}{%- for tag in post.tags -%}{% endraw %}`에서의 `tag`가 된다는 것 같다.
자식 요소를 선택하려면 [`.children()`](https://blog.naver.com/kimnx9006/220586673921)을 쓰면 된다.
<br/>
원래 카테고리 코드는 
```html
{% raw %}<li class="post-list-item post-category-{{post.categories}}">{% endraw %}
```
였기 때문에 `.children()`을 쓰지 않아도 됬지만, 태그는 자식 요소로 넣었기에 써야한다.

<p align="center">
  <img src="/assets/img19.png" align="center" width="50%">
</p>

원하는 대로 나왔다.
마지막으로 포스트에 적힌 태그가 태그 페이지로 갈 수 있게 링크를 붙여준다.
```html
{%raw%}<div>
    {% for tag in page.tags %}
    <a class="tags"href="{{ root_url }}/{{ site.tag_dir }}#{{ tag }}">
        <span>{{ tag }}</span>
    </a>
    {% endfor %}
</div>{%endraw%}
```
`class="tags"`를 죄다 붙였더니 css에서 정의한 폰트 사이즈 80%가 중첩되서 적용되서 글씨가 작게 보인다.
그래서 지워봤더니 원하는 대로 작동한다.
`<div>`같은거에 꼭 `class`가 붙지 않아도 되는 것 같다.
<br/>
<br/>
이렇게 해서 삽질을 끝냈다. 부정확한 내용이기 때문에 영양가 있는 글은 아니지만, 생판 모르는 언어를 갖고 어떻게 접근해야 하는지 고민해보았던 기회였기 때문에 기억하려고 작성해보았다. 다음번엔 꼭 인터넷프로그래밍 과목을 수강해봐야겠다...
