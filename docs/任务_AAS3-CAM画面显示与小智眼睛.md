# 任务上下文：AAS3-CAM 画面显示 + 小智眼睛功能

> 本文件用于在新对话窗口继续此任务。所有关键调研结论、改动进度、待办、命令均已记录在下。

## 一、任务目标

两块 ESP32-S3 板：

| 板名 | 角色 | 硬件 | 职责 |
|------|------|------|------|
| **ahuhu** | 小智+小车 | S3 彩屏板（ST7735S 128×160），四路电机 | 显示 + 大脑 + 电机执行 |
| **AAS3-CAM** | S3-CAM 摄像头板 | OV3660（QVGA 320×240 JPEG），8MB PSRAM | 采集图像，对外输出 MJPEG 流 + 本地跟随检测 + UDP 广播 |

要求：
1. **把 AAS3-CAM 的图像在 ahuhu 板子的屏幕上显示出来**（监控看画面）。
2. **让小智+小车增加眼睛功能**：跟随儿童自由行走、识别物体等能力。

参考项目：`d:\xiaozhi-esp32-main\esp32-eye-main\`（双 S3 板 + 服务端视觉识别的"眼睛"架构）。

**当前阶段**：正在实现需求 1（图像显示）。需求 2（跟随）的 **AAS3-CAM 端已完成**（本地跟随检测 + UDP 8093 广播），ahuhu 端电机执行与识别尚未开工。

---

## 二、调研结论（重要，先读）

### AAS3-CAM（发送端，独立工程，不依托小智框架）
- 独立 ESP-IDF 工程 `aas3cam_node/`，5 个源文件：`main.c`、`cam_camera.c`、`cam_http.c`、`cam_net.c`、`cam_config.h`。
- `/stream`、`/camera` 端点输出**标准 `multipart/x-mixed-replace` MJPEG 流**，boundary=`aas3camframe`。
- `/capture` 单帧 JPEG、`/healthz` JSON 状态。
- 帧格式：`--aas3camframe\r\nContent-Type: image/jpeg\r\nContent-Length: <n>\r\n\r\n` + JPEG 数据 + `\r\n`。
- 实现位置：`aas3cam_node/main/cam_http.c` 的 `stream_task()`（multipart 分帧 + 原始 send 绕过 httpd 锁）。
- 采集：`aas3cam_node/main/cam_camera.c`（OV3660 QVGA JPEG，15fps，共享帧槽 + 互斥锁）。
- **服务端口默认 80**（esp_http_server）。
- UDP 8092 端口响应 `XIAOZHI_CAM_DISCOVER` 广播，返回本机 IP（`cam_net.c` `discover_task`）。
- **UDP 8093 每 200ms 广播跟随结果** `FOLLOW <has> <zone> <dist>`（`cam_net.c` `follow_task`）。
- **本地跟随检测已实现**（`cam_camera.c` `follow_task`）：32×18 网格亮度差分 → 目标左右位置 L/C/R + 距离等级 0~3。

### ahuhu（接收端）
- `DiscoverCamera()`：UDP 广播 `XIAOZHI_CAM_DISCOVER` 找 AAS3-CAM，返回 IP（`ahuhu_board.cc` L181）。
- MCP 工具 `camera.view` / `camera.stop_view` 已注册（`ahuhu_board.cc` L1730 / L1771 附近）。
- 组装 URL：`http://<cam_ip>/camera`（**无端口** → 默认 80）。
- 用 `MjpegPlayer` 拉流显示：
  - 文件：`main/mjpeg_player.h` / `main/mjpeg_player.cc`
  - 解码上屏：`jpeg_to_image()` + 字节序交换（RB 交换）+ `lcd->DrawBitmap(0,0,w,h,...)`

### ⚠️ 核心矛盾（为何画面现在出不来）
`MjpegPlayer` **只支持私有二进制帧格式**（`type[1字节] + len[4字节] + payload`，type=0 图像，type=1 音频），而 AAS3-CAM 发的是**标准 multipart MJPEG**。两者协议不匹配 → `MjpegPlayer` 无法解析出画面。

**解析方式差异对比：**

