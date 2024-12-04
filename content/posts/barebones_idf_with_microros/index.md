---
title: 'Using Barebones ESP-ID with Microros'
date: 2024-12-04T13:15:06+03:00
tags: ["embeded", "esp32", "ros2", "microros", "docker"]
---

# Using Barebones ESP-ID with Microros

## Requirements

- [ROS2](https://www.ros.org)
- [Docker (for running micro ROS on host)](https://www.docker.com)

## Setting Up IDF

In order to setup IDF you first have to install its prerequisets. Use following
command for Ubuntu and Debian (taken from official [IDF documentation](https://docs.espressif.com/projects/esp-idf/en/v5.3.1/esp32/get-started/linux-macos-setup.html#get-started-prerequisites)):

```bash
sudo apt-get install git wget flex bison gperf python3 python3-pip python3-venv cmake ninja-build ccache libffi-dev libssl-dev dfu-util libusb-1.0-0
```

Next create a directory to install IDF (prefeberably accessible by you 
unpriviledged user) and clone the ESP-IDF repository:

```bash
export IDF_PATH="$HOME/.local/opt/esp-idf"
mkdir -p "$IDF_PATH"
git clone -b v5.3.1 --recursive https://github.com/espressif/esp-idf.git "$IDF_PATH"
```

Next cd into the directory and install the toolchain with

```bash
cd "$IDF_PATH"
./install.sh esp32 # Or other toolchains like esp32s2 etc
```

Next add "$IDF_PATH" to your shell config for persistancy (in this cas 
I am using bash, adopt it to your shell):

```bash
echo "export IDF_PATH=\"$IDF_PATH\"" >> "$HOME/.bashrc"
. "$HOME/.bashrc"
```
## Setting Up Microros ESP-IDF Component

Note: Most of the steps are taken from the [microros esp-idf componenet README](https://github.com/micro-ROS/micro_ros_espidf_component)
and modified to adopt this tutorial.

First clone the repository:

```bash
git clone --depth 1 --recurse-submodules https://github.com/micro-ROS/micro_ros_espidf_component.git -b "$ROS_DISTRO"
```

Then install the dependencies:

```bash
cd micro_ros_espidf_component
. $IDF_PATH/export.sh
pip3 install catkin_pkg lark-parser colcon-common-extensions
```

## Compiling Example Node

Now you can build the example node for your ESP32 and flash your firmware.

In project directory execute:

```bash
. $IDF_PATH/export.sh
. /opt/ros/$ROS_DISTRO/setup.bash
cd examples/int32_publisher
idf.py set-target esp32 # Set target board [esp32|esp32s2|esp32s3|esp32c3]
idf.py menuconfig # Set your micro-ROS configuration and WiFi credentials under micro-ROS Settings
idf.py build
```

Now that you have built your firmware you can start flashing it 
But before that check if you have access to serial ports for communicating with
ESP32. If not, you might need to add yourself to `dialout` group with:

```bash
sudo usermod -a -G dialout $(whoami)
```

or chown the serial device with:

```bash
sudo chown $(whoami) /dev/ttyUSB*
```

After ensuring that you have access to serial ports, flash the firmware:

```bash
idf.py flash
```

You can also monitor the serial port with ```idf.py monitor```.

## Testing your configuration

You can run microros agent with:

```bash
docker run -it --rm --net=host microros/micro-ros-agent:$ROS_DISTRO udp4 --port 8888 -v6
```

## Also Keep in Mind

If you have not configured an IP Address for ESP32, you need to have a DHCP
server. Look at `dnsmasq` for a lightweight dhcp server.
