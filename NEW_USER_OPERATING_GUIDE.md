# New User Operating Guide

## D455f + VL53L5CX + DA3/V6 ROS 2 depth-fusion system

**Audience:** A new researcher or engineer operating the existing device and software for
the first time.

**Tested platform:** Ubuntu 24.04, ROS 2 Jazzy, Python 3.12, Intel RealSense D455f,
STM32L476 and VL53L5CX.

## 1. Read this before starting

The deployed fusion system uses two sensor inputs:

1. D455f RGB, which is processed by frozen DA3; and
2. VL53L5CX multi-return ToF measurements, which are projected into the RGB image and
   supplied to the V4/V6 fusion network.

D455f depth is **not** a fusion input. It is enabled only when reference measurements and
MAE are required.

This GitHub repository currently contains documentation. To run the system, the operator
must also receive the complete project workspace from the maintainer. The workspace must
include the source code, ROS 2 packages, Python environment, DA3 model, V6 checkpoint,
mapping configuration and helper scripts described in the project report.

The STM32 board should already contain the tested 10 Hz firmware. Do not reflash it during
normal operation.

## 2. Expected equipment

- Intel RealSense D455f connected to a USB 3 port;
- VL53L5CX mounted rigidly beside the D455f;
- STM32L476 development board connected through ST-Link USB;
- the original calibrated sensor mount, without changing the relative sensor positions;
- Ubuntu 24.04 workstation with ROS 2 Jazzy; and
- the complete project workspace.

Moving either sensor relative to the other invalidates the existing projection calibration.

## 3. Tested software configuration

| Item | Tested value |
|---|---|
| ROS distribution | ROS 2 Jazzy |
| Python environment | `.venv312` |
| Camera stream | RGB 640×480 at 30 FPS |
| ToF stream | 8×8, four target slots, approximately 10 Hz |
| Serial protocol | `TOF4_BINARY_V1` with CRC32 |
| Serial baud rate | 1,000,000 |
| DA3 model | `DA3METRIC-LARGE` |
| Default DA3 process resolution | 168 |
| V6 input size | 160×120 |
| Default ToF prompt stride | five ToF frames, approximately 2 Hz |
| Preferred inference device | Intel XPU, with CPU fallback |

## 4. Daily start-up checklist

### Step 1: Connect the hardware

1. Keep the calibrated D455f–ToF mount rigid.
2. Connect the D455f directly to a USB 3 port.
3. Connect the STM32 ST-Link USB cable.
4. Close Intel RealSense Viewer if it is open. It can prevent ROS from opening the camera.
5. Close any older Python viewer or ROS launch process.

### Step 2: Open the project

Set the project location for the current terminal. Replace the example path if the workspace
has been installed elsewhere.

```bash
export TOF_PROJECT_ROOT=/home/dase-hw101/Documents/ChatGPT/tof
cd "$TOF_PROJECT_ROOT"
```

### Step 3: Confirm that both devices are visible

Check the RealSense camera:

```bash
rs-enumerate-devices
```

The output should identify an Intel RealSense D455f. Check the ToF serial interface:

```bash
ls -l /dev/ttyACM*
ls -l /dev/serial/by-id/
```

The stable serial path normally contains `STMicroelectronics_STM32_STLink` and ends in
`-if02`. Prefer this `/dev/serial/by-id/...` path because `/dev/ttyACM0` may change after a
reboot or reconnection.

If the serial device exists but access is denied, add the operator to the `dialout` group:

```bash
sudo usermod -aG dialout "$USER"
```

Log out and log in again before retrying.

### Step 4: Check the accelerator

```bash
./scripts/check_accelerator.sh
```

The tested fast path uses Intel XPU. If XPU is unavailable, `device:=auto` will fall back to
CPU, but the output will be slower.

### Step 5: Build the ROS 2 workspace

This is normally required only after installation or a source-code change:

```bash
./scripts/build_ros2.sh
```

Expected final message:

```text
ROS 2 workspace built.
```

## 5. Start the normal fusion viewer

For the first run, `/dev/ttyACM0` is the simplest choice:

```bash
./scripts/run_ros2_pipeline.sh \
  port:=/dev/ttyACM0 \
  baud:=1000000 \
  device:=auto \
  process_res:=168
```

For regular use, replace `/dev/ttyACM0` with the full path shown by:

```bash
ls -l /dev/serial/by-id/
```

The launch starts:

