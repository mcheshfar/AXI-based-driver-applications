
# AXI DMA Kernel Module (`axidma.c`)

## Overview

This Linux kernel module provides a character device interface for interacting with AXI DMA hardware and AXI-Lite peripherals on Xilinx Zynq and Zynq UltraScale+ MPSoC platforms.

The driver automatically discovers multiple AXI DMA channels from the Device Tree using the `dev-handles` property and maps an accelerator AXI-Lite register space using the `axi_lite-handle` property.

Through the `/dev/uniss_dma` character device, user-space applications can:

- Select a DMA channel.
- Transfer data to and from DMA buffers.
- Start DMA transfers.
- Reset individual or all DMA engines.
- Read DMA status registers.
- Read accelerator registers through AXI-Lite.
- Write accelerator registers through AXI-Lite.

This provides a single software interface for both high-speed AXI DMA transfers and low-bandwidth accelerator configuration.

## Features

- Automatic discovery of multiple AXI DMA channels from the Device Tree.
- Support for both MM2S and S2MM DMA channels.
- Coherent DMA buffer allocation for each channel.
- Character device interface (`/dev/uniss_dma`).
- AXI-Lite register access through ioctl commands.
- Userspace control of DMA transfers and accelerator configuration using a single driver.

## Device Tree Requirements

The driver expects a platform node with two properties:

- `dev-handles` — list of AXI DMA controller phandles.
- `axi_lite-handle` — phandle to the accelerator AXI-Lite slave.

Each DMA controller should define its MM2S and/or S2MM channels.

Example:

```dts
axi_dma_0: dma@...
{
    compatible = "xlnx,axi-dma";

    mm2s_channel {
        mm2s = <1>;
        xlnx,datawidth = <32>;
        xlnx,device-id = <0>;
    };

    s2mm_channel {
        mm2s = <0>;
        xlnx,datawidth = <32>;
        xlnx,device-id = <1>;
    };
};

s_accelerator_0: s_accelerator@a0010000 {
    compatible = "unica,s-accelerator";
    reg = <0x0 0xa0010000 0x0 0x10000>;
};

dmas {
    compatible = "uniss,dmas";
    status = "okay";

    dev-handles = <&axi_dma_0 &axi_dma_1 &axi_dma_2>;

    axi_lite-handle = <&s_accelerator_0>;
};
```

The `axi_lite-handle` property allows the driver to map the accelerator register space during probe, eliminating the need for user-space access through `/dev/mem`.



## Environment setup
The `set_env.sh` script automates the entire process of compiling and installing the AXI DMA driver on your system.

### What the script does:
1.  **Repository Cloning**: Downloads the latest version of the `AXI-based-drivers` repository directly from GitHub.
2.  **Dependency Installation**: Runs `install_libs.sh` to install the necessary system libraries and kernel headers required for compilation.
3.  **Driver Compilation**: Navigates to the `axidma-driver` directory and builds the source code using the `make` command.
4.  **Module Installation**:
    * Creates the required directory in `/lib/modules/` for extra kernel modules.
    * Copies the compiled binary file (`axidma.ko`) to the system module path.
    * Updates the kernel module dependency list using `depmod`.
5.  **Persistence at Boot**: Configures the system to automatically load the `axidma` module during every boot by creating a configuration file in `/etc/modules-load.d/`.
6.  **Loading and Verification**: Immediately loads the driver into the kernel using `modprobe` and displays the status via `lsmod` to confirm a successful installation.
---

## Script Usage
To set up your environment, run the following commands in your terminal:
```bash
# Make the script executable
chmod +x set_env.sh

# Start the installation
./set_env.sh
```

Alternatively, you can build the module yourself using these steps.

## Building the Module

### Prerequisites

- Kernel headers for your target platform.
- A cross-compiler toolchain (for embedded platforms like Zynq).

### Build

Create a simple Makefile:

```makefile
obj-m += axidma.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

Then run:
```bash
make
```

## Installation

```bash
sudo insmod axidma.ko
```

Check `dmesg` for logs confirming successful channel probing and DMA buffer allocation.

To remove:
```bash
sudo rmmod axidma
```

## Device Interface

Once loaded, the module creates a character device:

```
/dev/uniss_dma
```
## `ioctl` Commands

| Command | Code | Description |
|---------|------|-------------|
| `IOCTL_SELECT_CHANNEL` | `_IOW('D',0,int)` | Select active DMA channel |
| `IOCTL_DMA_WRITE_BUFFER` | `_IOW('D',1,unsigned char *)` | Copy data into an MM2S DMA buffer |
| `IOCTL_DMA_READ_BUFFER` | `_IOR('D',2,unsigned char *)` | Copy data from an S2MM DMA buffer |
| `IOCTL_DMA_START_TRANSFER` | `_IOW('D',3,size_t)` | Start a DMA transfer |
| `IOCTL_READ_STATUS_REGISTER` | `_IOR('D',4,unsigned int *)` | Read DMA status register |
| `IOCTL_DMA_RESET` | `_IOW('D',5,size_t)` | Reset the selected DMA channel |
| `IOCTL_DMA_RESET_ALL` | `_IOW('D',6,size_t)` | Reset all DMA channels |
| `IOCTL_AXILITE_WRITE_REG` | `_IOW('D',7,struct axi_lite_reg_io)` | Write an AXI-Lite register |
| `IOCTL_AXILITE_READ_REG` | `_IOWR('D',8,struct axi_lite_reg_io)` | Read an AXI-Lite register |

The DMA channel numbering follows the order specified in the Device Tree.

## AXI-Lite Register Access

The driver exposes the accelerator AXI-Lite register space through two ioctl commands.

Applications exchange register information using:

```c
struct axi_lite_reg_io {
    uint32_t offset;
    uint32_t value;
};
```

Writing a register:

```c
struct axi_lite_reg_io reg;

reg.offset = 0x04;
reg.value  = 128;

ioctl(fd, IOCTL_AXILITE_WRITE_REG, &reg);
```

Reading a register:

```c
struct axi_lite_reg_io reg;

reg.offset = 0x00;

ioctl(fd, IOCTL_AXILITE_READ_REG, &reg);

printf("REG0 = 0x%08X\n", reg.value);
```

This interface allows user-space applications to configure hardware accelerators without requiring direct `/dev/mem` access.
## Usage Example (C)

```c
int fd = open("/dev/uniss_dma", O_RDWR);

/* Configure accelerator through AXI-Lite */

struct axi_lite_reg_io reg;

reg.offset = ACCEL_REG1;
reg.value  = out_pixels;
ioctl(fd, IOCTL_AXILITE_WRITE_REG, &reg);

reg.offset = ACCEL_REG0;
reg.value  = (kernel << 24) | 0x1;
ioctl(fd, IOCTL_AXILITE_WRITE_REG, &reg);

/* Prepare output DMA */

ioctl(fd, IOCTL_SELECT_CHANNEL, 2);
ioctl(fd, IOCTL_DMA_START_TRANSFER, out_pixels * 4);

/* Send configuration */

ioctl(fd, IOCTL_SELECT_CHANNEL, 1);
ioctl(fd, IOCTL_DMA_WRITE_BUFFER, cfg);
ioctl(fd, IOCTL_DMA_START_TRANSFER, 4);

/* Send image */

ioctl(fd, IOCTL_SELECT_CHANNEL, 0);
ioctl(fd, IOCTL_DMA_WRITE_BUFFER, img);
ioctl(fd, IOCTL_DMA_START_TRANSFER, image_size);

/* Read processed image */

ioctl(fd, IOCTL_SELECT_CHANNEL, 2);
ioctl(fd, IOCTL_DMA_READ_BUFFER, output);
```

A complete userspace example is provided in `acc_test_edgedetect.c`.

## Debugging

The driver includes extensive `printk()` logging for DMA and AXI-Lite operations.

Useful commands include:

```bash
dmesg | grep uniss
dmesg | grep DMA
```

Typical messages include:

- DMA channel discovery
- DMA buffer allocation
- DMA transfer start
- DMA reset
- AXI-Lite register reads and writes
- Device probe and removal

## Known Limitations

- The module assumes `DMA_BIT_MASK(32)` support.
- No interrupt support — relies on polling DMA status.
- Currently no support for VDMA or Multi-channel DMA.

## License

**GPLv2** — See `MODULE_LICENSE("GPL")` in the source.

## Author

**Giuseppe Satta** — Original author and maintainer of the AXI DMA kernel module.

## Contributor

**Mohammad Cheshfar** — Added AXI-Lite register support, extended ioctl interface, Device Tree integration, userspace applications, and documentation.