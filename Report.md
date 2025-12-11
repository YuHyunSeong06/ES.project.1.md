# [최종 결과 보고서] IoT 기반 습관 형성 스마트 의자 제작

**작성자:** 성유현
**작성일:** 2025년 12월 11일
**개발 환경:** Arduino IDE, Processing, MIT App Inventor

---

## 1. 서론 (Introduction)

### 1.1. 제작 동기 및 목적
* **배경:** 학업이나 업무 수행 시 집중력을 유지하지 못하고 빈번하게 자리를 이탈하는 문제는 생산성 저하의 주된 원인이 됨. 단순한 의지력에 의존하기보다 물리적인 환경 조성을 통해 착석을 유도할 필요성을 느낌.
* **목적:** 사용자의 착석 여부를 감지하고, 앱을 통해 스스로 설정한 시간 동안 의자의 잠금 장치를 작동시켜 강제적인 학습/업무 환경을 조성하는 **IoT 스마트 의자 프로토타입**을 구현함.

---

## 2. 시스템 개요 (System Overview)

### 2.1. 전체 시스템 구성도
본 시스템은 크게 **하드웨어(제어 및 구동)**, **모바일 앱(사용자 인터페이스)**, **PC 모니터링(데이터 관제)**의 세 부분으로 구성된다.

1.  **Input (감지):** 로드셀 센서를 통해 사용자의 체중을 실시간 측정.
2.  **Process (제어):** Arduino UNO가 무게 데이터를 분석하여 착석 여부(5kg 이상)를 판단하고, HC-06 블루투스 모듈을 통해 스마트폰과 통신.
3.  **Output (구동):** 사용자의 명령에 따라 서보모터가 작동하여 물리적 잠금(Lock) 및 해제(Unlock) 수행.
4.  **Interface (앱):** MIT App Inventor로 제작된 앱에서 타이머 및 잠금 제어.

---
## 3. 하드웨어 구현 (Hardware Implementation)

**3.1. 주요 부품 구성**
| 구분 | 부품명 | 역할 및 특징 |
| :-- | :-- | :-- |
| **MCU** | Arduino UNO | 전체 시스템 제어 및 센서 데이터 처리 |
| **센서** | **Load Cell (3-wire) x 1** | **프로토타입 구현을 위해 1개의 3선식 로드셀 사용** |
| **증폭기** | HX711 | 로드셀의 미세 전압 변화를 증폭하여 디지털 신호로 변환 |
| **저항** | **1kΩ Resistor x 2** | **단일 로드셀 사용을 위한 휘트스톤 브리지 회로 구성용 (Dummy Resistor)** |
| **액추에이터** | Parallax Continuous Servo | 연속 회전 서보모터. 정지값(1508)을 기준으로 회전 속도 제어 |
| **통신** | HC-06 Bluetooth | 아두이노와 스마트폰 간의 시리얼 무선 통신 |

**3.2. 회로 설계**
* **로드셀 결선 (Half-bridge 구성):** 프로토타입 단계에서의 효율적인 테스트를 위해 1개의 3선식 로드셀과 2개의 1kΩ 저항을 사용하여 휘트스톤 브리지 회로를 구성함.
    * [cite_start]로드셀의 신호선과 전원선을 HX711 입력단에 연결하고, 나머지 전압 분배 구간을 1kΩ 저항 2개로 대체하여 안정적인 무게 감지가 가능하도록 설계함[cite: 1].
* **통신 및 제어 핀 연결:**
    * **HX711:** DT $\rightarrow$ D3, SCK $\rightarrow$ D2 (데이터 및 클럭 신호)
    * **Servo:** Signal $\rightarrow$ D9 (PWM 제어)
    * **Bluetooth:** TX $\rightarrow$ D4, RX $\rightarrow$ D5 (SoftwareSerial 통신)
* **전원:** 아두이노 5V 출력을 로드셀 모듈과 서보모터의 전원으로 공통 사용 (USB 전원 기반).

---

## 4. 소프트웨어 구현 (Software Implementation)

### 4.1. Arduino 펌웨어 (Firmware)
* **무게 감지 알고리즘:** `calibration_factor`를 적용하여 원시 데이터를 무게(kg)로 변환. 20kg 임계값(Threshold)을 기준으로 착석(`1`)과 공석(`0`) 상태를 구분.
* **서보모터 제어:** 일반 각도 제어가 아닌 연속 회전 서보의 특성을 반영하여 구현.
    * **잠금:** `1700`(회전) 신호 전송 → `delay`(이동 시간) → `1508`(정지)
    * **해제:** `1300`(역회전) 신호 전송 → `delay`(이동 시간) → `1508`(정지)
 
