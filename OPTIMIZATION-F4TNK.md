# 🛰️ gr-osmosdr — Optimization F4TNK

> **Branch** : `master-f4tnk`  
> **Base** : gr-osmosdr (osmocom)  
> **Station** : SatNOGS #3762  
> **Summary** : **4 modified files** — **3 optimization rounds** — 12 optimizations
> **Target** : AirSpy + SoapySDR backends for weak-signal satellite reception

---

## 📊 Overview — SDR Acquisition Chain

```mermaid
flowchart LR
    subgraph HW["🎛️ Hardware"]
        A["Airspy R2<br/>12-bit ADC"]
    end

    subgraph DRV["📟 Drivers"]
        B["airspyone_host<br/>(libairspy)"]
        C["SoapyAirspy"]
        D["SoapySDR"]
    end

    subgraph OSMOSDR["📡 gr-osmosdr (this repo)"]
        E["airspy_source_c<br/>Native backend"]
        F["soapy_source_c<br/>SoapySDR backend"]
    end

    subgraph GR["⚙️ GNU Radio"]
        G["doppler_correction_cc"]
        H["frame_decoder"]
        I["Satellite decoders"]
    end

    A --> B --> C --> D --> F
    A --> B --> E
    E --> G
    F --> G
    G --> H --> I

    style HW fill:#1a1a2e,stroke:#e94560,color:#fff
    style DRV fill:#16213e,stroke:#0f3460,color:#fff
    style OSMOSDR fill:#0f3460,stroke:#533483,color:#fff
    style GR fill:#533483,stroke:#e94560,color:#fff
```

---

## 🏗️ Modification Architecture

```mermaid
mindmap
  root((gr-osmosdr<br/>F4TNK))
    airspy_source_c
      FIFO 5M → 10M samples
      Bulk insert callback
      memcpy array_one/array_two
      Partial work return
      Timed wait 100ms
      Force FLOAT32_IQ
      USB bit packing ON
      set_output_multiple 1024
    soapy_source_c
      MTU-aware reads
      Stream args forwarding
      3-retry overflow recovery
      Timeout handling
      Anti-alias 0.75 → 0.80
      set_output_multiple 4096
    CMakeLists.txt
      march=native
      ffast-math
      ftree-vectorize
      flto
```

---

## ⚡ AirSpy Backend Optimizations

### 1. Expanded FIFO — 5M → 10M samples

```mermaid
flowchart LR
    subgraph AVANT["❌ Before: 5M samples"]
        A1["~1s buffer<br/>at 2.5 MSPS"]
        A1 --> A2["Frequent overflow<br/>on CPU spikes"]
    end

    subgraph APRES["✅ After: 10M samples"]
        B1["~2s buffer<br/>at 2.5/6 MSPS"]
        B1 --> B2["Absorbs CPU /<br/>scheduler spikes"]
    end

    style AVANT fill:#e74c3c,stroke:#333,color:#fff
    style APRES fill:#2ecc71,stroke:#333,color:#fff
```

### 2. USB Callback — Bulk Insert

```mermaid
sequenceDiagram
    participant USB as libairspy USB
    participant CB as airspy_rx_callback
    participant FIFO as circular_buffer

    Note over CB: ❌ Before: push_back loop × N
    USB->>CB: float* samples (IQ interleaved)
    loop Each sample
        CB->>FIFO: push_back(gr_complex(I, Q))
    end

    Note over CB: ✅ After: bulk insert
    USB->>CB: gr_complex* src (direct cast)
    CB->>FIFO: insert(end, src, src+N)
    Note over FIFO: Single call,<br/>internally optimized copy
```

**Why it works**: FLOAT32\_IQ guarantees `{float I, float Q}` contiguous in memory — identical to `gr_complex` (`std::complex<float>`) layout. The cast is safe and eliminates the loop.

### 3. work() — Partial Return + memcpy

