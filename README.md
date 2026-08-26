# RGBD Scanner

A lightweight ROS 2 Humble integration repository for RGB-D + LiDAR data collection and LiDAR-based pose estimation.

This repository is designed for a scanner system built around:

- Orbbec Gemini 2 RGB-D camera
- Livox MID-360 LiDAR
- FAST-LIO2 as the LiDAR-inertial pose source
- Optional ORB-SLAM3
- Optional Vicon bridge for ground-truth evaluation

The repository is intentionally lightweight. It is not intended to be a full robot stack. Its goal is to provide a reproducible data collection workflow: launch sensors, record useful topics, run FAST-LIO2 online or offline, convert pose frames, and post-process odometry for RGB-D association.

Third-party drivers and algorithms are managed as git submodules under `third_party/`. This repository mainly provides launch wrappers, configuration files, automation scripts, and data-checking tools.

## 1. Supported Environment

Tested target environment:

```
Ubuntu 22.04
ROS 2 Humble
Orbbec Gemini 2
Livox MID-360
```

This repository currently focuses on ROS 2 Humble only.

## 2. Repository Layout

```
rgbd_scanner/
├── README.md
├── scanner_bringup/
├── config/
├── scripts/
├── tools/
└── third_party/
```

Main responsibilities:

```
scanner_bringup/      ROS 2 launch package
config/               Sensor, frame, algorithm, and mode configs
scripts/              Setup, build, run, patch, postprocess, and cleanup scripts
tools/                Data checking and post-processing tools
third_party/          Git submodules for drivers and algorithms
```

### `scanner_bringup/`

This is the main ROS 2 package maintained by this repository.

```
scanner_bringup/
├── package.xml
├── CMakeLists.txt
└── launch/
    ├── full.launch.py
    ├── lio_capture.launch.py
    ├── lio_record.launch.py
    ├── lio_mapping.launch.py
    └── lio_replay.launch.py
```

The launch files wrap third-party launch files and expose scanner-level run modes.

### `config/`

Configuration is separated by responsibility:

```
config/
├── sensors/
├── frames/
├── algorithms/
└── modes/
```

Each layer has a different purpose:

```
sensors/      Sensor-level configs that rarely change between modes
frames/       Coordinate frame definitions and extrinsics
algorithms/   Algorithm-specific configs
modes/        Run-mode configs: what to launch, record, and post-process
```

### `scripts/`

```
scripts/
├── setup_humble.sh
├── build.sh
├── run.sh
├── patch_configs.sh
├── postprocess_lio.sh
├── activate.bash
└── clean.sh
```

The recommended user entry point is through `scripts/run.sh`, not directly through third-party launch files.

### `tools/`

```
tools/
├── check_rgbd_stream.py
├── check_bag_topics.py
├── convert_pose_frame.py
├── interpolate_fastlio_odom.py
└── export_pose_tum.py
```

These tools are intended for data validation and post-processing.

### `third_party/`

Third-party repositories are included as git submodules.

Expected third-party components include:

```
third_party/OrbbecSDK_ROS2
third_party/livox_ros_driver2
third_party/fast_lio2
third_party/orb_slam3_ros2
third_party/ros2-vicon-bridge
```

Some components may be optional depending on the selected build or run mode.

## 3. Clone

Create a ROS 2 workspace and clone this repository into `src`:

```
mkdir -p ~/scanner_ws/src
cd ~/scanner_ws/src

git clone --recursive <repo_url> rgbd_scanner
```

If the repository was cloned without `--recursive`, initialize submodules manually:

```
cd ~/scanner_ws/src/rgbd_scanner
git submodule update --init --recursive
```

## 4. Setup

Run the setup script once:

```
cd ~/scanner_ws
bash src/rgbd_scanner/scripts/setup_humble.sh
```

The setup script is expected to:

```
initialize git submodules
install common ROS 2 dependencies
install required system packages
install device permission rules when needed
prepare the workspace for building
```

For a new machine, this step should be run before building.

## 5. Build

### Build LiDAR pose scanner mode

This is the recommended default mode.

```
cd ~/scanner_ws
bash src/rgbd_scanner/scripts/build.sh lio-scanner
```

This mode is intended to build:

```
Orbbec Gemini 2 driver
Livox MID-360 driver
FAST-LIO2
scanner_bringup
```

### Build full mode

```
cd ~/scanner_ws
bash src/rgbd_scanner/scripts/build.sh full
```

This mode is intended to build all supported components:

```
Orbbec Gemini 2 driver
Livox MID-360 driver
FAST-LIO2
ORB-SLAM3
Vicon bridge
scanner_bringup
```

## 6. Run Modes

The repository is organized around practical data collection modes.

### `lio_capture`

Online RGB-D + LiDAR pose capture.

```
cd ~/scanner_ws
bash src/rgbd_scanner/scripts/run.sh lio_capture
```

This mode starts:

```
Gemini 2 camera driver
Livox MID-360 driver
FAST-LIO2
rosbag recording
```

Use this mode when the computer can run FAST-LIO2 during data collection.

Typical recorded data includes:

```
RGB image
Depth image
Camera info
LiDAR point cloud
LiDAR IMU
FAST-LIO2 odometry
TF
TF static
```

### `lio_record`

Raw sensor recording only.

```
cd ~/scanner_ws
bash src/rgbd_scanner/scripts/run.sh lio_record
```

This mode starts:

```
Gemini 2 camera driver
Livox MID-360 driver
rosbag recording
```

FAST-LIO2 is not launched in this mode.

Use this mode when:

```
online SLAM is not required
acquisition stability is more important
the machine is not powerful enough for online FAST-LIO2
pose will be recovered later through offline replay
```

### `lio_replay`

Offline FAST-LIO2 pose recovery from a recorded bag.

```
cd ~/scanner_ws
bash src/rgbd_scanner/scripts/run.sh lio_replay /path/to/rosbag
```

Use this after `lio_record` to recover LiDAR-based odometry.

### `lio_mapping`

LiDAR-only mapping.

```
cd ~/scanner_ws
bash src/rgbd_scanner/scripts/run.sh lio_mapping
```

This mode starts:

```
Livox MID-360 driver
FAST-LIO2 mapping
```

Gemini 2 is not required in this mode.

Use this mode when the goal is only to build a LiDAR point cloud map.

### `full`

Full system mode.

```
cd ~/scanner_ws
bash src/rgbd_scanner/scripts/run.sh full
```

This mode is intended for development, evaluation, and algorithm comparison.

It may include:

```
Gemini 2 camera driver
Livox MID-360 driver
FAST-LIO2
ORB-SLAM3
Vicon bridge
```

## 7. Configuration Structure

The configuration layer is split by responsibility.

```
config/
├── sensors/
├── frames/
├── algorithms/
└── modes/
```

### 7.1 Sensor configs

```
config/sensors/
├── orbbec_gemini2.yaml
├── livox_mid360.json
└── vicon.yaml
```

Sensor configs describe hardware-level parameters that are usually shared across modes.

Examples:

```
Gemini 2 color resolution
Gemini 2 depth resolution
RGB-D frame rate
depth-to-color alignment
Livox MID-360 network settings
Vicon host settings
sensor topic names
```

These configs are considered the base sensor configuration. In most cases, they should not be duplicated for every run mode.

### 7.2 Frame configs

```
config/frames/
├── scanner_frames.yaml
└── extrinsics.yaml
```

Frame configs define the coordinate system convention of the collected dataset.

They should describe:

```
what each frame means
which frame is treated as scanner body
LiDAR frame
camera body frame
camera optical frame
static transforms between sensors
pose conversion targets
```

Common frames may include:

```
map
odom
base_link
livox_frame
camera_link
camera_color_optical_frame
camera_depth_optical_frame
```

Recommended meaning:

```
map
  Global mapping frame produced by FAST-LIO2.

odom
  Local odometry frame. Depending on FAST-LIO2 configuration, this may be the same as map.

base_link
  Scanner body frame.

livox_frame
  Livox MID-360 LiDAR frame.

camera_link
  Orbbec Gemini 2 camera body frame.

camera_color_optical_frame
  RGB camera optical frame.

camera_depth_optical_frame
  Depth camera optical frame.
```

FAST-LIO2 odometry is usually not directly the RGB camera pose. If RGB-D frames need camera poses, the LiDAR/body pose must be transformed to the target camera frame using the calibrated extrinsics.

### 7.3 Algorithm configs

```
config/algorithms/
├── fast_lio2/
│   ├── scanner.yaml
│   └── mapping.yaml
└── orb_slam3/
    ├── rgbd.yaml
    └── vocabulary_path.yaml
```

FAST-LIO2 may use different configs for scanner pose estimation and LiDAR-only mapping.

Example:

