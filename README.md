# Translating Xbox One memory to PC D3D12/Vulkan: what we built, what broke, what we learned

*A technical write-up for someone starting an Xbox Series X translator.*

## 0. About this document

### 0.1 What this is
We run an Xbox One D3D12.X game on PC: Windows D3D12, and Linux through Proton/vkd3d-proton (D3D12 on Vulkan). The loader is a shared core that more than one title uses. It provides the console's system and graphics APIs on top of the desktop ones.

Most of our hard problems came from one place: **memory**. On the console the CPU and GPU share one physical memory and one address space. On a PC they do not. This document covers:

- how we bridged that gap;
- the invariants we ended up enforcing;
- the failure modes we hit and how we found them;
- what everything cost, in measured numbers;
- what we would do differently from day one.

It is long on purpose. Every section follows the same pattern: what the mechanism does, why it exists, what we tried and threw away, its exact invariants, how it failed, what it costs, and what we'd change.

### 0.2 How to read the numbers
- All numbers are from our own hardware:
  - an AMD desktop: Ryzen 7 5800X3D, Radeon RX 9070 XT, Linux/Proton;
  - an AMD handheld: Ryzen Z2 Extreme, Radeon 890M, SteamOS/Proton, measured at about 15 W in game;
  - an RTX 4090 desktop on Windows;
  - an RTX 3080 laptop on Windows;
  - a Ryzen 9 5900X desktop with an RX 6700 XT, used as a build and test box.
- Treat them as orders of magnitude and ratios. They are not benchmarks.
- "Slow frame" means a present interval over 33.4 ms, two 60 Hz refreshes. We also use the 16.7 ms and 50 ms lines.
- "Real" slow frames exclude frames our own test harness spent capturing screenshots. Section 9.4 explains why that matters so much.
- Every comparison was made against an A/B noise band. Each arm ran at least twice, with the "off vs off" spread quoted. Anything inside that spread is called noise, however attractive it looks.

### 0.3 Terms
| Term | Meaning here |
|---|---|
| Title | The game executable and its own code. |
| Title memory | Memory the title allocates through the console's VM APIs and writes directly. On the console it is also what the GPU reads. |
| Title VA | An address in the title's view, including addresses it believes are GPU addresses. |
| Desktop resource | The real D3D12 (or vkd3d-proton) resource we create to stand in for a title resource. |
| Span | A contiguous piece of a texture's title memory we track as a unit, usually one mip or one packed-mip tail. |
| Barrier | Our check-and-copy step that runs inside `ExecuteCommandLists`, before the title's command lists reach the GPU. |
| Head-start (HS) worker | An off-thread worker that prepares copies before the barrier asks for them. |
| Need | A span the barrier must have a fresh GPU copy of before this submission can go. |
| Stolen need | A need whose head-start job was queued but not started, so the barrier took it back and staged it inline on the submitting thread. |
| Exact | The rule that the GPU never sees anything other than the bytes the title last wrote (section 5.1). |

### 0.4 Short version
If you read nothing else:

1. **Decide your correctness contract first.** Ours became *submission equivalence*: when a command list reaches the GPU, every texture it can reach holds the bytes the title had written before it called `ExecuteCommandLists`. Every shortcut we tried that broke that contract (placeholders, lower mips, deferred copies, waiting a bit longer) came back as a visible bug: black road, black walls, magenta checkers in a debug fill.
2. **Own the address space from the first instruction.** Reserve the title's fixed ranges before the graphics driver starts allocating. Hook every way memory is allocated, protected, queried or freed, including `GetProcAddress` lookups and kernel-side writers.
3. **Give every GPU resource a generation, not just an address.** The title reuses addresses within microseconds. About 1,000 resources a second were created in our game.
4. **Page-protection traps beat write-watch harvesting** for finding CPU writes, but the fault handler must be written as if it were an interrupt handler.
5. **Your submit-time budget is tiny.** A synthetic 1 ms busy-wait per direct submission cost 42 % of presents; 0.2 ms cost 14 %.
6. **Under Wine/Proton the costs move.** Virtual memory calls take a process-wide lock. `VirtualQuery` can cost hundreds of microseconds. Thread priorities do nothing. In-process `/proc` reads describe the wrong thread.
7. **Instrument before you optimise, and make every instrument an observer.** Most of our "fixes" changed nothing measurable. The ones that worked came from a timer or counter that pointed at the right place.
8. **The GPU was mostly idle** (15-20 % busy on the RTX 4090). The frame limit was the CPU path between the title and the GPU.

---

## 1. The memory model gap

### 1.1 What the console gives the title
On the Xbox One, the title sees one physical memory pool with a unified address space. A few properties matter here:

- **CPU pointers are GPU addresses.** When the title places a resource at an address, the resource's GPU virtual address *is* that pointer. Titles rely on it. Ours frees a released resource's memory with `free(resource->GetGPUVirtualAddress())`.
- **The CPU writes GPU-visible memory directly.** Textures and buffers are filled by plain CPU stores into title memory, usually in the console's tiled layout. They are also filled by DMA engines and by file reads that land in that memory. No upload step, no copy, no `Map`/`Unmap` bracketing is needed.
- **Most title memory is write-combined.** That is fast for streaming writes and slow for CPU reads.
- **ESRAM.** There is a small, fast on-chip memory that the title "maps" at addresses it chooses.
- **Address ranges are fixed.** The title reserves large fixed virtual ranges at startup and carves its own pools from them. If a reservation fails, our game does not crash at once. It records a pool of size zero and later writes off the end of a heap block.
- **Placement is cheap.** A "placed resource at this address" call costs almost nothing, and the title makes them constantly. Ours creates about one resource per bundle buffer, roughly 1,000 a second, and destroys them as fast.
- **Descriptors are plain memory too.** A descriptor table is a block of memory the title writes with the CPU. The GPU reads it when a draw executes.

### 1.2 What a PC gives us
On desktop D3D12, and on vkd3d-proton, which maps D3D12 onto Vulkan:

- **CPU and GPU address spaces are separate.** A resource has a GPU VA chosen by the driver. You cannot ask for "this GPU address equal to that CPU pointer".
- **Heaps have types.** DEFAULT is GPU-local and not CPU-visible. UPLOAD is CPU-write, GPU-read, write-combined. READBACK goes the other way. CUSTOM heaps let you choose CPU page properties and memory pool. GPU_UPLOAD (Resizable BAR) is CPU-visible VRAM where the platform supports it.
- **Textures live in driver-chosen layouts.** The CPU cannot write a DEFAULT texture's bytes. Data arrives by a GPU copy from a buffer, or through an UPLOAD/CUSTOM placement with linear layout, which is restricted and slow.
- **Content must be copied explicitly.** Someone has to notice the CPU wrote, read those bytes, convert them (detile), and record a copy before the GPU uses them.
- **Placed resources need a heap.** `CreatePlacedResource` needs a heap object and an offset, not an arbitrary CPU address. `OpenExistingHeapFromAddress` can wrap existing CPU memory as a heap, with limits (section 3.3).
- **Lifetime is the application's job.** D3D12 already says the application must not free a resource while the GPU may still use it. The title obeys that *in console terms*, which do not always match the copies and stand-ins we create.

### 1.3 The gap as a picture

```
CONSOLE                                    PC (our translation)
-------                                    --------------------
title CPU stores ──► title memory ◄── GPU  title CPU stores ──► title memory (plain CPU pages)
                     (one copy)                                    │
                                                                   │ 1. detect the write  (traps / write-watch / I/O ledger)
                                                                   │ 2. read + hash + detile (snapshot)
                                                                   ▼
                                                              UPLOAD buffer
                                                                   │ 3. CopyTextureRegion, recorded on the same queue,
                                                                   │    ahead of the title's lists
                                                                   ▼
                                                    desktop texture (DEFAULT heap) ◄── GPU
title "GPU VA" = title pointer             title VA ──► (interval map, generation) ──► desktop GPU VA
```

The rest of the document is about the three numbered arrows and the address arrow at the bottom: making them correct, then making them cheap.

### 1.4 What we decided early, and would decide again
- **Textures are never aliased onto title memory.** The GPU always reads a desktop texture we own, and we copy content into it. Detiling alone makes direct aliasing impractical for the console's tiled layouts. It also gives us a clean point to enforce exactness.
- **Buffers can be CPU-visible.** The title `Map`s buffers that live in DEFAULT heaps, so we move DEFAULT-heap buffers into a CPU-visible heap (section 3.3).
- **Address identity is preserved for the title.** Every GPU address the title sees is a title VA. We translate it to the desktop GPU VA when the title hands it back to the API.

---

## 2. Owning the title's address space

### 2.1 Reserve before anyone else allocates
**What.** At the very start of the loader's `main`, before the D3D12 device or any driver DLL loads, we reserve the title's fixed ranges as `PAGE_NOACCESS` placeholders. When the title later asks for an address inside one, the hook frees the matching part of the placeholder and lets the real allocation through.

**Why.** If you create the device first, the driver sometimes puts its own allocations inside the title's ranges. The title then gets a failed reservation, and in our case a silent zero-size pool. That gave a crash minutes later, far from the cause.

**The remaining hole.** On native Windows, ASLR sometimes places the main thread's stack, the PEB or the first TEB inside a title range *before our code runs*. We measured this at about **1.85 % of launches**. No placeholder can win that race in-process. The fix is a relaunch:

- the loader detects the lost hold;
- it starts a fresh copy of itself, up to four times;
- the child exits with a dedicated retry code if it also loses;
- a kill-on-close job object keeps the process ID and exit code consistent for whatever launched us.

We never saw this under Proton, but the guard is on everywhere.

**Lesson.** A failure that shows up "1 in 50 launches, minutes in, as a crash in title code" was really a startup address-space race. Log every reservation result at startup with its address. It turns a mystery crash into a one-line diagnosis.

### 2.2 Hook every way memory changes
The title imports its VM API from the console's kernel library. We bind those imports to our own functions:

| Title API | What our hook does |
|---|---|
| `VirtualAlloc` | Strips console-only flags; maps console page protections to desktop ones (section 2.4); commits through the write-detection layer so new pages start tracked; records the range in the title-memory ledger; bumps a commit counter that tells waiting textures "memory you were waiting for may now exist". |
| `VirtualFree` | Resolves `size == 0` frees to the real allocation range (section 2.3); marks the range as freed or decommitted *before and after* the real call; updates the ledger. |
| `VirtualProtect` | Applies the change through the trap layer, so our own read-only arming is preserved or released correctly; updates the ledger. |
| `VirtualQuery` | Answers through the trap layer. A range we have armed read-only is first disarmed, so the title never sees our `PAGE_READONLY`. |
| `GetProcAddress` | Redirects dynamic lookups of `VirtualAlloc`/`Free`/`Protect` to our hooks. Without this, a title or middleware library that resolves them at runtime bypasses all tracking. |
| Console heap allocator (`XMemAlloc`/`XMemFree` and the custom-hooks setter) | Large allocations (64 KiB and up), and all allocations when exact mode is on, go through a write-watched `VirtualAlloc`, so their pages can be tracked. Small ones use an aligned CRT allocation (4 KiB alignment above 4 KiB, otherwise 64 bytes). A title-installed custom allocator is recorded, because its memory is no longer ours to see. |
| ESRAM mapping | Implemented as an ordinary commit at the title's chosen address, with the same tracking marks. |
| Memory status query | Reports console-sized totals (5 GB or 9 GB depending on console model) minus a fixed amount in use. An optional mode reports live numbers. |

All of these take one recursive "title memory mutation" lock while tracking is on. Each runs inside a *mapping scope* (section 4.6) that suspends trap arming over the range it touches.

**Invariants.**
- Every change to the mapping of title memory passes through one hook, under one lock.
- The hook marks the range dirty *before* the change and again *after* it. A concurrent reader either sees the old mapping with its old trust, or is told the span is untrusted.
- Hooks preserve `GetLastError`. Title code reads the error after calling hooked APIs. A diagnostic that calls something which clears the error changes title behaviour.

### 2.3 The size-zero free, our most expensive one-line bug
On the console, the title frees blocks with size 0, meaning "the whole allocation". Thousands of these happen at a level load. Our first marking code read `size == 0` as "everything", and marked *all title memory* dirty on each free. Each of those passes:

- held the submission lock;
- walked about 22,000 textures and 140,000 spans.

