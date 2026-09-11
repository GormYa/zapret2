Read this in: [Türkçe](README.tr.md)

# zapret2

**zapret2** is an autonomous, high-performance, multi-platform Deep Packet Inspection (DPI) circumvention tool and programmable network packet manipulator. It enables bypassing internet censorship, throttling, and traffic signature inspection without requiring external third-party VPNs or proxies.

---

## Key Features

- **Autonomous & Serverless**: Works locally on your router or workstation; no remote proxies, VPN servers, or subscription services required.
- **Hybrid High-Performance Architecture**:
  - **C Core (`nfqws2` / `winws2`)**: Handles OS-level packet interception (NFQUEUE, WinDivert, divert socket), stateful connection tracking, protocol parsing, and packet reassembly with near-zero overhead.
  - **Lua Desync Engine (`zapret-lib.lua`, `zapret-antidpi.lua`)**: Executes flexible, scriptable DPI circumvention strategies on parsed packet trees (dissects).
- **Extensive Desync Strategies**:
  - TCP multi-split, multi-disorder, overlapping sequences (`seqovl`), and out-of-band (`oob`) manipulation.
  - Fake packet injection with custom TTL/autottl, bad checksums, MD5 signatures, and TLS SNI/Kyber payload modifications.
  - TCP window size scaling (`wssize`), SYN data (`syndata`), and timestamp reordering (`tcp_ts_up`).
  - QUIC / HTTP/3 initial packet desynchronization and fake packet injection.
  - Dynamic IP fragmentation (`ipfrag`).
- **Programmable Protocol Obfuscation**: Obfuscate arbitrary protocols (e.g. tunneling WireGuard UDP over ICMP echo pings).
- **Multi-Platform Support**:
  - **Linux**: Traditional distributions via `nftables` or `iptables` (NFQUEUE).
  - **OpenWrt**: Optimized for low-resource embedded routers.
  - **FreeBSD & pfSense**: Kernel divert via `ipfw` or `pf`.
  - **OpenBSD**: Interception via `pf`.
  - **Windows**: Native support via `winws2` using the WinDivert kernel driver.

---

## What's New in zapret2 (Compared to zapret1)

In *zapret1* (`nfqws1`), desync strategies were hardcoded directly in C, resulting in hundreds of rigid command-line parameters and making it difficult to adapt to rapidly evolving DPI filtering techniques.

**zapret2 changes the paradigm:**
1. **Lua-Driven Strategies**: Desync attacks are written in Lua scripts. Anyone with networking knowledge can write, adjust, or chain strategies without modifying C source code or recompiling.
2. **Payload-Type Awareness**: Distinguishes between connection protocols and specific payload types (e.g., `tls_client_hello`, `http_req`, `quic_initial`).
3. **Automatic TCP Segmentation**: `zapret-lib.lua` tracks connection MSS and automatically segments oversized packets (e.g. TLS with post-quantum Kyber handshakes or large `seqovl` buffers).
4. **Flexible Filter Ranges**: Directional range controls (`--in-range` and `--out-range`) based on packet counts, data packets, or byte offsets to minimize Lua engine overhead.
5. **Universal Blobs**: Replace hardcoded fake payload flags with arbitrary binary blobs loaded from hex strings or files.

---

## Directory Structure

```text
zapret2/
├── binaries/           # Pre-compiled architecture binaries (if provided)
├── blockcheck2.sh      # Automated DPI bypass strategy diagnostic & testing tool
├── blockcheck2.d/      # Strategy definitions and test targets for blockcheck2
├── common/             # Helper scripts (OS detection, firewall, dialogs, installer)
├── config.default      # Default configuration template for system services
├── docs/               # In-depth technical documentation and build guides
│   ├── manual.en.md    # Complete zapret2 reference manual (English)
│   ├── manual.md       # Complete zapret2 reference manual (Russian)
│   ├── readme.md       # Technical architecture and getting started guide (Russian)
│   └── readme.tr.md    # Technical architecture and getting started guide (Turkish)
├── files/fake/         # Binary payload templates for fake packets (TLS, QUIC, etc.)
├── init.d/             # Service unit files (systemd, OpenWrt, OpenRC, SysV, etc.)
├── ipset/              # Automated blocklist downloaders and ipset/nftset generators
├── lua/                # Core Lua libraries (zapret-lib, zapret-antidpi, zapret-obfs)
├── nfq2/               # C source code for nfqws2, winws2, and dvtws2
├── install_easy.sh     # Interactive installer script for Linux / OpenWrt
└── uninstall_easy.sh   # Clean uninstaller script
```

---

## Quick Start

### 1. Finding the Best Strategy (`blockcheck2.sh`)
Before running zapret as a service, use `blockcheck2.sh` on Linux/BSD or run test presets on Windows to find which bypass strategies work for your ISP:
```bash
sudo ./blockcheck2.sh
```
Follow the interactive prompts to test HTTP, HTTPS (TLS 1.2 & 1.3), and QUIC against blocked domains.

### 2. Automatic Installation (Linux / OpenWrt)
To install zapret2 as an automated system service:
```bash
sudo ./install_easy.sh
```
The script will:
- Detect your OS, init system, and firewall backend (`nftables` or `iptables`).
- Verify or build the required binaries (`nfqws2`).
- Configure firewall redirection rules and register background services.

### 3. Windows Usage (`winws2`)
On Windows, zapret2 runs via `winws2.exe` powered by the WinDivert driver:
```cmd
winws2.exe --wf-tcp-out=80,443 ^
  --lua-init=@lua\zapret-lib.lua --lua-init=@lua\zapret-antidpi.lua ^
  --filter-tcp=80,443 --filter-l7=tls,http ^
  --payload=tls_client_hello --lua-desync=fake:blob=fake_default_tls:tcp_md5 ^
  --payload=tls_client_hello,http_req --lua-desync=multisplit:pos=1:seqovl=5
```

---

## Documentation

- [Full Manual (English)](docs/manual.en.md)
- [Technical Architecture & Migration Guide (Turkish)](docs/readme.tr.md)
- [Technical Architecture & Migration Guide (Russian)](docs/readme.md)
- [Compilation & Build Guides](docs/compile/)

---

## Donations

If you find this project valuable and wish to support ongoing development:
- **USDT ERC20**: `0x3d52Ce15B7Be734c53fc9526ECbAB8267b63d66E`
- **USDT TRC20**: `TEzAAtn4VhndqEaAyuCM78xh5W2gCjwWEo`
- **BTC**: `bc1qhqew3mrvp47uk2vevt5sctp7p2x9m7m5kkchve`
- **ETH**: `0x3d52Ce15B7Be734c53fc9526ECbAB8267b63d66E`

---

## License

This project is open-source software licensed under the terms of the MIT License. See [docs/LICENSE.txt](docs/LICENSE.txt) for details.
