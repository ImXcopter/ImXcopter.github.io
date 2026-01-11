# Arduino的定时器中断

当你想让代码在一个固定的时间间隔后执行时，你可以很容易的用delay()函数来实现。但这只是让程序暂停一个特定的时间段。特别是如果你需要同时让处理器做其他处理时，这么做同时也是一种浪费。
这时候就是定时器（Timer）和中断（Interrupt）的用武之地了。
Arduino UNO有三个timer
timer0 - 一个被Arduino的 delay() ，millis() 和 micros()使用的8位定时器
timer1 - 一个被Arduino的 Servo()库使用的16位定时器（Servo 库在Arduino Mega上使用timer5）
timer2 - 一个被Arduino的 Tone()库使用的8位定时器
"Arduino Mega"板有另外三个可使用的timer3/4/5，这些定时器都是16位定时器。

---

PWM和定时器 定时器和PWM输出之间存在固定的关系。 当您查看数据手册或处理器的引脚分配时，这些具有PWM功能的引脚具有OCRxA，OCRxB或OCRxC等名称（其中x表示定时器编号0..5）。PWM功能通常与其他引脚功能共享。 Arduino有3个定时器和6个PWM输出引脚。定时器和PWM输出之间的关系是：
**引脚5和6：由timer0控制**
**引脚9和10：由timer1控制**
**引脚11和3：由timer2控制**
在Arduino Mega上我们有6个定时器和15个PWM输出：
**引脚4和13：由timer0控制**
**引脚11和12：由timer1控制**
**引脚9和10：由timer2控制**
**引脚2,3和5：由定时器3控制**
**引脚6,7和8：由定时器4控制**
**引脚46,45和44 ::由定时器5控制**


---

在以下的例子中，我们将在我们的中断使用timer1。显然，如果你用了Servo()库就会有冲突，所以你应该选择其他timer。
下面是一个基本的中断驱动程序。这是基本的LED闪光灯程序。但是现在我们用中断而不是delay()来每半秒开启和关闭LED灯一次，从而实现让LED每秒闪一次的效果。

```
/*
  timer1 中断实例
  LED闪光灯每秒闪亮一下
*/

#define ledPin 13     //设置输出口为13口
int timer1_counter;

void setup() {

  pinMode(ledPin, OUTPUT);

  //初始化定时器1
  noInterrupts(); //禁止所有中断
  TCCR1A = 0;
  TCCR1B = 0;

  //为我们的中断设置timer1_counter为正确的时间间隔
  // timer1_counter = 64911;    //预加载timer1为65536-16MHz/256/100Hz
  // timer1_counter = 64286;    //预加载timer1为65536-16MHz/256/50HZ
  timer1_counter = 34286;     //预加载timer1为65536-16MHz/256/2Hz
  TCNT1 = timer1_counter;     //预加载timer
  TCCR1B |= (1 << CS12);      //256 分频器(256 prescaler?)
  TIMSK1 |= (1 << TOIE1);     //启用定时器溢出中断
  interrupts();               //允许所有中断
}

ISR(TIMER1_OVF_vect) {        //常规中断服务
  TCNT1 = timer1_counter;     //预加载timer digitalWrite(ledPin, digitalRead(ledPin) ^ 1);
}

void loop() {
  //你自己的程序
}
```

**陷阱**
在编写Arduino并使用使用定时器的函数或库时，您可能会遇到一些陷阱：
伺服库使用Timer1。
在Arduino上使用servo库时，不能在引脚9,10上使用PWM。
对于Arduino Mega来说，所需的计时器取决于servo的数量。每个计时器可以处理12个servo。对于前12个servo，将使用定时器5（在引脚44, 45, 46上松开PWM）。
对于24 servo定时器，将使用定时器5和1（在引脚11, 12, 44, 45, 46上松开PWM）。对于36个servo定时器5，将使用1和3（在引脚2, 3, 5上丢失PWM， 11, 12, 44, 45, 46）..对于48个servo，将使用所有16​​位定时器5, 1, 3和4（丢失所有PWM引脚）。
引脚11具有共用功能PWM和MOSI。 SPI接口需要MOSI，Arduino上不能同时使用引脚11上的PWM和SPI接口。在Arduino Mega上，SPI引脚位于不同的引脚上。
tone()函数至少使用timer2。当您在Arduino Mega上使用Arduino和Pin 9,10的tone()函数时，不能在引脚3, 11上使用PWM。