# Project Progress Report: Lightweight ToF–RGB Dense Depth Fusion

| Report field | Details |
|---|---|
| Date | 30 September 2026 |
| Project stage | Operational ROS 2 research prototype; final V6 generalisation test still required |
| Hardware | Intel RealSense D455f and VL53L5CX 8×8 multi-zone ToF sensor |
| Purpose | Technical progress review and reproducible handover |

## Executive summary

This project investigates whether a very low-resolution Time-of-Flight (ToF) sensor can
improve the dense depth estimated from a single RGB image. The RGB model, Depth Anything 3
(DA3), produces a detailed depth map, but its metric distance can be inaccurate. The
VL53L5CX provides only an 8×8 grid, but its valid measurements contain direct distance
information. Our goal is to combine their strengths while keeping the trainable fusion
network small.

The complete prototype is now operational in ROS 2 Jazzy. The sensors have been mounted
and calibrated, and the latest STM32 firmware preserves up to four distance returns per
zone while publishing at 10 Hz through a CRC32-protected binary protocol. The documented
dataset contains 98 compatible real captures assigned to 48 manifest-defined scene groups.
A frozen DA3 model provides the dense prior. The V4 fusion network has 599,476 trainable
parameters, while V6 adds an 11,691-parameter full-resolution router for difficult
multi-depth regions. D455f aligned depth is used only as a training and evaluation
reference; it is never given to the fusion model at inference time.

On a reused 15-capture development set, V4 reduced all-pixel mean absolute error (MAE) from
13.89 cm for DA3 to 10.79 cm. V6 further reduced it to 10.67 cm. The V6 improvement is more
visible inside the ToF footprint and in multi-return regions than over the whole image. A
live ROS 2 demonstration is now working on the laptop's Intel integrated GPU. The ToF topic
was measured at 10.14 Hz. The fused output normally runs at about 5 FPS; a 30-second test
averaged 4.51 FPS because one XPU inference pause lasted 1.46 seconds. These are separate
rates: faster sensor delivery does not make the sequential DA3 and V6 computation run at
10 FPS.

The result is promising but not yet equivalent to a D435i or D455 depth camera. The main
remaining scientific requirement is a strictly untouched test set for V6. The current
visual observation—that V6 is only slightly better than frozen DA3 over the full image—is
consistent with the measured results and motivates stronger feature-level ToF fusion in a
future model. The runtime work reported here improves delivery and responsiveness; it does
not change the previously reported model-accuracy results.

### Update since the 28 September report

- The live application has been separated into ROS 2 camera, serial, fusion and viewer nodes.
- The VL53L5CX firmware now delivers four-target 8×8 frames at a measured 10.11–10.14 Hz.
- The deployment viewer now reports a square overlap crop and centre distance; the optional
  reference mode retains D455f comparison and overlap-only MAE.
- Prompt construction has been reduced from about 903 ms to 47 ms without changing its
  pixel values.
- Steady fused output is around 5 FPS on Intel XPU, although occasional inference pauses
  reduce the 30-second measured average to 4.51 FPS.
- The automated verification suite has increased from 105 to 116 passing tests.

## 1. Research question and motivation

Dense monocular depth and lightweight ToF sensing have complementary strengths:

- **DA3 dense depth:** high spatial resolution and strong object structure, but imperfect
  absolute scale and distance.
- **VL53L5CX ToF:** direct metric measurements, but only 64 zones, with missing values and
  mixed foreground/background returns near boundaries.

The research question is:

> Can sparse multi-return ToF measurements correct a pretrained RGB depth model without
> training a large depth network from scratch?

The work is inspired by DELTAR's learned RGB–ToF fusion and by prompt-based depth models.
However, the current V4/V6 architecture is our own lightweight external fusion design; it
is not presented as a full reproduction of either reference.

## 2. Current system

