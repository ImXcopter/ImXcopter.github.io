# 51 单片机 - 实现数码管静态显示秒表倒计时代码

本程序代码为《手把手教你学 51 单片机》5.4 的课后练习题 5，并且已经在 KST-51 v1.3.2 开发板验证通过。

```c
#include <REGX52.H>

sbit ADDR0 = P1^0;
sbit ADDR1 = P1^1;
sbit ADDR2 = P1^2;
sbit ADDR3 = P1^3;
sbit ENLED = P1^4;

// 数码管真值表
unsigned char code LedChar[16] = {
    0xC0, 0xF9, 0xA4, 0xB0, 0x99, 0x92, 0x82, 0xF8,
    0x80, 0x90, 0x88, 0x83, 0xC6, 0xA1, 0x86, 0x8E
};

unsigned int cnt = 0;       // 记录进入 Timer0 中断次数
unsigned char sec = 15;     // 记录经过的秒数，倒计时初值 15，也可以写成十六进制的形式（0x0F）
bit flag1s = 0;             // 定义一个 bit 形的变量，作为 1 秒的标志位

void main() {
    EA = 1;                 // 使能总中断
    ENLED = 0;
    ADDR3 = 1;
    ADDR2 = 0;
    ADDR1 = 0;
    ADDR0 = 0;
    TMOD &= 0xF0;           // 设置 Timer0 为工作模式 1
    TMOD |= 0x01;
    TH0 = 0xDC;             // 为 Timer0 赋初值，定时 10ms
    TL0 = 0x00;
    ET0 = 1;                // 使能 Timer0 中断
    TR0 = 1;                // 启动 Timer0

    while (1) {
        P0 = LedChar[sec];  // 当前秒数对应的真值表中的值送到 P0 口

        if (flag1s) {
            flag1s = 0;
            if (sec > 0) {
                sec--;      // 当秒数大于 0 时，减 1
            } else {
                sec = 15;   // 当秒数等于 0 时，重新赋初值 0x0F（十进制的 15）
            }
        }
    }
}

void interruptTimer0() interrupt 1 {
    TH0 = 0xDC;             // 进入 Timer0 赋初值
    TL0 = 0x00;
    cnt++;                  // 每进入 1 次中断，中断次数加 1
    if (cnt >= 100) {       // 当进入了 100 次中断，即计时了 10ms * 100 = 1s 时
        cnt = 0;            // 记录进入中断次数清零
        flag1s = 1;         // 定时 1s 标志位赋 1
    }
}
```
