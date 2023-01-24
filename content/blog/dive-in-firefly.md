---
title: "Hardware Dive-In | FireFly: A High-Throughput and Reconfigurable Hardware Accelerator for Spiking Neural Networks"
description: "First post of the dive-in series: in these blog post, we will investigate thoroughly research papers about digital hardware design for neuromorphic applications."
image: /images/blog/firefly/dive-in-firefly-architecture.png
draft: true
date: 2023-01-23
type: "post"
tags: ["research", "hardware", "digital", "neuromorphic", "FPGA", "accelerator", "Verilog", "Xilinx", "Ultrascale",]
showTableOfContents: true
---

![FireFly architecture](/images/blog/dive-in-firefly/firefly-architecture.webp)

This is an SNN digital accelerator (in particular, an accelerator for Spiking Convolutional Neural Networks (SCNNs)) targeted to FPGA platforms; the target platformsare edge devices, such the [Avnet Ultra96-V2](https://www.xilinx.com/products/boards-and-kits/1-vad4rl.html). This kind of SoC is an ARM-based, Xilinx Zynq Ultrascale+ MPSoC development board, very popular among designers (now you know what I would like as a birthday gift :stuck_out_tongue_closed_eyes:). The microprocessor can communicate with the logic fabric (i.e. the FPGA itself) through an [AXI](https://support.xilinx.com/s/article/1053914?language=en_US) interface.

Differently from other FPGA solutions, here the Xilinx IPs available are exploited: in particular, the DSP, called **DSP48E2**, is used to perform the neurons membrane update during inference. This allows the architecture to reach a throughput of **5.52TSOP/s** at a clock frequency of **300MHz**.

## Introduction

Most FPGA implementation of SCNN accelerator are not able to automatically synthesise the RTL code to the arithmetic hard blocks available on the chip; for this reason, the neurons update circuit is placed on the fabric logic, without any further optimisations; this impacts negatively the number of resources occupied (LUTs, BRAMs and so on) and the design latency (and critical path). The authors, instead, map the SNN update functionality to the DSPs of the chip, increasing the design performance and efficiency.

Another issue that the authors deal with in the design is the **memory hierarchy**: it is well known that the on-chip BRAM is much faster than the off-chip DRAM; for this (and energy efficiency) reason(s), one needs to minimize the number of off-chip memory accesses, trying to maximize the number of data written to and read from it at each iteration. This is particularly true for SNNs: when a spiking neural networks evolves through time, the synapse weight and the membrane potential have to be retrieved at every timestep. For this reason, **data reusage** becomes crucial for both time and energy efficiencies, since access to off-chip memory consumes much more energy and time than accessing the same data on the BRAMs.

This work differs from others also because it is a **forward** architecture: this means thath the SNN is evaluated layer by layer, and each layer outputs a **feature map** of activations that, in the SNN case, are **spikes**, which corresponds to 0 and 1 bits. 

![Forward architecture of an SNN](/images/blog/dive-in-firefly/forward-architecture.webp)

