---
layout: post
title:  "ImGui 적용기"
categories: OpenGL
tags: [opengl, trivial]
---

ImGui를 추가하면서 사용한 `tasks.json`이다.
```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "type": "cppbuild",
      "label": "C/C++: g++ build active file",
      "command": "/usr/bin/g++",
      "args": [
        "-std=c++17",
        "-fdiagnostics-color=always",
        "-g",

        "-I${workspaceFolder}/dependencies/include",
        "-I${workspaceFolder}/dependencies/imgui",
        "-I${workspaceFolder}/dependencies/imgui/backends",
     
        "-L${workspaceFolder}/dependencies/library",
        "-lassimp",
        "-lglfw",
        "-lfreetype",
        
        "-rpath",
        "${workspaceFolder}/dependencies/library",
     
        "${file}",
        "${workspaceFolder}/dependencies/glad/glad.c",

        "-o",
        "${fileDirname}/${fileBasenameNoExtension}",
     
        "-framework",
        "OpenGL",
        "-framework",
        "Cocoa",
        "-framework",
        "IOKit",
        "-framework",
        "CoreVideo",
        "-framework",
        "CoreFoundation",
        "-Wno-deprecated",
      ],
      "options": {
        "cwd": "${fileDirname}"
      },
      "problemMatcher": [
        "$gcc"
      ],
      "group": "build",
      "detail": "compiler: /usr/bin/g++"
    }
  ]
}
```

그런데 ImGui를 추가하고 `ld:undefined symbols` 컴파일을 하려니까 오류가 발생했었다. 분명히 `#include "imgui.h"`가 있어서 include는 시켰는데 왜 symbol을 못찾는거지? 분명히 `"imgui.h"`는 인식을 한 것 같은데 링킹을 왜 못할까?  

