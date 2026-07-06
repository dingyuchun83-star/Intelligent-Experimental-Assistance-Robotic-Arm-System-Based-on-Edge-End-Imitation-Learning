# Edge-End Imitation Learning Based Intelligent Lab-Assist Robotic Arm System

<p align="center">
  <img src="docs/images/system_overview.jpg" alt="System Overview" width="600"/>
</p>

<p align="center">
  <a href="#-introduction">Introduction</a> •
  <a href="#-system-architecture">Architecture</a> •
  <a href="#-quick-start">Quick Start</a> •
  <a href="#-hardware-bom">Hardware</a> •
  <a href="#-performance">Performance</a> •
  <a href="#-project-structure">Structure</a> •
  <a href="#-license">License</a>
</p>

---

## 📖 Introduction

This project presents an **intelligent lab-assist robotic arm system based on edge-end imitation learning**, targeting desktop scenarios in university laboratories. The system is built around the [RDK X5](https://developer.d-robotics.cc/) edge computing development board by DiRobot, featuring a dedicated neural network acceleration unit (BPU). It employs the **ACT (Action Chunking with Transformers)** end-to-end imitation learning model as the core intelligent decision engine, with strategy training and deployment implemented via the open-source [LeRobot](https://github.com/huggingface/lerobot) framework.

### ✨ Key Features

| Feature | Description |
|---------|-------------|
| 🔒 **Fully Offline Privacy Protection** | All neural network inference runs locally on RDK X5; data never leaves the device, meeting lab confidentiality requirements |
| 🎙️ **Offline Voice Command Interaction** | Integrated TTS engine with high flexibility; ≥95% recognition accuracy, ≤220ms processing latency |
| 🧠 **End-to-End ACT Imitation Learning** | Single network replaces traditional multi-module pipeline; ~40% reduction in codebase, eliminating inter-module error propagation |
| ⚡ **High-Frequency Action Chunking Control** | Native 50Hz inference frequency, ~35ms single forward-pass latency, ensuring smooth motion |
| 🚀 **Lightweight Transfer Learning** | Only 50-100 human demonstration trajectories needed; scene adaptation training completed in 2-4 hours on a consumer GPU |
| 💰 **Low-Cost Desktop Solution** | Total hardware cost under **$280 USD**, democratizing industrial-grade intelligence to the desktop level |

---

## 🏗️ System Architecture

### Overall Pipeline

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ Audio Capture│───▶│  Local ASR  │───▶│ Rule Engine │───▶│ YOLO Detect │
│(Mic Array)   │    │(Whisper.cpp)│    │(Semantic)   │    │(Target Loc) │
└─────────────┘    └─────────────┘    └─────────────┘    └──────┬──────┘
                                                                  │
┌─────────────┐    ┌─────────────┐    ┌─────────────────────────┘
│ Robot Execute│◀───│ Interpolator│◀───│      ACT Model Inference │
│ (6DoF Joints)│    │   (10Hz)    │    │  ResNet-18 + Transformer │
└─────────────┘    └─────────────┘    └─────────────────────────┘
         ▲                                    ▲
         └────────────┬───────────────────────┘
                      │
              ┌───────┴───────┐
              │  USB Cameras  │
              │(640×480@30fps)│
              └───────────────┘
```

### Technical Chain

```
Voice Command → Local ASR → Rule Engine Semantic Matching → YOLO Visual Detection → ACT End-to-End Inference → Edge Action Generation
```

### Software Modules

| Module | Tech Stack | Function |
|--------|------------|----------|
| Multimodal Perception | PyAudio + Whisper.cpp + OpenCV | Audio capture, ASR inference, visual acquisition, multimodal alignment |
| VLA Policy Inference | LeRobot + ACT + ResNet-18 | Visual encoding, conditional embedding, CVAE action chunking |
| Real-Time Control | Python Multithreading + Serial | Action interpolation, joint limits, emergency stop, homing |

---

## 🚀 Quick Start

### Prerequisites

- **Controller**: DiRobot RDK X5 (Horizon Journey series chip with built-in BPU)
- **OS**: Linux (Ubuntu 22.04 or Horizon customized system)
- **Python**: ≥ 3.8
- **CUDA**: NVIDIA GPU required for training (inference runs purely on BPU)

### 1. Clone Repository

```bash
git clone https://github.com/your-org/edge-act-robot.git
cd edge-act-robot
```

### 2. Install Dependencies

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate

# Install core dependencies
pip install -r requirements.txt

# Install LeRobot (training side)
pip install -e ./third_party/lerobot

# Install Horizon OE toolchain (deployment side, RDK X5)
# Please refer to Horizon official docs: https://developer.d-robotics.cc/
```

### 3. Data Collection (Training Side)

```bash
# Start teleoperation collection
python scripts/teleop_collect.py   --robot-type so100   --cameras front top   --output-dir data/teleop_demo   --num-episodes 100
```

### 4. Model Training (Training Side)

```bash
# ACT policy training
python scripts/train_act.py   --dataset-dir data/teleop_demo   --config configs/act_so100.yaml   --output-dir checkpoints/act_so100   --batch-size 64   --num-steps 50000
```

### 5. Model Conversion & Deployment (RDK X5)

```bash
# 1. PyTorch → ONNX
python scripts/export_onnx.py   --checkpoint checkpoints/act_so100/best.pt   --output models/act_so100.onnx

# 2. ONNX → Horizon BPU Model (INT8 Quantization)
python scripts/convert_bpu.py   --onnx models/act_so100.onnx   --calib-data data/calib/   --output models/act_so100_bpu.bin   --quantization int8

# 3. Deploy & Run
python scripts/deploy.py   --act-model models/act_so100_bpu.bin   --yolo-model models/yolo_reagent_bpu.bin   --asr-model models/whisper_tiny.bin   --robot-port /dev/ttyUSB0
```

### 6. Launch System

```bash
# One-click full pipeline startup
python main.py --config configs/system.yaml
```

---

## 🔧 Hardware BOM

| Component | Model/Spec | Qty | Est. Price (USD) |
|-----------|------------|-----|------------------|
| Edge Computer | DiRobot RDK X5 (with BPU) | 1 | ~$170 |
| Desktop Robot Arm | 6-DoF Servo Arm Kit | 1 | ~$55 |
| USB Cameras | Industrial Camera 640×480@30fps | 2 | ~$28 |
| Microphone Array | USB Omnidirectional Mic | 1 | ~$7 |
| Structural Parts | 3D Printed Base + Camera Mount (PETG) | 1 set | ~$14 |
| Power Supply | 12V/5A DC Adapter + Regulator | 1 | - |
| **Total** | | | **~$274** |

> ⚠️ **Note**: It is recommended to add a 470μF electrolytic capacitor and TVS diode at the power input to prevent controller resets caused by ripple when the robot arm starts.

---

## 📊 Performance Metrics

| Metric | Target | Measured |
|--------|--------|----------|
| End-to-End System Latency | ≤2s | ~1.8s |
| Edge Pure Inference Latency | ≤0.05s | ~35ms |
| ACT Inference Frequency | 50 Hz | 50 Hz |
| Control Loop Frequency | 10 Hz | 10 Hz |
| Task Success Rate | ≥80% | **85%** (single pickup) / **82%** (sequential transfer) |
| Voice Command Recognition Accuracy | ≥95% | **92%** |
| Voice Processing Latency | ≤220ms | ≤220ms |
| Training Data Volume | 50-100 trajectories | 50-100 trajectories |
| Scene Adaptation Training Time | 2-4 hours | 2-4 hours |
| Total System Cost | ≤$280 | **~$274** |

---

## 📁 Project Structure

```
edge-act-robot/
├── configs/                    # Configuration files
│   ├── act_so100.yaml          # ACT model config
│   ├── system.yaml             # Full pipeline config
│   └── yolo_reagent.yaml       # YOLO detection config
├── data/                       # Data directory
│   ├── teleop_demo/            # Demonstration trajectories
│   └── calib/                  # BPU calibration data
├── models/                       # Model files
│   ├── act_so100_bpu.bin       # ACT BPU model
│   ├── yolo_reagent_bpu.bin    # YOLO BPU model
│   └── whisper_tiny.bin        # ASR model
├── src/                          # Source code
│   ├── perception/             # Perception module
│   │   ├── audio_capture.py    # Audio capture
│   │   ├── asr_engine.py       # ASR inference
│   │   ├── yolo_detector.py    # YOLO target detection
│   │   └── multimodal_align.py # Multimodal alignment
│   ├── policy/                 # Policy module
│   │   ├── act_model.py        # ACT model definition
│   │   ├── visual_encoder.py   # ResNet-18 visual encoder
│   │   └── action_chunker.py   # CVAE action chunking
│   ├── control/                # Control module
│   │   ├── interpolator.py     # Action interpolation
│   │   ├── robot_interface.py  # Robot communication
│   │   └── safety_monitor.py   # Safety monitoring
│   └── utils/                  # Utilities
│       ├── bpu_runtime.py      # BPU inference wrapper
│       └── logger.py           # Logging
├── scripts/                      # Scripts
│   ├── teleop_collect.py       # Teleoperation collection
│   ├── train_act.py            # ACT training
│   ├── export_onnx.py          # ONNX export
│   ├── convert_bpu.py          # BPU model conversion
│   └── deploy.py               # Deployment script
├── third_party/                  # Third-party dependencies
│   └── lerobot/                # LeRobot framework
├── docs/                         # Documentation
│   └── images/                 # Image assets
├── tests/                        # Unit tests
├── requirements.txt              # Python dependencies
├── main.py                       # System main entry
└── README.md                     # This file
```

---

## 🎯 Demo Scenarios

### Single Object Pickup
> Voice: "Pass me reagent A"
> 
> System recognizes target → YOLO locates → ACT plans trajectory → Arm executes grasp → Places at designated position
> 
> **Success Rate: 85%**

### Multi-Object Sequential Transfer
> Voice: "First pass the centrifuge tube, then transfer sample B"
> 
> Rule engine parses sequence → Executes multi-step operations sequentially
> 
> **Success Rate: 82%**

### Variable Height Placement
> Voice: "Place the centrifuge tube on the second rack layer"
> 
> YOLO recognizes target height → ACT adaptively adjusts end-effector pose
> 
> **Success Rate: 80%**

---

## 🔬 Technical Highlights

### 1. End-to-End ACT Imitation Learning Architecture
- Single ACT network replaces the traditional "detection → parsing → planning → execution" multi-module pipeline
- Unifies visual perception, target conditioning, and action generation in one end-to-end optimized network
- Codebase reduced by approximately **40%** compared to traditional solutions

### 2. Edge-End BPU Deep Adaptation
- Manual replacement of LayerNorm/GELU operators in CVAE encoder with BPU-compatible InstanceNorm/ReLU approximations
- INT8 quantization employs per-channel calibration and sensitive-layer skipping, keeping accuracy loss within **2%**
- ACT model compressed to **30%** of original size, YOLO to **25%**

### 3. High-Frequency Action Chunking Control
- ACT single inference outputs a future 100-step action sequence (covering ~2 seconds)
- Linear interpolation maps to 10Hz robot joint commands, ensuring motion smoothness
- Effectively masks edge-end single-step inference latency

---

## 🛣️ Roadmap

- [x] Single-arm end-to-end ACT control
- [x] Offline voice command interaction
- [x] YOLO target conditional injection
- [x] BPU INT8 quantized deployment
- [ ] Multi-arm collaborative control
- [ ] Force/tactile sensor integration
- [ ] Deep behavior prediction & memory learning
- [ ] Cloud federated learning optimization
- [ ] Mobile remote monitoring APP

---

## 🤝 Contributing

Issues and PRs are welcome! Please ensure:

1. Code follows PEP8 style guidelines
2. New features include unit tests
3. Run `pytest tests/` before submitting
4. Update relevant documentation

---

## 📄 License

This project is licensed under the [Apache License 2.0](LICENSE).

---

## 🙏 Acknowledgements

- [LeRobot](https://github.com/huggingface/lerobot) — Open-source robot learning framework
- [DiRobot](https://developer.d-robotics.cc/) — RDK X5 edge computing platform
- [Horizon Robotics](https://www.horizon.ai/) — Journey series BPU acceleration chips
- [Whisper.cpp](https://github.com/ggerganov/whisper.cpp) — Local ASR inference

---

<p align="center">
  <sub>Built with ❤️ for the 2026 ACT Embedded Systems Competition</sub>
</p>
