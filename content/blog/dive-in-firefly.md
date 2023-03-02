---
title: "Hardware Dive-In | FireFly: A High-Throughput and Reconfigurable Hardware Accelerator for Spiking Neural Networks"
description: "First post of the dive-in series: in these blog post, we will investigate thoroughly research papers about digital hardware design for neuromorphic applications."
image: /images/blog/firefly/dive-in-firefly-architecture.png
draft: false
date: 2023-03-02
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

## The SNN implementation 

The neuron models adopted in this work as the **IF** and **LIF** ones. The conventional discretised equation is adopted:

$$ v_{i}[t] = (1 - \frac{1}{\tau_{m}})u_{i}[t] + I[t] $$

As usual, the membrane potential at timestep $t$ is obtained as the sum of the pre-synaptic currents, $I[t]$, and the decayed membrane potential (the decay term is ignored when using the IF model). The pre-synaptic current is described by the following equation: 

$$ I[t] = \sum_{j}w_{ij}s_{j}[t] + b_{i} $$ 

The synapses weights $w_{ij}$ are multiplied by the incoming spikes $s_{j}[t]$ and summed up. A bias term $b_{i}$ is considered for each post-synaptic neuron.

The membrane potential $u_{i}$ update is then performed using the following equation:

$$ u_{i}[t+1] = 0~\mathrm{if}~v_{i}[t] \geq \theta~\mathrm{else}~v_{i}[t] $$
$$ s_{i}[t+1] = 1~\mathrm{if}~v_{i}[t] \geq \theta~\mathrm{else}~0 $$

where $\theta$ is the spiking threshold. 

From these equations, two claims can be made:
* the most **computationally** demanding task is $I[t]$, since all the weights have to be multiplied by the spikes and accumulated. 
* the most demanding taks from a **memory-access** point of view is the membrane potential $u_{i}[t]$, since it has to be retrieved from memory and updated at each timestep.

## Spiking convolution

Let us consider the discrete convolution operation in a spiking layer. Consider the following quantities:
* $c_{i}$ represents the number of channels in the **input** feature map. 
* $c_{o}$ representes the number of channels in the **output** feature map.
* $W_{i}$ is the **width** of the **input** feature map. 
* $H_{i}$ is the **height** of the feature map. 
* $K_{w}$ is the **width** of the **kernel**.
* $K_{h}$ is the **height** of the kernel.
* $W_{o}$ is the **width** of the **output** feature map. 
* $H_{o}$ is the **height** of the feature map. 

A convolutional kernel $K_{w} \times K_{h}$ represents the set of synapes connected to the output neurons. Considering a timestep $t$, these neurons are convolved with the whole feature map. This means that the spikes of each feature map convolutional window are used to **charge** the post-synaptic neurons. At the end of the convolution, the membrane potentials of these are **thresholded** and the corresponding spikes are generated. 

Let us write some C-like code to describe this operation. 

```c
/** Function thas performs a spiking convolution.
  *
  * @param[in]      featureMapIn    Input feature map.
  * @param[in]      kernel          The weights kernel.
  * @param[out]     featureMapOut   Output feature map.
  * @param[inout]   membranes       Membrane potentials.
  * @param[in]      theta           Spiking threshold.
  * @param[in]      lambda          Leakage factor. 
  */
void spiking_convolution (
    spike_t featureMapIn[c_i][H_i][W_i], 
    weights_t kernel[c_o][c_i][K_h][K_w], 
    spike_t featureMapOut[c_o][H_o][W_o],
    membrane_t membranes[c_o][H_o][W_o],
    membrane_t theta, 
    membrane_t lambda
    ) {

    int i, j, k, x, y, z; 
    
    // Looping over output channels.
    for (k=0; k < c_o; k++) { // Output channels.
        // Looping over input feature map.
        for (i=0; i < W_i-K_w; i++) { // Feature map height.
            for (j=0; j < H_i-K_h; j++) { // Feature map width.
                // Looping over input channels.
                for (z=0; z < c_i; z++) { // Input channels.
                    // Looping over the weights kernel.
                    for (y=0; y < K_h; y++) { // Kernel height.
                        for (x=0; x < K_h; x++) { // Kernel width.
                            // Multiplex and accumulate.
                            if (featureMapIn[z][i+y][j+x])
                                membranes[k][i+K_h/2][j+K_w/2] += 
                                    kernel[k][z][y][x]; 
                        }
                    } // End kernel loop.
                } // End input channels loop.
            }
        } // End input feature map loop.
        // Looping over the membranes. 
        for (i=0; i < H_o; i++) {
            for (j=0; j < W_o; j++) {
                // Leakage. 
                membranes[k][i][j] *= 1 - lambda; 
                // Thresholding.
                if (membranes[k][i][j] >= theta) {
                    featureMapOut[k][i][j] = 1; 
                    membranes[k][i][j] = 0; 
                } else {
                    featureMapOut[k][i][j] = 0; 
                }
            }
        }
    } // End output channels loop.
```

I know, it's kind of confusing. This is still a standard two-dimensional convolution! 

The input feature map is a $c_{i} \times H_{i} \times W_{i}$ tensor made of spikes. This is convoled with $c_{o}$ kernels, which values are the synapses connected to the neurons in the output feature map. Each kernel is a $c_{i} \times K_{h} \times K_{w}$ tensor. Supposing that we use a **stride** equal to 1 and that the input feature map uses a **padding** equal to 1, the output is a $c_{o} \times H_{o} \times W_{o}$ tensor, where:
* $H_{o} = \frac{H_{i} + 2p - K_{h}}{s} + 1 = H_{i} + 3 - K_{h}$.
* $W_{o} = \frac{W_{i} + 2p - K_{w}}{s} + 1 = W_{i} + 3 - K_{w}$.

What changes from standard two-dimensional convolution is that the **multiply-and-accumulate** operation is substituted by a **multiplex-and-accumulate** one. Let's take a look at that code snippet.

```c
// Multiplex and accumulate.
if (featureMapIn[z][i+y][j+x])
    membranes[k][i+K_h/2][j+K_w/2] += 
        kernel[k][z][y][x]; 
```

Instead of multiply the input feature with the synapse, knowing that the first is either 0 or 1, we choose to **accumulate** the synapse weight if the input spike is 1, otherwise we don't.

The other difference is that once the convolution with a filter is finished, we evaluate the resulting **membrane potentials**, in which the synapses weights have been accumulated during convolution; in particular, we compare these against the **spiking threshold** $\theta$ and, if these are larger than or equal to it, the associated neurons **spike** and their membranes are **reset** to 0. The following is the code that implements this operations.

```c
// Thresholding.
if (membranes[k][i][j] >= theta) {
    featureMapOut[k][i][j] = 1; 
    membranes[k][i][j] = 0; 
} else {
    featureMapOut[k][i][j] = 0; 
}
```
