---
layout: post
title:  "texture id와 소멸자"
categories: OpenGL
---

> [요약]\
> OpenGL의 경우 buffer/texture를 포인터마냥 GLuint(unsigned int)로 접근하도록 한다. 따라서 클래스 인스턴스에 저장된 GLuint값과 그 buffer/texture는 인스턴스를 복사한다고 해서 여러 개가 만들어지는 것이 아니다. 만약 (pass by value로) 복사된 instance가 함수가 끝나면서 소멸자가 실행된다면, 그 소멸자가 저장된 GLuint값에 해당하는 (즉 함수의 argument의 그것과도 동일한) buffer/texture를 지워버릴 수 있다.


Texture2D라는 클래스를 제작하였는데, 이 클래스는 다음과 같이 texture id를 저장하고 소멸자로는 class 소멸시에 해당 texture object(data)를 지우는 소멸자를 가진다. 
```cpp
class Texture2D{
private:
    unsigned int ID;

public:
    Texture2D(char* imagePath);
    Texture2D();
    ~Texture2D();
    ...
};

Texture2D::~Texture2D(){
    glDeleteTextures(1, &ID);
}
```

그리고서 shader 클래스에 sampler2D의 텍스쳐를 uniform으로 쓰도록 `setUniformTexture2D`라는 함수를 만들었다. 
```cpp
void Shader::setUniformTexture2D(const std::string name, const GLuint slot, const Texture2D texture){
    this->use();
    glActiveTexture(GL_TEXTURE0 + slot);
    texture.bind();
    std::cout << texture.getID() << std::endl;
    this->setInt(name, slot);
}
```
그런데 막상 실행을 하면 다음과 같은 오류가 발생한다. 텍스쳐가 쉐이더에 제대로 로딩이 되지 않았을때 뜨는 오류이다. 보통은 잘못된 경로라든지의 경우로 로드 자체가 안된 것이다.
```
UNSUPPORTED (log once): POSSIBLE ISSUE: unit 0 GLD_TEXTURE_INDEX_2D is unloadable and bound to sampler type (Float) - using zero texture because texture unloadable
```
무엇이 문제인지 계속 바꿔보다가 parameter인 `Texture2D texture`를 `Texture2D &texture`로 바꾸면 되는 것이었다. 그래서 pass by value/reference가 문제인가 하고 여러 시도를 해보았는데,
1. 함수 내에서 `texture`를 새로운 `Texture2D` 인스턴스에 복사해서 실행한다.
2. `Texture2D` 인스턴스를 argument로 넘기기 전에 새로운 `Texture2D` 인스턴스에 복사해서 실행한다.
3. `Texture2D`를 parameter로 하지 않고 `GLuint`를 parameter로 한다.

를 해본 결과 3만 정상적으로 작동했다. 그래서 `Texture2D` 클래스에 문제가 있다고 생각하면서 코드를 확인해보았다. 이전에 할 때에는 동일한 방법으로 하여도 잘 실행되었는데, 무엇이 문제일까 하다가 왠지 있으면 좋을 것 같은 소멸자를 한번 지워보았다. 결과는 123모두 잘 작동하였다.\
\
뭐가 문제일까 하고 보니 소멸자의 `glDeleteTextures`가 문제였다. 원래 의도는 `texture` 인스턴스를 소멸시킬 때 텍스쳐 데이터또한 지우려는 의도였지만, 문제는 텍스쳐 데이터는 코드에서 하나의 오브젝트로 다루는 것이 아니라 `GLuint`의 타입의 id로 마치 포인터마냥 다루어진다는 것이 문제였다.\
\
pass by value를 하면서 복사된 `texture` 인스턴스는 함수가 끝남과 동시에 소멸자를 호출하게된다. 하지만 인스턴스가 복사되었다고 해서 텍스쳐 데이터까지 복사된 것은 아니었기 때문에 복사된 인스턴스의 소멸자는 원본 인스턴스의 id가 가리키는 텍스쳐 데이터를 제거한다. 따라서 소멸자가 호출되지 않도록 pass by reference로 하던가, 아니면 texture ID만 넘기는 식으로 해결해야 한다.\
\
비단 OpenGL의 id를 다루는 형식 뿐만 아니라, 포인터를 쓰는 클래스에서의 소멸자에 대해서는 잘 생각해야 된다는 점을 알게되었고, pass by value로 복사된 인스턴스도 소멸자를 실행하게 된다는 점을 알게 되었다.