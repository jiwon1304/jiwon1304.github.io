---
layout: post
title: "M1 맥에서 OpenGL 설치하기"
date: 2023-08-26 12:25:23 +0900
category: OpenGL
tags: [opengl]
---

OpenGL을 통해서 혼자 그래픽스를 공부하려는데, 라이브러리라는 것을 처음 쓰려고 하니까 초기 설정하는 것부터 난관이었습니다. 학교에서는 이런걸 가르쳐주지 않고 그냥 빈칸 채워넣기마냥 과제를 주었기 때문에 단순히 코드만 입력하는 것 이외에는 아무것도 때문이었죠. 더 배우고 도전해오라는 뜻이었을까요? 하지만 꼭 배워보고 싶었기에 온갖 시행착오를 겪으면서 해결할 수 있었고, 그 방법을 적어두고자 합니다.<br/>
<br/>
사실 인터넷에 찾으면 글들이 나오긴 하는데, 애초에 초보를 상정하고 쓴 글도 아닌데다가, m1맥이 윈도우도 아니고 x86도 아니다 보니까 제대로 된 글을 찾기가 어려웠습니다. 그래서 나중에 다시 설치하게 되다면 볼 수 있게 정리해두려고 했고, 저처럼 visual studio, pycharm같은 ide만 써보고 그 외의 terminal같은 것 조차 써보지 않아 아무도것도 모르는 사람 기준으로 작성하였습니다. (솔직히 OpenGL에 관심있으실 정도면 이정도는 쉽게 하시지 않으실까요? 저는 아니었지만요...)<br/>
<br/>
이 글은 [learnopengl.com](https://learnopengl.com/)이라는 무료 OpenGL course을 따라가기 위한 기본사항이고, course의 [Creating a window](https://learnopengl.com/Getting-started/Creating-a-window)를 기준으로 작성하였습니다.

# 목차
- [목차](#목차)
- [1. OpenGL 구성요소 설치하기](#1-opengl-구성요소-설치하기)
  - [GLFW](#glfw)
  - [GLAD](#glad)
- [2. VSCode 설정하기](#2-vscode-설정하기)
  - [라이브러리 파일 가져오기](#라이브러리-파일-가져오기)
  - [라이브러리 파일 연결하기](#라이브러리-파일-연결하기)
- [3. 그래서 뭘 한거지?](#3-그래서-뭘-한거지)
  - [동적 라이브러리](#동적-라이브러리)
  - [`tasks.json`](#tasksjson)
  - [동적 라이브러리의 위치](#동적-라이브러리의-위치)

---

# 1. OpenGL 구성요소 설치하기
[OpenGL](https://www.opengl.org)은 그래픽스 관련된 작업을 하드웨어와 통신하기 위해서 만든 specification, 즉 소프트웨워와 하드웨어가 어떻게 통신할지 정해놓은 규격 같은 것입니다. [Khronos Group](https://www.khronos.org)에서 관리하지만 실제 구현은 각 하드웨어 제조사가 맡고있죠. 일반 PC라면 Intel, Nvidia, AMD등에서 하고 있을테고, 제가 쓰고있는 맥에서는 Apple이 당담했을 겁니다. 사실 Apple은 이미 OpenGL을 버리고 Metal으로 넘어갔고, 업데이트는 끊긴지 오래됬고, Khronos Group조차 Vulkan으로 넘어갔습니다. 하지만 많은 강의에서 아직까지 OpenGL 3.3을 이용해서 강의를 진행하고 있으며 LearnOpenGL에서도 이 버전을 기준으로 진행하고 있습니다.<br/>
<br/>
처음 프로그래밍을 배우기 시작했을 때 가장 궁금했던 것이 `#include <stdio.h>`었는데 나중에 가서야 standard IO, 즉 입출력을 당담하는 함수들을 미리 구현해놓고 모아놓은 헤더 파일이었다는 것을 알게 되었습니다. OpenGL도 마찬가지로 이러한 것이 필요하고, LearnOpenGL에서는 GLFW와 GLAD를 사용했습니다.  
## GLFW
[GLFW](https://www.glfw.org/download.html)는 OpenGL이 창에 그림을 그리거나, 사용자의 입력을 도와주는 라이브러리입니다. Building GLFW를 따라 `Source package`를 다운로드 받고, 그 다음을 진행하려고 했지만... CMake니 Compilation이니 처음 보는 내용과 Windows에 Visual Studio (Code아님)을 기준으로 설명하고 있었기에 하나도 이해를 할 수가 없었습니다. (사실 `64-bit macOS binaries`를 다운로드 받아서 사용해도 되지만, course에서 직접 컴파일하자고 했으니 저도 직접 컴파일을 해봤습니다. 그리고 이후 model파트에서 `assimp`라는 놈을 써야하는데, 얘는 바이너리를 따로 제공하질 않기 때문에 아래 방법대로 해야합니다.)<br/>
<br/>ㄴ
그래서 적혀있는 방법 대신 [vcpkg](https://github.com/microsoft/vcpkg)라는 마이크로소프트의 툴을 이용해서 설치하기로 했습니다. vcpkg는 여러 라이브러리를 사용자의 환경에 맞도록 알아서 설치하도록 도와주는 프로그램입니다. git을 쓸줄 모르므로 그냥 `Code`에서 `Download ZIP`을 통해서 다운로드합니다. 그러면 `vcpkg-master`라는 폴더가 생기는데, 이제부터는 이 경로에서 터미널을 이용해 작업을 진행합니다.  

> 터미널을 통해서 윈도우의 파일 탐색기, 맥의 파인더에서의 작업을 키보드만으로 할 수 있습니다. 간단한 명령어로썬
> > `cd (디렉토리(폴더))` : 해당 디렉토리로 이동  
> > `ls` : 현재 디렉토리의 내용을 표시  
> > `./(파일명)` : 파일 실행
> 
> 등이 있으며 처음 실행했을 때에는 보통 사용자의 루트 디렉토리에서 실행됩니다.

`Command + Space`를 이용해 Spotlight 검색에 **`terminal`**을 입력하면 터미널을 열 수 있는데요, 터미널을 열면 하얀 화면에 검은 글씨가 저희를 맞아줍니다. 다운로드 디렉토리(폴더)의 `vcpkg-master`에서 작업을 할 것이니 `cd`라는 명령어를 이용해서 해당 디렉토리로 이동해봅시다.
```
cd Downloads/
cd vcpkg-master/
```
위의 내용을 한줄씩 입력하면 이제 `(사용자이름)/Downloads/vcpkg-master/`에서 터미널을 실행하게 되는 것입니다. 굳이 위 명령어를 이용하지 않아도 finder를 통해 들어간 다음에 finder창 맨 아래의 directory에 적힌 `vcpkg-master`를 우클릭하여 `터미널에서 열기`를 눌러서 들어가도 됩니다. 이후 다운로드 받은 vcpkg를 설치하기 위해서 **`bootstrap-vcpkg.sh`**을 실행시킵니다.
```
./bootstrap-vcpkg.sh
```
설치가 완료되면 `/vcpkg-master`디렉토리에 vcpkg라는 실행파일이 생긴 것을 확인할 수 있습니다. 저희는 이제 이를통해 GLFW를 설치할 것이고, 터미널에서 계속해서 다음을 입력해서 설치를 해봅시다.
```
./vcpkg install glfw3:arm64-osx-dynamic
```
`install`, `glfw`, `osx`... 대충 단어를 보아하니 맥(osx) 버전의 glfw 3를 설치하는 것 같죠? 이제 우리는 GLFW를 m1맥에 맞게 실행할 수 있게 만들었습니다.


## GLAD
GLAD는 운영체제마다 이름이 다르게 정해진 OpenGL 함수들을 쉽게 사용할 수 있게 해주는 라이브러리입니다. LearnOpenGL에도 나와있는 과정도 쉽기 때문에 내용 그대로 [GLAD](https://glad.dav1d.de)에 들어가서 `3.3`과 `Core`를 선택하고 `GENERATE`를 눌러서 나오는 페이지에서 모두 다운로드 받으면 됩니다.

---
# 2. VSCode 설정하기
맥에서 C++를 코딩하는 방법엔 [CLion](https://www.jetbrains.com/ko-kr/clion/)이나 [Visual Studio Code](https://code.visualstudio.com) 등이 있습니다. 원래는 CLion을 썼지만, Clion에서 OpenGL을 어떻게 불러올 수 있는지 몰랐기 때문에 Visual Studio를 이용하기로 했습니다.

## 라이브러리 파일 가져오기
예전에 게임을 플레이하려고 할 때, 가끔씩 `d3dx9_XX.dll`이라는 파일을 찾을 수 업다면서 실행이 되지 않았던 적이 있었습니다. 그떄마다 인터넷에 검색해서 파일을 다운로드 받고 특정 위치에 갖다놓고 게임을 실행시키곤 했는데요, OpenGL도 똑같이 이러한 파일을 특정 위치에 위치시켜야지 정상적으로 실행이 됩니다. 여태까지 설치하고 다운로드 한 파일들은 모두 `d3dx9_XX.dll`와 같이 프로그램 실행에 필요한 파일들이고, 이제 우리는 이 파일을 적당한 위치에 이동시켜야 합니다.<br/>
<br/>
OpenGL을 배우다보면 과거에 딸랑 한두개의 파일만 사용해서 코딩했던 것과는 다르게 쉐이더, 라이브러리, 리소스(텍스쳐, 모델링 등)와 같이 많은 부분으로 나누어서 코딩하기 때문에 프로젝트 폴더를 조금 정리할 필요가 있습니다. LearnOpenGL에서는 다음과 같이 프로젝트 폴더를 구성해놓았습니다. (실제로 명시해놓진 않았고, [course 전체 파일](https://github.com/JoeyDeVries/LearnOpenGL)을 다운로드 받았을때 이렇게 되어있습니다.)
```
. (프로젝트의 루트 디렉토리)
├── includes
├── lib
├── resources
├── src
└── (그 외 파일들)
```
위에 적혀있는 대로 폴더들을 만들어도 되고, 아니면 [제가 참고했던 동영상](https://www.youtube.com/watch?v=7-dL6a5_B3I)을 기준으로 폴더를 만들어도 됩니다.
```
. (프로젝트의 루트 디렉토리)
├── dependencies
│   ├── include
│   └── library
├── resources
└── (그 외 파일들)
```
저는 처음부터 아래의 구조로 해놓았기 때문에 이를 기준으로 설명할 거지만, `/includes`대신 `/dependenceis/include`, `lib`대신 `/dependenceis/library`로 쓰였다는 것만 보이시면 상관없습니다. `resources` 폴더는 나중에 필요하시면 만드시고, 일단은 **`dependencies`** 폴더와 그 안에 **`include`**와 **`library`** 폴더를 만들어줍시다.\
그리고 아까 전 다운로드한 GLFW와 GLAD를 준비합시다. GLFW는 vcpkg를 설치했던 디렉토리에서 `/vcpkg-master/installed/arm64-osx-dynamic/`에 가시면 찾으실 수 있고, GLAD는 다운로드 받았던 폴더를 바로 사용하시면 됩니다.
* `/vcpkg-master/installed/arm64-osx-dynamic/`에 들어가시면 `include`와 `lib`폴더가 있습니다. `include`폴더 내부의 **`.dylib` 파일**을 전부 프로젝트 디렉토리의 `/dependencies/include`에 넣어주시고, `lib`폴더의 **`GLFW` 폴더**를 그대로 `/dependencies/library`에 넣어줍시다.
* GLAD도 마찬가지로 `include`폴더의 **`glad`, `KHR` 폴더**를 그대로 넣어주시고, `src`폴더의 **`glad.c`**는 `/dependencies/include/glad`에 넣어줍니다.

이러면 다음과 같은 구조로 저장되어 있을 것입니다.
```
. (프로젝트의 루트 디렉토리)
├── dependencies
│   ├── include
│   │   ├── GLFW
│   │   │   ├── glfw3.h
│   │   │   └── glfw3native.h
│   │   ├── KHR
│   │   │   └── khrplatform.h
│   │   ├── glad
│   │   │   ├── glad.c
│   │   │   └── glad.h
│   └── library
│       ├── libglfw.3.3.dylib
│       ├── libglfw.3.dylib
│       └── libglfw.dylib
└── (다른 파일들)
```

여기까지 해서 필요한 파일들을 모두 구성하였습니다. 이제 VSCode가 이 파일들을 찾아서 쓸 수 있도록 설정할 겁니다.

## 라이브러리 파일 연결하기
일단 VSCode에서 앞으로 사용하게 될 프로젝트를 하나 만들어주고 프로젝트의 최상위 디렉토리에 `main.cpp`에 간단한 hello world! 코드를 작성합니다. 그리고 VSCode의 메뉴에서 Terminal > Build를 누르면(단축키 `command + shift + B`) 여러 가지 컴파일러가 나오는데, 그 중 `clang++`에 커서를 올려서 옆에 뜨는 설정 아이콘을 누릅니다. (gcc나 clang이나 고르는게 상관 있는지는 모르겠음)<br/>
그러면 `tasks.json`이 생성되어 열린 것을 볼 수 있는데요, `tasks.json`파일은 빌드(코드를 실행파일로 만드는 작업) 과정에서 주는 옵션을 적어놓는 파일입니다. 처음 생성되면 다음과 같이 적혀있을 겁니다.
```json
{
	"version": "2.0.0",
	"tasks": [
		{
			"type": "cppbuild",
			"label": "C/C++: clang++ build active file",
			"command": "/usr/bin/clang++",
			"args": [
				"-fcolor-diagnostics",
				"-fansi-escape-codes",
				"-g",
				"${file}",
				"-o",
				"${fileDirname}/${fileBasenameNoExtension}"
			],
			"options": {
				"cwd": "${fileDirname}"
			},
			"problemMatcher": [
				"$gcc"
			],
			"group": "build",
			"detail": "compiler: /usr/bin/clang++"
		}
	]
}
```
파이썬도 아니고 c도 아니고 이상한 언어가 우리를 반겨줍니다. Pycharm이나 CLion썼을 때에는 딸깍 한번에 실행되었기에 잘 몰랐었지만 실제로 코드를 실행 가능한 파일로 만들기 위해서는 많은 과정이 필요하고, 이 파일은 우리가 원하는대로 옵션을 줄 수 있게 적어놓는 곳입니다.
중간에 보이는 `"args"` 아래의 괄호가 우리가 수정해야 하는 공간입니다. 괄호 마지막에 있는 `"${fileDirname}/${fileBasenameNoExtension}"` **뒤에 콤마를 적고**, 그 아래에 다음과 같은 코드를 입력합니다.
```
"${workspaceFolder}/dependencies/include/glad/glad.c",

"-I${workspaceFolder}/dependencies/include",

"-L${workspaceFolder}/dependencies/library",
"-lglfw",

"-rpath",
"@executable_path/dependencies/library"
```
그러면 이렇게 되겠죠?
```json
"args": [
    "-fcolor-diagnostics",
    "-fansi-escape-codes",
    "-g",
    "${file}",
    "-o",
    "${fileDirname}/${fileBasenameNoExtension}",

    "${workspaceFolder}/dependencies/include/glad/glad.c",

    "-I${workspaceFolder}/dependencies/include",
    
    "-L${workspaceFolder}/dependencies/library",
    "-lglfw",
    
    "-rpath",
    "@executable_path/dependencies/library",
]
```
그러면 마지막으로 `main.cpp`에 [예제 코드](https://learnopengl.com/code_viewer_gh.php?code=src/1.getting_started/2.2.hello_triangle_indexed/hello_triangle_indexed.cpp)를 넣고 빌드(`command + shift + B`)하고 VSCode에서 terminal을 열어서 `./main`을 입력하여 빌드한 파일을 실행해봅시다.
![square](/public/img/opengl-1.png)
이렇게 주황색 사각형이 나오면 완성! 여기까지 하셨으면 이제 LearnOpenGL의 [다음 페이지](https://learnopengl.com/Getting-started/Hello-Window)부터 시작하시면 됩니다.

---

# 3. 그래서 뭘 한거지?

## 동적 라이브러리
[vcpkg를 통해 GLFW를 컴파일하는 과정](#glfw)에서 `vcpkg`를 통해서 `libglfw.3.3.dylib`, `libglfw.3.dylib`, `libglfw.dylib` 파일을 얻을 수 있었습니다. 이 **`.dylib` 파일**은 동적 라이브러리 파일로, 미리 기계어로 번역된 파일입니다. 저희가 VSCode에서 컴파일 버튼 딸깍 누르면 바로 코드가 실행 파일로 변하지만, 실제로는 `전처리 - 컴파일 - 어셈블리 - 링킹`의 과정을 거칩니다. `.h`, `.cpp`와 같은 헤더, 소스 파일들은 하나하나 `전처리 - 컴파일 - 어셈블리` 과정을 거쳐서 각각 `.o` 파일로 변환됩니다. 이후 `링킹` 과정을 통해서 모두 하나로 합쳐서 하나의 실행 파일(프로그램)을 내놓는데요, `.dylib`와 같은 라이브러리 파일은 이미 앞의 과정을 거쳐서 만들어진 파일이며 다른 파일들과 링킹되지 않습니다. 대신 프로그램이 시작할 때 `로딩`되어서 프로그램이 사용하게 됩니다.<br/>
<br/>
굳이 이렇게 하는 이유는, 미리 컴파일하여 시간을 절약할 수 있고(동적/정적), 프로그램 외부에 코드를 위치시켜서 프로그램 자체의 크기를 절약할 수 있기 때문입니다(동적). 개발자가 직접 컴파일하여 배포해도 되겠지만, 그렇지 않은 경우에는 이렇게 직접 `vcpkg`와 같은 툴을 사용해서 사용자가 자신의 환경에서 직접 컴파일하여 이용하는 것이지요.

## `tasks.json`
그렇다면 방금 전의 [`tasks.json`](#라이브러리-파일-연결하기)파일에서 적었던 내용이 무엇인지 이해할 수 있습니다.
```json
"args": [
    "-fcolor-diagnostics",
    "-fansi-escape-codes",
    "-g",
    "${file}",
    "-o",
    "${fileDirname}/${fileBasenameNoExtension}",

    "${workspaceFolder}/dependencies/include/glad/glad.c",

    "-I${workspaceFolder}/dependencies/include",
    
    "-L${workspaceFolder}/dependencies/library",
    "-lglfw",
    
    "-rpath",
    "@executable_path/dependencies/library",
]
```

* **`"${workspaceFolder}/dependencies/include/glad/glad.c"`** : `#include`를 통해서`.h` 헤더 파일을 포함시킬 수 있지만, `.c`와 같은 소스 파일은 따로 지정을 해야합니다. 따라서 `glad.c`를 따로 지정해주었습니다.

* **`"-I${workspaceFolder}/dependencies/include", "-I${workspaceFolder}/"`** : `-I`는 소스 코드의 `#include`에서 사용할 경로입니다.

* **`"-L${workspaceFolder}/dependencies/library", "-lglfw"`** : `-L`은 라이브러리 파일의 경로를 지정합니다. 그리고 그 뒤의 `-lglfw`는 라이브러리로 glfw를 사용한다는 의미입니다. 라이브러리를 지정할 때에는 `-l`뒤에 바로 라이브러리 이름을 적습니다. 아까 `/dependencies/library/`의 `libglfw.dylib`파일을 기준으로, **앞의 `lib`와 `.dylib`를 뺀 것이 라이브러리 이름이 됩니다.** (또한 버전 관리를 위해서 실제 파일은 `libglfw.3.3.dylib`이고 나머지는 가상본인 것을 알 수 있습니다.)
* **`"-rpath", "@executable_path/dependencies/library"`** : 동적 라이브러리는 프로그램을 시작할 때 불러온다고 했는데, -rpath는 프로그램을 시작할 때 어디서 불러올건지에 대한 옵션입니다. 빌드가 된 프로그램은 시작할 때 자신의 디렉토리에서 `/dependencies/library"`로 들어가서 라이브러리를 찾습니다. 따라서 만약 실행 파일이 프로젝트의 루트 디렉토리, 즉 폴더 `dependencies`가 있는 디렉토리가 아니라 다른 곳에 있다면 라이브러리를 찾지 못하겠죠? **그러므로 프로그램을 실행시킬 때에는 프로젝트의 루트 디렉토리에 두고 사용하셔야 합니다.**

## 동적 라이브러리의 위치
하지만 항상 동적 라이브러리 파일을 프로그램의 위치에 따라서 옮겨야 한다면 매우 비효율적일 것입니다. 그래서 MacOS를 포함한 운영체제는 프로그램을 시작할 때 `-rpath`로 지정한 경로 뿐만 아니라 기본적으로 운영체제에서 정해 놓은 위치에서도 라이브러리를 찾습니다. Windows의 경우에는 `/Windows/system32/`, MacOS의 경우에는 `/usr/local/lib/`등이 있고, 더 지정할 수도 있습니다. 따라서 우리가 `.dylib` 파일을 굳이 `/dependencies/library/`에 넣지 않고 `/usr/local/lib/`에 넣어놓으면 프로그램을 실행할 때 굳이 라이브러리 파일을 옮길 필요가 없어지게 됩니다. 마치 `d3dx9_XX.dll`파일을 `/Windows/system32/`에 넣고 게임을 실행했던 것처럼 말이죠.