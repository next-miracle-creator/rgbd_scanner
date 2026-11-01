# Recording example: Gemini 2 + Mid-360 + Vicon

Open one terminal per sensor and start them in order. After each one, use the check terminal to confirm its topics are flowing before you start the next. Then record, then open the bag in rqt. Every terminal starts with the same line:

```bash
cd ~/Desktop/Github-project/rgbd_scanner && source third_party/ros2_ws/install/setup.bash
```

## Terminal 1: Gemini 2

```bash
cd ~/Desktop/Github-project/rgbd_scanner && source third_party/ros2_ws/install/setup.bash
ros2 launch orbbec_camera gemini2.launch.py time_domain:=system enable_ir:=false enable_point_cloud:=false color_width:=1280 color_height:=720 color_fps:=15 color_format:=YUYV depth_width:=1280 depth_height:=800 depth_fps:=15 depth_format:=Y16 color_ae_max_exposure:=20000
```

- `time_domain:=system` stamps frames with host time, so they line up with the LiDAR and Vicon.
- Color 1280×720@15 YUYV and depth 1280×800@15 Y16 are pinned explicitly (the defaults let the SDK pick a mode). `enable_ir` and `enable_point_cloud` are off to save USB bandwidth and CPU. `color_ae_max_exposure:=20000` caps auto-exposure to limit motion blur.
- Expected: the log ends with `Loaded node '/camera/camera'`, followed by device and stream info and no errors.

Check terminal: `ros2 topic hz /camera/color/image_raw` and `ros2 topic hz /camera/depth/image_raw` both show **~15 Hz**.

## Terminal 2: Mid-360

```bash
cd ~/Desktop/Github-project/rgbd_scanner && source third_party/ros2_ws/install/setup.bash
ros2 launch livox_ros_driver2 msg_MID360_launch.py
```

- Publishes the Livox custom point cloud on `/livox/lidar` (10 Hz) and the IMU on `/livox/imu`, using `config/MID360_config.json`.
- Expected: no `bind failed` line. If you see it, the network card is not set to 192.168.123.200 (see Notes).

Check terminal (Livox publishes best-effort, so `--qos-profile sensor_data` is required):
`ros2 topic hz /livox/imu --qos-profile sensor_data` shows **~200 Hz**, and `ros2 topic hz /livox/lidar --qos-profile sensor_data` shows **10 Hz**.

## Terminal 3: Vicon

[Update the Vicon IP here before every session](third_party/ros2_ws/src/ros2-vicon-bridge/launch/all_segments.launch.py#L6). Look up the Vicon host's current IP on the Vicon PC and put it into `vicon_computer_ip`. No rebuild is needed, because the workspace is built with `--symlink-install`.

```bash
cd ~/Desktop/Github-project/rgbd_scanner && source third_party/ros2_ws/install/setup.bash
ros2 launch vicon_bridge all_segments.launch.py
```

- Expected: `Connecting to Vicon Datastream SDK at <the IP you set>:801`, then `...connected!`. A line of dots every 5 s means the IP is wrong or unreachable.

Check terminal: `ros2 topic hz /vicon/rgbd/rgbd/pose` shows a steady non-zero rate, and `ros2 topic echo --once /vicon/rgbd/rgbd/pose` prints one pose.

## Terminal 5: Record

```bash
cd ~/Desktop/Github-project/rgbd_scanner && source third_party/ros2_ws/install/setup.bash
ros2 bag record -s sqlite3 -o bags/rig_1031_2330 /camera/color/image_raw /camera/color/camera_info /camera/depth/image_raw /camera/depth/camera_info /livox/lidar /livox/imu /vicon/rgbd/rgbd/pose
```

- `-s sqlite3` stores the bag as `.db3`. `-o bags/rig_1031_2330` is the name of the bag **directory** (here: Oct 31, 23:30). Use a new name for each recording.
- To stop, press Ctrl+C **once** and wait for it to exit. Then check the bag: `ros2 bag info bags/rig_1031_2330` should list all 7 topics with non-zero message counts.

## View the bag in rqt

```bash
cd ~/Desktop/Github-project/rgbd_scanner && source third_party/ros2_ws/install/setup.bash
rqt
```

In rqt, go to Plugins → Logging → Bag, click the open (folder) icon, and select the `bags/rig_1031_2330` **directory**. A ROS 2 bag is a directory; in ROS 1 it was a single `.bag` file.

## Notes

- First build (once): `cd ~/Desktop/Github-project/rgbd_scanner/third_party/ros2_ws && source /opt/ros/humble/setup.bash && colcon build --symlink-install --cmake-args -DROS_EDITION=ROS2 -DDISTRO_ROS=humble -DCMAKE_BUILD_TYPE=Release`. Run `conda deactivate` first if conda is active, so the build uses the system Python 3.10.
- Livox-SDK2 (once, system-wide): `cd ~/Desktop/Github-project/rgbd_scanner/third_party/Livox-SDK2 && mkdir -p build && cd build && cmake .. && make -j && sudo make install`.
- Orbbec udev rules (once): `sudo bash ~/Desktop/Github-project/rgbd_scanner/third_party/ros2_ws/src/OrbbecSDK_ROS2/orbbec_camera/scripts/install_udev_rules.sh`, then replug the camera.
- Set the wired network card connected to the Mid-360 to **192.168.123.200** ([MID360_config.json:16](third_party/ros2_ws/src/livox_ros_driver2/config/MID360_config.json#L16)). The Mid-360's own IP (192.168.123.20) is at [MID360_config.json:15](third_party/ros2_ws/src/livox_ros_driver2/config/MID360_config.json#L15) and [:28](third_party/ros2_ws/src/livox_ros_driver2/config/MID360_config.json#L28).
- Vicon IP: it changes every session (the Vicon host gets it over WiFi). Edit [all_segments.launch.py:6](third_party/ros2_ws/src/ros2-vicon-bridge/launch/all_segments.launch.py#L6) before launching. `192.168.10.2` is only an example value.