Measured:
- On the laptop, 8,648 such holds added up to 455 s in one run.
- On the 4090, 13,051 holds added up to 482 s, the longest 0.86 s.
- Submissions waited 138 s and 243 s in total.

The fix resolves each size-0 free to its real allocation range and runs callbacks outside the lock. The submission lock wait dropped from 243 s to 0.12 s, and the 4090 ran about as many presents as with the fix off.

**Lesson.** Any "whole memory" operation in a hook is a bug waiting for a title that calls it often. Resolve console shorthand (size 0, NULL base, flag tricks) to exact ranges at the hook boundary, once.

### 2.4 Page protections and write-combining
Console title memory is write-combined. A literal translation asks the PC for `PAGE_WRITECOMBINE` pages. That is faithful but expensive the moment *we* read the memory, and the exact path reads every byte it copies.

Measured:
- Write-combined reads on Windows: about **3.2-3.8 ms per MiB**, against 0.8-1.0 ms on the Linux handheld.
- A byte-wise zero scan: about 25 MB/s per worker on write-combined pages against about 890 MB/s on cached ones.
- A readability scan went from 52 MB/s to about 4,800 MB/s when we allocated title memory as ordinary cached pages.

With cached title memory on the 4090, hitches fell from about 27 a minute to about 1 a minute, and the write-detection audit stayed clean. So we remove `PAGE_WRITECOMBINE` and `PAGE_NOCACHE` from title allocations by default. A protection with no access bits falls back to read-write.

**Platform trap.** Windows commits real pages the first time you *read* never-written memory. Linux maps a shared zero page until the first write. So "scan to see whether it is zero" has very different costs, and very different memory growth, on the two.

### 2.5 The ledger: answering "what is this memory?" without asking the OS
**What.** A *title memory map*: an interval structure fed by the hooks. It holds spans of `{allocation base, state, protection}` plus reservations, with neighbours merged. A *mutation epoch* counter is bumped after every successful alloc, free, protect, heap alloc/free or ESRAM map.

**Why.** We need "is this span committed and readable?" thousands of times per frame. Under Wine, `VirtualQuery` is a call into the wineserver. On a sibling title we measured about **250 µs per call**, and its source-readability scan used about 30 % of the title's main thread in `VirtualQuery`. Earlier, our own re-check slice of 2 ms per present could afford only about 8 such calls, so the cost of *asking* limited how many textures we could notice per second.

**Invariants.**
- The map is a cache of facts *we* caused through the hooks. Any address outside it gets one real `VirtualQuery`, and that answer is never cached.
- If the map fails to allocate, proof is switched off for the process. We fall back to the OS, never to a guess.
- A separate *readability cache* sits on top. It must be invalidated by every mark. When it went stale, a span cached as "not readable" while it actually was readable could skip its copy. That is the black-road bug again (section 6.6).

**Results.** On the sibling title the map removed the scan cost: 69 s down to 0.12 s per run. Texture visits rose from about 0.1 M to 3-3.6 M per run, because noticing became cheap. Frame rate was unchanged within noise, which is a reminder that removing one cost exposes the next.

**What we'd change.** Build the ledger on day one. Make *every* consumer use it. Keep a debug mode that compares each answer with `VirtualQuery`.

### 2.6 Allocation bookkeeping costs real time
Once the exact path is on, every title VM call does more work than the kernel call underneath. A timing observer that split each hook into "native" (inside the kernel or driver) and "wrapper" (our code) found, on the AMD desktop, title-thread hooked create/alloc time of **24.7 ms per slow frame** against 2.2 ms in an average frame:

| Hooked call | ms per slow frame | inside the OS / driver | our wrapper |
|---|---|---|---|
| VirtualAlloc | 6.6 | 5.0 | 1.5 |
| VirtualFree | 5.6 | 1.6 | 4.0 |
| Placed resource (console variant) | 2.8 | 0.6 | 2.2 |
| Committed resource | 1.7 | 0.5 | 1.2 |
| Command list creation | 1.7 | | |

More measurements:
- Per call: VirtualFree about 211 µs (35 µs in the syscall), VirtualAlloc about 123 µs (16 µs in the syscall).
- The bare kernel time was only about 0.1 ms per frame. Almost all the cost was ours, or Wine's VM bookkeeping.
- vkd3d's own time inside `CreateCommittedResource` was only 0.5 ms per slow frame. Sub-allocating committed resources would have gained almost nothing. We measured that before building it.
- The cost follows streaming. A parked car gave 34 slow frames a minute against 86-95 while racing.
- One `VirtualFree` of a tracked range did three trap-registry window scans over about 111,000 spans, five registry lock acquisitions, three `VirtualQuery` walks and two marks.

We also hit plain memory growth:
- A bundle table was never pruned: 62,475 entries after three races, 55 % dead. A sweep cut private bytes 13-15 % (about 1.7 GiB).
- Tightening small-buffer alignment through a vkd3d-proton private flag cut private memory by a third (15.3 GiB to 10.2 GiB). The flag is hidden from `GetDesc`, with fallback if anything fails.
- About 940 upload jobs a second each created a committed UPLOAD buffer. Native creates ran at about 2,460 a second and frees at about 3,470 a second.

**Under Wine, `VirtualAlloc`/`VirtualFree` take a process-wide VM lock.** Every off-thread worker that allocates scratch memory competes with the title. Our snapshot buffers alone cost about 2.5 ms of CPU per frame in VM calls on two worker threads, until we pooled them (section 5.4).

**What we'd change.**
- Keep hook work O(change), never O(registry).
- Pool every scratch allocation from the start.
- Time "inside the OS" separately from "inside us" in every hook from day one.

---

## 3. GPU resources over title memory

### 3.1 The console "placed at address" call
The console API places a resource at a title address. We cannot place a desktop resource at an arbitrary CPU address, so:

1. We create a real desktop resource. In practice it is committed. A placed-heap scheme is possible, but this is the version that shipped.
2. We record `{title VA, size, desktop VA, generation}` in an address registry under a reader-writer lock.
3. We patch that resource's vtable:
   - `GetGPUVirtualAddress` returns the **title VA**, because the title frees memory by calling `free()` on that value;
   - `Release` is replaced, so we see destruction.
4. Every GPU address the title passes back to the API is translated title VA → desktop VA. That covers vertex and index buffer views, root CBV/SRV/UAV, indirect arguments and bundle contents. The translation is an ordered-map interval lookup, with a cached last interval keyed on a registry revision counter.

```
title VA space                         desktop GPU VA space
[ A gen 7 ][ B gen 3 ]   ──lookup──►   A' (resource for gen 7), B'
      ▲ title destroys A gen 7, immediately places C at the same base
[ C gen 8 ]                            C'   (A' may still be in flight on the GPU!)
```

### 3.2 Generations, address reuse and overlap
The title creates about 1,000 resources a second, and a destroyed resource's address is often reused by a new one *before our `Release` hook has removed the old entry*. So:

- every registry entry carries a **generation**, and every lookup is "address plus generation" wherever we can know it;
- when ranges overlap, we keep the newest live generation at each base. A fallback scan handles the case where an ended nearer interval hides a live older one, and logs it;
- a separate *identity classifier* sorts a descriptor's texture into `Current`, `OlderCompatible`, `Ambiguous` or `Unknown` among the live textures at the same title VA. It never dereferences anything. This is our ABA defence for diagnostics.

**Failure we hit.** On NVIDIA under Vulkan, device losses consistently pointed at a GPU read of memory that had been freed 5 to 389 seconds earlier. The faulting page was always in the last 4 KB (the small-mip tail) of streamed block-compressed textures, created and destroyed repeatedly at one title VA (section 8.4). Address reuse plus a stale descriptor copy is still our leading theory.

**What we'd change.** Make generation part of every handle from the start, including the ones that live inside descriptor bookkeeping. "Address equals identity" is never true here.

### 3.3 Buffers
- **The title `Map`s DEFAULT-heap buffers.** We reroute DEFAULT-heap buffers to a CPU-visible heap: GPU_UPLOAD (Resizable BAR VRAM) where available, otherwise a CUSTOM write-combined system-memory heap. An option forces write-back system memory instead.
- **Zero-copy title backing (optional).** For buffers, we can wrap the title's own pages as a heap with `ID3D12Device3::OpenExistingHeapFromAddress` and place the buffer at the right offset. The GPU then reads the title's memory directly, as on the console. Conditions are strict:
  - the whole span must be private, committed, read-write and not a guard page;
  - the wrapping heap is kept alive as private data on the resource, with no global cache, so no address-reuse confusion;
  - anything unusual falls back to the normal path.
  It is off by default. It works, but the interaction with our write traps and with Wine needs more proof than we have given it.
- **Buffer churn.** An optional slab allocator pre-creates 64 MiB buffer-only heaps (up to 1.5 GiB in total) on a low-priority thread and recycles buffers up to 4 MiB by key. Reuse is gated on fence completion on *all* queues and on pins, and memory is zeroed before reuse. It exists because of the 1,000 creates a second. It stayed off: the measured native create cost turned out small (section 2.6).
- **Console ABI quirks.** The console's custom-heap-properties call puts its hidden return buffer in a different place from the desktop one, so we need an adapter. Some console heap flags have no desktop equivalent and must be stripped. We retry heap creation without an optional "not zeroed" flag where a driver refuses it.