```text
D455f RGB (30 FPS) ──> frozen DA3 ──> dense metric-depth prior ───┐
                                                                   │
VL53L5CX ──> STM32 binary stream ──> ROS /tof/frame ──> rolling    │
window ──> quality filtering ──> RGB projection ──> cached prompt ┤
                                                                   ▼
                                                V4 local fusion network
                                                                   ▼
                                                V6 pixel-level router
                                                                   ▼
                                dense V6 depth + fusion mask + square crop
                                                                   │
                                                                   ▼
                                                   centre-distance estimate

D455f aligned depth ──> optional training/evaluation reference only
```

The deployed input consists of RGB and live VL53L5CX measurements. DA3, V4 and V6 do not
receive D455f depth. This separation is important because it prevents the reference camera
from secretly solving the task during deployment. The fusion mask is formed from valid
projected ToF coverage and valid V6 depth. When D455f reference evaluation is enabled, its
invalid or missing pixels are excluded from MAE rather than interpreted as zero distance.

## 3. Work completed

### 3.1 Hardware integration and calibration

- The D455f and VL53L5CX are fixed on one rigid mount.
- From the back of the rig, the ToF sensor is approximately 1.3 cm left, 3 cm above and
  1 cm behind the RGB camera centre.
- Directional target experiments established that the raw ToF grid must be rotated 90°
  counter-clockwise before projection into the RGB image.
- Multi-pose planar calibration estimates the ToF field of view and rigid transform.
- On an independent diagonal-plane check, mapping MAE fell from 5.42 cm to 1.45 cm.
- Camera/serial ownership, reconnect and clean shutdown problems were addressed in the
  capture and live-viewer workflow.

The grid direction is reliable, but the exact cone of each ToF zone, lens distortion and
occlusion geometry remain sources of error.

### 3.2 ToF firmware and data preservation

The firmware was upgraded from a simple one-distance stream to a structured multi-target
format. At 8×8 resolution, each zone can preserve up to four native returns together with:

- target status and number of detected targets;
- range uncertainty (`sigma`);
- signal, ambient light and reflectance;
- sensor temperature and timestamps where available.

This is important at object boundaries. A zone may observe both a near object and a far
background, so replacing all returns with one average would remove useful information.

The latest firmware and transport configuration is as follows:

- 10 Hz VL53L5CX ranging with a 15 ms integration period;
- four integrations per 8×8 frame, while retaining four target slots per zone;
- STM32L476 running from a 32 MHz MSI clock;
- approximately 1 MHz I²C Fast-mode Plus;
- interrupt-driven UART at 1,000,000 baud;
- a 2,896-byte `TOF4_BINARY_V1` frame protected by CRC32; and
- host-side resynchronisation and backward-compatible parsing of the earlier text protocol.

A connected-board raw serial test recorded 151 frames in 15 seconds, corresponding to
10.109 Hz. ROS 2 measurements were 10.11–10.14 Hz. Before flashing, the complete 1 MiB MCU
flash and the original STM32 project were backed up. The exact pre-experiment flash image
has SHA-256
`ad953313b21901c84e538c2410e44d412a8aed2d1392d5426ec2d234f6412cdb`.
The final 10 Hz binary has SHA-256
`cdc9c104ca77cc61fac771d0265f1b2ad6bad625a05e5d08200fdb555c05f629`.

### 3.3 Data collection and preparation

The current real dataset contains **98 unique compatible captures assigned to 48
manifest-defined scene groups**. This count was checked directly against the eight real-data
manifests used by the all-data fit: 28 calibration/fusion captures in 10 groups, 45 captures
from three 15-capture development sets in 15 groups, and 25 additional calibration and
mapping captures carrying 25 group labels. Two labels in the last set already occur in the
first set, giving 48 unique labels rather than 50.

Here, a *scene group* is a dataset split identifier, normally formed by removing the repeat
suffix from a case name. It must not be interpreted as 48 completely independent rooms or
object arrangements: some calibration groups represent the same rig or board at different
distances, positions or poses. The dataset includes planar boards, two- and three-layer
arrangements, small objects, dark objects, chairs, oblique walls and off-centre targets.
Repeated captures were collected to measure stability rather than selecting one favourable
reading.

