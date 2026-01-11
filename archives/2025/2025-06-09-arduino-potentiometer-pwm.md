# 电位器转换PWM输出Arduino代码

电位器转换PWM输出：
-----------

```
const int potPin = A0; // 电位器输入
const int pwmPin = 3; // 更换为支持 PWM 的 D3 引脚
const int ledPin = 13; // 板载LED

void setup() {
pinMode(pwmPin, OUTPUT); // PWM 输出
pinMode(ledPin, OUTPUT); // 板载LED
}

void loop() {
int potValue = analogRead(potPin); // 读取电位器值 (0~1023)
int pwmValue = map(potValue, 0, 1023, 0, 255); // 转换为 PWM 范围 (0~255)

analogWrite(pwmPin, pwmValue); // 输出 PWM 到 D3
analogWrite(ledPin, pwmValue); // 板载 LED 明暗同步变化
}
```