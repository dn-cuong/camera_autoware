# Lucid Vision Driver Integration for Autoware

The intended flow is:


## 1. Prerequisites

Before integrating the driver into Autoware, make sure you have:
- ROS 2 installed and sourced in your shell.
- Autoware workspace already set up.
- Lucid Vision ArenaSDK installed on the machine. (Already available)
- The `lucid_vision_driver` package available in a ROS 2 workspace. (Already available)


## 2. Install the driver

1. Open the Autoware repository list file at autoware.repos.

2. Add the `lucid_vision_driver` repository entry to that file so it is fetched together with the rest of the Autoware workspace. 
Use this driver: https://github.com/rancslab/lucid_vision_driver

    Example entry:

    ```yaml
    lucid_vision_driver:
      type: git
      url: https://github.com/rancslab/lucid_vision_driver
      version: main
    ```



3. Import the repositories into your Autoware workspace using the normal `vcs import` flow used by `autoware.mach_e`.


4. Make sure to re-build the workspace and source it


## 3. Camera parameter file

The Python launch file reads a parameter file from: 

```text
<share_of_lucid_vision_driver>/param/camera_0.param.yaml
```

With the default launch argument, that becomes:

```text
param/camera_0.param.yaml
```

The important parameters are:

- `camera_name`
- `frame_id`
- `pixel_format`
- `serial_no`
- `camera_info_url`
- `fps`
- `horizontal_binning`
- `vertical_binning`
- `use_default_device_settings`
- `exposure_auto`
- `exposure_target`
- `gain_auto`
- `gain_target`
- `gamma_target`
- `enable_compressing`
- `enable_rectifying`

The driver already provides the param file `param/camera_0.param.yaml` that works with our current lucid camera. I already tested the lucid vision camera driver running in ros2. Everything works fine. 

## 4. What each launch file does

### 4.1 camera.launch.xml & camera_node_container.launch.py
- Add two files to your Autoware sensor kit launch package.
- Currently, the XML file is finding the Python launch file (camera_node_container.launch.py) in `common_sensor_launch`. If you put the camera_node_container.launch.py file in a different package, you need to update the include path in the XML file.


### 4.2 Add the camera into the sensor kit launch file
- Add the camera launch file into your sensor kit launch file. For example, if you are using the `sensor_kit.launch.xml` file

