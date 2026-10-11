
# XMRig Cryptominer Analysis

**A Linux-focused malware-analysis and threat-hunting case study investigating XMRig through static analysis, controlled execution, process monitoring, and system-call tracing.**

## Overview

This project examines XMRig v6.26.0, an open-source cryptocurrency-mining application, to understand its binary structure, mining-related functionality, resource consumption, and runtime behaviour.

The investigation was conducted in an isolated Ubuntu Linux virtual machine. XMRig was executed using its official benchmark mode, without configuring a real mining pool or wallet.

The objective was to develop practical skills in Linux malware analysis, process investigation, behavioural analysis, and identifying evidence relevant to cryptojacking investigations.

## Objectives

- Perform static analysis of a Linux ELF executable.
- Identify mining-related strings, dependencies, and imported functions.
- Establish a system baseline before controlled execution.
- Monitor process activity and resource consumption using `htop` and Linux utilities.
- Investigate system calls and network-related behaviour using `strace` and `ss`.
- Identify potential defensive monitoring opportunities.
- Separate observed behaviour from capabilities inferred through static analysis.

## Tools and Technologies

| Category | Tools |
|---|---|
| Static analysis | `file`, `sha256sum`, `readelf`, `ldd`, `strings`, `nm`, `objdump` |
| Process monitoring | `htop`, `ps` |
| Linux investigation | `/proc`, `ss` |
| Dynamic analysis | XMRig benchmark mode, `strace` |
| Network monitoring | `tcpdump`, network and socket inspection |
| Defensive context | Linux SOC monitoring and cryptojacking detection concepts |

## Repository Structure

```text
XMRig-Analysis/
├── README.md
├── 01-Static-Analysis.md
├── 02-Dynamic-Analysis-and-Detection.md
└── 03-Case-Study-Report.md
```

## Analysis Documentation

### 1. Static Analysis

[`01-Static-Analysis.md`](01-Static-Analysis.md)

Covers sample identification, ELF structure, dynamic dependencies, embedded strings, imported functions, and findings about potential mining-related capabilities.

### 2. Dynamic Analysis and Detection

[`02-Dynamic-Analysis-and-Detection.md`](02-Dynamic-Analysis-and-Detection.md)

Documents baseline monitoring, `htop` observations, benchmark performance, process and memory analysis, system-call tracing, network observations, MSR optimization activity, and defensive detection opportunities.

### 3. Case-Study Report

[`03-Case-Study-Report.md`](03-Case-Study-Report.md)

Consolidates the investigation methodology, key findings, defensive implications, limitations, and conclusions.

## Key Findings

- Identified RandomX-, Monero-, Stratum-, and mining-pool-related strings through static analysis.
- Observed XMRig executing a multithreaded RandomX benchmark.
- Recorded a final benchmark result of **533.6 H/s** under the tested virtual-machine configuration.
- Examined CPU and memory usage using `htop`, process utilities, and Linux `/proc` interfaces.
- Identified local network-interface queries in the `strace` output.
- Observed an MSR optimization attempt that encountered a permission restriction.
- Found no remote `connect()` call in the recorded benchmark trace.

**Important:** The benchmark did not demonstrate successful mining-pool communication or unauthorized mining. Findings are limited to the execution and evidence collected.

## Defensive Relevance

Cryptojacking investigations benefit from correlating multiple signals rather than relying on CPU utilization alone.

This project explores how process identity, command-line arguments, resource consumption, system-call activity, network telemetry, and execution authorization can support a Linux SOC investigation.

The detection opportunities documented in this repository are proposed defensive approaches; the project does not claim that every behaviour generated a Wazuh alert.

## Safety and Scope

- Conducted in an isolated Linux virtual machine.
- Used the official XMRig benchmark mode.
- Did not configure a real mining pool or wallet.
- Did not attempt unauthorized mining, persistence, or privilege escalation.
- Distinguished observed runtime behaviour from potential capabilities identified statically.

## References

- [Official XMRig repository](https://github.com/xmrig/xmrig)
- [Linux `strace` manual](https://man7.org/linux/man-pages/man1/strace.1.html)
- [Linux `/proc` filesystem documentation](https://man7.org/linux/man-pages/man5/proc.5.html)

## Author's Note

This project is part of my ongoing hands-on development in cybersecurity, Linux malware analysis, SOC investigations, and threat detection.

The focus is not simply to execute a sample, but to collect evidence, interpret its behaviour, document limitations, and derive defensible security findings.