```mermaid
stateDiagram-v2
    [*] --> WaitData

    state WaitData {
        [*] --> Check
        Check --> Timeout: FIFO empty > 100ms
        Check --> HasData: FIFO not empty
        Timeout --> [*]: return 0
    }

    HasData --> Clamp: n = min(available, noutput_items)

    state Clamp {
        [*] --> Segment1
        Segment1: memcpy(out, array_one)
        Segment1 --> FullyCopied: arr1 ≥ n
        Segment1 --> Segment2: arr1 < n
        Segment2: memcpy(out+arr1, array_two)
        Segment2 --> FullyCopied
    }

    FullyCopied --> EraseBegin: erase_begin(n)
    EraseBegin --> [*]: return n
```

| Aspect | Before | After |
|--------|--------|-------|
| Wait | Blocks indefinitely | 100ms timeout (anti-hang) |
| Return | Exact `noutput_items` required | Partial (whatever is available) |
| Copy | `for` loop `at(i)` + `pop_front()` | `memcpy` × 2 segments max |
| Pipeline latency | High (waits for full buffer) | Low (immediate return) |

### 4. Force FLOAT32\_IQ + USB Bit Packing

```mermaid
flowchart TD
    A["airspy_set_sample_type<br/>FLOAT32_IQ"] --> B["Safe gr_complex* cast"]
    C["airspy_set_packing(1)"] --> D["4×12-bit → 3×16-bit"]
    D --> E["↓ 25% USB bandwidth"]
    E --> F["Fewer drops on RPi/USB2"]

    style B fill:#2ecc71,color:#fff
    style F fill:#2ecc71,color:#fff
```

---

## ⚡ SoapySDR Backend Optimizations

### 5. MTU-Aware Reads

```mermaid
flowchart LR
    A["getStreamMTU()"] --> B{"MTU > 0 ?"}
    B -->|Yes| C["toRead = min(noutput, MTU)"]
    B -->|No| D["toRead = min(noutput, 65536)"]
    C --> E["readStream(toRead)"]
    D --> E
    E --> F["Reads aligned<br/>to hardware buffers"]

    style F fill:#2ecc71,color:#fff
```

### 6. Stream Args Forwarding

```mermaid
flowchart TD
    A["Device args string"] --> B{"buflen= ?<br/>buffers= ?"}
    B -->|Present| C["SoapySDR::Kwargs<br/>streamArgs"]
    C --> D["setupStream(CF32, channels, streamArgs)"]
    B -->|Absent| E["setupStream(CF32, channels)"]

    style D fill:#3498db,color:#fff
```

Allows configuring the size and number of USB buffers directly from the GNU Radio device string:
```
driver=airspy,buflen=262144,buffers=8
```

### 7. Overflow Recovery + Timeout

```mermaid
flowchart TD
    A["readStream()"] --> B{"Result ?"}
    B -->|OVERFLOW| C{"retries > 0 ?"}
    C -->|Yes| D["Retry (flag reset)"]
    D --> A
    C -->|No| E["return last ret"]
    B -->|TIMEOUT| F["return 0<br/>(GNU Radio retries)"]
    B -->|Error| G["return 0<br/>(soft recovery)"]
    B -->|OK ≥ 0| H["return ret samples"]

    style H fill:#2ecc71,color:#fff
    style F fill:#f39c12,color:#fff
    style G fill:#e74c3c,color:#fff
```

**Before**: 1 retry on overflow, no timeout handling  
**After**: 3 overflow retries + proper timeout → no blocking

### 8. Anti-Alias Bandwidth 0.75 → 0.80

```mermaid
flowchart LR
    subgraph AVANT["0.75 × Fs"]
        A1["Fs = 2.5 MSPS"]
        A1 --> A2["BW = 1.875 MHz"]
        A2 --> A3["Signal loss<br/>at band edges"]
    end

    subgraph APRES["0.80 × Fs"]
        B1["Fs = 2.5 MSPS"]
        B1 --> B2["BW = 2.0 MHz"]
        B2 --> B3["Better tradeoff<br/>signal vs aliasing"]
    end

    style A3 fill:#e74c3c,color:#fff
    style B3 fill:#2ecc71,color:#fff
```

LEO satellite signals are narrowband at center — 80% offers a better tradeoff than 75% which was cutting too much.

