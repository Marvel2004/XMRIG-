# XMRig Cryptominer — Case Study Report

## 1. Executive Summary

This case study investigated XMRig v6.26.0, an open-source cryptocurrency-mining application, to understand its executable structure, runtime behaviour, resource consumption, and potential relevance to cryptojacking investigations.

The investigation followed a static-to-dynamic analysis workflow in an isolated Ubuntu Linux virtual machine. Static analysis examined the ELF executable, dependencies, embedded strings, and imported functions. Dynamic analysis used process monitoring, Linux system interfaces, network inspection, and `strace` during execution in the application's official benchmark mode.

The benchmark achieved a final reported rate of **533.6 hashes per second (H/s)** under the tested virtual-machine configuration. Runtime observations confirmed multithreaded execution, substantial memory consumption, local network-interface discovery, and an MSR optimization attempt that encountered a permission restriction.

No remote connection was observed during the recorded benchmark, and successful mining-pool communication was not demonstrated. The investigation therefore establishes XMRig's observed benchmark behaviour, not an incident of unauthorized mining.

## 2. Investigation Objectives

The investigation aimed to:

- Identify the executable's structure and mining-related capabilities.
- Observe CPU, memory, and thread activity during controlled execution.
- Examine operating-system interactions through system-call tracing.
- Investigate network-related behaviour during benchmark execution.
- Identify behaviours that could inform Linux SOC monitoring and cryptojacking investigations.
- Distinguish observed evidence from potential capabilities and unsupported conclusions.

## 3. Sample and Laboratory Context

| Attribute | Details |
|---|---|
| Sample | XMRig v6.26.0 |
| Platform | Linux x86-64 |
| Operating system | Ubuntu Linux 22.04 |
| Executable format | ELF |
| Execution mode | `--bench=1M` |
| Primary monitoring tools | `htop`, `ps`, `/proc`, `ss`, `strace` |
| Network monitoring | `ss`, system-call tracing, baseline packet capture |
| Mining-pool deployment | Not performed |

The sample was obtained from the official XMRig repository:

https://github.com/xmrig/xmrig

The downloaded archive's SHA-256 hash was verified during acquisition. The archive hash and executable hash should be recorded separately when documenting sample integrity.

All practical execution was limited to the isolated analysis environment. No real mining pool or wallet was configured.

## 4. Methodology

The investigation consisted of four stages.

### 4.1 Sample Acquisition and Static Analysis

The executable was identified and examined using `file`, `sha256sum`, `readelf`, `ldd`, `strings`, `nm`, and `objdump`.

The examination identified RandomX-, Monero-, Stratum-, pool-, and wallet-related strings, along with imported functions associated with networking, threading, memory allocation, and file operations.

These findings were treated as evidence of executable structure and potential capabilities, not proof that each capability was exercised at runtime.

### 4.2 Baseline Monitoring

System resource usage and network activity were recorded before execution. Monitoring included CPU, memory, process activity, active connections, listening sockets, network interfaces, DNS status, disk usage, and packet capture.

Suricata consumed approximately 101% CPU during one baseline observation. This highlighted the need to distinguish XMRig's process-specific resource consumption from background workloads.

### 4.3 Controlled Dynamic Analysis

XMRig was executed using:

```bash
./xmrig --bench=1M
```

The process was monitored using `htop`, `ps`, and Linux `/proc` interfaces. `strace` was then used to investigate process creation, file access, network-related system calls, and MSR optimization activity.

### 4.4 Evidence Correlation

Static and dynamic findings were compared to determine which capabilities were supported by runtime observations.

The investigation distinguished:

- **Observed:** Directly recorded during the benchmark.
- **Identified statically:** Supported by binary strings, imports, or structural inspection.
- **Not demonstrated:** Not established by the collected evidence.

## 5. Key Findings

### 5.1 Mining-Related Functionality

Static analysis identified strings associated with RandomX, Monero, Stratum, mining pools, wallet configuration, TLS, and HTTP API functionality.

These findings were consistent with XMRig's intended purpose as mining software. They did not establish that a remote mining session occurred.

### 5.2 Resource Consumption and Benchmark Performance

The improved benchmark reported:

