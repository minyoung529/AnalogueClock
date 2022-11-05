![image](https://user-images.githubusercontent.com/77655318/200126073-aba2c032-76f1-47dd-9cce-eca9bd8d22f0.png)

오늘은 WinAPI로 만들 수 있는 아주 간단한 시계를 만들 것이다.

[티스토리 링크](https://minyoung529.tistory.com/81)

삼각함수, 라인, PtInRect에 대해서 감을 잡을 수 있었다.

---

##   **세팅**  

#### **1\. 상수 세팅하기**

![image](https://user-images.githubusercontent.com/77655318/200126080-ac793564-0a89-4204-bb2e-1569cc0316c3.png)

상수를 모아서 넣을 창고 헤더를 만들어준다.

```
#pragma once

#define PI					3.141592	
#define DEG2RAD				PI/180			// degree to radian

#define HOUR_COLOR			RGB(243,97,166)	// 시침(버튼) 색상
#define MINUTE_COLOR		RGB(107,102,255)// 분침(버튼) 색상
#define SECOND_COLOR		RGB(47,157,39)	// 초침(버튼) 색상

#define CIRCLE_COLOR		RGB(255,216,216)// 원 색상
#define GRADUATION_COLOR	RGB(95,0,255)	// 눈금 색상
#define STROKE_COLOR		RGB(0,0,66)		// 시계 테두리 색상

#define SOUND_COLOR			RGB(250,237,125)// 사운드 버튼 색상

#define TIMER_DEFAULT		1				// 0.06초에 한번씩 돌아가는 타이머 코드
#define TIMER_SECOND		2				// 1초에 한번씩 돌아가는 타이머 코드

#define SCREEN_WIDTH		400				// 화면 가로
#define SCREEN_HEIGHT		550				// 화면 새로

// 상태 온오프 플래그 (비트마스크)
#define	HOUR				1
#define	MINUTE				2
#define	SECOND				4
#define	SOUND				8
```

시계를 만들 때 사용할 상수이다.

---

##   **시계 객체 만들기**  

main cpp에다가 모두 담지 않고 시계 객체, 바늘 객체, 버튼 객체 등 객체로 만들어 오브젝트를 관리할 것이다.

### **1\. 시계 객체 개요**

시계 객체는 반지름, 위치, 시계 상태와 같은 시계 전체에 관련된 것과 각 시곗바늘 객체들로 구성되어있다.

> Clock.h

```
#pragma once
#include "framework.h"
#include "Define.h"

class Hand;

class Clock
{
private:
	int m_radius;			// 시계의 반지름
	POINT m_position;		// 현재 센터 위치
	int m_clockState = HOUR | MINUTE | SECOND | SOUND;	// 비트마스크 (현 상태: 시침, 분침, 초침, 사운드 모든 걸 표현)

	Hand* m_hourHand;		// 시침 객체
	Hand* m_minuteHand;		// 분침 객체
	Hand* m_secondHand;		// 초침 객체

public:
	Clock();
	Clock(int radius, POINT position);
	~Clock();

public:
	void Init();
	void RenderClock(SYSTEMTIME time, HDC hdc);			// 시계 출력
	void PlaySound(SYSTEMTIME time);					// 사운드

	void AddClockState(int state)						// 시계 상태 더하기
	{
		m_clockState |= state;
	}

	void RemoveClockState(int state)					// 시계 상태 빼기
	{
		m_clockState ^= state;
	}

private:
	void RenderHour(SYSTEMTIME systemTime, HDC hdc);	// 시침 출력
	void RenderMinute(SYSTEMTIME systemTime, HDC hdc);	// 분침 출력
	void RenderSecond(SYSTEMTIME systemTime, HDC hdc);	// 초침 출력

	void RenderGraduation(HDC hdc);						// 눈금 출력
	void RenderNumber(HDC hdc);							// 숫자 출력
	void RenderCircle(HDC hdc);							// 시계 원 출력
};
```

### **2\. 시계 객체 구현**

> 생성시 (생성자, 초기화 함수)

```
Clock::Clock() : m_radius(0), m_position{}
{}

Clock::Clock(int radius, POINT position)
	: m_radius(radius)
	, m_position(position)
	, m_clockState(HOUR | SECOND | MINUTE | SOUND)
	, m_hourHand(nullptr), m_minuteHand(nullptr), m_secondHand(nullptr)
{}

Clock::~Clock()
{
	delete m_hourHand;
	delete m_minuteHand;
	delete m_secondHand;
}

void Clock::Init()
{
	// 시곗바늘 객체 만들기
	m_hourHand   = new Hand(HOUR_COLOR, 15, 80, m_position);
	m_minuteHand = new Hand(MINUTE_COLOR, 10, 100, m_position);
	m_secondHand = new Hand(SECOND_COLOR, 5, 125, m_position);
}
```

> 그리기 (Render)

```
void Clock::RenderClock(SYSTEMTIME time, HDC hdc)
{
	RenderCircle(hdc);
	RenderGraduation(hdc);
	RenderNumber(hdc);

	// 현재 상태에 시침, 분침, 초침이 각각 있다면...
	if (m_clockState & SECOND)
		RenderSecond(time, hdc);

	if (m_clockState & MINUTE)
		RenderMinute(time, hdc);

	if (m_clockState & HOUR)
		RenderHour(time, hdc);
}

// 시침 출력
void Clock::RenderHour(SYSTEMTIME time, HDC hdc)
{
	// minute을 더한 이유:	시간이 흘러감에 따라= 자연스럽게 시침이 변하는 것을 표현하기 위해서
	// ex ) 6시 40분			=> 6 * 30 + 40 / 2 => 180 + 20 => 200 degrees

	float x =  sin(DEG2RAD * ((double)time.wHour * 30 + time.wMinute / 2));
	float y = -cos(DEG2RAD * ((double)time.wHour * 30 + time.wMinute / 2));

	m_hourHand->Render(hdc, x, y);
}

// 분침 출력
void Clock::RenderMinute(SYSTEMTIME time, HDC hdc)
{
	float x =  sin(DEG2RAD * time.wMinute * 6);
	float y = -cos(DEG2RAD * time.wMinute * 6);

	m_minuteHand->Render(hdc, x, y);
}

// 초침 출력
void Clock::RenderSecond(SYSTEMTIME time, HDC hdc)
{
	float x =  sin(DEG2RAD * time.wSecond * 6);
	float y = -cos(DEG2RAD * time.wSecond * 6);

	m_secondHand->Render(hdc, x, y);
}
```

> RenderClock => 각 렌더 함수들을 호출함으로써 시계 전체를 그리는 함수

시침, 분침, 초침의 방향을 각각 객체에 넘겨주었다.  이때 삼각함수의 개념이 쓰이게 된다.

초침과 분침같은 경우에는 각 초와 분이 1, 2, 3일 때 시곗바늘의 방향 또한 같아서 코드가 똑같다. 각 초와 분이 1씩 늘어날 때마다 각도는 360/60 = **6**씩 오르게 된다. 

```
 sin(DEG2RAD * time.wSecond * 6);
-cos(DEG2RAD * time.wSecond * 6);
```

따라서 이와 같이 6을 곱해주고, sin cos 함수에 넣어야 하기 때문에 상수 함수에서 선언해준 DEG2RAD을 곱해주면 방향이 나오게 된다.

시침은 조금 다른데, 일단 시간에 따른 방향만 보면, 시가 1씩 늘어날 때마다 360/12 = **30**씩 오른다.

```
 sin(DEG2RAD * ((double)time.wHour * 30);
-cos(DEG2RAD * ((double)time.wHour * 30);
```

이것에 더해, 시침은 원래 분침에 따라서 방향이 조금씩 변한다.

분이 1씩 늘어나면 각도는 30/60씩 늘어나게 되므로...

```
 sin(DEG2RAD * ((double)time.wHour * 30 + time.wMinute / 2));
-cos(DEG2RAD * ((double)time.wHour * 30 + time.wMinute / 2));
```

시침 방향도 완성 ^\_\_^!!

> 눈금 출력

```
void Clock::RenderGraduation(HDC hdc)
{
	float radius = this->m_radius + 15;

	HPEN hPen = CreatePen(PS_SOLID, 2, GRADUATION_COLOR);
	HPEN hOldPen = (HPEN)SelectObject(hdc, hPen);

	for (int i = 0; i < 60; i++)
	{
		float x = sin(DEG2RAD * i * 6);
		float y = -cos(DEG2RAD * i * 6);

		// 5분마다 더 긴 눈금
		float len = (i % 5 == 0) ? 15 : 7;

		POINT startPos = { m_position.x + x * radius,	 m_position.y + y * radius };
		POINT endPos = { startPos.x - x * len, startPos.y - y * len};

		MoveToEx(hdc, startPos.x, startPos.y, nullptr);
		LineTo(hdc, endPos.x, endPos.y);
	}

	SelectObject(hdc, hOldPen);
	DeleteObject(hPen);
}
```

**여기 또한 삼각함수가 들어간다.**

```
POINT startPos = { m_position.x + x * radius,	 m_position.y + y * radius };
POINT endPos = { startPos.x - x * len, startPos.y - y * len};
```

![image](https://user-images.githubusercontent.com/77655318/200126110-e66d4007-01aa-4ab6-90a5-e44e89274b70.png)

이렇게 표현하면 이해가 쉬울 것 같다.

반지름 길이로 반복문의 방향 쪽을 간 점이 start point,

len만큼 startpoint에서 역방향으로 간 점이 end point이다.

```
// 시계 숫자 출력
void Clock::RenderNumber(HDC hdc)
{
	RECT rt;
	float space = 0.90f;

	for (int i = 0; i < 12; i++)
	{
		float x = m_position.x + sin(DEG2RAD * i * 30) * m_radius * space;
		float y = m_position.y - cos(DEG2RAD * i * 30) * m_radius * space;

		rt.left = x - 4;
		rt.right = x + 4;
		rt.top = y - 4;
		rt.bottom = y + 4;

		// 중앙 정렬
		SetTextAlign(hdc, TA_CENTER);

		// 숫자 출력
		TCHAR buffer[3];
		int time = (i == 0) ? 12 : i;
		wsprintf(buffer, TEXT("%d"), time);
		TextOut(hdc, x, y - 5, buffer, lstrlen(buffer));
	}
}
```

**숫자 출력은 눈금과 별로 다르지 않다.**

> 원 출력

```
void Clock::RenderCircle(HDC hdc)
{
	HBRUSH hBrush = CreateSolidBrush(STROKE_COLOR);
	HBRUSH oldBrush = (HBRUSH)SelectObject(hdc, hBrush);

	// 여유있게
	int radius = m_radius + 15;
    // 시계 테두리 굵기
	int stroke = 7;

	// 테두리 시계 먼저
	Ellipse(hdc, m_position.x - radius - stroke, m_position.y + radius + stroke, m_position.x + radius + stroke, m_position.y - radius - stroke);
	DeleteObject(hBrush);

	hBrush = CreateSolidBrush(CIRCLE_COLOR);

	SelectObject(hdc, hBrush);
    // 시계 원
	Ellipse(hdc, m_position.x - radius, m_position.y + radius, m_position.x + radius, m_position.y - radius);

	SelectObject(hdc, oldBrush);
	DeleteObject(hBrush);
}
```

> 소리 출력

```
void Clock::PlaySound(SYSTEMTIME time)
{
	if (m_clockState & SOUND)
	{
		if (time.wSecond >= 57 && time.wSecond <= 59)
			Beep(500, 400);

		else if (time.wSecond == 0)
			Beep(1000, 1000);
	}
}
```

시계 보면 57초부터 삐- 삐- 삐- 삐이이이 하는 코드이다.

### **2\. 바늘 객체 구현**

![image](https://user-images.githubusercontent.com/77655318/200126124-5ad8781a-b2fc-4bd6-9d3b-a8cafee50f08.png)

**바늘 객체는 별거 없다. Render만 하는 객체이다.**

> Hand.h

```
#pragma once
#include "framework.h"

class Hand
{
public:
	Hand();
	Hand(COLORREF color, int thickness, int length, POINT center);
	~Hand();

private:
	COLORREF m_color;	// 바늘 색상
	int m_thickness;	// 바늘 굵기
	int m_length;		// 바늘 길이
	POINT m_center;		// 시계 센터 포지션

public:
	void Render(HDC hdc, float sinX, float cosY);
};
```

> Hand.cpp

```
#include "Hand.h"
#include "Define.h"

Hand::Hand() : Hand(RGB(0, 0, 0), 1, 10, { SCREEN_WIDTH * 1 / 2, SCREEN_HEIGHT * 1 / 2 })
{}

Hand::Hand(COLORREF color, int thickness, int length, POINT center)
	: m_color(color)
	, m_thickness(thickness)
	, m_length(length)
	, m_center(center)
{}

Hand::~Hand()
{}

void Hand::Render(HDC hdc, float sinX, float cosY)
{
	HPEN hPen = CreatePen(PS_SOLID, m_thickness, m_color);
	HPEN hOldPen = (HPEN)SelectObject(hdc, hPen);

	// startPos에 x y를 더한 이유:	시침이 정 중앙이 아닌 방향의 반대쪽부터 시작하기 때문! (빠져나온 길이)
	POINT startPos = { m_center.x - sinX * 10, m_center.y - cosY * 10 };
	POINT endPos = { m_center.x + sinX * m_length, m_center.y + cosY * m_length };

	MoveToEx(hdc, startPos.x, startPos.y, nullptr);
	LineTo(hdc, endPos.x, endPos.y);

	SelectObject(hdc, hOldPen);
	DeleteObject(hPen);
}
```

위 시계 출력을 이해했다면, 시곗바늘 출력 또한 설명이 없어도 된다.

---

##   **버튼**  

토글 버튼 베이스인 Button,

클래스를 상속받은 ClockToggle, 

ClockToggle을 관리하는 ToggleManager로 구성된다.

### **1\. 버튼 베이스**

> Button.h

```
#pragma once
#include "Clock.h"

class Button
{
protected:
	COLORREF m_color;	// 버튼 색상
	POINT m_pos;		// 버튼 센터 포지션
	POINT m_scale;		// 버튼 스케일 (가로, 세로)
	RECT m_rect;		// 버튼 렉트
	bool m_toggle;		// 토글

public:
	Button();
	Button(COLORREF color, POINT pos, POINT scale);

	virtual ~Button() {}

public:
	void Init(HWND hWnd);
	void Update(HWND hWnd);
	virtual void Render(HDC hdc);

	// 버튼이 눌렸을 때
	// 자식이 정의
	virtual void OnClickButton() = 0;

private:
	void SetRect();		// Rect 초기세팅
};
```

**자식이 OnClickButton을 정의해서 쓸 수 있다. 따라서 Button은 추상 클래스이고 그 자체로 인스턴스가 되지 않는다.**

> Button.cpp

```
Button::Button() : Button(RGB(0, 0, 0), {}, { 20,15 })
{
}

Button::Button(COLORREF color, POINT pos, POINT scale) :
	m_color(color), m_pos(pos), m_scale(scale), m_toggle(false)
{
	SetRect();
}

void Button::Init(HWND hWnd)
{
	HDC hdc = GetDC(hWnd);
	Render(hdc);
	ReleaseDC(hWnd, hdc);
}

void Button::SetRect()
{
	m_rect.left		= m_pos.x - m_scale.x / 2;
	m_rect.right	= m_pos.x + m_scale.x / 2;
	m_rect.top		= m_pos.y - m_scale.y / 2;
	m_rect.bottom	= m_pos.y + m_scale.y / 2;
}
```

> 마우스 충돌, Down을 처리

```
void Button::Update(HWND hWnd)
{
	POINT mousePoint;
	GetCursorPos(&mousePoint);
	ScreenToClient(hWnd, &mousePoint);

	// 마우스와 충돌 감지
	if (PtInRect(&m_rect, mousePoint))
	{
		if (GetAsyncKeyState(VK_LBUTTON) & 1)
		{
			// 충돌하면 토글을 반대로, OnClickButton을 호출한다
			m_toggle = !m_toggle;
			OnClickButton();
		}
	}
}
```

PtInRect로 버튼과 마우스 포인트의 충돌을 감지한다.

감지가 되었다면, 버튼을 눌렀음을 체크해 토글을 반대로, 자식이 정의할 OnClickButton을 호출한다.

> Button Render

```
void Button::Render(HDC hdc)
{
	// 더블 버퍼링
	HDC memDC = CreateCompatibleDC(hdc);
	HBITMAP bitmap = CreateCompatibleBitmap(hdc, SCREEN_WIDTH, SCREEN_HEIGHT);
	SelectObject(memDC, bitmap);

	COLORREF buttonColor = (m_toggle) ? RGB(150, 150, 150) : m_color;
	HBRUSH hBlueBrush = CreateSolidBrush(buttonColor);
	HBRUSH oldBrush = (HBRUSH)SelectObject(memDC, hBlueBrush);

	Rectangle(memDC, m_rect.left, m_rect.top, m_rect.right, m_rect.bottom);
	BitBlt(hdc, m_rect.left, m_rect.top, m_scale.x, m_scale.y, memDC, m_rect.left, m_rect.top, SRCCOPY);

	DeleteObject(hBlueBrush);
	DeleteDC(memDC);
}
```

버튼의 렌더를 해준다.

더블 버퍼링으로 구현했다.

### **2\. 시계 토글 버튼**

**ClockToggle은 Button을 상속받고, 시계 포인터, 담당하는 시계 상태, 이름을 가지고 있다.**

**또한 OnClickButton을 정의한다.**

> ClockToggle.h

```
#pragma once
#include "Button.h"
#include "Clock.h"
#include "framework.h"

class ClockToggle : public Button
{
private:
	Clock* m_clock;
	int m_onOffState;
	TCHAR m_name[10];

public:
	ClockToggle();
	ClockToggle(COLORREF color, POINT pos, POINT scale, Clock* clock, int onOffState, const wchar_t* buttonName);
	~ClockToggle() {}

public:
	void OnClickButton() override;
	virtual	void Render(HDC hdc) override;
};
```

> ClockToggle.cpp

```
#pragma once
#include "ClockToggle.h"

ClockToggle::ClockToggle() : Button()
{
	m_clock = NULL;
	m_onOffState = 0;
}

ClockToggle::ClockToggle(COLORREF color, POINT pos, POINT scale, Clock* clock, int onOffState, const wchar_t* buttonName)
	: Button(color, pos, scale)
	, m_clock(clock)
	, m_onOffState(onOffState)
{
	wsprintf(m_name, buttonName);
}

void ClockToggle::OnClickButton()
{
	if (m_toggle)
	{
		// 시계에서 가지고 있는 상태를 뺀다
		m_clock->RemoveClockState(m_onOffState);
	}
	else
	{
		// 시계에서 가지고 있는 상태를 다시 더한다
		m_clock->AddClockState(m_onOffState);
	}
}

void ClockToggle::Render(HDC hdc)
{
	// 버튼 렉트 출력
	Button::Render(hdc);

	// 버튼 이름 출력
	SetTextAlign(hdc, TA_CENTER);	// 중앙 정렬
	TextOut(hdc, m_rect.left + m_scale.x / 2, m_rect.top + m_scale.y / 4, m_name, lstrlen(m_name));
}
```

멤버 변수가 가리키는 시계 객체에서 상태(시침, 분침, 초침, 사운드)를 빼거나 더하는 코드 완성 ^\_\_^!

### **3\. 토글 매니저**

딱히 하는 건 없지만, 4개의 버튼을 생성하고 관리한다.

> ToggleManager.h

```
#pragma once
#include "framework.h"

class ClockToggle;
class Clock;

class ToggleManager
{
public:
	void Init(Clock* clock);
	void Update(HWND hWnd);
	void Render(HDC hdc);
	void Release();

private:
	// 토글 버튼 UI
	ClockToggle* m_hourToggle;
	ClockToggle* m_minuteToggle;
	ClockToggle* m_secondToggle;
	ClockToggle* m_soundToggle;
};
```

> ToggleManager.cpp

```
#include "ToggleManager.h"
#include "ClockToggle.h"
#include "Define.h"

void ToggleManager::Init(Clock* clock)
{
	// 시계 토글들 생성
	m_hourToggle = new ClockToggle(HOUR_COLOR, { 40,450 }, { 60, 40 }, clock, HOUR, L"시침");
	m_minuteToggle = new ClockToggle(MINUTE_COLOR, { 140,450 }, { 60, 40 }, clock, MINUTE, L"분침");
	m_secondToggle = new ClockToggle(SECOND_COLOR, { 240,450 }, { 60, 40 }, clock, SECOND, L"초침");
	m_soundToggle = new ClockToggle(SOUND_COLOR, { 340,450 }, { 60, 40 }, clock, SOUND, L"소리");
}

void ToggleManager::Update(HWND hWnd)
{
	m_hourToggle->Update(hWnd);
	m_minuteToggle->Update(hWnd);
	m_secondToggle->Update(hWnd);
	m_soundToggle->Update(hWnd);
}

void ToggleManager::Render(HDC hdc)
{
	m_hourToggle->Render(hdc);
	m_minuteToggle->Render(hdc);
	m_secondToggle->Render(hdc);
	m_soundToggle->Render(hdc);
}

void ToggleManager::Release()
{
	delete m_hourToggle;
	delete m_minuteToggle;
	delete m_secondToggle;
	delete m_soundToggle;
}
```

**버튼도 완성이다 ^\_\_^!!!**

---

##   **프로그램 루프**  

#### **1\. 게임 객체 만들기**

![image](https://user-images.githubusercontent.com/77655318/200126143-3d17ecea-6117-40cc-8615-cef645b79afd.png)


> Game.h

```
#pragma once
#include "Clock.h"
#include "ClockToggle.h"

class ToggleManager;

class Game
{
public:
	Game();
	~Game();

private:
	HWND m_hWnd;					// 윈도우 핸들
	SYSTEMTIME m_systemTime;		// 가져올 시스템 시간
	int m_radius;					// 시계 반지름

private:
	
	Clock* m_clock;					// 시계 객체
	ToggleManager* m_toggleManager;	// 버튼 관리자

public:
	void Init(HWND hWnd);
	void Update();
	void Render(HDC hdc);
	void Release();
	void UpdatePerSec();			// 1초에 한번씩 호출
};
```

지금까지 구현한 클래스들을 최종적으로 관리하고 굴리는 루프이다.

> Game.cpp

```
#include "Game.h"
#include "Define.h"
#include "ToggleManager.h"

Game::Game()
	: m_hWnd(NULL)
	, m_systemTime{}
	, m_radius(0)
	, m_clock(nullptr)
{}

Game::~Game() {}

void Game::Init(HWND hWnd)
{
	m_hWnd = hWnd;

	RECT rt;
	GetClientRect(m_hWnd, &rt);
	int width = rt.right - rt.left;
	int height = rt.bottom - rt.top;

	// 작은 쪽 * 0.4를 반지름으로
	m_radius = (width > height) ? height * 0.4f : width * 0.4f;

	// 시계 생성
	m_clock = new Clock(m_radius, { (LONG)(width * 0.5f), (LONG)(height * 0.45f) });
	m_clock->Init();

	m_toggleManager = new ToggleManager();
	m_toggleManager->Init(m_clock);

	SetTimer(m_hWnd, TIMER_DEFAULT, 60, nullptr);
	SetTimer(m_hWnd, TIMER_SECOND, 1000, nullptr);
}

void Game::Update()
{
	m_toggleManager->Update(m_hWnd);
}

void Game::Render(HDC hdc)
{
	GetLocalTime(&m_systemTime);

	// 시계 출력하기 
	m_clock->RenderClock(m_systemTime, hdc);
	m_toggleManager->Render(hdc);
}

void Game::Release()
{
	delete m_clock;
}

void Game::UpdatePerSec()
{
	m_clock->PlaySound(m_systemTime);
}
```

이제 메인에서 게임 인스턴스를 만들어 호출해줄 때가 되었다!!!

#### **2\. 메인에서 게임 객체 호출**

```
#include "framework.h"
#include "AnalogueClock.h"
#include "Define.h"
#include "Game.h"
#include <math.h>

// 전역 변수:
//...
Game* game;

int APIENTRY wWinMain(_In_ HINSTANCE hInstance,
// ...

ATOM MyRegisterClass(HINSTANCE hInstance)
//...

BOOL InitInstance(HINSTANCE hInstance, int nCmdShow)
{
	hInst = hInstance; // 인스턴스 핸들을 전역 변수에 저장합니다.

	// ----- 여기서 화면 크기 설정 ------ 
	HWND hWnd = CreateWindowW(szWindowClass, szTitle, WS_OVERLAPPEDWINDOW,
		0, 0, SCREEN_WIDTH, SCREEN_HEIGHT, nullptr, nullptr, hInstance, nullptr);
	
    //...
}

LRESULT CALLBACK WndProc(HWND hWnd, UINT message, WPARAM wParam, LPARAM lParam)
{
	switch (message)
	{
	case WM_CREATE:
    	// 게임 객체를 생성하고 초기화
		game = new Game();
		game->Init(hWnd);
		break;

	case WM_TIMER:
		switch (wParam)
		{
		case TIMER_DEFAULT:
		{
        	// Update, Render
			game->Update();
			HDC hdc = GetDC(hWnd);
			game->Render(hdc);
			ReleaseDC(hWnd, hdc);
		}break;

		case TIMER_SECOND:
        	// 1초에 한번씩
			game->UpdatePerSec();
			break;
		}
		break;
        
	case WM_DESTROY:
    	// 게임 종료
		game->Release();
		PostQuitMessage(0);
		break;
	default:
		return DefWindowProc(hWnd, message, wParam, lParam);
	}
	return 0;
}

INT_PTR CALLBACK About(HWND hDlg, UINT message, WPARAM wParam, LPARAM lParam)
//...
```

바꾸지 않은 부분은 //...으로 처리했고 게임 인스턴스를 만드니 메인 루프가 돌아가는 것을 어렴풋이 알 수 있을 것이다.

메인 루프가 돌아가는 곳에서 이 클래스를 인스턴스해서 Init, Update(PerSec), Render,를 호출하면 WinProc의 스위치문에 방대한 코드를 쓸 필요가 없다. 

즉 깔끔하다는 뜻 ^\_\_^!!

---

![image](https://user-images.githubusercontent.com/77655318/200126151-0c53af5a-bf37-4937-8629-63821f6a7ecb.png)


길고도 험했던 아날로그 시계 대장정을 끝냈다 ^\_\_^

만든 프로그램은 형편 없지만... 많은 노력이 들어갔다.

아날로그 시계를 만들며 WinAPI의 기본 틀, 삼각함수, 도형 출력, 객체에 익숙해진 것 같아서 기분이 좋다 \*^\_\_^!!!!!
