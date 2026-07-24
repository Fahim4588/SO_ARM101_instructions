# LeRobot SO100/SO101 Leader-Follower Arm Setup

Setup and operation notes for a leader-follower robotic arm pair using [LeRobot](https://github.com/Seeed-Projects/lerobot).

**Roles:**
- `leader -> 0`
- `follower -> 1`

**Reference:**
[SO-101M LeRobot Wiki (Seeed Studio)](https://wiki.seeedstudio.com/lerobot_so100m_new/#teleoperate)
[so-101 Hugging Face community](https://huggingface.co/docs/lerobot/v0.6.0/en/so101)

---

## Requirements

### Ubuntu x86
- Ubuntu 22.04
- CUDA 12+
- Python 3.10
- Torch 2.6+

### Jetson Orin
- JetPack 6.0 or 6.1 (**JetPack 6.2 is not yet supported**)
- Python 3.10
- Torch 2.3+

---

## Installation (Ubuntu 22.04)

### 1. Install Miniforge
```bash
wget https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh
chmod +x Miniforge3-Linux-x86_64.sh
./Miniforge3-Linux-x86_64.sh

# Once installation is complete:
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

### 5. Install Build Tools (Optional)

On a freshly configured Ubuntu system, you may need to install GCC and Python build dependencies. If the required libraries are already installed, you can skip this step.

```bash
sudo apt update
sudo apt install build-essential
```

### 6. Install LeRobot with Feetech motor support
```bash
cd ~/lerobot && pip install -e ".[feetech]"
```

> **At this point, installation should be complete.**

---

## Motor Setup

### Find follower servo numbers
```bash
lerobot-setup-motors \
    --robot.type=so101_follower \
    --robot.port=/dev/ttyUSB1  # <- paste the port found in the previous step
```

### Find leader servo numbers
```bash
lerobot-setup-motors \
    --teleop.type=so101_leader \
    --teleop.port=/dev/ttyUSB0  # <- paste the port found in the previous step
```

> **Note:** This step used `--robot.type=so101_follower` in both branches of the original merge conflict, which looks like a copy-paste mistake since this command is for identifying the *leader* arm's servos. It's been corrected here to `--teleop.type=so101_leader`. Double-check this against your own setup before running it.

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

## Clearing Old Calibration Data (use this if you want to recalibrate)

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

## Running Teleoperation

Give the ports read/write permission before running:
```bash
sudo chmod 666 /dev/ttyACM*
```

Run teleoperation:
```bash
lerobot-teleoperate \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM1 \
  --robot.id=my_awesome_follower_arm \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM0 \
  --teleop.id=my_awesome_leader_arm
```

### Running from a new terminal
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
> **Note:** You do **not** need `conda init` or to source anything again in a new terminal — just activate the environment and run.

---

## Troubleshooting Helper Scripts

**If calibration doesn't take effect even after deleting old calibration files and creating a new one**, check:
1. **Wrong port** — double-check the port name (`/dev/ttyUSB*` or `/dev/ttyACM*`).
2. **Loose connection** — try unplugging and reconnecting the USB and electrical (power) connections; this often resolves it.

**Permission errors on the serial port:**
```bash
sudo chmod 666 /dev/ttyACM*
```
This must be run before the teleoperate command each session — it grants read/write permission on the port.

### Disable servo torque (to move joints by hand)
If you're unable to move a joint by hand to set it to the middle position before calibration:
```bash
python -m src.tools.servo_disable
```
This disables servo torque so the joints can be moved freely by hand.

### Set a custom middle (offset) position
To define whatever position the arm is currently in as the "middle"/offset position for calibration:
```bash
python -m src.tools.servo_middle_calibration
```
Whatever position the arm is in when this command is run becomes the new middle/offset.

### Verify the middle (offset) position
To check that the new middle/offset position was set correctly:
```bash
python -m src.tools.servo_center_test
```
If the middle position was set correctly, running this command will make the arm automatically move to that position.

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

## Finding Ports

```bash
ls /dev/ttyACM*
lerobot-find-port   # (not yet tested — need to check what exactly this does)
```
