# ESP32-S3-DevKitC-1开发板在不同版本的IDE做蓝牙开发遇到的问题

截至到发文前，Arduino IDE已经更新到2.0.4，1.8.19为老版本IDE的最后一个版本。前些天使用这块ESP32-S3-DevKitC-1开发板在Arduino框架下使用ESP32 BLE Mouse库做蓝牙鼠标的开发遇到了一些问题。

## 测试环境

- ESP32-S3-DevKitC-1开发板
- ESP32 2.0.6资源包和ESP32 2.0.7资源包
- Arduino IDE 1.8.19和Arduino IDE 2.0.3
- ESP32 BLE Mouse v0.3.1库（https://github.com/T-vK/ESP32-BLE-Mouse）
- VScode

![ESP32-S3开发板](/static/2023/2023-03-01-esp32s3-bluetooth-ide-issue_001.jpg)

## Arduino程序

本程序皆在测试是否可在PC电脑的蓝牙设备管理里正常识别到名为"Xcopter"的蓝牙鼠标设备。

```text
#include <BleMouse.h>
#include <BleKeyboard.h>
#include <BleGamepad.h>
#include <BleConnectionStatus.h>

BleMouse bleMouse("Xcopter", "Xcopter", 100);

void setup(){
    Serial.begin(115200);
    bleMouse.begin();
}

void loop(){

}
```

## 问题分析

不同的IDE不同版本的ESP32资源包下，呈现的问题不同，具体为：

- 在ESP32 2.0.6版本的资源包下：Arduino IDE 1.8.19可以正常编译并上传，同时可以识别到蓝牙鼠标设备；在Arduino IDE 2.0.3可以正常编译并上传，但无法识别到蓝牙鼠标设备。
- 在ESP32 2.0.7版本的资源包下：Arduino IDE 1.8.19可以正常编译但无法上传，提示上传错误；在Arduino IDE 2.0.3可以正常编译并上传，但无法识别到蓝牙鼠标设备。

另外：VScode在Arduino IDE 2.0.3版本下使用似乎有一些问题，总是提示找不到头文件，在1.8.19版本可正常编译。截至目前，我只能在VScode下使用Arduino IDE 1.8.19版本作为编译器。查了一些资料，可能是在Arduino IDE 2.x版本后，需要在VScode配置Arduino CLI才能正常使用。