---

## 🛰️ LEO Satellite Optimizations (Round 3)

### 9. rx\_time Tag PMT — Hardware Timestamps to GNU Radio

`timeNs` was read by `readStream()` but never exploited. Now each successful call
emits an `rx_time` tag in UHD-compatible format:

```
pmt::make_tuple(pmt::from_uint64(full_secs), pmt::from_double(frac_secs))
```

This tag is natively read by `gnuradio/blocks/tagged_file_sink`, `file_meta_sink`,
and any downstream block that aligns on physical time.
It carries the timestamp from our **SoapyAirspy Mod 23** (steady\_clock ns).

```mermaid
sequenceDiagram
    participant HW as AirSpy hardware
    participant SA as SoapyAirspy mod 23
    participant GR as soapy_source_c mod 9
    participant DS as Downstream GR blocks

    HW->>SA: USB callback
    SA->>SA: steady_clock::now() → _buf_timestamps[]
    SA->>GR: timeNs = _buf_timestamps[handle]
    GR->>DS: add_item_tag("rx_time",\nmake_tuple(full_secs, frac_secs))
    DS->>DS: Aligns demodulation / recording
```

**File:** `soapy_source_c.cc`

### 10. Forwarding Settings — Device String to writeSetting()

Before, only `buflen`/`buffers` were forwarded from the device string. Now
`sensitivity_gain`, `linearity_gain`, `ppm`, `biastee`, `bitpack` are also forwarded
to `writeSetting()` — **compatible with SoapyAirspy mods 16-18**.

```bash
# Before — these args were silently ignored by gr-osmosdr
soapy=0,driver=airspy,sensitivity_gain=15,ppm=1.2

# After — forwarded directly to the driver
# SoapyAirspy::writeSetting("sensitivity_gain", "15") is called
```

**File:** `soapy_source_c.cc`

### 11. \_\_builtin\_prefetch — Cache Warming

```mermaid
timeline
    title USB callback timeline (before / after)
    Before lock : [wait lock] → [cold copy]
    After lock  : [prefetch L2 stride 256B] → [lock] → [warm copy]
```

| Point | Prefetch | Locality | Reason |
|-------|----------|----------|--------|
| `rx_callback` | `src` stride 256B | L2 (`_MM_HINT_T1`) | USB data ~24KB, > L1 |
| `work()` | `out` write | L1 (`_MM_HINT_T0`) | Immediate destination |

**File:** `airspy_source_c.cc`

### 12. Atomic Overflow Counter

The old code printed `"O"` to stderr on each overflow — inaccessible flood.

```cpp
// Before
if (to_copy < num_samples)
    std::cerr << "O" << std::flush;  // flood, no counting

// After
std::atomic<uint32_t> _overflow_count;
// ...
const uint32_t cnt = ++_overflow_count;
if (cnt % 100 == 1)
    std::cerr << "AirSpy FIFO overflow #" << cnt << std::endl;
```

| Before | After |
|--------|-------|
| Print `"O"` on each overflow (flood) | Log every 100 |
| No counting | Cumulative atomic counter |
| Invisible in SatNOGS | `#N` visible in client logs |

**File:** `airspy_source_c.h`, `airspy_source_c.cc`

---

## 🔧 Build — CMake Flags

```mermaid
flowchart TD
    subgraph FLAGS["Compilation Flags (with guards)"]
        F1["-march=native<br/>Native CPU instructions"]
        F2["-ffast-math<br/>Relaxed IEEE → SIMD"]
        F3["-ftree-vectorize<br/>Auto-vectorization"]
        F4["-flto<br/>Link-Time Optimization"]
    end

    F1 --> CHECK1{"CheckCXXCompilerFlag"}
    F2 --> CHECK2{"CheckCXXCompilerFlag"}
    F3 --> CHECK3{"CheckCXXCompilerFlag"}
    F4 --> CHECK4{"CheckCXXCompilerFlag"}

    CHECK1 -->|OK| ADD1["add_compile_options"]
    CHECK2 -->|OK| ADD2["add_compile_options"]
    CHECK3 -->|OK| ADD3["add_compile_options"]
    CHECK4 -->|OK| ADD4["add_compile_options<br/>+ linker flags"]

    style FLAGS fill:#f39c12,stroke:#333,color:#fff
```

