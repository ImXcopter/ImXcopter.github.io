# 51 单片机 - 秒表计时代码（只显示有效位）

本程序代码为《手把手教你学 51 单片机》6.6 的课后练习题 4，并且已经在 KST-51 v1.3.2 开发板验证通过。

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

// 数码管显示缓冲区，初值 0xFF 确保启动时都不亮
unsigned char LedBuff[6] = {
    0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF
};

unsigned char flag1s = 0;   // 1 秒定时标志
unsigned int cnt = 0;       // 记录 T0 中断次数
unsigned char i = 0;        // 动态扫描的索引

void main() {
    unsigned long sec = 0;
    unsigned char buff[6];  // 中间转换缓冲区
    char j = 0;

    EA = 1;                 // 使能总中断
    ADDR3 = 1;              // 因为需要动态改变 ADDR0-2 的值，所以 ADDR0-2 不需要再初始化了
    ENLED = 0;              // 使能 U3
    TMOD &= 0xF0;           // 设置 Timer0 工作模式 1
    TMOD |= 0x01;
    TH0 = 0xFC;             // 为 Timer0 赋初值 0xFC67，定时 1ms
    TL0 = 0x67;
    ET0 = 1;                // 使能 Timer0 中断
    TR0 = 1;                // 启动 Timer0

    while (1) {
        if (flag1s) {         // 判断 1 秒定时标志
            flag1s = 0;       // 1 秒定时标志清零
            sec++;            // 秒计数自加 1
            // 将 sec 按十进制位从低到高依次提取到 buff 数组中
            buff[0] = sec % 10;
            buff[1] = sec / 10 % 10;
            buff[2] = sec / 100 % 10;
            buff[3] = sec / 1000 % 10;
            buff[4] = sec / 10000 % 10;
            buff[5] = sec / 100000 % 10;
            // 从最高位开始，遇到 0 不显示（赋值 0xFF），遇到非 0 退出循环
            for (j = 5; j >= 1; j--) {
                if (buff[j] == 0) {
                    LedBuff[j] = 0xFF;
                } else {
                    break;
                }
            }
            // 将剩余的有效数字位如实转换，for() 起始未对 j 操作，j 即保持上个循环结束时的值
            for (; j >= 0; j--) {
                LedBuff[j] = LedChar[buff[j]];
            }
        }
    }
}

/* 定时器 0 中断服务函数 */
void interruptTimer0() interrupt 1 {
    TH0 = 0xFC;         // 重新加载初值
    TL0 = 0x67;
    cnt++;              // 中断次数计数值加 1
    if (cnt >= 1000) {  // 中断 1000 次即 1 秒
        cnt = 0;        // 清零计数值以重新开始下 1 秒计时
        flag1s = 1;     // 设置 1 秒定时标志为 1
    }
    // 以下代码完成数码管动态扫描刷新
    P0 = 0xFF;          // 显示消隐
    switch (i) {
        case 0: ADDR2 = 0; ADDR1 = 0; ADDR0 = 0; i++; P0 = LedBuff[0]; break;
        case 1: ADDR2 = 0; ADDR1 = 0; ADDR0 = 1; i++; P0 = LedBuff[1]; break;
        case 2: ADDR2 = 0; ADDR1 = 1; ADDR0 = 0; i++; P0 = LedBuff[2]; break;
        case 3: ADDR2 = 0; ADDR1 = 1; ADDR0 = 1; i++; P0 = LedBuff[3]; break;
        case 4: ADDR2 = 1; ADDR1 = 0; ADDR0 = 0; i++; P0 = LedBuff[4]; break;
        case 5: ADDR2 = 1; ADDR1 = 0; ADDR0 = 1; i = 0; P0 = LedBuff[5]; break;
        default: break;
    }
}
```