```cpp
  #include "HX711.h"
#include <Servo.h>
#include <SoftwareSerial.h>

// 핀 설정
#define LOADCELL_DOUT_PIN 2
#define LOADCELL_SCK_PIN 3
#define BT_TX 4  // 블루투스 TX -> 아두이노 D4
#define BT_RX 5  // 블루투스 RX -> 아두이노 D5
#define SERVO_PIN 9 // 서보모터 신호선

HX711 scale;
Servo myServo;
SoftwareSerial BT(BT_TX, BT_RX);

float calibration_factor = 0.0; 
long weight_threshold = 20; // 20kg 기준

// 서보모터 설정값 (Parallax 연속회전 서보 기준)
const int SERVO_STOP = 1508; // 정지값 (사용자 지정)
const int SERVO_CW = 1300;   // 시계방향 회전 속도 (값 조절 가능)
const int SERVO_CCW = 1700;  // 반시계방향 회전 속도 (값 조절 가능)
const int MOVE_TIME = 1000;  // 모터가 움직이는 시간 (ms) -> 90도만큼 돌 시간을 테스트해서 수정 필요!

bool isLocked = false;

void setup() {
  Serial.begin(9600); // PC(Processing) 통신
  BT.begin(9600);     // 앱(App Inventor) 통신

  // 로드셀 초기화
  scale.begin(LOADCELL_DOUT_PIN, LOADCELL_SCK_PIN);
  scale.set_scale(calibration_factor);
  scale.tare(); 

  // 서보모터 초기화
  myServo.attach(SERVO_PIN);
  myServo.writeMicroseconds(SERVO_STOP); // 시작하자마자 정지 상태 유지
  isLocked = false;
}

void loop() {
  // 1. 무게 측정
  float weight = scale.get_units(5);

  // 2. 데이터 전송 (1: 착석, 0: 공석)
  // 값이 너무 튀는 것을 방지하기 위해 정수형으로 변환하거나 단순화
  if (weight >= weight_threshold) {
    Serial.println(1); // PC로 전송
    BT.write("1");     // 앱으로 전송
  } else {
    Serial.println(0); // PC로 전송
    BT.write("0");     // 앱으로 전송
  }

  // 3. 앱 명령 수신 (L: 잠금, U: 해제)
  if (BT.available()) {
    char cmd = BT.read();
    if (cmd == 'L') {
      lockChair();
    } else if (cmd == 'U') {
      unlockChair();
    }
  }
  delay(200);
}

// 잠금 함수
void lockChair() {
  if (!isLocked) {
    // 1. 잠금 방향으로 회전 시작
    myServo.writeMicroseconds(SERVO_CCW); // 1700 (방향 반대면 SERVO_CW로 변경)
    
    // 2. 90도 정도 움직일 때까지 기다림 (시간으로 제어)
    delay(MOVE_TIME); 
    
    // 3. 정지
    myServo.writeMicroseconds(SERVO_STOP); // 1508
    
    isLocked = true;
  }
}

// 해제 함수 (원상복구)
void unlockChair() {
  // 잠금 상태일 때만 동작 (중복 실행 방지)
  // 혹은 강제 해제를 위해 조건 없이 실행 가능
  
  // 1. 반대 방향(해제 방향)으로 회전 시작
  myServo.writeMicroseconds(SERVO_CW); // 1300 (방향 반대면 SERVO_CCW로 변경)
  
  // 2. 원래 위치로 돌아올 때까지 기다림
  delay(MOVE_TIME); 
  
  // 3. 정지
  myServo.writeMicroseconds(SERVO_STOP); // 1508
  
  isLocked = false;
}
```

### 4.2. 모바일 앱 (MIT App Inventor)
* **UI 디자인:**
    * 평상시: 검은색 배경 (비활성 상태)
    * 착석 시(무게 감지): 하얀색 배경으로 전환되며 **LOCK** 버튼 활성화.
* **주요 로직:**
    * **LOCK 버튼 클릭:** 서보모터 잠금 신호(`L`) 전송 → 30초 카운트다운 시작 → 버튼이 **EMERGENCY**로 변경됨.
    * **타이머 종료/EMERGENCY 클릭:** 해제 신호(`U`) 전송 → 시스템 초기화.

**[여기에 앱 인벤터 디자이너 화면 및 블록 코딩 캡처를 첨부하세요]**

### 4.3. PC 모니터링 (Processing)
* 아두이노와 Serial(USB) 통신을 통해 현재 센서 데이터 수신.
* 앱 화면을 보지 않고도 PC 화면에서 **"STATUS: SEATED"** 또는 **"EMPTY"** 상태를 시각적으로 모니터링 가능한 대시보드 구현.