The training pipeline also includes **200 public DIODE RGB-D images**. Dense public ground
truth is converted into simulated 8×8 multi-return prompts using a simulator profile fitted
from real VL53L5CX statistics. Public data increases scene variety, but it cannot reproduce
all real effects such as multipath interference, reflectance failures and boundary mixing.

Each training sample stores RGB, frozen DA3 depth, D455f reference depth, validity masks,
projected ToF hypotheses and sensor-quality features. Repeated captures from one physical
scene are grouped to reduce train/validation leakage.

### 3.4 Models and experiments

Several approaches were implemented and compared:

- **Geometric scale correction:** useful on simple planes, but it leaks foreground distance
  into the background near mixed-depth boundaries.
- **V1/V2 adapters:** introduced learned residual correction and explicit ToF candidate
  selection.
- **V3/V3-B attention and decoder prompting:** more flexible, but overfitted across the small
  real dataset. Adding complexity did not guarantee better generalisation.
- **Official DELTAR reproduction:** the released model was reproduced successfully, but
  direct transfer to our hardware performed poorly because its training domain and input
  assumptions differ from our rig.
- **V4 direct fusion:** preserves the local relation between RGB patches and physical ToF
  zones, selects from real sensor returns, and propagates information through image
  features. It has **599,476 trainable parameters**.
- **V5 DA3-feature ablation:** added 167,088 adapter parameters for frozen DA3 decoder
  features, but did not outperform V4. The safe selected checkpoint therefore remained V4.
- **V6 boundary router:** freezes DA3 and V4, then adds **11,691 trainable parameters**. For
  every covered pixel, it chooses whether to keep V4 or use one of up to four ToF returns.
  The total V4+V6 trainable size is **611,167 parameters**.

The main lesson is that retaining multiple ToF distances is not enough. The difficult task
is **candidate-to-pixel assignment**: deciding which physical return belongs to each RGB
pixel inside a large ToF zone.

### 3.5 ROS 2 deployment and runtime optimisation

The live system has been reorganised as a ROS 2 Jazzy pipeline with separate responsibilities:

- the RealSense driver publishes D455f RGB and, when requested, aligned reference depth;
- a serial node parses the CRC-protected ToF stream and publishes `/tof/frame`;
- the fusion node runs frozen DA3, builds or reuses the latest ToF prompt, executes V4/V6,
  and publishes depth, mask, crop and centre-distance topics; and
- the viewer subscribes to either the deployment crop or the optional six-panel reference
  image.

Camera inference is not clocked by ToF arrival. Prompt preparation runs separately, and the
latest valid prompt is reused until a newer prompt is ready. By default, the prompt is
rebuilt every five ToF frames, which is approximately 2 Hz with the 10 Hz firmware. This
keeps the displayed fusion output moving between sensor updates, although it introduces up
to about 0.5 seconds of prompt latency for a newly moving surface.

Several runtime changes were verified:

- prompt rasterisation was reduced from about 903 ms to 47 ms, with a unit test confirming
  pixel-equivalent depth, probability, confidence, validity and native-quality rasters;
- prompt tensors are cached on the Intel XPU and updated in place;
- large ROS image messages are serialised only when a subscriber exists;
- DA3 log noise is suppressed during the normal launch; and
- serial and fusion nodes perform guarded shutdown to avoid double-shutdown errors.

These changes improve throughput and responsiveness without changing checkpoint weights,
DA3 process resolution or the V6 spatial input size.

## 4. Main quantitative results

MAE is mean absolute error; lower values are better.

### 4.1 Reused complex development set

The following 15 captures come from complex, scene-separated groups, but they have already
influenced architecture and threshold choices. They are therefore development evidence,
not an unbiased final test.

