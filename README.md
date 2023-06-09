![Deploy](https://img.shields.io/badge/deploy-vivado-FF1010.svg)

FPGA-SATA
=============================

SATA Gen2 host (HBA) running on Xilinx FPGA with GTH. This library provides examples based on the official development board of [netfpga-sume](https://www.xilinx.com/products/boards-and-kits/1-6ogkf5.html), which can realize hard disk reading and writing.

This open source library provides a general performance version, and uses the .dcp netlist to encrypt the core code. If you need the netlist of the medium and high performance version, Verilog source code, or customize the SATA controller with other functions, please contact Mail : 629708558@qq.com QQ : 629708558

In addition, I wrote a technical article introducing SATA [Analysis of SATA Protocol: From Serial Signals to Reading and Writing Hard Disks](https://zhuanlan.zhihu.com/p/554251608) to help you understand the details of the SATA protocol stack .

# Introduction

SATA is the most widely used interface protocol for hard drives. **Figure 1** is the SATA architecture, in which the SATA host (HBA) is the hard disk read-write controller, which is often implemented by the motherboard chipset in the computer, and here it is implemented by the FPGA. The SATA device is a hard disk (mechanical hard disk or solid-state hard disk).

| ![SATA_ARCH](./figures/sata_arch.png) |
| :-------------------------------------: |
| **Figure 1**: SATA protocol stack |

The SATA protocol includes from bottom to top: Physical Layer (Physical Layer, PHY), Link Layer (Link Layer), Transport Layer (Transport Layer), Command Layer (Command Layer):

- **Physical layer**: The downstream uses two pairs of serial differential signal pairs to connect to the SATA device, including the differential pair for sending (SATA_A+, SATA_A-) and the differential pair for receiving (SATA_B+, SATA_B-). After clock recovery and serial-to-parallel conversion of the serial signal, the parallel signal is used to interact with the upstream link layer.
- **Link layer and transport layer**: Realize from bottom to top: 8b10b codec, scrambling/descrambling, CRC calculation and verification, flow control, data caching, encapsulation and analysis of FIS packets. Interact with the upstream command layer using a packet structure called **Frame Information Structures** (**FIS**).
- **Command Layer**: Accept upstream read and write commands, generate and parse FIS, and realize hard disk read and write operations.

This library only uses hardware to implement the functions from the physical layer to the transport layer, for the following reasons:

- Although the function from the physical layer to the transport layer is not simple, the purpose is relatively single, just to provide a reliable FIS transmission mechanism, with a high degree of internal coupling. Moreover, FIS is a unified data stream format, so the data interface from the transport layer to the command layer is relatively simple, easy to understand and use.
- The reason why the command layer is not implemented is that due to historical compatibility reasons, the ATA and ATAPI command sets supported by SATA have many hard disk read and write methods, including PIO, DMA, etc., so a complete command layer needs A state machine that implements numerous and complex commands. But its purpose is not complicated, it is all to achieve hard disk reading and writing in various ways.

The implementation of each layer in this library is as follows:

- Although I didn't use hardware to implement the command layer, I implemented a sample project using UART to send and receive FIS in FPGA, which provides UART "FIS transparent transmission" function: you can use the serial port software in the computer (such as minicom, putty, hyper -terminal, serial port assistant, etc.) to send FIS to Device, and can see the FIS sent by Device. Based on this, as long as we send data according to the FIS format specified by the command layer, the hard disk can be read and written.
- The link layer and transport layer are implemented in Verilog (encrypted as a dcp netlist), but 8b10b codec in the link layer is not included, because Xilinx's GT primitives can implement 8b10b codec.
- Physical Layer: Implemented with Xilinx's GTH primitives. Therefore, this project supports Xilinx FPGA with GTH, such as Virtex7, Zynq Ultrascale+, etc.



# code file

The following is the tree structure of the code files of this project:

- vivado_netfpgasume/**fpga_top.xdc** : Pin constraint file, constrain the port to [netfpga-sume](https://www.xilinx.com/products/boards-and-kits/1-6ogkf5.html) on the pin.
- RTL/fpga_uart_sata_example/**fpga_uart_sata_example_top.sv** : FPGA project top level (**Verilog**)
   - RTL/IP/clk_wiz_0 : Generate 60MHz clock for CPLL_refclk of SATA HBA core (**Xilinx clock wizard IP**)
   - RTL/sata_hba/**sata_hba_top.sv** : SATA HBA top layer, implements physical layer, link layer, transport layer (**Verilog**)
     - RTL/sata_hba/**sata_gth.sv** : implements the physical layer and calls Xilinx GTH related primitives (**Verilog**)
     - RTL/sata_hba/**sata_link_transport.dcp** : implement link layer and transport layer (**dcp encrypted netlist**)
   - RTL/fpga_uart_sata_example/**uart_rx.sv** : Receive the UART signal from Host-PC, parse it into FIS data stream and send it to HBA (**Verilog**)
   - RTL/fpga_uart_sata_example/**uart_tx.sv** : Receive FIS received by HBA, convert it into human-readable ASCII-HEX format and send it to Host-PC with UART (**Verilog**)

| ![fpga_example](./figures/fpga_example.png) |
| :------------------------------------------------ ----------: |
| **Figure 2**: The code structure of this library. Where xfis represents the FIS send stream, and rfis represents the FIS receive stream. |



# SATA HBA module interface description

RTL/sata_hba/**sata_hba_top.sv** is the top-level module of SATA HBA core, which has a simple FIS transceiver interface. If you want to implement other SATA applications, such as writing the command layer yourself, you need to fully understand the interface waveform of **sata_hba_top.sv**.

First of all, we need to understand the structure of the FIS packet, as shown in the following table. The length of FIS must be a multiple of 4 bytes. Wherein the CRC is invisible to the module caller, because **sata_hba_top.sv** is automatically inserted, checked and removed. The module caller only needs to send and receive the FIS-type field and the Payload field as a whole. In the context below, we use **FIS Length** to refer to the total length of the FIS-type + Payload field (minimum 1 dword, maximum 2049 dword), excluding CRC.

​ *Table: FIS packet structure*

| Field | FIS-type | Payload | CRC |
| :----------: | :----------: | :----------: | :--------- ------------------: |
| length (byte) | 4 | 0\~8192 | 4 |
| length (dword) | 1 | 0\~2048 | 1 |
| Sent | User Sent Required | User Sent Required | User Sent Not Required (Hardware Automatically Inserted) |
| Receive | Visible to user | Visible to user | Invisible to user (hardware check and delete) |

> :point_right: The dword term translates to "double word", which means 4 bytes.



The following is the interface definition of **sata_hba_top.sv**, where the signal beginning with **rfis_** constitutes an interface similar to AXI-stream source, which will send the received FIS to the user. The signals starting with **xfis_** form an interface similar to AXI-stream sink, and users need to use it to send FIS. They are all synchronized to the clk clock signal.

```verilog
module sata_hba_top ( // SATA gen2
     input wire rstn, // 0:reset 1:work
     input wire cpll_refclk, // 60MHz clock is required
     // SATA GT reference clock, connect to clock source on FPGA board
     input wire gt_refclkp, // 150MHz clock is required
     input wire gt_refclkn, //
     // SATA signals, connect to SATA device
     input wire gt_rxp, // SATA B+
     input wire gt_rxn, // SATA B-
     output wire gt_txp, // SATA A+
     output wire gt_txn, // SATA A-
     // user clock output
     output wire clk, // 75/150 MHZ user clock
     // =1 : link initialized
     output wire link_initialized,
     // to Command layer : RX FIS data stream (AXI-stream liked, without tready handshake, clock domain = clk)
     // example wave : // | no data | FIS1 (no error) | no data | FIS2 (error) | no data | FIS3 (no error) | no data |
     output wire rfis_tvalid, // 0000000000001111111111111111111000000000000000000000000000000000000000111111111111111111100000000000 000 // rfis_tvali
