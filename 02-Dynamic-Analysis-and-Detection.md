# XMRig Cryptominer — Dynamic Analysis and Detection

## 1. Introduction

This phase examined the runtime behaviour of XMRig v6.26.0 in an isolated Ubuntu Linux virtual machine. Following static analysis, the objective was to observe the sample's resource consumption, process and thread activity, memory and cpu usage, system calls, and network behaviour.

The sample was executed using its official benchmark mode. No real mining pool or wallet was configured.

## 2. Laboratory Environment

| Item | Details |
|---|---|
| Sample | XMRig v6.26.0 |
| Operating system | Ubuntu Linux 22.04 |
| Execution mode | `--bench=1M` |
| Monitoring tools | `htop`, `ps`, `/proc`, `ss`, `strace` |
| Network monitoring | `ss`, `strace`, baseline packet capture |
| Objective | Controlled behavioural analysis |
</br>

## 3. Establishing the System Baseline

Before executing XMRig, the virtual machine's resource usage and network activity were examined to establish a baseline.

The baseline collection included CPU utilization, memory consumption, active processes, network connections, listening sockets, network interfaces, DNS status, disk usage, and system information.

During one baseline observation, Suricata consumed approximately **_101% CPU and 10.7% memory_**. This was important because it demonstrated that background security monitoring could contribute substantially to the system's overall CPU usage.

Consequently, system-wide CPU utilization was interpreted alongside individual process activity rather than being attributed to XMRig automatically.

</br>
</br>
{picture of any baseline reading with the command}
</br>
</br>

## 4. Monitoring Resource Consumption with htop

`htop` was used to monitor the virtual machine's CPU utilization, memory usage, running processes, and process activity during the investigation.

The command used was:

```bash
htop
```

The baseline view provided a reference for the processes already running before XMRig was launched.

After executing XMRig, its process appeared in the process list under the command `./xmrig --bench=1M`. Its CPU and memory consumption could then be examined alongside the existing processes.

The appearance of XMRig in the process list and its associated resource consumption provided a practical way to correlate the observed workload with the executable being investigated.


{pictures of htop comparison before and after xmrig initialization}


### 4.1 Interpreting the Process-List Change

The change in the `htop` process list was significant because it connected the benchmark execution to a visible process on the host.

During analysis, the process was also examined using:

```bash
ps -p 3250 -o pid,ppid,stat,%cpu,%mem,nlwp,etime,cmd
```

The recorded PID was `3250`; process IDs can change between executions.

The process snapshot showed XMRig running with multiple threads (7) and substantial CPU and memory consumption. Subsequent measurements were taken from the improved benchmark run to document the final execution.

**Evidence note:** The process appearing in `htop` confirms that XMRig was running. High resource consumption alone does not establish malicious activity; it must be interpreted in the context of the executable, its command line, and whether its execution was authorized.

</br>
</br>
<img width="1920" height="1080" alt="Screenshot from 2026-10-02 22-06-11" src="https://github.com/user-attachments/assets/04d304aa-ae99-4d42-a373-0ea347a808bf" />
</br>
</br>

## 5. Controlled XMRig Execution

The benchmark was launched with:

```bash
./xmrig --bench=1M
```

This mode executes a predefined computational benchmark without requiring deployment against a real mining pool.

The improved run produced the following observations:

| Metric | Observed result |
|---|---|
| XMRig version | 6.26.0 |
| Execution mode | `--bench=1M` |
| RandomX dataset | Approximately 2,336 MB |
| Mining threads | 5 |
| Total process threads | 7 |
| Peak virtual memory | 2,883,944 kB |
| Resident memory | 2,418,116 kB |
| Swap usage | 668 kB |
| Huge-page usage | 0% |
| Final benchmark rate | **533.6 H/s** |

The benchmark completed at approximately 533.6 hashes per second under the tested virtual-machine configuration.

The reported RandomX dataset and resident memory demonstrate that the workload required substantial memory. Five mining threads were reported, while the process had seven threads in total.

Huge-page usage remained at 0%. Huge pages are a memory-management optimization that can improve performance for some workloads; their absence does not indicate malicious behaviour.

</br>
</br>
<img width="1920" height="1145" alt="Screenshot from 2026-10-02 22-35-31" src="https://github.com/user-attachments/assets/4ce55c44-8c20-4a7f-a5c0-89c1acb20ef6" />
</br>
</br>


### 5.1 Understanding the Final Hash Rate

The final rate of **533.6 H/s** represents the number of hashes calculated per second during the benchmark.

This result is specific to the tested virtual machine, its allocated resources, and the execution configuration. It is a performance measurement, not a malware verdict or an indicator that unauthorized mining occurred.

## 6. Process and Memory Investigation

Additional measurements were obtained through the Linux `/proc` filesystem.

```bash
cat /proc/3692/status
```

```bash
cat /proc/3692/io
```

```bash
ls -l /proc/3692/fd
```

```bash
cat /proc/3692/maps
```

These interfaces provided information about process memory, thread count, I/O counters, open file descriptors, and mapped memory regions.

The improved run showed approximately 2.31 GiB of resident memory and negligible swap usage compared with the earlier constrained environment. This was consistent with the reduced memory pressure after increasing the virtual-machine resources.

The process I/O counters and file descriptors were also examined to understand the process's interaction with the operating system. These measurements were interpreted in context rather than assuming that every recorded read or open descriptor represented suspicious file activity.

