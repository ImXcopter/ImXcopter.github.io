# 51单片机 - 1602液晶显示字符串（指针方式）

![](/static/2023/2023-04-10-51mcu-lcd1602-string-pointer_001.png)

```
#include 
#define LCD1602_DB P0

sbit LCD1602_RS = P1^0;
sbit LCD1602_RW = P1^1;
sbit LCD1602_E = P1^5;

void Init_LCD1602();
void LCD1602_ShowStr(unsigned char x, unsigned char y, unsigned char *ptrStr);

void main()
{
	unsigned char Str[] = {"Xcopter.cc"};

	Init_LCD1602();
	LCD1602_ShowStr(0, 0, Str);	//这里的Str是指针，等同于&Str[0]。因为在函数声明时，这个形参定义的就是个指针变量
	LCD1602_ShowStr(0, 1, "Xcopter's Blog");
	while (1);
}
/*1602忙状态检测函数*/
void LCD1602_Ready()
{
	unsigned char sta;

	LCD1602_DB = 0xFF;
	LCD1602_RS = 0;
	LCD1602_RW = 1;
	do
	{
		LCD1602_E = 1;
		sta = LCD1602_DB;
		LCD1602_E = 0;
	} while (sta & 0x80);
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

	if (y)
	{
		addr = 0x40 + x;
	}
	else
	{
		addr = 0x00 + x;
	}
	LCD1602_W_CMD(addr | 0x80);
}
/*1602显示字符函数，ptrStr为字符串指针*/
void LCD1602_ShowStr(unsigned char x, unsigned char y, unsigned char *ptrStr)
//注意这里的*ptrStr是定义这个ptrStr是个指针变量
{
	LCD1602_SetCursor(x, y);

	while (*ptrStr != '\0')
	{
		LCD1602_W_DATA(*ptrStr++);
	}
}
/*1602液晶初始化函数*/
void Init_LCD1602()
{
	LCD1602_W_CMD(0x38);
	LCD1602_W_CMD(0x0C);
	LCD1602_W_CMD(0x06);
	LCD1602_W_CMD(0x01);
}
```