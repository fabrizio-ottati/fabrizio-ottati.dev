---
title: "Neural inference at the frontier of energy, space and time - NorthPole, IBM"
description: "Translating to human language the new paper from IBM."
image: brain-to-chip.png
draft: false
date: 2023-11-19
type: "post"
tags: ["research", "hardware", "digital", "deep learning"]
showTableOfContents: true
---

[NorthPole](https://research.ibm.com/blog/northpole-ibm-ai-chip) is the new
shiny artificial intelligence (AI) accelerator developed by IBM. The creators
claim that NorthPole design has been driven by 10 axioms inspired by human
biology. We will use them as guidelines to analyze the paper.

The outline of this blog post is the same as the original article.

# Axiomatic design

> NorthPole, an architecture and a programming model for neural inference,
reimagines (Fig. 1) the interaction between compute and memory by embodying 10
interrelated, synergistic axioms that build on brain-inspired computing.

Fancy terminology :)

## Axiom 1

> Turning to architecture, NorthPole is specialized for neural inference.  For
example, it has no data-dependent conditional branching, and it does not support
training or scientific computation.

NorthPole is an application specific integrated circuit (ASIC): it can run only
deep neural networks (DNNs). Regarding "no data-dependent conditional
branching", it means that the "code" it is able to run has no conditional
branches: it is a pure sequence of instructions, to be executed one after the
other.  and pre-defined, as it should be when you are performing matrix
multiplications, _i.e._, running neural networks. There are no branches with
data-dependent `if`s.

