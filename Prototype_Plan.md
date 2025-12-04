### 1\. 🔄 알고리즘 흐름도 (Algorithm Flowchart)

시스템의 작동 순서를 \*\*[앱(App)]\*\*과 \*\*[아두이노(Arduino)]\*\*의 상호작용 중심으로 시각화했습니다.

```mermaid
graph TD
    A[시작 (초기 상태)] -->|배경: 검정 / 버튼: 숨김| B(아두이노: 무게 감지 대기)
    B --> C{무게 20kg 이상?}
    C -- 아니오 --> B
    C -- 예 (신호 '1' 전송) --> D[앱: 배경 흰색 변경 & LOCK 버튼 표시]
    
    D --> E{사용자 LOCK 버튼 클릭?}
    E -- 아니오 --> D
    E -- 예 (신호 'L' 전송) --> F[상태 전환: 잠금 모드]
    
    F --> G[아두이노: 서보모터 잠금 동작]
    F --> H[앱: 버튼 'EMERGENCY'로 변경]
    F --> I[앱: 타이머 시작 05:00 카운트다운]
    
    I --> J{EMERGENCY 클릭?}
    J -- 예 (신호 'U' 전송) --> K[즉시 잠금 해제 & 초기화]
    J -- 아니오 --> L{타이머 종료?}
    
    L -- 아니오 --> I
    L -- 예 (신호 'U' 전송) --> M[성공 시퀀스]
    
    M --> N[아두이노: 서보모터 원위치 복구]
    M --> O[앱: 'You did it!' 5초간 출력]
    O --> A
    K --> P[아두이노: 서보모터 원위치 복구]
    P --> A
```

-----

### 2\. 📱 MIT App Inventor 개발 계획서

아래 내용을 참고하여 앱 인벤터 프로젝트를 구성하시면 됩니다.

#### **1. 프로젝트 개요**

  * **프로젝트명:** Smart Chair Controller (Prototype)
  * **목표:** 블루투스 통신을 통해 아두이노의 무게 센서 값을 수신하고, 서보모터를 제어하여 의자의 잠금/해제 기능을 수행하는 안드로이드 앱 구현.
  * **핵심 기능:**
    1.  착석 감지 시 UI 자동 전환 (Dark → Light 모드)
    2.  타이머(5분) 기반 잠금 시스템
    3.  긴급 해제(Emergency) 및 성공 피드백(You did it\!)

#### **2. 통신 프로토콜 (약속된 신호)**

앱과 아두이노가 서로 주고받을 신호를 미리 정의합니다.

  * **아두이노 $\rightarrow$ 앱 (데이터 수신)**
      * `1`: 20kg 이상 감지됨 (착석)
      * `0`: 20kg 미만 (공석)
  * **앱 $\rightarrow$ 아두이노 (명령 송신)**
      * `L` (Lock): 서보모터 잠금 실행 (두 모터가 안쪽으로 회전)
      * `U` (Unlock): 서보모터 해제 실행 (원위치로 복귀)

#### **3. UI 화면 구성 (Designer 탭)**

| 컴포넌트 이름 | 종류 | 속성 설정 및 역할 |
| :-- | :-- | :-- |
| **Screen1** | 화면 | 초기 배경색: `Black` |
| **ListPicker1** | 리스트피커 | 블루투스 연결/해제 버튼 |
| **Label\_Timer** | 레이블 | 초기 텍스트: `00:00` (평소엔 숨김 또는 흐리게 표시) |
| **Button\_Action** | 버튼 | 초기 텍스트: `LOCK` / 초기 상태: `Visible = False` (숨김) |
| **Label\_Msg** | 레이블 | 성공 메시지(`You did it!`) 표시용 / 초기: 숨김 |
| **BluetoothClient1** | 연결 | 아두이노(HC-06)와 통신 담당 |
| **Clock1** | 시계 | 블루투스 데이터 수신 주기 (TimerInterval: 100\~200ms) |
| **Clock\_Countdown** | 시계 | 5분 카운트다운용 (TimerInterval: 1000ms, 초기: 비활성) |

#### **4. 블록 코딩 로직 (Blocks 탭)**

