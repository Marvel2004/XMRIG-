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

</br>
</br>
<img width="1358" height="372" alt="Screenshot from 2026-10-19 07-36-28" src="https://github.com/user-attachments/assets/97030f7e-bd30-4213-865c-d2679c0bb50f" />

</br>

<img width="1920" height="1080" alt="Screenshot from 2026-10-19 07-39-00" src="https://github.com/user-attachments/assets/44321b92-033f-4128-8afd-83169cf26da3" />

</br>
</br>
<img width="1920" height="1080" alt="Screenshot from 2026-10-19 07-39-25" src="https://github.com/user-attachments/assets/b55cac61-e505-4c19-b161-5d9d8793b0e9" />
</br>
</br>

During one baseline observation, Suricata consumed approximately **_101% CPU and 10.7% memory_**. This was important because it demonstrated that background security monitoring could contribute substantially to the system's overall CPU usage.

Consequently, system-wide CPU utilization was interpreted alongside individual process activity rather than being attributed to XMRig automatically.

</br>
</br>
<img width="1920" height="1080" alt="Screenshot from 2026-10-19 07-39-57" src="https://github.com/user-attachments/assets/f0783f49-69e2-41b7-9c52-1809853986e2" />
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

</br>
</br>
<img width="924" height="591" alt="Screenshot from 2026-10-02 21-54-21 - Copy" src="https://github.com/user-attachments/assets/81a41f08-94d0-47cf-b99d-a21fd826a25b" />
</br>
</br>

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
cat /proc/3250/status
```

```bash
cat /proc/3250/io
```

```bash
ls -l /proc/3250/fd
```

```bash
cat /proc/3250/maps
```

These interfaces provided information about process memory, thread count, I/O counters, open file descriptors, and mapped memory regions.

The improved run showed approximately 2.31 GiB of resident memory and negligible swap usage compared with the earlier constrained environment. This was consistent with the reduced memory pressure after increasing the virtual-machine resources.

The process I/O counters and file descriptors were also examined to understand the process's interaction with the operating system. These measurements were interpreted in context rather than assuming that every recorded read or open descriptor represented suspicious file activity.

</br>
</br>
<img width="1920" height="1145" alt="Screenshot from 2026-10-02 22-35-31" src="https://github.com/user-attachments/assets/5b457c9d-e460-4f2a-8ee4-ff480cff5052" />
</br>
</br>

## 7. System-Call Analysis Using `strace`

To examine interactions between XMRig and the Linux kernel, the benchmark was executed under `strace`, a Linux utility that records system calls and signals.

```bash
strace -f -o ~/XMRig/analysis/xmrig-strace.txt \
  ./xmrig --bench=1M
```

- `-f` follows child processes and threads.
- `-o` writes the trace to the specified file.
- `--bench=1M` runs XMRig in benchmark mode using the 1M workload.

The resulting trace contained **7,647 lines**. Targeted searches were used to investigate process execution, thread creation, file access, networking activity, and permission-related operations.

### 7.1 Trace Size and Termination

**Commands**

```bash
wc -l ~/XMRig/analysis/xmrig-strace.txt
tail ~/XMRig/analysis/xmrig-strace.txt
```

**Purpose:** `wc -l` counts the lines in the trace, while `tail` displays the final lines to help inspect activity near the end of execution.

**Findings:** The trace contained 7,647 lines. The final lines were examined for process termination, errors, and other system calls near the end of the recorded execution.

**Interpretation:** The line count represents the volume of recorded trace output, not the number of unique system calls or threads. The final lines provide termination context but cannot explain the program's entire execution independently.

</br>
</br>
<img width="1919" height="316" alt="Screenshot from 2026-10-19 08-42-19 - Copy" src="https://github.com/user-attachments/assets/20545f5d-8da7-4f6b-857b-ff8bc169a115" />
</br>
</br>

### 7.2 Process and Thread Creation

**Command**

```bash
grep -E 'execve\(|clone3\(' \
  ~/XMRig/analysis/xmrig-strace.txt
```

**Purpose:** Identify program execution through `execve()` and process or thread creation through `clone3()`.

**Findings:** The trace recorded a successful execution of the XMRig benchmark command:

```text
execve("./xmrig", ["./xmrig", "--bench=1M"], ...)
```

Repeated `clone3()` calls containing thread-related flags, including `CLONE_THREAD`, were also observed.

**Interpretation:** The `execve()` record confirms that the benchmark command was executed under tracing. The repeated thread-creation calls are consistent with XMRig's multithreaded workload.

This finding aligns with the benchmark output reporting five mining worker threads and seven total process threads.

</br>
</br>
<img width="1911" height="513" alt="Screenshot from 2026-10-19 08-42-19 - Copy (2)" src="https://github.com/user-attachments/assets/665d9dbe-3189-43f7-973d-278e365a6e8c" />
</br>
</br>

### 7.3 File and System Discovery

**Command**

```bash
grep -E 'openat\(|open\(|access\(' \
  ~/XMRig/analysis/xmrig-strace.txt
