# SO-ARM101 Local Training and Troubleshooting Handbook

This handbook explains how to take an SO-ARM101 from hardware checks to a locally trained ACT policy, then diagnose the failures seen in the supplied ChatGPT conversations.

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