</br>
</br>
<img width="1920" height="1145" alt="Screenshot from 2026-10-02 22-35-31" src="https://github.com/user-attachments/assets/5b457c9d-e460-4f2a-8ee4-ff480cff5052" />
</br>
</br>

## 7. System-Call Analysis Using strace

To examine the interactions between XMRig and the Linux kernel, the benchmark was executed under `strace`.

```bash
strace -f -o ~/XMRig/analysis/xmrig-strace.txt \
  ./xmrig --bench=1M
```



The resulting trace contained approximately 7,647 lines.

### 7.1 Process and Thread Creation

The trace recorded a successful `execve()` of the XMRig executable and multiple `clone3()` calls with thread-creation flags.

These observations confirmed that the executable started and created multiple threads during the benchmark.

### 7.2 File and System Discovery

The trace showed accesses to system and configuration paths, including:

- `/etc/ld.so.cache`
- `/etc/localtime`
- `/proc/cpuinfo`
- `/proc/cmdline`
- `/sys/devices/system/cpu/`

These accesses were consistent with library loading, environment inspection, and CPU or hardware discovery.

Some access attempts returned `ENOENT` or `EACCES`. These errors indicate missing paths or denied access and are not inherently evidence of malicious behaviour.

### 7.3 Network-Related Activity

The trace recorded a Netlink socket and requests using `RTM_GETLINK` and `RTM_GETADDR`.

These operations query local network-interface and address information. They are distinct from establishing a remote connection to a mining pool.

A search for `connect()` in the recorded trace returned no matches. No active network socket associated with the XMRig process was observed during the corresponding benchmark inspection.

**Finding:** No remote connection was observed during the recorded benchmark. Because the sample was run in benchmark mode without a mining-pool configuration, this finding should not be generalized to every possible XMRig execution mode.

### 7.4 MSR Optimization Attempt

The trace showed XMRig attempting to write to:

```text
/sys/module/msr/parameters/allow_writes
```

The operation returned `EACCES` (permission denied). The trace subsequently showed execution of `/bin/sh` and `/sbin/modprobe` with the arguments `msr allow_writes=on`.

The associated helper process inspected module configuration files under `/etc/modprobe.d/`.

This activity corresponded with the MSR optimization attempt reported during XMRig initialization.

The evidence demonstrates an attempted optimization operation that encountered a permission restriction. It does not demonstrate successful privileged modification, privilege escalation, or persistence.

## 8. Detection Opportunities

The investigation identified several behaviours that may support a defensive cryptomining investigation.

| Behaviour | Potential telemetry | Investigative value |
|---|---|---|
| Sustained high CPU consumption | `htop`, process telemetry, system metrics | Identify resource-intensive processes |
| Large memory footprint | Process and memory telemetry | Correlate memory use with executable behaviour |
| Multiple worker threads | Process telemetry | Understand computational activity |
| Mining-related execution arguments | Process execution logs | Identify the command and execution mode |
| Connections to mining infrastructure | Firewall, DNS, and network telemetry | Investigate remote destinations when connections occur |
| MSR-related access attempts | System-call auditing and endpoint telemetry | Investigate the process and the permission outcome |

No single indicator is sufficient to establish cryptojacking. Legitimate computational workloads can exhibit high CPU utilization, substantial memory consumption, and multiple threads.

A reliable investigation should correlate resource usage with process identity, execution context, network activity, authorization, and the expected role of the host.


## 9. Findings and Evidence Classification

| Finding | Evidence | Conclusion |
|---|---|---|
| XMRig executed in benchmark mode | Process observation and system-call trace | Confirmed |
| Multiple threads were created | `clone3()` calls and benchmark output | Confirmed |
| Substantial memory was allocated | Process measurements and benchmark output | Confirmed |
| Final benchmark rate was 533.6 H/s | XMRig benchmark output | Confirmed |
| Local network-interface queries occurred | Netlink system calls | Confirmed |
| A remote `connect()` occurred during the recorded trace | `strace` | Not observed |
| MSR write access was denied | `EACCES` in the trace | Confirmed |
| `modprobe` was invoked for the MSR operation | `execve()` in the trace | Confirmed |
| Successful mining-pool communication occurred | Runtime network evidence | Not demonstrated |
| Unauthorized mining occurred | Incident evidence | Not established |

The conclusions are limited to the specific execution and evidence collected. A behaviour that was not observed in this benchmark should not automatically be considered absent from every other configuration or execution mode.

## 10. Limitations

The investigation used the official benchmark mode in an isolated Linux environment. It did not deploy XMRig against a real mining pool or configure a wallet.

As a result, the investigation did not demonstrate mining-pool communication or unauthorized mining. Resource measurements are specific to the virtual-machine configuration, and the detection opportunities described here represent proposed defensive approaches rather than confirmed alert outcomes.

## 11. Conclusion

Dynamic analysis showed XMRig executing a memory-intensive, multithreaded RandomX benchmark, reaching a final performance rate of 533.6 H/s in the virtual-machine configuration.

Process monitoring, `/proc` inspection, and `strace` provided complementary evidence of resource consumption, thread creation, system discovery, local network-interface queries, and an MSR optimization attempt that encountered a permission restriction.

No remote connection was observed during the recorded benchmark session. The findings demonstrate the value of combining process monitoring, system-call analysis, and network inspection when investigating suspected cryptomining.

The next phase consolidates the static and dynamic evidence into a case-study report, with explicit distinctions between observed behaviour, identified capabilities, and conclusions that the available evidence does not support.