| 端 | 发送/解析格式 |
|----|--------------|
| 参考项目 esp32-eye 发送端 cam | 标准 multipart MJPEG（boundary=`catcamframe`） |
| **参考项目接收端 display** | **通用抽取器 `MjpegExtractor`：按 JPEG 的 FFD8(SOI)起、FFD9(EOI)停扫描字节流切帧，无视 multipart 文本头** ✅ |
| AAS3-CAM 发送端 | 标准 multipart MJPEG（boundary=`aas3camframe`） |
| ahuhu 的 `MjpegPlayer` | 私有二进制帧（type[1]+len[4]），**与 AAS3-CAM 不匹配** ❌ |

### ✅ 采用的方案（已确认）
**改 ahuhu 端 `MjpegPlayer`，照搬参考项目思路**：
- 不再解析私有帧头，改为**按 `FFD8` 起、`FFD9` 停扫描字节流切帧**，抽出每个完整 JPEG 后 `jpeg_to_image` 解码上屏。
- AAS3-CAM 端**一行不改**，风险最小，且未来兼容任意标准 MJPEG 摄像头。

### 参考项目抽取器核心逻辑（借鉴来源）
参考项目 `MjpegExtractor`（`esp32-eye-main/firmware/display/src/net/live.cpp` L407 附近）：
```cpp
// 只在字节流里找 JPEG 标识字节：
//   帧起点 = 0xFF 0xD8 (SOI)
//   帧终点 = 0xFF 0xD9 (EOI)
// multipart 的 --frame\r\nContent-Type... 文本头自动被无视跳过
```

---

## 三、已完成改动（MjpegPlayer）

> 注意：**改动未编译、未烧录、未验证**。下一步必须先编译。

### 1. `main/mjpeg_player.h`
- `RingBuffer` 新增方法 `size_t ReadSome(uint8_t* out, size_t max);`（非阻塞读当前可用数据）。
- `DownloadTask` 签名改为 `void DownloadTask(std::string host, int port, std::string path);`
- 删除 `AudioTask()` 声明、`audio_queue_` 成员。

### 2. `main/mjpeg_player.cc`
- 新增 `RingBuffer::ReadSome()` 实现。
- `Play()`：URL 解析重构 ——
  - 默认端口从 `8081` 改为 `80`；
  - 从 URL 中提取路径（默认 `/camera`，支持 `:port`）；
  - 日志改为 `http://<host>:<port><path>`；
  - 传给 `DownloadTask(host, port, path)`；
  - **删除 AudioTask 线程的创建**。
- `Stop()`：移除 `audio_thread_.joinable()` 引用。

### 未完成部分（改动中，下方待办里说明）

---

## 四、待办（按顺序执行）

- [ ] **1. 完成 `DownloadTask(host, port, path)` 改造**：请求字符串不再写死 `/stream`，改用传入的 `path`（`GET <path> HTTP/1.1`）。
- [ ] **2. 重写 `PlaybackTask()`**：移除"读 hdr[5] + type/len + payload"的私有帧解析，改为**通用 MJPEG 抽取器**（按 FFD8/FFD9 切帧），抽出完整 JPEG → `jpeg_to_image` → 字节序交换 → `DrawBitmap`。
- [ ] **3. 清理 `PlaybackTask` 末尾**：`playing_=false; abort_=true; ring_buf_.SetDone(); audio_cv_.notify_all(); join download_thread; ring_buf_.Deinit(); 触发 on_end_`（保留现有结束清理逻辑，去掉 audio 相关引用）。
- [ ] **4. 删除 `MjpegPlayer::AudioTask()` 及其引用**（已计划废弃，确认无残留调用）。
- [ ] **5. 编译验证**：无报错、诊断通过。
- [ ] **6. 烧录 + 监控日志验证**：确认 `Play(0) Stream: http://x.x.x.x:80/camera` 连接成功、出现 `Playback: N frames` 帧计数增长。
- [ ] **7. 语音/按钮触发 `camera.view`**，确认 ahuhu 屏幕显示 AAS3-CAM 画面。
- [ ] **8. Git 提交**（中文提交信息，说明"修复 AAS3-CAM 摄像头画面显示"）。
- [ ] **9. 需求 2（眼睛/跟随/识别）**：另起设计，基于 esp32-eye 架构（本文件暂不展开）。