1. the RealSense camera node;
2. the ToF serial node;
3. the DA3/V6 fusion node; and
4. the fusion viewer.

Successful start-up normally includes messages similar to:

```text
RealSense Node Is Up!
ToF serial connection opened
V6 fusion ready; waiting for D455 and ToF topics
Inference worker started
```

Model loading may take several seconds. Wait for the inference-worker message before
judging the output.

## 6. Understand the normal viewer

The default viewer shows only the square V6 fusion crop. This is intentional.

- The crop contains V6 depth within valid projected ToF coverage.
- It is displayed as a square without stretching.
- D455f depth is not required.
- A crosshair marks the centre measurement area.
- The reported centre distance is the median valid depth in an 11×11 region.
- The viewer also reports the measured fused-output FPS.

The main output topics are:

| Topic | Meaning |
|---|---|
| `/fusion/v6_fusion_crop_m` | Square V6 depth crop displayed by the viewer |
| `/fusion/v6_fusion_depth_m` | Full-resolution V6 depth masked by ToF coverage |
| `/fusion/fusion_mask` | Exact valid fusion mask |
| `/fusion/center_distance_m` | Median centre-region distance in metres |
| `/tof/frame` | Parsed four-target VL53L5CX frame |

## 7. Enable D455f reference evaluation

Use this mode when comparing DA3, V6, ToF and D455f. It is slower because it enables
D455f depth, alignment, additional image topics, overlap statistics and the six-panel view.

```bash
./scripts/run_ros2_pipeline.sh \
  port:=/dev/ttyACM0 \
  baud:=1000000 \
  device:=auto \
  process_res:=168 \
  enable_reference_outputs:=true \
  viewer_topic:=/fusion/comparison
```

MAE is calculated only inside the requested overlap region. Pixels are ignored when D455f
reports zero, NaN or another invalid distance. D455f depth remains a reference and is not
passed into the fusion network.

## 8. Check live topic rates

Keep the pipeline running and open a second terminal:

```bash
export TOF_PROJECT_ROOT=/home/dase-hw101/Documents/ChatGPT/tof
cd "$TOF_PROJECT_ROOT"
source /opt/ros/jazzy/setup.bash
source .venv312/bin/activate
source ros2_ws/install/setup.bash
```

Measure the ToF rate:

```bash
ros2 topic hz /tof/frame
```

The expected result is approximately 10.1 Hz. Press `Ctrl-C` after collecting enough
samples.

Measure the final fusion rate:

```bash
ros2 topic hz /fusion/v6_fusion_crop_m
```

The expected steady rate is approximately 5 FPS. A measured 30-second sample averaged
4.51 FPS because one Intel XPU inference pause lasted 1.46 seconds.

Read one centre-distance message:

```bash
ros2 topic echo /fusion/center_distance_m --once
```

Do not confuse the 10 Hz ToF topic with the final fusion rate. DA3 and V6 run sequentially,
so the complete fusion pipeline is slower than the sensor stream.

## 9. Prompt refresh options

The default `tof_prompt_stride:=5` rebuilds the prompt approximately twice per second while
reusing it for intermediate fusion frames. This provides the best tested balance between
responsiveness and CPU load.

For faster-moving scenes, update the prompt more often:

```bash
./scripts/run_ros2_pipeline.sh \
  port:=/dev/ttyACM0 \
  baud:=1000000 \
  device:=auto \
  process_res:=168 \
  tof_prompt_stride:=2
```

This increases prompt freshness to approximately 5 Hz but uses more CPU. It does not make
DA3/V6 inference run at 10 FPS.

## 10. Record a ROS 2 bag

Start the required pipeline first, then open a second configured ROS terminal. A compact
deployment recording can be created with:

```bash
ros2 bag record \
  /tof/frame \
  /camera/camera/color/image_raw \
  /camera/camera/color/camera_info \
  /fusion/v6_fusion_crop_m \
  /fusion/fusion_mask \
  /fusion/center_distance_m
```

For reference experiments, start the reference mode and also record:

```bash
ros2 bag record \
  /tof/frame \
  /camera/camera/color/image_raw \
  /camera/camera/aligned_depth_to_color/image_raw \
  /camera/camera/color/camera_info \
  /fusion/da3_depth_m \
  /fusion/v6_depth_m \
  /fusion/d455_depth_m \
  /fusion/tof_depth_m \
  /fusion/overlap_mask \
  /fusion/v6_overlap_depth_m \
  /fusion/overlap_stats
```

