This git is a documentation-only project.

I'm starting this as a "I" project, but just like my [Android GSI](https://github.com/TrebleDroid/treble_experimentations) I want it to become a community project.
Feel free to PR here if you want to add your own stuff, also GitHub has wiki pages enabled for this project, use it!

I do lots of experiments with a lot of NPUs with very different architectures, with very different capabilities.
Expect fully unmaintained vibe-coded changes. I might do human commits, but they'll likely be worse than the ones made by the LLM.

## Maybe useful projects

### Apple

- [bonsai-llama.cpp with Apple NPU prefill](https://github.com/phhusson/llama.cpp/blob/apple/bonsai-pq20/README-Apple.md) -- Bonsai 2 27B prefill upgrades from 60 tok/s to 115 tok/s on Apple M4 16GB
- [h3.c-ane with Metal co-work](https://github.com/phhusson/h3.c-ane/tree/dev/phh/mixed) -- h3.c does pure Metal inference, h3.c-ane does pure ANE inference... just merge them for faster inference! ~ 20% faster

### AMD

- [GGUF-quant kernels for AMD XDNA](https://github.com/phhusson/amd-npu-gguf)
- [llama.cpp with AMD NPU prefill](https://github.com/phhusson/llama.cpp/tree/amd/npu-worktree) -- Qwen 3.8-Flash-Next UD_IQ1_S prefill upgrade from to 300 tok/s on AMD Ryzen 8845HS
- sd.cpp with Vulkan/NPU co-work

### Qualcomm

- llama.cpp with more Hexagon kernels -- No split GPU/NPU prefill, no fences, just more quants in NPU

## Obsolete projects

- [Trying to understand Rockchip RK3588's NPU](https://github.com/phhusson/rknpu-reverse-engineering) -- This NPU now has a mainline Linux & Mesa driver *and* it has an acceptable TRM documenting it.

## Human contributions

In this section I'm listing what I think I contributed to those projects over just asking agents "speed go brrrr":

### Sharing model weights

For a number of reasons, it is usually not desirable to run token generation on NPU. [1]
So the idea is to do prompt processing on NPU, and token generation on GPU.

This requires two things:

- A mechanism is needed for that share. Under Linux, the standard mechanism is dma-buf.
- NPU and GPU need to be able to read the same model weight format [1]

[1] Notably, on Apple, the NPU seems to be too stupid for packed GGUF, so my agent decided to change the format in-memory, and had to change the GPU kernels to adapt.

### Row split

"Speed go brrr" + "share model weights with dma-buf" always leads to LLM doing the prompt processing 100% on NPU.
But hey, GPU already has decent computing power. We can add NPU and GPU's computes together, rather than just switching from one to the other!
That's easy to say for dense models. It's (much) harder for expert layers, my agents are still scratching their heads on those.
That gave +50%-100% performance.

### Compatible fences

In GPU, you can transparently chain, and append operations, without having to wait for each operation.
So you have already programmed the GPU to compute Layer 45, while the GPU is still computing Layer 2.
This is very useful, because if you waited Layer 1 to finish before starting Layer 2, because the GPU will sleep during the 500us you take to send the Layer 2 command.
If you insert NPU work at every layer, you'll make the GPU and CPU lose some precious time.
So basically you want to share the mechanisms the GPU uses to chain programs with the NPU.
On Linux, that's called a dma-fence.
My agent said it wasn't working and let it go, so I worked on fixing the Linux amdxdna driver (with another agent ofc).

That gave +10% performance.

I also requested the same from my agent on Apple NPU, and it did through the private ANE API.
(I have close to 0 knowledge of Apple world)


### NPU HW Architecture remarks

While agent is crunching its `/goal twice pp`, I read available NPU documentation, and I can find useful stuff. Some examples on AMD:

When agent mentioned that there was a 30% lock-contention probably due to DMA, I told it that it could program DMA directly from the NPU kernel. It unblocked a lot of experiments for the agents (notably for experts), though I think none were successful.

Early-on I mentioned the FIFO between neighbor NPU tiles, and it thought it was worth the experiment.

We discussed quickly about MemTiles, and it led it to share LUT across neighbors (so technically unrelated to MemTiles, more related to the neighbor FIFO).

## Additional comments

### For a number of reasons, it is usually not desirable to run token generation on NPU

Reminder: Prompt processing is bottlenecked by compute, Token generation is bottlenecked by RAM bandwidth.
A NPU is so fast at computation, that a non-optimal usage of NPU is still useful. In one experimentation, a NPU kernel had 30% compute usage, and that was enough to double pp.

When decoding, the bottleneck is RAM bandwidth. That means that you can't waste time with logic.
You must *always* have a DRAM-read in-flight. And doing computations during this read.
A NPU kernel that do DRAM-reads only 30% of the time gets only 30% of the token generation speed the hardware can do.
A CPU kernel that do DRAM-reads only 30% of the time can still reach 100%.

You've made a NPU dequant+matmul kernel that reaches 100% DMA usage? Congrats!
Now you're losing time (thus DRAM bandwidth) to synchronize with the CPU for all the ops you haven't implemented in the NPU.

So now you want to make NPU kernels that do a lot more ops to reduce synchronization time.
Well shit, not all NPUs are capable of anything. Rockchip RK3588's and Apple's ANE notably can't.

You've made a full NPU kernel that uses 100% of the RAM bandwidth. Fuck yeah, that's awesome!
Sorry to rain on your parade, but it's still possible NPU isn't the right solution. On my AMD 8845HS, the NPU can't use the whole RAM bandwidth (~ 65GB/s for 90GB/s on the system?).
The CPU can't use it 100% either. Only the GPU has the number of memory ports to the Infinity Fabric to saturate the RAM.

NPUs are very fun! They are very powerful! But you have to know what they /can't/ do

NPUs can still be useful to increase token generation, using speculative decoding.
But this is more complex. It will happen down the road, but agents will need to crunch much more work.

### NPU architectures

Just a very quick overview using words you might have to google, of the NPUs I've seen:

- Rockchip RK3588's NPU: fixed-pipeline matmul & conv
- Qualcomm Snapdragon NPU: one-core VLIW RISC
- AMD XDNA: 4x4 cores VLIW RISC with FIFO with neighbors (it's a real-life TIS100 <3)
- Apple ANE: It's completely opaque, no idea what that thing does. It seems much more limited than Qualcomm and AMD NPUs, but it looks too capable for a fixed-pipeline.

## My hardware

- Apple Mac Mini M4 16GB
- Minisforum UM880 Plus with 32GB + 24GB
- Lenovo T14s Gen 6 Snapdragon - 32GB RAM
- H96Max rk3588 TVBox with 8GB RAM

Donations I would appreciate:
- Ryzen AI 395+ 128GB
- A minipc with an Intel NPU (min 32GB RAM)
- I'm interested in other NPUs, but it must have at least 8GB of RAM. (There are some RAM-less M.2 NPU, like rpi ai hat+: no thanks)
- M.2 NVMe with CMB (modern marketting would call them AI SSD)
