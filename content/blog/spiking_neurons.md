---
title: "Spiking neurons: a digital hardware implementation"
date: 2022-12-27
description: "In this article, we will try to model a Leaky Spiking Neuron (LIF) using digital hardware: registers, memories, adders and so on."
cover: "/images/blog/spiking_neurons/neurons-connected.png"
math: true
draft: false
---

# Spiking neurons

In this article, we will try to model a Leaky Integrate and Fire (LIF) spiking neuron using digital hardware: registers, memories, adders and so on. To do so, we will consider a single output neuron connected to multiple input neurons.

![neurons-connected](/images/blog/spiking_neurons/neurons-connected.png)

In a Spiking Neural Network (SNN), neurons communicate by means of **spikes**: these activation voltages are then converted to currents through the **synapses**, charging the **membrane potential** of the destination neuron. In the following, the destination neuron is be denoted with the index $i$, while the input neuron under consideration is denoted with the index $j$. 

We denote the input spike train incoming from the $j$-neuron with $\sigma_{j}(t)$:
$$ \sigma_{j}(t) = \sum_{k} \delta(t-t_{k}) $$
where $t_{k}$ are the spike timestamps of the spike train $\sigma_{j}(t)$. 

The synapse connecting the $j$-neuron with the $i$-neuron is denoted with $w_{ij}$. All the incoming spike trains are then **integrated** by the $i$-neuron membrane; the integration function can be modeled by a **first-order low-pass filter**, denoted with $\alpha_{i}(t)$:
$$ \alpha_{i}(t) = \frac{1}{\tau_{u_{i}}} e^{-\frac{t}{\tau_{u_{i}}}}$$
The spike train incoming from the $j$-neuron, hence, is convolved with the membrane function; in real neurons, this corresponds to the **input currents** coming from the $
j$-neurons that **charge** the $i$-neuron membrane potential, $v_{i}(t)$. The sum of the currents in input to the $i$-neuron is denoted with $u_{i}(t)$ and modeled through the following equation:
$$ u_{i}(t) = \sum_{j \neq i}{w_{ij} \cdot (\alpha_{v} \ast \sigma_{j})(t)} $$
Each $j$-neuron contributes with a current (spike train multiplied by the $w_{ij}$ synapse) and these sum up at the input of the $i$-neuron. Given the membrane potential of the destination neuron, denoted with $v_{i}(t)$, the differential equation describing its evolution  through time is the following:
$$ \frac{\partial}{\partial t} v_{i}(t) = -\frac{1}{\tau_{v}} v_{i}(t) + u_{i}(t) - \theta \cdot \sigma_{i}(t)$$
In addition to the input currents, we have two terms: 
- the **membrane reset**, $\theta \cdot \sigma_{i}(t)$, due to the fact that, when a neuron **spikes**, its membrane potential goes back to the rest potential (usually equal to zero), and this is modeled by **subtracting the threshold** $\theta$ from the membrane potential $v_{i}(t)$ when an output spike occurs.
- the **neuron leakage**, $\frac{1}{\tau_{v}} v_{i}(t)$, modeled through a **leakage coefficient** $\frac{1}{\tau_{v}}$ that multiplies the membrane potential.

# Discretizing the model

Such a differential equation cannot be solved directly using discrete arithmetics, like we would like to do with our digital hardware; hence, we need to **discretize** the equation. This discretization leads to the following result:
$$ v_{i}[t] = \beta \cdot v_{i}[t-1] + u_{i}[t] - \theta \cdot S_{i}[t] $$
The input current is given by:
$$ u_{i}[t] = \sum_{j \neq i}{w_{ij} \cdot S_{j}[t]} $$  
We have introduced a new function, $S_{i}[t]$, that is equal to 1 at spike time (i.e. if at timestamp $t$ the membrane potential $v_{i}[t]$ is larger than or equal to the threshold $\theta$) and 0 elsewhere:
$$ S_{i}[t] = 1 ~\text{if}~ v_{i}[t] \geq \theta ~\text{else}~ 0 $$ 

# The neurons information: storage and addressing

To get started, we need to define the network **fan-in**, i.e. how many $j$-neurons are connected to input of each $i$-neuron; we denote this number with $N$. Then, we suppose to have $M$ neurons in total in our network.