| Evaluation region | Frozen DA3 | V4 | Calibrated V6 |
|---|---:|---:|---:|
| All valid pixels | 13.89 cm | 10.79 cm | **10.67 cm** |
| Inside ToF coverage | 11.28 cm | 5.84 cm | **5.48 cm** |
| Outside ToF coverage | 15.26 cm | **13.38 cm** | **13.38 cm** |
| Ground-truth depth boundaries | 32.12 cm | 30.82 cm | **30.28 cm** |
| Multi-hypothesis pixels | 18.43 cm | 8.14 cm | **7.40 cm** |

Compared with V4, V6 improves all-pixel MAE by 1.15%, ToF-region MAE by 6.17%, boundary
MAE by 1.76%, and multi-hypothesis MAE by 9.09%. Pixels outside the projected ToF coverage
are unchanged by design.

### 4.2 Final fit using all existing real data

After selecting the architecture and gate on development data, the deployment candidate was
fitted using all 98 real captures and 200 public samples. Since all 98 real captures took
part in optimisation, these numbers only confirm that the training pipeline works; they do
not measure generalisation.

| Region on 98 seen real captures | V4 fit | V6 fit |
|---|---:|---:|
| All valid pixels | 14.69 cm | **14.59 cm** |
| Inside ToF coverage | 3.01 cm | **2.73 cm** |
| Outside ToF coverage | **20.74 cm** | **20.74 cm** |
| Ground-truth depth boundaries | 33.39 cm | **33.17 cm** |
| Multi-hypothesis pixels | 3.71 cm | **3.26 cm** |

The modest full-image gain is expected. The 8×8 ToF footprint covers only part of the image,
and the safety gate deliberately avoids large corrections when evidence is uncertain.

## 5. Live prototype and measured performance

### 5.1 Deployment view

The default ROS 2 launch uses the faster fusion-only path. D455f reference depth, alignment
and MAE calculation are disabled. The viewer displays a square crop of V6 depth within the
valid ToF fusion footprint, with a centre crosshair, an 11×11 median centre-distance
estimate and the measured output FPS. The crop is the smallest square that contains the
fusion footprint; it is not stretched into a rectangle, and invalid black pixels outside
the crop are not included merely to fill the panel.

The primary runtime topics are:

- `/fusion/v6_fusion_crop_m`: square V6 depth crop used by the viewer;
- `/fusion/v6_fusion_depth_m`: full-resolution V6 depth masked by ToF coverage;
- `/fusion/fusion_mask`: exact ToF/V6 fusion mask; and
- `/fusion/center_distance_m`: centre-region median depth.

### 5.2 Reference and overlap view

The optional reference mode displays six synchronised panels and publishes DA3, V4, V6,
ToF, D455f and overlap diagnostics. The overlap MAE is calculated only where all required
values are valid. In particular, pixels with zero, NaN or otherwise invalid D455f depth are
ignored. This prevents D455f holes from being counted as large depth errors.

The displayed fusion region belongs to V6 and projected ToF coverage. D455f depth does not
control whether fusion occurs; it is used only to compare corresponding pixels when
reference mode is enabled.

### 5.3 Runtime measurements

The earlier CPU viewer remains a useful historical baseline:

- DA3 process resolution 504: approximately 3.3–4.0 seconds per update; and
- DA3 process resolution 252: approximately 0.5–0.8 seconds per update.

The current ROS 2 deployment uses `process_res=168`, a 160×120 V6 input and the Intel
integrated GPU through PyTorch XPU. Isolated DA3-Large inference at this setting was about
0.094 seconds on XPU and 0.262 seconds on CPU after warm-up. The complete pipeline is slower
because DA3 and V6 run sequentially and the result also requires preprocessing, tensor
transfer, prompt fusion, resizing and ROS publication.

The connected-system measurements were:

| Measurement | Result |
|---|---:|
| Raw/ROS ToF rate | 10.11–10.14 Hz |
| Normal steady fused output | approximately 5 FPS |
| 30-second fused-output average | 4.51 FPS |
| Longest pause in that sample | 1.46 seconds |
| Prompt construction before optimisation | approximately 903 ms |
| Prompt construction after optimisation | approximately 47 ms |

