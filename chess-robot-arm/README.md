# ♟️ Chess-Playing Robotic Arm

A fully autonomous chess-playing robotic arm that detects human moves using computer vision, computes responses using Stockfish, and physically executes moves via a 5-DOF servo arm — all coordinated through a ROS 2 pipeline.

> **"From YOLO to absdiff, from WebSocket to ROS topics — built iteratively, shipped working."**

---

## 📽️ Demo

<!-- Add your demo video here -->
> 🎥 Demo video coming soon — arm executing a Sicilian Defence response

---

## 🧠 System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     FULL PIPELINE                       │
│                                                         │
│  📷 Camera                                              │
│     │  (IP Webcam over HTTP)                            │
│     ▼                                                   │
│  chess_vision.py                                        │
│     │  absdiff → detect human move (e.g. e2e4)         │
│     │  publishes → ROS /white_move topic                │
│     ▼                                                   │
│  stockfish_node  (ROS 2 Node)                           │
│     │  validates move, computes best response           │
│     │  publishes → ROS /black_move topic                │
│     ▼                                                   │
│  robot_arm_node  (ROS 2 Node)                           │
│     │  loads angles from chess_arm_calib.json           │
│     │  executes state machine: PICK→LIFT→MOVE→PLACE     │
│     │  sends S<pin>:<pulse> over USB serial             │
│     ▼                                                   │
│  Raspberry Pi Pico                                      │
│     │  50Hz PWM → MG995 × 3, SG90 × 3                  │
│     ▼                                                   │
│  🦾 Robotic Arm moves the piece                        │
└─────────────────────────────────────────────────────────┘
```

---

## 🔄 Project Evolution

This project went through **3 distinct versions** — each teaching something new.

### V1 — ROS 2 + YOLO Vision (`v1_ros2_initial/`)
**What we built:** Full ROS 2 architecture with YOLO-based piece detection, Stockfish node, and a mock arm node with a state machine.

**What we learned:** YOLO was too sensitive to lighting conditions on a physical chessboard. Detection was inconsistent across different ambient light levels. The ROS architecture itself was solid — just the vision layer needed replacing.

**Key achievement:** Proved the modular ROS 2 pub/sub design worked end to end.

---

### V2 — Working System with absdiff + WebSocket (`v2_working/`)
**What we built:** Replaced YOLO with OpenCV `absdiff` frame differencing. Replaced ROS with a simpler WebSocket bridge. A Flutter/HTML calibration page saved servo angles for all 64 squares to a JSON file.

**What we learned:** `absdiff` is deterministic and lighting-robust when the camera is fixed. The WebSocket → HTML page pathway was unreliable — the page had to be open and connected at all times.

**Key achievement:** First time the arm physically moved a chess piece correctly. Fully calibrated all 64 squares.

---

### V3 — ROS 2 Final (`v3_ros2_final/`)
**What we built:** Brought ROS 2 back as the communication backbone, replacing the flaky WebSocket layer entirely. `chess_vision.py` now publishes directly to `/white_move`. The arm node reads angles straight from the calibration JSON and drives the Pico over serial — no browser, no HTML page needed.

**What we learned:** ROS 2 pub/sub is exactly the right tool for this kind of multi-node robotics pipeline. Reliable, decoupled, and clean.

**Key achievement:** Full end-to-end pipeline running on Linux with 3 independent ROS nodes.

---

## 🛠️ Hardware

| Component | Part | Role |
|---|---|---|
| Arm joints (×3) | MG995 servo | Base, Shoulder, Elbow |
| Wrist joints (×3) | SG90 servo | Wrist Pitch, Wrist Roll, Gripper |
| Microcontroller | Raspberry Pi Pico | 50Hz PWM via USB serial |
| Camera | Android phone (IP Webcam app) | Overhead board capture |
| Chess engine | Stockfish 14+ | Move computation |

---

## 📁 Repository Structure

```
chess-robot-arm/
│
├── README.md
├── demo/                        # Videos and screenshots
│
├── v1_ros2_initial/             # Version 1 — ROS 2 + YOLO attempt
│   └── chess_ros_pkg/
│       ├── stockfish_node.py
│       ├── robot_arm_node.py    # Mock state machine
│       └── bridge_node.py
│
├── v2_working/                  # Version 2 — Working system
│   ├── chess_vision.py          # absdiff + Stockfish + WebSocket
│   ├── pc_serial_bridge.py      # WebSocket hub
│   ├── chess_arm_calib.json     # 64-square servo calibration
│   └── pico_servo_listener.py   # Pico firmware
│
├── v3_ros2_final/               # Version 3 — ROS 2 final
│   ├── chess_vision.py          # ROS publisher version
│   └── chess_ros_pkg/
│       ├── stockfish_node.py    # Unchanged — was always solid
│       └── robot_arm_node.py    # Now reads JSON + drives Pico
│
└── docs/
    └── architecture.md
