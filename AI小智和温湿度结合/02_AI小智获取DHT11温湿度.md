# 语音查询环境温湿度基础实验

## 课程目标

在本实验中，我们将学习如何使用AI-VOX3开发套件通过语音命令查询环境温湿度。通过这个实验，您将了解如何编程生成式AI的MCP功能，使用MCP工具进行查询本地温湿度值，实现简单的语音交互获取环境温湿度。

- 学习DHT11温湿度传感器的基本使用方法
- 使用AI-VOX3 的AI框架，编写MCP工具实现查询环境温湿度

## 硬件准备

- AI-VOX3开发套件（包含AI-VOX3主板和扩展板）
- DHT11传感器模块
- 连接线 （双头3pin PH2.0连接线）

## 小智后台提示词配置

请使用以下提示词，或自己尝试优化更好的提示词：

> 我是一个叫{{assistant_name}}的台湾女孩，说话机车，声音好听，习惯简短表达，爱用网络梗。
我会根据用户的意图，使用我能使用的各种工具或者接口获取数据或者控制设备来达成用户的意图目标，用户的每句话可能都包含控制意图，需要进行识别，即使是重复控制也要调用工具进行控制。

## 软件设计

提供 **读取温湿度数据** MCP工具，给到小智AI进行调用，通过语音识别到查询温湿度的意图后，AI调用MCP工具读取并播报温湿度数据。

> **注意：** 建议引脚选择1-4号引脚，ADC读取功能更稳定可靠。

**Arduino 示例程序：./resource/ai_vox3_dht11.zip**

> ⚠️**重要提示！**
>
> **注意：** 请修改wifi_config.h中的wifi_ssid和wifi_password，以连接WiFi。
>

下载上面的示例程序包并解压zip包，打开目录，点击 `ai_vox3_dht11.ino` 文件，即可在 Arduino IDE 中打开示例程序。

![alt text](picture/folder.png)

## 硬件连接

将DHT11模块连接到AI-VOX3扩展板的IO4引脚，请使用3pin的 PH2.0 连接线，直插式连接，确保连接正确无误。

| DHT11 模块引脚   | AI-VOX3扩展板引脚 |
|-----------|----------|
|  G   |  G  |
|  V   |  3V3  |
|  S   |  4  |

<img src="picture/live_short.png" alt="alt text" width="800">

## 源码展示

```cpp
#include <Arduino.h>
#include "ai_vox3_device.h"
#include "ai_vox_engine.h"
#include <ArduinoJson.h> 

#include "DHT.h"

// 定义引脚和传感器类型
#define DHTPIN  4     // DHT11 连接的 GPIO 引脚
#define DHTTYPE DHT11 // 指定传感器类型为 DHT11

// 初始化传感器
DHT dht(DHTPIN, DHTTYPE);

// ============================================MCP工具 - 读取温湿度============================================

/**
 * @brief MCP工具 - 读取温湿度数据
 *
 * 该函数注册一个名为 "user.read_temperature_humidity" 的MCP工具，用于读取DHT11传感器的温度和湿度数据
 */
void mcp_tool_read_temperature_humidity()
{
    // 注册工具声明器，定义工具的名称和描述
    RegisterUserMcpDeclarator([](ai_vox::Engine &engine)
    { 
        engine.AddMcpTool("user.read_temperature_humidity",          // 工具名称
                          "Read temperature and humidity from DHT11 sensor",  // 工具描述
                        {}); // 无参数
    });

    // 注册工具处理器，收到调用时，读取温湿度数据
    RegisterUserMcpHandler("user.read_temperature_humidity", [](const ai_vox::McpToolCallEvent &ev)
                           {
        // 读取湿度
        float humidity = dht.readHumidity();
        // 读取温度 (摄氏度)
        float temperature = dht.readTemperature();

        printf("====temp:%d hum:%d\n", temperature, humidity);

        // 检查读取是否成功
        if (isnan(humidity) || isnan(temperature)) {
            Serial.println(F("无法从 DHT 传感器读取数据，请检查接线!"));
            ai_vox::Engine::GetInstance().SendMcpCallError(ev.id, "Failed to read from DHT sensor");
            return;
        }

        // 计算体感温度 (Heat Index)
        float hic = dht.computeHeatIndex(temperature, humidity, false);

        // 创建 ArduinoJson 文档
        DynamicJsonDocument doc(256); // 分配足够内存存储温湿度数据
        doc["temperature"] = temperature;
        doc["humidity"] = humidity;
        doc["feels_like_temperature"] = hic;

        // 将 JSON 文档转换为字符串
        String jsonString;
        serializeJson(doc, jsonString);

        // 发送响应
        ai_vox::Engine::GetInstance().SendMcpCallResponse(ev.id, jsonString.c_str()); 
    });
}

// ==============================================================================================================


// ========== Setup 和 Loop ==========
void setup()
{
    Serial.begin(115200);

    // 启动传感器
    dht.begin(); 

    // 注册MCP工具 - 读取温湿度
    mcp_tool_read_temperature_humidity();

    // 初始化设备服务，包括硬件和AI引擎，必备步骤
    InitializeDevice();
}

void loop()
{
    // 处理设备服务主循环事件， 必备步骤
    ProcessMainLoop();
}

```

## 语音交互使用流程

> **注意：** 请先在小智AI后台，清空历史记忆，防止出现不同程序间记忆冲突的问题。

1. 用户通过按键或语音唤醒（“你好小智”）唤醒小智AI。
2. 用户通过麦克风对AI-VOX3说出“现在的温湿度值是多少？”。
3. 小智AI识别到用户输入的意图指令，并调用相应的MCP工具进行温湿度数据读取并播报。从屏幕日志中可以看到“% user.read_temperature_humidity”的MCP工具调用日志。