The occasional long pause remains associated with DA3/XPU execution. Reducing prompt
frequency did not remove it, and the kernel reported no GPU reset or fault during the
diagnostic check. Therefore, the current system should be described as approximately 5 FPS
in steady operation, not as a guaranteed 10 FPS fusion system.

The visual result still looks only slightly better than frozen DA3 over the whole image.
This agrees with the development results: the strongest improvements are local, especially
where valid ToF measurements or multiple returns are available.

## 6. What did not work, and why it matters

- Hand-designed weights could not reliably separate foreground and background within one
  ToF zone.
- Global attention sometimes ignored physical spatial correspondence and over-corrected
  unrelated image regions.
- Decoder prompting with limited real data overfitted and failed to generalise in a frozen
  test, especially outside the ToF footprint.
- Extra frozen DA3 decoder features did not improve V4.
- Simulated-only training transferred poorly to real VL53L5CX measurements.
- Preserving four ToF returns without a good pixel-level assignment mechanism produced
  little benefit.

These negative results are useful because they narrow the research direction. More
parameters or more epochs alone are unlikely to solve the problem. Better spatial
assignment, boundary supervision and simulation-to-real transfer are more important.

## 7. Current limitations

1. **No untouched V6 final test:** the strongest V6 numbers are development or fitting
   diagnostics.
2. **Very sparse ToF input:** 64 zones cannot directly reproduce fine RGB boundaries.
3. **Mixed-depth zones:** one zone may contain foreground, background and invalid returns.
4. **Calibration and synchronisation:** the sensors are spatially calibrated but not
   hardware-synchronised; dynamic objects remain difficult.
5. **Reference uncertainty:** D455f depth also contains holes and errors near occlusions,
   reflective surfaces and thin objects.
6. **Limited correction outside ToF coverage:** the safe V6 design keeps V4 unchanged there.
7. **No direct D435i benchmark:** the project has not yet demonstrated the target of reaching
   90% of D435i performance.
8. **Runtime variability:** steady fusion is close to 5 FPS, but occasional Intel XPU pauses
   reduce the long-window average and remain under investigation.
9. **Prompt freshness trade-off:** rebuilding the prompt every five ToF frames improves
   throughput but can delay the response to a newly moving surface by about 0.5 seconds.

## 8. Next steps

### Immediate: frozen evaluation

The current checkpoint and confidence threshold should remain unchanged while collecting a
new test set. Recommended scenes are:

- a thin or slanted foreground object with a distant background;
- three clearly separated depth layers;
- an object smaller than one ToF zone;
- dark and bright/reflective objects;
- at least three repeats per physical scene.

Results should be reported for the whole image, inside and outside ToF coverage, depth
boundaries and multi-return regions. Failed scenes must not be deleted after seeing the
output.

### Next model direction

If frozen testing confirms that V6 gives only a small gain, the most valuable architectural
change is **ToF-conditioned feature fusion**. Instead of making only a final correction to
the depth map, ToF tokens would guide high-resolution image features earlier in a lightweight
decoder or propagation network. This is closer to DELTAR's learned reasoning while still
keeping DA3 frozen. It will require more varied real boundary data, careful
simulation-to-real fine-tuning and a clean test protocol. An H100 would speed up this larger
experiment, but better data and evaluation design remain more important than raw compute.

### Runtime engineering

The next deployment task is to profile the remaining irregular XPU pause around DA3 model
execution. Any proposed optimisation should be assessed against both output rate and depth
accuracy. Lower DA3 resolutions or a smaller backbone may increase FPS, but they can reduce
small-object and boundary quality. Optimisations that preserve the current model and
resolution should therefore be tested first.

### Hardware comparison

A D435i should be recorded in the same static scenes and mapped into the same RGB coordinate
system. Only then can the “90% of D435i performance” target be defined and measured fairly.

