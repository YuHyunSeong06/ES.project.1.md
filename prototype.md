## Arduino IDE
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

// ★ 캘리브레이션에서 찾은 값으로 수정 필수!
float calibration_factor = 0.0; 
long weight_threshold = 20; // 20kg 기준

void setup() {
  Serial.begin(9600); // PC(Processing) 통신
  BT.begin(9600);     // 앱(App Inventor) 통신

  // 로드셀 초기화
  scale.begin(LOADCELL_DOUT_PIN, LOADCELL_SCK_PIN);
  scale.set_scale(calibration_factor);
  scale.tare(); 

  // 서보모터 초기화 (Parallax Standard Servo)
  myServo.attach(SERVO_PIN);
  unlockChair(); // 초기 상태: 0도 (해제)
}

void loop() {
  // 1. 무게 측정
  float weight = scale.get_units(5);

  // 2. 데이터 전송 (1: 착석, 0: 공석)
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

void lockChair() {
  // 90도로 회전하여 잠금
  myServo.writeMicroseconds(1622);
  myServo.writeMicroseconds(1508);
}

void unlockChair() {
  // 0도로 원상복구
  myServo.writeMicroseconds(1508); 
}
```

## Processing
```java
import processing.serial.*;

Serial myPort; 
String val; 

void setup() {
  size(400, 400);
  // 포트 번호 확인 후 [0]을 [1] 등으로 수정해야 할 수 있음
  printArray(Serial.list());
  String portName = Serial.list()[0]; 
  myPort = new Serial(this, portName, 9600);
  textAlign(CENTER, CENTER);
  textSize(32);
}

void draw() {
  if (myPort.available() > 0) {
    val = myPort.readStringUntil('\n'); 
  }
  
  if (val != null) {
    val = trim(val);
    
    if (val.equals("1")) {
      background(255); // 하얀색 (착석)
      fill(0);         // 글씨 검은색
      text("SEATED ( > 20kg )", width/2, height/2);
    } else {
      background(0);   // 검은색 (공석)
      fill(255);       // 글씨 하얀색
      text("EMPTY", width/2, height/2);
    }
  }
}
```
