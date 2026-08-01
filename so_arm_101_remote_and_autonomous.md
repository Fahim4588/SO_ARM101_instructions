# SO-ARM101 Local Training and Troubleshooting Handbook (Updated)

This handbook explains how to take an SO-ARM101 from hardware checks to a locally trained ACT policy, then diagnose the failures seen in the supplied ChatGPT conversations.

> **Field-notes update (1 August 2026):** Sections 23-30 incorporate the commands and observations from the second hands-on run. The original handbook remains available in Sections 1-22. Commands in the update are tied to the Seeed LeRobot checkout and the machine paths shown below; always confirm the installed CLI with `--help` before running them.

It is tailored to this working setup:

- Ubuntu/Linux
- Leader arm: `/dev/ttyACM0`
- Follower arm: `/dev/ttyACM1`
- Wrist camera: `/dev/video0`
- LeRobot repository: `/home/postgrad/lerobot`
- Local dataset: `/home/postgrad/datasets/so101_task`
- Example task: a simple pick-and-place or arm-movement task

Do not assume those device names are correct on another computer. Discover the ports and camera first.

> **Version warning:** the supplied chats contain commands from LeRobot 0.4.4, 0.6.1, and a newer development version. LeRobot’s CLI changes between releases. Run the version and `--help` checks below before copying a command. This guide labels legacy behavior where necessary.

## 1. Safety rules

1. Keep an emergency-stop method within reach. Be prepared to disconnect motor power.
2. Start with the follower arm clear of people, cables, walls, and fragile objects.
3. Use the correct motor power supply. SO-101 motor voltage variants are not interchangeable; the leader uses the lower-voltage motor configuration. Verify the hardware documentation before applying power.
4. Never force a joint beyond its mechanical range during calibration.
5. Use the same robot and teleoperator IDs after calibration. The IDs select the calibration files.
6. Do not run two training jobs on the same GPU unless resource sharing is intentional.
7. Do not run a camera preview and LeRobot recording/inference at the same time.
8. Never delete a dataset merely because a command failed. Rename it as a backup first.

Safe backup pattern:

```bash
mv /home/postgrad/datasets/so101_task \
   /home/postgrad/datasets/so101_task.backup.$(date +%Y%m%d-%H%M%S)
```

Inspect the expanded path before using any destructive command. Avoid broad commands such as `rm -rf ~/datasets` or `rm -rf ~/.cache/huggingface/lerobot`; they can erase unrelated datasets and calibration files.

## 2. Record the exact software version

Activate the environment and capture a reproducibility report:

```bash
conda activate lerobot
cd /home/postgrad/lerobot

which python
which lerobot-record
python --version
python -m pip show lerobot torch torchvision torchcodec
python -c "import lerobot, torch; print('lerobot:', getattr(lerobot, '__version__', 'source checkout')); print('torch:', torch.__version__); print('cuda available:', torch.cuda.is_available()); print('cuda device:', torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'none')"
git rev-parse HEAD 2>/dev/null || true

lerobot-record --help > /tmp/lerobot-record-help.txt
lerobot-train --help > /tmp/lerobot-train-help.txt
lerobot-replay --help > /tmp/lerobot-replay-help.txt
```

If a flag in this guide is rejected, check the corresponding help file. Do not guess a replacement.

## 3. Clean installation on a new computer

The current official installation guide uses Python 3.12. For a new source installation:

```bash
conda create -y -n lerobot python=3.12
conda activate lerobot
conda install -y ffmpeg=7.1.1 -c conda-forge

git clone https://github.com/huggingface/lerobot.git
cd lerobot

python -m pip install --upgrade pip
python -m pip install -e ".[core_scripts,training,feetech]"
```

For SmolVLA, add its optional dependencies:

```bash
python -m pip install -e ".[smolvla]"
```

Verify the installation:

```bash
which lerobot-record
lerobot-record --help
lerobot-train --help
ffmpeg -version
ffmpeg -encoders | grep -E 'h264|libsvtav1'
```

Official references:

