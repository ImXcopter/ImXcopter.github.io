# 51 单片机 - 矩阵按键

## 原理图

![](/static/2023/2023-03-26-51mcu-matrix-keypad_001.png)

## 按键抖动

按键抖动必然存在，一般在 10ms 左右，采用定时器定期扫描，连续 16ms 按键状态一致，可以判定按键按下或者抬起状态。

## 单按键扫描

按键控制数码管显示，按一次，显示数字加一，0 到 9 循环往复。

```c
#include <REGX52.H>

sbit ADDR0 = P1^0;
sbit ADDR1 = P1^1;
sbit ADDR2 = P1^2;
sbit ADDR3 = P1^3;
sbit ENLED = P1^4;
sbit KEY1 = P2^4;
sbit KEY2 = P2^5;
sbit KEY3 = P2^6;
sbit KEY4 = P2^7;

unsigned char code LedChar[] = {  // 数码管显示字符转换表
    0xC0, 0xF9, 0xA4, 0xB0, 0x99, 0x92, 0x82, 0xF8,
    0x80, 0x90, 0x88, 0x83, 0xC6, 0xA1, 0x86, 0x8E
};

bit KeySta = 1;        // 当前按键状态，初始状态是高电平，即按键抬起
unsigned char THR0, TLR0;

void ConfigTimer(unsigned long ms);

void main()
{
    unsigned char cnt = 0;    // 保存按键按下次数，显示在数码管上
    bit backup = 1;           // 用于保留上一次按键状态，初始值为 1，按键抬起；

    ENLED = 0;
    ADDR3 = 1;
    ADDR2 = 0;
    ADDR1 = 0;
    ADDR0 = 0;

    ConfigTimer(1);           // 计算定时 1ms，需要赋的初值，存在在 THR0,TLR0 中
    TMOD = TMOD & 0xF0;
    TMOD = TMOD | 0x01;       // 设置定时器模式 1，不影响高四位
    TH0 = THR0;               // 定时器赋初值
    TL0 = TLR0;
    EA = 1;                   // 使能总中断
    ET0 = 1;                  // 使能 T0 中断
    TR0 = 1;                  // 启动定时器 T0
    P2 = 0xF7;                // P2.3 置 0，即 KeyOut1 输出低电平
    P0 = LedChar[cnt];        // 显示按键次数初值

    while(1)
    {
        if(KeySta != backup)  // 当前按键状态和上次按键状态不一致，有按键动作发生
        {
            if(KeySta == 0)   // 如果当前为 0，说明为按下动作
            {
                cnt++;
                if(cnt == 10)
                    cnt = 0;
                P0 = LedChar[cnt];
            }
            backup = KeySta;   // 更新上一次按键状态
        }
    }
}

void ConfigTimer(unsigned long ms)
{
    unsigned long temp;
    temp = 65536 - ms * 11059200 / 1000 / 12;  // ms 最大定时 71ms
    THR0 = (unsigned char)(temp >> 8);         // 取计数值高八位，计数值不会超过 65535
    TLR0 = (char)temp;                         // 取计数值低八位
}

void InterruptTimer0() interrupt 1  // 每间隔 1ms，点亮点整中的某一排，循环往复不停歇
{
    static unsigned char keybuf = 0xff;  // 按键状态保存区
    TH0 = THR0;                          // 定时器赋初值
    TL0 = TLR0;
    keybuf = (keybuf << 1) | KEY4;
    if(keybuf == 0x00)
    {
        KeySta = 0;   // 按键已经按下
    }
    else if(keybuf == 0xff)
    {
        KeySta = 1;   // 按键已经抬起
    }
    else
    {
        // 其他杂乱状态，保留原来按键状态，不更新
    }
}
```

## 矩阵按键

每 1ms 扫描一行按键，4ms 扫描完所有按键。如果 4 次扫描状态一致，可以确定按键状态（也就是 12ms 内状态一致）。

![](/static/2023/2023-03-26-51mcu-matrix-keypad_002.png)

