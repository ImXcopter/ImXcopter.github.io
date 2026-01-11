# 51单片机 - 1602液晶显示游动字体Coding练习

效果展示：
[bradmax\_video url="https://xcopter.cc/wp-content/uploads/2023/04/20230410-150.mp4" duration="19" poster="/static/2023/2023-04-10-51mcu-lcd1602-scrolling-text\_001.png"]
本程序代码为《手把手教你学51单片机》13.2的例程修改版，主程序中将2个不同的字符串数组分别赋值，本代码已经在KST-51 v1.3.2开发板验证通过。

```
#include 
#define LCD1602_DB P0

sbit LCD1602_RS = P1^0;
sbit LCD1602_RW = P1^1;
sbit LCD1602_E = P1^5;

unsigned char T0RH = 0;											//Timer0重载值的高字节
unsigned char T0RL = 0;											//Timer0重载值的低字节
bit flagMove = 0;												//定时标志位
unsigned char code str1[] = "Xcopter.cc";						//要显示的字符串1
unsigned char code str2[] = "Xcopter's Blog";					//要显示的字符串2

void ConfigTimer(unsigned int ms);
void Init_LCD1602();
void LCD1602_ShowStr(unsigned char x, unsigned char y, unsigned char *ptrStr, unsigned char len);

void main()
{
	unsigned char i;
	unsigned char index = 0;									//地址移动索引
	unsigned char pdata buffMove1[16 + sizeof(str1) + 16];		//移动显示缓冲区1
	unsigned char pdata buffMove2[16 + sizeof(str2) + 16];		//移动显示缓冲区1

	EA = 1;														//开总中断
	ConfigTimer(1);												//配置Timer0，定时1ms
	Init_LCD1602();												//初始化LCD1602液晶

	/*给buffMove1数组赋值*/
	for (i = 0; i < 16; i++)
	{
		buffMove1[i] = ' ';										//注意单引号之间是1个空格
	}
	for (i = 0; i < (sizeof(str1) - 1); i++)
	{
		buffMove1[i + 16] = str1[i];
	}
	for (i = (16 + (sizeof(str1) - 1)); i < sizeof(buffMove1); i++)
	{
		buffMove1[i] = ' ';
	}
	/*给buffMove2数组赋值*/
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
		if (flagMove)										//定时器标志位置1，定时时间到
		{
			flagMove = 0;
			LCD1602_ShowStr(0, 0, buffMove1 + index, 16);	//buffMove1数组的首地址加索引值，并取16个数据值
			LCD1602_ShowStr(0, 1, buffMove2 + index, 16);	//buffMove2数组的首地址加索引值，并取16个数据值
			//这里的buffMove1和buffMove2是数组指针，等同于&buffMove1[0]和&buffMove2[0]，传递的是地址
			//16来限制在显示字符串时只显示其前 len 个字符，以保证字符串长度的正确性
			index++;										//进入Timer0中断1次，地址索引值加1
			if (index >= (16 + sizeof(str2) - 1))			//起始位置到达字符串尾部后，即返回从头开始	
			{												//这里使用str2，是因为str2要显示的字符比str1多
				index = 0;
			}
		}
	}
}
/*配置并启动Timer0，ms为定时时间*/
void ConfigTimer(unsigned int ms)
{
	unsigned long reload;									//临时变量

	reload = 11059200 / 12;									//定时器计数频率
	reload = (reload * ms) / 1000;							//频率*时间变量，得到这个时间内的计数个数
	reload = 65536 - reload + 12;							//得出定时器初值，即重载值，并补偿中断延时的误差
	T0RH = (unsigned char)(reload >> 8);					//定时重载值拆分为高低字节
	T0RL = (unsigned char)reload;
	TMOD &= 0xF0;											//Timer0控制位清零
	TMOD |= 0x01;											//配置Timer0为模式1
	TH0 = T0RH;												//加载Timer0重载值
	TL0 = T0RL;
	ET0 = 1;												//使能Timer0中断
	TR0 = 1;												//启动Timer0
}
/*LCD1602忙状态检测函数*/
void LCD1602_Ready()
{
	unsigned char sta;

	LCD1602_DB = 0xFF;										//读外部状态，首先自己要先置为高电平
	LCD1602_RS = 0;
	LCD1602_RW = 1;
	do
	{
		LCD1602_E = 1;
		sta = LCD1602_DB;									//读状态字
		LCD1602_E = 0;
	} while (sta & 0x80);									//bit7等于1表示液晶在忙，重复检测直到其等于0为止
}
/*1602写命令函数，cmd为待写入的命令*/
void LCD1602_W_CMD(unsigned char cmd)
{
	LCD1602_Ready();
	LCD1602_RS = 0;
	LCD1602_RW = 0;
	LCD1602_DB = cmd;
	LCD1602_E = 1;
	LCD1602_E = 0;
}
/*1602写数据函数，dat为待写入的数据*/
void LCD1602_W_DATA(unsigned char dat)
{
	LCD1602_Ready();
	LCD1602_RS = 1;
	LCD1602_RW = 0;
	LCD1602_DB = dat;
	LCD1602_E = 1;
	LCD1602_E = 0;
}
/*设置显示RAM的其实地址，即光标位置*/
void LCD1602_SetCursor(unsigned char x, unsigned char y)
{
	unsigned char addr;

	if (y)													//由输入的屏幕坐标计算显示RAM的地址
	{
		addr = 0x40 + x;									//第二行字符地址从0x40开始
	}
	else
	{
		addr = 0x00 + x;									//第一行字符地址从0x00开始
	}
	LCD1602_W_CMD(addr | 0x80);								//写命令设置RAM地址
}
/*1602显示字符函数，ptrStr为字符串指针*/
void LCD1602_ShowStr(unsigned char x, unsigned char y, unsigned char *ptrStr, unsigned char len)
{
	LCD1602_SetCursor(x, y);								//设置显示字符串的起始地址

	while (len--)											//连续写入len个字符数据
	{
		LCD1602_W_DATA(*ptrStr++);							//先取ptrStr指向的数据，然后ptrStr自加1
	}
}
/*1602液晶初始化函数*/
void Init_LCD1602()
{
	LCD1602_W_CMD(0x38);									//16 * 2显示，5 * 7点阵，8位数据接口
	LCD1602_W_CMD(0x0C);									//显示器开，光标关闭
	LCD1602_W_CMD(0x06);									//文字不动，地址自动加1
	LCD1602_W_CMD(0x01);									//清屏
}
/*Timer0中断服务函数*/
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