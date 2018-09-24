# GEM driver test

This is some messing around with the Cadence GEM driver for the gigabit networking used on the Zynq.

I have a Red Pitaya and am wanting to write a couple of PL peripherals, but before doing that I wanted to make sure I had a better grasp of the performance traps involved with building Linux drivers on this kind of system.

What concerned me upfront was that I noticed the default driver had trouble saturating a gigabit connection even when doing pretty basic things like iperf3, and if I didn't at least understand what was going on I was probably going to create exactly the same issues with anything I wrote. Especially since the Cadence team knows what it's doing, and has made this work for a ton of platforms.

Dma_map_single/dma_unmap_single is horrendously slow to try and sync L1/L2 with RAM on the Zynq (is it on all ARMs?). Slow enough that it makes sense to do a dma_alloc_coherent once and then just copy the data in. Everything screams that doing this extra copy is horrific and yet it manages to perform better (maybe at the cost of further hammering overall DRAM bandwidth). 

Was also trying to reduce the number of interrupts handled to see if that would help, but less difference than the DMA copy changes.

TCP Results for taskset -c 1 iperf3 -b 0 run from the Red Pitaya goes from 
Default: 778Mbit/sec Ubuntu or 523Mbit/sec WSL
Test driver: 938Mbit/sec Ubuntu or 900Mbit/sec WSL

UDP Results for taskset -c 1 iperf3 -b 0 -u run from the Red Pitaya goes from 
Default: 774Mbit/sec Ubuntu or 764Mbit/sec WSL
Test driver: 862Mbit/sec Ubuntu or 863Mbit/sec WSL

The other option that is vaguely referenced by Xilinx in their Tech Notes is implementing something that connects to the ACP to read and write cache-coherent memory, and proxying requests from the network card through that. A pain to have to write a PL side helper, but might be tried later.

Connecting and sending to a TCP socket created by a WSL app uses way more CPU time on the Red Pitaya compared to a TCP socket created by a native Linux app. Looks like it's due to a smaller window and the Windows machine generating far more ACKs.