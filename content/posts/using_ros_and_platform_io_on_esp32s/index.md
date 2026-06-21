---
title: 'Using ROS2 and Platform IO on ESP32'
date: 2024-11-20T14:17:29+03:00
tags: ["embeded", "esp32", "ros2", "platformio", "microros", "docker"]
---

## Introduction

## Requirements

- [ROS2](https://www.ros.org)
- [Platform IO Core](https://platformio.org)
- [Docker (for running micro ROS on host)](https://www.docker.com)

## Setting up a platformio project

Firstly create a platform io project for your board with

```bash
pio project init -b esp32dev -d (PROJECT DIRECTORY)
```

if you have a different board, you can check available boards with `pio boards`.

## Adding microROS

Firstly specifly your ROS distribution by adding following lines to 
`platformio.ini` located at the root of your project:

```
board_microros_distro = humble
```

Change the value `humble` for other distributions. Then add following lines to
install micro-ROS as a dependency for platformio:

```
lib_deps =
    https://github.com/micro-ROS/micro_ros_platformio
```

## Writing firmware for ESP32

Copy following lines to `src/main.cpp` 

```c++
#include <Arduino.h>
#include <micro_ros_platformio.h>

#include <rcl/rcl.h>
#include <rclc/rclc.h>
#include <rclc/executor.h>

#include <std_msgs/msg/int32.h>

#if !defined(MICRO_ROS_TRANSPORT_ARDUINO_SERIAL)
#error This example is only avaliable for Arduino framework with serial transport.
#endif

rcl_publisher_t publisher;
std_msgs__msg__Int32 msg;

rclc_executor_t executor;
rclc_support_t support;
rcl_allocator_t allocator;
rcl_node_t node;
rcl_timer_t timer;

#define RCCHECK(fn) { rcl_ret_t temp_rc = fn; if((temp_rc != RCL_RET_OK)){error_loop();}}
#define RCSOFTCHECK(fn) { rcl_ret_t temp_rc = fn; if((temp_rc != RCL_RET_OK)){}}

// Error handle loop
void error_loop() {
  while(1) {
    delay(100);
  }
}

void timer_callback(rcl_timer_t * timer, int64_t last_call_time) {
  RCLC_UNUSED(last_call_time);
  if (timer != NULL) {
    RCSOFTCHECK(rcl_publish(&publisher, &msg, NULL));
    msg.data++;
  }
}

void setup() {
  // Configure serial transport
  Serial.begin(115200);
  set_microros_serial_transports(Serial);
  delay(2000);

  allocator = rcl_get_default_allocator();

  //create init_options
  RCCHECK(rclc_support_init(&support, 0, NULL, &allocator));

  // create node
  RCCHECK(rclc_node_init_default(&node, "micro_ros_platformio_node", "", &support));

  // create publisher
  RCCHECK(rclc_publisher_init_default(
    &publisher,
    &node,
    ROSIDL_GET_MSG_TYPE_SUPPORT(std_msgs, msg, Int32),
    "micro_ros_platformio_node_publisher"));

  // create timer,
  const unsigned int timer_timeout = 1000;
  RCCHECK(rclc_timer_init_default(
    &timer,
    &support,
    RCL_MS_TO_NS(timer_timeout),
    timer_callback));

  // create executor
  RCCHECK(rclc_executor_init(&executor, &support.context, 1, &allocator));
  RCCHECK(rclc_executor_add_timer(&executor, &timer));

  msg.data = 0;
}

void loop() {
  delay(100);
  RCSOFTCHECK(rclc_executor_spin_some(&executor, RCL_MS_TO_NS(100)));
}
```

(gracefully taken from [micro_ros_platformio](https://github.com/micro-ROS/micro_ros_platformio/blob/main/examples/micro-ros_publisher/src/Main.cpp) project)

## Flashing firmware

Thankfully it is simple to flash firmware with platformio. Run

```
pio run -t upload
```

If you cannot connect to device it might be related to permissions of the serial
port. Give yourself permissions for the serial port. If you can not find the
serial, try

```
pio device list
```

## Testing your configuration

Run following to check if our microros setup is running correctly:

```
docker run -it --rm -v /dev:/dev -v /dev/shm:/dev/shm --privileged --net=host microros/micro-ros-agent:$ROS_DISTRO serial --dev /dev/ttyUSB0 -v6
```

If everything has been setup up correctly you should be able to see your node 
sending messages. You can check if your node is connected by

```
ros2 node list
```

If you see an entry `/micro_ros_platformio_node` then everything is working.

