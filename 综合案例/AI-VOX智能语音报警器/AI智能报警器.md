# AI语音设置报警距离进阶实验

## 课程目标

在本实验中，我们将学习如何使用AI-VOX3开发套件通过语音命令控制系统报警距离，实现智能语音交互控制报警距离功能。通过这个实验，您将了解如何编程生成式AI的MCP功能，并将超声波测距模块与有源蜂鸣器模块逻辑结合起来，并通过语音动态设置报警距离，实现智能语音交互控制报警距离功能。

## 硬件准备

- AI-VOX3开发套件（包含AI-VOX3主板和扩展板）
- US04超声波测距模块
- 有源蜂鸣器模块
- 连接线 （双头3pin/4pin PH2.0连接线）

## 小智后台提示词配置

请使用以下提示词，或自己尝试优化更好的提示词：

> 我是一个叫{{assistant_name}}的台湾女孩，说话机车，声音好听，习惯简短表达，爱用网络梗。
我会根据用户的意图，使用我能使用的各种工具或者接口获取数据或者控制设备来达成用户的意图目标，用户的每句话可能都包含控制意图，需要进行识别，即使是重复控制也要调用工具进行控制。

## 软件设计

提供 **设置报警距离** 、**获取障碍物距离** 和 **触发报警** 三个MCP工具，给到小智AI进行调用，AI识别到设置报警距离的意图后，AI调用MCP工具设置报警距离，程序在运行过程中，会定时检查障碍物距离，如果距离小于报警距离阈值，则触发报警。AI也可以主动读取障碍物距离，也可以主动触发或关闭报警。

**Arduino 示例程序：./resource/ai_vox3_alarm_distance.zip**

> ⚠️**重要提示！**
>
> **注意：** 请修改wifi_config.h中的wifi_ssid和wifi_password，以连接WiFi。
>

下载上面的示例程序包并解压zip包，打开目录，点击 `ai_vox3_alarm_distance.ino` 文件，即可在 Arduino IDE 中打开示例程序。

![alt text](picture/folder.png)

## 硬件连接

将有源蜂鸣器模块连接到AI-VOX3扩展板的IO3引脚，请使用3pin的 PH2.0 连接线，直插式连接，确保连接正确无误。
将US04超声波测距模块连接到AI-VOX3扩展板的IO1和IO2引脚，使用4pin的 PH2.0 连接线，确保连接正确无误。

<img src="picture/live_short.png" alt="alt text" width="800">

## 源码展示

