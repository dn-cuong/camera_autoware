# V4L2 Camera Launch Guide (v4l2_camera)

This document describes how to run, configure, and modify the two camera launch files in this repo:

| File | Role |
|------|------|
| `camera.launch.xml` | Declares parameters, applies namespace `camera`, includes the Python launch file |
| `camera_node_container.launch.py` | Creates a composable container and loads the `v4l2_camera` driver |

The driver uses the ROS 2 package: [v4l2_camera](https://docs.ros.org/en/humble/p/v4l2_camera/) (Video4Linux2).

---

## Table of Contents

1. [System Requirements](#1-system-requirements)
2. [Structure and Launch Flow](#2-structure-and-launch-flow)
3. [Pre-launch Preparation](#3-pre-launch-preparation)
4. [How to Run](#4-how-to-run)
5. [Configuration Parameters](#5-configuration-parameters)
6. [Topics and Namespaces](#6-topics-and-namespaces)
7. [Calibration File](#7-calibration-file)
8. [Running in Autoware](#8-running-in-autoware)
9. [Modifying the Code](#9-modifying-the-code)
10. [Topic Remapping](#10-topic-remapping)
11. [Post-launch Verification](#11-post-launch-verification)
12. [Common Troubleshooting](#12-common-troubleshooting)
13. [Quick Checklist](#13-quick-checklist)

---

## 1. System Requirements

### Required ROS 2 packages

```bash
sudo apt-get install ros-${ROS_DISTRO}-v4l2-camera
sudo apt-get install ros-${ROS_DISTRO}-rclcpp-components
```

Optional (image compression via image_transport):

```bash
sudo apt-get install ros-${ROS_DISTRO}-image-transport-plugins
```

### Camera inspection tools (Linux)

```bash
sudo apt-get install v4l-utils
```

### Install launch files into a ROS package

Both files in this repo must live inside a launch package (e.g. `mach_e_sensor_kit_launch`):

```
mach_e_sensor_kit_launch/
└── launch/
    ├── camera.launch.xml
    └── camera_node_container.launch.py
```

Then build the workspace:

```bash
cd ~/autoware_ws   # or your workspace
colcon build --packages-select mach_e_sensor_kit_launch
source install/setup.bash
```

---

## 2. Structure and Launch Flow

```
camera.launch.xml
  │
  ├─ Declare launch args (video_device, resolution, ...)
  ├─ push-ros-namespace: camera
  │
  └─ include → camera_node_container.launch.py
                    │
                    ├─ Select component_container / component_container_mt
                    ├─ Create ComposableNodeContainer (namespace: traffic_light)
                    └─ Load v4l2_camera::V4L2Camera
                           │
                           └─ Publish: image_raw, camera_info
```

**Standalone** `camera.launch.xml`:

```
/camera/traffic_light/image_raw
/camera/traffic_light/camera_info
```

**Inside Autoware** (parent pushes namespace `sensing`):

```
/sensing/camera/traffic_light/image_raw
/sensing/camera/traffic_light/camera_info
```

---

## 3. Pre-launch Preparation

### 3.1. Identify the camera device

```bash
ls -l /dev/video*
v4l2-ctl --list-devices
```

Note the correct device (e.g. `/dev/video2`). Some `/dev/video*` entries are metadata-only and cannot capture images.

### 3.2. Check supported formats and resolution

```bash
v4l2-ctl -d /dev/video0 --list-formats-ext
```

Record:

- FOURCC: `YUYV`, `UYVY`, or `GREY`
- Resolution: e.g. `1920x1080`

### 3.3. Prepare calibration file (if available)

- Copy the YAML file to the machine running the launch
- `width` / `height` in the file must match `image_width` / `image_height`
- Use URI format: `file:///absolute/path/to/camera.yaml`

### 3.4. Match frame_id with URDF

`camera_frame_id` must match the camera link in the sensor kit URDF (default: `traffic_light/camera_link`).

---

## 4. How to Run

### 4.1. Run with default values

```bash
source /opt/ros/humble/setup.bash
source ~/autoware_ws/install/setup.bash

ros2 launch mach_e_sensor_kit_launch camera.launch.xml
```

### 4.2. Override parameters at launch

```bash
ros2 launch mach_e_sensor_kit_launch camera.launch.xml \
  video_device:=/dev/video2 \
  pixel_format:=UYVY \
  image_width:=1920 \
  image_height:=1080 \
  output_encoding:=rgb8 \
  camera_frame_id:=traffic_light/camera_link \
  camera_info_url:=file:///home/user/calib/camera.yaml
```

### 4.3. Disable camera driver (no process spawned)

```bash
ros2 launch mach_e_sensor_kit_launch camera.launch.xml launch_driver:=false
```

### 4.4. Run the Python file directly (debug)

```bash
ros2 launch mach_e_sensor_kit_launch camera_node_container.launch.py \
  video_device:=/dev/video0
```

Note: running the Python file directly **does not** apply `push-ros-namespace camera` from the XML — topics will differ from launching via `camera.launch.xml`.

### 4.5. View images

```bash
sudo apt-get install ros-${ROS_DISTRO}-rqt-image-view
ros2 run rqt_image_view rqt_image_view
```

Select topic: `/camera/traffic_light/image_raw` (or `/sensing/camera/traffic_light/image_raw` in Autoware).

---

## 5. Configuration Parameters

| Launch arg | `v4l2_camera` node param | Default | You need to change? |
|------------|--------------------------|---------|---------------------|
| `video_device` | `video_device` | `/dev/video0` | **Yes** — per machine |
| `pixel_format` | `pixel_format` | `YUYV` | **Yes** — per camera |
| `output_encoding` | `output_encoding` | `rgb8` | Rarely |
| `image_width` | `image_size[0]` | `640` | **Yes** |
| `image_height` | `image_size[1]` | `480` | **Yes** |
| `camera_frame_id` | `camera_frame_id` | `traffic_light/camera_link` | **Yes** — match URDF |
| `camera_info_url` | `camera_info_url` | `""` | If you have a calib file |
| `camera_container_name` | — | `traffic_light_camera_container` | Rarely |
| `use_intra_process` | `use_intra_process_comms` | `true` | Keep default |
| `use_multithread` | — (selects executable) | `false` | Enable if multiple components |
| `launch_driver` | — | `true` | Disable when camera not needed |

Camera controls (brightness, contrast, white balance, etc.) are auto-enumerated by `v4l2_camera` — no need to declare them in the launch files.

---

## 6. Topics and Namespaces

### Where namespaces are applied

| File | Line | Namespace |
|------|------|-----------|
| `camera.launch.xml` | `push-ros-namespace namespace="camera"` | `camera` |
| `camera_node_container.launch.py` | `namespace="traffic_light"` | `traffic_light` |

### Published topics

| Topic | Message type |
|-------|--------------|
| `image_raw` | `sensor_msgs/Image` |
| `camera_info` | `sensor_msgs/CameraInfo` |
| `image_raw/compressed` | Only available with `image-transport-plugins` installed |

### Node shown in `ros2 node list`

```
/camera/traffic_light_camera_container
```

(`v4l2_camera` runs inside the container and does not appear as a separate node.)

---

## 7. Calibration File

### Using a calibration file from another machine

**Yes**, if:

- Same physical camera (or same model, with lower accuracy)
- Same resolution and `pixel_format`
- Focus/lens has not been adjusted since calibration

### URI format

```bash
# Absolute file path
camera_info_url:=file:///home/user/calib/camera.yaml

# File inside a ROS package
camera_info_url:=package://my_pkg/config/camera_info.yaml
```

### Verify

```bash
ros2 topic echo /camera/traffic_light/camera_info --once
```

Compare `width`, `height`, `k`, and `d` with the YAML file.

---

## 8. Running in Autoware

### Do not run `camera.launch.xml` in isolation for full perception

The camera is launched from the sensing launch tree:

```
autoware.launch.xml
  └─ tier4_sensing_component
       └─ sensing.launch.xml  (namespace: sensing)
            └─ camera.launch.xml  (namespace: camera)
                 └─ camera_node_container.launch.py  (namespace: traffic_light)
```

### Launch full stack

```bash
ros2 launch autoware_launch autoware.launch.xml \
  map_path:=/path/to/map \
  vehicle_model:=mach_e_vehicle \
  sensor_model:=mach_e_sensor_kit \
  launch_sensing:=true \
  launch_perception:=true
```

Override camera parameters depending on how the sensor kit exposes args (in `sensing.launch.xml` or a config file).

### What Autoware perception requires

| Input | Topic |
|-------|-------|
| Image | `/sensing/camera/traffic_light/image_raw` |
| CameraInfo | `/sensing/camera/traffic_light/camera_info` |
| Map | `/map/vector_map` |
| TF | `/tf` |

Running only the camera driver gives you images but **not** traffic light recognition.

---

## 9. Modifying the Code

### 9.1. Change the include path

**File:** `camera.launch.xml` — line 18

```xml
<!-- Current -->
<include file="$(find-pkg-share mach_e_sensor_kit_launch)/launch/camera_node_container.launch.py">

<!-- Rename package -->
<include file="$(find-pkg-share your_package_name)/launch/camera_node_container.launch.py">
```

### 9.2. Change topic namespace

**Rename camera** (e.g. `front_camera` instead of `traffic_light`):

**File:** `camera_node_container.launch.py` — line 39

```python
namespace="front_camera",
```

New topic: `/camera/front_camera/image_raw`

**Rename camera group namespace** (e.g. `cameras` instead of `camera`):

**File:** `camera.launch.xml` — line 17

```xml
<push-ros-namespace namespace="cameras"/>
```

### 9.3. Change default values

Update **both files** for consistency:

- `camera.launch.xml` — `<arg ... default="..."/>`
- `camera_node_container.launch.py` — `add_launch_arg(...)`

Example: change default resolution to 1920x1080 in both places.

### 9.4. Add a new launch argument

**Step 1** — `camera.launch.xml`:

```xml
<arg name="output_encoding" default="rgb8" .../>   <!-- already exists -->
<arg name="new_arg" default="value" description="..."/>
```

Pass it to the include:

```xml
<arg name="new_arg" value="$(var new_arg)"/>
```

**Step 2** — `camera_node_container.launch.py`:

```python
# In generate_launch_description()
add_launch_arg("new_arg", "value", "Description")

# In launch_setup()
new_arg = LaunchConfiguration("new_arg").perform(context)
camera_params["ros_param_name"] = new_arg
```

### 9.5. Rename the container

**Files:** `camera.launch.xml` + `camera_node_container.launch.py`

Change the default of `camera_container_name` in both files.

### 9.6. Add another composable node to the container

**File:** `camera_node_container.launch.py` — inside `composable_node_descriptions`:

```python
composable_node_descriptions=[
    ComposableNode(
        package="v4l2_camera",
        plugin="v4l2_camera::V4L2Camera",
        name="v4l2_camera",
        parameters=[camera_params],
        extra_arguments=[{"use_intra_process_comms": use_intra_process}],
    ),
    ComposableNode(
        package="other_package",
        plugin="namespace::PluginClass",
        name="node_name",
        parameters=[...],
        extra_arguments=[{"use_intra_process_comms": use_intra_process}],
    ),
],
```

When running multiple components, consider `use_multithread:=true`.

### 9.7. After modifying code

```bash
colcon build --packages-select mach_e_sensor_kit_launch
source install/setup.bash
```

---

## 10. Topic Remapping

### Where to add remappings

**File:** `camera_node_container.launch.py` — inside `ComposableNode`, alongside `parameters`:

```python
ComposableNode(
    package="v4l2_camera",
    plugin="v4l2_camera::V4L2Camera",
    name="v4l2_camera",
    parameters=[camera_params],
    remappings=[
        ("image_raw", "image_raw"),       # (node's original name, output topic name)
        ("camera_info", "camera_info"),
    ],
    extra_arguments=[{"use_intra_process_comms": use_intra_process}],
),
```

### Example: rename a topic

```python
remappings=[
    ("image_raw", "image_rect_color"),
    ("camera_info", "camera_info"),
],
```

### When remapping is not needed

- Full Autoware stack with correct namespaces → topics resolve to `/sensing/camera/traffic_light/image_raw`
- Identity remap `image_raw → image_raw` has no effect

### Remapping from XML

`camera.launch.xml` **cannot** remap composable nodes directly. To configure from XML: add a launch arg → pass it to Python → use it in `remappings`.

---

## 11. Post-launch Verification

```bash
# Nodes
ros2 node list

# Topics
ros2 topic list | grep traffic_light

# Image rate
ros2 topic hz /camera/traffic_light/image_raw

# CameraInfo
ros2 topic echo /camera/traffic_light/camera_info --once

# frame_id in image header
ros2 topic echo /camera/traffic_light/image_raw --once

# TF (in Autoware)
ros2 run tf2_ros tf2_echo base_link traffic_light/camera_link
```

---

## 12. Common Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Cannot open device /dev/videoX` | Wrong device or camera not connected | `v4l2-ctl --list-devices`, try another device |
| Node runs but no images | `/dev/videoX` is a metadata-only device | Try `video0`, `video2`, etc. |
| Set format/resolution fails | Camera does not support that combination | `v4l2-ctl --list-formats-ext`, change `pixel_format` / resolution |
| `camera_info_url` error | File missing or invalid URI | Check path, use correct `file:///` format |
| Perception does not receive images | Camera launched standalone, missing `/sensing` prefix | Run full Autoware or remap topics |
| TF error / wrong ROI | `camera_frame_id` does not match URDF | Update `camera_frame_id` to match sensor kit URDF |
| `find-pkg-share mach_e_sensor_kit_launch` fails | Package not built/installed | `colcon build` + `source install/setup.bash` |

---

## 13. Quick Checklist

```text
[ ] Install ros-${ROS_DISTRO}-v4l2-camera
[ ] Copy both launch files into package + colcon build
[ ] Identify /dev/videoX
[ ] Check format + resolution (v4l2-ctl)
[ ] (Optional) Copy calib file + set camera_info_url
[ ] camera_frame_id matches URDF
[ ] Launch and verify topic hz
[ ] (Autoware) Run full stack, verify /sensing/camera/traffic_light/image_raw
```

---

## References

- [v4l2_camera — ROS 2 Humble docs](https://docs.ros.org/en/humble/p/v4l2_camera/)
- [Autoware — Integrating cameras](https://autowarefoundation.github.io/autoware-documentation/main/tutorials/integrating-autoware/integrating-sensors/integrating-cameras/)
- [Autoware — Topic namespaces](https://autowarefoundation.github.io/autoware-documentation/main/contributing/coding-guidelines/ros-nodes/topic-namespaces/)
