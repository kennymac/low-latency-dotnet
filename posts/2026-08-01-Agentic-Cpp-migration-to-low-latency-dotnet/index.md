---
layout: post
title: "Low-latency .NET - Migrating a C++ Matching Engine to C# Using Agentic ATDD"
description: "A spike to port a C++ trading engine to zero-allocation .NET, with Native AOT benchmarking"
date: 2026-08-01
categories: [dotnet, low-latency, C++, HFT, Microsoft Coyote, Agentic-ATDD]
---

## A spike to port a C++ trading engine to zero-allocation .NET, with Native AOT benchmarking

### .NET in the modern HFT ecosystem

Over the past decade, the architecture of high-frequency trading systems has fundamentally changed. The sub-microsecond "tick-to-trade" path has moved from the CPU to the FPGA, and software remaining on the CPU receives data streams via kernel bypass (e.g., `Solarflare OpenOnload` / `DPDK`) to route network packets directly into user-space memory.

Behind this raw byte layer, front- and middle-office systems process millions of events per second, with confirmed transactions streaming en masse to post-trade and clearing services.

As regulatory mandates like SEC T+1 and CSDR push risk, confirmation and reporting pipelines from batch to real-time, non-negotiable delivery timelines present critical throughput and reliability challenges across the trade lifecycle.

### Enter low-latency .NET

While FPGAs and specialized C++ remain standard on the absolute *tick-to-trade* path, modern .NET is a compelling choice for the surrounding real-time workloads, delivering sub-microsecond, low-jitter throughput directly on pinned silicon.

#### Runtime and OS primitives

* **Low-Level Memory:** `Span<T>` and `System.IO.Pipelines` enable zero-allocation buffer slicing over user-space network ring buffers.
* **Native AOT:** Compiles C# directly to static machine code, removing runtime JIT warm-up spikes and reflection overhead.
* **OS Alignment:** Integrates seamlessly with Linux core isolation (`isolcpus` / `nohz_full`) and thread affinity to eliminate OS context-switching jitter.

#### Java ecosystem parity

C# offers Java-level enterprise integration, with superior memory ergonomics, e.g. -

* `Confluent.Kafka` wraps `librdkafka` - socket I/O and buffering execute in unmanaged memory, delivering near-native C throughput via async C# APIs.
* gRPC & Binary Framing (`Grpc.AspNetCore` / `Grpc.Net.Client`) maintained by Microsoft, leverage `System.IO.Pipelines` and `Span<T>` to decode binary Protobuf frames directly from sockets, with zero heap allocations.

### An agent-assisted C++ porting experiment

