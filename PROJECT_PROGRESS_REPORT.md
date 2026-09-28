# Project Progress Report: Lightweight ToF–RGB Dense Depth Fusion

**Report date:** 28 September 2026  
**Project stage:** Working research prototype; final V6 generalisation test still required  
**Hardware:** Intel RealSense D455f and VL53L5CX 8×8 multi-zone ToF sensor

## Executive summary

This project investigates whether a very low-resolution Time-of-Flight (ToF) sensor can
improve the dense depth estimated from a single RGB image. The RGB model, Depth Anything 3
(DA3), produces a detailed depth map, but its metric distance can be inaccurate. The
VL53L5CX provides only an 8×8 grid, but its valid measurements contain direct distance
information. Our goal is to combine their strengths while keeping the trainable fusion
network small.

The complete prototype is now operational. The sensors have been mounted and calibrated,
the ToF firmware preserves up to four distance returns per zone, and 98 compatible real
captures have been collected across 48 scene groups. A frozen DA3 model provides the dense
prior. The current V4 fusion network has 599,476 trainable parameters, and V6 adds an
11,691-parameter full-resolution router for difficult multi-depth regions. D455f aligned
depth is used only as a training and evaluation reference; it is never given to the fusion
model at inference time.

On a reused 15-capture development set, V4 reduced all-pixel mean absolute error (MAE) from
13.89 cm for DA3 to 10.79 cm. V6 further reduced it to 10.67 cm. The V6 improvement is more
visible inside the ToF footprint and in multi-return regions than over the whole image. A
live four-panel demonstration is working, although CPU inference is not real-time: the
quality setting takes about 3.3–4.0 seconds per update, while the fast preview setting takes
about 0.5–0.8 seconds.

The result is promising but not yet equivalent to a D435i or D455 depth camera. The main
remaining requirement is a strictly untouched test set for V6. The current visual
observation—that V6 is only slightly better than frozen DA3 over the full image—is consistent
with the measured results and motivates stronger feature-level ToF fusion in the next model.

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
RGB image ──> frozen DA3 ──> dense metric-depth prior ───────────┐
                                                                  │
8×8 multi-return ToF ──> quality filtering ──> RGB projection ───┤
                                                                  ▼
                                               V4 local fusion network
                                                                  ▼
                                               V6 pixel-level router
                                                                  ▼
                                                final dense depth map

D455f aligned depth ──> training/evaluation reference only
```

The deployed input consists of RGB and live VL53L5CX measurements. DA3, V4 and V6 do not
receive D455f depth. This separation is important because it prevents the reference camera
from secretly solving the task during deployment.

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

### 3.3 Data collection and preparation

The current real dataset contains **98 compatible captures from 48 scene groups**. It
includes planar boards, two- and three-layer arrangements, small objects, dark objects,
chairs, oblique walls and off-centre targets. Repeated frames were collected to measure
stability rather than selecting one favourable reading.

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

## 5. Live prototype

The live program displays four synchronised panels:

1. D455f RGB;
2. frozen DA3 depth;
3. final V6 depth;
4. D455f aligned reference depth.

All depth panels use a common metric colour scale. The D455f reference is used only for
display and live error reporting. On the current CPU laptop:

- DA3 process resolution 504: approximately 3.3–4.0 seconds per update;
- DA3 process resolution 252: approximately 0.5–0.8 seconds per update.

The faster setting is appropriate for demonstrations, but it loses some boundary detail.
The live result currently looks only slightly better than frozen DA3 over the full image.
This qualitative observation agrees with the development table: the strongest improvements
are local, particularly where valid ToF measurements or multiple returns are available.

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
keeping DA3 frozen. It will require more varied real boundary data and careful simulation-to-
real fine-tuning. An H100 would speed up this larger experiment, but better data and a clean
test protocol remain more important than raw compute.

### Hardware comparison

A D435i should be recorded in the same static scenes and mapped into the same RGB coordinate
system. Only then can the “90% of D435i performance” target be defined and measured fairly.

## 9. Reproducibility and current artifacts

- Final V6 checkpoint:  
  `outputs/plane_v1/prompt_adapter_v6_boundary_router_all98_deployment/checkpoint_calibrated.pt`
- Checkpoint SHA-256:  
  `9fd8a126696f0f1648785ac15f05f6e27385fb4f7e142b809fd62f8ce0834b2c`
- V6 model: `src/prompt_adapter_v6.py`
- V6 training: `src/train_prompt_adapter_v6.py`
- V6 inference: `src/run_prompt_adapter_v6.py`
- Live comparison: `src/live_v6_viewer.py`
- Mapping configuration: `config/tof_rgb_mapping_plane_v1.json`
- Environment guide: [`DOCKER.md`](DOCKER.md)

The current codebase passes **105 automated unit tests**.

To run the fast live demonstration:

```bash
PYTHONPATH=src .venv312/bin/python src/live_v6_viewer.py \
  --port /dev/ttyACM0 --baud 115200 --device cpu --process-res 252
```

## Current conclusion

The project has progressed from a simple scale-correction idea to a complete hardware,
data, training, evaluation and live-inference pipeline. Sparse ToF information clearly helps
in its valid coverage area, and explicit multi-return routing improves difficult regions.
However, the latest V6 gain over V4 is small over the full image, and fine boundaries remain
the central weakness. The project is technically functional and scientifically informative,
but a new untouched test and a stronger feature-level fusion method are needed before making
claims comparable with a commercial RGB-D camera.
