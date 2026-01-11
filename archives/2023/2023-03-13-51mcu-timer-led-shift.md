# 51 单片机 - 使用定时器来实现左右移动的流水灯程序

本程序代码为《手把手教你学 51 单片机》5.4 的课后练习题 3，并且已经在 KST-51 v1.3.2 开发板验证通过。

```c
#include <REGX52.H>

sbit ADDR0 = P1^0;
sbit ADDR1 = P1^1;
sbit ADDR2 = P1^2;
sbit ADDR3 = P1^3;
sbit ENLED = P1^4;

unsigned int cnt = 0;           // 记录进入 Timer0 中断次数
bit flagTF = 0;                 // 定义一个 bit 形的变量，为定时标志位
unsigned char shift = 0x01;     // 定义循环移位变量 shift，并赋初值 0x01，这里不能赋值为 0！
bit dir = 0;                    // 定义一个 bit 形的移位方向变量 dir，用于控制移位的方向

void main() {
    EA = 1;         // 使能总中断
    ENLED = 0;
    ADDR0 = 0;
    ADDR1 = 1;
    ADDR2 = 1;
    ADDR3 = 1;
    TMOD &= 0xF0;   // 设置 Timer0 为工作模式 1
    TMOD |= 0x01;   // 设置 Timer0 为工作模式 1
    TH0 = 0xFC;     // 为 Timer0 赋初值，定时 1ms
    TL0 = 0x67;
    ET0 = 1;        // 使能 Timer0 中断
    TR0 = 1;        // 启动 Timer0

    while (1) {
        P0 = ~shift;                    // P0 等于循环移位变量取反，控制 8 个 LED

        if (flagTF == 1) {
            flagTF = 0;
            if (dir == 0) {               // 移位方向变量为 0 时，左移
                shift = shift << 1;        // 循环移位变量左移 1 位
                if (shift == 0x80) {       // 左移到最左端后，改变移位方向
                    dir = 1;
                }
            } else {                      // 移位方向变量不为 0 时，右移
                shift = shift >> 1;        // 循环移位变量右移 1 位
                if (shift == 0x01) {       // 右移到最右端后，改变移位方向
                    dir = 0;
                }
            }
        }
    }
}

void interruptTimer0() interrupt 1 {
    TH0 = 0xFC;                         // 每进入中断，为 Timer0 赋初值，定时 1ms
    TL0 = 0x67;
    cnt++;                              // 每进入 1 次中断，中断次数加 1
    if (cnt >= 60) {                    // 当进入了 60 次中断，即计时了 60ms 时
        cnt = 0;                         // 记录进入中断次数清零
        flagTF = 1;                      // 定时标志位赋 1
    }
}
```
