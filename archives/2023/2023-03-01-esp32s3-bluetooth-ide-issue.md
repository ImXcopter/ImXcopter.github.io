# ESP32-S3-DevKitC-1 开发板在不同版本的 IDE 做蓝牙开发遇到的问题

截至到发文前，Arduino IDE 已经更新到 2.0.4，1.8.19 为老版本 IDE 的最后一个版本。

前些天使用这块 ESP32-S3-DevKitC-1 开发板在 Arduino 框架下使用 ESP32 BLE Mouse 库做蓝牙鼠标的开发遇到了一些问题。

**测试环境：**

1. ESP32-S3-DevKitC-1 开发板
2. ESP32 2.0.6 资源包和 ESP32 2.0.7 资源包
3. Arduino IDE 1.8.19 和 Arduino IDE 2.0.3
4. ESP32 BLE Mouse v0.3.1 库（https://github.com/T-vK/ESP32-BLE-Mouse）
5. VScode

![测试环境](/static/2023/2023-03-01-esp32s3-bluetooth-ide-issue_001.jpg)

**Arduino 程序如下：**

```cpp
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLEMouse.h>

BleMouse bleMouse("Xcopter", "Xcopter", 100);

void setup() {
    Serial.begin(115200);
    bleMouse.begin();
}

void loop() {

}
```

本程序皆在测试是否可在 PC 电脑的蓝牙设备管理里正常识别到名为 "Xcopter" 的蓝牙鼠标设备。

不同的 IDE 不同版本的 ESP32 资源包下，呈现的问题不同，具体为：

1. **在 ESP32 2.0.6 版本的资源包下**：
   - Arduino IDE 1.8.19 可以正常编译并上传，同时可以识别到蓝牙鼠标设备
   - Arduino IDE 2.0.3 可以正常编译并上传，但无法识别到蓝牙鼠标设备

2. **在 ESP32 2.0.7 版本的资源包下**：
   - Arduino IDE 1.8.19 可以正常编译但无法上传，提示上传错误
   - Arduino IDE 2.0.3 可以正常编译并上传，但无法识别到蓝牙鼠标设备

另外：VScode 在 Arduino IDE 2.0.3 版本下使用似乎有一些问题，总是提示找不到头文件，在 1.8.19 版本可正常编译。截至目前，我只能在 VScode 下使用 Arduino IDE 1.8.19 版本作为编译器。

查了一些资料，可能是在 Arduino IDE 2.x 版本后，需要在 VScode 配置 Arduino CLI 才能正常使用。
