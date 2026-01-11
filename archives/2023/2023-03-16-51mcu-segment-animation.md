# 51单片机 - 数码管图片帧刷新coding训练

效果展示：**[视频演示](https://xcopter.cc/wp-content/uploads/2023/03/20230316-138.mp4)** (由于摄像头拍摄帧率的缘故，正好可以在视频中看到逐行刷新的效果)

本代码已经在 KST-51 v1.3.2 开发板验证通过，代码如下：

```text
#include <reg52.h>

sbit ADDR0 = P1^0;
sbit ADDR1 = P1^1;
sbit ADDR2 = P1^2;
sbit ADDR3 = P1^3;
sbit ENLED = P1^4;
//图片帧
unsigned char code images[6][8] = {
    {0x00,0x7E,0x7E,0x7E,0x7E,0x7E,0x7E,0x00},  //8x8点阵从外圈数第1圈显示的图片帧
    {0xFF,0x81,0xBD,0xBD,0xBD,0xBD,0x81,0xFF},  //8x8点阵从外圈数第2圈显示的图片帧
    {0xFF,0xFF,0xC3,0xDB,0xDB,0xC3,0xFF,0xFF},  //8x8点阵从外圈数第3圈显示的图片帧
    {0xFF,0xFF,0xFF,0xE7,0xE7,0xFF,0xFF,0xFF},  //8x8点阵从外圈数第4圈显示的图片帧
    {0xFF,0xFF,0xFF,0xFF,0xFF,0xFF,0xFF,0xFF},  //8x8点阵全部不显示的图片帧
    {0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00}   //8x8点阵全部显示的图片帧
};
bit flagcnt50ms = 0;                            //50ms的标志位变量
bit flagcnt3s = 0;                              //3s的标志位变量
unsigned char index = 0;                        //图片帧索引变量

void main() {
    bit dir = 1;                                //图片帧播放方向变量
    unsigned char cyclecnt = 0;                 //图片帧播放次数变量

    EA = 1;                                     //使能总中断
    ENLED = 0;                                  //使能U4，选择LED点阵
    ADDR3 = 0;
    TMOD &= 0xF0;                               //设置Timer0为模式1
    TMOD |= 0x01;
    TH0 = 0xFC;                                 //为Timer0赋初值0xFC67，定义1ms
    TL0 = 0x67;
    ET0 = 1;                                    //使能Timer0中断
    TR0 = 1;                                    //启动Timer0

    while(1){
        if(flagcnt50ms){                        //如果50ms标志位置1，即50ms到
            flagcnt50ms = 0;                    //50ms标志位清零
            if(dir){                            //如果dir = 1
                index++;                        //图片帧播放索引每次加1，图片帧正向播放
                if(index >= 4){                 //图片帧播放到第4帧
                    dir = 0;                    //dir置0
                }
            }
            else{
                index--;                        //如果dir = 0，图片帧索引每次减1，图片反向播放
                if(index <= 0){                 //图片索引播放到第0帧
                    dir = 1;                    //dir置1
                    cyclecnt++;                 //循环播放变量加1，也就是说正向播放后又反向播放了1次后加1。
                }
            }
        }
        if(cyclecnt >= 5){                      //如果图片帧循环播放了5次
            index = 5;                          //图片播放第5帧，即LED点阵全部点亮
            if(flagcnt3s){                      //当前帧播放3s时长到
                flagcnt3s = 0;                  //3s标志位置0
                index = 0;                      //图片帧播放索引置0
                cyclecnt = 0;                   //图片帧循环播放次数置0
                dir = 1;                        //播放方向从正向开始
            }
        }
    }
}

void interruptTimer0() interrupt 1 {
    static unsigned int i = 0;
    static unsigned int cnt1ms_1 = 0;
    static unsigned int cnt1ms_2 = 0;

    TH0 = 0xFC;                                 //重新加载Timer0初值
    TL0 = 0x67;
    cnt1ms_1++;                                 //中断次数计数值加1

    if(cnt1ms_1 >= 50){                         //中断进入50次，即计时50ms
        cnt1ms_1 = 0;                           //中断进入次数置零
        flagcnt50ms = 1;                        //50ms标志位置1
    }
    if(index == 5){                             //图片刷新最后1帧时
        cnt1ms_2++;                             //中断进入次数开始每次加1
        if(cnt1ms_2 >= 3000){                   //中断进入3000次，即计时3s
            cnt1ms_2 = 0;                       //中断进入次数置零
            flagcnt3s = 1;                      //3s标志位置1
        }
    }
    //以下代码完成LED点阵的动态扫描刷新
    P0 = 0xFF;
    switch(i){
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

另一种方法：也可将图片帧播放过程放进中断：

```text
#include <reg52.h>

sbit ADDR0 = P1^0;
sbit ADDR1 = P1^1;
sbit ADDR2 = P1^2;
sbit ADDR3 = P1^3;
sbit ENLED = P1^4;
//图片帧
unsigned char code images[6][8] = {
    {0x00,0x7E,0x7E,0x7E,0x7E,0x7E,0x7E,0x00},  //8x8点阵从外圈数第1圈显示的图片帧
    {0xFF,0x81,0xBD,0xBD,0xBD,0xBD,0x81,0xFF},  //8x8点阵从外圈数第2圈显示的图片帧
    {0xFF,0xFF,0xC3,0xDB,0xDB,0xC3,0xFF,0xFF},  //8x8点阵从外圈数第3圈显示的图片帧
    {0xFF,0xFF,0xFF,0xE7,0xE7,0xFF,0xFF,0xFF},  //8x8点阵从外圈数第4圈显示的图片帧
    {0xFF,0xFF,0xFF,0xFF,0xFF,0xFF,0xFF,0xFF},  //8x8点阵全部不显示的图片帧
    {0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00}   //8x8点阵全部显示的图片帧
};

void main() {
    EA = 1;                                     //使能总中断
    ENLED = 0;                                  //使能U4，选择LED点阵
    ADDR3 = 0;
    TMOD &= 0xF0;                               //设置Timer0为模式1
    TMOD |= 0x01;
    TH0 = 0xFC;                                 //为Timer0赋初值0xFC67，定义1ms
    TL0 = 0x67;
    ET0 = 1;                                    //使能Timer0中断
    TR0 = 1;                                    //启动Timer0

    while(1){

    }
}

void interruptTimer0() interrupt 1 {
    static unsigned int i = 0;
    static unsigned char index = 0;             //图片帧索引变量
    static unsigned int cnt1ms_1 = 0;
    static unsigned int cnt1ms_2 = 0;
    static bit dir = 1;                         //图片帧播放方向变量
    static unsigned char cyclecnt = 0;          //图片帧播放次数变量
    static bit flagcnt50ms = 0;                 //50ms的标志位变量
    static bit flagcnt3s = 0;                   //3s的标志位变量

    TH0 = 0xFC;                                 //重新加载Timer0初值
    TL0 = 0x67;
    cnt1ms_1++;                                 //中断次数计数值加1

    if(cnt1ms_1 >= 50){                         //中断进入50次，即计时50ms
        cnt1ms_1 = 0;                           //中断进入次数置零
        flagcnt50ms = 1;                        //50ms标志位置1
    }
    if(index == 5){                             //图片刷新最后1帧时
        cnt1ms_2++;                             //中断进入次数开始每次加1
        if(cnt1ms_2 >= 3000){                   //中断进入3000次，即计时3s
            cnt1ms_2 = 0;                       //中断进入次数置零
            flagcnt3s = 1;                      //3s标志位置1
        }
    }

    if(flagcnt50ms){                            //如果50ms标志位置1，即50ms到
        flagcnt50ms = 0;                        //50ms标志位清零
        if(dir){                                //如果dir = 1
            index++;                            //图片帧播放索引每次加1，图片帧正向播放
            if(index >= 4){                     //图片帧播放到第4帧
                dir = 0;                        //dir置0
            }
        }
        else{
            index--;                            //如果dir = 0，图片帧索引每次减1，图片反向播放
            if(index <= 0){                     //图片索引播放到第0帧
                dir = 1;                        //dir置1
                cyclecnt++;                     //循环播放变量加1，也就是说正向播放后又反向播放了1次后加1。
            }
        }
    }
    if(cyclecnt >= 5){                          //如果图片帧循环播放了5次
        index = 5;                              //图片播放第5帧，即LED点阵全部点亮
        if(flagcnt3s){                          //当前帧播放3s时长到
            flagcnt3s = 0;                      //3s标志位置0
            index = 0;                          //图片帧播放索引置0
            cyclecnt = 0;                       //图片帧循环播放次数置0
            dir = 1;                            //播放方向从正向开始
        }
    }
    //以下代码完成LED点阵的动态扫描刷新
    P0 = 0xFF;
    switch(i){
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