---

## 五、需求 2 方向（路线① AAS3-CAM 端已完成，ahuhu 端待做）

见之前的分析：ahuhu 无本机摄像头（PSRAM 被占用）、电机已齐全（MCP 工具 `motor.forward/backward/turn_left/turn_right/dance` 已注册，见 `ahuhu_board.cc` L1259-1330）。

参考 esp32-eye 三段式：cam(采集) → server(识别) → display(显示)。但小车实时跟随对延迟敏感，两个方向：
- **路线①（不依赖电脑，推荐先做）**：AAS3-CAM 本地做画面分析（求目标质心在左/中/右），把**判断结果**（非整帧图）用 UDP 发给 ahuhu，ahuhu 据方位驱动电机追人/绕障。数据小、延迟低。
  - ✅ **AAS3-CAM 端已完成**：`cam_camera.c` 的 `follow_task` 本地跟随检测（32×18 网格亮度差分 → 目标左右位置 `L/C/R` + 距离等级 `0~3`）；`cam_net.c` 的 `follow_task` 每 200ms 通过 UDP 8093 广播 `FOLLOW <has> <zone> <dist>`。
  - ✅ **ahuhu 端已完成**：`ahuhu_board.cc` 的 `StartFollowService()`（约 L94-155）监听 UDP 8093、`sscanf` 解析 `FOLLOW %d %c %d`，按方位驱动电机（`L`→左转 / `R`→右转 / `C`→前进，dist 大慢、小快，无目标时左右扫描找人）；语音 MCP 工具 `follow.start` / `follow.stop` 开关跟随。
- **路线②（进阶，识别物体种类）**：整帧图像给电脑/大模型识别（像 esp32-eye 接 MiniCPM-V），能"认出这是什么"，但依赖局域网电脑在线。

**待用户拍板决策点**（需要在新对话中确认）：跟随放本地（路线①）还是接电脑（路线②）；识别范围是"障碍/人在左中右"还是"认出具体物体"。

---

## 六、AAS3-CAM ↔ AHUHU 对接验证（路线①，已确认）

### 6.1 协议完全匹配 ✅

| 项 | AAS3-CAM（发送） | AHUHU（接收） | 结果 |
|----|------------------|---------------|------|
| 端口 | UDP 8093 广播 | UDP 8093 绑定 INADDR_ANY | ✅ 一致 |
| 格式 | `FOLLOW <has> <zone> <dist>\n` | `sscanf(buf,"FOLLOW %d %c %d",...)` | ✅ 一致 |
| has | 1有目标 / 0无 | 同 | ✅ |
| zone | L左 / C中 / R右（目标在画面位置） | L→左转 / C→前进 / R→右转 | ✅ 语义正确 |
| dist | 运动面积等级 1远 2中 3近 | dist大→慢速、dist小→快速 | ✅ 近慢远快，正确 |
| 周期 | 每 200ms | 2 秒无上报判丢失 | ✅ |

### 6.2 语音调用链路（已打通）

```
孩子说"跟紧我/离我近一点" → 小智语音 → LLM 决策 → 调 MCP 工具 follow.start
  → g_follow_active = true → cam_follow_move 任务每 220ms 依据 UDP 8093 上报驱动电机
孩子说"停/够了/别跟了" → follow.stop → 停止电机
```

### 6.3 能做什么 / 不能做什么

| 场景 | 能否实现 | 原因 |
|------|---------|------|
| 跟随移动的小朋友（如"跟紧我"） | ✅ 能 | 帧间运动差分能锁定移动目标 |
| 靠近移动目标（dist 越大越慢） | ✅ 能 | 运动面积近似距离，近慢远快 |
| "往门口一点"（静态目标导航） | ❌ 不能 | 运动差分检测不到静止的门口 |
| 区分"小朋友"和其他移动物体 | ❌ 不能 | 无目标识别，只有运动检测，多人时可能跟错 |

### 6.4 遗留问题

