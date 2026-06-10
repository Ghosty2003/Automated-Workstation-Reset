# Automated Workstation Reset

A bimanual SO-101 robotic system that detects loose tools on a makerspace workstation and returns them to designated bins — automating the "Sort" step of industrial 5S.

**Course:** TECHIN 517 · Spring 2026
**Team:** Suzy Hong · Chenming Ge · Lya Liu

---

## Demo

[Video demo — Google Drive link to be added](#)

Five tool types handled: **screwdriver, plier, tape, pen, scissor**. Two SO-101 arms split the workspace left/right; perception runs YOLOv8m on a RealSense D435 stream; planning combines MoveIt IK with per-arm Rosetta policy clients (ACT-trained) for the grasp; placement is per-tool MoveIt trajectory to the matching rack slot.

---

## System architecture

Three-stage software pipeline on top of a bimanual hardware setup.

- **Stage A · Perception** — YOLOv8m (fine-tuned on 5 tool types) via `aruco_tracker_ros2` + `ros2_handeye_calibration` for camera ↔ arm frame
- **Stage B · Planning** — MoveIt IK (hover pose) → Rosetta-served ACT policy (grasp refinement) → per-tool MoveIt placement trajectory
- **Stage C · Execution** — `bi_soa_moveit_bringup` + per-arm `grasp_state_machine`, orchestrated by `grasp_sequencer`, with hardcoded workspace zoning and pause-on-cross collision avoidance

Each cycle ends in one of three states: **Pick & place (happy path)**, **Stop-on-Cross (pause then resume)**, or **Skip & Retry (detection miss → loop back to perception)**.

Grasping priority (set in `bi_grasp_pipeline.launch.py`, override with `left_sequence:=` / `right_sequence:=`):
- Left arm: `tape → pen → plier`
- Right arm: `screwdriver → scissor`

---

## Quantitative results

**Evaluation methodology**
- **Success (binary):** object correctly placed in the designated bin and the full pipeline completes. Fail = wrong box, dropped, or timeout (>60 s).
- **3 conditions × 10 trials = 30+ runs**
  - **A** — 1–2 identical objects, randomly placed
  - **B** — 3–5 mixed / repeated objects, randomly placed
  - **C** — 5 random objects + clutter (paper scraps, debris)

### Success rate by condition and tool

| Tool        | Cond A | Cond B | Cond C |
|-------------|--------|--------|--------|
| Screwdriver | 80%    | 80%    | 60%    |
| Plier       | 0%     | 0%     | 0%     |
| Tape        | 90%    | 70%    | 50%    |
| Pen         | 80%    | 20%    | 10%    |
| Scissor     | 50%    | 60%    | 40%    |
| **Average** | **60%**| **46%**| **32%**|

> Plier consistently fails — handle geometry causes the gripper to slip before lift. Pen drops sharply A→B because grasp position has to be precise and gets confused under multi-object scenes. Scissor and screwdriver hold up well even in clutter.

### Cycle time · range (s)

| Stage          | Cond A   | Cond B   | Cond C   |
|----------------|----------|----------|----------|
| Hover          | 7–9      | 9–12     | 9–14     |
| Pickup         | 20       | 20       | 20       |
| Placement      | 20       | 20       | 20       |
| **Full cycle** | **47–90**| **56–115**| **65–135**|
| Attempts/trial | 1–3      | 3–6      | 5–8      |

Lower bound = sum of stages + minimal overhead; upper bound = failed attempts hitting the 60 s timeout or YOLO re-detecting under clutter.

### Failure modes — 80 fails / 130 attempts

| Mode | Share | Count | What it looks like |
|---|---|---|---|
| Pickup miss | 50% | 40 | Grasp slips before the lift completes (loose grip or wrong grasp pose) |
| Drop | 25% | 20 | Object lost mid-trajectory between hover and placement |
| Launch fail | 15% | 12 | Dual-arm launch instability — opposing arm fires unexpectedly |
| Wrong-box placement | 10% | 8 | Released into the wrong rack slot — mostly Cond C |

Raw trial data is committed alongside the report as `left_*.csv` / `right_*.csv` (per-arm, per-tool home poses) and the trial log.

---

## Setup

The repo ships a `.devcontainer/` so you can reproduce the environment without polluting your host.

### Prerequisites

- Docker + VSCode with the **Dev Containers** extension
- CUDA-capable GPU (we used an RTX 5090 for training; an RTX 3060+ is enough for inference)
- Bimanual SO-101 follower + leader arms, RealSense D435, ArUco calibration cube

### Open in the devcontainer

```bash
git clone https://github.com/Ghosty2003/Techin517.git
cd Techin517
code .                       # then: "Reopen in Container"
```

The container builds from `docker/Dockerfile` (CUDA 12.4.1 · Ubuntu 22.04 · Python 3.10), mounts the repo at `/home/ubuntu/techin517`, persists HuggingFace cache to `huggingface/`, and runs `docker/setup.sh` on first start (installs LeRobot, applies USB latency / V4L2 fixes, builds the ROS workspace).

### Build the ROS 2 workspace (re-run after pulls)

```bash
cd ~/techin517 && source ros2_ws/install/setup.bash
bash ros2_ws/src/soa_ros2/build.sh
```

### Pre-trained models

Models live under `~/techin517/outputs/train/`. Download from the team's HuggingFace org:

| Policy | HF path | Local path |
|---|---|---|
| Right arm — scissors | `<HuggingFace repo>` | `outputs/train/pick_up_right_scissors/checkpoints/100000/pretrained_model` |
| Left arm — pen | `<HuggingFace repo>` | `outputs/train/pick_up_left_scissors/checkpoints/100000/pretrained_model` |
| YOLOv8m — 5-tool detector | `<HuggingFace repo>` | `outputs/yolo/best.pt` |

The HuggingFace cache is symlinked into `huggingface/` via `docker/setup.sh` so models persist across container restarts.

### Hardware checklist (every session)

```bash
ls -la /dev/ttyACM* /dev/video*              # confirm arms + cameras enumerated
sudo chmod 666 /dev/ttyACM* /dev/video*      # permissions reset on every replug
# If /dev/ttyACM* numbers shifted, update soa_params.yaml and rebuild soa_bringup
```

---

## Usage

The full pipeline runs in three terminals — MoveIt bringup, two Rosetta policy clients, then the grasp pipeline. Replace `<HF_USER>` and policy checkpoint paths with your own.

### 1. Calibrate the arms (one-time per arm)

```bash
lerobot-calibrate \
    --robot.type=so101_follower \
    --robot.port=/dev/ttyACM0 \
    --robot.id=Kid_right

lerobot-calibrate \
    --teleop.type=so101_leader \
    --teleop.port=/dev/ttyACM1 \
    --teleop.id=Mom_right
```

Reference pose: `assets/so101_calibration_L-pose.png`. Calibration JSONs land in `huggingface/lerobot/calibration/robots/` and `.../teleoperators/`.

### 2. Hand-eye calibration (one-time per camera mount)

```bash
# Terminal 1 — ArUco cube tracker
ros2 launch aruco_tracker_ros2 aruco_cube_tracker.launch.py

# Terminal 2 — eye-on-base calibration for the right arm
ros2 launch hand_eye_calibration calibration.launch.py \
    calibration_type:=eye-on-base \
    robot_base_frame:=follower/right_base_link \
    robot_effector_frame:=follower/right_gripper_frame_link
```

Repeat with `follower/left_base_link` / `follower/left_gripper_frame_link` for the left arm.

### 3. Launch the dual-arm MoveIt + hardware stack

```bash
ros2 launch bi_soa_moveit_config bi_soa_moveit_bringup.launch.py cameras:=true
```

This brings up both arms, controllers, TF tree, MoveGroup, and the RealSense camera node. **Always start this first** — `bi_grasp_pipeline.launch.py` assumes it is already running.

### 4. Start the per-arm Rosetta policy clients

```bash
# Right arm — port 8080
ros2 launch rosetta rosetta_client_launch.py \
    node_name:=rosetta_client_right \
    contract_path:=$HOME/techin517/ros2_ws/src/soa_ros2/soa_bringup/rosetta_contracts/bi_soa_right_arm_contract.yaml \
    pretrained_name_or_path:=$HOME/techin517/outputs/train/pick_up_right_scissors/checkpoints/100000/pretrained_model \
    server_address:=127.0.0.1:8080

# Left arm — port 8081
ros2 launch rosetta rosetta_client_launch.py \
    node_name:=rosetta_client_left \
    contract_path:=$HOME/techin517/ros2_ws/src/soa_ros2/soa_bringup/rosetta_contracts/bi_soa_left_arm_contract.yaml \
    pretrained_name_or_path:=$HOME/techin517/outputs/train/pick_up_left_scissors/checkpoints/100000/pretrained_model \
    server_address:=127.0.0.1:8081
```

### 5. Run the autonomous grasp pipeline

```bash
ros2 launch soa_bringup bi_grasp_pipeline.launch.py
```

This spawns, per arm: `move_to_pose_server`, `gripper_server`, `grasp_state_machine` (IDLE → GRASPING → IDLE). The single `grasp_sequencer` node listens to YOLO detections and dispatches per-object grasps using the per-arm priority sequence and the `left_*.csv` / `right_*.csv` home-pose tables.

Useful overrides:

```bash
# Skip the home-return go_to_poses between grasps (faster, less safe)
ros2 launch soa_bringup bi_grasp_pipeline.launch.py run_go_to_poses:=false

# Override checkpoint paths or grasp sequences inline
ros2 launch soa_bringup bi_grasp_pipeline.launch.py \
    right_model:=/path/to/right_model \
    scissor_csv:=$HOME/techin517/right_scissor.csv \
    left_sequence:='["tape","pen"]'
```

### 6. Recording / teleop (for re-training)

```bash
ros2 launch soa_bringup bi_soa_bringup.launch.py leader:=true cameras:=true display:=true
```

Then use `lerobot-record` against the bi-arm setup to log demonstrations to `huggingface/lerobot/project/bi_so101_test`.

---

## Repo layout

```
.
├── .devcontainer/                          # devcontainer.json (mounts repo to /home/ubuntu/techin517)
├── docker/
│   ├── Dockerfile                          # CUDA 12.4.1 + Ubuntu 22.04 + Python 3.10 + LeRobot + ROS 2 Humble
│   ├── setup.sh                            # post-start: HF cache symlink, USB latency, LeRobot install
│   └── bash_history.txt                    # mounted as ~/.bash_history inside the container
├── ros2_ws/src/
│   ├── soa_ros2/
│   │   ├── soa_bringup/                    # launch files + rosetta_contracts (bi_grasp_pipeline.launch.py, bi_soa_bringup.launch.py)
│   │   ├── soa_moveit_config/
│   │   │   ├── soa_moveit_config/          # single-arm MoveIt
│   │   │   └── bi_soa_moveit_config/       # bimanual MoveIt
│   │   ├── soa_apps/                       # top-level apps (grasp_sequencer, grasp_state_machine, hover_to_object, …)
│   │   ├── soa_functions/                  # action servers (move_to_pose_server, gripper_server, …)
│   │   ├── soa_interfaces/                 # action + srv definitions
│   │   ├── soa_description/                # URDF + meshes
│   │   ├── soa_teleop/                     # teleop nodes
│   │   └── feetech_ros2_driver/            # motor driver
│   ├── rosetta/                            # contract-based policy execution framework
│   ├── rosetta_interfaces/
│   ├── lerobot-robot-rosetta/              # robot bridge
│   ├── lerobot-teleoperator-rosetta/       # teleop bridge
│   ├── aruco_tracker_ros2/                 # ArUco cube tracker for hand-eye calibration
│   ├── ros2_handeye_calibration/           # hand-eye calibration
│   ├── yolo_ros/                           # YOLO ROS 2 integration (GPL-3.0)
│   ├── pymoveit2/                          # Python MoveIt 2 wrapper
│   └── warehouse_ros_mongo/                # MoveIt warehouse backend
├── third_party/lerobot/                    # vendored LeRobot framework
├── huggingface/
│   ├── lerobot/calibration/                # arm + teleop calibration JSONs
│   └── lerobot/project/bi_so101_test/      # recorded demonstrations
├── assets/so101_calibration_L-pose.png     # arm calibration reference pose
├── left_pen.csv / left_plier.csv / left_tape.csv     # left-arm per-tool home poses
├── right_scissor.csv / right_screwdriver.csv         # right-arm per-tool home poses
├── soa_ws/joints.csv                       # joint-state log
├── LICENSE                                 # Apache-2.0 (team code)
├── THIRD_PARTY_LICENSES                    # third-party component licenses
└── README.md
```

---

## Known limitations

- **Dual-arm launch is not yet stable** — still iterating on namespaces + TF prefixes. The "Launch fail" failure mode (15% of fails) reflects this.
- **Plier (0% success)** — gripper geometry can't catch the handle reliably. Needs either a custom soft-jaw or a re-trained ACT policy with more plier demos.
- **Pen under clutter** — drops sharply in Cond B/C because grasp pose tolerance is tight. Possible fix: tighter YOLO bbox + closer-grip ACT policy.

---

## License

This project's code is released under **Apache-2.0** — see [LICENSE](LICENSE).

Third-party components ship under their own licenses; the full list with copyright notices lives in [THIRD_PARTY_LICENSES](THIRD_PARTY_LICENSES). Notable:

- **Apache-2.0:** `lerobot`, `rosetta`, `lerobot_robot_rosetta`, `lerobot_teleoperator_rosetta`, `aruco_tracker_ros2`, `soa_ros2`, ROS 2 core, OpenCV, HuggingFace libs, `rerun-sdk`, `draccus`, `packaging`
- **BSD-2-Clause / BSD-3-Clause:** `pymoveit2`, `warehouse_ros_mongo`, `feetech_ros2_driver`, `soa_moveit_config`, PyTorch / torchvision / torchcodec, NumPy, `pyserial`, PyAV, `imageio`, `lap`, `isaaclab` / `isaaclab_rl`
- **MIT:** `lerobot` (dual-licensed), `gymnasium`, `einops`, `wandb`, `termcolor`, `deepdiff`, `jsonlines`, `setuptools`
- **GPL-3.0:** `yolo_ros` — copyleft, see warning below
- **AGPL-3.0:** `ultralytics` (YOLOv8) — copyleft + network-use clause, see warning below
- **LGPL-3.0:** `pynput`

> ⚠️ **Copyleft notice:** `yolo_ros` (GPL-3.0) and `ultralytics` (AGPL-3.0) carry copyleft terms. If you redistribute this software or run it as a networked service, those licenses may apply to the combined work. Consult a legal professional before redistribution.

---

## Acknowledgements

- TECHIN 517 teaching team — for the SO-101 hardware, the 5090 desktops, and the Rosetta framework
- [LeRobot](https://github.com/huggingface/lerobot) — dataset + policy training pipeline
- [pymoveit2](https://github.com/AndrejOrsula/pymoveit2) — MoveIt 2 Python interface
- [Ultralytics YOLOv8](https://github.com/ultralytics/ultralytics) — detector backbone
