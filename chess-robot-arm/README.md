<div align="center">

# ♟️ Chess Robot Arm

### *A machine that thinks, sees, and plays chess.*

![Python](https://img.shields.io/badge/Python-3.10-3776AB?style=for-the-badge&logo=python&logoColor=white)
![ROS2](https://img.shields.io/badge/ROS2-Humble-22314E?style=for-the-badge&logo=ros&logoColor=white)
![OpenCV](https://img.shields.io/badge/OpenCV-absdiff-5C3EE8?style=for-the-badge&logo=opencv&logoColor=white)
![Stockfish](https://img.shields.io/badge/Stockfish-14.1-B58863?style=for-the-badge)
![Raspberry Pi](https://img.shields.io/badge/Raspberry_Pi-Pico-A22846?style=for-the-badge&logo=raspberrypi&logoColor=white)

<br/>

> *"Built iteratively. Broke things. Fixed things. Shipped it."*

<br/>

<!-- Add your demo GIF here -->
![Chess Robot Arm](chess-robot-arm/demo/arm.jpg)

</div>

---

## 🧠 What is this?

A fully autonomous **chess-playing robotic arm** that:

- 👁️ **Sees** the board using a phone camera + OpenCV frame differencing
- 🧩 **Understands** the move using `python-chess` for validation
- ♟️ **Thinks** using Stockfish 14 running inside a ROS 2 node
- 🦾 **Moves** a physical 5-DOF servo arm via a Raspberry Pi Pico

No pre-programmed openings. No manual input. Just play — and watch it respond.

---

## 🎬 Demo

<!-- Replace with your actual demo video/gif -->
> 🎥 *Arm executing a Sicilian Defence response — c7 → c5*

---

## 🔄 The Journey — 3 Versions

```
V1 ──────────────────────────────────────────────────────── V3
ROS 2 + YOLO          absdiff + WebSocket         ROS 2 + absdiff
     │                        │                         │
     │  YOLO failed on        │  WebSocket was          │  Clean. Reliable.
     │  inconsistent          │  flaky — HTML page      │  No browser.
     │  lighting              │  had to stay open       │  No WebSocket.
     ▼                        ▼                         ▼
  Scrapped vision         Arm moves!              Full pipeline ✅
  Kept ROS arch ✅        Kept absdiff ✅          Ships on Linux
```

| Version | Vision | Communication | Status |
|---|---|---|---|
| [V1 — ROS 2 Initial](v1_ros2_initial/) | YOLO | ROS 2 topics | ⚠️ Vision abandoned |
| [V2 — Working System](v2_working/) | absdiff | WebSocket + HTML | ✅ Arm moves |
| [V3 — ROS 2 Final](v3_ros2_final/) | absdiff | ROS 2 topics | ✅ Full pipeline |

---

## ⚡ System Pipeline

```
 ┌──────────────┐     HTTP      ┌──────────────────┐
 │ 📱 IP Webcam │ ──────────── ▶│ chess_vision.py  │
 └──────────────┘               │                  │
                                │  absdiff detect  │
                                │  e.g.  e2 → e4   │
                                └────────┬─────────┘
                                         │ /white_move
                                         ▼
                                ┌──────────────────┐
                                │  stockfish_node  │
                                │                  │
                                │  validates move  │
                                │  computes reply  │
                                │  e.g.  c7 → c5   │
                                └────────┬─────────┘
                                         │ /black_move
                                         ▼
                                ┌──────────────────┐
                                │  robot_arm_node  │
                                │                  │
                                │  loads JSON      │
                                │  slews servos    │
                                │  PICK→LIFT→PLACE │
                                └────────┬─────────┘
                                         │ S<pin>:<pulse>
                                         ▼
                                ┌──────────────────┐
                                │  Raspberry Pi    │
                                │     Pico         │
                                │  50Hz PWM out    │
                                └────────┬─────────┘
                                         │
                                         ▼
                                   🦾 Arm moves
```

---

## 🛠️ Hardware

```
5-DOF Robotic Arm
├── Base          MG995  ── GP0
├── Shoulder      MG995  ── GP1
├── Elbow         MG995  ── GP2
├── Wrist Pitch   SG90   ── GP3
├── Wrist Roll    SG90   ── GP4
└── Gripper       SG90   ── GP5
                  │
                  └── Raspberry Pi Pico (USB Serial @ 115200)
```

---

## 🗂️ Repository Structure

```
chess-robot-arm/
│
├── 📄 README.md
│
├── 🎬 demo/                      ← videos & screenshots
│
├── 📁 v1_ros2_initial/           ← ROS 2 + YOLO attempt
│   └── chess_ros_pkg/
│       ├── stockfish_node.py     ← still used in V3 unchanged
│       ├── robot_arm_node.py     ← mock state machine
│       └── bridge_node.py
│
├── 📁 v2_working/                ← first working version
│   ├── chess_vision.py           ← absdiff + WebSocket
│   ├── pc_serial_bridge.py       ← WebSocket hub
│   ├── chess_arm_calib.json      ← 64-square calibration
│   └── pico_servo_listener.py    ← Pico firmware
│
├── 📁 v3_ros2_final/             ← final system
│   ├── chess_vision.py           ← absdiff + ROS publisher
│   ├── chess_arm_calib.json
│   ├── pico_servo_listener.py
│   └── chess_ros_pkg/
│       ├── stockfish_node.py     ← unchanged from V1
│       └── robot_arm_node.py     ← JSON + serial + state machine
│
└── 📁 docs/
    └── architecture.md
```

---

## 🚀 Quick Start (V3)

```bash
# Clone
git clone https://github.com/Adi-gyt/chess-robot-arm.git
cd chess-robot-arm/v3_ros2_final

# Install deps
pip install opencv-python numpy chess requests pyserial
sudo apt install stockfish ros-humble-desktop

# Build ROS package
cd ~/ros2_ws && colcon build --packages-select chess_ros_pkg

# Run — 3 terminals
source ~/ros2_ws/install/setup.bash && ros2 run chess_ros_pkg stockfish_node
source ~/ros2_ws/install/setup.bash && ros2 run chess_ros_pkg robot_arm_node
source ~/ros2_ws/install/setup.bash && python3 chess_vision.py

# Manual test (no camera needed)
ros2 topic pub --once /white_move std_msgs/msg/String "{data: 'e2e4'}"
```

---

## 📐 Calibration Format

Every square on the board has a hand-calibrated servo position:

```json
"e2": { "base":80, "shldr":38, "elbow":45, "wrpitch":36, "wrroll":18, "grip":5 },
"e4": { "base":81, "shldr":54, "elbow":70, "wrpitch":33, "wrroll":18, "grip":5 }
```

Plus special zones:
```json
"rest":    { "base":143, "shldr":110, "elbow":124, "wrpitch":90, "wrroll":17 },
"capture": { "base":153, "shldr":95,  "elbow":112, "wrpitch":46, "wrroll":17 }
```

---

## 🔑 Engineering Highlights

| Concept | Where |
|---|---|
| Deterministic vision via fixed ROI | `chess_vision.py` — absdiff |
| Decoupled nodes via ROS 2 pub/sub | `/white_move` → `/black_move` |
| Safety-first — no unvalidated moves | `stockfish_node.py` |
| State machine with error handling | `robot_arm_node.py` |
| Smooth servo slewing (°/ms control) | `robot_arm_node.py` — slew engine |
| Full calibration in portable JSON | `chess_arm_calib.json` |

---

## 🧾 One Line

> *A modular chess-playing robotic system — vision detects, Stockfish decides, ROS 2 coordinates, the arm executes.*

---

<div align="center">

Made with frustration, iteration, and too much coffee ☕

⭐ **Star this repo if you find it interesting!**

</div>