Stop recording with `Ctrl-C` and record the scene description, sensor position and test
purpose in a separate experiment log.

## 11. Standalone viewer fallback

The earlier non-ROS viewer remains available:

```bash
PYTHONPATH=src .venv312/bin/python src/live_v6_viewer.py \
  --port /dev/ttyACM0 \
  --baud 1000000 \
  --device auto \
  --process-res 168
```

Do not run the standalone viewer and ROS pipeline at the same time. Both attempt to open the
same camera and serial device.

## 12. Stop the system safely

1. Focus the terminal that started the pipeline.
2. Press `Ctrl-C` once.
3. Wait until the camera, serial and fusion processes report that they have stopped.
4. Close the viewer window if it remains open.
5. Disconnect the hardware only after the processes have ended.

Do not use `kill -9` during normal shutdown because it can leave camera or serial resources
in an unclear state.

## 13. Troubleshooting

### D455f is not found or is busy

- Close Intel RealSense Viewer.
- Stop any previous ROS launch or Python viewer.
- Reconnect the camera to a USB 3 port.
- Confirm detection with `rs-enumerate-devices`.

### `/dev/ttyACM0` does not exist

- Check `ls -l /dev/serial/by-id/`.
- Reconnect the STM32 ST-Link USB cable.
- Use the stable by-id path instead of assuming the device is `ttyACM0`.

### Serial permission is denied

Add the user to `dialout`, then log out and back in:

```bash
sudo usermod -aG dialout "$USER"
```

### No ToF frames appear

- Confirm that the launch uses `baud:=1000000`.
- Confirm that the board contains the current 10 Hz binary firmware.
- Check the physical sensor and I²C connections.
- Do not change to 115200 unless the older ASCII firmware has deliberately been restored.

### “No valid overlap reference” appears

- Wait for at least three ToF frames after start-up.
- Place a valid object inside the calibrated ToF field of view.
- Check `/tof/frame` with `ros2 topic hz`.
- Remember that D455f depth is optional in normal deployment mode; fusion is based on V6
  depth and projected ToF coverage.

### XPU is unavailable

Run:

```bash
./scripts/check_accelerator.sh
```

Keep `device:=auto` for automatic CPU fallback, or use `device:=cpu` explicitly. CPU mode is
expected to be slower.

### The fused FPS is below the ToF rate

This is expected. The sensor publishes at approximately 10 Hz, but DA3, V6, preprocessing
and ROS publication form a sequential pipeline. Normal steady fusion is approximately
5 FPS on the tested Intel XPU.

### RealSense prints “Parameter ... is not supported” warnings

The included RealSense launch file may inspect top-level launch arguments and print these
warnings during start-up. If the camera later reports `RealSense Node Is Up!`, the warnings
are not normally fatal.

### The model or checkpoint is missing

Confirm that these local artifacts have been provided:

- `third_party/models/DA3METRIC-LARGE`;
- `outputs/plane_v1/prompt_adapter_v6_boundary_router_all98_deployment/checkpoint_calibrated.pt`; and
- `config/tof_rgb_mapping_plane_v1.json`.

## 14. Firmware and recovery warning

Normal users should not flash the STM32. The tested board already runs the 10 Hz firmware.
Before the upgrade, the complete 1 MiB MCU flash and original STM32 project were backed up.

Important checksums:

| Artifact | SHA-256 |
|---|---|
| Pre-upgrade 1 MiB flash | `ad953313b21901c84e538c2410e44d412a8aed2d1392d5426ec2d234f6412cdb` |
| Final 10 Hz firmware binary | `cdc9c104ca77cc61fac771d0265f1b2ad6bad625a05e5d08200fdb555c05f629` |

Recovery instructions are stored in
`firmware_backups/pre_10hz_20260930/RESTORE.md` in the operational project tree. Firmware
restoration should be performed only by the maintainer or a user familiar with STM32 SWD.

## 15. Reporting results responsibly

- State whether the run used deployment mode or D455f reference mode.
- Report ToF topic rate and fused-output rate separately.
- Do not describe development-set MAE as final generalisation performance.
- Exclude invalid D455f pixels from reference MAE.
- Record any change to process resolution, V6 size, prompt stride, firmware or calibration.
- Do not claim equivalence to a D435i or D455 depth camera without a controlled benchmark.

For model history, calibration results and quantitative evidence, read
[the full project progress report](PROJECT_PROGRESS_REPORT.md).