How do we describe a neuron in hardware? First of all, we need to list some basic information associated to each $i$-neuron:
- its **membrane potential** $v_{i}[t]$.
- the **weights  associated to the synapses**, $w_{ij}$; since each $i$-neuron is connected in input to $N$ neurons, these synapses can be grouped in an $N$-entries vector $W_{i}$.

Since there are $M$ neurons in the array, we need an $M$-entries vector to store all the membrane potentials, denoted with $V[t]$, meaning that the potentials stored in it are those "sampled" at timestamp $t$. This vector can be associated to a **memory array** in our hardware architecture.

![potentials-memory](/images/blog/spiking_neurons/membrane-potentials.png)

To each neuron, an **address** is associated, which can be thought as the $i$ index in the $V[t]$ vector; to obtain $v_{i}[t]$, we use the $i$-neuron address to index the membrane potentials memory, also denoted with $V[t]$.

We are now able to store and retrieve an $i$-neuron membrane potential through a memory: we need to do something with it! In particular, we would like to **charge it with some currents**. To do that, we need to get the corresponding synapses $W_{i}$, **multiply** these by the spikes of the associated input neurons, sum them up and, then, accumulate them in the $i$-neuron membrane. 

Let su proceed step by step, starting from a single input $j$-neuron: 
$$ u_{ij}[t] = w_{ij} \cdot S_{j}[t] $$
We know that $S_{j}[t]$ is either 1 or 0; hence, we have either $u_{ij}[t] = w_{ij}$ or $u_{ij}[t] = 0$; this means that the synapse weight is **either added or not**. What does this mean for us? It means that we read the $w_{ij}$ synapse from memory only if the $j$-neuron connected to the $i$-neuron spikes! Given our array of $M$ neurons, each of which is connected in input to $N$ synapses, we can think of grouping the $M \cdot N$ weights in a **matrix**, which can be associated to another memory array for its storage, that we denote with $W$.

![synapses-memory](/images/blog/spiking_neurons/synapses-weights.png)

This memory has to be addressed with the input $j$-neuron and the destination $i$-neuron, in order to obtain the weight $w_{ij}$ in output, which automatically corresponds to the $u_{ij}[t]$ current being accumulated in the $i$-neuron membrane when the $j$-neuron spikes. 

# Spikes accumulation

We have now all the information about our array of spiking neurons thanks to the memory arrays described above; let us start to implement some neural functionalities! 

We start with the **membrane potential charging** of a generic neuron $i$-neuron. When the $j$-neuron spikes, its synapse weight $w_{ij}$ gets extracted from the synapse memory $W$ and multiplied by the spike. Since the spike is nothing but a single bit equal to 1, this is equivalent to using $w_{ij}$ itself as input current to the $i$-neuron. To accumulate this current, we need to use a **digital adder**!

![accumulation](/images/blog/spiking_neurons/accumulation.png)

The membrane potential $v_{i}[t]$ is read from the potentials memory $V[t]$ and added to the corrispondent synapse current $w_{ij}$; the result is the membrane potential  of the next time step, $v_{i}[t+1]$, that will be written back to memory in the next clock cycle. The intermediate value is stored in a **register**, denoted as "membrane register" from now on.

To **prevent multiple read-write cycles**, one can think of adding a loop to the register in order to **accumulate all the currents** of the $j$-neurons that are spiking at timestep $t$ and writing the final value $v_{i}[t+1]$ back to memory only once.

# Excitatory and inhibitory neurons

Now our neuron is able to accumulate spikes but we need to differentiate between different kinds of spikes! In fact, an input neuron can be **excitatory** (i.e. it **charges** the destination neuron membrane) or **inhibitory** (i.e. it **discharges** the destination neuron membrane). In our digital circuit, this phenomenon corresponds to **adding** or **subtracting**, respectively, the synapse weight $w_{ij}$. We can add this functionality by transforming the adder in a configurable circuit, that either **adds or subtracts its inputs depending on a control signal**, that has to be generated by an **FSM (Finite State Machine)**. 

![inhibitory](/images/blog/spiking_neurons/inhibitory.png)

