## ESP的开发

以下是 **ESP32 烧录上述 TCP 服务器代码（含 JSON 解析）** 的详细步骤，涵盖 **开发环境配置、代码上传、监控调试** 等全流程：

------

### **1. 开发环境准备**

#### **(1) 安装 Arduino IDE**

- 下载地址：[Arduino 官网](https://www.arduino.cc/en/software)

- 安装后打开，进入 

  首选项

  ```
  File > Preferences
  ```

  在 附加开发板管理器网址中添加 ESP32 的 URL：

  ```
  https://dl.espressif.com/dl/package_esp32_index.json
  ```

- 进入 **工具 > 开发板 > 开发板管理器**，搜索 **ESP32** 并安装。

#### **(2) 安装 ArduinoJson 库**

- 进入 **工具 > 管理库**（`Sketch > Include Library > Manage Libraries`），搜索 **ArduinoJson**，安装最新版本（推荐 v7.0+）。

------

### **2. 代码准备**

- 将以下代码复制到 Arduino IDE 的新项目中（文件名如 ）：

  ```
  tcp_json_server.ino
  ```

  ```
  #include <WiFi.h>
  #include <ArduinoJson.h>
  
  // WiFi配置
  const char* WIFI_SSID = "your_SSID";       // 替换为你的WiFi名称
  const char* WIFI_PASSWORD = "your_PASSWORD"; // 替换为你的WiFi密码
  
  // TCP服务器配置
  WiFiServer tcpServer(5000);  // 监听端口5000
  WiFiClient client;
  
  void setup() {
    Serial.begin(115200);
    WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
    while (WiFi.status() != WL_CONNECTED) {
      delay(500);
      Serial.print(".");
    }
    Serial.println("\nWiFi connected!");
    Serial.print("IP Address: ");
    Serial.println(WiFi.localIP());  // 打印ESP32的IP地址
  
    tcpServer.begin();
    Serial.println("TCP server started on port 5000");
  }
  
  void loop() {
    if (!client || !client.connected()) {
      client = tcpServer.available();
      if (client) {
        Serial.println("Client connected!");
      }
      delay(10);
      return;
    }
  
    if (client.available()) {
      String jsonStr = client.readStringUntil('\n');
      jsonStr.trim();
  
      StaticJsonDocument<256> doc;
      DeserializationError error = deserializeJson(doc, jsonStr);
      if (error) {
        Serial.print("JSON解析失败: ");
        Serial.println(error.c_str());
        return;
      }
  
      float linear_x = doc["linear_x"];
      float angular_z = doc["angular_z"];
  
      Serial.print("Received: linear_x=");
      Serial.print(linear_x);
      Serial.print(", angular_z=");
      Serial.println(angular_z);
  
      // TODO: 添加电机控制逻辑（如UART发送到STM32）
    }
    delay(10);
  }
  ```

------

### **3. 配置 ESP32 开发板**

1. **选择开发板型号**：
   - 进入 **工具 > 开发板 > ESP32 Arduino**，选择你的 ESP32 型号（如 `ESP32 Dev Module`）。
   - 关键设置：
     - **CPU Frequency**: 240MHz（默认）
     - **Flash Mode**: QIO（默认）
     - **Flash Size**: 4MB（或更大，根据你的ESP32型号）
     - **Partition Scheme**: Default（默认）
2. **连接 ESP32 到电脑**：
   - 使用 USB 数据线连接 ESP32 和电脑，确保驱动已安装（Windows 可能需要 [CP210x 或 CH340 驱动](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/get-started/establish-serial-connection.html)）。
   - 在 Arduino IDE 的 **工具 > 端口** 中选择对应的 COM 端口（如 `COM3` 或 `/dev/ttyUSB0`）。

------

### **4. 烧录代码到 ESP32**

1. 编译代码：

   - 点击 Arduino IDE 的 **验证**（✔️图标）按钮，检查代码是否有语法错误。

2. 上传代码：

   - 点击 **上传**（→图标）按钮，ESP32 会自动重启并开始烧录。

   - 观察串口监视器（波特率 ），

     ```
     Ctrl+Shift+M
     ```

     ```
     115200
     ```

     确认输出类似以下信息：

     ```
     WiFi connected!
     IP Address: 192.168.1.100
     TCP server started on port 5000
     Client connected!
     Received: linear_x=0.5, angular_z=0.2
     ```

------

### **5. 监控与调试**

#### **(1) 串口监视器**

- 在 Arduino IDE 中打开串口监视器（`Ctrl+Shift+M`），设置波特率为 `115200`，实时查看 ESP32 的日志输出。
- 关键日志：
  - `WiFi connected!`：WiFi 连接成功。
  - `IP Address: xxx`：ESP32 的本地 IP 地址（用于 TCP 连接）。
  - `Client connected!`：有客户端（如 ROS 2 节点）连接。
  - `Received: linear_x=...`：成功解析 JSON 数据。

#### **(2) 网络调试**

- 测试 TCP 连接：

  在电脑上使用 

  ```
  telnet
  ```

   或 

  ```
  nc
  ```

   工具手动发送 JSON 数据到 ESP32：

  ```
  # Linux/macOS
  echo '{"linear_x":0.5, "angular_z":0.2}' | nc 192.168.1.100 5000
  
  # Windows（需安装 telnet 客户端）
  telnet 192.168.1.100 5000
  # 然后手动输入 JSON 字符串并按 Ctrl+] 退出
  ```

------

### **6. 常见问题解决**

#### **(1) WiFi 连接失败**

- 可能原因：
  - WiFi 名称或密码错误。
  - ESP32 未获取到 IP 地址（检查路由器是否启用 DHCP）。
- 解决方案：
  - 确认 `WIFI_SSID` 和 `WIFI_PASSWORD` 正确。
  - 在串口监视器中检查 ESP32 是否打印 `WiFi connected`。

#### **(2) TCP 客户端无法连接**

- 可能原因：
  - ESP32 的防火墙或路由器阻止了端口 `5000`。
  - IP 地址错误（确保 ROS 2 节点连接的是 ESP32 的 `IP Address`）。
- 解决方案：
  - 在路由器后台检查端口是否开放。
  - 在 ESP32 代码中打印 `WiFi.localIP()` 确认 IP 地址。

#### **(3) JSON 解析失败**

- 可能原因：
  - JSON 格式错误（如缺少引号、逗号）。
  - 字段名不匹配（如 `linear_x` 拼写错误）。
- 解决方案：
  - 在串口监视器中打印原始 `jsonStr`，确认格式正确。
  - 使用在线工具（如 [JSONLint](https://jsonlint.com/)）验证 JSON。

------

### **7. 进阶操作**

#### **(1) 添加电机控制逻辑**

在 `loop()` 中解析 JSON 后，通过 UART 发送指令到 STM32：

```
// 示例：通过 UART 发送 PWM 指令
if (client.available()) {
  String jsonStr = client.readStringUntil('\n');
  // ...（解析 JSON）
  
  // 计算 PWM 值（示例逻辑）
  int left_pwm = map(linear_x * 100, 0, 100, 0, 1000);
  int right_pwm = map(angular_z * 50, 0, 50, 0, 1000);

  // 通过 UART 发送指令到 STM32
  String uartCmd = "L" + String(left_pwm) + ",R" + String(right_pwm) + "\n";
  uart2.println(uartCmd);  // 假设 uart2 已初始化
}
```

#### **(2) 动态配置 WiFi**

通过 Arduino 的 **Preferences** 库存储 WiFi 配置，避免硬编码：

```
#include <Preferences.h>
Preferences prefs;
void setup() {
  prefs.begin("wifi", false);
  const char* ssid = prefs.getString("ssid", "default_SSID").c_str();
  const char* pass = prefs.getString("pass", "default_PASS").c_str();
  WiFi.begin(ssid, pass);
  // ...
}
```

------

### **总结**

以下是 **ESP32 烧录上述 TCP 服务器代码（含 JSON 解析）** 的详细步骤，涵盖 **开发环境配置、代码上传、监控调试** 等全流程：

------

`[头标志(0xAA)][线速度高字节][线速度低字节][角速度高字节][角速度低字节][校验和(低字节)]`

## Arduino开发

### wifi连接

![image-20250715111024394](/home/wybie/dailywork/笔记/ROS/ROS2项目学习.assets/image-20250715111024394.png)

![image-20250715111132751](/home/wybie/dailywork/笔记/ROS/ROS2项目学习.assets/image-20250715111132751.png)

- AP模式：通过热点连接但电脑不能上网
- STA模式：常见的客户端连接模式

执行命令：

```
cd ~/arduino-1.8.19
./arduino
```

如果ch340驱动不能正确加载即没有设备号时，有概率是一个盲人软件brltty占用

https://blog.csdn.net/qq_27865227/article/details/125538516

![image-20250715144021235](/home/wybie/dailywork/笔记/ROS/ROS2项目学习.assets/image-20250715144021235.png)

![image-20250715144103922](/home/wybie/dailywork/笔记/ROS/ROS2项目学习.assets/image-20250715144103922.png)

![image-20250715144236391](/home/wybie/dailywork/笔记/ROS/ROS2项目学习.assets/image-20250715144236391.png)

![image-20250715145935025](/home/wybie/dailywork/笔记/ROS/ROS2项目学习.assets/image-20250715145935025.png)

![image-20250715145959845](/home/wybie/dailywork/笔记/ROS/ROS2项目学习.assets/image-20250715145959845.png)

![image-20250715150905465](/home/wybie/dailywork/笔记/ROS/ROS2项目学习.assets/image-20250715150905465.png)

解析网址：`https://tool.juhe.cn/`

![image-20250715151019899](/home/wybie/dailywork/笔记/ROS/ROS2项目学习.assets/image-20250715151019899.png)

### sd卡

![image-20250715152720847](/home/wybie/dailywork/笔记/ROS/ROS2项目学习.assets/image-20250715152720847.png)

![image-20250715152922194](/home/wybie/dailywork/笔记/ROS/ROS2项目学习.assets/image-20250715152922194.png)

![image-20250715153129550](/home/wybie/dailywork/笔记/ROS/ROS2项目学习.assets/image-20250715153129550.png)

### 串口

![image-20250715154420901](/home/wybie/dailywork/笔记/ROS/ROS2项目学习.assets/image-20250715154420901.png)

![image-20250715154514976](/home/wybie/dailywork/笔记/ROS/ROS2项目学习.assets/image-20250715154514976.png)

![image-20250715154710868](/home/wybie/dailywork/笔记/ROS/ROS2项目学习.assets/image-20250715154710868.png)

jsmn解析json：

https://blog.csdn.net/qq_36347513/article/details/108640754

## 