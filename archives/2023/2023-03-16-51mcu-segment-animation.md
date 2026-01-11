# 51 单片机 - 数码管图片帧刷新 coding 训练

**效果展示：**

> [点击播放效果视频](https://attachment.xcopter.cc/2023/2023-03-16-51mcu-segment-animation.mp4)

由于摄像头拍摄帧率的缘故，正好可以在视频中看到逐行刷新的效果。本代码已经在 KST-51 v1.3.2 开发板验证通过，代码如下：

```c
#include <REGX52.H>

sbit ADDR0 = P1^0;
sbit ADDR1 = P1^1;
sbit ADDR2 = P1^2;
sbit ADDR3 = P1^3;
sbit ENLED = P1^4;

// 图片帧
unsigned char code images[6][8] = {
    {0x00, 0x7E, 0x7E, 0x7E, 0x7E, 0x7E, 0x7E, 0x00},  // 8x8 点阵从外圈数第 1 圈显示的图片帧
    {0xFF, 0x81, 0xBD, 0xBD, 0xBD, 0xBD, 0x81, 0xFF},  // 8x8 点阵从外圈数第 2 圈显示的图片帧
    {0xFF, 0xFF, 0xC3, 0xDB, 0xDB, 0xC3, 0xFF, 0xFF},  // 8x8 点阵从外圈数第 3 圈显示的图片帧
    {0xFF, 0xFF, 0xFF, 0xE7, 0xE7, 0xFF, 0xFF, 0xFF},  // 8x8 点阵从外圈数第 4 圈显示的图片帧
    {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF},  // 8x8 点阵全部不显示的图片帧
    {0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00}   // 8x8 点阵全部显示的图片帧
};

bit flagcnt50ms = 0;                            // 50ms 的标志位变量
bit flagcnt3s = 0;                              // 3s 的标志位变量
unsigned char index = 0;                        // 图片帧索引变量

void main() {
    bit dir = 1;                                // 图片帧播放方向变量
    unsigned char cyclecnt = 0;                 // 图片帧播放次数变量

    EA = 1;                                     // 使能总中断
    ENLED = 0;                                  // 使能 U4，选择 LED 点阵
    ADDR3 = 0;
    TMOD &= 0xF0;                               // 设置 Timer0 为模式 1
    TMOD |= 0x01;
    TH0 = 0xFC;                                 // 为 Timer0 赋初值 0xFC67，定义 1ms
    TL0 = 0x67;
    ET0 = 1;                                    // 使能 Timer0 中断
    TR0 = 1;                                    // 启动 Timer0

    while (1) {
        if (flagcnt50ms) {                      // 如果 50ms 标志位置 1，即 50ms 到
            flagcnt50ms = 0;                    // 50ms 标志位清零
            if (dir) {                          // 如果 dir = 1
                index++;                        // 图片帧播放索引每次加 1，图片帧正向播放
                if (index >= 4) {               // 图片帧播放到第 4 帧
                    dir = 0;                    // dir 置 0
                }
            } else {
                index--;                        // 如果 dir = 0，图片帧索引每次减 1，图片反向播放
                if (index <= 0) {               // 图片索引播放到第 0 帧
                    dir = 1;                    // dir 置 1
                    cyclecnt++;                 // 循环播放变量加 1，也就是说正向播放后又反向播放了 1 次后加 1
                }
            }
        }
        if (cyclecnt >= 5) {                    // 如果图片帧循环播放了 5 次
            index = 5;                          // 图片播放第 5 帧，即 LED 点阵全部点亮
            if (flagcnt3s) {                    // 当前帧播放 3s 时长到
                flagcnt3s = 0;                  // 3s 标志位置 0
                index = 0;                      // 图片帧播放索引置 0
                cyclecnt = 0;                   // 图片帧循环播放次数置 0
                dir = 1;                        // 播放方向从正向开始
            }
        }
    }
}

void interruptTimer0() interrupt 1 {
    static unsigned int i = 0;
    static unsigned int cnt1ms_1 = 0;
    static unsigned int cnt1ms_2 = 0;

    TH0 = 0xFC;                                 // 重新加载 Timer0 初值
    TL0 = 0x67;
    cnt1ms_1++;                                 // 中断次数计数值加 1

    if (cnt1ms_1 >= 50) {                       // 中断进入 50 次，即计时 50ms
        cnt1ms_1 = 0;                           // 中断进入次数置零
        flagcnt50ms = 1;                        // 50ms 标志位置 1
    }
    if (index == 5) {                           // 图片刷新最后 1 帧时
        cnt1ms_2++;                             // 中断进入次数开始每次加 1
        if (cnt1ms_2 >= 3000) {                 // 中断进入 3000 次，即计时 3s
            cnt1ms_2 = 0;                       // 中断进入次数置零
            flagcnt3s = 1;                      // 3s 标志位置 1
        }
    }
    // 以下代码完成 LED 点阵的动态扫描刷新
    P0 = 0xFF;
    switch (i) {
        case 0: ADDR2 = 0; ADDR1 = 0; ADDR0 = 0; i++; P0 = images[index][0]; break;
        case 1: ADDR2 = 0; ADDR1 = 0; ADDR0 = 1; i++; P0 = images[index][1]; break;
        case 2: ADDR2 = 0; ADDR1 = 1; ADDR0 = 0; i++; P0 = images[index][2]; break;
        case 3: ADDR2 = 0; ADDR1 = 1; ADDR0 = 1; i++; P0 = images[index][3]; break;
        case 4: ADDR2 = 1; ADDR1 = 0; ADDR0 = 0; i++; P0 = images[index][4]; break;
        case 5: ADDR2 = 1; ADDR1 = 0; ADDR0 = 1; i++; P0 = images[index][5]; break;
        case 6: ADDR2 = 1; ADDR1 = 1; ADDR0 = 0; i++; P0 = images[index][6]; break;
        case 7: ADDR2 = 1; ADDR1 = 1; ADDR0 = 1; i = 0; P0 = images[index][7]; break;
    }
}
```

## 另一种方法

也可将图片帧播放过程放进中断：

```c
#include <REGX52.H>

sbit ADDR0 = P1^0;
sbit ADDR1 = P1^1;
sbit ADDR2 = P1^2;
sbit ADDR3 = P1^3;
sbit ENLED = P1^4;

// 图片帧
unsigned char code images[6][8] = {
    {0x00, 0x7E, 0x7E, 0x7E, 0x7E, 0x7E, 0x7E, 0x00},
    {0xFF, 0x81, 0xBD, 0xBD, 0xBD, 0xBD, 0x81, 0xFF},
    {0xFF, 0xFF, 0xC3, 0xDB, 0xDB, 0xC3, 0xFF, 0xFF},
    {0xFF, 0xFF, 0xFF, 0xE7, 0xE7, 0xFF, 0xFF, 0xFF},
    {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF},
    {0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00}
};

void main() {
    EA = 1;                                     // 使能总中断
    ENLED = 0;                                  // 使能 U4，选择 LED 点阵
    ADDR3 = 0;
    TMOD &= 0xF0;                               // 设置 Timer0 为模式 1
    TMOD |= 0x01;
    TH0 = 0xFC;                                 // 为 Timer0 赋初值 0xFC67，定义 1ms
    TL0 = 0x67;
    ET0 = 1;                                    // 使能 Timer0 中断
    TR0 = 1;                                    // 启动 Timer0

    while (1) {
        // 空循环，逻辑都在中断中处理
    }
}

void interruptTimer0() interrupt 1 {
    static unsigned int i = 0;
    static unsigned char index = 0;             // 图片帧索引变量
    static unsigned int cnt1ms_1 = 0;
    static unsigned int cnt1ms_2 = 0;
    static bit dir = 1;                         // 图片帧播放方向变量
    static unsigned char cyclecnt = 0;          // 图片帧播放次数变量
    static bit flagcnt50ms = 0;                 // 50ms 的标志位变量
    static bit flagcnt3s = 0;                   // 3s 的标志位变量

    TH0 = 0xFC;                                 // 重新加载 Timer0 初值
    TL0 = 0x67;
    cnt1ms_1++;                                 // 中断次数计数值加 1

    if (cnt1ms_1 >= 50) {                       // 中断进入 50 次，即计时 50ms
        cnt1ms_1 = 0;                           // 中断进入次数置零
        flagcnt50ms = 1;                        // 50ms 标志位置 1
    }
    if (index == 5) {                           // 图片刷新最后 1 帧时
        cnt1ms_2++;                             // 中断进入次数开始每次加 1
        if (cnt1ms_2 >= 3000) {                 // 中断进入 3000 次，即计时 3s
            cnt1ms_2 = 0;                       // 中断进入次数置零
            flagcnt3s = 1;                      // 3s 标志位置 1
        }
    }

    if (flagcnt50ms) {                          // 如果 50ms 标志位置 1，即 50ms 到
        flagcnt50ms = 0;                        // 50ms 标志位清零
        if (dir) {                              // 如果 dir = 1
            index++;                            // 图片帧播放索引每次加 1，图片帧正向播放
            if (index >= 4) {                   // 图片帧播放到第 4 帧
                dir = 0;                        // dir 置 0
            }
        } else {
            index--;                            // 如果 dir = 0，图片帧索引每次减 1，图片反向播放
            if (index <= 0) {                   // 图片索引播放到第 0 帧
                dir = 1;                        // dir 置 1
                cyclecnt++;                     // 循环播放变量加 1
            }
        }
    }
    if (cyclecnt >= 5) {                        // 如果图片帧循环播放了 5 次
        index = 5;                              // 图片播放第 5 帧，即 LED 点阵全部点亮
        if (flagcnt3s) {                        // 当前帧播放 3s 时长到
            flagcnt3s = 0;                      // 3s 标志位置 0
            index = 0;                          // 图片帧播放索引置 0
            cyclecnt = 0;                       // 图片帧循环播放次数置 0
            dir = 1;                            // 播放方向从正向开始
        }
    }
    // 以下代码完成 LED 点阵的动态扫描刷新
    P0 = 0xFF;
    switch (i) {
        case 0: ADDR2 = 0; ADDR1 = 0; ADDR0 = 0; i++; P0 = images[index][0]; break;
        case 1: ADDR2 = 0; ADDR1 = 0; ADDR0 = 1; i++; P0 = images[index][1]; break;
        case 2: ADDR2 = 0; ADDR1 = 1; ADDR0 = 0; i++; P0 = images[index][2]; break;
        case 3: ADDR2 = 0; ADDR1 = 1; ADDR0 = 1; i++; P0 = images[index][3]; break;
        case 4: ADDR2 = 1; ADDR1 = 0; ADDR0 = 0; i++; P0 = images[index][4]; break;
        case 5: ADDR2 = 1; ADDR1 = 0; ADDR0 = 1; i++; P0 = images[index][5]; break;
        case 6: ADDR2 = 1; ADDR1 = 1; ADDR0 = 0; i++; P0 = images[index][6]; break;
        case 7: ADDR2 = 1; ADDR1 = 1; ADDR0 = 1; i = 0; P0 = images[index][7]; break;
    }
}
```
