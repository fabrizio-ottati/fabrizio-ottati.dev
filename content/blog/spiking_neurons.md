---
title: "Spiking neurons: a digital hardware implementation"
date: 1970-01-01
description: "In this article, we will try to model a Leaky Spiking Neuron (LIF) using digital hardware: registers, memories, adders and so on."
math: true
draft: true
---

# Spiking neurons

In this article, we will try to model a Leaky Spiking Neuron (LIF) using digital hardware: registers, memories, adders and so on. To do so, we will consider a single output neuron connected to multiple input neurons.

![neurons-connected](/images/blog/spiking_neurons/neurons_connected.png)

In a Spiking Neural Network (SNN), neurons communicate by means of **spikes**: these activation voltages are then converted to currents through the **synapses**, charging the **membrane potential** of the destination neuron. In the following, the destination neuron is be denoted with the index $i$, while the input neuron under consideration is denoted with the index $j$. 

We denote the input spike train incoming from the $j$-neuron with $\sigma_j(t)$, while the synapse connect the $j$-neuron with the $i$-neuron is denoted with $w_{ij}$. All the incoming spike trains are then integrated by the $i$-neuron membrane; the integration function can be modeled with a first-order low-pass filter, denoted with $\alpha_i(t)$:

$$ \alpha_i(t) = \frac{1}{\tau_{u_i}} e^{-\frac{t}{\tau_{u_i}}}$$

The spike train incoming from the $j$-neuron, hence, is convolved with the membrane function; this correspond to the input currents charging the $i$-neuron membrane potential, $v_i(t)$. The total current in input to the $i$-neuron is denoted with $u_i(t)$ and modeled through the following equation:

$$ u_i(t) = \sum_{j \neq i}{w_{ij} \cdot (\alpha_v \ast \sigma_j)(t)} $$

Each $j$-neuron contributes with a current (spike train multiplied by the $w_{ij}$ synapse) and these sum up at the input of the $i$-neuron. Given the membrane potential of the destination neuron, denoted with $v_i(t)$, the differential equation describing its evolution is the following:

$$ \frac{\partial}{\partial t} v_i(t) = -\frac{1}{\tau_v} v_i(t) + u_i(t) - \theta \cdot \sigma_i(t)$$

In addition to the input currents, we have two terms: the **membrane reset**, due to the fact that, when a neuron **spikes**, its membrane potential goes back to the rest potential (usually equal to zero), and this is modeled by subtracting the threshold $\theta$ from the membrane potential $v_i(t)$; the **neuron leakage**, modeled with a **leakage factor** $\frac{1}{\tau_v}$ multiplied by the membrane potential.

# Discretizing the model

Such a differential equation cannot be solved directly using discrete arithmetics, like we would like to do with our digital hardware; hence, we need to **discretize** the equation. This discretization leads to the following result:

$$ v_i[t] = \beta \cdot v_i[t-1] + u_i[t] - \theta \cdot S_i[t] $$

$$ u_i[t] = \sum_{j \neq i}{w_{ij} \cdot S_j[t]} $$  

$$ S_i[t] = 1 ~\text{if}~ v_i[t] \geq \theta ~\text{else}~ 0 $$ 

# The neurons information: storage and addressing

How do we describe a neuron in hardware? First of all, we need to list the **information** associated to each $i$-neuron:
- its membrane potential $v_i[t]$.
- the weights  associated to its synapses, $w_{ij}$, that can be grouped in a vector $W_i$.

To store all the membrane potentials of a given layer of $M$ neurons, we can use a **memory array**, addressed by the position of the $i$-neuron in the layer.

![potentials-memory](/images/blog/spiking_neurons/membrane_potentials.png)

Given our layer of $M$ neurons, each of which is connected in input to $N$ synapses, we can think of grouping the $M \cdot N$ weights in a **matrix**, which can be associated to another memory array for its storage.

![synapses-memory](/images/blog/spiking_neurons/synapses_weights.png)

This memory has to be addressed with the input $j$-neuron and the destination $i$-neuron, in order to obtain the weight $w_{ij}$ in output. 

# Spike accumulation

