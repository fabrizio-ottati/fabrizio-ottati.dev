---
title: "Installing Vitis on Fedora 36"
date: 2023-01-09
description: "Troubleshooting Vitis installation on a Fedora machine, officially unsupported."
draft: false
type: "post"
tags: ["vitis", "hardware", "vidado", "fedora", "hls"]
---

Fedora 36 is not officially supported by Xilinx (or AMD, I should say). Hence, I had to use some workarounds to install it on my work machine.

# Downloading the web installer

First of all, you need to download the [Vitis Linux web installer](https://www.xilinx.com/member/forms/download/xef.html?filename=Xilinx_Unified_2022.2_1014_8888_Lin64.bin), available at [this page](https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/vitis.html). I chose the **Vitis Core Development Kit - 2022.2** package, and saved it in my `~/Downloads` directory.

# Running the installer

First of all, we make the installer executable:

```bash
fab@fedora:~ $ cd ~/Downloads
fab@fedora:Downloads $ chmod u+x ./Xilinx_Unified_2022.2_1014_8888_Lin64.bin
```

Then, we follow the instructions in [this post](https://support.xilinx.com/s/question/0D52E00007Evd2XSAR/vivado-xsetup-jvm-crash-for-full-installer?language=en_US) (search for `vlntmrx` answer). We extract the installer in a folder called `vitis` using the following command:

```bash
fab@fedora:Downloads $ ./Xilinx_Unified_2022.2_1014_8888_Lin64.bin --target vitis
```

The installer will run and crash, but a folder `vitis` will be there with all the stuff that we need:

```bash
fab@fedora:Downloads $ cd vitis
fab@fedora:vitis $ ls
bin  data  hs_err_pid811776.log  lib  tps  xsetup
```

Now, we execute the following commands: 

```bash
# Removing the harzbuff library that is causing the crash
fab@fedora:vitis $ rm ./tps/lnx64/jre11.0.11_9/lib/libharfbuzz.so
# Creating a symlink to my system harbuzz library.
fab@fedora:vitis $ cp -s /lib64/libharfbuzz.so.0 ./tps/lnx64/jre11.0.11_9/lib/libharfbuzz.so
# Running the installer using xsetup.
fab@fedora:vitis $ ./xsetup
```

The Xilinx GUI installer should start now. I installed the tool in my home directory, under `~/.local/`.