### 3.4 Textures
- A console tiled texture is registered in a *tiled texture table* with its detiling helper (the console's tiling library computes the layout), its epoch, and its exact-tracking state.
- The desktop texture is a normal DEFAULT resource. Its content is always uploaded by us (sections 4-5).
- **Committed resources start zeroed.** A mip we have not copied yet reads as zero, which is black. There is no placeholder, and the GPU does not wait. That one fact explains the whole "black pop-in" history in section 6.

---

## 4. Detecting CPU writes to GPU-visible data

### 4.1 The problem
To copy changed texture bytes before the GPU needs them, we must know *which* bytes changed. The title writes them with plain stores, sometimes across many threads, often in bursts during level loads. We found four writer classes:

1. **CPU stores** by title threads. This is the big one. In our game, large mips reach title memory by CPU stores; no completed file read landed in any texture we traced at a scene cut.
2. **File reads** that land directly in title memory (`ReadFile`, I/O completion ports). These are *kernel* writes. They do not fault on a read-only page; they fail with `ERROR_NOACCESS`. They also do not reliably set write-watch bits.
3. **DMA and copy-engine work** the title schedules. We emulate it, so we see it.
4. **GPU writes** (render targets, UAVs). These never touch title memory in our design, because the GPU writes the desktop resource.

### 4.2 Options we tried

| Approach | How it works | Why it lost or won |
|---|---|---|
| Periodic re-check (polling) | Revisit each waiting texture in a slice of time per present and compare. | The early design. 800-1,700 waiting textures shared a 2 ms slice, so each was revisited every 0.16-0.31 s on the handheld. That matched the black pops users saw. Also, each visit paid `VirtualQuery` costs. |
| Write-watch (`MEM_WRITE_WATCH` + `GetWriteWatch(RESET)`) | Every title reservation is made write-watched; harvest dirty pages. | Correct, but cost scaled with tracked memory. One 600 s run made **76 million** `GetWriteWatch` calls (about 1,370 per submission, 133 s in total). On the laptop: 41,000-56,000 calls and about 63 ms of harvest per present. Unplayable on the handheld. |
| Hash every reachable texture at submit | Hash, compare, upload changed bytes. | Correct and simple, and far too slow for full scans. We kept hashing to decide "equal, no upload needed" on already-captured spans. |
| **Page-protection trap** | Arm a texture's pages `PAGE_READONLY`; the first title write faults; record "dirty", restore the page and let the write retry. | Cost scales with *changes*, not with memory size. Our default, with write-watch kept as an independent checker in test builds. |

### 4.3 Trap design

**Data.**
- Each tracked texture has a state with a list of **spans**. A span is one mip, or one run of packed small mips.
- Each span owns a **token** holding:
  - its page-aligned range and exact content range;
  - the resource and generation it belongs to;
  - an atomic **phase**: `Dirty`, `Armed` or `Handling`;
  - a sequence number, the arm epoch, the last fault sequence, and a small per-span lock.
- The registry has two parts:
  - an interval map of spans;
  - a per-page table: how many armed tokens own the page, the page's original protection, its allocation, and which token will restore it.
- A per-page epoch survives until the allocation is freed, so a stale fault can be told apart from a new one.

**Arming.** Under the exclusive registry lock, a span may be armed only if every page is:
- committed and private;
- inside one allocation;
- plain read-write (optionally write-combined);
- not inside an active *exclusion* (section 4.6).

Otherwise the span is refused with a reason (unmanaged, busy, query failed, protection mismatch, mixed, neighbour conflict, protect failed). It falls back to write-watch only. Arming bumps the per-page counts, calls `VirtualProtect(PAGE_READONLY)`, and stamps the token with an arm epoch from a global sequence. That stamp lets us later order a capture against a fault.

**Arm merge.** Small textures put five or six mips in one page. Re-arming each one separately cost **132 ms for 24,453 re-arms** at the end of one track load. With merging, we skip the `VirtualProtect` when every page is already armed by a live neighbour.

### 4.4 The fault handler
A vectored exception handler. It acts only on write access violations inside a registered span. Steps:

1. Take the registry lock **shared**.
2. For each overlapping span, take that span's lock.
3. If the span is `Armed`, compare-and-swap it to `Handling`. Drop its page counts. The thread that takes a page's count from 1 to 0 restores that page's original protection, in batched runs, after checking with `VirtualQuery` that the page is still ours.
4. Push a pre-allocated event node onto a lock-free pending list, and the token onto a cleanup list. Set the phase to `Dirty`.
5. Signal the worker's semaphore.
6. Return `EXCEPTION_CONTINUE_EXECUTION`. The title's store retries and succeeds.

**Invariants. Treat the handler like an interrupt handler.**
- **No allocation, no logging, no container erase, no `shared_ptr` destruction, no external callback.** The fault can arrive on any thread, holding any lock, including the CRT heap lock.
- **Fixed lock order:** registry (shared), then one span, then atomic list publication. Nothing else.
- **No static thread-local storage.** We learned this from a crash under a debugger. The handler also runs on threads created without the loader's thread-attach step: debugger break-in threads, and threads created with "skip thread attach". On those, the TLS pointer is NULL. Any static TLS access faults inside the handler, which recurses until the stack is gone. A function-local `static` with a compiler-generated guard counts, because the guard uses TLS. Per-thread retry state now lives in `TlsAlloc` slots allocated at process start, with indices below 64, so `TlsSetValue` never allocates.
- **Stale and duplicate faults.** If no armed span matches, the handler checks under the same shared hold whether the page is now writable. That closes a read-write → read-only race against a concurrent re-arm. If it is writable, the handler retries the instruction, at most three times per address and epoch, then gives up and counts the event.

### 4.5 Drain and the event ring
One exclusive consumer **drains** the pending list. It turns events into "this span is dirty" *marks*, removes dead entries, and refreshes trust.

**Failure.** Our first event ring had a fixed size. It overflowed **127,391 times** in one run, and each overflow forced a re-check of *every* texture. The exact path then ran at 1-5 fps. Because the title's own streaming is clocked by frames (section 6.5), the world on a long straight vanished and came back about 10 seconds later.

The fix made draining O(changes), with no overflow, and batched the harvest. On the laptop, per present:

| | Before | After |
|---|---|---|
| Barrier stall | 203 ms | 7.0 ms |
| Drain | 182 ms | 0.3 ms |
| Harvest | 65 ms | 0.3 ms |
| Frame rate | about 2 fps | about 16 fps |

**Lesson.** A bounded queue whose overflow means "assume everything changed" is a cliff, not a degradation. Size it from measured peaks, make the overflow path cheap, and count every overflow.

### 4.6 Disarm on purpose: exclusions and mapping scopes
Our own `PAGE_READONLY` must never leak into title-visible behaviour:

- **Mapping scopes.** Every VM hook, ESRAM map, `VirtualQuery`, and file read begin/end opens an *exclusion* over its range. Opening one disarms every overlapping span in one exclusive hold and holds an exclusion count. Arming refuses any range under a live exclusion, so no trap can be armed during a mapping change.
- **Kernel writers.** If the title uses a kernel-side write path we do not cover (raw file reads, device I/O control, socket receive, file mapping views), we switch trapping off globally. A kernel write to an armed page fails instead of faulting, and the title would see an I/O error. A blocked kernel write is counted as a correctness breach.
- **File reads into tracked memory** go through an **I/O ledger**: a bounded open-addressing table of in-flight reads and their completions. When a read is issued, its range is excluded and marked "read issued". When it completes, the range is marked "read complete". The ledger is explicit because kernel writes neither fault nor reliably set write-watch bits.

### 4.7 Retire and "retire own"
When the title releases a texture, we **retire** its state and disarm its tokens. Originally, retire disarmed the whole page range, including *other textures' tokens* on shared pages. Two problems followed:

- A neighbour's token, disarmed by someone else's retire, looks dirty. The next barrier then re-checks and re-hashes it, finds it equal, and re-arms it. That is about **130,000 needless "equal after disarm" re-checks** per run.
- Those re-checks were what made the head-start worker look idle while the barrier staged inline. They misled one of our own design reviews (section 9.5).

The **retire-own** mode disarms only the retiring state's own tokens. Measured on the AMD desktop:

| | Before | After |
|---|---|---|
| "Equal after disarm" re-checks per run | about 130,000 | under 100 |
| Mark-staging time per slow frame | 1.9 ms | 0.08 ms |
| Stolen needs | baseline | about halved |
| Slow frames | baseline | unchanged, within noise |

The correctness run showed 0 "changed without mark" events over 3.1 million audit checks, 738,832 own-token disarms and 0 foreign-range tokens.

It is a clean, exact improvement. It did not fix hitches by itself. It is still worth having, because it removes false work that hides the real bottleneck.

### 4.8 Trust, the oracle and self-tests

**When a span is trusted.** A span is *trusted* (its last GPU copy still matches title memory) when all of these hold:
- it has a landed copy newer than its last mark;
- it is not being read by I/O;
- the capture is provably ordered after the last fault epoch.

**The oracle.** In test builds, the write-watch harvest still runs as a *second, independent detector*. When it reports a dirty page under a trusted span, we hash that span in bounded slices: SHA-256 in 64 KiB pieces, with an 8 MiB budget. A mismatch is a **trap miss**, recorded as a breach. A separate audit mode re-hashes trusted spans at random. A typical clean audit row: 1-3 million checks, 0 "changed without mark".

**Breaches never drop the title's work.** After a breach the run is flagged invalid, but the title's submission is always preserved.

**Self-tests at startup** check that the platform behaves as the design assumes: faults are delivered, write-watch works, and `ReadFile` into a protected page fails rather than faults. Under Wine, check each of these on every Wine/Proton version you support.

**Counters in a healthy run.** In one audited run:
- 39,860 traps and 1.71 million arms, matched by 1.71 million disarms;
- 14,715 refused arms;
- zero trap misses, armed-but-not-read-only, trusted-but-unarmed, blocked kernel writes and failed protects.

Before retire-own, arms were 2.2 million.

### 4.9 Lessons for write detection
- Build two independent detectors, and keep the second one in test builds. The disagreements are your bug reports.
- Every arm, disarm, fault, refusal and overflow gets a counter, and every run prints them. "Trap health all zero" became a required check before reading any performance number.
- Kernel writers, file reads and anything not on a CPU store path need their own ledger.
- Treat the trap layer's own `VirtualProtect` and `VirtualQuery` calls as hot-path costs. Under Wine they are.

---

## 5. Exact texture staging

### 5.1 The contract: submission equivalence
After many attempts (section 6), we wrote down one rule and made everything else serve it:

> **Before the title's command lists in an `ExecuteCommandLists` call reach the GPU, every committed span of every CPU-produced texture those lists can reach has a GPU copy made from bytes that were not marked dirty after they were read.**

What the rule rules out:
- **Placeholders, neutral fills or "last good" content.** A console has no such state. The faithful value of an unwritten tile is what the console would show, not something we make up.
- **Lowering the mip, distance or quality** so that something "safe" draws instead.
- **Skipping or deferring a copy** to a later frame.
- **Waiting on a GPU fence** to "let things settle", or **dropping a submission**.

What the rule allows:
- **One-submission lateness for genuine races.** If the title writes a span *while* our barrier is reading it, the next barrier sees the mark and copies again. On the console the GPU and CPU would race in the same way. A racing title write becomes visible one submission later, and we count these events.
- **Doing the work early** (section 5.5), as long as the barrier can prove the early copy is still current.

The copy runs on the **same direct queue, ahead of the title's lists in the same submission**, so no cross-queue synchronisation is needed. The only thing the title can observe is the order of submissions, and that order is preserved.

An earlier experimental path tried to publish content when a *descriptor* was created ("first use"). It is off everywhere. The lesson we wrote into it: **creating a descriptor does not publish content.** Only something that sees *every* writer may treat bytes as final.

### 5.2 The barrier, step by step
The barrier runs inside our `ExecuteCommandLists` hook, under one *submission boundary* lock, before the native submit:

```
q_ExecuteCommandLists(lists)
 └─ boundary lock
     1. REACH      which textures can these lists sample?          (section 7.2)
     2. HARVEST    collect write-watch bits (if any)
        DRAIN      turn trap events into dirty marks
     3. CLASSIFY   per reached texture, per span:
                     trusted?  readable?  head-start ticket present and fresh?
     4. STAGE      per span that needs bytes:
                     fresh head-start ticket ─► use it
                     queued but not started  ─► take it back ("stolen"), stage inline
                     running                 ─► wait for it (or claim and continue, optional)
                     none / stale            ─► stage inline
     5. SUBMIT     record barriers → COPY_DEST, CopyTextureRegion × N, barriers back,
                   execute on the direct queue, signal the upload fence, mark spans "landed"
     6. POST-RACE  any mark that landed after its snapshot was read → span stays untrusted
                   for the NEXT barrier (counted, never retried inside this one)
 └─ native ExecuteCommandLists(title lists)
```

Lists bound for other queues get the reach walk for bookkeeping but no staging. All texture copies go on the direct queue in our current design.

### 5.3 Snapshot, hash, detile
For each span that needs bytes:

1. **Snapshot.**
   - Arm the trap for the span first. Arming before reading is what makes a racing write detectable.
   - Copy the span's title bytes into a private snapshot region, using a guarded read. The memory can be decommitted under us, and the guard turns that into "this capture failed", not a crash.
   - Hash the snapshot (SHA-256).
   - Record a capture: span, sequence, hash, whether it was all zero, and a mapping signature (section 5.4).
2. **Equal accept.** If the hash equals the last landed capture, the span becomes trusted again with no upload. After the retire-own fix, this path stopped being a large share of the work.
3. **Detile.** The console tiling library converts the tiled bytes to a linear subresource in an UPLOAD buffer.
4. **Record the copy** into the barrier's upload list.

Costs we hit, and what fixed them:
- **Hashing under Wine.** Wine's crypto provider did not use the CPU's SHA instructions. Hashing cost about **300 ms of CPU per second**, and the presenting thread waited for it. We wrote our own SHA-256 with SHA-NI (scalar where the CPU lacks it). A startup self-test checks that its digests are bit-identical to the system provider's, or we keep the system one. A later mode hashes the snapshot in place and folds the all-zero test into the same loop.
- **Detiling.** The console tiling library cost about 65 µs per 21 KiB call on the handheld: 3.2 ms per MiB, 55-60 seconds per race across all threads.
  - For the one layout class this game uses heavily, the detile is a pure byte permutation made of short runs.
  - A *learned run table*, built on a lowest-priority thread, does the same copy about 10x faster.
  - The first four uses of each layout class are compared byte for byte with the library's output. One mismatch disables the class for good.
  - It is off by default until it has run in more titles.
- **Scratch allocation.** Snapshot regions were reserved and freed per job. Under Wine each `VirtualAlloc`/`VirtualFree` takes the process-wide VM lock: about 2.5 ms of CPU per frame on the worker threads. A snapshot pool now reuses reservations with their committed pages. For exactness, a reused block is made indistinguishable from a fresh one before any job reads it:
  - pages outside the needed range are decommitted;
  - reused pages are zeroed except the bytes the caller will fill;
  - a test mode verifies all of this.
- **Upload batching.** A barrier pass's copies are gathered into one command list, one `ExecuteCommandLists` and one `Signal`, capped at 64 jobs per batch. A conflict (same resource twice, or an ineligible job) flushes early.
  - An earlier batching variant built batches of **4,470 members** on the 5900X desktop and **53,405** on the handheld in one submission. Every 5900X run hung about 30 s in, and one handheld run lost the device at 158 s.
  - **Cap batch sizes.**
- **Upload buffer churn.** An upload-block pool and a layout-footprint cache exist. They are off by default because they did not move frame time.

### 5.4 Leases: keeping the memory mapped while you read it
**Failure.** In every run with exactness on, we saw "snapshot read" breaches: a guarded copy faulted while the span looked readable. Root cause:

1. the barrier checked a span, found it readable, and armed it;
2. meanwhile the title, on another thread, decommitted the same pages with `VirtualFree(MEM_DECOMMIT)`;
3. our resource pin kept the *desktop resource* alive, but nothing kept the *title memory* mapped.

One test run had 2,502 such rows. 2,492 were on the speculative head-start worker, which is harmless because the barrier re-stages. Ten were on the barrier thread, which is not harmless.

**Fix: a striped snapshot lease.**
- Title VA is cut into 64 KB regions, mapped onto 1,024 stripes. Each stripe has a reader-writer lock and an epoch.
- Readers (snapshots) take stripes **shared**.
- Mutators take stripes **exclusive**, always in ascending stripe order, so no deadlock is possible. Mutators are free, decommit, protect-to-inaccessible and `MEM_RESET`.
- Each reservation has an identity record `{base, end, id, epoch}`, and every mutation bumps the epochs.
- A capture stores a signature (a hash of identity records and region epochs) taken under its shared lease. Whoever accepts the capture recomputes it. If it changed, the mapping moved under the copy, and the capture is refused.

**Invariant.** A capture is valid only if the memory it read was continuously the same mapping from before the read until after it.

### 5.5 The head-start worker
Staging inline on the submitting thread costs frame time. The head-start (HS) worker prepares captures *early*, from the moment a texture is marked dirty or first viewed. The barrier then usually just picks up a finished **ticket**.

**When the barrier asks for a span, it finds one of these:**

| Outcome | Meaning | Barrier action |
|---|---|---|
| Used | A finished ticket whose capture is still current. | Accept, record the copy. |
| Stale | A ticket exists, but the span was marked after the capture. | Drop it, stage fresh inline once. |
| Stolen | The job is queued but not started. | Remove it from the queue, stage inline. |
| Running | The worker is in the middle of it. | Wait (about 0.37 ms per wait; 22,000 waits and about 8 s per race on the 4090), or claim it and keep going (optional). |
| Missing | Nobody ever requested it. | Stage inline. |

**Refresh.** In the first design, a repeat request for an entry that already existed was dropped. A job that ran before the title had finished writing therefore always ended in an inline stage. With *refresh*, a request advances a per-entry "wanted" revision in any state. A finished pass publishes immutable tickets, and a new pass is queued if "wanted" moved. A refresh keeps the still-valid tickets and re-stages only the others. The chain is capped, so a texture the title rewrites every frame cannot monopolise the worker. This is on by default.

**Eager mode** stages as soon as a mark arrives, rather than when the texture is next viewed.

**What we tried for the "stolen" problem, in order:**

1. **A cheaper exclusion lookup** (an exact page-aligned key replacing a window walk). About 700,000 walks skipped per run with 0 misses; no change to frame time.
2. **Hot/cold queues** (viewed textures first). 37,000-50,000 hot-stolen needs per run remained; no change.
3. **Both together.** No change.
4. **Preview** (stage readable textures before their first view). Worse in one race (112 against 89 slow frames per minute), slightly better in the other. Rejected.
5. **Normal thread priority for the worker.** No effect under Proton, because every Windows thread there runs at the same Linux nice value (section 9.8).
6. **A second worker.** Stolen needs fell **85 %** (36.5 to 5.5 per slow frame), but real slow frames were unchanged on the AMD desktop and *worse* on the handheld at 15 W: zero-need slow frames went from 11.6 to 28 per minute, which looks like CPU contention.
7. **A second worker plus eager staging.** The first measured gain (section 9.6).

**Why ordering fixes did nothing.** The *lead time* was already long. A histogram of how long before the need each stolen job was queued showed 33,913 of about 37,500 queued **16 ms or more** ahead. The jobs were not late. One worker simply could not get through them. Throughput was the limit, not order.

### 5.6 Fan-out
When the barrier still has a big inline batch, an optional fan-out splits it across up to eight helper threads plus the submitting thread, by texture, largest first. It starts above 16 spans or 2 MiB. One texture's spans stay on one thread, because its detiling helper is not thread-safe. On hybrid CPUs the helpers avoid the presenting thread's fast cores. This mattered most for level loads.

The original level-load frame took **12 s**. The barrier copied 84,058 spans (65 MB) one by one: about 5.1 s staging, 4.0 s preparing and 2.7 s submitting. The time was per-span overhead, not bytes. Batching per texture, pooled upload memory, batched submission and fan-out brought the level-load stall down to a few hundred milliseconds. In one checked run, the race-load window was 435 ms and its largest present 216 ms, where it had been 11.7 s in a single present.

### 5.7 Why exactness, and what it costs
**The "speed is correctness" proof.** We ran the game with exactness *off*, deliberately paced to about 2.5 fps. The game itself dropped trees, an overpass and other scenery, because its own streaming runs on frame-clocked logic. So a correct but slow implementation is not correct to the player: the title's world depends on frame rate. Any design that makes frames slow enough breaks the game in a different way.

**Measured cost of the exact path once mature** (fix on against fix off, same build):

| Machine | Race start | In race |
|---|---|---|
| RTX 4090 desktop | about -19 % fps (52.3 vs 64.5) | about -13 % (62.4 vs 72.0) |
| RTX 3080 laptop | about -45 % (20.8 vs 36.9) | about -30 % (28.9 vs 41.4) |
| Handheld 15 W | | 22.6-23.5 vs 28.1-32.9 fps |

**It adds almost no GPU work.** Traced on the 4090: 3.1 vs 3.3 ms of GPU work per frame, with the same 24 command buffers per frame. On the handheld: 22.6 vs 22.5 ms. The whole cost is CPU on the path to the GPU. In GPU terms it shows up as a **9-26 ms idle gap before the title's first big render list** (section 12).

After the exact path was correct, a viewer on the 4090 reported that, apart from performance, the game finally looked right.

---

## 6. The black pop-in history, and what it taught us

This section is the story behind section 5.1's contract. We spent weeks on it, and most of that time went on attempts that could never have worked.

### 6.1 The symptom
Streamed textures, the road surface most visibly, sometimes drew **black** for 0.2-0.3 s. At scene cuts, whole regions were black for seconds, worst on lap 1 of a race. The original game shows no such thing.

### 6.2 Timeline
| Phase | What we believed | What we did | What happened |
|---|---|---|---|
| 1 | The upload pipeline was too slow. | Removed caps on the uploader (it was limited to a few uploads per present). | **Real win for lap 1**: 981 vs 231 uploads/s on the handheld, and the lap-1 road lit from 0:10 instead of 1:18. Short pops remained. |
| 2 | Upload ordering or priority was wrong. | Priority queues, "first lane" for early draws, fair queues, wake-ups, admission tuning, cancelling obsolete jobs, more drain jobs per present. | Big latency improvements (one fix cut ready-to-landed p95 from 29.0 s to 2.3 s, another cut landings over 1 s from 10.7 % to 0.1-0.2 %). **None removed the black.** One left 0 live jobs in the pipeline while 19,750 of 94,696 viewed generations never got mip 0. |
| 3 | The CPU detile was the bottleneck. | GPU detile. | Driver CPU time 49.96 s to 1.12 s. Still black. |
| 4 | Copy when needed: before the first draw. | "Demand pull", an inline copy before the draw, with a 64 MiB budget. | Hitches over 250 ms went from 2 to **115**, and 6-8 fps. Unbounded inline work on the draw path is not an option. |
| 5 | Make it not *look* black. | Clamp the descriptor's most-detailed mip to what has landed; a "floor" mip; a top-up policy. | Marginal at best. The clamp could act on only about 12-17 % of descriptor decisions, because most black samples had *nothing* resident yet. The floor made opponents' cars near-black. |
| 6 | Waiting textures are found too late. | Measured the re-check: each waiting texture was revisited only every 0.16-0.31 s. Built an event-driven *settle wake*. | Black road per lap on the AMD desktop: **6.5 / 6.3 / 5.3 s → 0 / 0 / 0.27 s**. Starving the re-check deliberately brought it back: 88 % of whole-road shots black. With the wake, 1.1 %. Shipped. Short flashes remained. |
| 7 | Remaining flashes are the title rendering black. | Painted the road draw a solid colour (*pixel ownership*: is that draw actually there?), then filled every not-yet-uploaded mip with **magenta**. | The road draw was there every frame. Magenta appeared in 69 of 73 marked moments: trees, fences, kerbs, decals. Worst after scene cuts (garage menu about 50-61 % magenta for about 3 s). **The data was simply not on the GPU when sampled.** |
| 8 | We must know exactly when the title finished writing. | The exact path (sections 4-5). | Correct images. The rest of the work was performance. |

### 6.3 Why the early fixes could not work
- **Unwritten mips read zero.** Committed resources are zeroed, draws never wait, and there was no placeholder. A late upload is black, whatever the queue order.
- **"Late" was mostly "not noticed".** Data reached title memory by CPU stores. Our polling noticed it fractions of a second later. Faster *uploading* cannot fix slow *noticing*.
- **Lowering the mip only helps when a lower mip is resident.** Most black samples had none.
- **The title asked for its data correctly.** Changing what or how much the title requested (texture detail, budgets) made no difference. Peak texture memory stayed at about 1.3 GiB with a budget of 2 GiB or 3.5 GiB.

### 6.4 Debugging methods that worked
1. **Prove pixel ownership before analysing shaders.** Tint the suspect draw a flat colour. If the tint is solid every frame, the draw is fine and its *inputs* are wrong. This removed a whole class of theories in one short run.
2. **Fill missing data with an impossible colour.** Magenta checkers, filled at registration time, made "not uploaded yet" visible everywhere at once. A control run showed 25 of 25 probe pixels at exact zero. Be aware that the fill cannot cover render targets, depth or UAV textures, and that it changes timing.
3. **Starve the suspected mechanism on purpose.** If shrinking the re-check budget makes the bug worse in proportion, you have the mechanism.
4. **Trust eyes over scorers.** Automated "percent black in a region" scorers were blind to some cases: cars and shadows covered the road, and the scorer compared different shots. When a human viewer saw it and the scorer did not, the viewer was right every time.
5. **Keep a "tried / ruled out" table** for each problem, with the switch, machine, result and what it rules out. Ours grew to over 40 rows. It stopped us retrying ordering variants of the same failed idea.

### 6.5 The frame-clocked world
When the exact path was slow (1-5 fps in early builds), the world on a long straight vanished for about 10 seconds. That was not our bug in the usual sense. The title's own streaming and our wall-clock uploader both fell behind because frames were slow. Moving our base uploader to a wall clock (60 Hz, below-normal priority) and making the drain O(changes) fixed it. **Measure performance and correctness together. A fix that halves frame rate changes what the title streams.**

### 6.6 Exactness bugs we found in our own exact path
Each of these would have brought black textures back silently:

- **Stale readability cache.** A long proof run found 27,830 events where a span was cached as unreadable while its memory was readable, so its copy could be skipped. Cause: a birth race, where readability was computed before the state was inserted, so a commit in between was never seen. Fixed, and the count went to 0.
- **Snapshot reads of memory being decommitted** (section 5.4).
- **Bundle template shortcut.** A proposed optimisation recorded a bundle's descriptor-table use once and reused it. A caller that inherits a table set *inside* a bundle would then record the old table: silent under-inclusion. Blocked in review before it was built.
- **"No title backing" rename.** Three menu textures are filled every frame by *buffer copies*, which are not CPU stores, so the barrier never saw them. A proposed rename would have dropped them. We added copy sources to the census instead.
- **Size-zero free** (section 2.3) was a cost bug, not a correctness bug. But it shows how one console shorthand can dominate everything.

---

## 7. Where draws meet memory

### 7.1 What a draw needs from us
On the console, a draw reads whatever is in memory when the GPU executes it. On the PC, by the time a command list reaches the GPU, we must have:

1. translated every title GPU address in it to a desktop address (section 3.1);
2. made sure every texture it can sample has current content (section 5);
3. made sure every descriptor it reads names a live desktop resource of the right type (section 7.5).

The first is done while the title records commands, in our command-list wrappers. The other two are settled at submission. The hard part is item 2's first question: **which textures can this submission reach?**

### 7.2 Draw reach
**Why it is hard.** A draw does not list its textures. It binds descriptor tables: pointers into descriptor heaps. Its root signature and shaders decide which slots it reads. Our game uses **unbounded descriptor tables**, which we discovered while testing an early "prepare at submit" design. That design found 40.6 million unresolved table references, because an unbounded table from a non-zero base could in principle reach any later slot in the heap.

**How we answer it.**
- While recording, our command-list wrappers keep the *binding state* of each list: root signature, pipeline, bound tables per root parameter, and the heaps set.
- At each draw, we resolve which (heap, slot range) pairs the bound shader can read. We use reflection where it is precise, and the declared ranges where it is not. Repeated draws with an unchanged binding revision are skipped.
- Each descriptor heap keeps **attention sets**: which slots currently name a tracked texture, and which slots are "unknown" (unregistered, dying, stale, or of a layout we cannot classify).
- At submission, the union over the submitted lists is the reach.

**Cost history of reach per direct submission:**

| Version | Cost per submission | Why |
|---|---|---|
| Whole-heap attention walk | about 5,235 µs | Visited about 21,600-24,500 *unknown* slots per submission, which belonged to only about 28-37 textures. |
| Per-heap (class, resource) table | about 324 µs | Same answers, fewer visits. A second reviewer re-derived every count and could not break it. |
| Per-texture index of unknown slots | **14.2 µs** mean (p99 under 63 µs) | Under our 30 µs budget. |

**Invariant.** Reach may over-include (copy something not needed). It must never under-include. Every shortcut is checked against the slow path in test builds by a *differential* mode: 14.18 million bundle checks and 5.99 million direct-draw checks with 0 mismatches before the fast path became the default.

### 7.3 Bundles: in-place reach
The title records many small bundles, at about one new bundle buffer per bundle, and executes them from direct lists. A bundle can set the pipeline, root signature and tables, and those settings are visible to the caller afterwards.

- **Default handling.** For each `ExecuteBundle`, compute the bundle's reach ranges from a cache and apply them to the caller's state. That made 13.5 million cache misses (and 5 hits) per run.
- **In-place handling.** Walk the bundle's recorded commands and replay only the *immutable metadata setters* (pipeline, tables, root parameters) into the caller's binding state. Resolve reach at each draw inside the bundle. **Never replay native commands.** Then restore the caller's topology and bump its binding revision exactly as before. A memo variant caches per-(binding revision, pipeline) results.
- **Result.** -1.5 to -3.5 ms per frame on the laptop, with 0 mismatches in the differential check.

**Lesson.** The cache that seemed obviously right missed almost always, because bundle buffers are recreated constantly. Measure the hit rate before trusting a cache.

### 7.4 Descriptor writes, "flat" bookkeeping and release-flip
To keep attention sets current, we observe every descriptor write: `CreateShaderResourceView` and friends into a heap slot, and `CopyDescriptors`. Our game makes millions of these.

**First version.** Tree maps per heap under one global mutex on the title thread:
- 30.2 million acquisitions in one run, 47.4 s of wait plus hold (4.2 ms per present);
- with everything else in the exact path switched off, this bookkeeping alone made the title run at 0.67 of the control's frame rate.

**Flat bookkeeping.** Per-heap flat arrays indexed by slot, with per-heap locking, and an early-out for writes that change nothing about tracked state. About 20 million writes per run took the lock-free early-out.
- Title-thread descriptor lock time dropped from **4.5 ms to 0.47 ms per present**.
- The side-effect-only cost went from 0.67 to **0.97** of control.
- Median frame on the laptop with the fix on went from 44.3 ms to 32.2 ms (control 22.4 ms).

A shadow mode builds the old maps in parallel and compares every answer: 0 flat mismatches. Flat bookkeeping is now the default.

The descriptor writes we *keep* still cost about 2.5 µs each (about 4.5 ms per present on the title thread in early builds), so their count matters.

**Release-flip.** When the title releases a resource, the exact path flips that texture's state to "dying" and reads the generation, in one hold of the texture lock. That keeps any later reach from treating a dying texture as current. We first got this wrong in a subtle way: a fast path that skipped the lock for plain buffers fired **0 times**. The real flag combination always reported "exact active", while the test fixture had stubbed that flag. Test with the real switch combinations.

**The release path in numbers** (AMD desktop):
- about 2,500 hooked `Release` calls per frame, 80 % of them for resources the exact path does not care about;
- each originally took the texture lock;
- the presenting thread took that lock about 1,400-1,450 times per frame (0.30 ms wait plus 0.27 ms hold).

A lock-free negative filter for untracked resources, plus a release fast path, cut the presenter's acquisitions from about 1,443 to 303 per frame (138.5 million filtered, 0 missed). This was a **real hitch win**: AMD desktop race-1 slow frames went from 98.2 to 85.5 per minute, and the handheld from 161.3 to 129.9 (-19 %).

### 7.5 Descriptor volatility
D3D12 root signature 1.0, which our game uses, treats table descriptors as **volatile**:
- the application may rewrite a descriptor after recording a list and before submitting it;
- the only forbidden window is "submitted and not finished executing";
- the data the descriptors point to is volatile in the same way.

(This is in Microsoft's public root signature version 1.1 documentation, which defines 1.0 behaviour as "descriptors volatile, data volatile".) vkd3d-proton honours this by treating every 1.0 descriptor as update-after-bind/bindless.

**Why this matters to a translator.** Any design that **copies a descriptor at record time** and binds the copy is not 1.0-conformant. The title may legally:
1. record a list using slot S;
2. rewrite S to point at texture B;
3. release texture A, the old content of S, because no submitted list references it;
4. submit.

A record-time copy still points at A, now destroyed: a use-after-free on the GPU. We had exactly such a copy in an NVIDIA-specific guard that rebinds tables to private copies. An audit at submit counted frozen slots that the title had `rewritten`, `changed`, `dead` or `dead-changed` since recording. The last one is the dangerous case. The legal fix is to **refill changed slots at submit**. No fence wait is needed: if the title changed a slot, no in-flight list can be reading it, or the title itself broke the rule.

**Caveats we found.**
- A slot that no shader references may legally change while a list is in flight. A refill that reads *every* slot can race there.
- Lists can be submitted twice. Refill only slots that changed.

### 7.6 Per-hook costs: none of them is the hitch
A sampled timing observer (1 in 64 calls) on the five draw-path hooks (flat-copy, default bundle, in-place bundle, draw reach and release-flip) measured them in normal and slow frames:

- Together they explain only **1.5-3 ms (15-25 %)** of the +9 to +12 ms of extra title-thread time in slow frames.
- The largest single difference is release-flip: +0.98 ms, with about **3x the calls** in slow frames. Slow frames are frames where the title releases a lot, which is streaming again.
- Named loader locks explain about 3 ms of the 12-13 ms the presenter spends blocked.

We switched off groups of hooks to price the whole side-effect cost on the laptop:

| Hook group off | Frames vs control |
|---|---|
| Barrier skipped | 0.54 |
| Draw recording off | 0.55 |
| Bundle reach off | 0.58 |
| Descriptor path off | 0.68 (the largest single item; it also slows the title's own code, probably through cache pressure) |
| All five hooks off | 0.85, so about 15 % lies outside the hooks entirely |

**Lesson.** Per-draw instrumentation costs grow with draw count, about 35-40 µs of extra cost per additional draw in early builds. Design draw-path hooks as table lookups, not searches, from the start.

---

## 8. Residency, lifetime and device loss

### 8.1 The lifetime mismatch
The title releases a resource when, *in console terms*, the GPU is done with it. The title tracks that with its own fences. Our translation adds objects the title cannot see:

- upload buffers;
- private copies of descriptor tables;
- desktop resources standing in for title VAs;
- in some modes, resources that live longer than their title counterparts.

So "the title released it" and "nothing on the GPU can touch it" are not always the same moment for *our* objects.

### 8.2 Deferred release
**What.** An optional retirement queue. A title `Release` that would destroy a desktop resource becomes a *retirement*:

- We hold credits on the resource. All intercepted references share one logical count: the native count minus our outstanding credits.
- Each retirement records a goal per queue: an epoch plus a fence value. The resource is destroyed only after **every registered queue** has certified completion past its goal.
- Placed resources and aliased heap ranges delay the release of their backing, so an early title heap release cannot free storage that a retained resource still uses.
- The collector is a worker thread. Pressure thresholds come from the DXGI local memory budget. **Retirement never makes a caller sleep.**

**Invariants.**
- A completion value, once assigned to a captured epoch, never moves. A later queue certificate must not move an older ticket's goal.
- `UINT64_MAX` from a fence is *device removed*, not "completed".
- An undrained batch that is destroyed **leaks on purpose** rather than release something the GPU may still use.
- If the bookkeeping is ever inconsistent, the process stops with a diagnostic file. A wrong release is worse than a crash with a report.

**Outcome.** As a *device-loss fix* it did not work (section 8.4), and it stays off by default. As infrastructure it is sound, and we would build it again earlier, with the invariants above.

### 8.3 Fences, and the GPU memory guard
- **Fence waits.** A signalled event is only a wake-up. The fence *value* is the authority. Every wait re-checks `GetCompletedValue` on timeout and maps `UINT64_MAX` to device removed. A wait with a null event handle is refused, because `SetEventOnCompletion` with a null event blocks until completion, possibly forever.
- **GPU memory guard (optional).** Wraps committed resource creation:
  - samples the DXGI budget every 250 ms and estimates what was admitted between samples;
  - keeps headroom of at least 256 MiB or 10 % of the budget, capped at 1 GiB;
  - under pressure, or after an out-of-memory error, places the resource in system memory instead.
  It never invents a resource, a fake success or a smaller image.
- **Budgets the console expected.** The game reads a console-sized texture budget. Raising or lowering the budget we report changed nothing visible: peak texture memory was about 1.3 GiB either way.

### 8.4 Device loss: what we learned
This is our longest-running open problem, so here are the parts that bear on memory.

**The signature.** NVIDIA, Vulkan path, desktop and laptop:
- an MMU read fault in the graphics engine, where every resource at the faulting address had already been destroyed;
- the faulting page always in the small-mip tail (last 4 KB) of streamed block-compressed textures that are created and destroyed repeatedly at one title VA;
- the vendor crash tool reported a texture fetch through a stale or wrong descriptor, into memory freed 5-389 s earlier;
- time to loss on the 4090 at normal speed: 195-430 s. A slow laptop configuration lost the device in 5 of 5 runs.

**Fixes that did not work.**

| Attempt | Result |
|---|---|
| vkd3d-proton's own deferred resource destruction | Still lost (314 s). |
| Our deferred release behind full queue completion | Failed 3 of 3 on the laptop. A 60 s hold made it worse (3 of 6). |
| A retention patch keeping memory alive longer | 0 of 20 losses with it against 0 of 3 without: unproven. Cost +265 MiB VRAM and +1.8 GiB RAM. |
| Typed null descriptors for empty slots | Lost at 374 s despite 1,086 substitutions. |
| An NVIDIA guard copying descriptor tables at record time | Hides a class of faults, but violates descriptor volatility (section 7.5). One of its sub-guards also omitted real world draws. |

**What we learned.**
- **Fence-based holds cannot fix a stale *descriptor*.** If a copy of a descriptor points at a dead resource, holding the resource longer only moves the fault. The bug is the descriptor, not the lifetime.
- **Crash tools can name the wrong resource when GPU addresses are reused.** "Freed minutes earlier" may partly be an artefact. Log your own address history and resolve faults against it.
- **Diagnostic layers change the odds.** Our timestamp layer was the prime suspect in one traced loss: the traced row lost the device and untraced rows did not. RenderDoc-on-Vulkan captures lost the device in 2 of 2 runs (section 12.3).
- **Correlations at small n mislead.** Eager staging was "on" in 5 of 8 lost runs and 0 of 17 clean ones. A paired 6-pair A/B then gave 1 of 6 losses either way.
- **Churn may multiply the risk without causing it.** On a sibling title, a survival correlated with a frame cap and about 25x fewer copy lists (4,000 against 101,000).

**What we'd change.** From day one, keep a ring of `{descriptor heap, slot, resource, generation, time}` writes and a ring of resource creations and destructions with GPU VAs. Dump both on device removal. Our late-added slot dump was the most useful single artefact for this class of bug. Check device removal regularly on submit paths (every sixteenth `ExecuteCommandLists`, plus every queue `Signal`), log it once, and dump everything you know.

### 8.5 Commit limits
On a 32 GB machine, the Windows commit limit (about 34 GB) invalidated test runs that peaked at 27 GB private. Several runs ended with "host commit exhausted", which looked like random crashes. Watch private bytes and commit charge as well as the working set. The translator's memory overhead (snapshots, tables, retained objects) can be gigabytes.

---

## 9. Performance lessons

### 9.1 The submit-time budget is microseconds
The most useful experiment of all was cheap. With the exact path **off**, we inserted a synthetic busy-wait in every direct `ExecuteCommandLists`:

| Busy-wait per submission | Effect |
|---|---|
| 15 ms or 5 ms | The game froze at the first track load. |
| 1 ms | **-42 %** presents. |
| 0.2 ms | **-14 %** presents. |

The title submits many times per frame, and some of its own logic waits on those submissions. That set our budget for reach at **30 µs per submission**, and told us that inline staging in the barrier would always show up in frame time. **Run this experiment first** on any title you translate.

### 9.2 Slow frames come in three classes
With every frame split into exclusive timed parts, and each slow frame labelled, slow frames on the AMD desktop fell into three classes:

| Class | Share of real slow frames | Defining feature |
|---|---|---|
| Streaming | about two-thirds | The barrier had at least one need; barrier CPU about 5.3 ms against 0.4 ms in ordinary frames. |
| Title submission burst | 13-19 per minute | 30 or more submissions in one frame (about 88 against a usual 15), and **zero** needs. |
| Zero-need | about 3 per minute | Plain heavier title frames. |

More findings:
- The burst threshold is robust: anywhere from 22 to 40 moves the count by at most 0.4 per minute. Not one burst frame had a need (0 of 495).
- **Bursts and streaming frames are one event.** 84-95 % of heavy streaming frames fall 1-10 frames after a burst, with a peak 11-16x the base rate at +3 to +5 frames. The title's streaming step is a burst of submissions, then frames that consume new textures.
- On the 15 W handheld, bursts were the *largest* class: of 91 real slow frames per minute, 55 were bursts, 25 streaming and 12 zero-need.
- **Only streaming frames respond to head-start work.** Any fix aimed at staging can at best touch two-thirds of slow frames on the desktop, and under a third on the handheld.
- **Ceiling.** Even perfect staging would cut slow frames by 56-72 % on the desktop, but those frames would still be about 31 ms. Slow frames sit just over the line: median excess 3.4-3.9 ms, p90 13-14.5 ms.

### 9.3 Average frames lie; frame populations differ
- Only about 40 % of a test run's frames were race laps (25,310 of 64,982 on one row). Menus, loading and replays had about 4x the slow-frame rate. **Always quote one defined stretch** (race laps), never a whole run.
- Our desktop presented with vsync on, which gave about 3 ms of slack in ordinary frames. Savings smaller than that are invisible to fps there and appear only in timers.

### 9.4 Your capture harness can be half your measurement
Our test harness took a route screenshot every 2 s to track progress. Every capture frame was over 33.4 ms: 159 of 159 in one race. That made about **30 of the 84 slow frames per minute** in the headline number. A real change of 10-20 % disappeared under it, and several earlier "no change" verdicts were made on the inflated numbers.

Since then:
- "real" slow frames exclude capture frames;
- every reader script must apply the exclusion (one of ours didn't);
- we accept that 0.4-1.9 real hitches a minute hide inside capture frames, in every arm alike.

Related trap: with the display off, presents throttled to about 4 Hz on one machine, and the slow starts this caused looked like a performance regression *and* a pop-in regression. Keep the display on, and record its state.

### 9.5 The head-start worker: throughput, not lead time
- A design review read worker CPU at about 12 % and concluded the worker was idle, so the limit had to be *lead time*.
- A second reviewer found that most of the "first view" needs behind that reading were *disarm-only re-checks*: neighbour tokens disarmed by another texture's retire (section 4.7). Those are drained by the barrier microseconds before its own take.
- In burst-adjacent frames, with 17 or more takes, one slow-waking worker was saturated.
- The lead-time histogram (section 5.5) settled it: most stolen jobs were queued 16 ms or more ahead.

**Lessons.**
- An "idle worker" average can hide saturation inside the frames that matter. Measure per frame class, not per run.
- Before blaming a queue's order, histogram how long before the need each item was queued.
- Remove false work (the retire-own fix) before reading what the remaining work tells you.

### 9.6 The first measured gain: a second worker plus eager staging
A second head-start worker *alone* cut stolen needs by 85 % and did not help: level on the AMD desktop, worse on the handheld. With eager staging as well, real slow frames per minute (capture frames excluded) were:

| Machine | Race 1 off → on | Race 2 off → on | Off/off spread |
|---|---|---|---|
| AMD desktop, first test | 52.3 → 40.2 (-23 %) | 61.5 → 50.7 (-18 %) | 1.9 / 2.7 |
| AMD desktop, confirmation | 53.8 → 42.7 | 65.5 → 54.8 | |
| Handheld at a measured 15 W, two passes | 106 → 98, and 88 → 70 | | |
| Handheld, frames over 50 ms | 15.3 → 12.8, and 15.8 → 11.6 per minute | | |

There was no fps cost, and the change went into the play configuration. The rule we keep: **the second worker only together with eager staging.** The second worker alone added 10-13 burst or zero-need slow frames a minute, and we never found out why.

Why removing stealing helped so little: each streaming slow frame saved about 1.8 ms of barrier time, but about 1.1 ms of that came back as extra *title* run time in the same frame. Frame-time savings are not additive when the title's own threads compete for the same cores and locks.

### 9.7 The burst lock
In burst slow frames the presenting thread is **blocked** for 16-17 ms:
- 6-10 ms is a contended title critical section;
- 2-3 ms is a title `WaitForSingleObject`.

At the same time, another title thread spends **10-16 ms per burst frame inside our `VirtualFree` hook**: about 71 calls at about 200 µs each. The tempting theory: a title thread holds that critical section while calling `VirtualFree`, and our slow hook stretches the hold.

What a second reviewer found:
- The correlation drops from 0.76/0.90 to 0.53/0.06 when *all* usable burst frames are counted, not a convenient subset.
- Two other title threads contend 42-61 times at 90-109 µs even in burst frames with **zero** `VirtualFree` calls.
- One presenter wait spans 7-9 hook calls.

The proposed fix was a *lean allocation path*: skip empty trap walks, reuse one allocation walk per free, and skip the protect-back. Its last part was **not exact**. It left a page's "restorer" set, so it could later un-protect a shared page that another token still armed: a silent trap miss. That part was dropped. The cheaper exclusion lookup, tested against the burst lock, shortened the other threads' `VirtualFree` hook time by only 5-20 % and did not shorten the presenter's wait. So the test was not decisive.

Our next instrument records the *owner* of a contended critical section and whether it is inside one of our hooks. Decide from data, not correlation.

**Lessons.**
- **Your hooks run inside the title's locks.** A hook that is "only" 200 µs becomes a 10 ms stall for whichever thread waits on the lock the caller holds. Make VM hooks O(change) and lock-light. Never do registry-wide walks there.
- Every "skip work" optimisation in the trap layer needs a written exactness proof and a sampled re-check in test builds. Ours re-walks every 64th skipped walk and raises a breach on a miss.

### 9.8 Measuring under Wine/Proton
- **Thread priorities do nothing.** Under Proton, every Windows thread of the process ran at nice 0, `SCHED_OTHER`, rtprio 0, and the process's maximum nice priority was 0. `SetThreadPriority` to below-normal, normal or a temporary boost had no effect on the Linux scheduler. Only Wine's own internal disk threads ran at nice 19. Any design lever that relies on priority is a placebo there unless the session gets a nice rlimit.
- **In-process `/proc` reads describe the wrong thread.** Our in-process wait sampler opened `/proc/thread-self/...` through Wine's Unix path mapping. Under Wine, regular files are opened by the wineserver, so "thread-self" resolved to **wineserver's** thread. The CPU, run-queue, syscall and wait-channel fields were silently wrong, with 0 open failures in 130 status rows. Use a **Linux-side sampler** that reads `/proc/<pid>/task/<tid>/...` from outside the process. Ours cost about 8 % of one core. It matched the in-process presenter CPU per frame exactly (14.90 against 14.90 ms, correlation 0.90).
- **`GetThreadTimes` ticks at 10 ms under Wine.** Totals over many frames are fine. Per-frame CPU-versus-blocked splits are not. The external sampler showed the slow-frame excess was about **14-15 ms CPU plus 9 ms blocked**, not mostly blocked as the 10 ms ticks had suggested.
- **Run-queue delay was negligible** (under 0.35 ms per frame in every class). The CPU was not starved. The cost was real work and real lock waits.
- **Timer resolution.** On Windows with the default timer resolution, a 34 ms `WaitForSingleObject` ticks at about 21.5 Hz, so a "30 fps floor" was really about 21 Hz. Wine's waits are precise.
- **Power is not what the firmware says.** The handheld's firmware power limits read 35/37/45 W, but the GPU's measured power in game was 15.0 W. Record *measured* power on every handheld run.
- **The VM lock and `VirtualQuery`** (sections 2.5 and 2.6). Wine-specific costs drove several of our features: snapshot pooling, our own SHA-256, the title memory map, and the Wine-dependent defaults for detection mode.

---

## 10. Worked examples

### 10.1 One texture's life, from title write to GPU copy
A composite of what the traces show for a streamed road texture with exact mode, traps, the head-start worker with refresh, eager staging and the lease all on. Slot numbers, generations and mip counts are illustrative; the sequence and the decisions are real.

```
t=0      Title VirtualAlloc(MEM_COMMIT) for a pool block
           hook: strip console flags, drop write-combine, commit (write-watched)
           ledger: +span {committed, RW};  epoch++ ; commit counter++
t=1      Title "places" a tiled texture at an address inside that block
           create desktop texture (DEFAULT, committed, zeroed), note {title VA, gen 41}
           register tiled state: 11 mips → 7 spans (big mips) + 1 packed tail span
           arm traps on committed spans (PAGE_READONLY); tail page shared with
           a neighbour → arm merge, no extra VirtualProtect
t=2      Title thread 3 memcpy()s mip 2 into title memory
           first store faults → handler: span 2 Armed→Handling→Dirty, page restored,
           event pushed, worker semaphore released; store retried, continues
t=2+δ    HS worker (eager): drain → mark span 2 (mark_seq=9)
           arm span 2 again, take lease (shared stripes), snapshot 64 KiB,
           SHA-256, detile into an UPLOAD block, publish ticket(seq=9, sig)
t=3      Title writes the rest of mip 2 (a second burst)
           fault again → span 2 Dirty (mark_seq=10) → refresh: wanted=10
           worker re-stages span 2 only; new ticket(seq=10)
t=4      Title creates an SRV in heap H slot 812 (flat bookkeeping: slot 812 → tex gen 41)
           Title lowers the view's most-detailed mip to 2 (mips 0-1 not committed yet)
t=5      Title records a draw that binds a table covering slot 812
           draw reach: (H, range) → attention set includes tex gen 41
t=6      Title ExecuteCommandLists on the direct queue
           barrier: drain (nothing new) → classify: span 2 ticket seq 10 == mark 10 → USED
                    lease signature recomputed == stored → accept
                    spans 3-10 trusted (landed after last mark) → nothing to do
                    mips 0-1 uncommitted → nothing to copy
           record: barrier→COPY_DEST, CopyTextureRegion(span 2), barrier back
           execute upload list, Signal(upload fence) → spans "landed"
           native ExecuteCommandLists(title lists)  ← GPU now sees current mip 2
t=7      Title streams mip 1: VirtualAlloc commit → hook marks Commit on mip 1 span
           ... same cycle as t=2..t=6; if the worker hasn't started it by the
           next barrier, the barrier STEALS it and stages inline (+0.1-0.3 ms)
t=8      Title releases the texture
           Release hook: release-flip (state → dying, generation read in one hold)
           retire-own: disarm only this texture's tokens (neighbour on the shared
           tail page stays armed and trusted)
t=9      Title VirtualFree(block, 0, MEM_RELEASE)
           hook: resolve size 0 → real allocation range; exclusive lease stripes;
           mark Free (pre and post); ledger −span; epoch++
           desktop texture freed when its references go (or retired, if deferred
           release is on)
t=10     Title places a NEW texture at the same address: gen 42
           interval map keeps gen 42 as the live entry at that base; any stale
           lookup that still carries gen 41 is classified OlderCompatible/Unknown
```

Each step has a counter. The checks we ran on every test run:
- trap misses 0;
- trusted-but-unarmed 0;
- snapshot-read breaches only on the speculative worker;
- "changed without mark" 0 in the audit;
- lease refusals counted.

### 10.2 One streaming slow frame, broken down
AMD desktop, exact path on, the second worker off, a race lap. The figures are typical values from the frame-split and Linux-side sampler rows. They come from separate instruments, so they do not add up exactly.

```
present interval ................................ ~39-40 ms   (ordinary frame ~14.8 ms)
  presenting thread on-CPU ...................... ~29-30 ms   (ordinary ~11-11.6)
  presenting thread blocked ..................... ~9.4-11.2 ms (ordinary ~3.2-3.7)
  run-queue wait ................................ ~0.3 ms     (CPU never starved)

inside the barrier (one or more submissions with needs)
  inline staging of stolen/missing spans ........ ~9.9 ms for ~60 spans (up to 13.7 ms / 73)
  upload submit (record + execute + signal) ..... ~2.8 ms
  barrier "rest" (classify, bookkeeping) ........ ~2.7 ms
  joins on running head-start passes ............ ~1.6-3.1 ms
  harvest / drain ............................... ~1 ms
  loader locks .................................. ~1.7-2.9 ms running + ~1-1.9 ms blocked

outside the barrier
  title's own code .............................. ~18.8 ms   (ordinary ~12.6 ms)
     → about +4-6 ms of title CPU we cannot yet attribute (cache pressure,
       our hooks' side effects, the title's own streaming work)
```

Reading it:
- The barrier is the biggest single block, so the head-start work helps.
- Almost as much again is extra *title* time outside the barrier. That is why perfect staging still leaves a ~31 ms frame (section 9.2).
- The next instrument has to go inside the title-side excess and inside the barrier's wait.

### 10.3 One burst slow frame, broken down
Same machine, same configuration.

```
present interval ................................ ~37-38 ms
  submissions this frame ........................ ~88   (usual ~15)
  barrier needs ................................. 0
  presenting thread on-CPU ...................... ~20.5-21 ms
  presenting thread blocked ..................... ~15.8-17.2 ms
     contended title critical section ........... ~6.2-9.8 ms
     title WaitForSingleObject .................. ~2-3.3 ms
meanwhile, another title thread
  inside our VirtualFree hook ................... ~10-16 ms (~71 calls × ~200 µs)
then, 3-5 frames later
  streaming slow frames (the textures from this step arrive) → section 10.2
```

Reading it:
- No staging work happens in this frame, so head-start changes cannot help it.
- The suspects are the title's own lock and the cost of our VM hooks under it. The ownership instrument (section 9.7) decides between them.

---

## 11. Observability: the instruments we built

Every instrument below is an **observer**. Each one's header states the same contract:
- it changes no decision, call, order, copy, fence or wait;
- it uses no lock, allocation or TLS on hot paths;
- it preserves `GetLastError`;
- when disabled, it costs one relaxed atomic load.

A profiler that perturbs the hitch it measures is useless. Ours did, several times, before we enforced this.

| Instrument (switch, as an example name) | What it measures | Cost and lessons |
|---|---|---|
| **Frame split** (`XBOX_EXACT_FRAME_SPLIT`) | Cuts each presenter frame into exclusive parts: title run, hooked waits, pipeline creation, trap faults, and the barrier's harvest, drain, enumerate, classify, wait, stage, submit and "rest". Attributes each slow frame's excess over the 16.7/33.4/50 ms lines to parts. | Exclusive parts that sum exactly matter more than precision per part. An unbracketed remainder is reported, never hidden. |
| **Stall split** (`XBOX_EXACT_STALL_SPLIT`) | Presenter CPU against blocked time, lock-acquire waits on each named loader lock, barrier need classes (used, stale, no job, stolen, missing), joins. Binary side file plus summary rows. | About 0.05 ms per frame on the desktop, 0.3-0.5 ms of CPU on the handheld. Its CPU split relies on `GetThreadTimes`, which ticks at 10 ms under Wine, so use it for totals only. |
| **Hook timing** (`XBOX_TEX_DRAW_EXACT_HOOK_TIMING`, test builds only) | Samples 1 in 64 calls of each draw-path hook. | No measurable cost. Showed that no single hook is the hitch (section 7.6). |
| **Create split** | Times every hooked create/alloc entry point as whole hook, native part and wrapper part, outermost call per thread only. | 0.34 → 0.39 ms per frame of presenter CPU. Found that our wrappers, not the kernel, own most VM hook time (section 2.6). |
| **Lock slow / lock census** | Per lock, per call site, per thread role: hold and wait times. | **Lesson:** the first version used a per-event ring (256 entries, with the presenter scanning 128 slots × 256 events per frame). It overwrote history and *doubled* slow frames (166 against 86-91 a minute). It also cost the handheld 6 fps. The second version keeps fixed per-thread counters only: added lock-pair cost 65.7 → 25.0 ns. |
| **Wait split** (`XBOX_EXACT_WAIT_SPLIT`) | An activity word for the presenting thread, plus a per-thread record of every hooked title wait, sleep, yield, I/O completion wait and VM call; a 1 kHz sampler thread reads the activity word. | The activity word and timed waits are valid. **Its in-process `/proc` reads are not valid under Wine** (section 9.8). The sampler costs about 11 % of one core. |
| **Linux-side sampler** | An external script that reads `/proc/<pid>/task/<tid>/{stat,schedstat,wchan,syscall}` for every thread at about 1 kHz and joins the result to the loader's frame marks. | About 8 % of one core. Became the standard for CPU-against-blocked splits on Linux. |
| **Lock owner** (`XBOX_EXACT_LOCK_OWNER`) | On a contended title critical section: the owning thread, and whether it is inside one of our hooks. | Built to settle the burst lock question (section 9.7). |
| **Lead histograms** (`XBOX_TEX_DRAW_EXACT_MEASURE`) | For each need, how long before it the bytes became final and the job was queued, by cause. | This one ended the "ordering" theory (section 5.5). |
| **Thread CPU table** | Helper threads register themselves; per-thread CPU every 5 s. | Cheap, and catches runaway workers. |
| **Memory meta rows** | Private bytes, working set, peak, DXGI local usage and budget, plus the cost of the query itself. | Replaces ad-hoc queries; never adds a second reader. |
| **Object ownership census** (`XBOX_MEMORY_OWNERSHIP`) | Live native objects by kind (committed, placed, heap, descriptor, pipeline, upload, title-committed, title-imported). | Uses a COM private-data sentinel as a token, so no vtable hooks are needed. |
| **Switch census** | At startup, the effective value and source of every switch: default, profile or environment. | Removes "which build/config was this?" from every investigation. |
| **Trap attribution** (observation only) | Walks the faulting thread's call chain, outside any lock, to say which module family wrote trapped pages. | Useful once per title to learn its writer classes. |
| **Debug fills** (`XBOX_TEX_UNRESIDENT_FILL`) | Fills unuploaded data with magenta checkers at registration time. | Not timing-neutral. Cannot cover render targets, depth or UAVs. Undercounts if scored by pixel percentage. |
| **GPU timestamp layer** | Section 12.1. | |

**Habits that made these useful.**
- Every run prints a **health block**: trap counters, breaches, lease refusals, device-loss checks and the switch census. Performance numbers from an unhealthy run are discarded.
- Rows are **machine-readable** and versioned (`schema=1`). Reader scripts are kept with the data.
- Diagnostics that cost real time are **test-build only**. Play builds compile them out, so they cannot leak into players' frame time.

---

## 12. GPU-side profiling: what worked and what didn't

### 12.1 A Vulkan timestamp layer (worked)
Under vkd3d-proton everything becomes Vulkan, so a small **explicit Vulkan layer**, enabled only by environment variables, can time the GPU without admin rights or GPU performance counters.
- It writes timestamps at the start and end of every `vkQueueSubmit` batch and every command buffer.
- It maps presents, CPU submit times and the loader's frame marks onto the GPU clock.

**Results on the RTX 4090:**
- the GPU was busy only **15-17 %** of each frame traced (about 17-18 % untraced);
- most of the idle came *before* any work had been submitted: 8-10 ms of idle before the frame's first list of 0.2 ms or more;
- the exact path added no GPU work, but at lap start it made that idle gap 2.3 ms longer.

**Costs and cautions.**
- About 19 % fps in CPU-bound stretches. GPU times are real; frame times and idle shares are inflated.
- A traced run lost the device and the matching untraced runs did not. Keep a "batch begin/end only" light mode, and never trust a device-loss result from a traced run.

### 12.2 Radeon GPU Profiler on AMD Linux (worked, with limits)
RADV, Mesa's Vulkan driver, can write RGP thread traces through its built-in capture. We set the trace mode to RGP with a trigger file, a large trace buffer and queue events on. A small watcher touched the trigger at chosen moments of the run, for example 6 s and 14 s into each lap, to get one-frame captures.

- **Headless:** the trace file's chunks can be parsed for queue event timings and the GPU clock. That gives per-command-buffer GPU start and end per captured frame, with no GUI.
- **Per draw:** the SQTT token stream (per draw, barrier and layout transition) has no open decoder we could find. Per-draw and per-pass cost needs the RGP GUI.
- **Distortions:** capture slows the run a lot (about 44 → 21 fps), and RADV fixes a stable GPU clock state during capture. GPU busy time is valid like-for-like; CPU-side idle is inflated.

**Findings on the handheld.**
- Fix on and fix off had the same GPU busy time (22.6 against 22.5 ms) and the same command buffer count (50 against 51).
- The difference was a **9-26 ms GPU-idle gap before the title's first big render list** with the fix on.
- Lap starts carried 60-263 tiny upload command buffers per frame in both arms. Those are upload storms.

All of this said: **the GPU is not the problem; the CPU path to the GPU is.**

### 12.3 RenderDoc (useful per draw, but has pitfalls under vkd3d-proton)
RenderDoc gives per-draw timing and pixel history in a capture, which no other tool gave us. Under vkd3d-proton we hit all of these problems:

- **It changed the reported GPU vendor.** With RenderDoc loaded, our loader saw a different vendor, and our vendor-specific guards switched themselves off. The captured frames therefore ran *a different code path* from the one with the bug. Pin vendor-dependent policy explicitly in capture runs, or log it in every row.
- **It captured the wrong frames.** Presents counted with capture active did not line up with the moments we meant. Several captures were of the menu before the race, not the frames with the bug. Each captured frame took about 5.9 s. Trigger captures from an in-game event you log, not from a present number planned in advance, and check a thumbnail before analysing.
- **Replay failures with ray tracing enabled.** vkd3d-proton allocates all device memory with device-address support when DXR is requested. Captures then failed to replay with `VK_ERROR_INVALID_OPAQUE_CAPTURE_ADDRESS` on the same machine and driver. Disabling DXR in vkd3d-proton for the capture run is the thing to try.
- **Device losses.** Two of two Vulkan-level capture runs lost the device during capture.
- **Which layer you capture.** Calling the in-application capture API with null device and window captured the D3D12 layer *above* vkd3d-proton. Replaying that needed the matching D3D12 redistributable and developer mode. To capture at the Vulkan level, hand RenderDoc vkd3d-proton's `VkInstance` and device, which vkd3d-proton exposes through its device extension interface.
- **Small traps.** Over an elevated remote shell on Windows, the Vulkan loader ignores layer environment variables, so the capture layer never loads. RenderDoc's Python host blocked on its first-run analytics prompt until that was answered in its config.

### 12.4 NVIDIA tools
- **Nsight Graphics GPU Trace** fails with "GPU performance counters unavailable" unless counter access is allowed for all users. The NVIDIA Control Panel calls this *developer settings → manage GPU performance counters*. The default is admin-only. That is a machine-level setting for the owner to change, not something to script past.
- **Nsight Systems** needs admin to install and run on Windows.
- **PresentMon** and similar ETW-based frame tools need admin, or membership of the Performance Log Users group.
- **vkd3d-proton's queue profiling** is CPU-side only.

The timestamp layer (section 12.1) was our only option that avoided all of these.

### 12.5 What to take from this
- Build a **GPU busy against wall clock** measurement on day one. It tells you whether to work on the GPU or the CPU. For us it was CPU, by a wide margin, on every machine.
- Per-command-buffer timing is enough for most decisions. Use per-draw tools only once you know which pass to look at.
- Every capture tool changes timing, and some change *behaviour* (vendor ID, memory allocation flags). Write down what each changes before trusting a capture.

---

## 13. Measurement discipline we learned the hard way

1. **Two passes per arm, always, plus an off-against-off pass.** Quote the spread next to every delta. Our noise on slow frames per minute was often 2-8 a minute, as large as many "improvements".
2. **Define the stretch:** race laps only, with a lap-progression check. One run had a stuck car that passed a simple "reached lap 1" test.
3. **Exclude your own capture frames** (section 9.4), and make the exclusion live in shared reader code.
4. **Record the machine's state:**
   - display on or off;
   - measured power, not the firmware limit;
   - remote desktop software (one remote-access tool on the 4090 cut a control from 88 fps to 57);
   - which GPU a hybrid laptop actually used (some early laptop runs were on the integrated GPU);
   - commit limit headroom.
5. **Separate frame classes** before averaging (section 9.2).
6. **Per-frame CPU splits need fine clocks.** Under Wine use an external sampler (section 9.8).
7. **Test builds are not play builds.** Some exactness checks ran only in test builds and cost time play builds never pay. The reverse also happened: a per-span log row in test builds alone (85,504 rows in one present, written under the barrier's locks) caused a 12-13 s level-load freeze that play builds never had.
8. **Promote a switch to default only after it passes a recorded gate.** Our play profile table records, for each switch, the gate it passed. For example, cached title memory: "4090 hitches about 27 → about 1 per minute, audit clean". An explicit environment value always overrides.
9. **Eyes over scorers** for visual correctness (section 6.4).

---

## 14. What changes for an Xbox Series X translator

We have not translated a Series X title. This section combines what we learned with Microsoft's *public* descriptions of the Series X|S platform, from its 2020 announcements and architecture talks and the public DirectX 12 Ultimate and DirectStorage documentation. Verify every point against your own title.

### 14.1 Platform differences that matter for memory (public information)
- **No ESRAM.** The Series X has 16 GB of GDDR6, described publicly as 10 GB of faster "GPU-optimal" memory and 6 GB of slower "standard" memory, still one unified address space. Expect the title to *place* GPU-heavy resources in the fast range. The PC does not care, but address ranges and reservation patterns will differ from the Xbox One's, so your startup holds (section 2.1) need the title's real ranges.
- **More memory, more objects.** Every per-span, per-page and per-slot structure we built grew with memory size. Design them O(change) from the start. Our registry walks became visible at about 111,000 spans and about 22,000 textures.
- **DirectStorage and the hardware decompression block** (zlib and the console's BCPack texture codec, per the public material). **This is the biggest change to write detection.** Data can land in title memory decompressed by hardware, written by no CPU store. Page traps will not see it, and write-watch may not either, just as kernel file reads did not for us (section 4.6). Treat the storage and decompression completion as a *producer event*: an I/O ledger keyed by destination range, marked at request and completion, as our file-read ledger was. You will also need a CPU or GPU BCPack decoder, since PC GPUs do not sample BCPack.
- **Sampler Feedback Streaming and tiled resources.** Series titles are publicly expected to use sampler feedback plus reserved (tiled) resources: map only the tiles the GPU samples, and use GPU-written feedback maps to drive streaming. That changes the residency model:
  - residency becomes **per 64 KiB tile**, through `UpdateTileMappings`, not per page of a placed resource;
  - you must translate tile-mapping calls onto vkd3d-proton's sparse resources, and decide what an *unmapped* tile reads. A D3D12 tiled-resources tier 2+ unmapped read returns zero, much like our zeroed mips;
  - the "exact" contract becomes: every *mapped* tile the title wrote has current content before the submission that samples it;
  - feedback maps are GPU-written. Read them back on the GPU timeline, not by CPU polling.
- **DirectX 12 Ultimate features:** mesh shaders, DXR, variable rate shading and sampler feedback, with root signature 1.1 more likely. More GPU addresses appear in more places:
  - raw buffer pointers in root constants;
  - acceleration structure build inputs;
  - indirect argument buffers.
  Address translation (section 3.1) must cover all of them. Some, like addresses inside GPU-written buffers, cannot be translated on the CPU at all. That argues for **identity mapping of GPU VAs** where you can get it, for example by placing desktop resources at addresses you control through placed heaps and a fixed GPU VA allocator, rather than a lookup.
- **Bindless is the norm.** Modern titles index large descriptor heaps from shaders. Our reach problem (section 7.2) becomes "the whole heap is potentially reachable". The per-texture unknown-slot index, or a resource-level "dirty since last submit" set with no reach at all, is the design to start from. Root signature 1.1 *static* descriptors, if used, actually make record-time decisions legal, so check the flags per table.

### 14.2 What carries over unchanged
- Reserving the title's ranges before the driver starts, and relaunching on a lost hold.
- Hooking every VM path, including `GetProcAddress` bypasses, kernel writers and size-zero frees.
- Generations on every address.
- The submission-equivalence contract and its rejected alternatives.
- The fault-handler rules: no allocation, no static TLS, fixed lock order.
- Leases while reading memory that the title can unmap.
- Descriptor volatility under root signature 1.0.
- Wine/Proton cost facts: VM lock, `VirtualQuery`, priorities, `/proc`, timer resolution.
- The 1 ms submission experiment.
- Observer-only instrumentation and the GPU timestamp layer.

---

## 15. If we started again: day-one list

**Architecture**
1. Write the correctness contract (section 5.1) and the list of things it forbids before writing code.
2. Reserve all title ranges at process start. Log every reservation result and relaunch on failure.
3. One hook layer for every VM change (alloc, free, protect, query, `GetProcAddress`, heap allocator, I/O completions, DMA and decompression completions). One mutation lock, pre- and post-marks, `GetLastError` preserved.
4. A title memory ledger as the only source of "what is this memory", with a debug compare against the OS.
5. Generation-tagged identity for every resource, descriptor and address lookup.
6. Page traps as the primary writer detector, a second detector in test builds, and an explicit producer ledger for every non-CPU writer.
7. Leases for any read of title memory the title can unmap concurrently.
8. Copies on the same queue, ahead of the title's lists. No fence waits, no placeholders.
9. A staging worker pool with refresh semantics and eager start, sized from measured throughput. Under Proton, more workers, never higher priority.
10. Descriptor handling that is volatility-correct: refill changed slots at submit, never freeze record-time copies.
11. Fail-closed lifetime code: leak rather than free early, and stop with a report rather than guess.

**Costs**

12. Budget the submit path at tens of microseconds. Run the busy-wait experiment on day one.
13. Every hook O(change). No registry-wide walks, ever, on a path the title can call thousands of times.
14. Pool every scratch allocation. Under Wine every `VirtualAlloc`/`VirtualFree` takes a process-wide lock.
15. Use cached pages for title memory you will read back. Measure write-combined read cost on each OS.
16. Check what your platform's crypto and system libraries actually do. Ours lacked SHA instructions under Wine.

**Observability**

17. Observer-only instruments with exclusive frame splits, a health block in every run, and a switch census at startup.
18. Frame classes from the start: streaming, burst, other.
19. An external Linux sampler for CPU against blocked time, a GPU timestamp layer for busy against idle, and descriptor/resource history rings dumped on device removal.
20. A "tried / ruled out" table for every problem, updated after every result.

**Process**

21. Two passes per arm plus off-against-off, quoted spreads, defined stretches, harness frames excluded, machine state recorded.
22. A second reviewer for every non-trivial exactness or performance claim, whose job is to break it. Our reviewers caught wrong attributions, a refuted numbering rule, a non-exact "lean" path and a bundle shortcut that would have brought black textures back.
23. Eyes on the screen before calling anything fixed.

---

## Appendix A. Switch names used as examples
These are our environment switches, named here only to make the mechanisms concrete. Code defaults are mostly off, and a play profile turns on the proven set.

| Example switch | Mechanism |
|---|---|
| `XBOX_TEX_DRAW_EXACT` | The exact texture path (sections 4-5). |
| `XBOX_TEX_DRAW_EXACT_DETECT=trap\|watch` | Writer detection mode. |
| `XBOX_TEX_DRAW_EXACT_ARM_MERGE` | Skip re-protecting pages already armed by neighbours. |
| `XBOX_TEX_DRAW_EXACT_RETIRE_OWN` | Retire disarms only the texture's own tokens. |
| `XBOX_TEX_TRAP_EXCLUSION_FAST` | Exact-key exclusion lookup instead of a window walk. |
| `XBOX_TEX_SNAPSHOT_LEASE` | Striped mapping leases for snapshot reads. |
| `XBOX_TEX_SNAPSHOT_POOL` | Reuse snapshot reservations. |
| `XBOX_TEX_HASH_FAST` | Own SHA-256 with SHA instructions. |
| `XBOX_TEX_DETILE_FAST` | Learned detile run tables, verified against the library. |
| `XBOX_TEX_UPLOAD_BATCH` | Batched upload submission, capped. |
| `XBOX_TEX_DRAW_EXACT_HS_REFRESH` | Head-start refresh semantics. |
| `XBOX_TEX_DRAW_EXACT_HS_EAGER` | Eager head-start staging. |
| `XBOX_TEX_HS_WORKER_BOOST=2` | Second head-start worker, used only together with eager staging. |
| `XBOX_TEX_STAGE_EARLY` | Hot/cold head-start queues (no gain). |
| `XBOX_TEX_DRAW_EXACT_STAGE_FANOUT` | Fan inline staging out to helper threads. |
| `XBOX_TEX_DRAW_EXACT_FLAT` | Flat descriptor bookkeeping. |
| `XBOX_TEX_DRAW_EXACT_BUNDLE_INPLACE` | In-place bundle reach. |
| `XBOX_TEX_READ_MAP` | Title memory map instead of `VirtualQuery`. |
| `XBOX_TEX_SETTLE_WAKE` | Event-driven re-check of waiting textures (pre-exact design). |
| `XBOX_TEX_UNRESIDENT_FILL` | Debug fill for missing data. |
| `XBOX_DEFER_RELEASE` | Fence-certified deferred release. |
| `XBOX_TITLE_VA_RELAUNCH` | Relaunch when a title range hold is lost. |
| `XBOX_BUFFER_TIGHT_ALIGN` | Tighter small-buffer alignment through a vkd3d-proton private flag. |
| `XBOX_EXACT_FRAME_SPLIT`, `XBOX_EXACT_STALL_SPLIT`, `XBOX_EXACT_WAIT_SPLIT`, `XBOX_EXACT_LOCK_OWNER`, `XBOX_TEX_DRAW_EXACT_HOOK_TIMING`, `XBOX_TEX_DRAW_EXACT_MEASURE` | Observers (section 11). |

## Appendix B. Numbers worth remembering
| Fact | Value |
|---|---|
| Resource creates by the title | about 1,000 per second |
| Hooked `Release` calls | about 2,500 per frame |
| Submit-time busy-wait that costs 42 % of presents | 1 ms |
| Reach budget per direct submission | 30 µs (achieved 14.2 µs mean) |
| `VirtualQuery` under Wine (sibling title) | about 250 µs per call |
| VirtualFree / VirtualAlloc hook (exact on, Proton) | about 211 / 123 µs per call |
| Write-combined CPU reads, Windows | 3.2-3.8 ms per MiB |
| Write-watch harvest at its worst | 76 million `GetWriteWatch` calls in 600 s |
| Exact-path GPU work added | about 0 (3.1 vs 3.3 ms per frame) |
| GPU busy share on the RTX 4090 | 15-20 % |
| Stolen jobs queued 16 ms or more ahead | about 90 % |
| Second worker plus eager staging | -18 to -23 % real slow frames (AMD desktop) |
| Harness capture frames in the headline | about 30 of 84 slow frames per minute |
| Lost title-range holds on native Windows | about 1.85 % of launches |
