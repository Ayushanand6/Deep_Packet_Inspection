# High-Performance Deep Packet Inspection (DPI) Engine

A multi-threaded C++17 Deep Packet Inspection (DPI) system designed to capture, parse, classify, and filter network traffic from PCAP files at wire speed. The engine extracts Layer 7 protocol metadata (including TLS Server Name Indication and HTTP Host headers) and applies real-time policy-based rule enforcement.

---

## Key Features

- **Protocol Parsing**: Decodes Ethernet, IPv4, TCP, and UDP headers with zero external runtime dependencies.
- **L7 SNI & Host Extraction**: Inspects TLS Client Hello records (SNI) and HTTP request payloads to identify services (YouTube, Google, Facebook, etc.) across encrypted channels.
- **Stateful Flow Tracking**: Tracks bidirectional network flows using a canonical 5-tuple hash table (`src_ip`, `dst_ip`, `src_port`, `dst_port`, `protocol`).
- **Concurrent Pipeline**: Scalable producer-consumer multi-threaded architecture with dynamic load balancing across Fast Path (FP) worker threads using consistent hashing.
- **Rule Enforcement & Filtering**: Real-time packet dropping/forwarding based on source IP, identified application signatures, and domain patterns.
- **PCAP Reconstruction**: Outputs filtered network captures directly into compliant PCAP format with summary diagnostics.

---

## System Architecture

The multi-threaded variant utilizes a staged pipeline to distribute traffic across dedicated Fast Path workers while preserving flow affinity:

```
                      ┌─────────────────┐
                      │  Reader Thread  │
                      │  (Ingests PCAP) │
                      └────────┬────────┘
                               │
                ┌──────────────┴──────────────┐
                │      hash(5-tuple) % N      │
                ▼                             ▼
      ┌─────────────────┐           ┌─────────────────┐
      │ Load Balancer 0 │           │ Load Balancer 1 │
      └────────┬────────┘           └────────┬────────┘
               │                             │
        ┌──────┴──────┐               ┌──────┴──────┐
        │hash % M     │               │hash % M     │
        ▼             ▼               ▼             ▼
  ┌───────────┐ ┌───────────┐   ┌───────────┐ ┌───────────┐
  │Fast Path 0│ │Fast Path 1│   │Fast Path 2│ │Fast Path 3│
  └─────┬─────┘ └─────┬─────┘   └─────┬─────┘ └─────┬─────┘
        │             │               │             │
        └─────────────┴───────┬───────┴─────────────┘
                              ▼
                  ┌───────────────────────┐
                  │  Thread-Safe Queue    │
                  └───────────┬───────────┘
                              ▼
                  ┌───────────────────────┐
                  │ Output Writer Thread  │
                  │ (Writes Filtered PCAP)│
                  └───────────────────────┘
```

---

## Directory Structure

```
packet_analyzer/
├── include/
│   ├── pcap_reader.h          # PCAP file parser & binary header decoder
│   ├── packet_parser.h        # L2-L4 protocol parsing logic
│   ├── sni_extractor.h        # TLS SNI and HTTP Host extraction
│   ├── types.h                # Core data structures (FiveTuple, Flow, AppType)
│   ├── rule_manager.h         # Flow filtering policy engine
│   ├── connection_tracker.h   # Stateful flow mapping table
│   ├── load_balancer.h        # Load distribution threads
│   ├── fast_path.h            # Worker threads for DPI processing
│   ├── thread_safe_queue.h    # Mutex & condition variable blocking queue
│   └── dpi_engine.h           # System orchestrator
│
├── src/
│   ├── pcap_reader.cpp
│   ├── packet_parser.cpp
│   ├── sni_extractor.cpp
│   ├── types.cpp
│   ├── main_working.cpp       # Single-threaded lightweight pipeline
│   └── dpi_mt.cpp             # Scalable multi-threaded engine
│
├── generate_test_pcap.py      # Test harness for mock packet generation
└── README.md
```

---

## Getting Started

### Prerequisites
- Modern C++17 compiler (`g++` >= 9.0 or `clang++` >= 10.0)
- POSIX-compliant operating system (Linux / macOS)
- `pthread` library

### Build Instructions

**Multi-threaded Engine (Production):**
```bash
g++ -std=c++17 -pthread -O3 -I include -o dpi_engine \
    src/dpi_mt.cpp \
    src/pcap_reader.cpp \
    src/packet_parser.cpp \
    src/sni_extractor.cpp \
    src/types.cpp
```

**Single-threaded Lightweight Variant:**
```bash
g++ -std=c++17 -O3 -I include -o dpi_simple \
    src/main_working.cpp \
    src/pcap_reader.cpp \
    src/packet_parser.cpp \
    src/sni_extractor.cpp \
    src/types.cpp
```

---

## Usage

### Basic Execution
Analyze an input PCAP and output the filtered result:
```bash
./dpi_engine input.pcap output.pcap
```

### Applying Filtering Rules
Filter traffic based on applications, subnets/IPs, or specific domain patterns:
```bash
./dpi_engine input.pcap output.pcap \
    --block-app YouTube \
    --block-app TikTok \
    --block-ip 192.168.1.50 \
    --block-domain facebook
```

### Performance & Thread Configuration
Configure load balancers and fast path worker thread counts based on CPU core availability:
```bash
./dpi_engine input.pcap output.pcap --lbs 4 --fps 8
```

---

## Diagnostics & Output

Upon execution, the engine reports processing throughput and protocol distribution metrics:

```
[Rules] Blocked: App(YouTube), IP(192.168.1.50)
[Reader] Processing completed: 150,000 packets ingested.

=================== RUNTIME METRICS ===================
Ingested Packets:   150,000 | Ingested Volume:  112.4 MB
Forwarded:          138,210 | Dropped:           11,790

Traffic Classification:
- HTTPS / TLS:       68.4%
- YouTube (Blocked):   7.8%
- DNS:                 5.1%
- Unclassified / Raw: 18.7%
======================================================
```