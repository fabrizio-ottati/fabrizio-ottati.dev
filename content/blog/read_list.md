---
title: "My neuromophic hardware read list"
description: "Here's a list of articles and theses related to digital hardware for neuromorphic applications."
date: 2022-12-25
draft: true
---

### [*A 0.086-mm2 12.7-pJ/SOP 64k-Synapse 256-Neuron Online-Learning Digital Spiking Neuromorphic Processor in 28nm CMOS*](https://arxiv.org/abs/1804.07858), Charlotte Frenkel et al., 2018.

In this paper, a digital neuromorphic processor employing the LIF and Izhikevic neuron models is presented. The Verilog is also [open source](https://github.com/ChFrenkel/ODIN)! 

Online-learning capabilities are allowed with an hardware-efficient implementation of the Spike-Driven Sinaptic Plasticity (SDSP) rule.

### [*Loihi: A Neuromorphic Manycore Processor with On-Chip Learning*](https://redwood.berkeley.edu/wp-content/uploads/2021/08/Davies2018.pdf), Mike Davies et al., 2018.

Probably the most popular neuromorphic processor right now. What distinguishes it from the other ones is the **completely asynchronous** design: cores and routing network are completely clock-less! 

### [*MorphIC: A 65-nm 738k-Synapse/mm2 Quad-Core Binary-Weight Digital Neuromorphic Processor With Stochastic Spike-Driven Online Learning*](https://arxiv.org/abs/1904.08513), Charlotte Frenkel et al., 2019.

This is the quad-core version of Odin.

### [*Bottom-Up and Top-Down Neuromorphic Processor Design: Unveiling Roads to Embedded Cognition*](https://dial.uclouvain.be/pr/boreal/object/boreal%3A226494/datastream/PDF_01/view), Charlotte Frenkel, 2020.

This is Charlotte Frenkel's thesis. In this, all her learning process is accurately documented, with more details on ODIN and MorphIC and other projects.

### [*μBrain: An Event-Driven and Fully Synthesizable Architecture for Spiking Neural Networks*](https://www.frontiersin.org/articles/10.3389/fnins.2021.false664208/full), Jan Stuijt et al., 2021.

This is another asynchronous architecture, which is programmable at synthesis time.

### [*Training Spiking Neural Networks Using Lessons From Deep Learning*](https://arxiv.org/abs/2109.12894), Jason Eshraghian et al., 2022.

### [*Spiking Neural Network Integrated Circuits: A Review of Trends and Future Directions*](https://arxiv.org/abs/2203.07006), Arindam Basu et al., 2022.

Nice survey paper that compares different ICs, both digital and mixed-signal ones. 

### [*ReckOn: A 28nm Sub-mm2 Task-Agnostic Spiking Recurrent Neural Network Processor Enabling On-Chip Learning over Second-Long Timescales*](https://arxiv.org/abs/2208.09759), Charlotte Frenkel and Giacomo Indiveri, 2022.

[open source](https://github.com/ChFrenkel/ReckOn)!
### [*THOR -- A Neuromorphic Processor with 7.29G TSOP2/mm2Js Energy-Throughput Efficiency*](https://arxiv.org/abs/2212.01696), Mayank Senapati et al., 2022.

This is what I call "Odin on steroids" :) Only the LIF hardware is left at neuron level, and the routing is completely upgraded.
