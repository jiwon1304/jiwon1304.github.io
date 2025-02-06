---
layout: post
title:  "프로그램에서 시간 측정하기"
categories: DirectX
tags: [directx]
---
애니메이션과 같이 시간에 따라서 변하는 요소들을 이용하기 위해서는 현재 프레임의 시각을 알아야한다. 이때 Win32의 `QueryPerformanceCounter()`를 이용하면 64비트 정수 형태의 하드웨어 카운터를 얻을 수 있다. `QueryPerformanceFrequency()`을 통해 이 카운터의 주파수 (1초당 카운터 수)를 얻을 수 있으므로, 두 값을 이용하면 시간을 구할 수 있다.

```cpp
__int64 countsPerSec;
QueryPerformanceFrequency((LARGE_INTEGER*)&countsPerSec);
mSecondsPerCount = 1.0 / (double)countsPerSec;

QueryPerformanceCounter((LARGE_INTEGER*)&currTime);

return (float)(currTime * mSecondsPerCount);
```

# `GameTimer` 클래스
이를 이용해서 GameTimer 클래스를 만들어본다. 현실에서 실제로 지나가는 시간과 다르게, 게임 내 시스템에서 시간을 정지시켜야 할 때가 있다. 이를 고려하여 시간 측정은 다음과 같이 이루어진다. 시각 측정은 모두 카운터 단위로 되고, 전체 시간이나 deltatime을 얻을 때에 초로 단위를 변환시킨다.
1. 시작 시각(`mBaseTime`)을 기록한다.
2. 매 프레임 현재 시각(`mCurrTime`)을 측정하고, 이전 프레임의 시각(`mPrevTime`)과 비교하여 그 사이 시간(`mDeltaTime`)을 알아낸다.
3. 게임을 정지할 때에는 정지한 시각(`mStopTime`)을 기록한다.
4. 게임을 다시 시작할 때에는 그동한 정지했던 시간을 누적해서 기록(`mPausedTime`)한다.
5. 게임 내에서 얼마나 시간이 흐른지 알고싶으면 (`현재시각` - `시작시각` - `정지된 상태일때 지나간 시간`)을 계산한다.

이를 코드로 바꾸면 다음과 같다. 자세한 코드는 [링크](https://github.com/williambong/3D_Game_Programming_with_DirectX_11/blob/master/Common/GameTimer.cpp)를 참조하자.
```cpp
class GameTimer
{
public:
	GameTimer();

	float TotalTime()const;  // pause 상태였던(그리고 현재 상태인) 경우를 고려하여 오직 pause되지 않은 시간만 리턴
	float DeltaTime()const;

	void Reset(); // 모든 값 초기화. mBaseTime을 새로 고침
	void Start(); // pause가 끝났을 때 호출. StopTime을 이용해서 pause후 지금까지의 시간을 mPausedTime에 누적
	void Stop();  // pause할때 호출. StopTime을 현재시각으로 설정
	void Tick();  // 매 프레임마다 호출. deltatime 계산

private:
	double mSecondsPerCount;
	double mDeltaTime;

	__int64 mBaseTime;
	__int64 mPausedTime;
	__int64 mStopTime;
	__int64 mPrevTime;
	__int64 mCurrTime;

	bool mStopped;
};
```


---
출처 : Introduction to 3D Game Programming with DirectX 11  
<https://learn.microsoft.com/ko-kr/windows/win32/sysinfo/acquiring-high-resolution-time-stamps>  
<https://github.com/williambong/3D_Game_Programming_with_DirectX_11/tree/master>