# XMRig Cryptominer — Static Analysis

## 1. Introduction

This case study examines **XMRig v6.26.0**, an open-source CPU/GPU cryptocurrency mining application commonly associated with Monero mining. </br>
</br>
The objective of this phase was to inspect the binary's structure, dependencies, imported functions, and embedded strings to understand its capabilities before examining its runtime behaviour.

</br>
XMRig is legitimate mining software; its presence does not, by itself, establish malicious activity. However, unauthorized deployment of cryptomining software can constitute cryptojacking, in which a system's computing resources are used for cryptocurrency mining without the owner's consent.

The analysis was performed in an isolated Linux virtual machine as part of a defensive malware-analysis and threat-hunting project.

## 2. Laboratory Context

| Item | Details |
|---|---|
| Sample | XMRig |
| Version | 6.26.0 |
| Platform | Linux x86-64 |
| File format | ELF executable |
| Analysis environment | Isolated Ubuntu Linux virtual machine |

The sample was obtained from the official XMRig project:

- Repository: https://github.com/xmrig/xmrig
- Sample archive: `xmrig-6.26.0-jammy-x64.tar.gz`
- SHA-256: `ca82fc8426187880dffa502363849af6258e65fdb675a9cc9984a2b843854087`

The SHA-256 hash was verified after acquisition to establish the identity of the downloaded archive.

---

## 3. Sample Identification and Integrity

The initial examination focused on identifying the executable format and recording the sample's cryptographic hash.

```bash
file xmrig
sha256sum xmrig
```

**{picture of file and sha256sum}


The `file` utility identifies the executable format and architecture, while `sha256sum` generates a cryptographic fingerprint that can be used to verify sample integrity.

The executable was subsequently examined using ELF inspection tools to understand its structure and runtime dependencies.


## 4. ELF Header and Section Analysis

Linux executable binaries commonly use the **Executable and Linkable Format (ELF)**. Examining the ELF header helps establish the target architecture and other structural properties.

Commands used:

```bash
readelf -h xmrig
readelf -S xmrig
```

- `readelf -h` displays ELF header information, including architecture and executable type.
- `readelf -S` lists the binary's sections, which organize information such as executable code, data, symbols, and relocation information.


**{picture of -h and -s readelf}


This inspection provides structural context for subsequent analysis. ELF sections alone do not establish whether a binary is malicious.

## 5. Dynamic Dependencies

The executable's dynamic dependencies were examined to identify shared libraries required for execution.

```bash
readelf -d xmrig
ldd xmrig
```

**{picture of -d readelf & ldd}


`readelf -d` displays the dynamic section, including dependency declarations. `ldd` displays the shared libraries resolved in the analysis environment.

Dependency information helps explain how the application interacts with its operating system and supporting libraries. The presence of a particular library should be interpreted in context rather than treated as evidence of malicious behaviour.

## 6. Embedded String Analysis

The binary's printable strings were examined for references to cryptocurrency mining, networking, configuration options, and other potentially useful indicators.

```bash

strings xmrig > strings-output.txt

grep -Ei 'randomx|monero|stratum|pool|wallet|tls|http|api' strings-output.txt

```

The filtered output contained references associated with:

- **RandomX:** A proof-of-work algorithm associated with Monero mining.
- **Monero:** The cryptocurrency associated with the mining functionality examined.
- **Stratum and pool-related terminology:** References associated with mining-pool communication.
- **Wallet configuration:** References associated with specifying a cryptocurrency wallet.
- **TLS and HTTP API terminology:** References indicating related communication or interface functionality.

These strings provide useful leads about the application's intended capabilities. However, an embedded string does not prove that the corresponding functionality was invoked during execution.


**{picture of above}

## 7. Imported Function and Symbol Analysis

Dynamic symbols and imported functions were examined to identify interfaces associated with networking, threading, memory allocation, and file operations.

Commands used:

```bash

nm -D xmrig | grep -Ei 'socket|connect|recv|send|pthread|malloc|free|open|close|dns' > interesting-symbols.txt

objdump -T xmrig | grep -Ei 'socket|connect|recv|send|pthread|malloc|free|open|close|ssl|crypto|curl' > interesting-imports.txt

```

The inspection identified functions associated with several operating-system activities:

| Category | Examples observed | Analytical relevance |
|---|---|---|
| Networking | `socket`, `connect`, `send`, `recvmsg` | Interfaces that can support network communication |
| Thread management | `pthread_create`, `pthread_join`, `pthread_detach` | Support for concurrent execution |
| Memory management | `malloc`, `free`, `realloc` | Dynamic memory allocation and release |
| File operations | `open`, `openat`, `close`, `fclose` | File access and descriptor management |

These imports are consistent with a multithreaded application that performs computational work and includes networking and file-handling functionality.

**Important limitation:** Imported functions establish that the binary links to relevant interfaces; they do not prove that every function was executed. Runtime observations are required to establish actual behaviour.

---

## 8. Static Analysis Findings

The static examination produced the following findings:

| Finding | Evidence | Interpretation |
|---|---|---|
| Linux executable | ELF identification and header inspection | The sample is a Linux executable |
| Cryptocurrency mining functionality | RandomX and Monero-related strings | Consistent with XMRig's mining purpose |
| Mining-pool-related functionality | Stratum and pool-related strings | Indicates support for mining-pool communication |
| Wallet configuration references | Wallet-related strings | Consistent with configurable mining operation |
| Network interfaces | Imported networking functions | The binary contains interfaces capable of supporting network communication |
| Multithreading | Imported pthread functions | Consistent with concurrent mining workers |
| Dynamic memory management | Memory-allocation imports and RandomX-related strings | Consistent with a memory-intensive workload |

The findings describe capabilities and structural evidence identified during static analysis. They do not independently establish unauthorized mining, successful pool communication, persistence, or other malicious activity.


## 9. Conclusion

Static analysis established that the examined XMRig v6.26.0 binary is consistent with a Linux cryptocurrency-mining application. Its embedded strings and imported functions provided leads concerning RandomX, Monero, mining-pool functionality, threading, memory management, and network communication.

These findings established a foundation for the next phase: controlled dynamic analysis. That phase examined process behaviour, resource consumption, system calls, and network activity during execution in official benchmark mode.

The central analytical principle was to distinguish **what the binary appears capable of doing from what the investigation actually observed it doing**.