This FSM, given the operation to be executed on the $i$-neuron, controls the adder properly to add or subtract the synapse current. However, does this design make sense?

Inhibitory and excitatory neurons are chosen at **chip programming time**. This means that the neuron kind does not change during operation (however, we'll see that even if it does, it is not a problem for the solution we are about to propose). Hence, we can **embed this information** in the neuron description, **adding a bit to the synapse weight** that, depending on its value, denotes that neuron as excitatory or inhibitory.

![synapse-encoding](/images/blog/spiking_neurons/synapse-encoding.png)

Given $n$ bits of quantization for the synapse, we add a bit denoted with $e$ that identifies the neuron type: if it is **excitatory**, $e=1$ and the weights is **added**; is it is **inhibitory**, $e=0$ and the weight is **subtracted**. In this way, the $e$ field of the synapse can be used to drive directly the adder. 

![modified-adder](/images/blog/spiking_neurons/modified-adder.png)

# Leakage 

Now we have to have to introduce the characteristic feature of the LIF neuron: the **leakage**! Being this proportional to the membrane potential, we shall choose a (constant) leakage factor $\beta$ and multiply it by $v_{i}[t]$ to obtain $v_{i}[t+1]$.
$$ v_{i}[t+1] = \beta \cdot v_{i}[t] $$ 
However, multiplication is an **expensive** operation in hardware; furthermore, the leakage factor is smaller than one, so we would need to perform a **fixed-point multiplication** or, even worse, a **division**! How to solve this?

Well, if the leakage factor $\beta$ is a power of $\frac{1}{2}$, such as $2^{-n}$, the multiplication becomes **equivalent to a $n$-positions right shift**! A really **hardware-friendly** operation! 

![leak](/images/blog/spiking_neurons/leak.png)

# Spike mechanism 

Well, our neuron needs to spike! If this is encoded as a logic one, given a threshold $\theta$, we simply need to **compare $v_{i}[t]$ to $\theta$** and generate a logic 1 in output **when the membrane potential is equal or larger than the threshold**. This can be implemented using a **comparator**. 

![spike](/images/blog/spiking_neurons/spike.png)

The output of the comparator is used directly as spike bit.

The membrane has to be **reset** when the neuron spikes; hence, we need to subtract $\theta$ from $v_{i}[t]$ when the neuron fires. This can be done by driving the input multiplexer of the membrane register to provide $\theta$ in input to the adder and perform a subtraction.

![reset](/images/blog/spiking_neurons/reset.png)

However, do we need all this hardware? We can be smarter than this:
- by choosing $\theta = 2^m-1$, where $m$ is the bitwidth of the membrane register and the adder, having $v_{i}[t] \geq \theta$ is **equivalent to having an overflow in the addition**. Hence, the comparation result is equal to the **overflow flag** of the adder, and it can be provided as spike bit.
- instead of subtracting $\theta$ from the membrane register, we can save an operation by simply **resetting** $v_{i}[t]$ to 0 when a spike occurs; this is equivalent to using the oveflow flag of the adder as **reset signal for the membrane register**. This should not be done in an actual implementation: at least a register should be used to prevent glitches in the adder circuit from resetting the membrane register when it should not be.

The resulting circuit is the following.

![smart-reset](/images/blog/spiking_neurons/smart-reset.png)

Way simpler!

# Conclusion

Here we are, with a first prototype of our LIF array digital circuit. In the next episodes, I plan to make it actually work (we'll see) and provide a Verilog implementation. Of course, we will use only open source tools :) 

# Bibliography

- [*Loihi: A Neuromorphic Manycore Processor with On-Chip Learning*](https://redwood.berkeley.edu/wp-content/uploads/2021/08/Davies2018.pdf), Mike Davies et al., 2018.
- [*Training Spiking Neural Networks Using Lessons From Deep Learning*](https://arxiv.org/abs/2109.12894), Jason Eshraghian et al., 2022.
- [*Bottom-Up and Top-Down Neuromorphic Processor Design: Unveiling Roads to Embedded Cognition*](https://dial.uclouvain.be/pr/boreal/object/boreal%3A226494/datastream/PDF_01/view), Charlotte Frenkel, 2020
