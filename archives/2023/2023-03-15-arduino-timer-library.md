# Arduino的定时器中断 - 基于库函数

Arduino 已经内置了闹钟，它们叫做定时器，可以设定 Arduino 隔多长时间干一件其他事情。不过，中断时间不能太长，否则会影响正常的工作。

可以调用 millis() 的方式来实现定时，但在程序中断有一个问题：占用CPU资源、效率不高、而且不准确（millis() 会被外部中断打断）。而 Arduino 内置的三个硬件定时器：timer0、timer1、timer2，不会占用CPU资源，而且非常精确。

要调用硬件定时器并不困难，因为 Arduino 社区已经有不少库函数供使用（否则，就要非常复杂的编程）。常用的有三个：TimerOne、MsTimer2 和 FlexiTimer2，使用方法都很类似。

## TimerOne库演示

先看基于 TimerOne 库的演示程序：

```text
#include <TimerOne.h>

void setup(){
  pinMode(13, OUTPUT);
  Timer1.initialize(500000); // 初始化 Timer1，定时器每间隔 0.5s（500000us = 500ms = 0.5s）执行中断函数一次
  Timer1.pwm(9, 512); // 设置D9 PWM 占空比为50%
  Timer1.attachInterrupt(Flash); // 设定 callback 为 Timer 的中断函数
}

void Flash(){
  digitalWrite(13, digitalRead(13) ^ 1);// "^"异或符，如果为HIGH，输出 LOW，反之亦然
}

void loop(){
  //这家伙很轻松，啥都不用做
}
```

顾名思义 TimerOne 库函数调用的是 Timer1 定时器。

注意 Arduino 的 PWM 输出是依靠内置的 3 个 Timer 来控制的，所以 Timer1 会同时影响到 D9、D10 两个端口的 analogWrite() 方法，但可以通过调用 Timer1.pwm(pin, duty, period) 来设定，duty 是占空比（分辨率为 10bits，取值 0~1023），period 是可选参数，设定周期，如果不设定则为默认值，范围为 1 ~ 8388480us，即最大可产生 1MHz 方波。

## MsTimer2库演示

再看看 MsTimer2：

```text
#include <MsTimer2.h>

void Flash() {
  digitalWrite(13, digitalRead(13) ^ 1);
}

void setup() {
  pinMode(13, OUTPUT);
  MsTimer2::set(500, Flash); // 定时器间隔 0.5s （500ms = 0.5s）
  MsTimer2::start();//开始计时
}

void loop() {
  //这家伙很轻松，啥都不用做
}
```

程序实在很简单，但 TimerOne 和 MsTimer2 的区别非常大，前者调用的是 Timer1 是 16bit 定时器，分辨率可以达到 1us，而后者调用 Timer2 是 8bit 定时器分辨率只能达到 1ms。而且 TimerOne 的函数方法要比 MsTimer2 丰富，更多用法可参考**官方文档**。

## FlexiTimer2库演示

MsTimer2 目前已经更新为 FlexiTimer2，用法完全是一样的，但后者增加了"分辨率"参数：

```text
#include <FlexiTimer2.h>

void Flash() {
  digitalWrite(13, digitalRead(13) ^ 1);
}

void setup() {
  pinMode(13, OUTPUT);
  FlexiTimer2::set(500, 1.0 / 1000, Flash);
  FlexiTimer2::start();
}

void loop() {
  //这家伙很轻松，啥都不用做
}
```

分辨率是个 double 参数，如果将 1/1000 设置为 1/1280，即每秒会调用中断函数 Flash() 1280 次，也可以理解为 1/1280 秒，那么中断周期就是 500*1/1280。

## 霍尔传感器测速示例

现在有了硬件定时器，霍尔传感器测速程序就可以不用 millis() 来计时，提高效率和精确度：

```text
/*
  效果：霍尔传感器测试方法3
*/
#include <MsTimer2.h>

const byte interruptPin = 3;
float Val = 0; //设置变量Val，计数

void setup( ) {
  Serial.begin(9600);
  attachInterrupt(digitalPinToInterrupt(interruptPin), count, FALLING);//触发信号必须是变化的，上升或下降皆可
  MsTimer2::set(1000, Print);//每秒打印一次
  MsTimer2::start();
}

void loop( ) {
  //这家伙很轻松，啥都不用做
}

void count() {
  Val += 1;
}

void Print() {
  Serial.println(Val * 60);
  Val = 0;
}
```

这里关于中断只是入门，介绍了最基础的用法，已经满足现阶段应用层面的开发需求了。我们将继续深入探讨各种问题，比如中断的优先级别，利用定时器产生更频率的 PWM，中断能做的和不能做的事（提醒：中断程序内不能使用 I2C、SPI、串口等通信协议）等一系列问题。
