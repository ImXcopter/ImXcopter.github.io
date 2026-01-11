# 51 单片机 - 同时点亮 KST-51 开发板的 8x8 点阵、LED 数码管和 LED 小灯

**效果展示：**

> [点击播放效果视频](https://attachment.xcopter.cc/2023/2023-03-19-51mcu-matrix-segment-led-contro.mp4)

本程序代码为《手把手教你学 51 单片机》7.6 的课后练习题 6，并且已经在 KST-51 v1.3.2 开发板验证通过。

```c
#include <REGX52.H>

sbit ADDR0 = P1^0;
sbit ADDR1 = P1^1;
sbit ADDR2 = P1^2;
sbit ADDR3 = P1^3;
sbit ENLED = P1^4;

// 点阵数码管图片帧
unsigned char code images[6][8] = {
    {0x00, 0x7E, 0x7E, 0x7E, 0x7E, 0x7E, 0x7E, 0x00},  // 8x8 点阵从外圈数第 1 圈显示的图片帧
    {0xFF, 0x81, 0xBD, 0xBD, 0xBD, 0xBD, 0x81, 0xFF},  // 8x8 点阵从外圈数第 2 圈显示的图片帧
    {0xFF, 0xFF, 0xC3, 0xDB, 0xDB, 0xC3, 0xFF, 0xFF},  // 8x8 点阵从外圈数第 3 圈显示的图片帧
    {0xFF, 0xFF, 0xFF, 0xE7, 0xE7, 0xFF, 0xFF, 0xFF},  // 8x8 点阵从外圈数第 4 圈显示的图片帧
    {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF},  // 8x8 点阵全部不显示的图片帧
    {0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00}   // 8x8 点阵全部显示的图片帧
};

// 数码管数值真值表
unsigned char code LedChar[16] = {
    0xC0, 0xF9, 0xA4, 0xB0, 0x99, 0x92, 0x82, 0xF8,
    0x80, 0x90, 0x88, 0x83, 0xC6, 0xA1, 0x86, 0x8E
};

// 数码管顺时针旋转真值表
unsigned char code LedCW[6] = {
    0xF7, 0xEF, 0xDF, 0xFE, 0xFD, 0xFB
};

// 数码管显示缓冲区
unsigned char LedBuff[7] = {
    0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF
};

bit flag1s = 0;                             // 1s 标志位
bit flag50ms = 0;                           // 50ms 标志位
bit flag100ms = 0;                          // 100ms 标志位
bit flag3s = 0;                             // 3s 标志位
unsigned char index = 0;                    // 图片帧索引变量
bit flagcyclecnt = 0;                       // 图片循环次数标志位

void main() {
    unsigned char sec = 0;                  // 计秒初始值
    unsigned char tempLedBuff = 0;          // 8 个 LED 小灯转换变量
    unsigned char buff[3];                  // 中间转换缓冲区
    signed char i = 0;
    unsigned char j = 0;
    bit flagLedDP = 0;                      // 数码管小点标志位
    bit dir = 1;                            // 图片帧播放方向变量
    unsigned char cyclecnt = 0;             // 图片帧播放次数变量

    EA = 1;                                 // 使能总中断
    ENLED = 0;                              // 使能 LED
    TMOD &= 0xF0;                           // 设置 Timer0 工作模式 1
    TMOD |= 0x01;
    TH0 = 0xFC;                             // 为 Timer0 赋初值 0xFC67，定时 1ms
    TL0 = 0x67;
    ET0 = 1;                                // 使能 Timer0 中断
    TR0 = 1;                                // 启动 Timer0

    while (1) {
        if (flag1s) {                       // 1s 标志位
            flag1s = 0;                     // 1s 标志位清零
            flagLedDP = ~flagLedDP;         // 1 秒数码管小点标志位取反
            if (sec < 255) {                // 如果秒数小于 255
                sec++;                      // 秒计数自加 1
                tempLedBuff++;              // LED 小灯当前值自加 1
            } else {
                sec = 0;                    // 秒数清零
                tempLedBuff = 0;            // LED 小灯值清零
            }
        }

        if (flag100ms) {                    // 100ms 定时位
            flag100ms = 0;                  // 100ms 定时位清零
            if (j >= 5) {                   // 循环数码管旋转真值表的数组下标
                j = 0;
            } else {
                j++;
            }
        }

        if (flag50ms) {                     // 50ms 定时位
            flag50ms = 0;                   // 50ms 定时位清零
            if (dir) {                      // 如果 dir = 1
                index++;                    // 图片帧播放索引每次加 1，图片帧正向播放
                if (index >= 4) {           // 图片帧播放到第 4 帧
                    dir = 0;                // dir 置 0
                }
            } else {                        // 如果 dir = 0，图片帧索引每次减 1，图片反向播放
                index--;
                if (index <= 0) {           // 图片索引播放到第 0 帧
                    dir = 1;                // dir 置 1
                    cyclecnt++;             // 循环播放变量加 1
                }
            }
            if (cyclecnt >= 5) {            // 如果图片帧循环播放了 5 次
                index = 5;                  // 图片播放第 5 帧，即 LED 点阵全部点亮
                flagcyclecnt = 1;           // 图片循环次数标志位置 1
                if (flag3s) {               // 3s 定时位
                    flag3s = 0;             // 3s 定时位清零
                    index = 0;              // 图片帧索引清零
                    cyclecnt = 0;           // 图片帧播放次数清零
                    flagcyclecnt = 0;       // 图片循环次数标志位置清零
                    dir = 1;                // dir 置 1
                }
            }
        }
        // 将 sec 按十进制位从低到高依次提取到 buff 数组中，由于 sec 最大只有 255，所以只计算三位
        buff[0] = sec % 10;
        buff[1] = sec / 10 % 10;
        buff[2] = sec / 100 % 10;
        // 从最高位开始，遇到 0 不显示（赋值 0xFF），遇到非 0 退出 for 循环
        for (i = 2; i >= 1; i--) {
            if (buff[i] == 0) {
                LedBuff[i] = 0xFF;
            } else {
                break;
            }
        }
        // 将剩余的有效数字位如实转换
        for (; i >= 0; i--) {
            LedBuff[i] = LedChar[buff[i]];
        }
        // 如果 flagLedDP 置 1 了，说明 1 秒的时间到，在当前 LedCW 的元素上面与运算 0x7F，点亮数码管小点
        if (flagLedDP) {
            LedBuff[3] = LedCW[j] & 0x7F;
            LedBuff[4] = LedCW[j] & 0x7F;
            LedBuff[5] = LedCW[j] & 0x7F;
        } else {
            LedBuff[3] = LedCW[j];
            LedBuff[4] = LedCW[j];
            LedBuff[5] = LedCW[j];
        }
        // LED 小灯当前值取反，赋给 LedBuff[6]
        LedBuff[6] = ~tempLedBuff;
    }
}

/* 定时器 1 中断服务函数 */
void interruptTimer0() interrupt 1 {
    static unsigned int cnt_1 = 0;
    static unsigned int cnt_2 = 0;
    static unsigned int cnt_3 = 0;
    static unsigned int cnt_4 = 0;
    static unsigned char i = 0;

    TH0 = 0xFC;                 // 重新加载初值
    TL0 = 0x67;

    cnt_1++;
    if (cnt_1 >= 1000) {
        cnt_1 = 0;
        flag1s = 1;
    }
    cnt_2++;
    if (cnt_2 >= 100) {
        cnt_2 = 0;
        flag100ms = 1;
    }
    cnt_3++;
    if (cnt_3 >= 50) {
        cnt_3 = 0;
        flag50ms = 1;
    }
    if (flagcyclecnt) {         // 图片循环次数标志位置 1
        cnt_4++;                // 进入中断次数 cnt_4 开始自加 1
        if (cnt_4 >= 3000) {    // 进入中断 3000 次，即 3s
            cnt_4 = 0;          // 中断次数 cnt_4 清零
            flag3s = 1;         // 3s 标志位置 1
        }
    }
    // 以下代码完成 8x8 点阵，数码管和 LED 小灯的动态扫描刷新
    P0 = 0xFF;                  // 显示消隐
    switch (i) {
        case 0: ADDR3 = 1; ADDR2 = 0; ADDR1 = 0; ADDR0 = 0; i++; P0 = LedBuff[0]; break;
        case 1: ADDR3 = 1; ADDR2 = 0; ADDR1 = 0; ADDR0 = 1; i++; P0 = LedBuff[1]; break;
        case 2: ADDR3 = 1; ADDR2 = 0; ADDR1 = 1; ADDR0 = 0; i++; P0 = LedBuff[2]; break;
        case 3: ADDR3 = 1; ADDR2 = 0; ADDR1 = 1; ADDR0 = 1; i++; P0 = LedBuff[3]; break;
        case 4: ADDR3 = 1; ADDR2 = 1; ADDR1 = 0; ADDR0 = 0; i++; P0 = LedBuff[4]; break;
        case 5: ADDR3 = 1; ADDR2 = 1; ADDR1 = 0; ADDR0 = 1; i++; P0 = LedBuff[5]; break;
        case 6: ADDR3 = 1; ADDR2 = 1; ADDR1 = 1; ADDR0 = 0; i++; P0 = LedBuff[6]; break;
        case 7: ADDR3 = 0; ADDR2 = 0; ADDR1 = 0; ADDR0 = 0; i++; P0 = images[index][0]; break;
        case 8: ADDR3 = 0; ADDR2 = 0; ADDR1 = 0; ADDR0 = 1; i++; P0 = images[index][1]; break;
        case 9: ADDR3 = 0; ADDR2 = 0; ADDR1 = 1; ADDR0 = 0; i++; P0 = images[index][2]; break;
        case 10: ADDR3 = 0; ADDR2 = 0; ADDR1 = 1; ADDR0 = 1; i++; P0 = images[index][3]; break;
        case 11: ADDR3 = 0; ADDR2 = 1; ADDR1 = 0; ADDR0 = 0; i++; P0 = images[index][4]; break;
        case 12: ADDR3 = 0; ADDR2 = 1; ADDR1 = 0; ADDR0 = 1; i++; P0 = images[index][5]; break;
        case 13: ADDR3 = 0; ADDR2 = 1; ADDR1 = 1; ADDR0 = 0; i++; P0 = images[index][6]; break;
        case 14: ADDR3 = 0; ADDR2 = 1; ADDR1 = 1; ADDR0 = 1; i = 0; P0 = images[index][7]; break;
        default: break;
    }
}
```
