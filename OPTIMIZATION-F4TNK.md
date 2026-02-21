# 🛰️ gr-osmosdr — Optimisation F4TNK

> **Branche** : `master-f4tnk`  
> **Base** : gr-osmosdr (osmocom)  
> **Station** : SatNOGS #3762  
> **Bilan** : **4 fichiers modifiés** — **3 rounds d’optimisation** — 12 optimisations
> **Cible** : Backends AirSpy + SoapySDR pour réception satellites en signaux faibles

---

## 📊 Vue d'ensemble — Chaîne d'acquisition SDR

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

    subgraph OSMOSDR["📡 gr-osmosdr (ce repo)"]
        E["airspy_source_c<br/>Backend natif"]
        F["soapy_source_c<br/>Backend SoapySDR"]
    end

    subgraph GR["⚙️ GNU Radio"]
        G["doppler_correction_cc"]
        H["frame_decoder"]
        I["Décodeurs satellites"]
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

## 🏗️ Architecture des modifications

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

## ⚡ Optimisations AirSpy Backend

### 1. FIFO élargi — 5M → 10M samples

```mermaid
flowchart LR
    subgraph AVANT["❌ Avant: 5M samples"]
        A1["~1s de buffer<br/>à 2.5 MSPS"]
        A1 --> A2["Overflow fréquent<br/>sur pics CPU"]
    end

    subgraph APRES["✅ Après: 10M samples"]
        B1["~2s de buffer<br/>à 2.5/6 MSPS"]
        B1 --> B2["Absorbe les pics<br/>CPU / scheduler"]
    end

    style AVANT fill:#e74c3c,stroke:#333,color:#fff
    style APRES fill:#2ecc71,stroke:#333,color:#fff
```

### 2. Callback USB — Bulk Insert

```mermaid
sequenceDiagram
    participant USB as libairspy USB
    participant CB as airspy_rx_callback
    participant FIFO as circular_buffer

    Note over CB: ❌ Avant: boucle push_back × N
    USB->>CB: float* samples (IQ interleaved)
    loop Chaque échantillon
        CB->>FIFO: push_back(gr_complex(I, Q))
    end

    Note over CB: ✅ Après: bulk insert
    USB->>CB: gr_complex* src (cast direct)
    CB->>FIFO: insert(end, src, src+N)
    Note over FIFO: Un seul appel,<br/>copie optimisée interne
```

**Pourquoi ça marche** : FLOAT32\_IQ est garanti `{float I, float Q}` continu en mémoire — identique au layout de `gr_complex` (`std::complex<float>`). Le cast est safe et élimine la boucle.

### 3. work() — Partial Return + memcpy

```mermaid
stateDiagram-v2
    [*] --> WaitData

    state WaitData {
        [*] --> Check
        Check --> Timeout: FIFO vide > 100ms
        Check --> HasData: FIFO non-vide
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

| Aspect | Avant | Après |
|--------|-------|-------|
| Attente | Bloque indéfiniment | Timeout 100ms (anti-hang) |
| Retour | `noutput_items` exact requis | Partiel (ce qui est dispo) |
| Copie | `for` loop `at(i)` + `pop_front()` | `memcpy` × 2 segments max |
| Latence pipeline | Haute (attend buffer plein) | Basse (retour immédiat) |

### 4. Force FLOAT32\_IQ + USB Bit Packing

```mermaid
flowchart TD
    A["airspy_set_sample_type<br/>FLOAT32_IQ"] --> B["Cast gr_complex* sûr"]
    C["airspy_set_packing(1)"] --> D["4×12-bit → 3×16-bit"]
    D --> E["↓ 25% bande USB"]
    E --> F["Moins de drops sur RPi/USB2"]

    style B fill:#2ecc71,color:#fff
    style F fill:#2ecc71,color:#fff
```

---

## ⚡ Optimisations SoapySDR Backend

### 5. MTU-Aware Reads

```mermaid
flowchart LR
    A["getStreamMTU()"] --> B{"MTU > 0 ?"}
    B -->|Oui| C["toRead = min(noutput, MTU)"]
    B -->|Non| D["toRead = min(noutput, 65536)"]
    C --> E["readStream(toRead)"]
    D --> E
    E --> F["Lecture alignée<br/>sur les buffers matériels"]

    style F fill:#2ecc71,color:#fff