| Metric | Result |
|---|---|
| RandomX dataset | Approximately 2,336 MB |
| Mining threads | 5 |
| Total process threads | 7 |
| Peak virtual memory | Approximately 2.75 GiB |
| Resident memory | Approximately 2.31 GiB |
| Swap usage | Approximately 668 kB |
| Huge-page usage | 0% |
| Final benchmark rate | **533.6 H/s** |

The process appeared in `htop` during execution, allowing its activity to be examined alongside other running processes.

The observations demonstrated a computationally intensive workload with substantial memory requirements. The benchmark rate was specific to the tested environment and is not a measure of maliciousness.

### 5.3 Process and System-Call Behaviour

The recorded `strace` output contained approximately 7,647 lines. It showed successful execution of the XMRig binary and repeated `clone3()` calls associated with thread creation.

File and system discovery activity included accesses to `/proc`, `/sys`, and standard library configuration paths. These observations were consistent with runtime initialization and hardware/environment inspection.

Some attempted accesses returned `ENOENT` or `EACCES`. Such errors were interpreted in context rather than classified as malicious behaviour independently.

### 5.4 Network Behaviour

The trace recorded a Netlink socket and `RTM_GETLINK` and `RTM_GETADDR` requests, which are used to query local network interfaces and addresses.

No `connect()` call was found in the recorded trace, and no active network socket associated with XMRig was observed during the corresponding benchmark inspection.

Because execution used benchmark mode without a mining-pool configuration, the investigation did not establish successful remote mining-pool communication.

### 5.5 MSR Optimization Attempt

The trace showed an attempted write to:

```text
/sys/module/msr/parameters/allow_writes
```

The operation returned `EACCES`. The trace subsequently recorded execution of `/bin/sh` and `/sbin/modprobe` with the arguments `msr allow_writes=on`, followed by inspection of module configuration files.

This activity corresponded to XMRig's MSR optimization attempt. The evidence demonstrated an attempted operation that encountered a permission restriction, not successful privilege escalation or persistence.

## 6. Defensive Implications

The investigation identified several behaviours that may support a cryptojacking investigation when correlated with host and network context.

| Indicator | Potential defensive use |
|---|---|
| Sustained high CPU utilization | Identify and investigate resource-intensive processes |
| Large process memory footprint | Correlate memory usage with executable identity and workload |
| Multiple worker threads | Understand computational activity |
| Mining-related command-line arguments | Investigate execution context and authorization |
| Connections to mining infrastructure | Correlate destinations with DNS, firewall, and network telemetry when observed |
| MSR-related access attempts | Examine process identity, permissions, and system-call evidence |

No single indicator is sufficient to confirm cryptojacking. Legitimate applications can produce similar resource and process patterns.

For a Linux SOC, the strongest investigation combines process execution data, resource telemetry, network evidence, authorization context, and the expected purpose of the host. Wazuh can support such investigations when the relevant telemetry is collected and suitable detection rules are configured.

This case study did not establish that a Wazuh alert was triggered.

## 7. Limitations

The investigation was restricted to static analysis and controlled benchmark execution.

The following were not demonstrated:

- Successful mining-pool communication.
- Unauthorized cryptocurrency mining.
- Persistence.
- Privilege escalation.
- A confirmed Wazuh detection alert.

The findings describe the collected evidence and should not be generalized to every XMRig configuration or execution mode.

## 8. Conclusion

The investigation established that the examined XMRig v6.26.0 binary was consistent with a Linux cryptocurrency-mining application. Static analysis identified mining-related functionality, while dynamic analysis demonstrated multithreaded benchmark execution, substantial memory consumption, a final performance rate of 533.6 H/s, local network-interface discovery, and an MSR optimization attempt that encountered a permission restriction.

No remote connection was observed during the recorded benchmark, and unauthorized mining was not established.

The central lesson is that effective malware analysis requires a clear distinction between a binary's potential capabilities and its observed runtime behaviour. For cryptojacking investigations, resource consumption is an important starting point, but conclusions should be based on correlated evidence and the context in which the software executes.

## 9. References

- XMRig official repository: https://github.com/xmrig/xmrig
- Linux `strace` manual: https://man7.org/linux/man-pages/man1/strace.1.html
- Linux `proc` filesystem documentation: https://man7.org/linux/man-pages/man5/proc.5.html
- Linux `readelf` documentation: https://sourceware.org/binutils/docs/binutils/readelf.html
