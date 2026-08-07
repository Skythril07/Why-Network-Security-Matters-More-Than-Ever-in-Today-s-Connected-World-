# GPU architecture and SIMD parallelism

**Self-Learning Article — Computer Architecture**
Jash Patel | PRN 125BTCS1008 | Roll No. FY_CS_38 | FY B.Tech Computer Science Engineering, SAKEC

---

## Introduction

Run the same hash-cracking workload twice, once on a modern desktop processor and once on the graphics card sitting in the same machine, and the graphics card finishes so much faster that the first result looks like a measurement error. The obvious explanation is that the GPU has thousands of cores and the CPU has sixteen.

That answer is wrong, and it stays wrong in an interesting way. A "CUDA core" is not a smaller version of a CPU core. It cannot fetch its own instruction, it has no branch predictor of its own, and on its own it is close to useless. The speed comes from a completely different architectural bet about what a processor is for, and understanding that bet explains both why GPUs dominate certain workloads and why they are hopeless at others. This article covers the design principles behind that bet, how SIMD and SIMT execution actually work, where the model breaks down, and which real workloads depend on it.

## Two different bets

A CPU core is built to finish one instruction stream as quickly as possible. Most of its transistor budget goes not to arithmetic but to control: out-of-order scheduling, register renaming, speculative execution, branch prediction, and several megabytes of cache. All of that machinery exists to keep a single thread from stalling. The actual adders and multipliers are a small fraction of the die.

A GPU makes the opposite bet. It assumes you have not one instruction stream but millions of independent pieces of work, and it optimises for total throughput rather than the completion time of any one of them. Control logic gets cut to the bone and the space goes to arithmetic units and register files instead. The design does not try to avoid memory latency at all. It hides it: when one group of threads stalls waiting on memory, the scheduler swaps in another group that is ready, and with enough resident threads the arithmetic units never sit idle.

The numbers make the divergence concrete. An RTX 4090 carries 16,384 CUDA cores and about 1 TB/s of memory bandwidth. A high-end desktop CPU of the same era has sixteen cores and roughly 100 GB/s from dual-channel DDR5. Ten times the bandwidth, a thousand times the lanes — but only if the work can be arranged to feed them.

## SIMD, and the twist that SIMT adds

Flynn's taxonomy classifies architectures by how many instruction and data streams they handle. A classical scalar processor is SISD, one instruction acting on one datum. SIMD is single instruction, multiple data: one instruction applied simultaneously to a whole vector of operands.

CPUs have had SIMD for decades through vector extensions. SSE gave 128-bit registers, AVX2 widened them to 256 bits, AVX-512 to 512. An AVX2 add operates on eight 32-bit floats in one instruction, and the burden of exploiting this sits with the compiler or with the programmer writing intrinsics by hand. Auto-vectorisation is fragile; a pointer the compiler cannot prove is non-aliasing is often enough to make it give up.

GPUs use the same underlying idea with a friendlier interface, which NVIDIA calls SIMT — single instruction, multiple threads. You write ordinary scalar code describing what one thread does. The hardware then bundles threads into groups of 32, called a warp, and issues one instruction to the whole warp per cycle. AMD's equivalent grouping is the wavefront, historically 64 wide and configurable to 32 in RDNA. The programmer sees independent threads; the silicon sees a vector.

Physically, those lanes are organised into streaming multiprocessors. An SM holds a set of arithmetic units, a register file, some fast local memory, and a few warp schedulers whose only job is to pick, every cycle, which resident warp has an instruction ready to issue. A single SM can keep dozens of warps in flight simultaneously, all with their registers live at once, which is why the register file on a GPU is larger than its L2 cache — an inversion that makes no sense on a CPU and perfect sense here. Launching a kernel with a few hundred threads leaves most of this idle. The hardware expects tens of thousands.

That abstraction is convenient and slightly dishonest, and the dishonesty is where performance goes to die.

## Divergence: the cost of the lie

Since all 32 threads in a warp share one program counter, they must execute the same instruction. So what happens at an `if` statement where half the threads take the true branch?

The hardware executes both paths, one after the other, masking off the lanes that should not participate in each. A warp where every thread agrees costs one pass. A warp split by `if (threadIdx.x % 2 == 0)` costs two passes with half the lanes idle throughout, so throughput halves. Nested divergent branches compound this.

This reframes the usual claim that GPUs are bad at branchy code. Branches are perfectly legal and often free — a branch that every thread in a warp resolves the same way costs nothing extra. The penalty is specifically for *divergence within a warp*, which is why experienced kernel authors restructure data so that threads with similar control flow land in the same warp. Sorting work by branch outcome before launching a kernel can be worth more than any arithmetic optimisation.

## Memory is usually the real limit

Newcomers optimise arithmetic. Experienced people optimise memory access, because most kernels are bandwidth-bound rather than compute-bound.