## 9. Reproducibility and current artifacts

This public repository currently contains the project documentation. The paths below refer
to the operational local project tree and identify the exact implementation and artifacts
used for the reported results; they are not all included in this documentation repository.

### 9.1 Model and calibration artifacts

- Final V6 checkpoint:  
  `outputs/plane_v1/prompt_adapter_v6_boundary_router_all98_deployment/checkpoint_calibrated.pt`
- Checkpoint SHA-256:  
  `9fd8a126696f0f1648785ac15f05f6e27385fb4f7e142b809fd62f8ce0834b2c`
- V6 model: `src/prompt_adapter_v6.py`
- V6 training: `src/train_prompt_adapter_v6.py`
- V6 inference: `src/run_prompt_adapter_v6.py`
- Live comparison and shared fusion worker: `src/live_v6_viewer.py`
- Mapping configuration: `config/tof_rgb_mapping_plane_v1.json`
- Environment guide: `DOCKER.md`

### 9.2 ROS 2 and firmware artifacts

- ROS 2 package: `ros2_ws/src/d455_tof_fusion`
- ROS 2 messages: `ros2_ws/src/d455_tof_msgs`
- Pipeline launch file: `ros2_ws/src/d455_tof_fusion/launch/live_v6_pipeline.launch.py`
- Build helper: `scripts/build_ros2.sh`
- Run helper: `scripts/run_ros2_pipeline.sh`
- Accelerator check: `scripts/check_accelerator.sh`
- 10 Hz firmware source: `firmware/stm32_tof_10hz`
- Final firmware binary: `firmware_builds/stm32_tof_10hz_4target_20260930.bin`
- Pre-experiment recovery instructions: `firmware_backups/pre_10hz_20260930/RESTORE.md`

The current codebase passes **116 automated unit tests**. The test set includes binary
stream fragmentation, CRC failure and resynchronisation, compatibility with the earlier
text protocol, four-target preservation, overlap-only MAE, square crop behaviour, and
pixel equivalence between the original and optimised prompt rasterisers.

### 9.3 Running the ROS 2 pipeline

Build the workspace once:

```bash
./scripts/build_ros2.sh
```

Close RealSense Viewer and any previous camera or serial process, then run:

```bash
./scripts/run_ros2_pipeline.sh \
  port:=/dev/ttyACM0 \
  baud:=1000000 \
  device:=xpu \
  process_res:=168
```

A stable `/dev/serial/by-id/...` path is preferable to `/dev/ttyACM0` when available. To
enable D455f reference depth, overlap statistics, MAE and the six-panel comparison view:

```bash
./scripts/run_ros2_pipeline.sh \
  port:=/dev/ttyACM0 \
  baud:=1000000 \
  device:=xpu \
  process_res:=168 \
  enable_reference_outputs:=true \
  viewer_topic:=/fusion/comparison
```

The earlier standalone viewer remains available with the new binary protocol:

```bash
PYTHONPATH=src .venv312/bin/python src/live_v6_viewer.py \
  --port /dev/ttyACM0 --baud 1000000 --device xpu --process-res 168
```

To use the earlier ASCII firmware, the board must first be restored from the corresponding
backup and the host baud rate must be changed back to 115200.

## Current conclusion

The project has progressed from a simple scale-correction idea to a complete hardware,
data, training, evaluation and ROS 2 live-inference pipeline. Sparse ToF information clearly
helps within valid coverage, and explicit multi-return routing improves difficult regions.
The upgraded sensor path now delivers 10 Hz ToF data without blocking camera inference, and
the full fusion pipeline normally produces about 5 FPS on the integrated Intel GPU.

However, the latest V6 gain over V4 remains small over the whole image, fine boundaries are
still the central modelling weakness, and occasional XPU pauses reduce runtime consistency.
The project is technically functional and scientifically informative, but a new untouched
test set and a stronger feature-level fusion method are still required before making claims
comparable with a commercial RGB-D camera.
