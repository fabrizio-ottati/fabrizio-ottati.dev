---
title: "List of open source tools for digital hardware design"
date: 2023-01-01
description: "This is my list of tools that I find useful to work on digital hardware design on my local machine."
draft: false
---

These are the tools that I use on my local machine for digital hardware design. Click on the titles to be redirected to the sources.

# [Verilator](https://www.veripool.org/verilator/)

![verilator](/images/blog/open_source_hw_tools/verilator.png)

Verilator is a tool that compiles Verilog and SystemVerilog sources to highly optimized (and optionally multithreaded) cycle-accurate C++ or SystemC code. The converted modules can be instantiated and used in a C++ or a SystemC testbench, for verification and/or modelling purposes.

To start with Verilator, I *strongly* suggest to take a look at [Norbert Kremeris blog](https://www.itsembedded.com/dhd/verilator/).

# [OpenLane](https://openlane.readthedocs.io/en/latest/)

OpenLane is an automated RTL to GDSII flow based on several components including OpenROAD, Yosys, Magic, Netgen, CVC, SPEF-Extractor, KLayout and a number of custom scripts for design exploration and optimization. It also provides a number of custom scripts for design exploration, optimization and ECO.

The flow performs all ASIC implementation steps from RTL all the way down to GDSII. Currently, it supports both A and B variants of the sky130 PDK, but support for the newly released GF180MCU PDK is in the works, and instructions to add support for other (including proprietary) PDKs are documented.

OpenLane abstracts the underlying open source utilities, and allows users to configure all their behavior with just a single configuration file.

# [OpenRAM](https://openram.org/)

![openram](/images/blog/open_source_hw_tools/OpenRAM.svg)

OpenRAM is an award winning open-source Python framework to create the layout, netlists, timing and power models, placement and routing models, and other views necessary to use SRAMs in ASIC design. OpenRAM supports integration in both commercial and open-source flows with both predictive and fabricable technologies.

# Bibliography

- [Norbert Kremeris blog](https://www.itsembedded.com/).
- [OpenLane documentation](https://openlane.readthedocs.io/en/latest/getting_started/).
- [OpenRAM documentation](https://openram.org/).
