# AXI-based driver - Test applications

The driver is available into the platform-specific folders, and test-specific applications have been implemented to verify its functionality.

## Requirements for Custom IP

The `/dev/uniss_dma` driver controls an AXI DMA and only supports transfers on AXI4-Stream interfaces. It uses the MM2S channel of the DMA to send data and the S2MM channel to receive data. Consequently, to use this driver, a custom IP must have at least:

* **1 AXIS input port** (e.g., `im_axis_in`) connected to `M_AXIS_MM2S` of the DMA
* **1 AXIS output port** (e.g., `current_pos`) connected to `S_AXIS_S2MM` of the DMA

## Repository Structure

The repository is organized by target hardware boards. Inside each board folder, you will find the operating system drivers and the test applications:

### 1. Target Boards

* **`kv260/`**: Contains the drivers and applications configured for the Xilinx Kria KV260 Vision AI Starter Kit.
* **`zedboard/`**: Contains the drivers and applications configured for the Avnet ZedBoard Zynq-7000 Development Board.

### 2. Inner Directory Structure

Inside each board's directory (`kv260/` and `zedboard/`), the content is structured as follows:

* **`ubuntu-driver/`** *(Available for kv260)*: Contains the specific instructions and source code to compile and install the driver on Ubuntu.
* **`yocto-driver/`**: Contains the specific instructions and source code to compile and install the driver using the Yocto Project build system.
* **`applications/`**: Contains examples of hardware accelerators that implement the AXI4-Stream interface described above, including their test applications to verify correct operation.