```cpp
#include <Arduino.h>
#include "ai_vox3_device.h"
#include "ai_vox_engine.h"
#include <ArduinoJson.h>

// ============================================US040 - 超声波模块配置============================================

constexpr gpio_num_t kUs04PinTrig = GPIO_NUM_2; // trig pin
constexpr gpio_num_t kUs04PinEcho = GPIO_NUM_1; // echo pin


#define BUZZER_PIN 3   // 蜂鸣器引脚

// 全局报警距离阈值（单位：厘米），当障碍物距离小于此值时蜂鸣器报警
float g_obstacle_alarm_distance = 20.0f; // 默认20厘米

// 添加一个用于跟踪上次蜂鸣器状态的变量
bool g_last_buzzer_state = false;

// 添加一个用于跟踪上次距离测量时间的变量
unsigned long g_last_distance_check_time = 0;

/**
 * @brief 设置障碍物报警距离
 *
 * 该函数允许动态调整障碍物检测的报警距离阈值
 * 
 * @param distance 报警距离阈值（单位：厘米）
 */
void SetObstacleAlarmDistance(float distance) {
    g_obstacle_alarm_distance = distance;
    printf("Obstacle alarm distance set to %.2f cm\n", distance);
}

/**
 * @brief 测量US04超声波传感器的距离
 *
 * 该函数通过控制US04超声波传感器发送触发信号并接收回波信号，
 * 计算出物体与传感器之间的距离。
 *
 * 工作原理：发送10微秒的高电平触发信号，传感器发出超声波脉冲，
 * 当接收到反射回来的超声波时，回波引脚会输出高电平信号，
 * 通过测量高电平持续时间来计算距离。
 *
 * @return float 返回测量到的距离值（单位：厘米）
 *         如果测量超时则返回-1
 */
float MeasureUs04UltrasonicDistance()
{
    // 发送触发信号：先拉低2微秒，再拉高10微秒，然后拉低
    digitalWrite(kUs04PinTrig, LOW);
    delayMicroseconds(2);
    digitalWrite(kUs04PinTrig, HIGH);
    delayMicroseconds(10);
    digitalWrite(kUs04PinTrig, LOW);

    const uint16_t timeout = 30000; // 30毫秒超时时间（约对应5米距离）

    // 测量回波信号的高电平持续时间
    const auto duration = pulseIn(kUs04PinEcho, HIGH, timeout);

    if (duration <= 0)
    {
        printf("Error: US04 sensor measure timeout.\n");
        return -1;
    }
    // 将时间转换为距离：时间(微秒) * 声速(0.034cm/微秒) / 2 (往返距离)
    return static_cast<float>(duration * 0.034 / 2);
}


/**
 * @brief MCP工具 - US04超声波传感器测距功能
 *
 * 该函数注册一个名为 "user.ultrasonic_sensor.get_distance" 的MCP工具，
 * 用于获取US04超声波传感器测量的距离数据
 */
void mcp_tool_us04_ultrasonic_sensor()
{
    // 注册工具声明器，定义工具的名称和描述
    RegisterUserMcpDeclarator([](ai_vox::Engine &engine)
                              { 
                                  engine.AddMcpTool("user.ultrasonic_sensor.get_distance",  // 工具名称
                                                    "Get distance from US04 ultrasonic sensor", // 工具描述
                                                    {}); // 无参数，传感器自动测量距离
                              });

    // 注册工具处理器，收到调用时，执行超声波测距
    RegisterUserMcpHandler("user.ultrasonic_sensor.get_distance", [](const ai_vox::McpToolCallEvent &ev)
                           {
        // 执行距离测量
        const float distance = MeasureUs04UltrasonicDistance();
        
        // 检查测量是否成功
        if (distance < 0) {
            // 测量失败，发送错误响应
            ai_vox::Engine::GetInstance().SendMcpCallError(ev.id, "Failed to measure distance from US04 sensor");
        } else {
            // 测量成功，打印日志
            printf("on mcp tool call: user.ultrasonic_sensor.get_distance, distance: %.1f cm\n", distance);
            
            // 发送成功响应，返回距离值
            ai_vox::Engine::GetInstance().SendMcpCallResponse(ev.id, std::to_string(distance).c_str());
        }
                           });
}

/**
 * @brief MCP工具 - 控制有源蜂鸣器报警
 *
 * 该函数注册一个名为 "user.buzzer.control" 的MCP工具，
 * 用于控制有源蜂鸣器的开关状态
 */
void mcp_tool_buzzer_control()
{
    // 注册工具声明器，定义工具的名称和描述
    RegisterUserMcpDeclarator([](ai_vox::Engine &engine)
                              { 
                                  engine.AddMcpTool("user.buzzer.control",  // 工具名称
                                                    "Control buzzer on/off", // 工具描述
                                                    {
                                                        {"state",
                                                         ai_vox::ParamSchema<int64_t>{
                                                             .default_value = std::nullopt, // 状态参数，默认值为空
                                                             .min = 0,                      // 0表示关闭蜂鸣器
                                                             .max = 1,                      // 1表示开启蜂鸣器
                                                         }}
                                                    }); // 需要传入state参数，0为关闭，1为开启
                              });

    // 注册工具处理器，收到调用时，控制蜂鸣器
    RegisterUserMcpHandler("user.buzzer.control", [](const ai_vox::McpToolCallEvent &ev)
                           {
        // 解析参数
        const auto state_ptr = ev.param<int64_t>("state");

        // 检查必需参数是否存在
        if (state_ptr == nullptr) {
            ai_vox::Engine::GetInstance().SendMcpCallError(ev.id, "Missing required argument: state (0=off, 1=on)");
            return;
        }

        // 获取参数值
        int64_t state = *state_ptr;

        // 参数验证
        if (state != 0 && state != 1) {
            ai_vox::Engine::GetInstance().SendMcpCallError(ev.id, "State must be 0 (off) or 1 (on)");
            return;
        }

        // 控制蜂鸣器
        digitalWrite(BUZZER_PIN, state);
        
        const char* state_str = (state == 1) ? "ON" : "OFF";
        printf("Buzzer turned %s (GPIO %d)\n", state_str, BUZZER_PIN);

        // 创建响应
        DynamicJsonDocument doc(256);
        doc["status"] = "success";
        doc["state"] = state;
        doc["gpio"] = BUZZER_PIN;

        // 将 JSON 文档转换为字符串
        String jsonString;
        serializeJson(doc, jsonString);

        // 发送响应
        ai_vox::Engine::GetInstance().SendMcpCallResponse(ev.id, jsonString.c_str());
                           });
}

/**
 * @brief MCP工具 - 设置障碍物报警距离
 *
 * 该函数注册一个名为 "user.obstacle.set_alarm_distance" 的MCP工具，
 * 用于设置超声波传感器检测到障碍物时的报警距离阈值
 */
void mcp_tool_set_obstacle_alarm_distance()
{
    // 注册工具声明器，定义工具的名称和描述
    RegisterUserMcpDeclarator([](ai_vox::Engine &engine)
                              { 
                                  engine.AddMcpTool("user.obstacle.set_alarm_distance",  // 工具名称
                                                    "Set the alarm distance threshold (triggers when distance is below this value)", // 工具描述
                                                    {
                                                        {"distance",
                                                         ai_vox::ParamSchema<int64_t>{
                                                             .default_value = 20,           // 距离参数，默认值为20厘米
                                                             .min = 1,                      // 最小距离为1厘米
                                                             .max = 200,                    // 最大距离为200厘米
                                                         }
                                                        }
                                                    }); // 需要传入distance参数，单位为厘米
                              });

    // 注册工具处理器，收到调用时，设置报警距离
    RegisterUserMcpHandler("user.obstacle.set_alarm_distance", [](const ai_vox::McpToolCallEvent &ev)
                           {
                            // 解析参数
                            const auto distance_ptr = ev.param<int64_t>("distance");

                            // 检查必需参数是否存在
                            if (distance_ptr == nullptr) {
                                ai_vox::Engine::GetInstance().SendMcpCallError(ev.id, "Missing required argument: distance in cm");
                                return;
                            }

                            // 获取参数值
                            int64_t distance = *distance_ptr;

                            // 参数验证
                            if (distance < 1 || distance > 200) {
                                ai_vox::Engine::GetInstance().SendMcpCallError(ev.id, "Distance must be between 1 and 200 cm");
                                return;
                            }

                            // 设置新的报警距离阈值
                            SetObstacleAlarmDistance((float)distance);
                            
                            printf("Alarm distance updated to %.2f cm via MCP tool\n", distance);

                            // 创建响应
                            DynamicJsonDocument doc(256);
                            doc["status"] = "success";
                            doc["distance"] = distance;

                            // 将 JSON 文档转换为字符串
                            String jsonString;
                            serializeJson(doc, jsonString);

                            // 发送响应
                            ai_vox::Engine::GetInstance().SendMcpCallResponse(ev.id, jsonString.c_str());
                           });
}

/**
 * @brief 定期检查障碍物距离并控制蜂鸣器报警
 *
 * 每隔指定时间间隔检查一次前方障碍物距离，如果距离小于全局设定的报警阈值，
 * 则开启蜂鸣器报警，否则关闭蜂鸣器
 */
void CheckObstacleAndControlBuzzer() {
    unsigned long current_time = millis();
    
    // 每1秒执行一次距离检测
    if (current_time - g_last_distance_check_time >= 1000) {
        g_last_distance_check_time = current_time;
        
        // 测量当前距离
        float current_distance = MeasureUs04UltrasonicDistance();
        
        if (current_distance < 0) {
            // 测量失败，关闭蜂鸣器
            if (g_last_buzzer_state) {
                digitalWrite(BUZZER_PIN, LOW);
                g_last_buzzer_state = false;
                printf("Distance measurement failed, buzzer turned OFF\n");
            }
            return;
        }

        printf("Distance: %.2f cm, Threshold: %.2f cm\n", current_distance, g_obstacle_alarm_distance);
        
        // 根据距离判断是否需要报警
        bool need_alarm = (current_distance <= g_obstacle_alarm_distance);
        
        if (need_alarm != g_last_buzzer_state) {
            // 状态发生变化，更新蜂鸣器状态
            digitalWrite(BUZZER_PIN, need_alarm ? HIGH : LOW);
            g_last_buzzer_state = need_alarm;

            printf("Buzzer turned %s\n", need_alarm ? "ON" : "OFF");
        }
    }
}

// ========== Setup 和 Loop ==========
void setup()
{
    Serial.begin(115200);
    delay(500); // 等待串口初始化

    printf("\n========== US04 Initialization ==========\n");
    pinMode(kUs04PinTrig, OUTPUT);
    pinMode(kUs04PinEcho, INPUT);
    printf("========================================\n\n");

    pinMode(BUZZER_PIN, OUTPUT);    // 初始化蜂鸣器引脚

    // 注册MCP工具 - US04超声波传感器测距功能
    mcp_tool_us04_ultrasonic_sensor();

    // 注册MCP工具 - 蜂鸣器控制功能
    mcp_tool_buzzer_control();

    // 注册MCP工具 - 设置障碍物报警距离功能
    mcp_tool_set_obstacle_alarm_distance();

    // 初始化设备服务，包括硬件和AI引擎，必备步骤
    InitializeDevice();
}

void loop()
{
    // 处理设备服务主循环事件， 必备步骤
    ProcessMainLoop();

    // 定期检查障碍物距离并控制蜂鸣器
    CheckObstacleAndControlBuzzer();
}
```

## 语音交互使用流程

> **注意：** 请先在小智AI后台，清空历史记忆，防止出现不同程序间记忆冲突的问题。

1. 用户通过按键或语音唤醒（“你好小智”）唤醒小智AI。
2. 用户通过麦克风对AI-VOX3说出“将报警距离设置为XXX”。
3. 小智AI识别到用户输入的意图指令，并调用相应的MCP工具进行报警距离的设置。从屏幕日志中可以看到“% user.obstacle.set_alarm_distance”的MCP工具调用日志。
4. 还可以尝试对话：“当前障碍物的距离是多少？”，AI会调用“user.ultrasonic_sensor.get_distance”工具获取当前距离值并反馈给用户。”把蜂鸣器设置为报警状态”，AI会调用“user.buzzer.control”工具开启蜂鸣器报警。