```

**Purpose:** Identify file-access operations and path checks performed during execution.

**Findings:** The trace showed accesses involving the following system paths:

- `/etc/ld.so.cache`
- `/etc/localtime`
- `/proc/cpuinfo`
- `/proc/cmdline`
- `/sys/devices/system/cpu/`

**Interpretation:** These accesses are consistent with library loading, environment inspection, and CPU or hardware discovery.

For example, `/proc/cpuinfo` exposes processor information, while `/sys/devices/system/cpu/` provides access to CPU-related system information. `/etc/ld.so.cache` is associated with dynamic library resolution.

Some access attempts returned `ENOENT` or `EACCES`:

- `ENOENT` indicates that a requested path or file was not found.
- `EACCES` indicates that access was denied by the applicable permissions.

These errors are not inherently evidence of malicious behaviour. The return value of each system call must be considered when determining whether an operation succeeded.

</br>
</br>
<img width="1917" height="343" alt="Screenshot from 2026-10-19 08-44-57 - Copy" src="https://github.com/user-attachments/assets/b7ebe6cd-548d-498f-ba28-c779fa5e4d45" />
</br>
</br>
<img width="1918" height="616" alt="Screenshot from 2026-10-19 08-45-52 - Copy" src="https://github.com/user-attachments/assets/018048f1-446d-454e-ad68-e06fe1b973c3" />
</br>
</br>

### 7.4 Network-Related Activity

**Commands**

```bash
grep 'connect(' ~/XMRig/analysis/xmrig-strace.txt
```

returned nothing.
</br>
To search for a broader set of networking-related calls:

```bash
grep -E 'socket\(|connect\(|sendto\(|recvfrom\(|sendmsg\(|recvmsg\(' \
  ~/XMRig/analysis/xmrig-strace.txt
```

</br>
</br>
<img width="1918" height="274" alt="Screenshot from 2026-10-19 08-42-19 - Copy (3)" src="https://github.com/user-attachments/assets/5ccab00d-1be2-4e49-bb34-b3a5b79da230" />
</br>
</br>
<img width="1920" height="1080" alt="Screenshot from 2026-10-19 08-42-45" src="https://github.com/user-attachments/assets/f58c9085-9932-4aec-92c5-0365babff371" />
</br>
</br>

The active socket table was also inspected during execution:

```bash
sudo ss -tunap | grep 3250
```

**Purpose:** Search for connection attempts and other networking-related system calls, then check for active TCP/UDP sockets associated with the process at the time of observation.

**Findings:**

- The search for `connect()` returned no matching calls.
- No active network socket associated with the XMRig process was observed in the corresponding socket snapshot.
- The trace recorded Netlink operations resembling:

```text
socket(AF_NETLINK, SOCK_RAW|SOCK_CLOEXEC, NETLINK_ROUTE)
sendto(... RTM_GETLINK ...)
sendto(... RTM_GETADDR ...)
```

**Interpretation:** The Netlink operations query local networking information, including network interfaces and addresses. They are distinct from establishing a remote connection to a mining pool.

No remote connection was observed during the recorded benchmark. XMRig was executed in benchmark mode without a configured mining pool, so successful pool communication was not demonstrated.

These observations are limited to the recorded trace and socket snapshot. They do not establish that XMRig lacks networking capabilities or would never communicate with a remote endpoint in another configuration.


### 7.5 MSR Optimization Attempt

**Command**

```bash
grep -E 'EACCES|allow_writes|modprobe.*msr' \
  ~/XMRig/analysis/xmrig-strace.txt
