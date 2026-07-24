# LeRobot SO-101 Setup & Teleoperation Guide

Notes for setting up and running teleoperation with the SO-101 leader/follower arms using LeRobot.

**Roles:**
- `leader -> 0`
- `follower -> 1`

**Reference:** <br>[SO-101M LeRobot Wiki (Seeed Studio)](https://wiki.seeedstudio.com/lerobot_so100m_new/#teleoperate)<br>
               [so-101 Hugging Face community](https://huggingface.co/docs/lerobot/v0.6.0/en/so101)

---

## Requirements

**Ubuntu x86:**
- Ubuntu 22.04
- CUDA 12+
- Python 3.10
- Torch 2.6+

**Jetson Orin:**
- JetPack 6.0 or 6.1 (JetPack 6.2 not yet supported)
- Python 3.10
- Torch 2.3+

---

## Installation (Ubuntu 22.04)

### 1. Install Miniforge
```bash
wget https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh
chmod +x Miniforge3-Linux-x86_64.sh
./Miniforge3-Linux-x86_64.sh
# Once the installation is complete:
source ~/.bashrc
# Initialize all shells
conda init --all
```

### 2. Create and activate a conda environment
```bash
conda create -y -n lerobot python=3.10 && conda activate lerobot
```

### 3. Clone LeRobot
```bash
git clone https://github.com/Seeed-Projects/lerobot.git ~/lerobot
```

### 4. Install ffmpeg (via Miniforge)
```bash
conda install ffmpeg -c conda-forge
```

### 5. Install build tools
On a freshly configured Ubuntu system, gcc and Python build libraries are needed:
```bash
sudo apt update
sudo apt install build-essential
```

### 6. Install LeRobot with Feetech motor dependencies
```bash
cd ~/lerobot && pip install -e ".[feetech]"
```

> ✅ At this point, installation should be complete.

---

## Motor Setup

**Find follower servo numbers:**
```bash
lerobot-setup-motors \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyUSB1  # <- paste the port found in the previous step
```

**Find leader servo numbers:**
```bash
lerobot-setup-motors \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyUSB0  # <- paste the port found in the previous step
```

---

## Clearing Old Calibration Data

**Delete follower calibration:**
```bash
find ~/.cache -type f -name "*my_awesome_follower_arm*" -delete
find ~/.local -type f -name "*my_awesome_follower_arm*" -delete
find ~ -type f -name "*my_awesome_follower_arm*" 2>/dev/null
```

**Delete leader calibration:**
```bash
find ~/.cache -type f -name "*my_awesome_leader_arm*" -delete
find ~/.local -type f -name "*my_awesome_leader_arm*" -delete
find ~ -type f -name "*my_awesome_leader_arm*" 2>/dev/null
```

---

## Calibration

**Set follower calibration:**
```bash
lerobot-calibrate \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyUSB1 \
  --robot.id=my_awesome_follower_arm
```

**Set leader calibration:**
```bash
lerobot-calibrate \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM0 \
  --teleop.id=my_awesome_leader_arm
```

> ⚠️ **Important:** During calibration, all servos must be positioned at their middle point before you press Enter to start. The reason: whatever angle each follower servo sits at during calibration, the leader's corresponding servo must be set to that same angle. So when the calibration command prompts you to press Enter, first move every servo to its middle position, *then* press Enter to begin calibration.

---

## Running Teleoperation

Give read/write permission to the ports first:
```bash
sudo chmod 666 /dev/ttyACM*
```

Then run:
```bash
lerobot-teleoperate \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM1 \
  --robot.id=my_awesome_follower_arm \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM0 \
  --teleop.id=my_awesome_leader_arm
```

---

## Running From a New Terminal

If you open a new terminal later, you don't need to run `conda init` or source anything again — just:

```bash
conda activate lerobot
cd ~/lerobot
sudo chmod 666 /dev/ttyACM*

lerobot-teleoperate \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM1 \
  --robot.id=my_awesome_follower_arm \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM0 \
  --teleop.id=my_awesome_leader_arm
```

---

## Troubleshooting

**If calibration doesn't take effect even after deleting old calibration files and creating a new one**, check:
1. **Wrong port** — double-check the port name (`/dev/ttyUSB*` or `/dev/ttyACM*`).
2. **Loose connection** — try unplugging and reconnecting the USB and electrical (power) connections; this often resolves it.

**Permission errors on the serial port:**
```bash
sudo chmod 666 /dev/ttyACM*
```
This must be run before the teleoperate command each session — it grants read/write permission on the port.

---

## Viewing Saved Calibration Data

**Follower:**
```bash
cat ~/.cache/huggingface/lerobot/calibration/robots/so_follower/my_awesome_follower_arm.json
```

**Leader:**
```bash
cat ~/.cache/huggingface/lerobot/calibration/teleoperators/so_leader/my_awesome_leader_arm.json
```

---

## Finding the Port

```bash
ls /dev/ttyACM*
```

There's also a `lerobot-find-port` command — not yet tested, need to check exactly what it does.
