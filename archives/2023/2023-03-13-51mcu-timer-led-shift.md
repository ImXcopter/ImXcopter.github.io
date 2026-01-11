# 51单片机 - 使用定时器来实现左右移动的流水灯程序

本程序代码为《手把手教你学51单片机》5.4的课后练习题3，并且已经在KST-51 v1.3.2开发板验证通过。

```text
#include <reg52.h>

sbit ADDR0 = P1^0;
sbit ADDR1 = P1^1;
sbit ADDR2 = P1^2;
sbit ADDR3 = P1^3;
sbit ENLED = P1^4;

unsigned int cnt = 0;           //记录进入Timer0中断次数
bit flagTF = 0;                 //定义一个bit形的变量，为定时标志位
unsigned char shift = 0x01;     //定义循环移位变量shift，并赋初值0x01，这里不能赋值为0！
bit dir = 0;                    //定义一个bit形的移位方向变量dir，用于控制移位的方向

void main(){

    EA = 1;         //使能总中断
    ENLED = 0;
    ADDR0 = 0;
    ADDR1 = 1;
    ADDR2 = 1;
    ADDR3 = 1;
    TMOD &= 0xF0;   //设置Timer0为工作模式1
    TMOD |= 0x01;   //设置Timer0为工作模式1
    TH0 = 0xFC;     //为Timer0赋初值，定时1ms
    TL0 = 0x67;
    ET0 = 1;        //使能Timer0中断
    TR0 = 1;        //启动Timer0

    while(1){
        P0 = ~shift;                    //P0等于循环移位变量取反，控制8个LED

        if(flagTF == 1){
            flagTF = 0;
            if(dir == 0){               //移位方向变量为0时，左移
                shift = shift << 1;     //循环移位变量左移1位
                if(shift == 0x80){      //左移到最左端后，改变移位方向
                    dir = 1;
                }
            }
            else{                       //移位方向变量不为0时，右移
                shift = shift >> 1;     //循环移位变量右移1位
                if(shift == 0x01){      //右移到最右端后，改变移位方向
                    dir = 0;
                }
            }
        }
    }
}

void interruptTimer0() interrupt 1{
    TH0 = 0xFC;                         //每进入中断，为Timer0赋初值，定时1ms
    TL0 = 0x67;
    cnt++;                              //每进入1次中断，中断次数加1
    if(cnt >= 60){                      //当进入了60次中断，即计时了60ms时
        cnt = 0;                        //记录进入中断次数清零
        flagTF = 1;                     //定时标志位赋1
    }
}
```