The key mechanism is coalescing. When the 32 threads of a warp read 32 consecutive 4-byte words, the memory controller services them with a small number of wide transactions. When those same threads read addresses scattered across memory, the hardware must issue up to 32 separate transactions to fetch the same 128 bytes of useful data. Identical arithmetic, an order of magnitude difference in time. This is why traversing a matrix along rows versus columns changes GPU performance so dramatically, and it is the same locality principle that governs CPU caches, just with harsher consequences.

The on-chip hierarchy supports this. Each streaming multiprocessor has an enormous register file, tens of kilobytes of shared memory that the programmer controls explicitly as a scratchpad, and an L1 cache; behind them sit a shared L2 and the VRAM. Shared memory is the interesting one, because unlike a CPU cache it is not automatic. You decide what lives there. A standard tiled matrix multiplication loads a block of each input into shared memory once and then reuses it many times, raising arithmetic intensity — the ratio of computation to bytes moved — until the kernel becomes compute-bound instead of starved.

Occupancy ties it together. Each SM has a fixed pool of registers and shared memory, so a kernel that uses a lot of either allows fewer warps to be resident at once. Fewer resident warps means less work available to hide memory latency, and the whole latency-hiding strategy weakens. Cutting register usage can speed up a kernel that does exactly the same arithmetic.

## The bus nobody plans for

A discrete GPU lives at the far end of a PCIe link, and data has to get there. PCIe 4.0 x16 offers about 32 GB/s in theory and rather less in practice, so moving a gigabyte each way costs tens of milliseconds. If the kernel itself runs in five milliseconds, the transfer dominates and the CPU would have been faster.

This overhead is why GPU acceleration pays off on large batches and long-running work, and loses on small, frequent operations. It also explains the industry's push toward unified memory: Apple silicon puts CPU and GPU on the same memory pool, and NVIDIA's Grace Hopper packages a CPU with the GPU over a much wider interconnect, both attacking the same bottleneck.

## Where this shows up

Rasterising graphics was the original workload and remains a perfect fit, since shading two million pixels involves two million independent computations of the same shader program.

Deep learning inherited the hardware almost by accident. Training a neural network is dominated by matrix multiplication, which is regular, dense, and reuses data heavily. Modern GPUs now include tensor cores, fixed-function units that perform small matrix multiply-accumulate operations at reduced precision, a specialisation that only makes sense because one operation dominates the workload so completely.

Security tooling shows the same pattern. Password cracking with hashcat evaluates millions of independent candidate guesses against the same hash function, with no dependencies between them, which is the ideal shape for SIMT execution. The more instructive half is the defence. Memory-hard key derivation functions — bcrypt, scrypt, Argon2 — are deliberately designed to require a large working buffer per guess. A GPU has thousands of lanes but only a small amount of fast memory per lane, so forcing each guess to touch hundreds of kilobytes destroys the parallel advantage. That is a cryptographic defence built by reasoning about cache sizes and register files rather than about mathematics.

Scientific simulation was an early adopter for the same structural reason. Finite-difference and finite-element solvers update a grid where each cell's new value depends on its neighbours' old values, so every cell can be computed independently within a timestep. Weather models, fluid dynamics and molecular dynamics all reduce to that pattern, and the stencil access it produces is exactly the coalesced, high-reuse memory behaviour the hardware was built around.

One caveat on the video side: hardware transcoding on a GPU usually runs on a dedicated ASIC block such as NVENC, not on the shader cores. It is a fixed-function accelerator that happens to share the card, not an example of SIMD at work.

## What GPUs cannot fix

Amdahl's law sets the ceiling. If 95% of a program parallelises perfectly and 5% is inherently serial, infinite parallel hardware still caps the speedup at 20x. The serial fraction, not the core count, decides the outcome.

Beyond that, some computations simply have the wrong shape. Traversing a linked list is a chain of dependent loads where each address depends on the previous one, and no width helps. Recursion with unpredictable depth maps badly, and so does anything that needs threads to talk to each other constantly rather than work in isolation. There is also a real engineering cost: a CUDA kernel with explicit memory management takes far longer to write and debug than the loop it replaces, and that time only pays back if the workload is genuinely large.

## Why it matters

Understanding this material changes the question one asks about performance. It stops being "how fast is this processor" and becomes "does the shape of this computation match the shape of this machine". A GPU is a machine for wide, regular, independent work with high arithmetic intensity, and it punishes everything else.

That framing generalises well beyond graphics cards. Cache blocking, vectorising an inner loop, choosing a batch size, deciding whether an offloaded operation is worth the transfer — these are all the same question. The bcrypt case is the sharpest illustration of why it matters, because it shows architectural knowledge applied in the opposite direction: not to make a computation faster, but to make one deliberately impossible to accelerate.
