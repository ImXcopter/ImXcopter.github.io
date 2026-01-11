# 51 单片机 - LED 灯显示二进制代码进阶版

之前写过一个 [51 单片机 - LED 灯显示二进制代码](https://xcopter.cc/archives/2023/2023-03-14-51mcu-led-binary-display.html/)，为了锻炼一下综合能力，这里又加了一些花里胡哨的效果进去。

这里提一下 coding 过程遇到的坑：在只有 0 和 1 两个状态的时候，或者说想使用取反运算符得到 0 或者 1 的时候，尽量选择 bit 形变量。如果使用了非 bit 形的变量，例如使用了 unsigned char 形的变量，需要手动置 1 或者手动置 0，就无法使用 ~ 取反运算得到 0 或者 1 的结果了。

代码已经在 KST-51 v1.3.2 开发板验证通过。

![运行效果](/static/2023/2023-03-15-51mcu-led-binary-advanced_001.png)

> 效果视频：https://xcopter.cc/wp-content/uploads/2023/03/20230315-275.mp4

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

// 数码管顺时针旋转真值表
unsigned char code LedCW[6] = {
    0xF7, 0xEF, 0xDF, 0xFE, 0xFD, 0xFB
};

// 数码管逆时针旋转真值表
unsigned char code LedCCW[6] = {
    0xFB, 0xFD, 0xFE, 0xDF, 0xEF, 0xF7
};

// 前六个为数码管显示缓冲区，最后 1 个为 8 个 LED 小灯的初始值，初值 0xFF 确保启动时都不亮
unsigned char LedBuff[7] = {
    0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF
};

bit flag1s = 0;                         // 1 秒定时标志
bit flag100ms = 0;                      // 100ms 定时标志

void main() {
    unsigned char sec = 0;              // 计秒初始值
    unsigned char tempLedBuff = 0;      // 8 个 LED 小灯转换变量
    unsigned char j = 0;
    unsigned buff[3];                   // 中间转换缓冲区
    signed char k = 0;
    bit flagLedDP = 0;                  // 数码管小点标志位

    EA = 1;                             // 使能总中断
    ENLED = 0;                          // 使能 U3
    ADDR3 = 1;                          // 因为需要动态改变 ADDR0-2 的值，所以 ADDR0-2 不需要再初始化了
    TMOD &= 0x0F;                       // 设置 Timer1 工作模式 1
    TMOD |= 0x10;
    TH1 = 0xFC;                         // 为 Timer1 赋初值 0xFC67，定时 1ms
    TL1 = 0x67;
    ET1 = 1;                            // 使能 Timer1 中断
    TR1 = 1;                            // 启动 Timer1

    while (1) {
        if (flag1s) {                   // 判断 1 秒定时标志
            flag1s = 0;                 // 1 秒定时标志清零
            flagLedDP = ~flagLedDP;     // 1 秒数码管小点标志位取反
            if (sec < 255) {            // 如果秒数小于 255
                sec++;                  // 秒计数自加 1
                tempLedBuff++;          // LED 小灯当前值自加 1
            } else {
                sec = 0;                // 秒数清零
                tempLedBuff = 0;        // LED 小灯值清零
            }
        }

        if (flag100ms) {                // 100ms 定时位
            flag100ms = 0;              // 100ms 定时位清零
            if (j >= 5) {               // 循环数码管旋转真值表的数组下标
                j = 0;
            } else {
                j++;
            }
        }
        // 将 sec 按十进制位从低到高依次提取到 buff 数组中，由于 sec 最大只有 255，所以只计算三位
        buff[0] = sec % 10;
        buff[1] = sec / 10 % 10;
        buff[2] = sec / 100 % 10;
        // 从最高位开始，遇到 0 不显示（赋值 0xFF），遇到非 0 退出 for 循环
        for (k = 2; k >= 1; k--) {
            if (buff[k] == 0) {
                LedBuff[k] = 0xFF;
            } else {
                break;
            }
        }
        // 将剩余的有效数字位如实转换，for() 起始未对 j 操作，j 即保持上个循环结束时的值
        for (; k >= 0; k--) {
            LedBuff[k] = LedChar[buff[k]];
        }
        // LED 小灯当前值取反，赋给 LedBuff[6]，待每 1ms 进定时器中断刷新显示出来。取反的原因在于每个 LED 小灯低电平点亮。
        LedBuff[6] = ~tempLedBuff;
        // 如果 flagLedDP 置 1 了，说明 1 秒的时间到，在当前 LedCW 的元素上面与运算 0x7F，点亮数码管小点
        // 下一秒 flagLedDP 标志位置 0，则直接显示当前旋转数码管的段位
        if (flagLedDP) {
            LedBuff[3] = LedCW[j] & 0x7F;
            LedBuff[4] = LedCW[j] & 0x7F;
            LedBuff[5] = LedCW[j] & 0x7F;
        } else {
            LedBuff[3] = LedCW[j];
            LedBuff[4] = LedCW[j];
            LedBuff[5] = LedCW[j];
        }
    }
}

/* 定时器 1 中断服务函数 */
void interruptTimer1() interrupt 3 {
    static unsigned char i = 0;
    static unsigned int cnt = 0;
    static unsigned int ledcnt = 0;

    TH1 = 0xFC;             // 重新加载初值
    TL1 = 0x67;
    cnt++;                  // 中断次数计数值加 1
    if (cnt >= 1000) {      // 中断 1000 次即 1 秒
        cnt = 0;            // 清零计数值以重新开始下 1 秒计时
        flag1s = 1;         // 设置 1 秒定时标志为 1
    }
    ledcnt++;               // 中断次数计数值加 1
    if (ledcnt >= 100) {    // 中断 100 次即 100ms
        ledcnt = 0;         // 清零计数值以重新开始下 1 秒计时
        flag100ms = 1;      // 设置 100ms 定时标志为 1
    }
    // 以下代码完成数码管和 LED 小灯的动态扫描刷新
    // 这里每 1ms 进入中断 1 次，每次刷新 1 个段位，循环完 7 个段位，共需要 7ms
    P0 = 0xFF;              // 显示消隐
    switch (i) {
        case 0: ADDR2 = 0; ADDR1 = 0; ADDR0 = 0; i++; P0 = LedBuff[0]; break;
        case 1: ADDR2 = 0; ADDR1 = 0; ADDR0 = 1; i++; P0 = LedBuff[1]; break;
        case 2: ADDR2 = 0; ADDR1 = 1; ADDR0 = 0; i++; P0 = LedBuff[2]; break;
        case 3: ADDR2 = 0; ADDR1 = 1; ADDR0 = 1; i++; P0 = LedBuff[3]; break;
        case 4: ADDR2 = 1; ADDR1 = 0; ADDR0 = 0; i++; P0 = LedBuff[4]; break;
        case 5: ADDR2 = 1; ADDR1 = 0; ADDR0 = 1; i = 0; P0 = LedBuff[5]; break;
        case 6: ADDR2 = 1; ADDR1 = 1; ADDR0 = 0; i = 0; P0 = LedBuff[6]; break;
        default: break;
    }
}
```
