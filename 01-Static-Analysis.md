# XMRIG Cryptominer - Static Analysis

## 1. Objective and scope

This stage examined the XMRig executable without relying on runtime behaviour. The aim was to identify the artifact, inspect its ELF structure and dependencies, and use embedded strings and dynamic symbols to identify functionality worth investigating during dynamic analysis.

**Context:** XMRig is legitimate open-source cryptocurrency-mining software. Its functionality can be abused in cryptojacking, but the presence of XMRig alone does not establish malicious deployment. This report distinguishes what the binary contains from what was observed when it ran.

</br>

## 2. Sample identification and integrity

```bash
file xmrig
sha256sum xmrig
```

`file` identifies the format and architecture using file signatures and metadata. `sha256sum` calculates a hash that can be used to verify that the same artifact is being examined later.

The sample investigated was XMRig 6.26.0, from the official Linux Jammy x64 release. The SHA-256 value recorded during acquisition was:

```text
ca82fc8426187880dffa502363849af6258e65fdb675a9cc9984a2b843854087
```
</br>
</br>
<img width="1920" height="1080" alt="Screenshot from 2026-08-29 15-50-05" src="https://github.com/user-attachments/assets/871c23e4-3c78-4dd6-86cf-047eac789114" />

</br>
</br>

## 3. ELF header: architecture and executable type

```bash
readelf -h xmrig
```

The `-h` option displays the ELF header. The evidence shows an ELF64, little-endian executable for AMD x86-64. Its ELF type is `DYN`, consistent with a position-independent executable (PIE) on modern Linux systems.

Why this matters:

- **ELF64** identifies the 64-bit ELF format.
- **Little-endian** describes how multi-byte values are represented in the file.
- **AMD x86-64** identifies the target processor architecture.
- **DYN** identifies the ELF object type. In this executable context, it is consistent with PIE; it does not mean the file is merely a shared library.

These details establish the binary's technical format and intended platform. They are not indicators of maliciousness by themselves.

## 4. ELF sections: internal structure

```bash
readelf -S xmrig
```

The `-S` option lists ELF section headers. The captured output reports **33 section headers**. Sections organise content such as executable code, read-only data, writable data, symbol tables, and relocation information.

Examples of common section names include `.text` (typically executable code), `.rodata` (often constants and strings), `.data` (initialised writable data), and `.bss` (zero-initialised data). Refer to the actual section names and sizes in the screenshot rather than assuming every common section is present.

The section table is useful structural context, but section names and sizes do not independently establish malicious behaviour.

</br>
</br>
<img width="1920" height="1080" alt="Screenshot from 2026-08-29 15-54-12" src="https://github.com/user-attachments/assets/9a9052fc-213b-4b9e-9dd2-bb252eef6391" />

</br>
</br>

## 5. Dynamic dependencies

```bash
readelf -d xmrig
ldd xmrig
```

`readelf -d` displays the dynamic section, including dynamic-linking information. `ldd` shows shared libraries resolved for the executable in the current environment.

These commands help document the executable's relationship with system libraries and may expose compatibility issues or dependencies worth further investigation. The exact dependency names should be copied from the captured output. A library being present is not proof that a particular function was executed, and dependency resolution can vary between environments.

</br>
</br>

<img width="1920" height="1080" alt="Screenshot from 2026-10-19 08-00-19" src="https://github.com/user-attachments/assets/7e903bd4-e1c0-4869-bc54-9035a88e6efe" />

</br>
</br>


## 6. Embedded strings: mining-related indicators

```bash
strings -a xmrig | grep -Ei \
'RandomX|Monero|Stratum|pool|wallet|TLS|HTTP|API'
```

`strings -a` extracts readable character sequences from the file. The case-insensitive `grep` filter narrows the output to terms relevant to mining and communication functionality.

The investigation identified references associated with:

- **RandomX and Monero:** consistent with XMRig's Monero mining functionality.
- **Stratum and pool terminology:** consistent with mining-pool communication functionality.
- **Wallet configuration:** consistent with configuration for the destination of mining rewards.
- **TLS and HTTP API:** references associated with encrypted communication and an HTTP-based interface

