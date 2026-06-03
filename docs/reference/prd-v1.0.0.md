Here is a detailed Product Requirement Document (PRD) for the **AeroRing** project, incorporating the CUE (Coroutines + io_uring + eBPF) architecture designed as a pure C low-level core for maximum performance, multi-language bindings, and future extensibility.

---

# Product Requirement Document (PRD)

## Project Name: AeroRing

**Document Version:** 1.0.0

**Target Architecture:** Linux (Kernel 6.0+)

**Implementation Language:** Pure C (with targeted Assembly for context switching)

---

## 1. Executive Summary & Problem Statement

### 1.1 Context

Modern high-performance network applications (e.g., API gateways, reverse proxies, edge security scrubbers, and CDN nodes) face unprecedented concurrency and throughput demands. Traditional asynchronous architectures on Linux primarily rely on `epoll` combined with application-level thread pools (e.g., `epoll` + `read`/`write` syscalls).

### 1.2 Problem Statement

Traditional architectures introduce substantial bottlenecks at extreme scales:

1. **High Syscall Overhead:** Frequent context switches between user space and kernel space due to non-stop I/O readiness notifications (`epoll_wait`) and I/O execution (`recv`/`send`).
2. **Expensive Memory Copies:** Data must be shuttled from network hardware/disk buffers into kernel memory, then copied into user-space buffers, and vice versa.
3. **Wasted CPU Cycles on Malicious Traffic:** High-frequency malicious requests (e.g., DDoS, SQL injections) are brought all the way up to user space to be parsed and dropped by application code, wasting immense CPU computation.
4. **Integration Overhead:** Modern language runtimes (Node.js, Python, Go) use complex abstractions or incompatible C++ ABIs, making the deployment of unified, universal low-level network engines brittle and inefficient.

### 1.3 Solution: AeroRing

`AeroRing` is a zero-dependency, ultra-high-performance asynchronous network and file engine implemented in pure C. It establishes a "CUE" paradigm—integrating **C Coroutines**, Linux **`io_uring`**, and **eBPF**—into a unified, lockless loop. By operating as a pure C engine, it exposes a completely clean, stable C ABI, enabling friction-free FFI (Foreign Function Interface) bindings for all major high-level programming languages.

---

## 2. Product Objectives & Target Audience

### 2.1 Objectives

* **Sub-Microsecond Latency:** Maximize pure memory-speed processing by reducing application-to-kernel friction.
* **Near-Zero Syscall Overhead:** Achieve a completely non-blocking, syscall-less event loop under load using kernel-side submission polling.
* **True Zero-Copy Data Path:** Keep packet and file byte streams confined within kernel-mapped DMA regions from ingestion to egress.
* **Universal Extensibility:** Provide a universal data-plane substrate that any high-level language runtime can embed with zero performance degradation.

### 2.2 Target Audience

* Infrastructure Architects building next-generation Service Meshes, Edge Gateways, or high-throughput WAFs.
* Cloud-Native Developers seeking underlying building blocks for distributed storage or zero-copy message queues.
* Language Runtime Engineers requiring an optimized, native asynchronous network library for scripting or managed languages (e.g., Python `asyncio`, Node.js Addons).

---

## 3. High-Level System Architecture

`AeroRing` functions as an invisible, aerodynamic highway for data. The user space issues instructions, while the kernel executes and filters them in-place.

### 3.1 Components

1. **AeroRing Pure C Core (User Space):** * Manages highly optimized, custom stackful coroutines via highly compact assembly context switching routines.
* Manages the lifecycle of the shared memory rings (`io_uring`).


2. **`io_uring` Interface (Kernel/User Boundary):**
* Utilizes dual ring buffers (Submission Queue / Completion Queue) mapped into user-space memory.
* Enables `SQPOLL` to allow a dedicated kernel thread to actively consume network and file requests asynchronously.


3. **eBPF In-Kernel Coprocessor (Kernel Space):**
* Ingests network packets via XDP (eXpress Data Path) or Traffic Control (TC) hooks.
* Modifies or validates data streams and triggers linked execution states within `io_uring` before notifying user-space.



---

## 4. Detailed Feature & Functional Requirements

### 4.1 Low-Level Pure C Stackful Coroutines

* **Description:** The system must implement a highly performant, custom stackful coroutine mechanism natively in C, removing dependencies on POSIX `ucontext` (which introduces signal-mask syscall overhead).
* **Technical Specification:**
* Context switching must be handled through localized `.S` assembly files (supporting `x86_64` and `aarch64` architectures).
* The context switch routine (`ctx_switch`) must only save and restore Callee-saved registers (`RSP`, `RBP`, `RBX`, `R12-R15` for x86_64), achieving sub-10 nanosecond context switch speeds.
* Every incoming connection/task must map to an allocated, fixed-size memory segment acting as the coroutine private stack (e.g., configurable from 16KB to 64KB).



