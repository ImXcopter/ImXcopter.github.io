# 51 单片机 - 秒表程序 Coding 练习

本程序是矩阵按键扫描，数码管动态扫描，定时器中断的综合练习，其中秒表显示计数函数用了 2 种写法，是 2 种不同的思路。

代码已经在 KST-51 v1.3.2 开发板验证通过。

![](/static/2023/2023-04-03-51mcu-stopwatch-practice_001.png)

```c
#include <REGX52.H>

sbit ADDR0 = P1^0;
sbit ADDR1 = P1^1;
sbit ADDR2 = P1^1;
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

// 数码管显示字符转换表
unsigned char code ledChar[16] = {
    0xC0, 0xF9, 0xA4, 0xB0, 0x99, 0x92, 0x82, 0xF8,
    0x80, 0x90, 0x88, 0x83, 0xC6, 0xA1, 0x86, 0x8E
};

// 数码管显示缓冲区
unsigned char ledBuff[6] = {
    0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF
};

// 矩阵按键真值表
unsigned char code keyCodeMap[4][4] = {
    { 0x31, 0x32, 0x33, 0x26 },  // 数字键 1、数字键 2、数字键 3、向上键
    { 0x34, 0x35, 0x36, 0x25 },  // 数字键 4、数字键 5、数字键 6、向左键
    { 0x37, 0x38, 0x39, 0x28 },  // 数字键 7、数字键 8、数字键 9、向下键
    { 0x30, 0x1B, 0x0D, 0x27 }   // 数字键 0、ESC 键、回车键、向右键
};

// 按键当前状态
unsigned char keySta[4][4] = {
    {1, 1, 1, 1}, {1, 1, 1, 1}, {1, 1, 1, 1}, {1, 1, 1, 1}
};

unsigned char T0RH = 0;       // Timer0 重载值的高字节（高 8 位）
unsigned char T0RL = 0;       // Timer0 重载值的低字节（低 8 位）
bit stopWatchRunning = 0;     // 秒表运行标志位
bit stopWatchRefresh = 1;     // 秒表刷新标志位
unsigned int integerPart = 0; // 秒表的整数部分
unsigned char decimalPart = 0; // 表秒的小数部分

void ConfigTimer0(unsigned int ms);
void StopWatchDisplay();
void KeyDriver();

void main()
{
    EA = 1;                    // 使能总中断
    ENLED = 0;                 // 选择数码管进行显示
    ADDR3 = 1;
    ConfigTimer0(2);           // 配置 Timer0，定时 2ms

    while (1)
    {
        if (stopWatchRefresh) // 秒表数刷新
        {
            stopWatchRefresh = 0;
            StopWatchDisplay(); // 显示当前的秒表数值
        }
        KeyDriver();           // 按键驱动函数
    }
}

// 配置并启动 Timer0，ms 为 Timer0 的定时时间，单位毫秒
void ConfigTimer0(unsigned int ms)
{
    unsigned long reload;      // 临时变量

    reload = 11059200 / 12;    // 定时器的计数频率
    reload = (reload * ms) / 1000; // 计算所需的计数值，也就是溢出值
    reload = 65536 - reload + 18;  // 计算定时器的重载值，也就是初值，并且补偿中断响应误差
    T0RH = (unsigned char)(reload >> 8); // 定时器重载值拆分为高低字节
    T0RL = (unsigned char)reload;
    TMOD &= 0xF0;              // 清零 Timer0 的控制位
    TMOD |= 0x01;              // 配置 Timer0 为模式 1
    TH0 = T0RH;                // 加载 Timer0 的重载值
    TL0 = T0RL;
    ET0 = 1;                   // 使能 Timer0 中断
    TR0 = 1;                   // 启动 Timer0
}

/*
// 秒表计数显示函数
void StopWatchDisplay()
{
    signed char i;
    unsigned char buff[4];

    // 将小数部分分离，并给数码管缓冲区赋值
    ledBuff[0] = ledChar[decimalPart % 10];
    ledBuff[1] = ledChar[decimalPart / 10 % 10];
    // 将整数部分分离
    buff[0] = integerPart % 10;
    buff[1] = integerPart / 10 % 10;
    buff[2] = integerPart / 100 % 10;
    buff[3] = integerPart / 1000 % 10;
    // 整数部分高位为零不显示
    for (i = 3; i >= 1; i--)
    {
        if (buff[i] == 0)
        {
            ledBuff[i + 2] = 0xFF;
        }
        else
        {
            break;
        }
    }
    for ( ; i >= 0; i--)
    {
        ledBuff[i + 2] = ledChar[buff[i]];
    }
    // 点亮小数点
    ledBuff[2] &= 0x7F;
}
*/

// 秒表计数显示函数
void StopWatchDisplay()
{
    signed char i;
    static unsigned char buff[6];
    static unsigned int temp;

    temp = integerPart;        // 将整数部分赋给静态变量 temp
    buff[0] = decimalPart % 10; // 分离小数部分，并赋值给数值缓冲区
    buff[1] = decimalPart / 10 % 10;

    for (i = 2; i <= 5; i++)   // 循环分离整数部分，并赋值给数值缓冲区
    {
        buff[i] = temp % 10;
        temp /= 10;
    }

    for (i = 5; i >= 3; i--)   // 数码管的高位遇到 0 不显示
    {
        if (buff[i] == 0)
        {
            ledBuff[i] = 0xFF;
        }
        else
        {
            break;
        }
    }
    for ( ; i >= 0; i--)
    {
        ledBuff[i] = ledChar[buff[i]];
    }
    ledBuff[2] &= 0x7F;        // 点亮数码管小数点
}

// 按键动作函数
void KeyAction(unsigned char keyCode)
{
    if (keyCode == 0x0D)       // 按下回车键
    {
        if (stopWatchRunning)  // 如果秒表是运行状态
        {
            stopWatchRunning = 0; // 让秒表停止状态
        }
        else if (stopWatchRunning == 0) // 如果秒表是停止状态
        {
            stopWatchRunning = 1; // 让秒表运行状态
        }
    }
    else if (keyCode == 0x1B)  // 按下 ESC 键
    {
        stopWatchRunning = 0;  // 秒表停止
        integerPart = 0;       // 秒表整数部分清零
        decimalPart = 0;       // 秒表小数部分清零
        stopWatchRefresh = 1;  // 秒表刷新
    }
}

// 按键驱动函数
void KeyDriver()
{
    unsigned char i, j;
    static unsigned char keyBackup[4][4] = { // 按键备份值，保存前一次的值
        {1, 1, 1, 1}, {1, 1, 1, 1}, {1, 1, 1, 1}, {1, 1, 1, 1}
    };

    for (i = 0; i < 4; i++)   // 循环检测 4 * 4 的矩阵按键
    {
        for (j = 0; j < 4; j++)
        {
            if (keySta[i][j] != keyBackup[i][j]) // 检测当前按键值和备份值不一致，说明按键有动作了
            {
                if (keySta[i][j] == 0) // 如果当前按键是按下的状态
                {
                    KeyAction(keyCodeMap[i][j]); // 调用动作函数
                }
                keyBackup[i][j] = keySta[i][j]; // 备份一次当前的按键值
            }
        }
    }
}

// 数码管扫描函数
void LedScan()
{
    static unsigned char i = 0;

    P0 = 0xFF;                 // 关闭所有段选位，消隐
    P1 = (P1 & 0xF8) | i;      // 位选索引赋值到 P1 口的低 3 位
    P0 = ledBuff[i];           // 缓冲区索引位置的数据送到 P0 口，
    if (i < 5)                 // 索引递增循环，遍历整个缓冲区
    {
        i++;
    }
    else
    {
        i = 0;
    }
}

// 按键扫描函数
void KeyScan()
{
    unsigned char i;
    static unsigned keyout = 0;
    // 按键扫描缓冲区
    static unsigned char keyBuff[4][4] = {
        {0xFF, 0xFF, 0xFF, 0xFF}, {0xFF, 0xFF, 0xFF, 0xFF},
        {0xFF, 0xFF, 0xFF, 0xFF}, {0xFF, 0xFF, 0xFF, 0xFF}
    };
    // 按键值一位一位的移入缓冲区
    keyBuff[keyout][0] = (keyBuff[keyout][0] << 1) | keyIn1;
    keyBuff[keyout][1] = (keyBuff[keyout][1] << 1) | keyIn2;
    keyBuff[keyout][2] = (keyBuff[keyout][2] << 1) | keyIn3;
    keyBuff[keyout][3] = (keyBuff[keyout][3] << 1) | keyIn4;
    // 每行 4 个按键，所以循环 4 次
    for (i = 0; i < 4; i++)
    {
        if ((keyBuff[keyout][i] & 0x0F) == 0x00) // 连续 4 次扫描值为 0
        {
            keySta[keyout][i] = 0;
        }
        else if ((keyBuff[keyout][i] & 0x0F) == 0x0F) // 连续 4 次扫描值为 1
        {
            keySta[keyout][i] = 1;
        }
    }

    keyout++;                  // 输出索引递增
    keyout &= 0x03;            // 输出索引加到 4 即归零

    switch (keyout)            // 根据索引，释放当前的输出引脚，拉低下次的输出引脚
    {
        case 0: keyOut4 = 1; keyOut1 = 0; break;
        case 1: keyOut1 = 1; keyOut2 = 0; break;
        case 2: keyOut2 = 1; keyOut3 = 0; break;
        case 3: keyOut3 = 1; keyOut4 = 0; break;
        default: break;
    }
}

// 秒表计数函数，每 10ms 调用一次秒表计数累加
void StopWatchCount()
{
    if (stopWatchRunning)      // 秒表处于运行状态
    {
        decimalPart++;         // 小数部分 +1
        if (decimalPart >= 100) // 小数部分加到 100 时进位到整数部分
        {
            decimalPart = 0;   // 小数部分清零
            integerPart++;     // 整数部分 +1
            if (integerPart >= 10000) // 整数部分加到 10000 时归零
            {
                integerPart = 0;
            }
        }
        stopWatchRefresh = 1;  // 秒表刷新
    }
}

void InterruptTimer0() interrupt 1
{
    static unsigned int tmr10ms = 0;
    TH0 = T0RH;                // 重新加载重载值
    TL0 = T0RL;

    LedScan();                 // 数码管刷新
    KeyScan();                 // 矩阵按键刷新
    // 定时 10ms 进行一次秒表计数
    tmr10ms++;
    if (tmr10ms >= 5)
    {
        tmr10ms = 0;
        StopWatchCount();
    }
}
```