功能：一共 4 行 4 列，16 个按键，按下对应按键，显示对应数字。

```c
#include <REGX52.H>

sbit ADDR0 = P1^0;
sbit ADDR1 = P1^1;
sbit ADDR2 = P1^2;
sbit ADDR3 = P1^3;
sbit ENLED = P1^4;
sbit KEY_IN_1 = P2^4;
sbit KEY_IN_2 = P2^5;
sbit KEY_IN_3 = P2^6;
sbit KEY_IN_4 = P2^7;
sbit KEY_OUT_1 = P2^3;
sbit KEY_OUT_2 = P2^2;
sbit KEY_OUT_3 = P2^1;
sbit KEY_OUT_4 = P2^0;

unsigned char code LedChar[] = {  // 数码管显示字符转换表
    0xC0, 0xF9, 0xA4, 0xB0, 0x99, 0x92, 0x82, 0xF8,
    0x80, 0x90, 0x88, 0x83, 0xC6, 0xA1, 0x86, 0x8E
};

unsigned char KeySta[4][4] = {  // 全部矩阵按键的当前状态
    {1, 1, 1, 1}, {1, 1, 1, 1}, {1, 1, 1, 1}, {1, 1, 1, 1}
};

unsigned char THR0, TLR0;

void ConfigTimer(unsigned long ms);

void main()
{
    unsigned char i, j;
    unsigned char backup[4][4] = {
        {1, 1, 1, 1}, {1, 1, 1, 1}, {1, 1, 1, 1}, {1, 1, 1, 1}
    };

    ENLED = 0;
    ADDR3 = 1;
    ADDR2 = 0;
    ADDR1 = 0;
    ADDR0 = 0;

    ConfigTimer(1);           // 计算定时 1ms
    TMOD = TMOD & 0xF0;
    TMOD = TMOD | 0x01;       // 设置定时器模式 1
    TH0 = THR0;
    TL0 = TLR0;
    EA = 1;
    ET0 = 1;
    TR0 = 1;
    P0 = LedChar[0];          // 数码管默认显示 0

    while(1)
    {
        for(i = 0; i < 4; i++)
        {
            for(j = 0; j < 4; j++)
            {
                if(KeySta[i][j] != backup[i][j])  // 当前按键状态和之前不一致
                {
                    if(KeySta[i][j] == 0)          // 按键按下
                    {
                        P0 = LedChar[i * 4 + j];
                    }
                    backup[i][j] = KeySta[i][j];   // 保存当前按键状态
                }
            }
        }
    }
}

void ConfigTimer(unsigned long ms)
{
    unsigned long temp;
    temp = 65536 - ms * 11059200 / 1000 / 12;
    THR0 = (unsigned char)(temp >> 8);
    TLR0 = (char)temp;
}

void InterruptTimer0() interrupt 1
{
    static unsigned char keyout = 0;
    unsigned char i;
    static unsigned char keybuf[4][4] = {
        {0xff, 0xff, 0xff, 0xff}, {0xff, 0xff, 0xff, 0xff},
        {0xff, 0xff, 0xff, 0xff}, {0xff, 0xff, 0xff, 0xff}
    };

    TH0 = THR0;
    TL0 = TLR0;

    keybuf[keyout][0] = (keybuf[keyout][0] << 1) | KEY_IN_1;
    keybuf[keyout][1] = (keybuf[keyout][1] << 1) | KEY_IN_2;
    keybuf[keyout][2] = (keybuf[keyout][2] << 1) | KEY_IN_3;
    keybuf[keyout][3] = (keybuf[keyout][3] << 1) | KEY_IN_4;

    for(i = 0; i <= 3; i++)
    {
        if((keybuf[keyout][i] & 0x0f) == 0x00)
        {
            KeySta[keyout][i] = 0;
        }
        else if((keybuf[keyout][i] & 0x0f) == 0x0f)
        {
            KeySta[keyout][i] = 1;
        }
    }

    keyout++;
    if(keyout >= 4)
    {
        keyout = 0;
    }

    switch(keyout)  // 扫描下一行
    {
        case 0:  KEY_OUT_4 = 1; KEY_OUT_1 = 0; break;
        case 1:  KEY_OUT_1 = 1; KEY_OUT_2 = 0; break;
        case 2:  KEY_OUT_2 = 1; KEY_OUT_3 = 0; break;
        case 3:  KEY_OUT_3 = 1; KEY_OUT_4 = 0; break;
        default: break;
    }
}
```

