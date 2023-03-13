---
title: "Hardware Dive-In | TrueNorth"
description: "Analysis of the TrueNorth chip and article."
image: /images/blog/dive-in-truenorth/brain-to-chip.png
draft: true
date: 2023-03-12
type: "post"
tags: ["research", "hardware", "digital", "neuromorphic"]
showTableOfContents: true
---

![The beautiful TrueNorth article image](/images/blog/dive-in-truenorth/brain-to-chip.png) 

## Introduction

The TrueNorth design has been driven by seven principles:
* the architecture is a **purely event-driven one**, being Globally Asyncrhonous Locally Synchronous (GALS), with the being **completely asynchronous** inteconnection fabric among the cores, with **synchronous** cores. 
* the CMOS process employed is a **low-power** one, with the goal of minimising **static power**.
* since the brain is a **massively parallel** architecture, employing 100 billion neurons each of which has a fanout of approximately 10 thousand synapses, parallelism is a key feature of TrueNorth: a chip employes **1 million neurons** and **256 million synapses**.
* the authors claim **real-time** operation, which translates to a global time synchronisations of **1ms**, i.e. the neurons are updated and spike every millisecond.
* the architecture is **scalable**: multiple cores can be put together, and the local synchronicity of the cores prevents the problems related to **global clock signal skew** in modern digital circuits.
* **redundancy** is employed in the design, especially in the **memory circuits**, to make the chip **tolerant to defects**.
* the chip operation corresponds perfectly to the software operation, when using the IBM TrueNorth application design software.

Designing an asynchronous circuit is a very difficult task, since no VLSI EDAs are available for this kind of design; hence, the TrueNorth designers have decided to use conventional EDAs for the synchronous cores design and **custom design tools and flows** for the asynchronous interconnection fabric. 

## Architecture 
