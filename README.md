# micro-ROS on STM32F446 with FreeRTOS — Setup Guide

> **Target MCU:** STM32F446ZE (Nucleo-144)  
> **ROS 2 Distro:** Humble  
> **RTOS:** FreeRTOS (CMSIS-RTOS V2)  
> **Transport:** UART3 + DMA  
> **Author:** Mubasher | Updated: March 2026

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [One-Time Setup: Build the micro-ROS Library](#one-time-setup-build-the-micro-ros-library)
3. [Project Structure](#project-structure)
4. [Creating a New micro-ROS Project](#creating-a-new-micro-ros-project)
5. [STM32CubeIDE Configuration](#stm32cubeide-configuration)
6. [Code Template](#code-template)
7. [Building & Flashing](#building--flashing)
8. [Testing with micro-ROS Agent](#testing-with-micro-ros-agent)
9. [Troubleshooting](#troubleshooting)

---

## Prerequisites

Ensure the following are installed on your Ubuntu system:

```bash
# Docker (for library generation)
sudo apt install docker.io
sudo usermod -aG docker $USER   # allow Docker without sudo (re-login after)

# ROS 2 Humble + micro-ROS agent
sudo apt install ros-humble-desktop
pip install micro-ros-agent       # or build from source

# STM32CubeIDE 1.19+
# Download from: https://www.st.com/en/development-tools/stm32cubeide.html

# ARM toolchain (bundled with STM32CubeIDE)
arm-none-eabi-gcc --version
```

---

## One-Time Setup: Build the micro-ROS Library

This step is done **only once**. The generated library is reused across all future STM32F446 + Humble projects.

### Step 1: Create a base project and clone utils

In any initial micro-ROS project (e.g., `pubsub_micro_ros`), clone the STM32CubeMX utils:

```bash
cd /home/mubasher/rtos_codes/pubsub_micro_ros
git clone -b humble https://github.com/micro-ROS/micro_ros_stm32cubemx_utils.git
```

### Step 2: Build the static library via Docker

```bash
cd /home/mubasher/rtos_codes/pubsub_micro_ros

docker run --rm \
  -v /home/mubasher/rtos_codes/pubsub_micro_ros:/project \
  --env MICROROS_LIBRARY_FOLDER=micro_ros_stm32cubemx_utils/microros_static_library_ide \
  microros/micro_ros_static_library_builder:humble
```

> ⏱ First run downloads ~800MB Docker image and takes 10–15 min to compile.  
> Subsequent runs are fast — if `libmicroros.a` already exists, it skips rebuilding.

### Step 3: Save the library to a shared location

```bash
mkdir -p /home/mubasher/microros_shared/humble_f446
cp -r /home/mubasher/rtos_codes/pubsub_micro_ros/micro_ros_stm32cubemx_utils/microros_static_library_ide/libmicroros \
      /home/mubasher/microros_shared/humble_f446/
```

Verify:
```bash
ls -lh /home/mubasher/microros_shared/humble_f446/libmicroros/
# Expected:
# libmicroros.a   (~14 MB)
# include/        (ROS 2 message headers)
```

### Step 4: Clean up build artifacts (recover ~1GB)

```bash
rm -rf /home/mubasher/rtos_codes/pubsub_micro_ros/micro_ros_stm32cubemx_utils/microros_static_library_ide/build
rm -rf /home/mubasher/rtos_codes/pubsub_micro_ros/micro_ros_stm32cubemx_utils/microros_static_library_ide/install
```

---

## Project Structure

After setup, your shared library lives here and is never duplicated:

```
/home/mubasher/
├── microros_shared/
│   └── humble_f446/
│       └── libmicroros/
│           ├── libmicroros.a          ← shared across all projects (~14MB)
│           └── include/               ← ROS 2 message headers
│
└── rtos_codes/
    ├── pubsub_micro_ros/              ← project 1
    │   ├── Core/
    │   ├── Drivers/
    │   ├── Middlewares/
    │   └── micro_ros_stm32cubemx_utils/
    │       └── extra_sources/         ← transport layer (tiny, ~50KB)
    │
    └── new_project/                   ← project 2, 3, ...
        ├── Core/
        └── micro_ros_stm32cubemx_utils/
            └── extra_sources/         ← just copy this folder
```

---

## Creating a New micro-ROS Project

For every new STM32F446 micro-ROS project:

### Step 1: Generate base project in STM32CubeMX

Configure in STM32CubeMX:
- **MCU:** STM32F446ZE
- **FreeRTOS:** CMSIS-RTOS V2
- **UART3:** Async mode, 115200 baud, DMA TX+RX enabled
- **USB OTG FS:** CDC (optional)
- **Heap size:** minimum 24KB recommended
- **Stack size for micro-ROS task:** `3000 * 4` bytes (12KB)

### Step 2: Copy the transport sources

```bash
cp -r /home/mubasher/rtos_codes/pubsub_micro_ros/micro_ros_stm32cubemx_utils/extra_sources \
      /home/mubasher/rtos_codes/NEW_PROJECT/micro_ros_stm32cubemx_utils/
```

> ✅ Do NOT copy the full `micro_ros_stm32cubemx_utils` folder — only `extra_sources` is needed.

---

## STM32CubeIDE Configuration

Go to `Project → Properties → C/C++ Build → Settings`:

### Compiler Include Paths
`MCU GCC Compiler → Include paths` — add:
```
/home/mubasher/microros_shared/humble_f446/libmicroros/include
../micro_ros_stm32cubemx_utils/extra_sources
```

### Linker Library Path
`MCU GCC Linker → Libraries → Library search path (-L)` — add:
```
/home/mubasher/microros_shared/humble_f446/libmicroros
```

### Linker Library Name
`MCU GCC Linker → Libraries → Libraries (-l)` — add:
```
microros
```

### Source Paths
`C/C++ Build → Source Location` — add source path:
```
micro_ros_stm32cubemx_utils/extra_sources/microros_transports
```
With exclusion filter:
```
usb_cdc_transport.c|udp_transport.c|it_transport.c
```

### Pre-build Step (Optional — clear this)
`Build Steps → Pre-build steps` — **clear this field** if it contains a Docker command.  
The library is pre-built and shared; no pre-build step is needed.

---

## Code Template

Minimal working pub/sub example for STM32F446 + FreeRTOS:

```c
#include "main.h"
#include "cmsis_os.h"
#include <rcl/rcl.h>
#include <rclc/rclc.h>
#include <rclc/executor.h>
#include <std_msgs/msg/int32.h>
#include <rmw_microros/rmw_microros.h>

extern bool cubemx_transport_open(struct uxrCustomTransport *transport);
extern bool cubemx_transport_close(struct uxrCustomTransport *transport);
extern size_t cubemx_transport_write(struct uxrCustomTransport *transport, const uint8_t *buf, size_t len, uint8_t *errcode);
extern size_t cubemx_transport_read(struct uxrCustomTransport *transport, uint8_t *buf, size_t len, int timeout, uint8_t *errcode);
extern void *microros_allocate(size_t size, void *state);
extern void microros_deallocate(void *pointer, void *state);
extern void *microros_reallocate(void *pointer, size_t size, void *state);
extern void *microros_zero_allocate(size_t num, size_t size, void *state);

extern UART_HandleTypeDef huart3;

rcl_publisher_t publisher;
rcl_subscription_t subscriber;
std_msgs__msg__Int32 pub_msg;
std_msgs__msg__Int32 sub_msg;

void subscription_callback(const void *msgin)
{
    const std_msgs__msg__Int32 *msg = (const std_msgs__msg__Int32 *)msgin;
    pub_msg.data = msg->data * 5;
    rcl_publish(&publisher, &pub_msg, NULL);
}

void StartDefaultTask(void *argument)
{
    rmw_uros_set_custom_transport(
        true, (void *)&huart3,
        cubemx_transport_open, cubemx_transport_close,
        cubemx_transport_write, cubemx_transport_read
    );

    rcl_allocator_t freeRTOS_allocator = rcutils_get_zero_initialized_allocator();
    freeRTOS_allocator.allocate      = microros_allocate;
    freeRTOS_allocator.deallocate    = microros_deallocate;
    freeRTOS_allocator.reallocate    = microros_reallocate;
    freeRTOS_allocator.zero_allocate = microros_zero_allocate;
    rcutils_set_default_allocator(&freeRTOS_allocator);

    rclc_support_t support;
    rclc_support_init(&support, 0, NULL, &freeRTOS_allocator);

    rcl_node_t node;
    rclc_node_init_default(&node, "stm32_node", "", &support);

    rclc_publisher_init_default(
        &publisher, &node,
        ROSIDL_GET_MSG_TYPE_SUPPORT(std_msgs, msg, Int32),
        "stm32_reply"
    );

    rclc_subscription_init_default(
        &subscriber, &node,
        ROSIDL_GET_MSG_TYPE_SUPPORT(std_msgs, msg, Int32),
        "stm32_command"
    );

    rclc_executor_t executor;
    rclc_executor_init(&executor, &support.context, 1, &freeRTOS_allocator);
    rclc_executor_add_subscription(&executor, &subscriber, &sub_msg,
                                   &subscription_callback, ON_NEW_DATA);

    for (;;) {
        rclc_executor_spin_some(&executor, RCL_MS_TO_NS(10));
        osDelay(10);
    }
}
```

> 💡 Set FreeRTOS task stack size to `3000 * 4` (12KB) for the micro-ROS task.

---

## Building & Flashing

### Build in STM32CubeIDE
- `Project → Build Project` (Ctrl+B)
- The `.elf` file is at: `Debug/YOUR_PROJECT.elf`

> ⚠️ STM32CubeIDE may show "Build Failed. 1 errors" even when the `.elf` is successfully generated.  
> This is a known false-positive IDE bug. Verify with:
> ```bash
> ls -lh /home/mubasher/rtos_codes/YOUR_PROJECT/Debug/YOUR_PROJECT.elf
> ```
> If the file exists with a reasonable size (>100KB), the build succeeded.

### Flash via ST-Link

```bash
# Using st-flash
arm-none-eabi-objcopy -O binary Debug/YOUR_PROJECT.elf Debug/YOUR_PROJECT.bin
st-flash write Debug/YOUR_PROJECT.bin 0x8000000
```

Or use STM32CubeIDE: `Run → Run` (or `Run → Debug`)

---

## Testing with micro-ROS Agent

### Start the agent on your Linux PC

```bash
source /opt/ros/humble/setup.bash

# Via USB/UART (adjust port as needed)
ros2 run micro_ros_agent micro_ros_agent serial --dev /dev/ttyACM0 -b 115200
```

### Test pub/sub

```bash
# Terminal 1 — send command to STM32
ros2 topic pub /stm32_command std_msgs/msg/Int32 "{data: 5}"

# Terminal 2 — receive reply from STM32 (should return data * 5 = 25)
ros2 topic echo /stm32_reply

# Check node is visible
ros2 node list
# Expected: /stm32_node
```

---

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| `cannot find -lmicroros` | Wrong library path | Check `-L` path in linker settings |
| Build fails first time, passes second | Pre-build Docker race condition | Clear pre-build step; library is already shared |
| `_gettimeofday` warning | Expected on bare-metal | Safe to ignore |
| `warn_unused_result` warnings | rcl return values ignored | Safe to ignore, or capture return values |
| micro-ROS agent not connecting | Wrong UART port or baud | Check `/dev/ttyACM0` vs `/dev/ttyUSB0`, verify 115200 |
| FreeRTOS stack overflow | micro-ROS task stack too small | Set stack to minimum `3000 * 4` bytes |
| `Build Failed` but `.elf` exists | STM32CubeIDE false-positive bug | Verify `.elf` size; flash and test normally |

---

## Reuse Checklist for New Projects

- [ ] Copy `extra_sources/` to new project
- [ ] Set include path → `/home/mubasher/microros_shared/humble_f446/libmicroros/include`
- [ ] Set library path `-L` → `/home/mubasher/microros_shared/humble_f446/libmicroros`
- [ ] Set library `-l` → `microros`
- [ ] Add `extra_sources/microros_transports` as source path (with exclusions)
- [ ] Clear pre-build Docker command
- [ ] Set micro-ROS FreeRTOS task stack to `3000 * 4`

---

*Shared library path: `/home/mubasher/microros_shared/humble_f446/libmicroros/`*  
*Only rebuild library if changing MCU target or ROS 2 distro.*
