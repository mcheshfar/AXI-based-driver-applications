# Edge Detection Accelerator — AXI4-Stream Hardware Accelerator

This project implements a configurable hardware accelerator for image edge detection using the **Roberts** and **Sobel** operators. The accelerator is integrated into a Xilinx Zynq MPSoC system through **AXI4-Stream** interfaces for image transfer and an **AXI4-Lite** interface for accelerator configuration.

The accelerator is controlled by the custom **`/dev/uniss_dma`** Linux driver, which provides both:

- AXI DMA data transfers
- AXI-Lite register access through ioctl commands

The userspace application demonstrates complete accelerator execution, from image loading to hardware configuration, DMA transfers, and result verification.

---

# Features

- AXI4-Stream image input
- AXI4-Stream configuration input (image width)
- AXI4-Stream processed image output
- AXI4-Lite accelerator control interface
- Runtime selection of:
  - Roberts operator
  - Sobel operator
- Custom Linux DMA driver exposing:
  - DMA buffer management
  - DMA control
  - AXI-Lite register read/write
- Userspace C application demonstrating complete execution flow

---

# Accelerator Interfaces

The accelerator exposes four interfaces:

| Interface | Description |
|-----------|-------------|
| AXIS Input 0 | Input grayscale image |
| AXIS Input 1 | Image width configuration |
| AXIS Output | Edge detection result |
| AXI-Lite | Accelerator configuration and control |

The AXI-Lite interface is used to configure the accelerator before DMA transfers begin.

---

# AXI-Lite Registers

| Offset | Register | Description |
|---------|----------|-------------|
| 0x00 | REG0 | Control register |
| 0x04 | REG1 | Number of output pixels |

REG0 contains:

- bit 0 : Start
- bit 2 : Enable output counter
- bits 31:24 : Kernel selection

Kernel IDs:

| Kernel | Value |
|---------|------|
| Roberts | 1 |
| Sobel | 2 |

Example:

```c
REG1 = output_pixels;
REG0 = (kernel << 24) | 0x4;
REG0 = (kernel << 24) | 0x1;
```

---

# DMA Channel Assignment

The reference Vivado design connects three AXI DMA channels:

| DMA Channel | Direction | Connected Interface |
|-------------|-----------|--------------------|
| Channel 0 | MM2S | Input image stream |
| Channel 1 | MM2S | Image width/configuration stream |
| Channel 2 | S2MM | Edge detection output |

---

# Software Flow

The userspace application performs the following operations.

## 1. Load input image

The application loads an 8-bit grayscale image (PGM format), allocates DMA buffers, and converts each pixel into a 32-bit word.

```
PGM image
      │
      ▼
32-bit DMA input buffer
```

---

## 2. Configure accelerator

Before any DMA transfer begins, the accelerator is configured through AXI-Lite registers exposed by the `/dev/uniss_dma` driver.

The application writes:

- output image size (REG1)
- selected kernel (REG0)
- start command (REG0)

using the AXI-Lite ioctl interface.

---

## 3. Reset DMA

```c
ioctl(fd, IOCTL_DMA_RESET_ALL, 0);
```

All DMA channels are reset before starting a new execution.

---

## 4. Start output DMA

The output DMA is started first.

```text
Channel 2 (S2MM)
```

This prepares the output buffer to receive processed pixels.

---

## 5. Send configuration

The image width is transferred through DMA Channel 1.

```text
Channel 1 (MM2S)
```

---

## 6. Send image

The grayscale image is transferred through DMA Channel 0.

```text
Channel 0 (MM2S)
```

---

## 7. Wait for completion

The software polls the DMA status registers until all channels become idle.

DMA error conditions (slave errors, decode errors, DMA internal errors) are also checked.

---

## 8. Read processed image

The processed pixels are copied back into userspace.

The application prints the resulting edge-detected image.

---

# Dataflow

```
                 AXI-Lite
                     │
                     ▼
             Configure Accelerator
                     │
                     ▼

        Channel 0 (MM2S)
Input Image ----------------------->

        Channel 1 (MM2S)
Image Width ----------------------->

                         Edge Detection
                             IP
                               │
                               ▼

        Channel 2 (S2MM)

Processed Image <-------------------
```

---

# Extended AXI DMA Driver

The custom `/dev/uniss_dma` driver provides both DMA management and AXI-Lite register access.

## DMA IOCTLs

| IOCTL | Function |
|--------|----------|
| IOCTL_SELECT_CHANNEL | Select active DMA channel |
| IOCTL_DMA_WRITE_BUFFER | Copy userspace data into DMA buffer |
| IOCTL_DMA_READ_BUFFER | Copy DMA buffer to userspace |
| IOCTL_DMA_START_TRANSFER | Start DMA transaction |
| IOCTL_READ_STATUS_REGISTER | Read DMA status |
| IOCTL_DMA_RESET | Reset current DMA |
| IOCTL_DMA_RESET_ALL | Reset all DMA channels |

## AXI-Lite IOCTLs

| IOCTL | Function |
|--------|----------|
| IOCTL_AXILITE_WRITE_REG | Write accelerator register |
| IOCTL_AXILITE_READ_REG | Read accelerator register |

The driver maps the AXI-Lite register space during probe using the Device Tree property:

```dts
axi_lite-handle = <&s_accelerator_0>;
```

This eliminates the need for `/dev/mem` access from userspace.

---

# Directory Layout

```text
edge_detector/
├── sw/
│   ├── acc_test_edgedetect.c
│   ├── input.pgm
│   ├── pl_s.dtbo
│   └── pl_s.dtsi
├── vivado/
└── README.md
```

---

# Build

```bash
gcc acc_test_edgedetect.c -o acc_test_edgedetect
```

---

# Run

Roberts operator

```bash
sudo ./acc_test_edgedetect roberts input.pgm
```

Sobel operator

```bash
sudo ./acc_test_edgedetect sobel input.pgm
```

If no arguments are supplied, the application executes the Roberts operator using `input.pgm`.

---

# Example Execution

```text
Input image (64x64)

Accelerator configured:
REG0 = 0x02000001
REG1 = 3969

DMA transfers started.

DMA0 status = DONE
DMA1 status = DONE
DMA2 status = DONE

Sobel output (3969 words)

...
```

The output image consists of one 32-bit word per output pixel, with the processed grayscale value stored in the least significant byte.