```

**Purpose:** Investigate permission failures and MSR-related operations recorded during execution.

**Findings:** The trace showed an attempt to open:

```text
/sys/module/msr/parameters/allow_writes
```

The operation returned:

```text
EACCES (Permission denied)
```

The trace subsequently recorded execution of `/bin/sh` and `/sbin/modprobe` with the arguments:

```text
/sbin/modprobe msr allow_writes=on
```

The associated helper process also inspected module configuration files under `/etc/modprobe.d/`.

**Interpretation:** MSR stands for *Model-Specific Register*. The observations correspond to an MSR-related operation associated with CPU optimisation during XMRig's execution.

The permission failure establishes that the recorded access attempt was denied. The subsequent `modprobe` invocation establishes that the helper command was launched, but does not establish that privileged modification succeeded.

This activity should not be classified as confirmed privilege escalation, persistence, or successful system modification without additional evidence.

</br>
</br>
<img width="1919" height="747" alt="Screenshot from 2026-10-19 08-44-57" src="https://github.com/user-attachments/assets/6339d312-7b58-49c0-abd0-1c2f6ca0065e" />
</br>
</br>

### 7.6 Summary of System-Call Findings

| Investigation area | Observation | Significance |
|---|---|---|
| Trace volume | 7,647 lines | Establishes the size of the recorded output |
| Process execution | `execve()` launched `./xmrig --bench=1M` | Confirms execution of the benchmark command |
| Thread creation | Repeated thread-related `clone3()` calls | Consistent with concurrent processing |
| File and system discovery | Accesses to library, process, and CPU-related paths | Consistent with environment discovery |
| Remote connection search | No matching `connect()` calls | No connection attempt was observed through this call in the trace |
| Socket inspection | No associated active socket observed | No active socket was identified in the captured snapshot |
| Local network discovery | Netlink requests using `RTM_GETLINK` and `RTM_GETADDR` | Consistent with local interface and address queries |
| MSR operation | Access denied; `modprobe` subsequently invoked | MSR-related optimisation was attempted, but successful modification was not established |

</br>

## 8. Detection Opportunities

The investigation identified behaviours that may be useful when investigating suspected cryptomining activity.

| Behaviour | Potential telemetry | Investigative value |
|---|---|---|
| Sustained high CPU consumption | `htop`, process telemetry, system metrics | Identify resource-intensive processes |
| Large memory footprint | Process and memory telemetry | Correlate memory consumption with executable behaviour |
| Multiple worker threads | Process telemetry | Understand concurrent computational activity |
| Mining-related execution arguments | Process execution logs | Identify the command and execution mode |
| Connections to mining infrastructure | Firewall, DNS, and network telemetry | Investigate remote destinations when connections occur |
| MSR-related access attempts | System-call auditing and endpoint telemetry | Investigate the process and permission outcome |

No single indicator is sufficient to establish cryptojacking. Legitimate computational workloads can also exhibit high CPU usage, substantial memory consumption, and multiple threads.

A reliable investigation should correlate resource consumption with process identity, executable location, execution context, network activity, authorization, and the expected role of the host.

**Detection scope:** These are proposed defensive investigation opportunities, not claims that a Wazuh or other security alert was generated during this experiment.

</br>

## 9. Findings and Evidence Classification

| Finding | Evidence | Conclusion |
|---|---|---|
| XMRig executed in benchmark mode | Process observation and system-call trace | Confirmed |
| Multiple threads were created | `clone3()` calls and benchmark output | Confirmed |
| Substantial memory was consumed | Process measurements and benchmark output | Confirmed |
| Final benchmark rate was 533.6 H/s | Completed XMRig benchmark output | Confirmed, subject to verification against the final output |
| Local network-interface queries occurred | Netlink system calls | Confirmed |
| A remote connection occurred during the recorded trace | `strace` output | Not observed |
| An active socket was associated with XMRig during the captured inspection | `ss` output | Not observed |
| MSR write access was denied | `EACCES` in the trace | Confirmed |
| `modprobe` was invoked for the MSR operation | `execve()` in the trace | Confirmed |
| Successful mining-pool communication occurred | Runtime network evidence | Not demonstrated |
| Unauthorized mining occurred | Incident evidence | Not established |
| Successful privilege escalation or persistence occurred | Supporting runtime or system evidence | Not established |

These conclusions are limited to the specific execution and evidence collected. Behaviour not observed during this benchmark should not automatically be considered absent from every other configuration or execution mode.

</br>

## 10. Limitations

The investigation used XMRig's official benchmark mode in a controlled Linux environment. It did not configure a real mining pool or wallet.

Consequently, the experiment did not demonstrate successful mining-pool communication, cryptocurrency earnings, or unauthorized mining. The resource measurements reflect the virtual-machine configuration and execution conditions used during the test.

The detection opportunities described in this report are proposed defensive approaches rather than confirmed security-alert outcomes.

The absence of a matching `connect()` call or an active socket in a captured snapshot is also not proof that the executable has no networking capability. The result applies to the observed benchmark session.

</br>

## 11. Conclusion

Dynamic analysis showed XMRig executing a memory-intensive, multithreaded RandomX benchmark, with a reported final performance rate of **533.6 H/s**, subject to confirmation against the completed benchmark output.

Process monitoring, memory inspection, and `strace` provided complementary evidence of resource consumption, thread creation, system discovery, local network-interface queries, and an MSR-related optimisation attempt that encountered a permission restriction.

No remote connection was observed during the recorded benchmark session, and successful mining-pool communication was not demonstrated.

The investigation illustrates the value of combining process monitoring, system-call analysis, and network inspection when examining suspected cryptomining activity. Most importantly, it separates observed behaviour from inferred capability and avoids treating individual system calls or resource measurements as proof of a confirmed cryptojacking incident.