Each flag is tested before activation → cross-compiler compatible.

---

## 📈 Estimated Impact

```mermaid
xychart-beta
    title "Gain per Optimization"
    x-axis ["FIFO x2", "Bulk insert", "memcpy work", "USB packing", "MTU reads", "3x retry", "Partial ret"]
    y-axis "Signal impact" 0 --> 10
    bar [3, 5, 7, 4, 6, 3, 8]
```

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Buffer overflow (2.5 MSPS) | Frequent on RPi | Rare | ~10× fewer |
| Pipeline latency | 100+ ms | < 10 ms | Partial return |
| USB bandwidth | 100% | ~75% | Bit packing |
| Callback CPU/sample | ~4 ops | ~0.5 ops | Bulk insert |
| GR Timestamp | Absent | `rx_time` PMT tag | Mod 9 |
| Device string gain | Ignored | `writeSetting()` | Mod 10 |
| Overflow log | flood `O` | `#N` every 100 | Mod 12 |

---

## 📋 Commit History

| Commit | Description |
|--------|------------|
| `f60a1b9` | **Round 1**: MTU reads, FIFO 10M, bulk insert, memcpy work(), CMake flags |
| `56cb6ac` | **Round 2**: Force FLOAT32\_IQ, USB packing, partial return, timed wait, anti-alias 0.80 |
| `8e17305` | **Round 3**: rx\_time PMT tag, settings forwarding, prefetch L2, atomic overflow counter |

---

## 📂 Modified Files (4 files)

```mermaid
pie title Modification Distribution
    "airspy_source_c.cc" : 88
    "soapy_source_c.cc" : 53
    "CMakeLists.txt" : 21
    "soapy_source_c.h" : 1
```

---

## 📖 Usage

```bash
# Clone
git clone -b master-f4tnk https://github.com/f4tnk/gr-osmosdr.git

# Build
cd gr-osmosdr && mkdir build && cd build
cmake .. -DCMAKE_INSTALL_PREFIX=/usr/local
make -j$(nproc)
sudo make install
sudo ldconfig
```

### Optimized Device Strings

```bash
# Native AirSpy with packing (default)
airspy=0

# AirSpy via SoapySDR — sensitivity_gain + ppm from device string (mod 10)
soapy=0,driver=airspy,sensitivity_gain=15,ppm=1.2

# AirSpy via SoapySDR with custom buffers
soapy=0,driver=airspy,buflen=262144,buffers=8

# Disable USB bit packing (if issues)
airspy=0,pack=0
```

---

> 🛰️ **F4TNK — SatNOGS Station #3762**  
> C++ Optimizations of SDR Backends for Real-Time Satellite Reception on Raspberry Pi / x86_64

---

## Session 4: Cross-Repo Compatibility Fix (2026-02-20)

### O-X1. C++ Standard Upgrade: C++11 → C++17

**File**: `CMakeLists.txt`  
**Issue**: gr-osmosdr forced `CMAKE_CXX_STANDARD 11` since its inception. GNU Radio 3.10+ requires **C++17** minimum. While CMake auto-upgraded the standard through transitive target requirements, the explicit C++11 setting created:
- **ABI mismatch risk**: C++11 `std::string` (COW) vs C++17 `std::string` (SSO) can differ depending on compiler/libstdc++ version  
- **PMT header mismatch**: GNU Radio 3.10+ PMT uses `std::any`, `std::optional` (C++17 types) in headers  
- **Static analysis confusion**: linters/analyzers reported C++11 while actual compilation used C++17  

**Fix**: Changed `set(CMAKE_CXX_STANDARD 11)` to `set(CMAKE_CXX_STANDARD 17)`.  
**Docker**: Also added `-DCMAKE_CXX_STANDARD=20 -DCMAKE_CXX_STANDARD_REQUIRED=ON` to Dockerfile cmake invocation for full ABI match with the GNU Radio build.
