# SO101 Pick & Place — SmolVLA Training

## Table of Contents
- [1. Setup & Port Discovery](#1-setup--port-discovery)
- [2. Camera Preview](#2-camera-preview)
- [3. Recording the Dataset](#3-recording-the-dataset)
- [4. Recording Controls](#4-recording-controls)
- [5. Training (SmolVLA)](#5-training-smolvla)
- [6. Running Evaluation](#6-running-evaluation)
- [7. Extra / Utility Commands](#7-extra--utility-commands)

---

## 1. Setup & Port Discovery

Identify the serial ports for the leader/follower arms and the video device for the camera:

```bash
ls -l /dev/ttyACM* /dev/video*
```

## 2. Camera Preview

Preview the camera feed to adjust its position before recording:

```bash
ffplay -f v4l2 -input_format mjpeg -video_size 1920x1080 -framerate 30 /dev/video0
```

## 3. Recording the Dataset

Dataset save location: `/home/postgrad/fahim/datasets_2`
Dataset name used: `pick_place_second_test`

```bash
lerobot-record \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM1 \
  --robot.id=my_awesome_follower_arm \
  --robot.cameras="{front: {type: opencv, index_or_path: 0, width: 1920, height: 1080, fps: 30, fourcc: MJPG}}" \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM0 \
  --teleop.id=my_awesome_leader_arm \
  --display_data=true \
  --display_compressed_images=true \
  --dataset.repo_id=local/pick_place_second_test \
  --dataset.root="/home/postgrad/fahim/datasets_2/pick_place_second_test" \
  --dataset.single_task="pick the cube and place it inside the box" \
  --dataset.num_episodes=70 \
  --dataset.episode_time_s=90 \
  --dataset.reset_time_s=0 \
  --dataset.vcodec=h264 \
  --dataset.push_to_hub=false
```

To resume a paused/interrupted recording loop, append `--resume=true` to the command above.

## 4. Recording Controls

Useful keyboard shortcuts while recording:

| Action | Key |
|---|---|
| Re-record current episode | Left arrow (once) |
| End episode early (before episode time ends) | Right arrow (once) |
| Start next recording early (before reset time ends) | Right arrow (once) |
| Pause (saves current recording, then pauses) | `Esc` |
| Stop the loop **without saving** current recording | `Ctrl+C` |
| Resume recording after a pause | Add `--resume=true` to the start command |

> ⚠️ `Ctrl+C` discards the in-progress episode — use `Esc` if you want to preserve it.

## 5. Training (SmolVLA)

```bash
cd ~/lerobot

PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True lerobot-train \
  --policy.path=lerobot/smolvla_base \
  --dataset.repo_id=local/pick_place_second_test \
  --dataset.root="/home/postgrad/fahim/datasets_2/pick_place_second_test" \
  --rename_map='{"observation.images.front":"observation.images.camera1"}' \
  --policy.empty_cameras=0 \
  --policy.resize_imgs_with_padding='[256,256]' \
  --policy.use_amp=true \
  --policy.device=cuda \
  --batch_size=1 \
  --num_workers=0 \
  --steps=30000 \
  --policy.scheduler_decay_steps=30000 \
  --save_freq=5000 \
  --log_freq=10 \
  --output_dir="/home/postgrad/fahim/outputs_second/train/smolvla_pick_place_second" \
  --job_name=smolvla_pick_place_second_30k \
  --wandb.enable=false \
  --policy.push_to_hub=false
```

### Skipping a faulty episode

If a specific episode needs to be excluded from training, generate the episode list first and pass it in:

```bash
EPISODES="$(python -c 'import json; print(json.dumps([i for i in range(71) if i != x]))')"
```

Replace `x` with the faulty episode number, then include `$EPISODES` in the training command.

## 6. Running Evaluation

```bash
conda activate lerobot
cd ~/lerobot

EVAL_ID="eval_smolvla_$(date +%Y%m%d_%H%M%S)"

lerobot-record \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM1 \
  --robot.id=my_awesome_follower_arm \
  --robot.cameras="{camera1: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30, fourcc: MJPG}}" \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM0 \
  --teleop.id=my_awesome_leader_arm \
  --policy.path="/home/postgrad/fahim/so_arm_101(training)/outputs/outputs_second/train/smolvla_pick_place_second/checkpoints/030000/pretrained_model" \
  --policy.empty_cameras=0 \
  --policy.resize_imgs_with_padding='[256,256]' \
  --policy.use_amp=true \
  --policy.device=cuda \
  --display_data=true \
  --display_compressed_images=true \
  --dataset.repo_id="local/${EVAL_ID}" \
  --dataset.root="/home/postgrad/fahim/datasets_2/${EVAL_ID}" \
  --dataset.single_task="pick the cube and place it inside the box" \
  --dataset.num_episodes=100 \
  --dataset.episode_time_s=1000 \
  --dataset.reset_time_s=0 \
  --dataset.vcodec=h264 \
  --dataset.push_to_hub=false
```

## 7. Extra / Utility Commands

**Teleoperation only** (control follower arm with leader arm, no recording):

```bash
conda activate lerobot
cd ~/lerobot
sudo chmod 666 /dev/ttyACM*
lerobot-teleoperate \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM1 \
  --robot.id=my_awesome_follower_arm \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM0
```

**View a previously recorded episode:**

```bash
lerobot-dataset-viz \
  --repo-id local/pick_place_second_test \
  --root "/home/postgrad/fahim/datasets_2/pick_place_second_test" \
  --mode local \
  --episode-index 0 \
  --display-compressed-images=true
```

**Check disk storage:**

```bash
df -h ~
```

**Empty trash after training:**

```bash
gio trash --empty
```

**Verify checkpoints saved successfully:**

```bash
find "/home/postgrad/fahim/outputs_second" \
  -name pretrained_model \
  -printf '%T@ %p\n' |
  sort -nr
```

**Quick snapshot of RAM, GPU, and disk usage:**

```bash
free -h
nvidia-smi
df -h
```