- [LeRobot installation](https://huggingface.co/docs/lerobot/main/en/installation)
- [SO-101 setup, motors, and calibration](https://huggingface.co/docs/lerobot/so101)
- [Real-world robot workflow](https://huggingface.co/docs/lerobot/il_robots)

### Avoid mixed installations

Errors such as:

```text
AttributeError: 'OpenCVCameraConfig' object has no attribute 'copy'
```

often indicate that the command-line executable and imported Python package came from different LeRobot installations.

Check:

```bash
which lerobot-teleoperate
head -n 1 "$(which lerobot-teleoperate)"
python -c "import lerobot; print(lerobot.__file__)"
python -m pip show lerobot
```

If the paths point to different environments or repositories, create a fresh environment and install one pinned LeRobot checkout. Do not patch the camera class until the environment mismatch is eliminated.

## 4. Discover ports and grant serial access

Do not permanently assume that leader is `ttyACM0` and follower is `ttyACM1`. Device numbers may swap after reconnecting or rebooting.

```bash
ls -l /dev/ttyACM*
lerobot-find-port
```

Follow the prompt and disconnect one controller at a time. Record the result:

```bash
FOLLOWER_PORT=/dev/ttyACM1
LEADER_PORT=/dev/ttyACM0
FOLLOWER_ID=my_follower_arm
LEADER_ID=my_leader_arm
```

For a persistent Linux permission fix:

```bash
sudo usermod -aG dialout "$USER"
```

Log out and back in afterward. A temporary fix is:

```bash
sudo chmod 666 /dev/ttyACM0 /dev/ttyACM1
```

Run `chmod` separately. One supplied chat accidentally pasted `lerobot-teleoperate` options onto the `chmod` command, producing:

```text
chmod: unrecognized option '--robot.type=so101_follower'
```

## 5. One-time motor setup

Only use these commands when configuring new or repurposed motors:

```bash
lerobot-setup-motors \
  --robot.type=so101_follower \
  --robot.port="$FOLLOWER_PORT"

lerobot-setup-motors \
  --teleop.type=so101_leader \
  --teleop.port="$LEADER_PORT"
```

During setup, connect only the motor requested by the script. Ensure motor power, USB, controller jumpers, and three-pin cables are correct.

## 6. Calibrate both arms

Follower:

```bash
lerobot-calibrate \
  --robot.type=so101_follower \
  --robot.port="$FOLLOWER_PORT" \
  --robot.id="$FOLLOWER_ID"
```

Leader:

```bash
lerobot-calibrate \
  --teleop.type=so101_leader \
  --teleop.port="$LEADER_PORT" \
  --teleop.id="$LEADER_ID"
```

Do not leave a bare `--teleop` at the end of the command. It causes:

```text
lerobot-calibrate: error: argument --teleop: expected one argument
```

Calibration procedure:

1. Put the requested arm near the middle of every joint’s range.
2. Press Enter.
3. Sweep each joint through its full safe range.
4. Open and close the gripper fully.
5. Save the calibration.
6. Reuse the exact same IDs in teleoperation, recording, replay, and deployment.

If LeRobot reports a calibration mismatch, first confirm that the correct arm, port, and ID are being used. Accept an existing calibration only when it belongs to that physical arm.

## 7. Find and test the camera

```bash
ls -l /dev/video*
lerobot-find-cameras
v4l2-ctl --device=/dev/video0 --list-formats-ext
```

Some older versions use:

```bash
lerobot-find-cameras opencv
```

Use `lerobot-find-cameras --help` to determine which form applies.

Test the selected mode without leaving the preview open:

```bash
ffplay -f v4l2 -video_size 640x480 -framerate 30 /dev/video0
```

Quit `ffplay` before starting LeRobot.

For stable camera naming, inspect:

```bash
ls -l /dev/v4l/by-id/
```

When supported by the installed LeRobot version, a `/dev/v4l/by-id/...` path is safer than an index that may change.

## 8. Teleoperation preflight

Start with one wrist camera:

```bash
lerobot-teleoperate \
  --robot.type=so101_follower \
  --robot.port="$FOLLOWER_PORT" \
  --robot.id="$FOLLOWER_ID" \
  --robot.cameras="{wrist: {type: opencv, index_or_path: /dev/video0, width: 640, height: 480, fps: 30}}" \
  --teleop.type=so101_leader \
  --teleop.port="$LEADER_PORT" \
  --teleop.id="$LEADER_ID" \
  --display_data=true
```

Confirm all of the following:

- Moving the leader moves the corresponding follower joint in the correct direction.
- The follower does not jump when control starts.
- The gripper opens and closes correctly.
- Rerun displays the wrist camera.
- No joint state or camera stream freezes.

Do not begin data collection until teleoperation is reliable.

## 9. Choose a learnable first task

For a first ACT model, use one narrow task:

> Pick up the cube and place it in the marked target area.

Keep these conditions consistent:

- robot starting pose;
- object shape and approximate position;
- camera position and exposure;
- background and lighting;
- task wording;
- episode length and reset procedure.

Record successful demonstrations with smooth motion. Discard episodes containing collisions, long pauses, missed grasps, blocked camera views, or incomplete placement.

Five episodes can verify that the pipeline runs, but they are usually not enough for a robust policy. Start with 30–50 good episodes for a constrained task; move toward 100 if behavior is inconsistent or the scene varies.

ACT is the recommended first policy for one specific task. SmolVLA is more appropriate when language conditioning and broader task variation are genuine requirements, but it needs image observations, more compute, and typically more diverse data. Switching to SmolVLA will not fix a corrupt dataset, broken camera, DataLoader failure, or USB problem.

## 10. Record a fresh local dataset

Create only the parent directory:

```bash
mkdir -p /home/postgrad/datasets
```

For LeRobot versions that create `dataset.root` themselves, the final target must not already exist. If it does, preserve it:

```bash
test ! -e /home/postgrad/datasets/so101_task || \
  mv /home/postgrad/datasets/so101_task \
     /home/postgrad/datasets/so101_task.backup.$(date +%Y%m%d-%H%M%S)
```

Record five pipeline-test episodes:

```bash
lerobot-record \
  --robot.type=so101_follower \
  --robot.port="$FOLLOWER_PORT" \
  --robot.id="$FOLLOWER_ID" \
  --robot.cameras="{wrist: {type: opencv, index_or_path: /dev/video0, width: 640, height: 480, fps: 30}}" \
  --teleop.type=so101_leader \
  --teleop.port="$LEADER_PORT" \
  --teleop.id="$LEADER_ID" \
  --display_data=true \
  --dataset.repo_id=local/so101_task \
  --dataset.root=/home/postgrad/datasets/so101_task \
  --dataset.single_task="Pick up the cube and place it in the target area" \
  --dataset.num_episodes=5 \
  --dataset.push_to_hub=false
```

Important:

- `--dataset.single_task` is required by the versions seen in the chats. Omitting it caused the “missing task field” sequence.
- `--dataset.push_to_hub=false` keeps the dataset local and prevents a later 403 Hub upload failure.
- `--dataset.root` is the dataset directory containing `meta/`, `data/`, and `videos/`, not merely its parent.
- A repo ID still uses `namespace/name` form even for local work; `local/so101_task` is valid for the supplied setup.
- If `--dataset.root` is rejected, inspect `lerobot-record --help`; do not combine syntax from another version.

### Recording controls

Use the keyboard controls printed by your installed version. Common controls include ending an episode early, discarding and re-recording the current episode, and stopping the session. Read the startup instructions rather than assuming the key mapping.

### Resume recording

The official workflow supports adding episodes by reusing the same command with:

```bash
--resume=true
```

Also use the same `dataset.root`, `repo_id`, task, robot IDs, and camera configuration. On current LeRobot, `dataset.num_episodes` is the number of **additional** episodes, not the final total.

Example: add 20 episodes:

```bash
lerobot-record \
  --robot.type=so101_follower \
  --robot.port="$FOLLOWER_PORT" \
  --robot.id="$FOLLOWER_ID" \
  --robot.cameras="{wrist: {type: opencv, index_or_path: /dev/video0, width: 640, height: 480, fps: 30}}" \
  --teleop.type=so101_leader \
  --teleop.port="$LEADER_PORT" \
  --teleop.id="$LEADER_ID" \
  --dataset.repo_id=local/so101_task \
  --dataset.root=/home/postgrad/datasets/so101_task \
  --dataset.single_task="Pick up the cube and place it in the target area" \
  --dataset.num_episodes=20 \
  --dataset.push_to_hub=false \
  --resume=true
```

Verify that the first log line continues after the existing episode count. Stop immediately if it starts overwriting from episode 0.

## 11. Validate the dataset before training

### Structure

```bash
find /home/postgrad/datasets/so101_task -maxdepth 3 -type f | sort | head -n 100
du -sh /home/postgrad/datasets/so101_task
```

A current LeRobot dataset normally contains:

```text
so101_task/
├── data/
├── meta/
└── videos/
```

Exact filenames differ between dataset format versions.

### Local visualization

For versions supporting local mode:

```bash
lerobot-dataset-viz \
  --repo-id local/so101_task \
  --root /home/postgrad/datasets \
  --mode local \
  --episode-index 0
```

The supplied chats sometimes used the dataset directory itself as `--root`. Current official documentation defines `--root` as the directory under which the `local/so101_task` layout can be resolved. Run `lerobot-dataset-viz --help` and adjust to the installed version.

Inspect multiple episodes:

- camera frames are present, correctly oriented, and smooth;
- no missing, black, or frozen frames;
- joint positions and actions change smoothly;
- gripper channels open and close;
- every episode completes the same task;
- task text is consistent.

Official reference: [LeRobot local dataset visualization](https://huggingface.co/docs/lerobot/using_dataset_tools#local-visualization).

### Video integrity

```bash
find /home/postgrad/datasets/so101_task/videos -type f -name '*.mp4' -print0 |
while IFS= read -r -d '' video; do
  if ! ffprobe -v error -show_entries stream=codec_name,width,height,avg_frame_rate \
       -of default=noprint_wrappers=1 "$video" >/dev/null; then
    echo "BAD VIDEO: $video"
  fi
done
```

Decode-test every video:

```bash
find /home/postgrad/datasets/so101_task/videos -type f -name '*.mp4' -print0 |
while IFS= read -r -d '' video; do
  if ! ffmpeg -v error -i "$video" -f null -; then
    echo "DECODE FAILED: $video"
  fi
done
```

Do not train until bad videos or episodes are removed/re-recorded using a tool compatible with that dataset format.

## 12. Replay a demonstration

Replay is a hardware safety and data sanity check:

```bash
lerobot-replay \
  --robot.type=so101_follower \
  --robot.port="$FOLLOWER_PORT" \
  --robot.id="$FOLLOWER_ID" \
  --dataset.repo_id=local/so101_task \
  --dataset.root=/home/postgrad/datasets/so101_task \
  --dataset.episode=0
```

The supplied LeRobot 0.4.4 help output used `--dataset.episode`, not `--episode`. If rejected, consult `lerobot-replay --help`.

Keep the workspace clear: replay sends recorded actions to the real follower arm.

## 13. Train ACT locally

Training does not require the robot, leader, or camera to remain connected. Only the validated dataset and training computer are required.

### Check the accelerator

```bash
python -c "import torch; print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'CPU mode')"
nvidia-smi
```

Use `cuda` only when PyTorch reports it as available. Use `mps` on supported Apple silicon. CPU works but can be very slow.

### Stable first training run

This configuration reflects the successful recovery in the supplied chats: 10,000 steps, batch size 8, frequent checkpoints, and zero DataLoader workers to avoid the earlier TorchCodec/segmentation fault.

```bash
mkdir -p /home/postgrad/lerobot/outputs/train

lerobot-train \
  --dataset.repo_id=local/so101_task \
  --dataset.root=/home/postgrad/datasets/so101_task \
  --policy.type=act \
  --policy.device=cuda \
  --policy.repo_id=local/so101_act_model \
  --policy.push_to_hub=false \
  --output_dir=/home/postgrad/lerobot/outputs/train/act_so101_task \
  --job_name=act_so101_task \
  --steps=10000 \
  --batch_size=8 \
  --num_workers=0 \
  --save_freq=500 \
  --wandb.enable=false
```

Why these settings:

- `--num_workers=0`: safest diagnostic setting after a DataLoader or video-decoder crash;
- `--save_freq=500`: limits lost work while stabilizing the pipeline;
- absolute `output_dir`: avoids ambiguity about whether `~` was expanded;
- `push_to_hub=false`: keeps the policy local;
- W&B disabled: no login or network dependency.

After the run is stable past the previous failure point, test `--num_workers=2` or `4` in a new output directory if faster data loading is needed.

Do not reuse a non-empty output directory for a fresh run. Preserve it:

```bash
test ! -e /home/postgrad/lerobot/outputs/train/act_so101_task || \
  mv /home/postgrad/lerobot/outputs/train/act_so101_task \
     /home/postgrad/lerobot/outputs/train/act_so101_task.backup.$(date +%Y%m%d-%H%M%S)
```

### Expected startup evidence

Healthy startup logs include:

```text
Creating dataset
Creating policy
Creating optimizer and scheduler
Start offline training on a fixed dataset
```

Healthy progress includes increasing step/sample counters and a finite loss:

```text
step:400 ... loss:2.871 ...
Checkpoint policy after step 500
```

Loss decreasing is useful, but real-robot success rate—not training loss alone—determines whether the policy works.

## 14. Monitor training

In another terminal:

```bash
watch -n 2 nvidia-smi
```

Check the process:

```bash
pgrep -af lerobot-train
```

Check output growth:

```bash
watch -n 5 'find /home/postgrad/lerobot/outputs/train/act_so101_task -maxdepth 4 -type f -printf "%TY-%Tm-%Td %TH:%TM:%TS %p\n" 2>/dev/null | sort | tail -n 20'
```

Locate checkpoints:

```bash
find /home/postgrad/lerobot/outputs/train/act_so101_task \
  -type d -name pretrained_model -print
```

Do not classify training as “stuck” from a quiet terminal alone. Check:

- process exists;
- GPU or CPU utilization is active;
- output files or logs continue changing;
- step counters advance;
- disk is not full;
- the process has not returned to the shell prompt.

## 15. Resume training correctly

Current official LeRobot resumes from the training configuration saved with a checkpoint:

```bash
lerobot-train \
  --config_path=/home/postgrad/lerobot/outputs/train/act_so101_task/checkpoints/last/pretrained_model/train_config.json \
  --resume=true
```

If there is no `last` alias, locate the newest checkpoint:

```bash
find /home/postgrad/lerobot/outputs/train/act_so101_task/checkpoints \
  -type f -name train_config.json -print | sort
```

Do not use:

```text
--policy.pretrained_path=/local/checkpoint/path
```

to resume an interrupted training run unless the installed help explicitly documents it. In the supplied chat, LeRobot treated the local path as a Hugging Face repo ID and raised `HFValidationError`.

Official resume reference: [LeRobot ACT training and checkpoint resume](https://github.com/huggingface/lerobot/blob/main/docs/source/il_robots.mdx).

## 16. Deploy and evaluate the trained policy

First locate a valid policy directory:

```bash
find /home/postgrad/lerobot/outputs/train/act_so101_task \
  -type d -name pretrained_model -print
```

### Current LeRobot

Current LeRobot uses `lerobot-rollout`:

```bash
lerobot-rollout \
  --strategy.type=base \
  --policy.path=/home/postgrad/lerobot/outputs/train/act_so101_task/checkpoints/last/pretrained_model \
  --robot.type=so101_follower \
  --robot.port="$FOLLOWER_PORT" \
  --robot.id="$FOLLOWER_ID" \
  --robot.cameras="{wrist: {type: opencv, index_or_path: /dev/video0, width: 640, height: 480, fps: 30}}" \
  --task="Pick up the cube and place it in the target area" \
  --duration=30 \
  --display_data=true
```

Official reference: [LeRobot policy deployment](https://huggingface.co/docs/lerobot/main/inference).

### LeRobot 0.5.1 and older

Older releases evaluate with `lerobot-record --policy.path=...`. Generate the exact command from:

```bash
lerobot-record --help
```

The deployment camera names, count, resolution, and task wording must match training data. A policy trained with `observation.images.wrist` cannot be safely evaluated with a differently named or missing camera feature.

Run at least 10 evaluation episodes and report:

```text
success rate = successful episodes / total episodes
```

Compare multiple checkpoints; the final checkpoint is not automatically the best.

## 17. Troubleshooting guide

### 17.1 `FileExistsError` while creating a dataset

Example:

```text
FileExistsError: [Errno 17] File exists
```

Cause: that LeRobot version expects to create the final dataset root, but it already exists. A previous failed recording may have left an incomplete directory.

Safe recovery:

```bash
ls -la /home/postgrad/datasets/so101_task
mv /home/postgrad/datasets/so101_task \
   /home/postgrad/datasets/so101_task.incomplete.$(date +%Y%m%d-%H%M%S)
```

Then rerun recording. Do not recreate the final root before recording.

If the log names a cache path such as:

```text
/home/postgrad/.cache/huggingface/lerobot/local/so101_move_task
```

inspect and rename that exact path. Do not delete a guessed path containing an extra `datasets/` component.

### 17.2 `'NoneType' object has no attribute 'push_to_hub'`

This is usually a secondary cleanup error. In the supplied chat, dataset creation first failed with `FileExistsError`; cleanup then tried to push a dataset object that had never been created.

Fix the first error in the traceback. Also set:

```bash
--dataset.push_to_hub=false
```

for fully local recording.

### 17.3 `403 Forbidden` while creating a Hugging Face repository

Cause: uploading remained enabled, but the account/token lacked permission or local work was intended.

For local work:

```bash
--dataset.push_to_hub=false
--policy.push_to_hub=false
```

No W&B or Hugging Face login is required for a fully local ACT workflow.

### 17.4 Missing task field

Use one consistent task:

```bash
--dataset.single_task="Pick up the cube and place it in the target area"
```

Do not alternate task wording between demonstrations and evaluation.

### 17.5 Dataset root points to the parent directory

For training, `dataset.root` must normally be the folder containing the dataset’s `meta/`, `data/`, and `videos/`:

```bash
--dataset.root=/home/postgrad/datasets/so101_task
```

not:

```bash
--dataset.root=/home/postgrad/datasets
```

Visualization commands can use parent-root semantics, so do not assume that `lerobot-dataset-viz --root` and `lerobot-train --dataset.root` mean the same thing in every release.

### 17.6 Camera width mismatch or `VIDIOC_QBUF: Bad file descriptor`

Example:

```text
failed to set capture_width=1280 (actual_width=640)
ioctl(VIDIOC_QBUF): Bad file descriptor
```

Recovery:

```bash
fuser -v /dev/video0
sudo fuser -k /dev/video0
v4l2-ctl --device=/dev/video0 --list-formats-ext
v4l2-ctl --device=/dev/video0 --get-fmt-video
```

If the camera supports MJPEG 1280×720:

```bash
v4l2-ctl \
  --device=/dev/video0 \
  --set-fmt-video=width=1280,height=720,pixelformat=MJPG
```

Then make the LeRobot camera configuration match exactly. Alternatively use the known-stable 640×480 at 30 FPS configuration throughout recording and deployment.

Close `ffplay`, OpenCV, Rerun camera scripts, browsers, and other camera users before retrying.

### 17.7 Camera works in `ffplay` but LeRobot shows no camera

Check the startup configuration. If it prints:

```text
cameras: {}
```

the camera was not added to the robot configuration. Add:

```bash
--robot.cameras="{wrist: {type: opencv, index_or_path: /dev/video0, width: 640, height: 480, fps: 30}}"
```

SmolVLA requires visual observations. A state-only dataset cannot train the intended VLA policy.

### 17.8 Corrupt AV1 video, TorchCodec failure, or segmentation fault

Symptoms:

- decode exception;
- training consistently stops at a sample/step;
- `Segmentation fault (core dumped)`;
- DataLoader workers hang;
- AV1 video fails while H.264 succeeds.

Procedure:

1. Run both video integrity loops in Section 11.
2. Visualize the affected episodes.
3. Back up the dataset.
4. Remove/re-record bad episodes with version-compatible LeRobot dataset tools.
5. Retry training with `--num_workers=0`.
6. Confirm training passes the former crash step.
7. Increase workers only after stability is proven.

Current LeRobot includes a dataset re-encoding tool. Back up first, then check:

```bash
lerobot-edit-dataset --help
```

The current documented operation is `reencode_videos` with an H.264 encoder. Do not replace video files manually without preserving LeRobot metadata.

Also verify TorchCodec/FFmpeg compatibility:

```bash
python -m pip show torch torchcodec
ffmpeg -version
```

The current official installation guide recommends FFmpeg 7.1.1 when the default FFmpeg version or encoder set conflicts with TorchCodec.

### 17.9 Training is quiet or appears stuck

Run:

```bash
pgrep -af lerobot-train
nvidia-smi
df -h
df -i
find /home/postgrad/lerobot/outputs/train -type f -mmin -5 -print
```

Interpretation:

- no process and shell prompt returned: training exited;
- process plus active GPU/CPU and changing files: training is running;
- process with no compute and no file changes: inspect the last log lines and dataset decoder;
- memory at capacity: lower batch size, close other GPU jobs, or use CPU;
- disk/inodes full: free space without deleting the active dataset/checkpoints.

Changing ACT to SmolVLA does not fix this class of failure.

### 17.10 Training finished but checkpoint saving failed

Before retraining, search for intermediate policies:

```bash
find /home/postgrad/lerobot -type d -name pretrained_model -print
find /home/postgrad/lerobot -type f \( -name '*.safetensors' -o -name 'train_config.json' \) -print
df -h
df -i
```

Check output permissions:

```bash
ls -ld /home/postgrad/lerobot/outputs \
       /home/postgrad/lerobot/outputs/train \
       /home/postgrad/lerobot/outputs/train/act_so101_task
```

Use absolute output paths in future runs. If an intermediate checkpoint contains `pretrained_model`, it can be evaluated even when final saving failed. If no checkpoint was ever written, in-memory progress cannot be recovered after the process exits.

### 17.11 `HFValidationError` for a local checkpoint

Cause: a local filesystem path was passed through a field that the installed version interpreted as a Hub repository ID.

Resume with:

```bash
lerobot-train \
  --config_path=/absolute/checkpoint/pretrained_model/train_config.json \
  --resume=true
```

Do not use `--policy.pretrained_path` as a substitute for training resume.

### 17.12 `policy.repo_id argument missing`

Some older releases require a syntactically valid repo ID even with local output. Use:

```bash
--policy.repo_id=local/so101_act_model \
--policy.push_to_hub=false
```

If `policy.push_to_hub` does not exist in that release, consult `lerobot-train --help` and pin a release whose offline behavior is known.

### 17.13 CUDA unavailable

Example:

```text
Device 'cuda' is not available. Switching to 'cpu'.
```

Verify:

```bash
nvidia-smi
python -c "import torch; print(torch.__version__); print(torch.version.cuda); print(torch.cuda.is_available())"
```

If there is no NVIDIA GPU, use:

```bash
--policy.device=cpu
```

An AMD GPU does not support CUDA. ROCm requires a separately compatible PyTorch/LeRobot stack; do not set `cuda` merely because the machine has a GPU.

### 17.14 USB `There is no status packet`

Example:

```text
ConnectionError: Failed to sync read 'Present_Position' ...
There is no status packet!
```

Check in this order:

1. Correct leader/follower port mapping.
2. Motor power supply is on and correct.
3. USB and three-pin cables are seated.
4. Controller jumpers/mode are correct.
5. Motors have unique IDs 1–6 and the correct common baud rate.
6. No second process owns the same serial port.
7. Calibration ID belongs to the connected arm.

Useful commands:

```bash
fuser -v /dev/ttyACM0 /dev/ttyACM1
dmesg --ctime | tail -n 50
```

If ports may be swapped, disconnect one arm and rerun `lerobot-find-port`.

### 17.15 Bash syntax errors from copied commands

Examples from the chats:

```text
bash: syntax error near unexpected token '('
command '--wandb.enable=true' not found
```

Causes:

- terminal output or explanatory prose was pasted as a command;
- a multiline command lost a trailing `\`;
- a command was appended to the previous command;
- smart quotes replaced normal quotes.

Rules:

- Copy only the fenced Bash block.
- Every continued line except the last must end with `\`.
- Nothing—not even a space—should follow the continuation backslash.
- Run permission commands separately.
- Use plain ASCII quotes.

Validate a saved script without executing it:

```bash
bash -n training_command.sh
```

### 17.16 W&B login prompt

Weights & Biases is optional experiment tracking. For offline/simple work:

```bash
--wandb.enable=false
```

If enabled:

```bash
wandb login
```

Never paste the W&B API key into shared logs or chat.

### 17.17 LeLab camera dropdown is empty

One supplied chat showed that LeLab’s backend `/available-cameras` endpoint detected the camera but the frontend could not match its name. This is a LeLab UI issue, not a blocker for training.

Use the CLI workflow:

```text
lerobot-calibrate → lerobot-teleoperate → lerobot-record → lerobot-train → lerobot-rollout
```

Return to LeLab only after its frontend camera matching is fixed.

## 18. Diagnostic bundle

When asking for help, provide the command and the first error—not only the final exception.

```bash
conda activate lerobot
cd /home/postgrad/lerobot

{
  date
  uname -a
  python --version
  which python
  which lerobot-record
  python -m pip show lerobot torch torchvision torchcodec
  python -c "import lerobot, torch; print(lerobot.__file__); print(torch.__version__); print(torch.version.cuda); print(torch.cuda.is_available())"
  git rev-parse HEAD 2>/dev/null || true
  ls -l /dev/ttyACM* /dev/video* 2>&1
  fuser -v /dev/ttyACM0 /dev/ttyACM1 /dev/video0 2>&1
  v4l2-ctl --device=/dev/video0 --get-fmt-video 2>&1
  nvidia-smi 2>&1
  df -h
  df -i
} | tee /tmp/so101_diagnostics.txt
```

Do not include API keys, Hugging Face tokens, passwords, or private repository credentials.

Also provide:

- the exact command that failed;
- complete traceback from its first error;
- last 50 training log lines;
- whether the shell prompt returned;
- dataset structure;
- failing video filename, if any;
- recent kernel messages after a USB failure.

## 19. Restart checklist after shutdown or disconnection

```bash
conda activate lerobot
cd /home/postgrad/lerobot

ls -l /dev/ttyACM*
lerobot-find-port
lerobot-find-cameras
fuser -v /dev/ttyACM0 /dev/ttyACM1 /dev/video0
```

Then:

1. update `FOLLOWER_PORT` and `LEADER_PORT`;
2. apply temporary serial permissions only if group permissions are not yet configured;
3. power the correct arm controllers;
4. test teleoperation without a camera;
5. test the camera;
6. test teleoperation with the camera;
7. record, replay, or deploy.

Do not recalibrate after every reboot. Recalibrate when hardware geometry changes, calibration is missing/mismatched, motors were reconfigured, or leader/follower alignment is incorrect.

## 20. End-to-end success checklist

- [ ] LeRobot version, commit, Python, PyTorch, and CUDA state recorded.
- [ ] Leader and follower ports positively identified.
- [ ] Serial permissions work without combining commands.
- [ ] Motor IDs/baud rate configured once.
- [ ] Leader and follower calibrated with stable IDs.
- [ ] Teleoperation is smooth and correctly mapped.
- [ ] Camera mode is supported and no other process owns it.
- [ ] One narrow task and consistent reset procedure selected.
- [ ] Dataset recorded locally with task text and camera images.
- [ ] Dataset visualized and videos fully decode-tested.
- [ ] Episode replay is physically correct.
- [ ] ACT training starts and step counters advance.
- [ ] Checkpoints appear at the configured frequency.
- [ ] Resume uses `train_config.json` plus `--resume=true`.
- [ ] Deployment uses the same observation schema and task text.
- [ ] At least 10 physical evaluation episodes measured.
- [ ] Dataset, environment report, commands, and best checkpoint backed up.

## 21. Supplied-chat audit

Twenty-four shared chats were reviewed. The following were intentionally excluded because they are unrelated to SO-ARM101 or LeRobot:

1. [Chat 19 — Application for Hall Access](https://chatgpt.com/share/6a6b17e9-adec-83ee-8aa3-cf9b9857d7d8)
2. [Chat 20 — Special Permission Application](https://chatgpt.com/share/6a6b17fc-9684-83e8-9eb4-8fd9c7ede6ce)

Chat 2, [Training Completion Advice](https://chatgpt.com/share/6a6b1722-dcb8-83ee-95dc-331371d89b66), was not excluded, but its shared page exposed only an uploaded-image marker and the question “I don't want to stop…what should I choose.” It contained too little readable evidence to establish a technical fix, so no command was taken from it.

The other 21 readable, relevant chats informed this guide:

1. Training completed, checkpoint failed
2. Fix training process
3. Training process issue
4. Training progress update
5. Corrupted video in dataset
6. Training folder cleanup
7. Training progress update / Isaac Sim clarification
8. SO-ARM101 local training
9. Missing task field error
10. Calibration command error
11. LeRobot dataset error
12. SO-ARM101 local training workflow
13. Fresh-start cleanup
14. LeRobot 0.6.1 dataset-root/FileExists behavior
15. Dataset visualization
16. Recording more episodes / SmolVLA
17. LeLab command and camera UI issue
18. USB port connection error
19. W&B login explanation
20. SO-ARM101 dataset creation
21. Camera feedback in Rerun and ACT versus SmolVLA

Unsafe broad deletion advice, contradictory port assumptions, outdated inference syntax, and unverified checkpoint-loading suggestions from the chats were not copied as instructions. They were replaced with backup-first operations and version-aware commands.

## 22. Authoritative references

- [LeRobot installation](https://huggingface.co/docs/lerobot/main/en/installation)
- [SO-101 hardware setup and calibration](https://huggingface.co/docs/lerobot/so101)
- [LeRobot real-world imitation-learning workflow](https://huggingface.co/docs/lerobot/il_robots)
- [LeRobot command cheat sheet](https://huggingface.co/docs/lerobot/main/cheat-sheet)
- [Local dataset visualization and dataset tools](https://huggingface.co/docs/lerobot/using_dataset_tools)
- [Current policy deployment with `lerobot-rollout`](https://huggingface.co/docs/lerobot/main/inference)
- [LeRobot source repository](https://github.com/huggingface/lerobot)

This handbook was prepared on 30 July 2026. Because LeRobot evolves rapidly, preserve the exact release/commit used for every successful dataset and policy.

## 23. Second-run setup summary

This is the port mapping observed during the second run:

```text
Leader   -> port number 0 -> /dev/ttyACM0
Follower -> port number 1 -> /dev/ttyACM1
Camera   -> index 0       -> usually /dev/video0
```

These names are observations, not permanent hardware identities. Linux may change them after a reboot or USB reconnection. Confirm them before every hardware session:

```bash
ls -l /dev/ttyACM*
lerobot-find-port
ls -l /dev/video*
```

`lerobot-find-port` was not tested in the second run. It normally asks you to disconnect and reconnect a controller so it can identify the corresponding serial device. Follow the prompts printed by the installed version.

Useful references:

- [Seeed Studio SO-ARM100/SO-ARM101 guide](https://wiki.seeedstudio.com/lerobot_so100m_new/#teleoperate)
- [LeRobot v0.6.0 SO-101 guide](https://huggingface.co/docs/lerobot/v0.6.0/en/so101)

## 24. Ubuntu 22.04 installation used in the second run

### Requirements recorded in the notes

Ubuntu x86:

- Ubuntu 22.04
- CUDA 12+
- Python 3.10
- PyTorch 2.6+

Jetson Orin:

- JetPack 6.0 or 6.1
- JetPack 6.2 was not supported by the referenced Seeed guide at the time of the run
- Python 3.10
- PyTorch 2.3+

The commands below reproduce the Seeed-based Ubuntu x86 setup used in the notes. They differ from the newer Hugging Face installation in Section 3, so choose one checkout and do not mix the two environments.

### Install Miniforge

```bash
wget https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh
chmod +x Miniforge3-Linux-x86_64.sh
./Miniforge3-Linux-x86_64.sh

# Required only after the initial Miniforge installation:
source ~/.bashrc
conda init --all
```

### Create the environment and install LeRobot

```bash
conda create -y -n lerobot python=3.10
conda activate lerobot

git clone https://github.com/Seeed-Projects/lerobot.git ~/lerobot

conda install -y ffmpeg -c conda-forge

sudo apt update
sudo apt install -y build-essential

cd ~/lerobot
python -m pip install -e ".[feetech]"
```

Installation should now be complete. Verify which installation the shell is using:

```bash
which python
which lerobot-teleoperate
python --version
python -m pip show lerobot torch
git -C ~/lerobot rev-parse HEAD
```

## 25. Motor setup and calibration notes

Motor setup is normally a one-time operation for new or repurposed motors. During setup, use the port discovered on the current machine; it may appear as either `ttyUSB*` or `ttyACM*`.

Follower:

```bash
lerobot-setup-motors \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM1
```

Leader:

```bash
lerobot-setup-motors \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM0
```

The leader uses `--teleop.type`, not `--robot.type`, in the CLI used by this handbook.

### Inspect or remove an old calibration

First inspect the exact files:

```bash
ls -l ~/.cache/huggingface/lerobot/calibration/robots/so_follower/
ls -l ~/.cache/huggingface/lerobot/calibration/teleoperators/so_leader/
```

If a calibration definitely belongs to the arm being recalibrated, back it up instead of using a broad `find ... -delete` command:

```bash
mv ~/.cache/huggingface/lerobot/calibration/robots/so_follower/my_awesome_follower_arm.json \
   ~/.cache/huggingface/lerobot/calibration/robots/so_follower/my_awesome_follower_arm.json.backup

mv ~/.cache/huggingface/lerobot/calibration/teleoperators/so_leader/my_awesome_leader_arm.json \
   ~/.cache/huggingface/lerobot/calibration/teleoperators/so_leader/my_awesome_leader_arm.json.backup
```

If the file does not exist at those paths, locate it without deleting anything:

```bash
find ~/.cache ~/.local -type f -name '*my_awesome_follower_arm*' 2>/dev/null
find ~/.cache ~/.local -type f -name '*my_awesome_leader_arm*' 2>/dev/null
```

### Calibrate the follower and leader

Follower:

```bash
lerobot-calibrate \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM1 \
  --robot.id=my_awesome_follower_arm
```

Leader:

```bash
lerobot-calibrate \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM0 \
  --teleop.id=my_awesome_leader_arm
```

The missing leader ID in the raw notes has been restored here so the saved calibration can be reused reliably.

### Important alignment observation

Before pressing Enter to begin calibration, place every joint near the middle of its safe mechanical range. For the second-run workflow, the corresponding leader and follower joints were placed at approximately the same angles. Then sweep every joint through its full safe range as requested by the calibration program.

If a new calibration is not created after an old one is removed, check:

1. The leader/follower port mapping.
2. USB, motor power, and servo cable connections.
3. Whether another process owns the serial port.
4. Whether the CLI is using the intended environment and repository.

Power off before unplugging or reconnecting servo, controller, USB, or power cables.

### Local helper scripts from the Seeed checkout

The following modules were recorded in the notes. They are repository-specific helpers, not standard LeRobot CLI commands. Confirm that each module exists in the current checkout before running it:

```bash
cd ~/lerobot
test -f src/tools/servo_disable.py && echo 'servo_disable is available'
test -f src/tools/servo_middle_calibration.py && echo 'servo_middle_calibration is available'
test -f src/tools/servo_center_test.py && echo 'servo_center_test is available'
```

If available:

```bash
# Disable torque so joints can be moved by hand for calibration positioning.
python -m src.tools.servo_disable

# Treat the arm's current pose as its middle/offset pose.
python -m src.tools.servo_middle_calibration

# Command the arm to the saved middle pose to verify the offset.
python -m src.tools.servo_center_test
```

Keep the arm workspace clear before `servo_center_test`; it may move immediately.

### View saved calibration data

```bash
cat ~/.cache/huggingface/lerobot/calibration/robots/so_follower/my_awesome_follower_arm.json
cat ~/.cache/huggingface/lerobot/calibration/teleoperators/so_leader/my_awesome_leader_arm.json
```

## 26. Starting teleoperation from a new terminal

After the one-time Miniforge initialization, a normal new terminal does not need `conda init` or `source ~/.bashrc` again:

```bash
conda activate lerobot
cd ~/lerobot

# Temporary serial permission workaround for this session/device connection:
sudo chmod 666 /dev/ttyACM0 /dev/ttyACM1

lerobot-teleoperate \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM1 \
  --robot.id=my_awesome_follower_arm \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM0 \
  --teleop.id=my_awesome_leader_arm
```

Prefer the persistent `dialout` group setup from Section 4 when possible. `chmod 666` grants every local user read/write access to those serial devices and must be repeated when the device nodes are recreated.

## 27. Camera and system checks from the second run

Check disk, memory, and GPU state before recording or training:

```bash
df -h ~
df -h
free -h
nvidia-smi
```

If the repository contains the custom Rerun camera script:

```bash
cd ~/lerobot
test -f camera_rerun.py && python camera_rerun.py
```

Raw camera tests:

```bash
# 640x480 uncompressed/default mode
ffplay -f v4l2 -video_size 640x480 -framerate 30 /dev/video0

# 1920x1080 MJPEG mode
ffplay \
  -f v4l2 \
  -input_format mjpeg \
  -video_size 1920x1080 \
  -framerate 30 \
  /dev/video0
```

Quit `ffplay` and any camera preview before starting LeRobot recording or evaluation.

## 28. Confirmed dataset recording commands

The second-run data root was:

```text
/home/postgrad/fahim
```

The main 720p test configuration was:

```bash
lerobot-record \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM1 \
  --robot.id=my_awesome_follower_arm \
  --robot.cameras="{front: {type: opencv, index_or_path: 0, width: 1280, height: 720, fps: 30, fourcc: MJPG}}" \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM0 \
  --teleop.id=my_awesome_leader_arm \
  --display_data=true \
  --display_compressed_images=true \
  --dataset.repo_id=local/pick_place_second_720p_test \
  --dataset.root="/home/postgrad/fahim/datasets_2/pick_place_second_720p_test" \
  --dataset.single_task="pick the cube and place it inside the box" \
  --dataset.num_episodes=2 \
  --dataset.episode_time_s=30 \
  --dataset.reset_time_s=5 \
  --dataset.vcodec=h264 \
  --dataset.push_to_hub=false
```

The 640x480 configuration used for the longer second dataset was:

```bash
lerobot-record \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM1 \
  --robot.id=my_awesome_follower_arm \
  --robot.cameras="{front: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30, fourcc: MJPG}}" \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM0 \
  --teleop.id=my_awesome_leader_arm \
  --display_data=true \
  --display_compressed_images=true \
  --dataset.repo_id=local/pick_place_second_test \
  --dataset.root="/home/postgrad/fahim/datasets_2/pick_place_second_test" \
  --dataset.single_task="pick the cube and place it inside the box" \
  --dataset.num_episodes=2 \
  --dataset.episode_time_s=90 \
  --dataset.reset_time_s=10 \
  --dataset.vcodec=h264 \
  --dataset.push_to_hub=false
```

Press `Esc` only if the installed recorder prints that key as its pause/stop control. Key mappings can change between versions.

To add 20 episodes to that exact dataset:

```bash
lerobot-record \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM1 \
  --robot.id=my_awesome_follower_arm \
  --robot.cameras="{front: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30, fourcc: MJPG}}" \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM0 \
  --teleop.id=my_awesome_leader_arm \
  --display_data=true \
  --display_compressed_images=true \
  --dataset.repo_id=local/pick_place_second_test \
  --dataset.root="/home/postgrad/fahim/datasets_2/pick_place_second_test" \
  --dataset.single_task="pick the cube and place it inside the box" \
  --dataset.num_episodes=20 \
  --dataset.episode_time_s=90 \
  --dataset.reset_time_s=0 \
  --dataset.vcodec=h264 \
  --dataset.push_to_hub=false \
  --resume=true
```

The repo ID, root, task text, camera name, resolution, codec, robot IDs, and port mapping must match the existing dataset when resuming. In the observed CLI, `num_episodes` means additional episodes when `--resume=true`.

Visualize an episode:

```bash
lerobot-dataset-viz \
  --repo-id local/pick_place_second_test \
  --root "/home/postgrad/fahim/datasets_2/pick_place_second_test" \
  --mode local \
  --episode-index 0 \
  --display-compressed-images=true
```

Check `lerobot-dataset-viz --help` if this version expects the parent directory rather than the dataset directory for `--root`.

## 29. SmolVLA training used in the second run

The recorded 30,000-step command was:

```bash
conda activate lerobot
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

Do not reuse a non-empty output directory for a fresh run unless the installed CLI explicitly supports that workflow.

To exclude episode 14 when the dataset contains episode indices 0 through 70:

```bash
cd ~/lerobot

EPISODES="$(python -c 'import json; print(json.dumps([i for i in range(71) if i != 14]))')"

PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True lerobot-train \
  --policy.path=lerobot/smolvla_base \
  --dataset.repo_id=local/pick_place_second_test \
  --dataset.root="/home/postgrad/fahim/datasets_2/pick_place_second_test" \
  --dataset.episodes="$EPISODES" \
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
  --output_dir="/home/postgrad/fahim/outputs_second/train/smolvla_pick_place_second_without_ep14" \
  --job_name=smolvla_pick_place_second_without_ep14 \
  --wandb.enable=false \
  --policy.push_to_hub=false
```

The range `range(71)` is correct only for a 71-episode dataset indexed from 0 to 70. Confirm the dataset metadata before generating the list.

Verify saved checkpoints:

```bash
find "/home/postgrad/fahim/outputs_second" \
  -type d -name pretrained_model \
  -printf '%T@ %p\n' | \
  sort -nr
```

## 30. SmolVLA real-robot evaluation

This command evaluates the saved 30,000-step checkpoint and records one evaluation episode. The camera is named `camera1` to match the policy input after the training rename map:

```bash
conda activate lerobot
cd ~/lerobot

EVAL_ID="eval_smolvla_$(date +%Y%m%d_%H%M%S)"

lerobot-record \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM1 \
  --robot.id=my_awesome_follower_arm \
  --robot.cameras="{camera1: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30, fourcc: MJPG}}" \
  --policy.path="/home/postgrad/fahim/outputs_second/train/smolvla_pick_place_second/checkpoints/030000/pretrained_model" \
  --policy.empty_cameras=0 \
  --policy.resize_imgs_with_padding='[256,256]' \
  --policy.use_amp=true \
  --policy.device=cuda \
  --display_data=true \
  --display_compressed_images=true \
  --dataset.repo_id="local/${EVAL_ID}" \
  --dataset.root="/home/postgrad/fahim/datasets_2/${EVAL_ID}" \
  --dataset.single_task="pick the cube and place it inside the box" \
  --dataset.num_episodes=1 \
  --dataset.episode_time_s=30 \
  --dataset.reset_time_s=5 \
  --dataset.vcodec=h264 \
  --dataset.push_to_hub=false
```

The raw notes also included leader teleoperation arguments in an evaluation command. They are intentionally omitted here: autonomous policy evaluation should not send leader actions at the same time. Keep an emergency-stop method ready and clear the entire arm workspace before starting inference.

## 31. Second-run quick checklist

- [ ] `conda activate lerobot` succeeds.
- [ ] `which lerobot-teleoperate` points to the intended environment.
- [ ] Leader is positively identified as `/dev/ttyACM0` for this session.
- [ ] Follower is positively identified as `/dev/ttyACM1` for this session.
- [ ] Calibration JSON files exist for both exact IDs.
- [ ] Leader/follower teleoperation is smooth before adding a camera.
- [ ] `/dev/video0` supports the selected resolution, FPS, and MJPEG mode.
- [ ] Camera preview is closed before recording.
- [ ] `df -h`, `free -h`, and `nvidia-smi` show sufficient resources.
- [ ] Dataset root does not conflict with an unrelated existing dataset.
- [ ] Resume settings match the existing dataset exactly.
- [ ] Dataset episodes and videos are inspected before training.
- [ ] SmolVLA camera feature names match between training and evaluation.
- [ ] Output directory and checkpoint path are verified before evaluation.
- [ ] Robot workspace is clear and an emergency stop is ready.

This update was prepared on 1 August 2026 from the second-run field notes. Preserve the successful LeRobot commit, environment package list, dataset metadata, and checkpoint configuration together.