```
config/algorithms/fast_lio2/scanner.yaml   # used as RGB-D pose source
config/algorithms/fast_lio2/mapping.yaml   # used for LiDAR-only mapping
```

ORB-SLAM3-related configs are optional and mainly used in full mode or algorithm comparison experiments.

### 7.4 Mode configs

```
config/modes/
├── full.yaml
├── lio_capture.yaml
├── lio_record.yaml
├── lio_mapping.yaml
└── lio_replay.yaml
```

Mode configs describe how components are combined.

They should answer:

```
which sensors are enabled
which algorithms are enabled
which config files are applied
whether rosbag recording is enabled
which topics are recorded
whether post-processing is required
```

For example:

```
lio_capture
  uses Gemini 2
  uses Livox MID-360
  uses FAST-LIO2
  records RGB-D, LiDAR, IMU, odometry, TF

lio_record
  uses Gemini 2
  uses Livox MID-360
  does not use FAST-LIO2 online
  records raw sensor topics for later replay

lio_mapping
  uses Livox MID-360
  uses FAST-LIO2 mapping config
  does not require Gemini 2
```

## 8. Data Collection Notes

This repository is a data collection stack, so launching nodes is only the first step. Before a dataset is considered valid, the following issues must be checked.

### 8.1 RGB-D stream alignment

For RGB-D collection, the color and depth streams must be checked before recording.

Users should confirm:

```
color image resolution
depth image resolution
color image frequency
depth image frequency
color/depth timestamp difference
whether depth is aligned to color
camera_info topic availability
```

Use:

```
python3 src/rgbd_scanner/tools/check_rgbd_stream.py
```

A typical check should report:

```
color topic
depth topic
camera info topic
color resolution
depth resolution
color FPS
depth FPS
mean color-depth timestamp offset
maximum color-depth timestamp offset
```

A recording should only be considered valid if color and depth have the expected resolution, frequency, and acceptable timestamp alignment.

### 8.2 Recorded topics and coordinate frames

Recorded topics must be interpreted together with their coordinate frames.

The frame definitions are configured in:

```
config/frames/scanner_frames.yaml
config/frames/extrinsics.yaml
```

The recorded pose should always be documented with:

```
source frame
child frame
target frame
timestamp convention
whether the pose is raw or transformed
```

FAST-LIO2 odometry is usually expressed in its own LiDAR/body convention. It should not be blindly treated as the RGB camera pose.

If RGB-D frames require camera poses, the processing pipeline should include:

```
FAST-LIO2 odometry
LiDAR/body frame definition
scanner extrinsics
target camera optical frame
pose conversion
timestamp association
```

Relevant tools:

```
python3 src/rgbd_scanner/tools/check_bag_topics.py
python3 src/rgbd_scanner/tools/convert_pose_frame.py
python3 src/rgbd_scanner/tools/export_pose_tum.py
```

### 8.3 FAST-LIO2 odometry interpolation

FAST-LIO2 odometry is initially expected to be around 10 Hz.

For RGB-D timestamp association, this repository provides a post-processing step to interpolate FAST-LIO2 odometry to 100 Hz.

The current assumption is:

```
the scanner moves slowly during data collection
motion between two adjacent FAST-LIO2 odometry messages is approximated as constant velocity
translation is linearly interpolated
rotation should be interpolated with quaternion SLERP
```

Important note:

```
The 100 Hz pose is an interpolated pose sequence.
It should not be treated as a higher-accuracy pose estimate than the original FAST-LIO2 odometry.
```

Example command:

```
python3 src/rgbd_scanner/tools/interpolate_fastlio_odom.py \
  --input /path/to/fastlio_odom.txt \
  --output /path/to/fastlio_odom_100hz.txt \
  --rate 100
```

Or for rosbag-based processing:

```
python3 src/rgbd_scanner/tools/interpolate_fastlio_odom.py \
  --bag /path/to/rosbag \
  --odom-topic /Odometry \
  --output /path/to/fastlio_pose_100hz.txt \
  --rate 100
```

## 9. Typical Workflows

### 9.1 Online pose capture

```
lio_capture
  -> start Gemini 2
  -> start Livox MID-360
  -> start FAST-LIO2
  -> record RGB-D, LiDAR, IMU, pose, TF
  -> optionally post-process FAST-LIO2 odometry to 100 Hz
```

Command:

```
cd ~/scanner_ws
bash src/rgbd_scanner/scripts/run.sh lio_capture
```

### 9.2 Raw recording and offline pose recovery