Now we have our array of spiking neurons thanks to the memory arrays described above. We need to start to implement some functionalities! 

We start with the **membrane potential charging** of a generic neuron $i$-neuron. When the $j$-neuron spikes, its synapse weight $w_{ij}$ gets extracted from the synapse memory and multiplied by the spike. Since the spike is nothing but a single bit equal to 1, this is equivalen to using $w_{ij}$ itself as input current to the $i$-neuron. 

To accumulate this current, we need an adder! The equivalent digital circuit is shown in the following.

![accumulation](/images/blog/spiking_neurons/accumulation.png)

The membrane potential $v_i[t]$ is read from the potentials memory $V[t]$ and summed to the corrispondent synapse current $w_{ij}$, read from the synapses memory $W$. The resulting value is the next time step potential, $v_i[t+1]$, that will be written back to memory in the next clock cycle. The intermediate value is stored in a register. 

To prevent multiple read-write cycles, one can think of adding a loop to the register in order to accumulate all the currents for the time step $t$ and writing the final value $v_i[t+1]$ back to memory only once.

# Excitatory and inhibitory neurons

Now our neuron is able to accumulate spikes but we need to differentiate between different kinds of spikes! In fact, an input neuron can be **excitatory** (i.e. it **charges** the destination neuron membrane) or **inhibitory** (i.e. it **discharges** the destination neuron membrane). In our digital circuit, this phenomenon corresponds to **adding** or **subtracting**, respectively, the synapse current $w_{ij}$. We can add this functionality by transforming the adder in a configurable circuit, that either adds or subtracts its inputs depending on a control signal, that has to be generated by an FSM (Finite State Machine). 

![inhibitory](/images/blog/spiking_neurons/inhibitory.png)

This FSM, given the operation to be executed on the $i$-neuron, controls the adder properly to add or subtract the synapse current.

# Leakage 

Now we have to have to introduce that makes a LIF neuron different from an Integrate and Fire one (IF): the leakage! Being this proportional to the membrane potential, we should choose a (constant) leakage factor and multiply it by $v_i[t]$ to obtain $v_i[t+1]$.

However, multiplication is a costly operation in hardware; further more, the leakage factor is lower than one, so we should perform a fixed-point multiplication or, even worse, a division! How to solve this? Well, if the leakage factor $\beta$ is a power of $\frac{1}{2}$, such as $2^{-n}$, the multiplication becomes equivalent to a $n$-positions right shift! A really hardware friendly operation! 

Since the leakage factor is proportional to time, how can we do this in hardware? Well, if we imagine to write:

$$ \beta[t] = (2^{-n})^{\lfloor (t-t_{last})/T \rfloor} $$ 

where $T$ is the number of time steps needed to decay the membrane potential of a factor $2^{-n}$ and $t_{last}$ is the timestamp of the last membrane update operation. 

Now, the number of right shifts for a decay operation is equal to $n \cdot \lfloor \frac{t - t_{last}}{T} \rfloor$. Instead of calcultating this factor with an arithmetic circuit, we perform an $n$-positions right shift $\lfloor \frac{t - t_{last}}{T} \rfloor$ times!

At the same time, if the number of bits on which the membrane potential is encoded, we can think of using a barrel shifter to perform the leakage operation in a single clock cycle. 

![leak](/images/blog/spiking_neurons/leak.png)

# Spike mechanism 

Well, our neuron needs to spike! If this is encoded as a logic one, given a threshold $\theta$, we simply need to compare $v_i[t]$ to $\theta$ and generate a logic $1$ in output when the membrane potential is equal or larger than the threshold. This can be implemented using a comparator, as it is shown in the circuit. 

![spike](/images/blog/spiking_neurons/spike.png)

Of course, the membrane has to be reset when the neuron spikes, hence we need to subtract $\theta$ from $v_i[t]$ when the neuron fires. This can be done by driving the input multiplexer of the membrane register to provide $\theta$ in input to the membrane register and by performing a subtraction.

![reset](/images/blog/spiking_neurons/reset.png)

# Verilog implementation

# Conclusion

# Bibliography