This is extremely important, because engineers have spent a lot of time making
branches impact on performance minimum, especially in superscalar processors
that have to deal with any possible code being compiled for them. For more
information on this, the reader is referred to [Computer
architecture](https://books.google.it/books/about/Computer_Architecture.html?id=v3-1hVwHnHwC&redir_esc=y),
by Hennessy, Patterson and Asanovic. If the code being run does not contain
any of these conditional statements, there is no need for supporting them in 
hardware, which gives a lot of space to optimizations.

Moreover, NorthPole is an inference-only accelerator, _i.e._, you cannot train a
DNN on it, but only run it in inference. Training efficiency and speed is
important, but inference costs cannot be ignored: for instance, consider that at
the current moment each ChatGPT inference costs 0.04$ to OpenAI
[[Semianalysis](https://www.semianalysis.com/p/the-inference-cost-of-search-disruption)]
to run in the cloud, just taking into account the number of machines and the
electricity needed to power them and keep them cool.

Things change if you take into account the sparsity of the data inside a neural
network, _i.e._, when you take into account that layers in DNNs output a lot of
zeros; since multiplying by zero is useless, a smart strategy would be to skip
all the operations with zeros involved. To put this in code terms:

```python
def dot_prod(a: torch.Tensor, b: torch.Tensor) -> torch.Tensor:
    res = torch.Tensor([0])
    for i in range(len(a)):
        a_i, b_i = a[i], b[i] # Reading the inputs.
        res += a_i * b_i # When either a[i]==0 or b[i]==0, you are accumulating zeros,
                         # which is useless.
    return res

def dummy_sparse_dot_prod(a: torch.Tensor, b: torch.Tensor) -> torch.Tensor:
    res = torch.Tensor([0])
    for i in range(len(a)):
        a_i, b_i = a[i], b[i] # Reading the inputs.
        if a_i != 0 and b_i != 0:
            res += a_i * b_i # Now all the zeros are being skipped.
    return res
```

It seems easy, isn't it? Well, it is not. The `if` in `dummy_sparse_dot_prod` is
a mess. Why so?  Well, the problem is that when this code is running in
hardware, the line `a_i, b_i = a[i], b[i]` is much more costly (_i.e._, it takes
more _energy_ to execute it) than `a_i * b_i` 
[[Horowitz](https://ieeexplore.ieee.org/document/6757323)]! This is due to how 
we design our digital circuits to perform these operations. Hence, what you
would like to avoid is to _read_ the inputs, more than to multiply them! And if
to check that these are not zero you need to read them, well, you have lost the
game :)

Hence, when you want to skip zero-computations, you need to introduce a
structured approach (_i.e._, the zeros distribution in the matrices cannot be
random, but it has to follow a structure) if you want to improve performance
using sparsity [[Wu et al.](https://arxiv.org/abs/2305.12718)], which means
reading and processing only non-zero samples. For instance, Nvidia graphics
processing units (GPUs) support 2:4 sparsity, which means that every 4 elements
in a matrix, 2 are zeros (more or less, I am not being extremely precise on
this).

## Axiom 2

> Inspired by biological precision , NorthPole is optimized for 8, 4, and 2-bit
low-precision. This is sufficient to achieve state-of-the-art inference accuracy
on many neural networks while dispensing with the high-precision required for
training.

Well, we have been running DNNs using INT8 since ~2017 without claiming
biological inspiration. Recently, however, progress has been made and we can use
INT4 quantization with marginal loss in performance compared to the 32-bit
floating point (FP32) baseline that you trained on your GPU 
[[Keller et
al.](https://ieeexplore.ieee.org/abstract/document/10019275?casa_token=fmLtbZfys2cAAAAA:UQvvJ3LWrATwWYtBQZ7HSAZigZdRe-k06Z9rOcKVc4c1LrrqXCe49E5IFgKRyC952n0Fmp_9UQ)].

Moreover, FP16 precision is starting to be enough for training. State of the art
GPUs are also supporting FP8 and _integer_ precision [[NVIDIA H100 Tensor Core
GPU Architecture](https://resources.nvidia.com/en-us-tensor-core)]).

## Axiom 3

> NorthPole has a distributed, modular core array (16-by-16), with each core
capable of massive parallelism (8192 2-bit operations per cycle) (Fig. 2F).

When claiming 8192 operations per clock cycle, using INT2 operands, it 
means that NorthPole has 2048 multiply-and-accumulate (MAC) units that work on
INT8 precision operands. Why do we care about MACs?

DNNs are basically matrix multipliers: a neural network receives a vector in 
input and multiplies it by a matrix of weights, producing in output another 
vector. Let us consider a _naive_ matrix-vector multiplication.

```python
def naive_mat_vec_mul(v: torch.Tensor, m: torch.Tensor) -> torch.Tensor:
    m_rows, m_cols = m.shape
    assert m_cols == len(v)
    res = torch.zeros((m_rows,))
    for r in range(m_rows):
        for c in range(m_cols): 
            res[r] += v[c] * m[r, c] # Hey, this is multiplication and 
                                     # accumulation! Here's our MAC!
    return r
```

As you can see, the MAC is the fundamental operation employed in a matrix-vector
product. That's why we care about it.

These MACs can be probably configured in single-instruction-multiple-data (SIMD)
mode, _i.e._, you can "glue" together 4 INT2 operands to form an INT8 word and
work on these in parallel. I know this might sound strange if you have never
dealt with hardware, so I will try my best using fancy Python.

Let's start by defining our MAC unit. A MAC accepts three inputs: `a` and `b`,
that are the operands to be multiplied, and `c`, which is the operand to which
`a * b` is added to.

```python
class MAC:
    def __init__(self, SIMD_mode: str = "INT8") -> None:
        self.SIMD_mode = SIMD_mode
        return

    @property.setter
    def SIMD_mode(self, mode) -> None:
        assert mode in ("INT8", "INT4", "INT2")
        self.SIMD_mode = mode
        return

    def __call__(
        self, a: torch.Tensor, b: torch.Tensor, c: torch.Tensor
        ) -> torch.Tensor:
        if self.SIMD_mode == "INT8":
            # In this case, it means that a, b, and c contain a single value.
            res = torch.zeros(shape=(1,))
            res = c + a * b
        elif self.SIMD_mode == "INT4":
            # a, b and c contain 2 values to be elaborated _separately_.
            res = torch.zeros(shape=(2,))
            for i in range(2):
                res[i] = c[i] + a[i] * b[i]
        elif self.SIMD_mode == "INT2":
            # a, b and c contain 4 values to be elaborated _separately_.
            res = torch.zeros(shape=(4,))
            for i in range(4):
                res[i] = c[i] + a[i] * b[i]
        return res

```

Look at our shiny MAC unit: depending on how we configure it, _i.e._, to which
value we set its configuration parameter `SIMD_mode`, when performing a MAC, it
will work on a single triple of INT8 values, 2 triples of INT4 values or 4
triples of INT2 values.

Now, our NorthPole core will have 256 of these units.

```python
class NorthPoleCore:
    def __init__(self, MACs_cfg: str = "INT8") -> None:
        self.MACs = [MAC() for i in range(256)]
        assert cfg in ("INT8", "INT4", "INT2")
        self.MACs_cfg = cfg
        return

    def config_MACs(self, cfg) -> None:
        for i in range(256):
            self.MACs[i].SIMD_mode = cfg
        return

    def __call__(
        self, a: torch.Tensor, b: torch.Tensor, c: torch.Tensor
        ) -> torch.Tensor:
        if self.MACs_cfg == "INT8":
            assert a.shape == b.shape == c.shape == torch.Size([256])
        elif self.MACs_cfg == "INT4":
            assert a.shape == b.shape == c.shape == torch.Size([256, 2])
        elif self.MACs_cfg == "INT2":
            assert a.shape == b.shape == c.shape == torch.Size([256, 4])
        for i, mac in enumerate(self.MACs): # Imagine that all the iterations of this
                                            # loop are performed in parallel.
            c[i] = mac(a[i], b[i], c[i])
        return c
```

In the hardware implementation, the MACs work in parallel.

Not bad but, also, nothing new: usually state-of-the-art (SotA) deep learning
accelerators have 2048 MAC units per core.

> Cortex-like modularity of the tiled core array enables homogeneous scalability
in two dimensions and, perhaps, even in three dimensions and is also amenable to
heterogeneous chiplet integration.

Regarding the cortex-like modularity, here I can only guess. Deep neural
networks have been inspired by the cortex, since this is a multi-layer structure
in which information is passed among layers. Each layer performs a certain
function, like in deep convolutional neural networks: the first layers extract
high level features (_e.g._, edges), while the deep layers combine this
information to get something useful out of it.

What the heck does this have to do with hardware, you might ask? Well, since
NorthPole hosts all the layers on chip, it is likely that the activation from
one layer are passed to the next layer. Each layer has a set of cores mapped to
it, hence you have groups of cores that exchange data: here's your cortex!

One can do a more formal description of such architecture. In deep learning
accelerators, one can distinguish among _overlay_ and _layer-fuse_ architectures
[[Gilbert et
al.](https://eems.mit.edu/wp-content/uploads/2023/07/2023_ispass_looptree.pdf)].
We will use a matrix multiplication example to wrap your head around it.

{{<
figure
src="overlay.png"
width=600px
caption="An example of overlay architecture."
>}}

Suppose you want to multiply 3 matrices, A, B and C. Your goal is to maximize
the number of processing engines (PEs) being used in your architecture at any
time. A PE in nothing but a MAC unit with a bunch of registers and memories
around. In an overlay accelerator, you would map as many PEs as possible to
compute A\*B, store this partial matrix off-chip (or somewhere else with a
fancier memory hierarchy), retrieve it later and use it to compute the final
product (A\*B)\*C. Cool, but you need to store a (possibly) huge matrix off-chip
and then retrieve it, which costs a lot of time and energy!

{{<
figure
src="layer-fused.png"
width=600px
caption="An example of layer-fuse architecture."
>}}

In layer-fuse architectures (also called _dataflow_, which also means another
thing in hardware accelerators just to mess with you), instead, the PEs work on
all the operands at once. The secret is that, when the operands are too big to
fit on the available PEs, you perform part of the computations in an iteration,
and the remaning part in another, as it is shown in the figure above for the
matrix multiplication example.

## Axiom 4

> NorthPole distributes memory among cores (Figs. 1B and 2F) and, within a core,
not only places memories near compute (2) but also intertwines critical compute
with memory (Fig. 2, A and B). The nearness of memory and compute enables each
core to exploit data locality for energy efficiency. NorthPole dedicates a large
area to on-chip memory that is neither centralized nor organized in a tradi-
tional memory hierarchy.

NorthPole is a _spatial_ architecture, in contrast to GPUs and TPUs that are
_temporal_ architectures [[Sze et
al.](https://ieeexplore.ieee.org/abstract/document/8114708)]: this means that
instead of having a single on-chip memory that all the PEs share, (part of)
memory is co-located with the processing units.

{{<
figure
src="temporal-vs-spatial.png"
width=600px
caption="Spatial (left) and temporal (right) architectures."
>}}

Eyeriss [[Chen et al.](https://dspace.mit.edu/bitstream/handle/1721.1/101151/eyeriss_isscc_2016.pdf)]
proposed this approach and taxonomy in 2016. Field programmable gate arrays
(FPGAs) have been doing this since the beginning, with distributed SRAM near the
logic or the special purpose macros available on the silicon. I do not know if
it is brain-inspired but it makes sense from a silicon perspective if you want
to maximize efficiency.

## Axiom 5

> NorthPole uses two dense networks- on-chip (NoCs) (20) to interconnect the
cores, unifying and integrating the distributed computation and memory (Fig. 2,
C and D) that would otherwise be fragmented. These NoCs are inspired by
long-distance white-matter and short-distance gray-matter pathways in the brain
and by neuroanatomical topological maps found in cortical sensory and motor
systems (21). One gray matter–inspired NoC enables spatial computing between
adjacent cores (Fig. 3 and fig. S1). Another white matter–inspired NoC enables
neuron activations to be spatially redistributed among all cores.

Take-home message: PEs communicate using dedicated busses, in what is called a
network-on-chip (NoC). There are two NoCs in NorthPole: one to exchange the
intermediate results among PEs (the _gray_ matter NoC), one for the inputs of
the neural network (the _white_ matter).

## Axiom 6

> Another two NoCs enable reconfiguring synaptic weights and programs on each
core for high-speed operation of compute units (Fig. 2, C and D). The brain’s
organic biochemical substrate is suitable for supporting many slow analog
neurons, where each neuron is hardwired to a fixed set of synaptic weights.
Directly following this architectural construct leads to an inefficient use of
inorganic silicon, which is suitable for fewer and faster digital neurons.
Reconfigurability resolves this key dilem- ma by storing weights and programs
just once in the distributed memory and reconfiguring the weights during the
execution of each layer using one NoC and reconfiguring the programs before the
start of the layer using another NoC. Stated differently, these two NoCs serve
to substantially increase (up to 256 times, in some cases) the effective on-core
memory sizes for weights and programs such that each core computes as if the
weights and program for the entire network are stored on every core.
Consequently, NorthPole achieves 3000 times more computation and 640 times
larger network models than TrueNorth (14), although it has only four times more
transistors (supplementary text S1).

My bad: two more NoCs, to load the weights to the PEs and the instructions to be
performed (_i.e._, the sequence of operations to be carried out). The comparison
with TrueNorth is not really fair: completely different designs, completely
different goals.

## Axiom 7

> NorthPole exploits data-independent branching to support a fully pipelined,
stall-free, deterministic control operation for high temporal utilization
without any memory misses, which are a hallmark of the von Neumann architecture.
Lack of memory misses eliminates the need for speculative, nondeterministic
execution. Deterministic operation enables a set of eight threads for various
compute, memory, and communication operations to be synchronized by construction
and to operate at a high utilization.

This comes from the fact that NorthPole is running neural networks: if you know
_exactly_ which operations will be performed, with no branching in your program,
and all the data is as close as possible to the PEs, and data movement is fully
deterministic (_e.g._, first I process the channel dimension, then the width,
then the height etc.), I would be _very_ worried if I had stalls or cache misses
:)

## Axiom 8

> Turning to algorithms and software, co-optimized training algorithms (fig. S3)
enable state-of-the-art inference accuracy to be achieved by incorporating
low-precision constraints into training. Judiciously selecting precision for
each layer enables optimal use of on-chip resources without compromising
inference accuracy (sup- plementary texts S9 and S10).

In short: IBM will provide a quantization aware training (QAT) toolchain with
the NorthPole system. QAT takes starts, usually, from a full precision FP32 
model and converts all the weights and activations to integers, in order to 
reduce their precision. This leads to information loss that worsens the accuracy
of the network: to recover this, the DNN is trained for few more epochs to use
backprop to tune the network taking into account the approximations brought by
the quantization process.

## Axiom 9

> Codesigned software (fig. S3) automatically determines an explicit
orchestration schedule for computation, memory, and communication to achieve
high compute utilization in space and time while ensuring that there are no
resource collisions. Network computation is broken down into layers, and each
layer computation is spatially mapped to the core array. To minimize
long-distance communication, spatial locality of neural activations is
maintained across layers when possible. To prevent resource idling, core
computation and intra and intercore data movement are temporally synchronized so
that data and programs are present on each core before use. Together, algorithms
and software constitute an end-to-end toolchain that exploits the full
capabilities of the architecture while providing a path to migrate existing
applications and workflows to the architecture.

So, they will provide a quantization aware training framework for DNNs, and they
state that the dataflow of the accelerator is optimized to maximize efficiency.
Eyeriss [[Chen et
al.](https://dspace.mit.edu/bitstream/handle/1721.1/101151/eyeriss_isscc_2016.pdf)]
strikes again.

## Axiom 10

> NorthPole employs a usage model that consists of writing an input frame and
reading an output frame (Figs. 1D and 3), which enables it to operate
independently of the attached general processor (16). Once NorthPole is
configured with network weights and an orchestration schedule, it is fully
self-contained and computes all layers on-chip, requiring no off-chip data or
weight movement. Thus, externally, the entire NorthPole chip appears as a form
of active memory with only three commands (write input, run network, read
output), with the minimum possible input-output bandwidth requirement; these
features make NorthPole well suited for direct integration with high-bandwidth
sensors, for embedding in computing and IT infrastructure, and for real-time,
embedded control of complex systems (22).

In short, the network is hosted completely on the chip, and you pass the inputs
via PCIe to the chip, wait some time and read back the result. On the point of
energy efficiency: they write the following.

> [...] these features make NorthPole well suited for direct integration with
high-bandwidth sensors, for embedding in computing and IT infrastructure, and
for real-time, embedded control of complex systems (22).

Uhm, real-time embedded system. So it must be super efficient to be run on such
a limited system, right? However, in Table 1 of the paper, the power consumption
required to run an INT8 version of ResNet50 is 74 W. Ouch :)

# Silicon implementation

> NorthPole has been fabricated in a 12-nm pro- cess and has 22 billion
transistors in an 800-mm2 area, 256 cores, 2048 (4096 and 8192) operations per
core per cycle at 8-bit (at 4- and 2-bit, respectively) precision, 224 megabytes
of on-chip memory (768 kilobytes per core, 32-megabyte framebuffer for
input-output), more than 4096 wires crossing each core both horizontally and
vertically, and 2048 threads.

Each core can carry out 2048 INT8 operations in parallel per iteration, which
means that are 2048 PEs per core. Pay attention to the on-chip memory
capability: 768 kB per core! That is _a lot_ of memory. To understand how much,
you can consider a neural network in which each parameter (for simplicity,
weights only) is stored as INT8, occupying 1 B in memory. This means that a
network with 768 k parameters can be hosted on a single core (forgive me, it is
not fully precise as I am considering only the weights).

# Energy, space and time

> For methodological rigor that ensures a fair and level comparison of various
implementations, it is critical that all evaluation metrics be independent of
the details of the implementations, which can vary arbitrarily across
architectures at the discretion of the designers. The architecture-independent
goodness metric adopted here is that all implementations must be measured at
state-of-the-art inference accuracy.

This means that they will consider the data from other papers in which they used
the data format (for NorthPole, either INT8, INT4 or INT2) that gives the
highest accuracy on the network being run

> The architecture-independent cost metrics adopted here are now introduced.
Turning first to energy, > different integrated- circuit (IC) implementations
have different throughputs > [in frames per second (FPS)] at different power
consumptions (in watts). Therefore, FPS per watt > (equals frames per joule) is
a widely used energy metric for comparing ICs.

To measure efficiency on image classification tasks, the author consider how
many inferences you can run on the accelerator using only one joule of energy.
Here, I will consider only two GPUs from Nvidia and NorthPole, because that is
the most interesting comparison. The DNN being used is a ResNet50, and it is
benchmarked on ImageNet. They also consider other "exotic" figures of merit,
such as "frames per second per watt per billion of transistor", which I do not
consider to be meaningful.

| Accelerator | Power [W] | Throughput (FPS) | Efficiency [inferences / J] |   Data format   | Only DNNs? | Training? |
|:-----------:|:---------:|:----------------:|:---------------------------:|:---------------:|:----------:|:---------:|
| Nvidia A100 |    400    |      30,814      |              80             |  FP32,16; INT8  |      ✘     |     ✔     |
| Nvidia H100 |    700    |    **81,292**    |             116             | FP32,16; INT8,4 |      ✘     |     ✔     |
|  NorthPole  |   **74**  |      42,460      |           **571**           |     INT8,4,1    |      ✔     |     ✘     |

Northpole seems to be winning! And by a lot! This is mostly due to the fact that A100 and H100 are
incredibly powerful devices (look at the power consumption!) and you can use
them to run _any_ model you want, also for training in FP32 and FP16. NortPole,
instead, is meant only for inference of quantized (_i.e._, INT8 parameters)
neural networks.

Moreover, the GPUs being considered use large amount of off-chip (hence,
energy-hungry) memory to deal with any kind of DL workload being run on them.
Hence, most of the inefficiency is due to
the large energy consumption associated to retrieving data from external DRAM.
In fact, the authors write:

> Even relative to the H100 GPU, which is implemented using a 4 nm silicon
process node, NorthPole delivers five times more frames per joule. NorthPole has
high energy efficiency because of less data movement, which is enabled by a lack
of off-chip memory, the intertwining of on-chip memory with compute, the
locality of spatial computing, and a better use of low-precision computation.

I am not so sure about

> [...] and a better use of low-precision computation.

because I suspect they are comparing themselves to the FP32 execution on H100. 
Moreover:

> All compared architectures implement considerably faster clock rates than
NorthPole, and yet NorthPole outperforms on the space metric by achieving
substantially higher instantaneous parallelism (through high utilization of many
highly parallel compute units specialized for neural inference) and
substantially lower transistor count owing to low precision (Fig. 4B).

NorthPole runs at 400 MHz while an A100 GPU can run up to 1.4 GHz. Of 
course, GPUs have much more "redundant" hardware to be programmable for more 
tasks. Hence, is it fair this comparison with general purpose hardware? Sure.
However, shall we call in a 
[fairer competitor](https://ieeexplore.ieee.org/abstract/document/10019275)? :)

|  Accelerator  | Power [W] | Throughput (FPS) | Efficiency [inferences / J] |   Data format   | Only DNNs? | Training? |
|:-------------:|:---------:|:----------------:|:---------------------------:|:---------------:|:----------:|:---------:|
|  Nvidia A100  |    400    |      30,814      |              80             |  FP32,16; INT8  |      ✘     |     ✔     |
|  Nvidia H100  |    700    |    **81,292**    |             116             | FP32,16; INT8,4 |      ✘     |     ✔     |
|   NorthPole   |   **74**  |      42,460      |             571             |     INT8,4,1    |      ✔     |     ✘     |
| Keller et al. |    N/A    |        212       |           **4714**          |      INT8,4     |      ✔     |     ✘     |

The competitor (Keller et al.) is a chip developed by Nvidia and published in
2022 at ISSCC. The data are taken from the JSSC journal version of the paper. A
_big_ disclaimer: the measurements come from a tapeout of Nvidia, _i.e._, they
have designed a new accelerator and produced a test chip, and then they have run
a neural network on it and measured performance. It is not (yet) a plug-in
accelerator like NorthPole. This explains why the throughput is so low and the
efficiency so high.

The Nvidia accelerator is meant for inference only, just like NorthPole, and it
uses a very fancy quantization technique 
[[Dai et al.](https://proceedings.mlsys.org/paper_files/paper/2021/file/48a6431f04545e11919887748ec5cb52-Paper.pdf)]
to use INT4 precision without compromising accuracy. Moreover, the accelerator
is designed to run large Transformers on it, but I have used their ResNet50 data
for fair comparison with NorthPole.

Another reason for which Keller et al. is much more efficient than NorthPole is
that it supports _sparsity-aware processing_, _i.e._, it skips zero computations
without reading the zero values (I am simplifying).

# (My) conclusions

In conclusion, the following statement

>  Inspired by the brain, which has no off-chip memory, NorthPole is optimized
for on-chip networks [...]

will make me sleep better tonight, knowing that I do not have a DDR4 stick 
on top of my head. Sigh.

NorthPole is an interesting experiment: it is an extremely large accelerator, 
with a distributed memory hierarchy that allows extreme efficiency when 
parallelizable workloads, such as DNNs, are targeted. Regarding the brain 
inspiration, I do not agree at all. This is an excellent engineering work, 
that takes into account key factors:
* reduced precision operations are much more efficient that high-precision ones.
An FP32 multiplication costs _much_ more than an INT8 one.
* DNNs are extremely robust to quantization, and with INT8 precision there is 
basically no accuracy degradation.
* memory accesses are much more costly than computations when going up the 
memory hierarchy (_i.e._, external DRAMs and so on).

I can see clusters of NorthPole being stacked in servers to improve inference
efficiency (see the OpenAI case). It is not an edge computing solution. I wish
there where more technical details in the paper, since it very divulgative. I 
would expect a presentation like this at some commercial conference where new
products are advertised. For sure, we would have got a different paper if they 
chose an IEEE journal instead of Science, where hardware is not really common.

# Bibliography

* [_Neural inference at the frontier of energy, space, and time_](https://www.science.org/doi/10.1126/science.adh1174), Dharmendra S. Modha et al., Science, 2023.
* [_HighLight: Efficient and Flexible DNN Acceleration with Hierarchical Structured Sparsity_](https://arxiv.org/abs/2305.12718), Yannan Nellie Wu et al., IEEE Micro, 2023.
* [_A 95.6-TOPS/W Deep Learning Inference Accelerator With Per-Vector Scaled 4-bit Quantization in 5 nm_](https://ieeexplore.ieee.org/abstract/document/10019275?casa_token=fmLtbZfys2cAAAAA:UQvvJ3LWrATwWYtBQZ7HSAZigZdRe-k06Z9rOcKVc4c1LrrqXCe49E5IFgKRyC952n0Fmp_9UQ), Ben Keller et al., IEEE Journal of Solid State Circuits (JSSC), 2023.
*  [_Computing's energy problem (and what we can do about it)_](https://ieeexplore.ieee.org/document/6757323), Mark Horowitz, IEEE International Solid-State Circuits Conference (ISSCC), 2014.
* [_NVIDIA H100 Tensor Core GPU Architecture_](https://resources.nvidia.com/en-us-tensor-core), Nvidia, 2023.
* [_LoopTree: Enabling Exploration of Fused-layer Dataflow Accelerators_](https://eems.mit.edu/wp-content/uploads/2023/07/2023_ispass_looptree.pdf), Michael Gilbert et al., IEEE International Symposium on Performance Analysis of Systems and Software (ISPASS), 2023.
* [_Efficient Processing of Deep Neural Networks: A Tutorial and Survey_](https://ieeexplore.ieee.org/abstract/document/8114708), Vivienne Sze et al., IEEE Proceedings, 2017.
* [_VS-Quant: Per-Vector Scaled Quantization for Accurate Low-Precision Neural Network Inference_](https://proceedings.mlsys.org/paper_files/paper/2021/file/48a6431f04545e11919887748ec5cb52-Paper.pdf), Steve Dai et al, Machine Learning and Systems (MLSys), 2021.
