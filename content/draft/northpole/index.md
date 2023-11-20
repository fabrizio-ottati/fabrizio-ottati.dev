---
title: "Neural inference at the frontier of energy, space and time - NorthPole, IBM"
description: "Translating to human language the new paper from IBM on NorthPole."
image: brain-to-chip.png
draft: false
date: 2023-11-19
type: "post"
tags: ["research", "hardware", "digital", "deep learning"]
showTableOfContents: true
---

# Still in draft mode!

In NorthPole, there is this *axiomatic* terminology: the authors claim that NorthPole design has been driven by 10 axioms inspired by human biology. We will use them as guidelines to analyze the paper. 

## Axiomatic design

> NorthPole, an architecture and a programming model for neural inference, reimagines (Fig. 1) the interaction between compute and memory by embodying 10 interrelated, synergistic axioms that build on brain-inspired computing. 

Fancy terminology :)

### Axiom 1
> Turning to architecture, NorthPole is specialized for neural inference. For example, it has no data-dependent conditional branching, and it does not support training or scientific computation.

NorthPole is an application specific integrated circuit (ASIC): it can run only deep neural networks (DNNs). Regarding "no data-dependent conditional branching", it means that is supports a *static scheduling*: the succession of instructions to be performed is static and pre-defined, as it should be when you are performing matrix multiplication and running neural networks. There are no branches with data-dependent `if`s. 