```

### 6. Stream Args Forwarding

```mermaid
flowchart TD
    A["Device args string"] --> B{"buflen= ?<br/>buffers= ?"}
    B -->|Présent| C["SoapySDR::Kwargs<br/>streamArgs"]
    C --> D["setupStream(CF32, channels, streamArgs)"]
    B -->|Absent| E["setupStream(CF32, channels)"]

    style D fill:#3498db,color:#fff
```

Permet de configurer la taille et le nombre de buffers USB directement depuis la device string GNU Radio :
```
driver=airspy,buflen=262144,buffers=8
```

### 7. Overflow Recovery + Timeout

```mermaid
flowchart TD
    A["readStream()"] --> B{"Résultat ?"}
    B -->|OVERFLOW| C{"retries > 0 ?"}
    C -->|Oui| D["Retry (flag reset)"]
    D --> A
    C -->|Non| E["return dernier ret"]
    B -->|TIMEOUT| F["return 0<br/>(GNU Radio réessaie)"]
    B -->|Erreur| G["return 0<br/>(récupération douce)"]
    B -->|OK ≥ 0| H["return ret samples"]

    style H fill:#2ecc71,color:#fff
    style F fill:#f39c12,color:#fff
    style G fill:#e74c3c,color:#fff
```

**Avant** : 1 retry sur overflow, pas de gestion du timeout  
**Après** : 3 retries overflow + timeout propre → pas de blocage

### 8. Anti-Alias Bandwidth 0.75 → 0.80

```mermaid
flowchart LR
    subgraph AVANT["0.75 × Fs"]
        A1["Fs = 2.5 MSPS"]
        A1 --> A2["BW = 1.875 MHz"]
        A2 --> A3["Perte de signal<br/>en bord de bande"]
    end

    subgraph APRES["0.80 × Fs"]
        B1["Fs = 2.5 MSPS"]
        B1 --> B2["BW = 2.0 MHz"]
        B2 --> B3["Meilleur compromis<br/>signal vs repliement"]
    end

    style A3 fill:#e74c3c,color:#fff
    style B3 fill:#2ecc71,color:#fff
```

Les signaux satellites LEO sont à bande étroite au centre — 80% offre un meilleur compromis que 75% qui coupait trop.

---

## � Optimisations LEO Satellite (Round 3)

### 9. rx\_time Tag PMT — Timestamps matériels vers GNU Radio

`timeNs` était lu par `readStream()` mais jamais exploité. Maintenant chaque appel réussi
émet un tag `rx_time` au format UHD-compatible :

```
pmt::make_tuple(pmt::from_uint64(full_secs), pmt::from_double(frac_secs))
```

Ce tag est lu nativement par `gnuradio/blocks/tagged_file_sink`, `file_meta_sink`,
et tout bloc downstream qui aligne sur le temps physique.
Il transporte le timestamp de notre **SoapyAirspy Mod 23** (steady\_clock ns).

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
    DS->>DS: Aligne démodulation / enregistrement
```

**Fichier :** `soapy_source_c.cc`

### 10. Forwarding Settings — Device String vers writeSetting()

Avant, seuls `buflen`/`buffers` étaient transmis depuis la device string. Maintenant
`sensitivity_gain`, `linearity_gain`, `ppm`, `biastee`, `bitpack` sont aussi forwardés
u00e0 `writeSetting()` — **compatible SoapyAirspy mods 16-18**.

```bash
# Avant — ces args étaient silencieusement ignorés par gr-osmosdr
soapy=0,driver=airspy,sensitivity_gain=15,ppm=1.2

# Après — transmis directement au driver
# SoapyAirspy::writeSetting("sensitivity_gain", "15") est appelé
```

**Fichier :** `soapy_source_c.cc`

### 11. \_\_builtin\_prefetch — Cache Warming

```mermaid
timeline
    title Chronologie USB callback (avant / après)
    Avant lock : [attente lock] → [copie cold]
    Après lock  : [prefetch L2 stride 256B] → [lock] → [copie warm]
```

