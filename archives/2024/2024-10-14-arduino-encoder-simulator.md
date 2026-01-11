# Arduino模拟发送编码器

Arduino模拟编码器代码
本代码在Arduino Mini PRO下编译通过！

```cpp
// 定义引脚
#define AP_pin 10
#define AN_pin 9
#define BP_pin 8
#define BN_pin 7
#define IP_pin 6
#define IN_pin 5
#define KeyA_pin 2
#define KeyB_pin 3

// 定义相位延迟
#define PHASE_DELAY 5 // 微秒

void setup() {
  // 设置输出引脚
  pinMode(AP_pin, OUTPUT);
  pinMode(AN_pin, OUTPUT);
  pinMode(BP_pin, OUTPUT);
  pinMode(BN_pin, OUTPUT);
  pinMode(IP_pin, OUTPUT);
  pinMode(IN_pin, OUTPUT);

  // 设置输入引脚
  pinMode(KeyA_pin, INPUT_PULLUP);
  pinMode(KeyB_pin, INPUT_PULLUP);
}

void loop() {
  bool keyAState = digitalRead(KeyA_pin) == LOW;
  bool keyBState = digitalRead(KeyB_pin) == LOW;

  if (keyAState && !keyBState) {
    rotateClockwise();
  } else if (keyBState && !keyAState) {
    rotateCounterClockwise();
  } else {
    stopMotor();
  }
}

void rotateClockwise() {
  // 模拟A相正向
  setPhase(HIGH, LOW, LOW, HIGH);
  delayMicroseconds(PHASE_DELAY);

  // 模拟B相正向
  setPhase(HIGH, LOW, HIGH, LOW);
  delayMicroseconds(PHASE_DELAY);

  // 模拟A相反向
  setPhase(LOW, HIGH, HIGH, LOW);
  delayMicroseconds(PHASE_DELAY);

  // 模拟B相反向
  setPhase(LOW, HIGH, LOW, HIGH);
  delayMicroseconds(PHASE_DELAY);
}

void rotateCounterClockwise() {
  // 模拟B相反向
  setPhase(LOW, HIGH, LOW, HIGH);
  delayMicroseconds(PHASE_DELAY);

  // 模拟A相反向
  setPhase(LOW, HIGH, HIGH, LOW);
  delayMicroseconds(PHASE_DELAY);

  // 模拟B相正向
  setPhase(HIGH, LOW, HIGH, LOW);
  delayMicroseconds(PHASE_DELAY);

  // 模拟A相正向
  setPhase(HIGH, LOW, LOW, HIGH);
  delayMicroseconds(PHASE_DELAY);
}

void stopMotor() {
  // 停止电机，所有相都设为低电平
  setPhase(LOW, LOW, LOW, LOW);
}

void setPhase(int ap, int an, int bp, int bn) {
  digitalWrite(AP_pin, ap);
  digitalWrite(AN_pin, an);
  digitalWrite(BP_pin, bp);
  digitalWrite(BN_pin, bn);
  digitalWrite(IP_pin, HIGH);
  digitalWrite(IN_pin, LOW);
}
```