### 4.2 Syscall-less `io_uring` Event Loop

* **Description:** The engine must interface directly with Linux `io_uring` structures to completely decouple request submission from system calls.
* **Technical Specification:**
* **SQPOLL Mode:** Must support the `IORING_SETUP_SQPOLL` flag, spinning up a dedicated kernel thread to fetch SQEs (Submission Queue Entries) without user space triggering an explicit `enter` syscall.
* **Resource Registration:** Must implement `io_uring_register_files` to pin persistent target descriptors and `io_uring_register_buffers` to lock memory blocks into the kernel (Fixed Buffers), eliminating runtime page table traversal costs.
* **Linked Operations:** Must utilize `IOSQE_IO_LINK` to tightly chain dependent I/O sequences (e.g., linking a file read command to a network send command).



### 4.3 In-Kernel eBPF Packet Filtering & Cleaning

* **Description:** The system must bundle a kernel-side eBPF loader and verification subsystem that intercepts, parses, and discards or sanitizes data traffic directly inside the network driver subsystem.
* **Technical Specification:**
* **XDP/TC Hooking:** Inject eBPF programs directly into the network pipeline to process packets immediately upon arrival.
* **Linked CQE Interception:** Implement a mechanisms where the eBPF filter can natively determine if an `io_uring` chain should proceed. If a packet matches malicious heuristics, it is dropped or responded to with an error packet entirely inside the kernel.
* **Lockless Analytics Channel:** Utilize `BPF_MAP_TYPE_RINGBUF` to pipe analytical telemetry back to the user-space core engine without inducing spinlock contention.



### 4.4 Kernel-to-Hardware True Zero-Copy (`SEND_ZC`)

* **Description:** The system must chain file storage reading directly into network egress without loading data payloads into standard user-space structures.
* **Technical Specification:**
* File blocks are read using `IORING_OP_READ_FIXED` into pre-registered physical memory pages.
* Outbound transmission must trigger `IORING_OP_SEND_ZC` (Send Zero-Copy), instructing the network card's DMA engine to directly stream bytes out of the identical physical memory location.



### 4.5 Multi-Language Universal Bindings (FFI Layer)

* **Description:** The architecture must enforce a strict C-only public API boundary with zero naming decorations (Name Mangling) to serve as a universal target for wrappers.
* **Technical Specification:**
* Public headers (`aeroring.h`) must contain clean, packed C structs aligned perfectly with kernel expectations.
* Expose an inversion-of-control paradigm where external runtime loops (e.g., Python's Loop, Node's `uv_loop`) can pass tasks into the C core and receive completions via pointers mapped through predictable FFI channels.



---

## 5. Non-Functional Requirements & Performance Metrics

### 5.1 Throughput and Scalability

* **Thread-per-Core Alignment:** The system must support a completely independent architecture layout where each CPU core binds one OS thread, one `io_uring` ring, and its own batch of coroutines, scaling lineary across modern high-core-count processors (e.g., 64/128 core systems).
* **Saturating Network Capacity:** The engine must be capable of saturating 40Gbps/100Gbps network interfaces for static payload distribution using minimal CPU utilization.

### 5.2 Performance Benchmarks (KPIs)

* **Context Switch Time:** Must remain under 8 nanoseconds per coroutine jump.
* **Syscall Count:** Under steady-state network load, the ratio of processed network packets to user-space syscall entries must be maximized towards infinity (approaching zero active system calls).
* **Memory footprint:** Base memory overhead per inactive connection must not exceed the specified stack frame size (e.g., < 32 KB per idle channel).

### 5.3 Reliability and Safety

* **eBPF Verifier Compliance:** All built-in kernel-side programs must strictly pass the Linux kernel BPF verifier (finite bounds, safe pointer arithmetic, stack depth limits < 512 bytes).
* **Graceful Degradation:** If `SQPOLL` privileges are denied by host security profiles or kernel constraints, the engine must safely fall back to coordinated, batch-based user-space ring submissions via fallback syscall intervals.

---

## 6. Implementation Timeline & Technical Roadmap

```
Phase 1: Foundation (Days 1-2)
├── Design and compile ctx_switch.S (Assembly context switcher)
└── Set up fundamental coroutine scheduler and basic memory frames

Phase 2: Kernel I/O Bridging (Days 3-4)
├── Initialize liburing with SQPOLL and Fixed Buffers
└── Map coroutine pointer handles to sqe->user_data fields

Phase 3: Kernel Coprocessing (Days 5-6)
├── Author C-based eBPF scripts for XDP packet filtering
└── Establish BPF Ring Buffer for lockless telemetry feedback

Phase 4: Optimization & Interface (Day 7+)
├── Wire up IORING_OP_SEND_ZC for hardware DMA zero-copying
└── Standardize aeroring.h and expose the public C ABI surface

```
