# ROS2 diff drive
Implementation of a differential drive package in ROS2 Humble with ros2_control.

Most of the code is taken from the [ros2_control with ROS2 [1h20 Crash Course]](https://www.youtube.com/watch?v=B9SbYjQSBY8&list=WL&index=10) by Robotics Back-End.

## Install dependencies
```bash
sudo apt install ros-humble-ros2-control ros-humble-ros2-controllers
```

## Package Structure

### my_robot_bringup/
```
my_robot_bringup/
├── CMakeLists.txt          # Build config
├── package.xml             # Package metadata
├── config/
│   ├── my_robot_controllers.yaml       # Controller params (diff_drive, joint_state)
│   ├── 0__simco_velocity.yaml          # Old config, replaced by SIM2015D_slave_config.yaml
│   └── SIM2015D_slave_config.yaml      # Simco slave configuration
└── launch/
    └── my_robot.launch.xml # Main launch: robot_state_publisher, ros2_control, controllers, rviz
```

### my_robot_description/
```
my_robot_description/
├── CMakeLists.txt          # Build config
├── package.xml             # Package metadata
├── launch/
│   ├── display.launch.py   # Visualization launch (robot_state_publisher, joint_state_publisher_gui, rviz)
│   └── display.launch.xml  # Alternative launch format
├── rviz/
│   └── urdf_config.rviz    # RViz visualization config
└── urdf/
    ├── my_robot.urdf.xacro              # Main robot URDF (includes all components)
    ├── mobile_base.xacro                # Base link, wheels, joints geometry
    ├── mobile_base.ros2_control.xacro   # ros2_control interface for mobile base
    ├── arduino_mobile_base.ros2_control.xacro  # Arduino hardware interface
    ├── simco_ros2_control.xacro         # Simco drive hardware interface
    └── common_properties.xacro          # Shared properties/constants
```
## Simulation
You can run a simulation with mock hardware. First launch the launch file:
```bash
source install/setup.bash
ros2 launch my_robot_bringup my_robot.launch.xml
```
Then run `twist_teleop_keyboard` to control the robot:
```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard --ros-args -r /cmd_vel:=/diff_drive_controller/cmd_vel -p stamped:=true
```
>In order to move the robot and not the environment set `Fixed Frame` to `base_footprint`.

## Connecting to physical hardware

### Testing with Arduino as a driver

#### Hw setup
Connect an L298N driver to Arduino.

#### Flash the fw
Clone https://github.com/joshnewans/ros_arduino_bridge, then set the right pin configuration inside `motor_drive.h`. Finally flash in the Arduino.

> Before moving on it is recommended to test the connection with #2.1 troubleshooting instruction.

#### Test with ros2_control
Install `joshnewans/diffdrive_arduino` which then requires `joshnewans/serial`:
```bash
cd src
git clone https://github.com/RedstoneGithub/diffdrive_arduino
git clone https://github.com/joshnewans/serial
cd ..
colcon build
source install/setup.bash
```
Since this is still WIP a proper switch hasn't been implemented yet. In order to enable the Arduino interface:
- select the right port inside `my_robot_description/urdf/arduino_mobile_base.ros2_control.xacro`
- at line 6 of `my_robot_description/urdf/my_robot.ros2_control.xacro` switch from `mobile_base.ros2_control.xacro` to `arduino_mobile_base.ros2_control.xacro`
- rebuild
Now you can run:
```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard --ros-args -r /cmd_vel:=/diff_drive_controller/cmd_vel -p stamped:=true
```
The motor should spin at this point.

### Connecting to Simco drive

#### Setup
Install `ethercat_driver_ros2` from [here](https://icube-robotics.github.io/ethercat_driver_ros2/quickstart/installation.html).
> You need to do this on bare metal Ubuntu as virtualized environment (Docker, WSL2) will prevent the package from accessing the physical resources it needs.

#### Code
#1 Pray!

#2 At line 6 of `my_robot_description/urdf/my_robot.ros2_control.xacro` switch from `mobile_base.ros2_control.xacro` to `simco_mobile_base.ros2_control.xacro`, then rebuild.
Now pray again, then start the `teleop_twist_keyboard` to control it.

## Trubleshooting

### #1 Build issues
In case of recurring building problems (expecially related to `catkin`) try clearing the cache:
```bash
rm -rf build/*
rm -rf install/*
```

### #2 Debug Arduino connection

#### #2.1 Test via serial
Make sure to have `pyserial` installed:
```bash
sudo apt install python3-serial
```
Depending on the one that works on your system run:
```bash
miniterm -e /dev/ttyACM0 57600
```
or
```bash
pyserial-miniterm -e /dev/ttyACM0 57600
```
followed by `o 255 255`:
```bash
usr@ubuntu:~$ pyserial-miniterm -e /dev/ttyACM0 57600
--- Miniterm on /dev/ttyACM0  57600,8,N,1 ---
--- Quit: Ctrl+] | Menu: Ctrl+T | Help: Ctrl+T followed by Ctrl+H ---
o 255 255
```
>Be careful to select the right port.

The motor should spin at this point.

#### #2.2 Test with ROS2
>You can skip this part if the above worked.

Clone https://github.com/joshnewans/serial_motor_demo iside the `src` folder of your workspace. Then build:
```bash
cd src
git clone https://github.com/joshnewans/serial_motor_demo
cd ..
colcon build
source install/setup.bash
```
Then run:
```bash
ros2 run serial_motor_demo driver --ros-args -p encoder_cpr:=3440 -p loop_rate:=30 -p serial_port:=/dev/ttyUSB0 -p baud_rate:=57600
```
>Be careful to select the right port.

Then launch the GUI from a separate terminal:
```bash
ros2 run serial_motor_demo gui
```
The motor should spin at this point.

### #3 Manually controlling motors

#### Setup
Set profile velocity:
```bash
sudo ethercat download -p0 --type int8 0x6060 0x00 0x03
```

Shotdown
```bash
sudo ethercat download -p0 --type uint16 0x6040 0x00 0x06
```

Switch ON
```bash
sudo ethercat download -p0 --type uint16 0x6040 0x00 0x07
```

Enable Operation
```bash
sudo ethercat download -p0 --type uint16 0x6040 0x00 0x0F
```

#### Stop
```bash
sudo ethercat download -p0 --type int32 0x60FF 0x00 100
sudo ethercat download -p0 --type int8 0x6060 0x00 0x00
```

#### Set velocity
Refer to the following table:
VEL_LOW  = 0x186A0          # 100,000   (right?)    0xFFFE7960  (left?)
VEL_MID  = 0x7A120          # 500.000   (left?)     0xFFF85EE0  (right?)
VEL_HIGH = 0X000C3500       # 800.000   0XFFF3CB00
VEL_HIGHER = 0x000F4240     # 1.000.000 0xFF0BDC0
VEL_SO_HIGHER = 0x0016E340  # 1.500.000 0xFFE91CC0 

```bash
sudo ethercat download -p0 --type uint32 0x60FF -- 0x00 0x186A0
```