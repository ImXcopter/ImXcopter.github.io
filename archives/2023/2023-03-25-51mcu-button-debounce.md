# 51 单片机 - 独立按键消抖 Coding 练习

本程序代码为《手把手教你学 51 单片机》8.6 课后练习题 4，并且已经在 KST-51 v1.3.2 开发板验证通过。

```c
#include <REGX52.H>

sbit ADDR0 = P1^0;
sbit ADDR1 = P1^1;
sbit ADDR2 = P1^2;
sbit ADDR3 = P1^3;
sbit ENLED = P1^4;
sbit keyIn1 = P2^4;
sbit keyIn2 = P2^5;
sbit keyIn3 = P2^6;
sbit keyIn4 = P2^7;
sbit keyOut1 = P2^3;
sbit keyOut2 = P2^2;
sbit keyOut3 = P2^1;
sbit keyOut4 = P2^0;

// 数码管真值表
unsigned char code LedChar[16] = {
    0xC0, 0xF9, 0xA4, 0xB0, 0x99, 0x92, 0x82, 0xF8,
    0x80, 0x90, 0x88, 0x83, 0xC6, 0xA1, 0x86, 0x8E
};

// 当前按键的状态
bit keySta = 1;

void main()
{
    bit keyBackup = 0;              // 备份按键状态，保存前一次的按键状态
    unsigned char cnt = 15;         // 按键计数，记录按键按下的次数

    EA = 1;                         // 开启总中断
    ENLED = 0;                      // 使能数码管的 U3
    ADDR3 = 1;
    ADDR2 = 0;                      // 选择数码管 DS1 进行显示
    ADDR1 = 0;
    ADDR0 = 0;
    TMOD &= 0xF0;                   // 设置 Timer0 为模式 1
    TMOD |= 0x01;
    TH0 = 0xF8;                     // 为 Timer0 赋初值 0xF8CD，定时 2ms
    TL0 = 0xCD;
    ET0 = 1;                        // 使能 Timer0 中断
    TR0 = 1;                        // 启动 Timer0
    P2 = 0xF7;                      // P2^3 置 0，即 keyOut1 输出低电平
    P0 = LedChar[cnt];              // 显示按键次数的初始值 15

    while (1)
    {
        if (keySta != keyBackup)    // 如果 Key 当前值不得高于 Key 的备份值，说明按键有变化了
        {
            if(keySta == 0)         // 如果 Key 当前值为 0，说明当前按键已经按下状态
            {
                if (cnt == 0)
                {
                    cnt = 15;
                }
                else
                {
                    cnt--;
                }
                P0 = LedChar[cnt];  // 计数值显示到数码管上
            }
            keyBackup = keySta;     // 更新备份为当前值，以备下次比较使用
        }
    }
}

void InterruptTimer0() interrupt 1
{
    static unsigned char keyBuff = 0xFF;   // 扫描缓冲区，保存一段时间内的扫描值，所以用 static 来修饰变量
    TH0 = 0xF8;                            // 重载定时器初值
    TL0 = 0xCD;

    // 每次进入中断左移 1 位，并且和 keyIn1 或运算。
    // 或运算的作用是：左移 1 位后面补位为 0，keyIn1 是 0 的话或运算后还是 0，keyIn1 是 1 的话或运算后则为 1
    keyBuff = (keyBuff << 1) | keyIn1;

    if (keyBuff == 0x00)                  // 如果连续 8 次都是 0，则将 keySta 赋为 0，说明按键为稳定的按下去的状态
    {
        keySta = 0;
    }
    if (keyBuff == 0xFF)                  // 如果连续 8 次都是 1，则将 keySta 赋为 1，说明按键为稳定的弹起状态
    {
        keySta = 1;
    }
}
```