```
lio_record
  -> record RGB-D, LiDAR, IMU, TF only

lio_replay
  -> replay rosbag
  -> run FAST-LIO2 offline
  -> recover pose

postprocess_lio.sh
  -> check bag topics
  -> convert pose frame
  -> interpolate odometry
  -> export pose
```

Commands:

```
cd ~/scanner_ws
bash src/rgbd_scanner/scripts/run.sh lio_record
bash src/rgbd_scanner/scripts/run.sh lio_replay /path/to/rosbag
bash src/rgbd_scanner/scripts/postprocess_lio.sh /path/to/rosbag
```

### 9.3 LiDAR-only mapping

```
lio_mapping
  -> start Livox MID-360
  -> start FAST-LIO2 mapping
  -> build LiDAR point cloud map
```

Command:

```
cd ~/scanner_ws
bash src/rgbd_scanner/scripts/run.sh lio_mapping
```

### 9.4 Full evaluation workflow

```
full
  -> start all required hardware drivers
  -> start selected algorithms
  -> optionally start Vicon bridge
  -> collect data for comparison or evaluation
```

Command:

```
cd ~/scanner_ws
bash src/rgbd_scanner/scripts/run.sh full
```

## 10. Third-Party Repositories

Third-party repositories are managed as git submodules.

To initialize or update submodules:

```
cd ~/scanner_ws/src/rgbd_scanner
git submodule update --init --recursive
```

To check submodule versions:

```
git submodule status
```

Third-party repositories should be pinned to tested commits. Changing submodule versions may require rebuilding the workspace and re-checking configuration compatibility.

Do not edit third-party config files directly unless necessary. Prefer editing files under `config/`, then let `scripts/patch_configs.sh` apply the selected configs to the corresponding third-party packages.

## 11. Clean

To clean workspace build artifacts:

```
cd ~/scanner_ws
bash src/rgbd_scanner/scripts/clean.sh
```

This usually removes:

```
build/
install/
log/
```

## 12. Troubleshooting

### Submodules are empty

Run:

```
cd ~/scanner_ws/src/rgbd_scanner
git submodule update --init --recursive
```

### ROS 2 environment is not found

Make sure ROS 2 Humble is installed and available:

```
source /opt/ros/humble/setup.bash
```

### Workspace package is not found

Rebuild and source the workspace:

```
cd ~/scanner_ws
bash src/rgbd_scanner/scripts/build.sh lio-scanner
source install/setup.bash
```

### Camera permission issue

Re-run setup:

```
cd ~/scanner_ws
bash src/rgbd_scanner/scripts/setup_humble.sh
```

Then unplug and replug the camera.

### RGB and depth are not aligned

Check:

```
color resolution
depth resolution
color FPS
depth FPS
timestamp difference
depth-to-color alignment option
camera_info topic
```

Run:

```
python3 src/rgbd_scanner/tools/check_rgbd_stream.py
```

### Livox MID-360 is not detected

Check:

```
LiDAR power
Ethernet connection
host IP address
Livox config file
firewall settings
```

The Livox configuration is expected at:

```
config/sensors/livox_mid360.json
```

### FAST-LIO2 pose looks wrong

Check:

```
LiDAR IMU topic
LiDAR point cloud topic
FAST-LIO2 config file
LiDAR-to-body extrinsics
body-to-camera extrinsics
time synchronization
```

Relevant configs:

```
config/algorithms/fast_lio2/scanner.yaml
config/frames/extrinsics.yaml
```

### Pose frame does not match RGB-D frame

FAST-LIO2 pose is usually not directly the RGB camera pose.

Check:

```
FAST-LIO2 odometry frame
LiDAR frame
scanner body frame
camera optical frame
extrinsic calibration
```

Use:

```
python3 src/rgbd_scanner/tools/convert_pose_frame.py
```

### Interpolated pose has artifacts

The 100 Hz pose is interpolated from lower-rate FAST-LIO2 odometry.

Check:

```
original odometry frequency
large timestamp gaps
fast scanner motion
sudden rotation
missing odometry messages
```

The interpolation assumes slow and approximately constant-velocity motion between adjacent odometry messages.

## 13. Notes

This repository intentionally avoids unnecessary components at the current stage.

It does not provide a custom scanner URDF package or RViz configuration by default.

The main goal is to provide a simple and reproducible data collection stack for:

```
RGB-D capture
LiDAR-based pose estimation
raw sensor recording
offline pose recovery
LiDAR mapping
pose frame conversion
pose interpolation for RGB-D association
```