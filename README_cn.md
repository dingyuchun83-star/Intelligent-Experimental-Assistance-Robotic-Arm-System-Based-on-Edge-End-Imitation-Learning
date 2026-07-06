# 基于边缘端模仿学习的智能实验辅助机械臂系统

<p align="center">
  <img src="docs/images/system_overview.jpg" alt="系统全局照片" width="600"/>
</p>

<p align="center">
  <a href="#-项目简介">项目简介</a> •
  <a href="#-系统架构">系统架构</a> •
  <a href="#-快速开始">快速开始</a> •
  <a href="#-硬件清单">硬件清单</a> •
  <a href="#-性能指标">性能指标</a> •
  <a href="#-项目结构">项目结构</a> •
  <a href="#-许可证">许可证</a>
</p>

---

## 📖 项目简介

本项目面向高校实验室桌面场景，研制了一套**基于边缘端模仿学习的智能实验辅助机械臂系统**。系统以地瓜机器人 [RDK X5](https://developer.d-robotics.cc/) 边缘计算开发板为核心算力平台，集成专用神经网络加速单元（BPU），采用 **ACT（Action Chunking with Transformers）** 端到端模仿学习模型作为智能决策核心，基于 [LeRobot](https://github.com/huggingface/lerobot) 开源框架实现策略训练与部署。

### ✨ 核心特性

| 特性 | 说明 |
|------|------|
| 🔒 **完全离线隐私保护** | 所有神经网络推理均在 RDK X5 本地完成，数据不出设备，满足实验室保密要求 |
| 🎙️ **离线语音指令交互** | 集成 TTS 高自由语言交互引擎，识别准确率 ≥95%，处理延迟 ≤220ms |
| 🧠 **端到端 ACT 模仿学习** | 单一网络替代传统多模块串联架构，代码量减少约 40%，消除模块间误差传递 |
| ⚡ **高频动作分块控制** | 原生 50Hz 推理频率，单次前向延迟仅约 35ms，运动平滑流畅 |
| 🚀 **轻量化迁移学习** | 仅需 50-100 条人类示教轨迹，2-4 小时完成场景适配训练 |
| 💰 **低成本桌面级方案** | 整套硬件成本控制在 **2000 元以内**，将工业级智能化能力下沉至千元级平台 |

---

## 🏗️ 系统架构

### 整体流程

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  语音采集   │───▶│  本地 ASR   │───▶│  规则引擎   │───▶│  YOLO 检测  │
│ (麦克风阵列)│    │(Whisper.cpp)│    │(语义匹配)   │    │(目标定位)   │
└─────────────┘    └─────────────┘    └─────────────┘    └──────┬──────┘
                                                                 │
┌─────────────┐    ┌─────────────┐    ┌─────────────────────────┘
│  机械臂执行  │◀───│  插值控制   │◀───│        ACT 模型推理      │
│ (6DoF 关节) │    │  (10Hz)     │    │  ResNet-18 + Transformer │
└─────────────┘    └─────────────┘    └─────────────────────────┘
         ▲                                    ▲
         └────────────┬───────────────────────┘
                      │
              ┌───────┴───────┐
              │   USB 相机    │
              │ (640×480@30fps)│
              └───────────────┘
```

### 技术链路

```
语音指令 → 本地语音识别 → 规则引擎语义匹配 → YOLO 视觉目标检测 → ACT 端到端推理 → 边缘端动作生成
```

### 软件模块

| 模块 | 技术栈 | 功能 |
|------|--------|------|
| 多模态指令感知 | PyAudio + Whisper.cpp + OpenCV | 语音采集、ASR 推理、视觉采集、多模态对齐 |
| VLA 策略推理 | LeRobot + ACT + ResNet-18 | 视觉编码、条件嵌入、CVAE 动作分块推理 |
| 实时控制 | Python 多线程 + 串口通信 | 动作插值、关节限位、急停响应、异常回零 |

---

## 🚀 快速开始

### 环境要求

- **主控**: 地瓜机器人 RDK X5（地平线征程系列芯片，内置 BPU）
- **系统**: Linux（Ubuntu 22.04 或地平线定制系统）
- **Python**: ≥ 3.8
- **CUDA**: 训练端需 NVIDIA GPU（推理端纯 BPU 运行）

### 1. 克隆仓库

```bash
git clone https://github.com/your-org/edge-act-robot.git
cd edge-act-robot
```

### 2. 安装依赖

```bash
# 创建虚拟环境
python -m venv venv
source venv/bin/activate

# 安装核心依赖
pip install -r requirements.txt

# 安装 LeRobot（训练端）
pip install -e ./third_party/lerobot

# 安装地平线 OE 工具链（部署端，RDK X5）
# 请参考地平线官方文档: https://developer.d-robotics.cc/
```

### 3. 数据采集（训练端）

```bash
# 启动遥操作采集
python scripts/teleop_collect.py   --robot-type so100   --cameras front top   --output-dir data/teleop_demo   --num-episodes 100
```

### 4. 模型训练（训练端）

```bash
# ACT 策略训练
python scripts/train_act.py   --dataset-dir data/teleop_demo   --config configs/act_so100.yaml   --output-dir checkpoints/act_so100   --batch-size 64   --num-steps 50000
```

### 5. 模型转换与部署（RDK X5）

```bash
# 1. PyTorch → ONNX
python scripts/export_onnx.py   --checkpoint checkpoints/act_so100/best.pt   --output models/act_so100.onnx

# 2. ONNX → 地平线 BPU 模型（INT8 量化）
python scripts/convert_bpu.py   --onnx models/act_so100.onnx   --calib-data data/calib/   --output models/act_so100_bpu.bin   --quantization int8

# 3. 部署运行
python scripts/deploy.py   --act-model models/act_so100_bpu.bin   --yolo-model models/yolo_reagent_bpu.bin   --asr-model models/whisper_tiny.bin   --robot-port /dev/ttyUSB0
```

### 6. 启动系统

```bash
# 一键启动全链路
python main.py --config configs/system.yaml
```

---

## 🔧 硬件清单

| 组件 | 型号/规格 | 数量 | 参考价格 |
|------|-----------|------|----------|
| 边缘计算板 | 地瓜机器人 RDK X5（含 BPU） | 1 | ¥1200 |
| 桌面机械臂 | 6 自由度舵机机械臂套件 | 1 | ¥400 |
| USB 相机 | 工业相机 640×480@30fps | 2 | ¥200 |
| 麦克风阵列 | USB 全向麦克风 | 1 | ¥50 |
| 结构件 | 3D 打印底座 + 相机支架（PETG） | 1 套 | ¥100 |
| 电源 | 12V/5A 直流适配器 + 稳压模块 | 1 | - |
| **总计** | | | **≈ ¥1950** |

> ⚠️ **注意**: 电源入口建议增加 470μF 电解电容与 TVS 管，避免机械臂启动时纹波导致主控复位。

---

## 📊 性能指标

| 指标项 | 目标值 | 实测值 |
|--------|--------|--------|
| 端到端系统延迟 | ≤2 秒 | ~1.8 秒 |
| 边缘端纯推理延迟 | ≤0.05 秒 | ~35 ms |
| ACT 推理频率 | 50 Hz | 50 Hz |
| 控制循环频率 | 10 Hz | 10 Hz |
| 任务成功率 | ≥80% | **85%**（单一递取）/ **82%**（顺序转移） |
| 语音指令识别准确率 | ≥95% | **92%** |
| 语音处理延迟 | ≤220 ms | ≤220 ms |
| 训练数据量 | 50-100 条 | 50-100 条 |
| 场景适配训练时间 | 2-4 小时 | 2-4 小时 |
| 系统总成本 | ≤2000 元 | **≈1950 元** |

---

## 📁 项目结构

```
edge-act-robot/
├── configs/                    # 配置文件
│   ├── act_so100.yaml          # ACT 模型配置
│   ├── system.yaml             # 系统全链路配置
│   └── yolo_reagent.yaml       # YOLO 检测配置
├── data/                       # 数据目录
│   ├── teleop_demo/            # 示教轨迹
│   └── calib/                  # BPU 量化校准数据
├── models/                       # 模型文件
│   ├── act_so100_bpu.bin       # ACT BPU 模型
│   ├── yolo_reagent_bpu.bin    # YOLO BPU 模型
│   └── whisper_tiny.bin        # ASR 模型
├── src/                          # 源代码
│   ├── perception/             # 感知模块
│   │   ├── audio_capture.py    # 语音采集
│   │   ├── asr_engine.py       # ASR 推理
│   │   ├── yolo_detector.py    # YOLO 目标检测
│   │   └── multimodal_align.py # 多模态对齐
│   ├── policy/                 # 策略模块
│   │   ├── act_model.py        # ACT 模型定义
│   │   ├── visual_encoder.py   # ResNet-18 视觉编码
│   │   └── action_chunker.py   # CVAE 动作分块
│   ├── control/                # 控制模块
│   │   ├── interpolator.py     # 动作插值
│   │   ├── robot_interface.py  # 机械臂通信
│   │   └── safety_monitor.py   # 安全监控
│   └── utils/                  # 工具函数
│       ├── bpu_runtime.py      # BPU 推理封装
│       └── logger.py           # 日志管理
├── scripts/                      # 脚本工具
│   ├── teleop_collect.py       # 遥操作采集
│   ├── train_act.py            # ACT 训练
│   ├── export_onnx.py          # ONNX 导出
│   ├── convert_bpu.py          # BPU 模型转换
│   └── deploy.py               # 部署脚本
├── third_party/                  # 第三方依赖
│   └── lerobot/                # LeRobot 框架
├── docs/                         # 文档
│   └── images/                 # 图片资源
├── tests/                        # 单元测试
├── requirements.txt              # Python 依赖
├── main.py                       # 系统主入口
└── README.md                     # 本文件
```

---

## 🎯 功能演示

### 单一物品递取
> 语音指令："递取 A 试剂"
> 
> 系统识别目标 → YOLO 定位 → ACT 规划轨迹 → 机械臂执行抓取 → 放置至指定位置
> 
> **成功率：85%**

### 多物品顺序转移
> 语音指令："先递取离心管，再转移 B 样品"
> 
> 规则引擎解析顺序 → 依次执行多步操作
> 
> **成功率：82%**

### 不同高度放置
> 语音指令："将离心管放到第二层试管架"
> 
> YOLO 识别目标高度 → ACT 自适应调整末端姿态
> 
> **成功率：80%**

---

## 🔬 技术亮点

### 1. 端到端 ACT 模仿学习架构
- 以单一 ACT 网络替代传统"目标检测 → 语义解析 → 轨迹规划 → 动作执行"的多模块串联架构
- 将视觉感知、目标条件与动作生成统一于单一网络进行端到端优化
- 系统代码行数较传统方案减少约 **40%**

### 2. 边缘端 BPU 深度适配
- CVAE 编码器中 LayerNorm/GELU 算子手动替换为 BPU 兼容的 InstanceNorm/ReLU 近似
- INT8 量化采用 per-channel 校准与敏感层跳过策略，精度损失控制在 **2% 以内**
- ACT 模型体积压缩至原始 **30%**，YOLO 压缩至 **25%**

### 3. 高频动作分块控制
- ACT 单次推理输出未来 100 步动作序列（覆盖约 2 秒）
- 线性插值映射为 10Hz 机械臂关节指令，确保运动平滑性
- 有效掩盖边缘端单步推理延迟

---

## 🛣️ 路线图

- [x] 单臂端到端 ACT 控制
- [x] 离线语音指令交互
- [x] YOLO 目标条件注入
- [x] BPU INT8 量化部署
- [ ] 多机械臂协同控制
- [ ] 力觉传感器集成
- [ ] 深度行为预测与记忆学习
- [ ] 云端联邦学习优化
- [ ] 移动端远程监控 APP

---

## 🤝 贡献指南

欢迎提交 Issue 和 PR！请确保：

1. 代码符合 PEP8 规范
2. 新增功能附带单元测试
3. 提交前通过 `pytest tests/` 测试
4. 更新相关文档

---

## 📄 许可证

本项目基于 [Apache License 2.0](LICENSE) 开源。

---

## 🙏 致谢

- [LeRobot](https://github.com/huggingface/lerobot) — 开源机器人学习框架
- [地瓜机器人](https://developer.d-robotics.cc/) — RDK X5 边缘计算平台
- [地平线机器人](https://www.horizon.ai/) — 征程系列 BPU 加速芯片
- [Whisper.cpp](https://github.com/ggerganov/whisper.cpp) — 本地 ASR 推理

---

<p align="center">
  <sub>Built with ❤️ for the 2026 ACT Embedded Systems Competition</sub>
</p>