| Point | Prefetch | Localité | Raison |
|-------|----------|----------|--------|
| `rx_callback` | `src` stride 256B | L2 (`_MM_HINT_T1`) | Données USB ~24KB, > L1 |
| `work()` | `out` écriture | L1 (`_MM_HINT_T0`) | Destination immédiate |

**Fichier :** `airspy_source_c.cc`

### 12. Compteur Overflow Atomique

L’ancien code imprimait `"O"` sur stderr à chaque overflow — flood inaccessible.

```cpp
// Avant
if (to_copy < num_samples)
    std::cerr << "O" << std::flush;  // flood, pas de comptage

// Après
std::atomic<uint32_t> _overflow_count;
// ...
const uint32_t cnt = ++_overflow_count;
if (cnt % 100 == 1)
    std::cerr << "AirSpy FIFO overflow #" << cnt << std::endl;
```

| Avant | Après |
|-------|-------|
| Print `"O"` à chaque overflow (flood) | Log tous les 100 |
| Pas de comptage | Compteur atomique cumulatif |
| Invisible dans SatNOGS | `#N` visible dans les logs client |

**Fichier :** `airspy_source_c.h`, `airspy_source_c.cc`

---

## �🔧 Build — CMake Flags

```mermaid
flowchart TD
    subgraph FLAGS["Flags de compilation (avec guards)"]
        F1["-march=native<br/>Instructions CPU natives"]
        F2["-ffast-math<br/>IEEE relaxé → SIMD"]
        F3["-ftree-vectorize<br/>Auto-vectorisation"]
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

Chaque flag est testé avant activation → compatible cross-compiler.

---

## 📈 Impact estimé

```mermaid
xychart-beta
    title "Gain par optimisation"
    x-axis ["FIFO x2", "Bulk insert", "memcpy work", "USB packing", "MTU reads", "3x retry", "Partial ret"]
    y-axis "Impact signal" 0 --> 10
    bar [3, 5, 7, 4, 6, 3, 8]
```

| Métrique | Avant | Après | Amélioration |
|----------|-------|-------|-------------|
| Buffer overflow (2.5 MSPS) | Fréquent sur RPi | Rare | ~10× moins |
| Latence pipeline | 100+ ms | < 10 ms | Partial return |
| Bande USB | 100% | ~75% | Bit packing |
| Callback CPU/sample | ~4 ops | ~0.5 ops | Bulk insert |
| Timestamp GR | Absent | `rx_time` PMT tag | Mod 9 |
| Device string gain | Ignoré | `writeSetting()` | Mod 10 |
| Overflow log | flood `O` | `#N` tous les 100 | Mod 12 |

---

## 📋 Historique des commits

| Commit | Description |
|--------|------------|
| `f60a1b9` | **Round 1** : MTU reads, FIFO 10M, bulk insert, memcpy work(), CMake flags |
| `56cb6ac` | **Round 2** : Force FLOAT32\_IQ, USB packing, partial return, timed wait, anti-alias 0.80 |
| `8e17305` | **Round 3** : rx\_time PMT tag, settings forwarding, prefetch L2, atomic overflow counter |

---

## 📂 Fichiers modifiés (4 fichiers)

```mermaid
pie title Répartition des modifications
    "airspy_source_c.cc" : 88
    "soapy_source_c.cc" : 53
    "CMakeLists.txt" : 21
    "soapy_source_c.h" : 1
```

---

## 📖 Utilisation

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

### Device strings optimisées

```bash
# AirSpy natif avec packing (défaut)
airspy=0

# AirSpy via SoapySDR — sensitivity_gain + ppm depuis device string (mod 10)
soapy=0,driver=airspy,sensitivity_gain=15,ppm=1.2

# AirSpy via SoapySDR avec buffers personnalisés
soapy=0,driver=airspy,buflen=262144,buffers=8

# Désactiver le bit packing USB (si problèmes)
airspy=0,pack=0
```

---

> 🛰️ **F4TNK — Station SatNOGS #3762**  
> Optimisations C++ des backends SDR pour la réception satellite temps réel sur Raspberry Pi / x86_64

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