Together, these strings support the conclusion that the binary contains mining-related functionality and configuration/communication references. Strings are leads, not proof of execution: they do not demonstrate that a pool was contacted, a share was submitted, cryptocurrency was earned, or a remote API was accessed during the observed run.

</br>
</br>
<img width="1920" height="1080" alt="Screenshot from 2026-10-19 08-02-06" src="https://github.com/user-attachments/assets/df24069c-720a-4aa8-9da1-58c09494077e" />
</br>
</br>
<img width="1920" height="1080" alt="Screenshot from 2026-10-19 08-29-50" src="https://github.com/user-attachments/assets/fd7b25ae-532f-48a9-aa5b-e7dcfd564ec7" />
</br>
</br>
<img width="1920" height="1080" alt="Screenshot from 2026-10-19 08-31-14" src="https://github.com/user-attachments/assets/fd4127c1-88ce-4dd5-81b9-9c71c607f520" />
</br>
</br>

## 7. Dynamic symbols: functionality suggested by imports

```bash
nm -D xmrig | grep -Ei \
'socket|connect|recv|send|pthread|malloc|free|open|close|dns'

objdump -T xmrig | grep -Ei \
'socket|connect|recv|send|pthread|malloc|free|open|close|ssl|crypto|curl'
```

`nm -D` lists dynamic symbols; `objdump -T` displays the dynamic symbol table. Filtering makes it easier to examine function names associated with networking, threading, memory management, and file operations.

The captured symbol output included examples in these groups:

| Group | Examples from the investigation | Interpretation |
|---|---|---|
| Networking | `socket`, `connect`, `send`, `sendmsg`, `recvmsg`, `sendmmsg` | Networking interfaces are available to the program. |
| Threading | `pthread_create`, `pthread_join`, `pthread_detach`, mutex/condition functions | Consistent with concurrent work and coordination between threads. |
| Memory | `malloc`, `free`, `realloc` | Dynamic memory-management interfaces. |
| File I/O | `open`, `open64`, `openat`, `close`, `fdopen`, `fclose` | File-handling interfaces. |

These groups are consistent with a computationally intensive program that uses multiple threads and has networking and file-handling functionality. They do not prove that each imported function was called, nor that any network operation was malicious.

</br>
</br>
<img width="1920" height="1080" alt="Screenshot from 2026-10-19 08-03-01" src="https://github.com/user-attachments/assets/92627b6f-b831-4f98-b58a-7b1304cb2829" />
</br>
</br>
<img width="1920" height="1080" alt="Screenshot from 2026-10-19 08-04-00" src="https://github.com/user-attachments/assets/73f70341-fb04-4817-a835-dd46a4928108" />
</br>
</br>

## 8. Static-analysis findings

The static investigation established that:

1. The analyzed artifact was the XMRig 6.26.0 Linux x64 release.
2. The ELF header identified an ELF64, little-endian AMD x86-64 executable with type `DYN`, consistent with PIE.
3. The section-header output reported 33 sections.
4. Embedded strings included RandomX, Monero, pool/Stratum, wallet, TLS, and HTTP API-related references.
5. Dynamic symbols included networking, threading, memory-management, and file-I/O interfaces.

Taken together, these findings are consistent with XMRig's role as cryptocurrency-mining software. Static analysis did not establish successful mining-pool communication, cryptocurrency earnings, unauthorized deployment, persistence, or privilege escalation. Those questions require separate runtime evidence and contextual investigation.

## 9. Evidence and screenshot checklist

- [ ] Sample identification and archive hash, with the hashed filename visible.
- [ ] ELF header showing class, endianness, type, and machine.
- [ ] Section table showing the reported count of 33 section headers.
- [ ] Mining-related strings (RandomX, Monero, pool/Stratum, wallet/API references).
- [ ] Selected networking and threading symbols.
- [ ] Dependency output, if it adds useful information beyond routine library names.

## References

- XMRig repository: https://github.com/xmrig/xmrig
- XMRig documentation: https://xmrig.com/docs
- `readelf` manual: https://man7.org/linux/man-pages/man1/readelf.1.html
- `ldd` manual: https://man7.org/linux/man-pages/man1/ldd.1.html
- GNU Binutils documentation: https://sourceware.org/binutils/docs/
