# 51单片机 - LED灯显示二进制代码

KST-51的开发板P0口只能每次只能控制一个外设，我们这次要控制6个数码管和1组（8个）LED小灯，由于数码管和LED小灯都属于显示设备，所以我们可以用动态刷新的办法来"同时"点亮数码管和8个LED小灯。

本程序实现的效果：数码管从0加到255，相应的LED小灯以二进制的状态显示当前的数值。由于LED小灯只有8个，也就相当于可以表达8位的二进制数值，即1个字节的二进制数值状态。所以我们的程序是从十进制数0开始，每秒加1个数，直到加到255。

本程序使用定时器1中断来刷新数码管和LED小灯，并精确计时。代码已经在KST-51 v1.3.2开发板验证通过。

![LED二进制显示效果](/static/2023/2023-03-14-51mcu-led-binary-display_001.jpg)

```text
#include <reg52.h>

sbit ADDR0 = P1^0;
sbit ADDR1 = P1^1;
sbit ADDR2 = P1^2;
sbit ADDR3 = P1^3;
sbit ENLED = P1^4;

//数码管真值表
unsigned char code LedChar[16] = {
    0xC0, 0xF9, 0xA4, 0xB0, 0x99, 0x92, 0x82, 0xF8,
    0x80, 0x90, 0x88, 0x83, 0xC6, 0xA1, 0x86, 0x8E
};

//前六个为数码管显示缓冲区，最后1个为8个LED小灯的初始值，初值0xFF确保启动时都不亮
unsigned char LedBuff[7] = {
    0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF
};
unsigned char flag1s = 0;           //1秒定时标志
unsigned int cnt = 0;               //记录Timer1中断次数
unsigned char i = 0;                //动态扫描的索引

void main(){
    unsigned long sec = 0;          //计秒初始值
    unsigned char buff[6];          //中间转换缓冲区
    signed char j = 0;
    unsigned char tempLedBuff = 0;  //8个LED小灯转换变量

    EA = 1;                         //使能总中断
    ENLED = 0;                      //使能U3
    ADDR3 = 1;                      //因为需要动态改变ADDR0-2的值，所以ADDR0-2不需要再初始化了
    TMOD &= 0x0F;                   //设置Timer1工作模式1
    TMOD |= 0x10;
    TH1 = 0xFC;                     //为Timer1赋初值0xFC67，定时1ms
    TL1 = 0x67;
    ET1 = 1;                        //使能Timer1中断
    TR1 = 1;                        //启动Timer1

    while(1){
        if(flag1s){                 //判断1秒定时标志
            flag1s = 0;             //1秒定时标志清零
            if(sec < 255){          //如果秒数小于255
                sec++;              //秒计数自加1
                tempLedBuff++;      //LED小灯当前值自加1
            }
            else{
                sec = 0;            //秒数清零
                tempLedBuff = 0;    //LED小灯值清零
            }
        }
        //LED小灯当前值取反，赋给LedBuff[6]，待每1ms进定时器中断刷新显示出来。取反的原因在于每个LED小灯低电平点亮。
        LedBuff[6] = ~tempLedBuff;
        //将sec按十进制位从低到高依次提取到buff数组中
        buff[0] = sec % 10;
        buff[1] = sec / 10 % 10;
        buff[2] = sec / 100 % 10;
        buff[3] = sec / 1000 % 10;
        buff[4] = sec / 10000 % 10;
        buff[5] = sec / 100000 % 10;
        //从最高为开始，遇到0不显示(赋值0xFF)，遇到非0退出for循环
        for(j = 5; j >= 1; j--){
            if(buff[j] == 0){
                LedBuff[j] = 0xFF;
            }
            else{
                break;
            }
        }
        //将剩余的有效数字位如实转换，for()起始未对j操作，j即保持上个循环结束时的值
        for( ; j >= 0; j--){
            LedBuff[j] = LedChar[buff[j]];
        }
    }
}

/* 定时器1中断服务函数 */
void interruptTimer1() interrupt 3{
    TH1 = 0xFC;             //重新加载初值
    TL1 = 0x67;
    cnt++;                  //中断次数计数值加1
    if(cnt >= 1000){        //中断1000次即1秒
        cnt = 0;            //清零计数值以重新开始下1秒计时
        flag1s = 1;         //设置1秒定时标志为1
    }
    //以下代码完成数码管和LED小灯的动态扫描刷新
    P0 = 0xFF;              //显示消隐
    switch(i){
        case 0: ADDR2 = 0; ADDR1 = 0; ADDR0 = 0; i++; P0 = LedBuff[0]; break;
        case 1: ADDR2 = 0; ADDR1 = 0; ADDR0 = 1; i++; P0 = LedBuff[1]; break;
        case 2: ADDR2 = 0; ADDR1 = 1; ADDR0 = 0; i++; P0 = LedBuff[2]; break;
        case 3: ADDR2 = 0; ADDR1 = 1; ADDR0 = 1; i++; P0 = LedBuff[3]; break;
        case 4: ADDR2 = 1; ADDR1 = 0; ADDR0 = 0; i++; P0 = LedBuff[4]; break;
        case 5: ADDR2 = 1; ADDR1 = 0; ADDR0 = 1; i++; P0 = LedBuff[5]; break;
        case 6: ADDR2 = 1; ADDR1 = 1; ADDR0 = 0; i = 0; P0 = LedBuff[6]; break;
        default: break;
    }
}
```