**Step 1: 무게 감지 및 UI 변화 (Clock1 타이머)**

  * **기능:** `BluetoothClient1`에서 데이터가 들어오는지 0.2초마다 확인.
  * **로직:**
      * 만약 받은 데이터가 `1` (착석) 이라면 $\rightarrow$ `Screen1` 배경색을 `White`로, `Button_Action`을 보이게(True) 설정.
      * 만약 받은 데이터가 `0` (공석) 이라면 $\rightarrow$ `Screen1` 배경색을 `Black`으로, `Button_Action`을 숨김(False).

**Step 2: LOCK 버튼 클릭 (잠금 시작)**

  * **기능:** 사용자가 `Button_Action`(`LOCK`)을 누름.
  * **로직:**
    1.  블루투스로 텍스트 `L` 전송 (아두이노가 서보모터를 움직임).
    2.  `Button_Action`의 텍스트를 `EMERGENCY`로 변경.
    3.  `Button_Action`의 배경색을 빨간색으로 변경 (경고 느낌).
    4.  `Label_Timer`를 `05:00`으로 설정.
    5.  `Clock_Countdown` 타이머 활성화 (Enabled = True).

**Step 3: 5분 카운트다운 및 성공 (Clock\_Countdown 타이머)**

  * **기능:** 1초마다 시간이 줄어듦.
  * **로직:**
      * 남은 시간(초)을 1씩 감소시키며 `Label_Timer` 갱신 (예: 04:59, 04:58...).
      * **시간이 0이 되면:**
        1.  블루투스로 텍스트 `U` 전송 (잠금 해제).
        2.  `Clock_Countdown` 비활성화.
        3.  `Label_Msg`에 `You did it!` 표시 및 보이게 설정.
        4.  (별도의 시계 또는 딜레이 후) 5초 뒤 초기 화면(검은 배경)으로 리셋.

**Step 4: EMERGENCY 버튼 클릭**

  * **기능:** 카운트다운 도중 버튼을 누름.
  * **로직:**
      * (버튼 텍스트가 `EMERGENCY`일 때만 동작하도록 `if`문 사용)
      * 블루투스로 텍스트 `U` 전송 (즉시 해제).
      * `Clock_Countdown` 비활성화.
      * 화면 초기화 (배경 검정, 타이머 00:00 등).

-----

### 3\. 아두이노 제어 코드 (참고용 로직)

앱 인벤터와 짝이 되는 아두이노 코드는 다음과 같은 형태여야 합니다.

```cpp
#include <Servo.h>
#include <SoftwareSerial.h>

Servo servoLeft;  // 왼쪽 서보
Servo servoRight; // 오른쪽 서보
SoftwareSerial BT(2, 3); // RX, TX (블루투스)

void setup() {
  servoLeft.attach(8);
  servoRight.attach(9);
  
  // 초기 상태: 열림 (Open)
  servoLeft.write(0);   
  servoRight.write(180); // 서로 반대 방향이므로 각도가 다름

  BT.begin(9600);
}

void loop() {
  // 1. 무게 감지 및 앱으로 상태 전송
  int weight = GetWeight(); // 로드셀 값 읽기 함수 (가상)
  
  if (weight >= 20) {
    BT.write("1"); // 착석 신호
  } else {
    BT.write("0"); // 공석 신호
  }

  // 2. 앱에서 명령 수신
  if (BT.available()) {
    char cmd = BT.read();
    
    if (cmd == 'L') { // 잠금 명령
      servoLeft.write(90);  // 중앙으로 이동
      servoRight.write(90); // 중앙으로 이동 (맞물림)
    } 
    else if (cmd == 'U') { // 해제 명령
      servoLeft.write(0);    // 원위치
      servoRight.write(180); // 원위치
    }
  }
  delay(200); // 통신 안정성을 위한 딜레이
}
```

이 계획서를 바탕으로 앱 인벤터에서 블록을 조립하시면 됩니다. 혹시 블록 조립 과정에서 "이 블록은 어디에 있나요?" 같은 질문이 생기면 언제든 물어봐 주세요\!
