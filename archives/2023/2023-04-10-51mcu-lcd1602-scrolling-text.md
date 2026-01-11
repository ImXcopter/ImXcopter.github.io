# 51 单片机 - 1602 液晶显示游动字体 Coding 练习

效果展示：https://xcopter.cc/wp-content/uploads/2023/04/20230410-150.mp4

本程序代码为《手把手教你学 51 单片机》13.2 的例程修改版，主程序中将 2 个不同的字符串数组分别赋值，本代码已经在 KST-51 v1.3.2 开发板验证通过。

```c
#include <REGX52.H>
#define LCD1602_DB P0

sbit LCD1602_RS = P1^0;
sbit LCD1602_RW = P1^1;
sbit LCD1602_E = P1^5;

unsigned char T0RH = 0;                      // Timer0 重载值的高字节
unsigned char T0RL = 0;                      // Timer0 重载值的低字节
bit flagMove = 0;                            // 定时标志位
unsigned char code str1[] = "Xcopter.cc";    // 要显示的字符串 1
unsigned char code str2[] = "Xcopter's Blog"; // 要显示的字符串 2

void ConfigTimer(unsigned int ms);
void Init_LCD1602();
void LCD1602_ShowStr(unsigned char x, unsigned char y, unsigned char *ptrStr, unsigned char len);

void main()
{
    unsigned char i;
    unsigned char index = 0;                                   // 地址移动索引
    unsigned char pdata buffMove1[16 + sizeof(str1) + 16];     // 移动显示缓冲区 1
    unsigned char pdata buffMove2[16 + sizeof(str2) + 16];     // 移动显示缓冲区 2

    EA = 1;                                                    // 开总中断
    ConfigTimer(1);                                            // 配置 Timer0，定时 1ms
    Init_LCD1602();                                            // 初始化 LCD1602 液晶

    /* 给 buffMove1 数组赋值 */
    for (i = 0; i < 16; i++)
    {
        buffMove1[i] = ' ';                                    // 注意单引号之间是 1 个空格
    }
    for (i = 0; i < (sizeof(str1) - 1); i++)
    {
        buffMove1[i + 16] = str1[i];
    }
    for (i = (16 + (sizeof(str1) - 1)); i < sizeof(buffMove1); i++)
    {
        buffMove1[i] = ' ';
    }

    /* 给 buffMove2 数组赋值 */
    for (i = 0; i < 16; i++)
    {
        buffMove2[i] = ' ';
    }
    for (i = 0; i < (sizeof(str2) - 1); i++)
    {
        buffMove2[i + 16] = str2[i];
    }
    for (i = (16 + (sizeof(str2) - 1)); i < sizeof(buffMove2); i++)
    {
        buffMove2[i] = ' ';
    }

    while (1)
    {
        if (flagMove)                                         // 定时器标志位置 1，定时时间到
        {
            flagMove = 0;
            LCD1602_ShowStr(0, 0, buffMove1 + index, 16);     // buffMove1 数组的首地址加索引值，并取 16 个数据值
            LCD1602_ShowStr(0, 1, buffMove2 + index, 16);     // buffMove2 数组的首地址加索引值，并取 16 个数据值
            // 这里的 buffMove1 和 buffMove2 是数组指针，等同于 &buffMove1[0] 和 &buffMove2[0]，传递的是地址
            // 16 来限制在显示字符串时只显示其前 len 个字符，以保证字符串长度的正确性
            index++;                                          // 进入 Timer0 中断 1 次，地址索引值加 1
            if (index >= (16 + sizeof(str2) - 1))             // 起始位置到达字符串尾部后，即返回从头开始
            {
                index = 0;
            }
        }
    }
}

/* 配置并启动 Timer0，ms 为定时时间 */
void ConfigTimer(unsigned int ms)
{
    unsigned long reload;                                     // 临时变量

    reload = 11059200 / 12;                                   // 定时器计数频率
    reload = (reload * ms) / 1000;                            // 频率 * 时间变量，得到这个时间内的计数个数
    reload = 65536 - reload + 12;                             // 得出定时器初值，即重载值，并补偿中断延时的误差
    T0RH = (unsigned char)(reload >> 8);                      // 定时重载值拆分为高低字节
    T0RL = (unsigned char)reload;
    TMOD &= 0xF0;                                             // Timer0 控制位清零
    TMOD |= 0x01;                                             // 配置 Timer0 为模式 1
    TH0 = T0RH;                                               // 加载 Timer0 重载值
    TL0 = T0RL;
    ET0 = 1;                                                  // 使能 Timer0 中断
    TR0 = 1;                                                  // 启动 Timer0
}

/* LCD1602 忙状态检测函数 */
void LCD1602_Ready()
{
    unsigned char sta;

    LCD1602_DB = 0xFF;                                        // 读外部状态，首先自己要先置为高电平
    LCD1602_RS = 0;
    LCD1602_RW = 1;
    do
    {
        LCD1602_E = 1;
        sta = LCD1602_DB;                                     // 读状态字
        LCD1602_E = 0;
    } while (sta & 0x80);                                     // bit7 等于 1 表示液晶在忙，重复检测直到其等于 0 为止
}

/* 1602 写命令函数，cmd 为待写入的命令 */
void LCD1602_W_CMD(unsigned char cmd)
{
    LCD1602_Ready();
    LCD1602_RS = 0;
    LCD1602_RW = 0;
    LCD1602_DB = cmd;
    LCD1602_E = 1;
    LCD1602_E = 0;
}

/* 1602 写数据函数，dat 为待写入的数据 */
void LCD1602_W_DATA(unsigned char dat)
{
    LCD1602_Ready();
    LCD1602_RS = 1;
    LCD1602_RW = 0;
    LCD1602_DB = dat;
    LCD1602_E = 1;
    LCD1602_E = 0;
}

/* 设置显示 RAM 的其实地址，即光标位置 */
void LCD1602_SetCursor(unsigned char x, unsigned char y)
{
    unsigned char addr;

    if (y)                                                    // 由输入的屏幕坐标计算显示 RAM 的地址
    {
        addr = 0x40 + x;                                      // 第二行字符地址从 0x40 开始
    }
    else
    {
        addr = 0x00 + x;                                      // 第一行字符地址从 0x00 开始
    }
    LCD1602_W_CMD(addr | 0x80);                               // 写命令设置 RAM 地址
}

/* 1602 显示字符函数，ptrStr 为字符串指针 */
void LCD1602_ShowStr(unsigned char x, unsigned char y, unsigned char *ptrStr, unsigned char len)
{
    LCD1602_SetCursor(x, y);                                  // 设置显示字符串的起始地址

    while (len--)                                             // 连续写入 len 个字符数据
    {
        LCD1602_W_DATA(*ptrStr++);                            // 先取 ptrStr 指向的数据，然后 ptrStr 自加 1
    }
}

/* 1602 液晶初始化函数 */
void Init_LCD1602()
{
    LCD1602_W_CMD(0x38);                                      // 16 * 2 显示，5 * 7 点阵，8 位数据接口
    LCD1602_W_CMD(0x0C);                                      // 显示器开，光标关闭
    LCD1602_W_CMD(0x06);                                      // 文字不动，地址自动加 1
    LCD1602_W_CMD(0x01);                                      // 清屏
}

/* Timer0 中断服务函数 */
void InterruptTimer0() interrupt 1
{
    static unsigned int cnt = 0;

    TH0 = T0RH;
    TL0 = T0RL;
    cnt++;
    if (cnt >= 300)
    {
        cnt = 0;
        flagMove = 1;
    }
}
```