This is extremely important, because engineers have spent a lot of time making branches impact on performance minimum, especially in superscalar processors that have to deal with any possible code being compiled for them. For more information on this, the reader is referred to [Computer architecture](https://books.google.it/books/about/Computer_Architecture.html?id=v3-1hVwHnHwC&redir_esc=y), by Hennessy, Patterson and Asanovic.

Moreover, NorthPole is an inference-only accelerator, *i.e.*, you cannot train a deep neural network (DNN) on it, but only run it in inference. Training is important, but inference is as fundamental: for instance, consider that at the current moment each ChatGPT inference costs 0.04$ to OpenAI [[Semianalysis](https://www.semianalysis.com/p/the-inference-cost-of-search-disruption)] to run in the cloud, just considering the number of machines and the electricity needed to power them and keep them cool. 

Things change if you take into account sparsity, *i.e.*, when you take into account that layers in DNNs output a lot of zeros; since mutiplying by zero is useless, a smart approach would lead to skip all the operations with zeros involved. However, also in that case you need to introduce a structured approach (*i.e.*, the zeros distribution in the matrices cannot be random, but it has to follow a structure) if you want to improve performance using sparsity [[Wu et al.](https://arxiv.org/abs/2305.12718)]. For instance, Nvidia graphics processing units (GPUs) support 2:4 sparsity, which means that every 4 elements in a matrix, 2 are zeros (more or less, I am not being extremely precise on this).

### Axiom 2 
> Inspired by biological precision , NorthPole is optimized for 8, 4, and 2-bit low-precision. This is sufficient to achieve state-of-the-art inference accuracy on many neural networks while dispensing with the high-precision required for training.

Well, we have been running DNNs using INT8 since ~2017 without claiming biological inspiration. Recently, however, progress has been made and we can use INT4 quantization with marginal loss in performance compared to the 32-bit floating point (FP32) baseline that you trained on your GPU [[Keller et al.](https://ieeexplore.ieee.org/abstract/document/10019275?casa_token=fmLtbZfys2cAAAAA:UQvvJ3LWrATwWYtBQZ7HSAZigZdRe-k06Z9rOcKVc4c1LrrqXCe49E5IFgKRyC952n0Fmp_9UQ)]. 

Moreover, FP16 precision is starting to be enough for training. State of the art GPUs are also supporting FP8 and *integer* precision [[NVIDIA H100 Tensor Core GPU Architecture](https://resources.nvidia.com/en-us-tensor-core)]).

### Axiom 3
> NorthPole has a distributed, modular core array (16-by-16), with each core capable of massive parallelism (8192 2-bit operations per cycle) (Fig. 2F). Cortex-like modularity of the tiled core array enables homogeneous scalability in two dimensions and, perhaps, even in three dimensions and is also amenable to heterogeneous chiplet integration.

Here comes the fancy terminology.

When claiming 8192 operations per clock cycle, using INT2 operands, it might mean that NorthPole has 2048 multiply-and-accumulate (MAC) units that work on INT8 precision operands. These can be probably configured in single-instruction-multiple-data mode, *i.e.*, you can "glue" together 4 INT2 operands to form an INT8 word and work on these in parallel. Not bad but, also, nothing new: usually state-of-the-art (SotA) deep learning accelerators have 2048 MAC units per core.  

Regarding the cortex-like modularity, here I can only guess. Deep neural networks have been inspired by the cortex, since this is a multi-layer structure in which information is passed among layers. Each layer performs a certain function, like in deep convolutional neural networks: the first layers extract high level features (*e.g.*, edges), while the deep layers combine this information to get something useful out of it. 

What the heck does this have to do with hardware, you might ask? Well, since NorthPole hosts all the layers on chip, it is likely that the activation from one layer are passed to the next layer. Each layer has a set of MACs mapped to it, hence you have layers of MACs that exchange data: here's your cortex! 

One can do a more formal description of such architecture. In deep learning accelerators, one can distinguish among *overlay* and *layer-fuse* architectures [[Gilbert et al.](https://eems.mit.edu/wp-content/uploads/2023/07/2023_ispass_looptree.pdf)]. We will use a matrix multiplication example to wrap your head around it. 

{{< 
    figure 
    src="overlay.png" 
    width=600px 
    caption="An example of overlay architecture."
>}}

Suppose you want to multiply 3 matrices, A, B and C. Your goal is to maximize the number of processing engines (PEs) being used in your architecture at any time. A PE in nothing but a MAC unit with a bunch of registers and memories around. In an overlay accelerator, you would map as many PEs as possible to compute A\*B, store this partial matrix off-chip (or somewhere else with a fancier memory hierarchy), retrieve it later and use it to compute the final product (A\*B)\*C. Cool, but you need to store a (possibly) huge matrix off-chip and then retrieve it, which costs a lot of time and energy!


{{< 
    figure 
    src="layer-fused.png" 
    width=600px 
    caption="An example of layer-fuse architecture."
>}}

In layer-fuse architectures (also called *dataflow*, which also means another thing in hardware accelerators just to mess with you), instead, the PEs work on all the operands at once. The secret is that, when the operands are too big to fit on the available PEs, you perform part of the computations in an iteration, and the remaning part in another, as it is shown in the figure above for the matrix multiplication example.

### Axiom 4
> NorthPole distributes memory among cores (Figs. 1B and 2F) and, within a core, not only places memories near compute (2) but also intertwines critical compute with memory (Fig. 2, A and B). The nearness of memory and compute enables each core to exploit data locality for energy efficiency. NorthPole dedicates a large area to on-chip memory that is neither centralized nor organized in a tradi- tional memory hierarchy.

NorthPole is a spatial architecture, in contrast to GPUs and TPUs that are temporal architectures [[Sze et al.](https://ieeexplore.ieee.org/abstract/document/8114708)]: this means that instead of having a single on-chip memory that all the PEs share, (part of) memory is co-located with the processing units. 

{{< 
    figure 
    src="temporal-vs-spatial.png" 
    width=600px 
    caption="Spatial (left) and temporal (right) architectures."
>}}

Eyeriss [[Chen et al.](https://dspace.mit.edu/bitstream/handle/1721.1/101151/eyeriss_isscc_2016.pdf)] proposed this approach and taxonomy in 2016. Field programmable gate arrays (FPGAs) have been doing this since the beginning, with distributed SRAM near the login or the special purpose macros available on the silicon. I do not know if it is brain-inspired but it makes sense from a silicon perspective if you want to maximize efficiency.

### Axiom 5
> NorthPole uses two dense networks- on-chip (NoCs) (20) to interconnect the cores, unifying and integrating the distributed computation and memory (Fig. 2, C and D) that would otherwise be fragmented. These NoCs are inspired by long-distance white-matter and short-distance gray-matter pathways in the brain and by neuroanatomical topological maps found in cortical sensory and motor systems (21). One gray matter–inspired NoC enables spatial computing between adjacent cores (Fig. 3 and fig. S1). Another white matter–inspired NoC enables neuron activations to be spatially redistributed among all cores.

Take-home message is: PEs communicate using dedicated busses, in what is called a network-on-chip (NoC). There are two NoCs in NorthPole: one to exchange the intermediate results among PEs (the *gray* matter NoC), one for the inputs of the neural network (the *white* matter). 

### Axiom 6
> Another two NoCs enable reconfiguring synaptic weights and programs on each core for high-speed operation of compute units (Fig. 2, C and D). The brain’s organic biochemical substrate is suitable for supporting many slow analog neurons, where each neuron is hardwired to a fixed set of synaptic weights. Directly following this architectural construct leads to an inefficient use of inorganic silicon, which is suitable for fewer and faster digital neurons. Reconfigurability resolves this key dilem- ma by storing weights and programs just once in the distributed memory and reconfiguring the weights during the execution of each layer using one NoC and reconfiguring the programs before the start of the layer using another NoC. Stated differently, these two NoCs serve to substantially increase (up to 256 times, in some cases) the effective on-core memory sizes for weights and programs such that each core computes as if the weights and program for the entire network are stored on every core. Consequently, NorthPole achieves 3000 times more computation and 640 times larger network models than TrueNorth (14), although it has only four times more transistors (supplementary text S1).

My bad: two more NoCs, to load the weights to the PEs and the instructions to be performed (*i.e.*, the sequence of operations to be carried out). The comparison with TrueNorth is not really fair: completely different designs, completely different goals. Why should we count transistors in 2023, by the way?

### Axiom 7
> NorthPole exploits data-independent branching to support a fully pipelined, stall-free, deterministic control operation for high temporal utilization without any memory misses, which are a hallmark of the von Neumann architecture. Lack of memory misses eliminates the need for speculative, nondeterministic execution. Deterministic operation enables a set of eight threads for various compute, memory, and communication operations to be synchronized by construction and to operate at a high utilization.

This comes from the fact that NorthPole is running neural networks: if you know *exactly* which operations will be performed, with no branching in your program, and all the data is as close as possible to the PEs, and data movement is fully deterministic (*e.g.*, first I process the channel dimension, then the width, then the height etc.), I would be *very* worried if I had stalls or cache misses :) 

### Axiom 8
> Turning to algorithms and software, co-optimized training algorithms (fig. S3) enable state-of-the-art inference accuracy to be achieved by incorporating low-precision constraints into training. Judiciously selecting precision for each layer enables optimal use of on-chip resources without compromising inference accuracy (sup- plementary texts S9 and S10).

In short: IBM will provide a quantization aware training (QAT) toolchain with the NorthPole system.

### Axiom 9
> Codesigned software (fig. S3) automatically determines an explicit orchestration schedule for computation, memory, and communication to achieve high compute utilization in space and time while ensuring that there are no resource collisions. Network computation is broken down into layers, and each layer computation is spatially mapped to the core array. To minimize long-distance communication, spatial locality of neural activations is maintained across layers when possible. To prevent resource idling, core computation and intra and intercore data movement are temporally synchronized so that data and programs are present on each core before use. Together, algorithms and software constitute an end-to-end toolchain that exploits the full capabilities of the architecture while providing a path to migrate existing applications and workflows to the architecture.

So, they will provide a quantization aware training framework for DNNs, and they state that the dataflow of the accelerator is optimized to maximize efficiency. Eyeriss [[Chen et al.](https://dspace.mit.edu/bitstream/handle/1721.1/101151/eyeriss_isscc_2016.pdf)] strikes again.

### Axiom 10 
> NorthPole employs a usage model that consists of writing an input frame and reading an output frame (Figs. 1D and 3), which enables it to operate independently of the attached general processor (16). Once NorthPole is configured with network weights and an orchestration schedule, it is fully self-contained and computes all layers on-chip, requiring no off-chip data or weight movement. Thus, externally, the entire NorthPole chip appears as a form of active memory with only three commands (write input, run network, read output), with the minimum possible input-output bandwidth requirement; these features make NorthPole well suited for direct integration with high-bandwidth sensors, for embedding in computing and IT infrastructure, and for real-time, embedded control of complex systems (22).

In short, the network is hosted completely on the chip, and you pass the inputs via PCIe to the chip, wait some time and read back the result. On the point of energy efficiency: they write the following.

> [...] these features make NorthPole well suited for direct integration with high-bandwidth sensors, for embedding in computing and IT infrastructure, and for real-time, embedded control of complex systems (22).

Uhm, real-time embedded system. So it must be super efficient to be run on such a limited system, right? However, in Table 1, the power consumption required by running an INT8 version of the ResNet50 is 74 W. Ouch :)

## Silicon implementation