The experiment here is somewhat ideal, as we start with very clean C++ [source code](https://github.com/PacktPublishing/Building-Low-Latency-Applications-with-CPP) taken from Sourav Ghosh's excellent book *[Building Low Latency Applications with C++](https://amzn.to/3RMQZOO)*.

Ghosh's C++ architecture relies on custom memory pools, doubly-linked price level queues, and lock-free Single-Producer Single-Consumer (SPSC) ring buffers to process price-time priority (FIFO) order matching, without thread locks.

#### The translation strategy

The agent was asked to port the Ghosh codebase, step by step, and instructed to apply patterns of mechanical sympathy to the C# code generated -

1. **Cache line alignment and padding:** Padded sequences using `[StructLayout(LayoutKind.Explicit)]` to 128 bytes—providing native cache line alignment on ARM64 Apple Silicon (128B) and double-line padding on x86_64 Intel Xeon (64B) to neutralize L2 spatial/adjacent-line prefetching across cores and eliminate false sharing.
2. **Power-of-two masking:** Replaced modulo division (`% capacity`) with single-cycle bitwise `AND` masking (`sequence & mask`).
3. **Zero-allocation data payloads:** Replaced heap-allocated strings and objects with fixed-width `readonly struct` definitions.

```csharp
// Example: Cache-line padded lock-free sequence in C# (.NET 10)
[StructLayout(LayoutKind.Explicit, Size = 128)]
public struct PaddedSequence
{
    [FieldOffset(0)]
    public volatile long Value;
}
```

---

### Benchmarks

Using **BenchmarkDotNet** (`[MemoryDiagnoser]`), initially on an Intel Xeon W-3245 workstation running Native AOT, we measured the execution speed and memory allocations of the C# implementation:

| Benchmark method | Mean latency | Throughput (ops/sec) | Ratio vs SPSC | Allocated memory |
| --- | --- | --- | --- | --- |
| `SpscRingBuffer_PushAndPop` | 2.054 ns | 486,854,917 ops/sec | 1.00 (Baseline) | 0 B |
| `ConcurrentQueue_PushAndPop` | 12.252 ns | 81,619,327 ops/sec | 5.97x slower | 0 B |
| `SystemThreadingChannel_PushAndPop` | 45.225 ns | 22,111,663 ops/sec | 22.02x slower | 0 B |
| `DecodeAndEnqueueBinaryFrame` (Gateway) | 4.361 ns | 229,305,205 ops/sec | — | 0 B |
| `MatchOrderPair` *(Production Engine)* | 35.076 ns | 28,509,522 pairs/sec <br> *(57,019,044 orders/sec)* | — | 0 B |
| `MatchOrderPair` *(Idiomatic C#)* | 55.503 ns | 18,017,045 pairs/sec <br> *(36,034,090 orders/sec)* | 1.58x slower | 328 B |

#### Key takeaways:

* **2.054 nanoseconds:** The C# lock-free ring buffer executes a full push-and-pop in **~6 CPU clock cycles**, outperforming standard `ConcurrentQueue` by **6x** and `System.Threading.Channels` by **22x**.
* **35.08 nanoseconds:** The production-grade C# FIFO matching engine matches a complete Buy/Sell order pair in RAM at a rate of **28.51 million matched pairs per second** (**57.02 million orders/sec**).
* **0 bytes allocated:** Managed heap allocations across the entire hot path were exactly **0.00 B**, completely eliminating Garbage Collector pauses.
* **1.58x faster than idiomatic C#**: Executes 1.58x faster than standard object-oriented C# while eliminating **328 Bytes of heap allocations per matched pair**.


---

### Pairing with an agentic test engineer

In risk-critical environments like high-frequency trading, allowing an AI agent to write unvetted code on a production hot path is out of the question — a single race condition, off-by-one boundary bug, or unexpected heap allocation can trigger catastrophic failure.

*(We catch one such catastrophic error below!)*

Instead, high-reliability engineering teams adopt a **zero-trust model**: the human architect retains control over memory layout and system bounds, while the AI agent acts as a **risk-handling and testing engine**.

In this experiment, I applied this exact guardrail: I directed the agent to port Sourav Ghosh’s C++ source and generate test coverage step-by-step, but required it to pause for human review at every boundary. This allowed me to align coding style, enforce memory constraints, and ask clarifying questions as we went.

![An agentic code migration and test generation cycle](ATDD-Pryce-Freeman-Agentic-Spec.svg)
*Figure: An agentic code migration and test generation cycle.*

Without a step-by-step review gate, autonomous AI generation introduces a psychological disconnect—without being inside the TDD *red-green-refactor* loop, you lose the trust established by hands-on execution.

Enforcing strict human review gates alongside automated safety verification (asserting zero allocations in CI, property-based testing, and Microsoft Coyote concurrency exploration) converts AI from a risky code generator into a high-speed engine for formal rigour — even if, as we discover below, that rigour is somewhat 'eventually consistent'!

#### 💥 Stop press: Uncovering a catastrophic defect in the initial translation!

The necessity of this zero-trust model was proven during a post-migration stress audit. Sourav Ghosh’s reference C++ codebase was designed for a bounded benchmark scenario using direct 2D array indexing (`[client_id][client_order_id]`), which was mechanically sound for its intended $0 \dots 999$ scope. However, when the AI agent ported the code into C#, it collapsed the 2D array into a 1D flat array to reduce memory footprint—and introduced `clientOrderId % 1000` modulo indexing as a translation artifact.

While happy-path unit tests with bounded prices (`99`, `100`, `101`) passed completely (yielding a blistering 25 ns benchmark), real market price distributions (e.g. price `150` vs price `1150`) and sequential order IDs (`1` vs `1001`) caused 100% execution collision failure.

This illustrates the single most important lesson in agentic software engineering: **an AI agent applies knowledge currently in scope and will introduce subtle functional defects unless steered at all times by real-world production standards and zero-trust engineering rules.**

To remediate this:
1. **Updated Steering Rules**: Enforced zero-trust source translation, array indexing safety, and state-space property fuzzing in [`AGENTS.md`](https://github.com/kennymac/low-latency-csharp-exchange/blob/main/AGENTS.md) and [`.antigravity/rules.md`](https://github.com/kennymac/low-latency-csharp-exchange/blob/main/.antigravity/rules.md).
2. **Boundary & Property Test Suites**: Added FsCheck property-based tests and boundary limit condition test suites ([`OrderBookLimitConditionsTest.cs`](https://github.com/kennymac/low-latency-csharp-exchange/blob/main/LowLatency.ScratchPad.MatchingEngine.UnitTests/OrderBookLimitConditionsTest.cs)).
3. **Production Hardening**: Replaced naive modulo indexing with zero-allocation power-of-two open-addressing hash tables (`& mask` with linear probing).

Re-benchmarking the production-hardened engine measured **35.08 ns** mean latency (**57.0 Million orders/sec**) — retaining **0.00 B managed heap allocations** and executing **1.58x faster** than idiomatic C#, but now 100% defect-free and production-grade ([Documentation Addendum 12](https://github.com/kennymac/low-latency-csharp-exchange/blob/main/Documentation/12%20Bug%20Addendum%20Academic%20Spikes%20vs%20Production%20Exchange%20Defect%20Analysis.md)).

#### Establishing baseline agent rules - asking the agent to train itself

A first step was to ask *Google Gemini* *"Can we construct an automated, zero-allocation test harness, directly in C#, without relying on heavy external profiling tools?"*

The response outlined techniques that needed to be applied, and this became the [initial commit in the repo](https://github.com/kennymac/low-latency-csharp-exchange/blob/8e5ac25ba99ca944b56d8f173e18758339a34a2a/TestHarness/TestHarness.UnitTests/TestHarness/readme.md).

Here's an example unit test that asserts **zero bytes have been allocated on the current thread**. This becomes a zero-allocation safety gate - **eliminating a critical cause of fault in such a simple manner was a huge positive**:

```csharp
[Fact]
public void OrderExecution_MustBeZeroAllocation()
{
    // 1. Warm-up & clear transient setup memory
    var engine = new MatchingEngine();
    engine.ExecuteOrder(101, 150.50m, 100);
    GC.Collect();
    GC.WaitForPendingFinalizers();

    // 2. Capture baseline
    long bytesBefore = GC.GetAllocatedBytesForCurrentThread();

    // 3. Execute hot-path logic
    engine.ExecuteOrder(102, 150.55m, 200);

    // 4. Assert zero allocations
    long bytesAfter = GC.GetAllocatedBytesForCurrentThread();
    Assert.Equal(0, bytesAfter - bytesBefore);
}
```

The next step was to direct *Google Antigravity* to the local copy of the spike repo, and ask it to formulate its own rules, based upon the approach Gemini outlined. This became [`.antigravity/rules.md`](https://github.com/kennymac/low-latency-csharp-exchange/blob/main/.antigravity/rules.md).

I then pointed Antigravity at the C++ source code, and asked it to formulate a step by step plan to translate the repo into C#, working component by component, pausing at each step, so that generated code and tests could be reviewed. At each step, I'd ask Antigravity to explain techniques, adding notes to [`/Documentation`](https://github.com/kennymac/low-latency-csharp-exchange/tree/main/Documentation), run tests green and then commit!

---

### Step-by-step walkthrough

The migration was carried out incrementally across twelve structured steps—moving from core matching logic to lock-free buffers, binary network protocol gateways, benchmarking, Microsoft Coyote concurrency exploration, mutation testing, and production defect analysis.

* **Step 1: Core Matching Engine** — Ported the Ghosh `OrderBook` class to zero-allocation C#, implementing limit order placement, FIFO price/time priority matching, order cancellations, and partial/full fills [[view commit](https://github.com/kennymac/low-latency-csharp-exchange/commit/59daf99a8c7055b1a43db59d7d0b5d07b6b1e8ae) | [documentation](https://github.com/kennymac/low-latency-csharp-exchange/tree/main/Documentation)].
* **Step 2: Lock-Free SPSC Ring Buffer** — Built a single-producer single-consumer ring buffer using cache-line padding and power-of-two bitwise masking to avoid lock contention [[view commit](https://github.com/kennymac/low-latency-csharp-exchange/commit/18e504788c130b58a560ae46eac524883b63bc38)].
* **Step 3: Asynchronous Outbound Logging** — Added a background disk journaller (`LowLatencyLogger`) to isolate the matching core from disk and network I/O delays [[view commit](https://github.com/kennymac/low-latency-csharp-exchange/commit/ea63352b140b7e90c98b0615b6530820f5afd724)].
* **Step 4: Binary Client Gateway** — Implemented a 32-byte fixed binary packet protocol (`OrderServer`) with zero-copy `Span<byte>` deserialization [[view commit](https://github.com/kennymac/low-latency-csharp-exchange/commit/1deff0c4aba8319895ad2a7aec93409ed4691292)].
* **Step 5: Benchmark & Native AOT Suite** — Established a BenchmarkDotNet harness to continuously measure nanosecond latencies and enforce a `0.00 B` heap allocation policy [[view commit](https://github.com/kennymac/low-latency-csharp-exchange/commit/2746706da6874204bf9ebb960ea1e2e8237e9a04)].
* **Step 6: Multi-Hardware Profiling** — Benchmark-tested the compiled engine across Apple M3 Pro, Apple M1, and Intel Xeon hardware to verify microarchitectural scaling [[view commit](https://github.com/kennymac/low-latency-csharp-exchange/commit/586eb1ff63d3a5884388b6fd363797e284372b16)].
* **Step 7: Idiomatic C# Contrast** — Built a standard managed C# baseline (using `List<T>` and heap objects) to empirically prove how standard idiomatic code allocates memory compared to the zero-allocation engine [[view commit](https://github.com/kennymac/low-latency-csharp-exchange/commit/234dbb2dd055326fb738bbb90954af9860dcd0f3)].
* **Step 8: Microsoft Coyote Concurrency Exploration** — Integrated Microsoft Coyote for systematic multithreading exploration to catch inter-thread memory reordering race conditions [[view commit](https://github.com/kennymac/low-latency-csharp-exchange/commit/6425689354e2726f3316822a86701a73dc9ca254)].
* **Step 9: FsCheck Property-Based Testing** — Added property-based testing to verify queue FIFO ordering invariants under arbitrary sequence streams [[view commit](https://github.com/kennymac/low-latency-csharp-exchange/commit/6425689354e2726f3316822a86701a73dc9ca254)].
* **Step 10: Fault Injection Audit Setup** — Configured Stryker.NET for deliberate mutation testing of memory barriers and boundary checks [[view branch](https://github.com/kennymac/low-latency-csharp-exchange/tree/watchfails)].
* **Step 11: Systematic Mutation Testing Audit** — Completed full fault injection audit across memory barriers, capacity limits, and pool node recycling, achieving 100% mutation kill coverage [[documentation](https://github.com/kennymac/low-latency-csharp-exchange/blob/main/Documentation/11%20Systematic%20Mutation%20Testing%20and%20Fault%20Injection%20Audit.md)].
* **Step 12: Production Hardening & Defect Analysis** — Resolved academic modulo indexing collisions (`price % 1000`) by implementing zero-allocation power-of-two open-addressing hash tables (`& mask`), updating agent steering directives (`AGENTS.md`), and adding FsCheck property & boundary limit test suites [[documentation](https://github.com/kennymac/low-latency-csharp-exchange/blob/main/Documentation/12%20Bug%20Addendum%20Academic%20Spikes%20vs%20Production%20Exchange%20Defect%20Analysis.md)].


> Detailed technical deep-dives into hardware memory coherence, cache line alignment math, LMAX Disruptor mechanics, binary framing, mutation testing, and production defect analysis are available in the repository's [/Documentation](https://github.com/kennymac/low-latency-csharp-exchange/tree/main/Documentation) directory.


---


### Conclusion: From Academic Spike to Hardened Engine

Reflecting on the experience, the true power of agentic pairing isn’t that an AI will produce flawless, production-ready code on the first pass — as we discovered with the translation flaw, it won't.

Instead, the superpower lies in **velocity and verification**:
* **Initial Velocity**: In a single afternoon, we moved from a C++ codebase to a working C# port with Native AOT benchmarking, lock-free SPSC ring buffers, and zero-allocation assertions.
* **Iterative Rigour**: When static review uncovered the translation flaw, the agent allowed us to rapidly update steering rules (`AGENTS.md`), construct FsCheck property suites, and implement zero-allocation open-addressing hash maps in a fraction of the time it would take manually.

In not much time at all, we went from a raw C++ reference repo to a fully hardened, zero-allocation C# matching engine backed by property tests, Microsoft Coyote concurrency verification, and 100% test coverage.

#### Did I need to prep?

I prepped by refreshing my background knowledge of low-level concepts, listening to screen readers on my daily walk - and felt there might have been a few more areas I needed to revisit to see what developments have flown under my radar.

However, what was most interesting about the agent-assisted flow was that *I could ask questions to confirm my understanding was correct*, such as *"could you explain that concept in detail?"*, or *"aha, so the 128 padding aligns with the cache line size? What about other CPU architectures?"*

In this sense, the flow is very much like pair programming ("Do we need to add another test?", or "How on earth does that work?"), so I think the potential to learn by doing is the superpower of agent-assisted development - provided human oversight anchors the agent to real-world domain standards.


#### References and further reading

* Building Low Latency Applications with C++: Develop a complete low latency trading ecosystem from scratch using modern C++, Sourav Ghosh [Amazon](https://amzn.to/3RMQZOO) | [Packt](https://www.packtpub.com/en-gb/product/building-low-latency-applications-with-c-9781837634477)
* Pro .NET Memory Management: For Better Code, Performance, and Scalability, Konrad Kokosa, Christophe Nasarre, Kevin Gosse [Website](https://prodotnetmemory.com/) | [Amazon](https://amzn.to/3S86gK7)
* LMAX Disruptor Architecture & Mechanics: [LMAX Exchange Disruptor Technical Documentation](https://lmax-exchange.github.io/disruptor/)
* `Span<T>` Architecture & Mechanics: [MSDN: All About Span — Stephen Toub (2018)](https://learn.microsoft.com/en-us/archive/msdn-magazine/2018/january/csharp-all-about-span-exploring-a-new-net-mainstay)
* .NET Core 2.1 Foundations (`Span<T>` & `Pipelines`): [.NET Core 2.1 Performance Improvements](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-core-2-1/)
* Native AOT & Zero-JIT Latency (.NET 8): [.NET 8 Native AOT Benchmarks](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-8/#native-aot)
* Runtime & Memory Optimizations (.NET 9): [.NET 9 Performance Improvements](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-9/)
* Next-Gen Low-Latency Enhancements (.NET 10): [.NET 10 Performance Improvements](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-10/)
* Linux CPU Core Isolation Mechanics (`isolcpus` / `nohz_full`): [Linux Kernel Documentation: CPU Isolation](https://docs.kernel.org/admin-guide/cpu-isolation.html)
* Growing Object-Oriented Software, Guided by Tests, Steve Freeman and Nat Pryce, Addison-Wesley, 2009 [Website](https://growing-object-oriented-software.com/) | [Amazon](https://link.amazon/B06DnADaO)

---

***Kenneth McCormack** is a distributed systems architect and principal engineer, specializing in .NET, Continuous Delivery and zero-defect systems.* 

Reference code and benchmark harness: [github.com/kennymac/low-latency-csharp-exchange](https://github.com/kennymac/low-latency-csharp-exchange/)