1. **无目标识别**：`follow_update` 只做帧间亮度差分，分不清"人/宠物/其他"，多人多物时跟随最显著运动区域，不保证是"小朋友"。
2. **dist 是运动面积粗估**：`active>=90→3 / >=45→2 / else 1`，不是真实距离，受目标大小影响（大人近处 vs 小孩远处可能同面积）。
3. **1 秒更新滞后**：`CAM_FOLLOW_MS=1000` 每 1s 才重新解码分析，UDP 每 200ms 发的是缓存结果，跟随响应最多滞后 1s（可降到 300~500ms 提速，但增加 CPU/PSRAM 负载）。
4. **静态目标导航（"往门口"）需另做**：走路线②接大模型识别，或在 AAS3-CAM 加轻量目标检测（ESP32-S3 算力有限，只能跑颜色/形状特征或微型模型）。

---

## 七、相关文件速查

| 文件 | 作用 |
|------|------|
| `main/mjpeg_player.h` / `.cc` | ahuhu 端 MJPEG 播放器（**本任务主要修改对象**） |
| `aas3cam_node/main/cam_http.c` `stream_task()` | AAS3-CAM MJPEG 流输出 `/stream` `/camera`（**不改**） |
| `aas3cam_node/main/cam_camera.c` | AAS3-CAM 采集 + 本地跟随检测 |
| `aas3cam_node/main/cam_net.c` | AAS3-CAM UDP 8092 发现 + UDP 8093 跟随广播 |
| `aas3cam_node/main/cam_config.h` | AAS3-CAM 引脚 / WiFi / 参数配置 |
| `aas3cam_node/AAS3-CAM独立摄像头节点说明.md` | AAS3-CAM 构建 / 功能 / 工作逻辑说明 |
| `main/boards/ahuhu/ahuhu_board.cc` L181 `DiscoverCamera`、L1730 `camera.view` | ahuhu 发现摄像头 + MCP 工具 |
| `main/boards/ahuhu/ahuhu_board.cc` L1259-1330 | 电机 MCP 工具（需求 2 用） |
| `esp32-eye-main/firmware/cam/src/http_server.cpp` | 参考发送端（multipart MJPEG） |
| `esp32-eye-main/firmware/display/src/net/live.cpp` L407+ | 参考接收端 `MjpegExtractor`（**借鉴核心**） |

---

## 八、编译/烧录/监控命令（项目规则）

所有命令在 `d:\xiaozhi-esp32-main` 下执行，先设置环境变量：

```powershell
$env:IDF_PATH = "D:\S3\v5.5.4\esp-idf"
$env:PATH = "C:\Espressif\tools\cmake\3.30.2\bin;C:\Espressif\tools\ninja\1.12.1;C:\Espressif\tools\ccache\4.9;C:\Espressif\tools\xtensa-esp-elf\esp-14.2.0_20260121\xtensa-esp-elf\bin;" + $env:PATH
```

带进度脚本（优先）：
```powershell
.\build_progress.ps1 build      # 编译（显示 [N/M] 文件数）
.\build_progress.ps1 flash      # 烧录（esptool 百分比进度条）
.\build_progress.ps1 monitor    # 监控
.\build_progress.ps1 all        # 编译+烧录 一键
```

底层命令：
```powershell
# 编译
& "C:\Espressif\tools\python\v5.5.4\venv\Scripts\python.exe" "D:\S3\v5.5.4\esp-idf\tools\idf.py" build
# 烧录
& "C:\Espressif\tools\python\v5.5.4\venv\Scripts\python.exe" "D:\S3\v5.5.4\esp-idf\tools\idf.py" -p COM3 flash
# 监控
& "C:\Espressif\tools\python\v5.5.4\venv\Scripts\python.exe" "D:\S3\v5.5.4\esp-idf\tools\idf.py" -p COM3 monitor
```

> 遵循用户规则：写完代码自检一遍；编译烧录监控不必提醒、自开进度条；先工具尝试；改完必须 Git 提交。
> **注意**：COM3 串口易被残留 monitor 进程占用，烧录前如遇口占用需先结束 `idf.py -p COM3` / `idf_monitor.py` 进程。