# pre-processing
사실 `.h`파일은 직접적으로 컴파일이 되는 것이 아니다. gcc(정확하게는 컴파일러인 cc1)는 소스파일인 `.c`를 컴파일 하는것이다. [여기](https://bradbury.tistory.com/226)에 나와 있는 내용을 짧게 정리해보자면 pre-processing에서 `#include "imgui.h"`를 만나게 되면 그 부분을 말그대로 `imgui.h`에 있는 텍스트로 대치하게 된다. 그래서 `.h`파일에서 선언을 하고 `.cpp`파일에서 정의를 하면 pre-processing을 한 이후에는 앞에서 선언하고 뒤에서 정의하는 `.i`파일이 만들어지게 된다.<br/>
<br/>
간단한 코드로 g++의 `-E`옵션을 통해서 확인해보자<br/>
```c
// test.c
#include "my.h"

int main(){
	myfunc();
	return 0;
}
```

```c
// my.h
#include <stdio.h>

void myfunc(){
	printf("HELLO WORLD");
}
```

그리고 terminal에서 `g++ -E temp.c -o temp.i`을 입력하면 pre-processing만 진행된 파일인 `temp.i`를 얻을 수 있는데, 이를 열어보면 다음과 같다.<br/>
```c
// temp.i
# 1 "temp.c"
# 1 "<built-in>" 1
# 1 "<built-in>" 3
# 433 "<built-in>" 3
# 1 "<command line>" 1
# 1 "<built-in>" 2
# 1 "temp.c" 2
# 1 "./my.h" 1
# 1 "/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/c++/v1/stdio.h" 1 3
# 101 "/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/c++/v1/stdio.h" 3
# 1 "/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/c++/v1/__config" 1 3
# 13 "/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/c++/v1/__config" 3
# 1 "/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/c++/v1/__config_site" 1 3
# 39 "/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/c++/v1/__config_site" 3
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wmacro-redefined"

(생략)

# 109 "/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/c++/v1/stdio.h" 2 3
# 2 "./my.h" 2

void myfunc(){
 printf("HELLO WORLD");
}
# 2 "temp.c" 2

int main(){
 myfunc();
 return 0;
}
```
`#include`가 있어야 할 자리에 `temp.c` → `my.h` → `stdio.h` 순서로 나타나는 것을 확인할 수 있다. 즉, `#include`의 위치에 해당 파일로 대치하는 것이다.<br/>
따라서 g++를 통해서 빌드를 할 때 주는 옵션인 `-I`는 그저 `#include <>`를 만났을 때 대치하려는 파일의 위치만을 알려줄 뿐이다.<br/>
<br/>
소스파일의 경우는 `#include`를 통해서 빌드에 들어가는 것이 아니라 직접 object file로 만들어야 되기 때문에 g++ 명령어에 옵션 없이 argument로 들어가야 한다. 그렇기 때문에 `"${workspaceFolder}/dependencies/glad/glad.c"`를 argument로 넣었던 것이다. [LearnOpenGL](https://learnopengl.com/Getting-started/Creating-a-window)에서 말한 "add the glad.c file to your project." 라는 것이 이 의미였나보다.<br/>
<br/>
동일한 내용이 ImGui의 [Getting Started](https://github.com/ocornut/imgui/wiki/Getting-Started)에도 자세히 나와있다.  
> (3) Add files to your project or build system of your choice, so they get compiled and linked into your app.  
> Add all source files in the root folder: imgui/{*.cpp,*.h}.  
> Add selected imgui/backends/imgui_impl_xxxx{.cpp,.h} files corresponding to the technology you use from the imgui/backends/ folder (e.g. if your app uses SDL2 + DirectX11, add imgui_impl_sdl2.cpp, imgui_impl_dx11.cpp etc.). If your engine uses multiple APIs you may include multiple backends.  

여기서 말한대로 다음 옵션을 통해서 해당 파일을 추가하면 잘 작동하는 것을 확인할 수 있다.
```
"${workspaceFolder}/dependencies/imgui/*.cpp",
"${workspaceFolder}/dependencies/imgui/backends/imgui_impl_glfw.cpp",
"${workspaceFolder}/dependencies/imgui/backends/imgui_impl_opengl3.cpp",
```

# Makefile
ImGui에서 제공한 예시 파일을 보면 glfw3 / OpenGL을 이용할 때의 예시 코드를 제공하고 있다. 여기서 `Makefile`를 확인해보면 vscode에서 사용하는 `tasks.json`과는 조금 다르게 적힌 빌드 옵션을 확인할 수 있다. 

```makefile
SOURCES = main.cpp
SOURCES += $(IMGUI_DIR)/imgui.cpp $(IMGUI_DIR)/imgui_demo.cpp $(IMGUI_DIR)/imgui_draw.cpp \
$(IMGUI_DIR)/imgui_tables.cpp $(IMGUI_DIR)/imgui_widgets.cpp
SOURCES += $(IMGUI_DIR)/backends/imgui_impl_glfw.cpp $(IMGUI_DIR)/backends/imgui_impl_opengl3.cpp
```
```makefile
CXXFLAGS = -std=c++11 -I$(IMGUI_DIR) -I$(IMGUI_DIR)/backends
```
```makefile
LIBS = 
ifeq ($(UNAME_S), Darwin) #APPLE
	ECHO_MESSAGE = "Mac OS X"
	LIBS += -framework OpenGL -framework Cocoa -framework IOKit -framework CoreVideo
	LIBS += -L/usr/local/lib -L/opt/local/lib -L/opt/homebrew/lib
	#LIBS += -lglfw3
	LIBS += -lglfw

	CXXFLAGS += -I/usr/local/include -I/opt/local/include -I/opt/homebrew/include
	CFLAGS = $(CXXFLAGS)
endif
```
대충 해석해보면 비슷한 의미를 갖는다는 것을 알 수 있다.

# glfw는?
glfw는 `#include <GLFW/glfw3.h>`으로 `main.cpp`에 헤더파일만 포함시켰을 뿐 g++의 argument로 소스파일을 넘기진 않았다. 대신 shared library를 통해서 코드에 접근한다. argument에서
```
"-L${workspaceFolder}/dependencies/library",
"-lassimp",
"-lglfw",
"-lfreetype",
```
로 `libglfw.dylib`를 접근하게 된다. 헤더파일을 쓰는 것으로 봐서 빌드 타임에는 선언을 하고서 런타임에 dynamic linking을 통해서 정의를 하는 것 같다. 생각해보니 단순히 대치하는 것보다 더 복잡하기 때문에 GOT니 lazy binding이니 이런 것을 쓰나보다.  
실제로 `glfw3.h`를 뜯어보면 선언만 되어있지 정의는 되어있지 않다.  
```cpp
/*! @brief Returns the window whose context is current on the calling thread.
 *
 *  This function returns the window whose OpenGL or OpenGL ES context is
 *  current on the calling thread.
 *
 *  @return The window whose context is current, or `NULL` if no window's
 *  context is current.
 *
 *  @errors Possible errors include @ref GLFW_NOT_INITIALIZED.
 *
 *  @thread_safety This function may be called from any thread.
 *
 *  @sa @ref context_current
 *  @sa @ref glfwMakeContextCurrent
 *
 *  @since Added in version 3.0.
 *
 *  @ingroup context
 */
GLFWAPI GLFWwindow* glfwGetCurrentContext(void);
```
선언이랑 주석만 줄줄히 달려있고 정의는 되어있지 않다. 아마 `libglfw.dylib`정의되어있나 보다.