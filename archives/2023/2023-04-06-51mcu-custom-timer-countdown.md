# 51 单片机 - 自定义时间秒表倒计时代码

本次代码练习，是学习完《手把手教你学 51 单片机(C 语言版)》第十章的一个综合练习。

```c
#include <REGX52.H>

sbit BUZZ = P1^6;
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
unsigned char ledBuff[7] = {
    0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF
};

// 矩阵按键真值表
unsigned char code keyCodeMap[4][4] = {  // 矩阵按键编号到标准键盘键码的映射表
    { 0x31, 0x32, 0x33, 0x26 },  // 数字键 1、数字键 2、数字键 3、向上键
    { 0x34, 0x35, 0x36, 0x25 },  // 数字键 4、数字键 5、数字键 6、向左键
    { 0x37, 0x38, 0x39, 0x28 },  // 数字键 7、数字键 8、数字键 9、向下键
    { 0x30, 0x1B, 0x0D, 0x27 }   // 数字键 0、ESC 键、回车键、向右键
};

// 矩阵按键当前状态
unsigned char keySta[4][4] = {
    {1, 1, 1, 1}, {1, 1, 1, 1}, {1, 1, 1, 1}, {1, 1, 1, 1}
};

// 每个按键按下的持续时间，单位 ms
unsigned long pdata keyDownTime[4][4] = {
    {0, 0, 0, 0}, {0, 0, 0, 0}, {0, 0, 0, 0}, {0, 0, 0, 0}
};

bit enBuzz = 0;                  // 蜂鸣器使能标志
bit stopWatchRunning = 0;        // 秒表运行标志
bit stopWatchRefresh = 1;        // 秒表刷新标志
bit flagLed = 0;                 // LED 流水灯标志
unsigned char T0RH = 0;          // Timer0 高字节重载值
unsigned char T0RL = 0;          // Timer0 低字节重载值
unsigned int integerPart = 0;    // 秒表整数部分
signed char decimalPart = 0;     // 秒表小数部分

void ConfigTimer0(unsigned int ms);  // Timer0 配置函数
void KeyDriver();                    // 矩阵按键驱动函数
void StopWatchDisplay();             // 秒表显示函数
void LedMovement();                  // LED 流水灯函数

void main()
{
    EA = 1;                      // 使能总中断
    ENLED = 0;                   // 选择数码管和独立 LED 灯
    ADDR3 = 1;
    ConfigTimer0(1);             // 配置 Timer0，定时 1ms

    while (1)
    {
        if (stopWatchRefresh)   // 秒表刷新位置 1
        {
            stopWatchRefresh = 0; // 秒表刷新位清零
            StopWatchDisplay();  // 显示秒表数
        }
        if (stopWatchRunning)   // 秒表运行状态
        {
            if (flagLed)        // LED 流水灯标志位置 1
            {
                flagLed = 0;    // LED 流水灯标志位清零
                LedMovement();  // 调用 LED 流水灯函数
            }
        }
        KeyDriver();            // 调用矩阵按键驱动函数
    }
}

// 配置并启动 Timer0，ms 为 Timer0 的定时时间
void ConfigTimer0(unsigned int ms)
{
    unsigned long reload;        // 重载值计算临时变量

    reload = 11059200 / 12;      // 定时器的计数频率
    reload = (reload * ms) / 1000; // 计算定时器所需的计数值
    reload = 65536 - reload + 12; // 计算定时器的重载值并补偿中断误差
    T0RH = (unsigned char)(reload >> 8); // 定时器重载值拆分为高低两个字节
    T0RL = (unsigned char)reload;
    TMOD &= 0xF0;                // 清零 Timer0 控制位
    TMOD |= 0x01;                // 配置 Timer0 为模式 1
    TH0 = T0RH;                  // 加载 Timer0 重载值
    TL0 = T0RL;
    ET0 = 1;                     // 使能 Timer0 中断
    TR0 = 1;                     // 启动 Timer0
}

// LED 流水灯函数
void LedMovement()
{
    static bit dir = 1;          // 方向标志位
    static unsigned char shift = 0x01; // 转换变量

    ledBuff[6] = ~shift;         // 将转换变量取反，赋值给 LED 灯的缓冲区
    if (dir)                     // 方向标志位为 1
    {
        shift = shift << 1;      // 转换变量 shift 左移 1 位
        if (shift == 0x80)       // shift 为 0x80，即 1000 0000，移到了最左边
        {
            dir = 0;             // 方向标志位置 0
        }
    }
    else
    {
        shift = shift >> 1;      // 转换变量 shift 右移 1 位
        if (shift == 0x01)       // shift 为 0x01，即 0000 0001，移到了最右边
        {
            dir = 1;             // 方向标志位置 1
        }
    }
}

// 秒表显示函数
void StopWatchDisplay()
{
    signed char i;
    unsigned char buff[6];       // 秒表数值计算缓冲数组

    buff[0] = decimalPart % 10;            // 从最右侧数第 0 个数码管小数部分
    buff[1] = decimalPart / 10 % 10;       // 从最右侧数第 1 个数码管小数部分
    buff[2] = integerPart % 10;            // 从最右侧数第 2 个数码管整数部分
    buff[3] = integerPart / 10 % 10;       // 从最右侧数第 3 个数码管整数部分
    buff[4] = integerPart / 100 % 10;      // 从最右侧数第 4 个数码管整数部分
    buff[5] = integerPart / 1000 % 10;     // 从最右侧数第 5 个数码管整数部分

    for (i = 5; i >= 3; i--)   // 从最高位起，遇到数值为 0 的转换为不显示，遇到非 0 退出循环
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
    for ( ; i >= 0; i--)       // 剩余的低位都如实转换为数码管显示字符
    {
        ledBuff[i] = ledChar[buff[i]];
    }
    ledBuff[2] &= 0x7F;        // 点亮数码管的小数点
}

// 按键动作函数，根据键值码执行响应的操作，keyCode 为按键码
void KeyAction(unsigned char keyCode)
{
    if (stopWatchRunning == 0)  // 秒表未运行状态
    {
        if ((keyCode >= 0x30) && (keyCode <= 0x39)) // 按键码是数字部分
        {
            integerPart = (integerPart * 10) + (keyCode - 0x30); // 整数部分整体十进制左移，新数字进入个位
            StopWatchDisplay();       // 结果显示到数码管
        }
        else if ((keyCode == 0x26) && (integerPart < 9999)) // 向上键，并且整数部分小于 9999
        {
            integerPart++;            // 整数部分自加 1
            StopWatchDisplay();
        }
        else if ((keyCode == 0x28) && (integerPart > 0)) // 向下键，并且整数部分大于 0
        {
            integerPart--;            // 整数部分自减 1
            StopWatchDisplay();
        }
        else if (keyCode == 0x0D)   // 回车键
        {
            stopWatchRunning = 1;    // 秒表开始运行
        }
    }
}

void KeyDriver()
{
    unsigned char i, j;
    static unsigned char keyStaBackup[4][4] = { // 按键值备份，保存上一次的值
        {1, 1, 1, 1}, {1, 1, 1, 1}, {1, 1, 1, 1}, {1, 1, 1, 1}
    };
    static unsigned long pdata timeThr[4][4] = { // 连续输入执行时间的阈值
        {1000, 1000, 1000, 1000}, {1000, 1000, 1000, 1000},
        {1000, 1000, 1000, 1000}, {1000, 1000, 1000, 1000}
    };

    for (i = 0; i < 4; i++)       // 循环扫描 4 * 4 矩阵按键
    {
        for (j = 0; j < 4; j++)
        {
            if (keySta[i][j] != keyStaBackup[i][j]) // 检测按键动作
            {
                if (keySta[i][j] == 0)  // 按键按下时执行动作
                {
                    KeyAction(keyCodeMap[i][j]); // 调用按键动作函数
                }
                keyStaBackup[i][j] = keySta[i][j]; // 刷新前一次的备份值
            }
            if (keyDownTime[i][j] > 0) // 检测按键动作，也可理解为检测按键持续按下状态
            {
                if (keyDownTime[i][j] >= timeThr[i][j]) // 按键时间达到阈值执行一次动作
                {
                    KeyAction(keyCodeMap[i][j]); // 调用按键动作函数
                    timeThr[i][j] += 100;        // 阈值时间增加 100ms，以准备下次时间的比较
                }
            }
            else
            {
                timeThr[i][j] = 1000;  // 恢复 1000ms 的初始阈值时间
            }
        }
    }
}

// 秒表计数函数
void StopWatchCount()
{
    if (stopWatchRunning)         // 秒表处于运行状态
    {
        decimalPart--;            // 秒表小数部分自减 1
        if (decimalPart <= 0)     // 秒表小数部分值小于等于 0
        {
            if (integerPart == 0) // 秒表整数部分为 0
            {
                stopWatchRunning = 0; // 秒表停止运行
                enBuzz = 1;       // 使能蜂鸣器
                ledBuff[6] = 0x00; // LED 灯全部点亮
            }
            else
            {
                decimalPart = 100; // 秒表小数部分恢复值 100
                integerPart--;     // 秒表整数部分自减 1
            }
        }
        stopWatchRefresh = 1;     // 秒表刷新位置 1
    }
}

// 数码管扫描函数
void LedScan()
{
    static unsigned char i = 0;   // 动态扫描索引

    P0 = 0xFF;                    // 关闭数码管段，消隐
    P1 = (P1 & 0xF8) | i;         // 清零 P1 口的低 3 位，并且位选当前索引
    P0 = ledBuff[i];              // 缓冲区索引位置的数据，送到 P0 口
    if (i < 6)                    // 索引递增循环，遍历整个缓冲区
    {
        i++;
    }
    else
    {
        i = 0;
    }
}

// 矩阵按键扫描函数
void KeyScan()
{
    unsigned char i;
    static unsigned char keyOut = 0;          // 矩阵按键扫描输出索引
    static unsigned char keyBuff[4][4] = {    // 矩阵按键扫描缓冲区
        {0xFF, 0xFF, 0xFF, 0xFF}, {0xFF, 0xFF, 0xFF, 0xFF},
        {0xFF, 0xFF, 0xFF, 0xFF}, {0xFF, 0xFF, 0xFF, 0xFF}
    };
    // 将一行的 4 个按键值移入缓冲区
    keyBuff[keyOut][0] = (keyBuff[keyOut][0] << 1) | keyIn1;
    keyBuff[keyOut][1] = (keyBuff[keyOut][1] << 1) | keyIn2;
    keyBuff[keyOut][2] = (keyBuff[keyOut][2] << 1) | keyIn3;
    keyBuff[keyOut][3] = (keyBuff[keyOut][3] << 1) | keyIn4;
    // 消抖后更新按键状态
    for (i = 0; i < 4; i++)       // 每行 4 个按键，所以循环 4 次
    {
        if ((keyBuff[keyOut][i] & 0xFF) == 0x00) // 连续 4 次扫描值为 0
        {
            keySta[keyOut][i] = 0;
            keyDownTime[keyOut][i] += 4; // 按下的持续时间累加
        }
        else if ((keyBuff[keyOut][i] & 0xFF) == 0xFF) // 连续 4 次扫描值为 1
        {
            keySta[keyOut][i] = 1;
            keyDownTime[keyOut][i] = 0; // 按下的持续时间清零
        }
    }

    keyOut++;                     // 输出索引递增
    keyOut &= 0x03;               // 输出索引遇 4 归零
    switch (keyOut)               // 根据输出索引，释放当前的输出引脚，拉低下次的输出引脚
    {
        case 0: keyOut4 = 1; keyOut1 = 0; break;
        case 1: keyOut1 = 1; keyOut2 = 0; break;
        case 2: keyOut2 = 1; keyOut3 = 0; break;
        case 3: keyOut3 = 1; keyOut4 = 0; break;
        default: break;
    }
}

// 定时器 0 中断服务函数，完成数码管和矩阵键盘的扫描与定时
void InterruptTimer0() interrupt 1
{
    static unsigned int tmr10ms = 0;  // 10ms 定时变量
    static unsigned int tmrLed = 0;   // LED 流水灯定时变量

    TH0 = T0RH;                // 重新加载重载值
    TL0 = T0RL;
    LedScan();                 // 数码管动态扫描
    KeyScan();                 // 矩阵按键扫描

    tmr10ms++;
    tmrLed++;
    if (tmr10ms >= 10)         // 10ms 时间到
    {
        tmr10ms = 0;
        StopWatchCount();      // 调用秒表函数
    }
    if (tmrLed >= 50)          // LED 流水灯时间到
    {
        tmrLed = 0;
        flagLed = 1;           // LED 流水灯标志位置 1
    }
    if (enBuzz)                // 蜂鸣器标志位使能
    {
        BUZZ = ~BUZZ;          // 蜂鸣器发声处理
    }
    else
    {
        BUZZ = 1;              // 关闭蜂鸣器
    }
}
```
