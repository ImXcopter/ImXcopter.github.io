# Arduino 的定时器中断

当你想让代码在一个固定的时间间隔后执行时，你可以很容易的用 `delay()` 函数来实现。但这只是让程序暂停一个特定的时间段。特别是如果你需要同时让处理器做其他处理时，这么做同时也是一种浪费。

这时候就是定时器（Timer）和中断（Interrupt）的用武之地了。

## Arduino UNO 的定时器

Arduino UNO 有三个 timer：

- **timer0**：一个被 Arduino 的 `delay()`、`millis()` 和 `micros()` 使用的 8 位定时器
- **timer1**：一个被 Arduino 的 `Servo()` 库使用的 16 位定时器（Servo 库在 Arduino Mega 上使用 timer5）
- **timer2**：一个被 Arduino 的 `Tone()` 库使用的 8 位定时器

> "Arduino Mega" 板有另外三个可使用的 timer3/4/5，这些定时器都是 16 位定时器。

## PWM 和定时器

定时器和 PWM 输出之间存在固定的关系。当您查看数据手册或处理器的引脚分配时，这些具有 PWM 功能的引脚具有 `OCRxA`、`OCRxB` 或 `OCRxC` 等名称（其中 x 表示定时器编号 0..5）。PWM 功能通常与其他引脚功能共享。

Arduino 有 3 个定时器和 6 个 PWM 输出引脚。定时器和 PWM 输出之间的关系是：

| 引脚 | 控制定时器 |
|------|-----------|
| 5 和 6 | timer0 |
| 9 和 10 | timer1 |
| 11 和 3 | timer2 |

在 Arduino Mega 上我们有 6 个定时器和 15 个 PWM 输出：

| 引脚 | 控制定时器 |
|------|-----------|
| 4 和 13 | timer0 |
| 11 和 12 | timer1 |
| 9 和 10 | timer2 |
| 2, 3 和 5 | 定时器 3 |
| 6, 7 和 8 | 定时器 4 |
| 46, 45 和 44 | 定时器 5 |

## 中断使用示例

在以下的例子中，我们将在我们的中断使用 timer1。显然，如果你用了 `Servo()` 库就会有冲突，所以你应该选择其他 timer。

下面是一个基本的中断驱动程序。这是基本的 LED 闪光灯程序。但是现在我们用中断而不是 `delay()` 来每半秒开启和关闭 LED 灯一次，从而实现让 LED 每秒闪一次的效果。

```cpp
/*
  timer1 中断实例
  LED 闪光灯每秒闪亮一下
*/

#define ledPin 13     // 设置输出口为 13 口
int timer1_counter;

void setup() {
    pinMode(ledPin, OUTPUT);

    // 初始化定时器 1
    noInterrupts(); // 禁止所有中断
    TCCR1A = 0;
    TCCR1B = 0;

    // 为我们的中断设置 timer1_counter 为正确的时间间隔
    // timer1_counter = 64911;    // 预加载 timer1 为 65536-16MHz/256/100Hz
    // timer1_counter = 64286;    // 预加载 timer1 为 65536-16MHz/256/50Hz
    timer1_counter = 34286;     // 预加载 timer1 为 65536-16MHz/256/2Hz
    TCNT1 = timer1_counter;     // 预加载 timer
    TCCR1B |= (1 << CS12);      // 256 分频器
    TIMSK1 |= (1 << TOIE1);     // 启用定时器溢出中断
    interrupts();               // 允许所有中断
}

ISR(TIMER1_OVF_vect) {         // 常规中断服务
    TCNT1 = timer1_counter;     // 预加载 timer
    digitalWrite(ledPin, digitalRead(ledPin) ^ 1);
}

void loop() {
    // 你自己的程序
}
```

## 注意事项

在编写 Arduino 并使用使用定时器的函数或库时，您可能会遇到一些陷阱：

- **伺服库使用 Timer1**：在 Arduino 上使用 servo 库时，不能在引脚 9,10 上使用 PWM
- **Arduino Mega 的定时器**：所需的计时器取决于 servo 的数量。每个计时器可以处理 12 个 servo
- **引脚 11 的共用功能**：PWM 和 MOSI 共用引脚，SPI 接口需要 MOSI，不能同时使用引脚 11 上的 PWM 和 SPI 接口
- **tone() 函数**：至少使用 timer2。当您在 Arduino Mega 上使用 tone() 函数时，不能在引脚 3, 11 上使用 PWM
