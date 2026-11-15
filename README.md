# xiaozhi-vision-dev

小智 AI 视觉跟随系统的**开发源码子集** —— 从本地 `xiaozhi-esp32` 分支中抽出的视觉改造相关文件，不含上游固件全量与构建产物。

> 面向「可运行成品」的整理版见姊妹仓库 [xiaozhi-vision-follow](https://github.com/huhu321-daya/xiaozhi-vision-follow)（电脑端大脑 + 发送板 + 接收板 + 三阶段方案文档）。

## 与上游的关系

基于 [78/xiaozhi-esp32](https://github.com/78/xiaozhi-esp32)（MIT）的本地分支。本仓库只保留视觉相关改动：

- `main/boards/ahuhu/` 与 `main/boards/AAS3-CAM/` 需要**覆盖回上游同名路径**后随主框架编译
- 其余目录可独立使用

未包含上游代码与 `LICENSE`，使用时请自行 clone 上游并遵循其 MIT 协议。

## 目录

| 路径 | 说明 |
|---|---|
| `main/boards/ahuhu/` | 接收板（ahuhu 板）完整板级实现：`ahuhu_board.cc`(118KB) 集成 UDP 电机指令、彩屏待机、天气/新闻服务、OV7670/ST7735 驱动 |
| `main/boards/AAS3-CAM/` | 发送板板级说明 |
| `aas3cam_node/` | AAS3-CAM 独立摄像头节点（ESP-IDF 项目，可单独编译：MJPEG 推流 + 本地人体检测 + UDP 广播） |
| `phone_follow/` | 电脑端大脑（YOLOv8n 人体检测 + 跟随决策） |
| `tests/` | 视觉链路验证脚本：硅基流动 Qwen3-VL / 智谱 GLM-4V / 本地 Ollama |
| `docs/` | 视觉方案进度、任务记录 |
| `build_aas3cam.ps1`、`sdkconfig.aas3cam.defaults` | 发送节点编译脚本与配置基线 |

## 凭据配置

仓库**不含任何真实凭据**，全部为 `YOUR_*` 占位符。三种注入方式任选：

```c
// C 侧（cam_config.h / ahuhu_board.cc）已用 #ifndef 包裹，编译宏优先
idf.py -DCAM_WIFI_SSID=\"你的SSID\" -DCAM_WIFI_PASSWORD=\"密码\" -DCAM_VLM_KEY=\"sk-xxx\" build
```

```bash
# Python 测试脚本读环境变量
export VLM_API_KEY=sk-xxx        # tests/test_follow_vlm.py
export ZHIPU_API_KEY=xxx         # tests/test_zhipu_vision.py
```

## 说明

`phone_follow/yolov8n.onnx`（12.3MB）未纳入本仓库，可从姊妹仓库获取，或自行导出：

```bash
pip install ultralytics && yolo export model=yolov8n.pt format=onnx
```