## 简易加法器

说明：按键显示对应数字在数码管；↑ 按键代表加号，Enter 按键代表等号，ESC 按键清零。

![](/static/2023/2023-03-26-51mcu-matrix-keypad_003.jpg)

```c
#include <REGX52.H>

sbit ADDR0 = P1^0;
sbit ADDR1 = P1^1;
sbit ADDR2 = P1^2;
sbit ADDR3 = P1^3;
sbit ENLED = P1^4;
sbit KEY_IN_1 = P2^4;
sbit KEY_IN_2 = P2^5;
sbit KEY_IN_3 = P2^6;
sbit KEY_IN_4 = P2^7;
sbit KEY_OUT_1 = P2^3;
sbit KEY_OUT_2 = P2^2;
sbit KEY_OUT_3 = P2^1;
sbit KEY_OUT_4 = P2^0;

unsigned char code LedChar[] = {  // 数码管显示字符转换表
    0xC0, 0xF9, 0xA4, 0xB0, 0x99, 0x92, 0x82, 0xF8,
    0x80, 0x90, 0x88, 0x83, 0xC6, 0xA1, 0x86, 0x8E
};

unsigned char LedBuff[6] = {0xff, 0xff, 0xff, 0xff, 0xff, 0xff};  // 数码管显示缓冲区

unsigned char code KeyCodeMap[4][4] = {  // 矩阵按键编号到标准键盘键码的映射表
    { 0x31, 0x32, 0x33, 0x26 },  // 数字键 1、数字键 2、数字键 3、向上键
    { 0x34, 0x35, 0x36, 0x25 },  // 数字键 4、数字键 5、数字键 6、向左键
    { 0x37, 0x38, 0x39, 0x28 },  // 数字键 7、数字键 8、数字键 9、向下键
    { 0x30, 0x1B, 0x0D, 0x27 }   // 数字键 0、ESC 键、回车键、向右键
};

unsigned char KeySta[4][4] = {  // 全部矩阵按键的当前状态
    {1, 1, 1, 1}, {1, 1, 1, 1}, {1, 1, 1, 1}, {1, 1, 1, 1}
};

unsigned char THR0, TLR0;

void KeyDriver();
void ConfigTimer(unsigned long ms);

void main()
{
    EA = 1;
    ENLED = 0;
    ADDR3 = 1;
    ConfigTimer(1);
    TMOD = TMOD & 0xF0;
    TMOD = TMOD | 0x01;
    TH0 = THR0;
    TL0 = TLR0;
    ET0 = 1;
    TR0 = 1;
    LedBuff[0] = LedChar[0];

    while(1)
    {
        KeyDriver();
    }
}

void ConfigTimer(unsigned long ms)
{
    unsigned long temp;
    temp = 65536 - ms * 11059200 / 1000 / 12;
    THR0 = (unsigned char)(temp >> 8);
    TLR0 = (char)temp;
}

void ShowNumber(unsigned long num)
{
    signed char i;
    unsigned char buf[6];  // 将 num 六位数字按顺序放入 buf 数组中

    for(i = 0; i <= 5; i++)
    {
        buf[i] = num % 10;
        num = num / 10;
    }

    for(i = 5; i >= 1; i--)  // 从高位到低位依次扫描
    {
        if(buf[i] == 0)
        {
            LedBuff[i] = 0xff;
        }
        else
        {
            break;
        }
    }

    for(; i >= 0; i--)
    {
        LedBuff[i] = LedChar[buf[i]];
    }
}

void KeyAction(unsigned char keycode)
{
    static unsigned long result = 0;
    static unsigned long addend = 0;

    if((keycode >= 0x30) && (keycode <= 0x39))  // 输入的是数字
    {
        addend = (addend * 10) + (keycode - 0x30);
        ShowNumber(addend);
    }
    else if(keycode == 0x26)  // 上键，执行加法操作
    {
        result += addend;
        addend = 0;
        ShowNumber(result);
    }
    else if(keycode == 0x0D)  // 回车键，作用和上键等同
    {
        result += addend;
        addend = 0;
        ShowNumber(result);
    }
    else if(keycode == 0x1B)  // ESC 键，清零
    {
        addend = 0;
        result = 0;
        ShowNumber(addend);
    }
}

void KeyDriver()
{
    unsigned char i, j;
    static unsigned char backup[4][4] = {  // 一定定义成静态，否则 bug
        {1, 1, 1, 1}, {1, 1, 1, 1}, {1, 1, 1, 1}, {1, 1, 1, 1}
    };

    for(i = 0; i < 4; i++)
    {
        for(j = 0; j < 4; j++)
        {
            if(KeySta[i][j] != backup[i][j])
            {
                if(KeySta[i][j] == 0)
                {
                    KeyAction(KeyCodeMap[i][j]);
                }
                backup[i][j] = KeySta[i][j];
            }
        }
    }
}

void LedScan()
{
    static unsigned char i = 0;
    P0 = 0xff;
    switch(i)
    {
        case 0: ADDR2 = 0; ADDR1 = 0; ADDR0 = 0; P0 = LedBuff[i]; i++; break;
        case 1: ADDR2 = 0; ADDR1 = 0; ADDR0 = 1; P0 = LedBuff[i]; i++; break;
        case 2: ADDR2 = 0; ADDR1 = 1; ADDR0 = 0; P0 = LedBuff[i]; i++; break;
        case 3: ADDR2 = 0; ADDR1 = 1; ADDR0 = 1; P0 = LedBuff[i]; i++; break;
        case 4: ADDR2 = 1; ADDR1 = 0; ADDR0 = 0; P0 = LedBuff[i]; i++; break;
        case 5: ADDR2 = 1; ADDR1 = 0; ADDR0 = 1; P0 = LedBuff[i]; i = 0; break;
        default: break;
    }
}

void KeyScan()
{
    static unsigned char keyout = 0;
    unsigned char i;
    static unsigned char keybuf[4][4] = {
        {0xff, 0xff, 0xff, 0xff}, {0xff, 0xff, 0xff, 0xff},
        {0xff, 0xff, 0xff, 0xff}, {0xff, 0xff, 0xff, 0xff}
    };

    keybuf[keyout][0] = (keybuf[keyout][0] << 1) | KEY_IN_1;
    keybuf[keyout][1] = (keybuf[keyout][1] << 1) | KEY_IN_2;
    keybuf[keyout][2] = (keybuf[keyout][2] << 1) | KEY_IN_3;
    keybuf[keyout][3] = (keybuf[keyout][3] << 1) | KEY_IN_4;

    for(i = 0; i <= 3; i++)
    {
        if((keybuf[keyout][i] & 0x0f) == 0x00)
        {
            KeySta[keyout][i] = 0;
        }
        else if((keybuf[keyout][i] & 0x0f) == 0x0f)
        {
            KeySta[keyout][i] = 1;
        }
    }

    keyout++;
    if(keyout >= 4)
    {
        keyout = 0;
    }

    switch(keyout)  // 扫描下一行
    {
        case 0:  KEY_OUT_4 = 1; KEY_OUT_1 = 0; break;
        case 1:  KEY_OUT_1 = 1; KEY_OUT_2 = 0; break;
        case 2:  KEY_OUT_2 = 1; KEY_OUT_3 = 0; break;
        case 3:  KEY_OUT_3 = 1; KEY_OUT_4 = 0; break;
        default: break;
    }
}

void InterruptTimer0() interrupt 1
{
    TH0 = THR0;
    TL0 = TLR0;
    LedScan();
    KeyScan();
}
```
