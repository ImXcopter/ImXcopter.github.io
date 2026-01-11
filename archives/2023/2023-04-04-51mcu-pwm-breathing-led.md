# 51 单片机 - PWM 控制 P0 口 LED 呼吸灯

通过 IO 口模拟产生 PWM，并通过占空比的控制 LED 灯的亮度。定时器 0 用来产生 PWM，定时器 1 用来调整占空比的数值。

代码已经在 KST-51 v1.3.2 开发板验证通过。

效果展示：https://xcopter.cc/wp-content/uploads/2023/04/20230404-280.mp4

```c
#include <REGX52.H>

sbit ADDR0 = P1^0;
sbit ADDR1 = P1^1;
sbit ADDR2 = P1^2;
sbit ADDR3 = P1^3;
sbit ENLED = P1^4;

unsigned long periodCnt = 0;
unsigned char highRH = 0;       // 高电平重载值的高字节
unsigned char highRL = 0;       // 高电平重载值的低字节
unsigned char lowRH = 0;        // 低电平重载值的高字节
unsigned char lowRL = 0;        // 低电平重载值的低字节
unsigned char T1RH = 0;
unsigned char T1RL = 0;

void ConfigPWM(unsigned int freq, unsigned char dutyCycle);
void ConfigTimer1(unsigned int ms);
void AdjustDutyCycle(unsigned char dutyCycle);

void main()
{
    EA = 1;                     // 开总中断
    ENLED = 0;
    ADDR3 = 1;
    ADDR2 = 1;
    ADDR1 = 1;
    ADDR0 = 0;

    ConfigPWM(100, 10);         // 配置并启动 PWM
    ConfigTimer1(100);          // 用 Timer1 定时调整占空比

    while (1);
}

void ConfigPWM(unsigned int freq, unsigned char dutyCycle)
{
    unsigned int high, low;

    periodCnt = 11059200 / 12 / freq; // 计算一个周期所需的计数值
    high = (periodCnt * dutyCycle) / 100; // 计算高电平所需的计数值
    low = periodCnt - high;        // 计算低电平所需的计数值
    high = 65536 - high + 12;      // 计算高电平的定时器重载值并补偿中断延时
    low = 65536 - low + 12;        // 计算低电平的定时器重载值并补偿中断延时
    highRH = (unsigned char)(high >> 8); // 高电平重载值拆分为高低字节
    highRL = (unsigned char)high;
    lowRH = (unsigned char)(low >> 8);  // 低电平重载值拆分为高低字节
    lowRL = (unsigned char)low;
    TMOD &= 0xF0;              // 清零 Timer0 的控制位
    TMOD |= 0x01;              // 配置 Timer0 为模式 1
    TH0 = highRH;              // 加载 Timer0 重载值
    TL0 = highRL;
    ET0 = 1;                   // 使能 Timer0 中断
    TR0 = 1;                   // 启动 Timer0
    P0 = 0xFF;                 // P0 口全部输出高电平
}

void ConfigTimer1(unsigned int ms)
{
    unsigned long temp;

    temp = 11059200 / 12;          // 定时器计数频率
    temp = (temp * ms) / 1000;     // 计算所需的计数值
    temp = 65536 - temp + 12;      // 计算定时器重载值
    T1RH = (unsigned char)(temp >> 8);  // 定时器重载值拆分为高低字节
    T1RL = (unsigned char)temp;
    TMOD &= 0x0F;                  // 清零 Timer1 的控制位
    TMOD |= 0x10;                  // 配置 Timer1 为模式 1
    TH1 = T1RH;                    // 加载 Timer1 重载值
    TL1 = T1RL;
    ET1 = 1;                       // 使能 Timer1 中断
    TR1 = 1;                       // 启动 Timer1
}

void AdjustDutyCycle(unsigned char dutyCycle)
{
    unsigned int high, low;

    high = (periodCnt * dutyCycle) / 100;
    low = periodCnt - high;
    high = 65536 - high + 12;
    low = 65536 - low + 12;
    highRH = (unsigned char)(high >> 8);
    highRL = (unsigned char)high;
    lowRH = (unsigned char)(low >> 8);
    lowRL = (unsigned char)low;
}

void InterruptTimer0() interrupt 1
{
    if (P0 == 0xFF)            // 当前 P0 口输出为高电平时，装载低电平值并输出低电平
    {
        TH0 = lowRH;
        TL0 = lowRL;
        P0 = 0x00;
    }
    else if (P0 == 0x00)       // 当前 P0 口输出为低电平时，装载高电平值并输出高电平
    {
        TH0 = highRH;
        TL0 = highRL;
        P0 = 0xFF;
    }
}

void InterruptTimer1() interrupt 3
{
    static bit dir = 1;
    static signed char index = 0;
    unsigned char code table[19] = {
        5, 10, 15, 20, 25, 30, 35, 40, 45, 50, 55, 60, 65, 70, 75, 80, 85, 90, 95
    };

    TH1 = T1RH;
    TL1 = T1RL;

    AdjustDutyCycle(table[index]);
    if (dir)
    {
        index++;
        if (index >= 18)
        {
            dir = 0;
        }
    }
    else
    {
        index--;
        if (index <= 0)
        {
            dir = 1;
        }
    }
}
```