```java
import processing.serial.*;

Serial COM4;  // 아두이노와 통신할 객체
String val;     // 수신된 데이터를 저장할 변수

void setup() {
  size(600, 600); // 창 크기 설정
  
  // [중요] 사용자의 PC에 연결된 아두이노 포트를 찾습니다.
  // 콘솔창(검은 부분)에 뜨는 목록을 보고, 아두이노가 연결된 번호(인덱스)를 적어야 합니다.
  printArray(Serial.list());
  
  // 보통 0번이나 1번 포트입니다. 연결이 안 되면 [0]을 [1]이나 [2]로 바꿔보세요.
  // Windows의 경우 "COM3" 처럼 직접 이름을 적어도 됩니다. 예: new Serial(this, "COM3", 9600);
  String portName = Serial.list()[0]; 
  
  myPort = new Serial(this, portName, 9600);
  
  // 폰트 및 정렬 설정
  textAlign(CENTER, CENTER);
  textSize(40);
}

void draw() {
  // 아두이노에서 데이터가 들어오면 읽기
  if (myPort.available() > 0) {
    val = myPort.readStringUntil('\n'); // 줄바꿈 문자까지 읽기
  }
  
  // 데이터가 비어있지 않다면 처리
  if (val != null) {
    val = trim(val); // 공백 제거 (매우 중요)
    
    // --- [1] 착석 감지 (20kg 이상) ---
    if (val.equals("1")) {
      background(255); // 하얀색 배경 (앱과 동일)
      
      fill(0); // 검은색 글씨
      text("STATUS: SEATED", width/2, height/2 - 50);
      text("Waiting for LOCK...", width/2, height/2 + 50);
      
      // 시각적 효과 (의자 아이콘 등)
      noFill();
      stroke(0);
      strokeWeight(5);
      rectMode(CENTER);
      rect(width/2, height/2, 400, 400); // 테두리 박스
    } 
    // --- [0] 공석 (20kg 미만) ---
    else if (val.equals("0")) {
      background(0); // 검은색 배경 (앱과 동일)
      
      fill(255); // 하얀색 글씨
      text("STATUS: EMPTY", width/2, height/2);
      
      // 시각적 효과
      stroke(255);
      strokeWeight(2);
      rectMode(CENTER);
      rect(width/2, height/2, 400, 400); // 테두리 박스
    }
  }
}
```

---

## 5. 결과 및 검증 (Results)

### 5.1. 기능 테스트 목표 결과
1.  **무게 인식:** 20kg 이상의 하중이 가해졌을 때, 앱 화면이 즉시 흰색으로 전환됨을 확인.
2.  **잠금 장치:** 앱에서 LOCK 버튼을 누르면 서보모터가 지정된 시간만큼 회전하여 잠금 위치로 이동 후 정지함.
3.  **타이머 연동:** 30초 카운트다운이 종료되면 자동으로 모터가 역회전하여 잠금 해제됨을 확인.
4.  **긴급 해제:** 카운트다운 도중 EMERGENCY 버튼 클릭 시 즉시 잠금 해제됨을 확인.

### 5.2. 기능 테스트 목표 실제 결과
1.  **무게 인식:** 20kg 이상의 하중이 가해졌을 때, 앱 화면이 즉시 흰색으로 전환됨을 확인했지만 hx117회로가 자꾸 빠지면서 무게에 오류가 생김
2.  **잠금 장치:** 앱에서 LOCK 버튼을 누르면 서보모터가 지정된 시간만큼 회전하여 잠금 위치로 이동 후 정지함.
3.  **타이머 연동:** 30초 카운트다운이 종료되면 자동으로 모터가 역회전하여 잠금 해제됨을 확인.
4.  **긴급 해제:** 카운트다운 도중 EMERGENCY 버튼 클릭 시 즉시 잠금 해제됨을 확인.

### 5.2. 문제 해결 과정 (Troubleshooting)
* **문제:** 서보모터가 멈추지 않고 계속 회전하는 현상 발생.
* **해결:** 사용된 모터가 '연속 회전 서보'임을 파악하고, 각도(`write`) 제어가 아닌 정지 펄스(`1508`)와 시간 지연(`delay`)을 이용한 제어 방식으로 코드를 수정하여 해결함.
* **문제:** hx117 회로가 고정되지 않고 자꾸 빠져 무게에 측정에 어려움이 생김
---

## 6. 결론

본 프로젝트를 통해 저렴한 비용의 오픈소스 하드웨어(Arduino)와 노코딩 툴(App Inventor)을 활용하여, 실제 사용 가능한 수준의 **'습관 형성 스마트 의자' 프로토타입**을 성공적으로 구현하였습니다. 특히 하드웨어(센서/모터)와 소프트웨어(앱/PC) 간의 유기적인 통신 구조를 설계하고 구현하는 기술을 습득하였습니다. 회로를 여태 제대로 설계하고 만져본 경험이 없어 어려움을 겪었고 아직은 완전히 해결하지 못하였지만 이번 보고서를 제출하였다고 끝나는게 아닌 계속해서 문제를 해결하기 위해 노력하겠습니

---
**[별첨]**
1.  Arduino 전체 소스 코드
2.  Processing 소스 코드
3.  MIT App Inventor 블록 이미지