```

---

## 🚀 Running the System (V3)

### Prerequisites
```bash
# ROS 2 Humble
sudo apt install ros-humble-desktop

# Python deps
pip install opencv-python numpy chess requests pyserial

# Stockfish
sudo apt install stockfish
```

### Launch

**Terminal 1 — Stockfish Node**
```bash
source ~/ros2_ws/install/setup.bash
ros2 run chess_ros_pkg stockfish_node
```

**Terminal 2 — Arm Node**
```bash
source ~/ros2_ws/install/setup.bash
ros2 run chess_ros_pkg robot_arm_node
```

**Terminal 3 — Vision**
```bash
source ~/ros2_ws/install/setup.bash
python3 chess_vision.py
```

### Manual Test (no camera needed)
```bash
ros2 topic pub --once /white_move std_msgs/msg/String "{data: 'e2e4'}"
```

---

## 📐 Calibration

All 64 squares plus special zones (rest, capture) are stored in `chess_arm_calib.json`:

```json
{
  "squares": {
    "e2": {"base":80,"shldr":38,"elbow":45,"wrpitch":36,"wrroll":18,"grip":5},
    "e4": {"base":81,"shldr":54,"elbow":70,"wrpitch":33,"wrroll":18,"grip":5}
  },
  "zones": {
    "rest":    {"base":143,"shldr":110,"elbow":124,"wrpitch":90,"wrroll":17,"grip":5},
    "capture": {"base":153,"shldr":95, "elbow":112,"wrpitch":46,"wrroll":17,"grip":5}
  }
}
```

Each square was manually calibrated by positioning the arm over each of the 64 squares and recording all 6 servo angles.

---

## 🔑 Key Engineering Concepts

- **Layered system design** — vision, engine, and arm are fully independent
- **Deterministic vision** — fixed camera + fixed ROI = consistent detection
- **ROS 2 pub/sub** — decoupled nodes communicate via topics, not function calls
- **State machine** — arm follows IDLE→PICK→LIFT→MOVE→PLACE→HOME
- **Safety-first** — no move executes without Stockfish validation
- **Slew control** — servos move degree-by-degree at configurable speed to avoid jerking

---

## 📊 ROS 2 Topic Map

```
chess_vision_node
    └── publishes → /white_move (std_msgs/String)  e.g. "e2e4"

stockfish_node
    ├── subscribes → /white_move
    └── publishes  → /black_move (std_msgs/String)  e.g. "c7c5"

robot_arm_node
    └── subscribes → /black_move
        └── executes arm sequence via serial → Pico
```

---

## 🧾 One-Line Summary

> A modular chess-playing robotic system using computer vision to detect human moves, ROS 2 for inter-node coordination, Stockfish for decision-making, and a calibrated 5-DOF arm for physical move execution.

---

## 👤 Author

**Adith** — Built as a final year engineering project.

*If you find this useful or interesting, feel free to ⭐ the repo!*
