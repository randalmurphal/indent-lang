# io_uring Container Restrictions and Fallback Strategy Research

## Executive Summary

io_uring is blocked by default in most modern container environments due to its extensive history of kernel vulnerabilities. Google reported that 60% of kernel exploits submitted to their bug bounty program in 2022 targeted io_uring, leading to widespread restrictions. This document provides concrete strategies for Indent's async runtime to detect availability and fall back gracefully.

---

## 1. Current Container Restrictions

### 1.1 Docker Engine (Linux)

**Blocked since Docker 25.0.0** (January 2024)

Docker 25.0.0 and newer blocks io_uring by default using seccomp. The change syncs the seccomp profile with changes made to containerd's default profile.

The three io_uring syscalls blocked:
- `io_uring_setup`
- `io_uring_enter`  
- `io_uring_register`

**Error manifestation**: Applications receive `EPERM` (Operation not permitted) when calling `io_uring_setup()`.

### 1.2 Docker Desktop (macOS/Windows)

**Additional restrictions since Docker Desktop 4.42.0** (June 2025)

Docker 4.42.0 release notes states: "Block io_uring syscalls in containers." Even using `--security-opt seccomp=unconfined` doesn't work.

Docker Desktop 4.42.0+ blocks io_uring even with `--privileged --security-opt seccomp=unconfined`. This appears to be a VM-level restriction, not just seccomp.

**Key difference**: Docker Desktop uses a Linux VM to run containers. The VM itself may have io_uring disabled at the kernel level via sysctl, making it impossible to enable even with seccomp bypass.

### 1.3 containerd

The containerd project removed io_uring syscalls from the RuntimeDefault seccomp profile due to the security concerns raised by Google's research.

### 1.4 Podman

Podman containers also block io_uring by default. Applications like MariaDB show warnings: "mysqld: io_uring_queue_init() failed with ENOSYS: check seccomp filters"

### 1.5 Kubernetes

**Pod Security Standards**: Kubernetes doesn't mandate seccomp profiles by default, but:

In Autopilot clusters, GKE automatically applies the containerd default seccomp profile to all workloads. No further action is required. Attempts to make restricted syscalls fail. Autopilot disallows custom seccomp profiles because GKE manages the nodes.

For the Restricted Pod Security Standard level, pods MUST define a seccomp profile using `seccompProfile: type: RuntimeDefault` or `Localhost`.

### 1.6 Cloud Provider Summary

| Platform | io_uring Status | Can Enable? | Notes |
|----------|-----------------|-------------|-------|
| **GKE Standard** | Blocked by default | Yes, with custom seccomp | Requires node access |
| **GKE Autopilot** | Blocked | No | Custom seccomp profiles disallowed |
| **AWS Fargate** | Likely blocked | No | No custom seccomp support |
| **AWS EKS** | Blocked by default | Yes | With custom seccomp profiles |
| **Azure AKS** | Blocked by default | Yes | Via custom seccomp config |
| **Fly.io** | Unknown | Varies | Likely follows containerd defaults |
| **Railway** | Unknown | Unlikely | Managed platform |
| **Render** | Unknown | Unlikely | Managed platform |

**AWS Fargate specific**: Tasks on Fargate only support adding the SYS_PTRACE kernel capability. No custom seccomp profile support is documented.

---

## 2. Security Concerns and CVE Timeline

### 2.1 The Seccomp Bypass Issue

A side effect is that io_uring effectively bypasses the protections provided by seccomp filtering — we can't filter out syscalls we never make! This isn't a security vulnerability per se, but something you should keep in mind if you have especially paranoid seccomp rules.

io_uring allows operations that would normally require syscalls to be performed via io_uring opcodes instead. This means:
- A process could perform `connect()` via `IORING_OP_CONNECT` even if the `connect` syscall is blocked
- File operations can bypass file-related seccomp restrictions
- Network operations can bypass network-related seccomp restrictions

### 2.2 Google's Findings

Security experts generally believe io_uring to be unsafe. In fact Google ChromeOS and Android have turned it off, plus all Google production servers turn it off. Based on the blog published by Google, it seems like a bunch of vulnerabilities related to io_uring can be exploited to break out of the container.

Google noted that 60% of kernel exploits submitted to their bug bounty in 2022 targeted io_uring. This underscores the need for careful risk assessment and robust, modern monitoring techniques when dealing with io_uring.

### 2.3 Notable CVEs

| CVE | Year | Type | Impact |
|-----|------|------|--------|
| CVE-2021-41073 | 2021 | Memory handling | Local Privilege Escalation |
| CVE-2022-3910 | 2022 | File refcount | Container escape |
| CVE-2023-2598 | 2023 | Out-of-bounds access | Privilege escalation |
| CVE-2022-20409 | 2022 | Use-after-free | Android rooting |
| CVE-2023-21400 | 2023 | Various | Android exploitation |

### 2.4 Current Status

**Is it fixed in newer kernels?** No. While individual CVEs get patched, io_uring is complex and has had a significant history of security vulnerabilities since its introduction, many leading to Local Privilege Escalation (LPE).

The attack surface is inherently large due to io_uring's design as a general-purpose async syscall interface.

---

## 3. Detection Strategies

### 3.1 Recommended: Probe at Startup

```c
// Pseudo-code for io_uring availability detection
bool detect_io_uring_available() {
    struct io_uring_params params = {0};
    int fd = syscall(__NR_io_uring_setup, 1, &params);
    
    if (fd >= 0) {
        close(fd);
        return true;
    }
    
    // Check specific error codes
    switch (errno) {
        case EPERM:
            // Blocked by seccomp or sysctl io_uring_disabled
            log_info("io_uring blocked by security policy");
            return false;
        case ENOSYS:
            // Kernel doesn't support io_uring (< 5.1)
            log_info("io_uring not supported by kernel");
            return false;
        case ENOMEM:
            // Resource limit, but io_uring is available
            log_warn("io_uring available but resource constrained");
            return true; // May work with smaller ring
        default:
            log_warn("io_uring probe failed: %s", strerror(errno));
            return false;
    }
}
```

### 3.2 Error Codes to Handle

EPERM - /proc/sys/kernel/io_uring_disabled has the value 2, or it has the value 1 and the calling process does not hold the CAP_SYS_ADMIN capability or is not a member of /proc/sys/kernel/io_uring_group.

| Error Code | Meaning | Action |
|------------|---------|--------|
| `EPERM` | Blocked by seccomp or sysctl | Fall back to epoll |
| `ENOSYS` | Kernel < 5.1 or syscall not available | Fall back to epoll |
| `ENOMEM` | Insufficient memory | Try smaller ring or fall back |
| `EINVAL` | Invalid parameters | Fix parameters, not availability issue |

### 3.3 sysctl Detection

The kernel provides sysctl `io_uring_disabled` which can be 0 (allowed), 1 (requires CAP_SYS_ADMIN or io_uring_group membership), or 2 (disabled for all processes).

```c
bool check_sysctl_io_uring() {
    FILE *f = fopen("/proc/sys/kernel/io_uring_disabled", "r");
    if (!f) {
        // sysctl doesn't exist - older kernel or not configured
        return true; // Proceed with probe
    }
    int disabled;
    fscanf(f, "%d", &disabled);
    fclose(f);
    return disabled != 2;
}
```

### 3.4 Environment Variable Override

Provide explicit user control:

```c
typedef enum {
    IO_BACKEND_AUTO,    // Probe and select
    IO_BACKEND_IOURING, // Force io_uring (fail if unavailable)
    IO_BACKEND_EPOLL    // Force epoll
} io_backend_t;

io_backend_t get_configured_backend() {
    const char *env = getenv("INDENT_IO_BACKEND");
    if (!env) return IO_BACKEND_AUTO;
    if (strcmp(env, "iouring") == 0) return IO_BACKEND_IOURING;
    if (strcmp(env, "epoll") == 0) return IO_BACKEND_EPOLL;
    return IO_BACKEND_AUTO;
}
```

### 3.5 Cannot Reliably Check Seccomp from Userspace

There's no portable way to query whether a specific syscall is blocked by seccomp filters. The only reliable method is to attempt the syscall and handle the error.

---

## 4. Runtime Switching Considerations

### 4.1 Switching Mid-Execution

**Not recommended.** The io_uring and epoll programming models are fundamentally different:

Because io_uring differs significantly from epoll, Tokio must provide a new set of APIs to take full advantage of the reduced overhead.

Key differences:
- **Buffer ownership**: io_uring requires passing buffer ownership to kernel
- **Completion model**: io_uring is completion-based, epoll is readiness-based
- **Syscall batching**: io_uring batches operations, epoll handles one at a time

### 4.2 Decision at Runtime Init

**Recommended approach**: Detect availability once at startup and configure the runtime accordingly.

```c
// At program initialization
static io_backend_t selected_backend;

void init_async_runtime() {
    io_backend_t requested = get_configured_backend();
    
    switch (requested) {
        case IO_BACKEND_IOURING:
            if (!detect_io_uring_available()) {
                fatal("io_uring requested but not available");
            }
            selected_backend = IO_BACKEND_IOURING;
            break;
            
        case IO_BACKEND_EPOLL:
            selected_backend = IO_BACKEND_EPOLL;
            break;
            
        case IO_BACKEND_AUTO:
        default:
            selected_backend = detect_io_uring_available() 
                ? IO_BACKEND_IOURING 
                : IO_BACKEND_EPOLL;
            log_info("Auto-selected backend: %s", 
                selected_backend == IO_BACKEND_IOURING ? "io_uring" : "epoll");
            break;
    }
}
```

### 4.3 What tokio-uring Does

Creating the tokio-uring runtime initializes the io-uring submission and completion queues and a Tokio current-thread epoll-based runtime. Instead of waiting on completion events by blocking the thread on the io-uring completion queue, the tokio-uring runtime registers the completion queue with the epoll handle.

tokio-uring doesn't provide automatic fallback—it requires io_uring and will fail if unavailable. This is an explicit design choice:

Applications deployed exclusively on Linux kernels 5.10 or later may choose to use this crate when taking full advantage of io-uring's benefits provides measurable benefits. Examples of intended use-cases include TCP proxies, HTTP file servers, and databases.

### 4.4 Monoio's Approach

On Linux 5.6 or newer, Monoio can use uring or epoll as io driver. On lower versions of Linux, it can only run in epoll mode.

Monoio (ByteDance's async runtime) provides compile-time backend selection with runtime feature detection.

---

## 5. Seccomp Profile Configuration

### 5.1 Syscalls to Allowlist

To enable io_uring in containers, add these syscalls to your seccomp profile:

```json
{
  "names": [
    "io_uring_setup",
    "io_uring_enter",
    "io_uring_register"
  ],
  "action": "SCMP_ACT_ALLOW"
}
```

### 5.2 Complete Custom Seccomp Profile

To enable io_uring, add the syscalls to Docker's default profile: "io_uring_enter", "io_uring_register", "io_uring_setup"

Create `io_uring_enabled_profile.json`:

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "archMap": [
    {
      "architecture": "SCMP_ARCH_X86_64",
      "subArchitectures": [
        "SCMP_ARCH_X86",
        "SCMP_ARCH_X32"
      ]
    },
    {
      "architecture": "SCMP_ARCH_AARCH64",
      "subArchitectures": [
        "SCMP_ARCH_ARM"
      ]
    }
  ],
  "syscalls": [
    {
      "names": [
        "io_uring_setup",
        "io_uring_enter", 
        "io_uring_register"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
    // ... include all other syscalls from Docker's default profile
  ]
}
```

**Recommended approach**: Start with Docker's default profile and add the three io_uring syscalls rather than creating from scratch.

### 5.3 Docker Usage

```bash
# Run with custom seccomp profile
docker run --security-opt seccomp=io_uring_enabled_profile.json myimage

# Or disable seccomp entirely (NOT RECOMMENDED for production)
docker run --security-opt seccomp=unconfined myimage
```

### 5.4 Kubernetes Configuration

**For GKE Standard / EKS / AKS with custom profiles:**

1. Deploy seccomp profile to nodes at `/var/lib/kubelet/seccomp/profiles/`:

```yaml
# Deploy via DaemonSet or node configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: io-uring-seccomp-profile
data:
  io-uring-enabled.json: |
    {
      "defaultAction": "SCMP_ACT_ERRNO",
      "syscalls": [
        {
          "names": ["io_uring_setup", "io_uring_enter", "io_uring_register"],
          "action": "SCMP_ACT_ALLOW"
        }
        // ... rest of profile
      ]
    }
```

2. Reference in Pod spec:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: indent-app
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/io-uring-enabled.json
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
      runAsNonRoot: true
```

### 5.5 GKE Autopilot Limitation

Autopilot disallows custom seccomp profiles because GKE manages the nodes.

**Workaround**: Use GKE Standard mode if io_uring is required.

---

## 6. Security Tradeoffs

### 6.1 Is Allowing io_uring Safe Now?

**Short answer**: It's a risk-benefit tradeoff.

**Risks**:
- io_uring's complexity has led to numerous bugs, many allowing local privilege escalation (LPE).
- New vulnerabilities continue to be discovered
- Attack surface is larger than traditional syscalls

**Mitigations if enabled**:
1. Keep kernels updated with latest security patches
2. Use LSM (AppArmor/SELinux) as additional defense layer
3. Monitor io_uring usage with eBPF-based tools
4. Limit which containers/pods can use io_uring

### 6.2 Best Practices for Production

1. **Default to blocked**: Only enable io_uring where performance benefits are demonstrated
2. **Isolated workloads**: Enable only for specific, trusted workloads
3. **Latest kernels**: Many early io_uring vulnerabilities are patched in newer kernels (6.1+)
4. **Monitor**: Use eBPF-based security tools that can observe io_uring operations
5. **Network segmentation**: Limit blast radius if container is compromised

---

## 7. Performance Implications

### 7.1 Where io_uring Wins

Table 1 shows performance comparison of 1kB random reads at 100% CPU utilization using Direct I/O. We can see io_uring is a bit faster than linux-aio, but nothing revolutionary for basic use.

**Major benefits**:
- **Syscall overhead**: Each syscall costs ~2200ns with Spectre mitigations
- **Batching**: io_uring amortizes syscall overhead across many operations
- **File I/O**: True async file I/O (epoll doesn't work for regular files)
- **Zero-copy**: Registered buffers avoid kernel-userspace copies

### 7.2 Network I/O Comparison

The performance of io_uring is about 10% higher than epoll when handling 1000 connections, which conforms to modeling. From the quantitative analysis, the superiority of io_uring and epoll is determined by four variables.

In no single test have I seen io_uring beat epoll significantly for simple TCP echo scenarios with low connection counts. epoll is performing better by a measurable amount in some tests.

**Summary**:
| Scenario | io_uring Advantage |
|----------|-------------------|
| High connection count (>1000) | ~10% throughput improvement |
| Low connection count (<100) | Negligible or epoll faster |
| File I/O | Significant (true async vs thread pool) |
| Syscall-heavy workloads | Major (batching benefit) |

### 7.3 Fallback Performance Impact

| Operation Type | io_uring | epoll Fallback | Impact |
|---------------|----------|----------------|--------|
| Network (low connections) | Fast | Fast | Minimal |
| Network (high connections) | Faster | Good | ~10% slower |
| File read/write | True async | Thread pool | Can be significant |
| Mixed workload | Optimized | Adequate | Variable |

### 7.4 Hybrid Approach Consideration

**Question**: Use io_uring for file I/O and epoll for network?

By building tokio-uring on top of Tokio's runtime, existing Tokio ecosystem crates can work with the tokio-uring runtime. The completion queue is registered with the epoll handle.

**Verdict**: Technically possible but adds significant complexity:
- Two event loops to manage
- Complex buffer ownership rules
- Minimal benefit at low connection counts
- **Recommendation**: Not worth the complexity for most use cases

---

## 8. Recommended Implementation Strategy

### 8.1 Detection Algorithm

```c
typedef struct {
    bool io_uring_available;
    bool io_uring_preferred;
    const char *reason;
} io_backend_detection_t;

io_backend_detection_t detect_best_backend() {
    io_backend_detection_t result = {
        .io_uring_available = false,
        .io_uring_preferred = false,
        .reason = NULL
    };
    
    // 1. Check environment override
    const char *override = getenv("INDENT_IO_BACKEND");
    if (override) {
        if (strcmp(override, "epoll") == 0) {
            result.reason = "User requested epoll via INDENT_IO_BACKEND";
            return result;
        }
        // "iouring" or "auto" continues detection
    }
    
    // 2. Check kernel version (needs 5.10+ for stable features)
    struct utsname uname_buf;
    if (uname(&uname_buf) == 0) {
        int major, minor;
        sscanf(uname_buf.release, "%d.%d", &major, &minor);
        if (major < 5 || (major == 5 && minor < 10)) {
            result.reason = "Kernel < 5.10, io_uring features incomplete";
            return result;
        }
    }
    
    // 3. Check sysctl
    FILE *f = fopen("/proc/sys/kernel/io_uring_disabled", "r");
    if (f) {
        int disabled;
        if (fscanf(f, "%d", &disabled) == 1 && disabled == 2) {
            fclose(f);
            result.reason = "io_uring disabled via sysctl";
            return result;
        }
        fclose(f);
    }
    
    // 4. Probe syscall
    struct io_uring_params params = {0};
    int fd = syscall(__NR_io_uring_setup, 8, &params);
    
    if (fd < 0) {
        switch (errno) {
            case EPERM:
                result.reason = "io_uring blocked by seccomp/permissions";
                break;
            case ENOSYS:
                result.reason = "io_uring syscall not available";
                break;
            default:
                result.reason = "io_uring probe failed";
                break;
        }
        return result;
    }
    
    close(fd);
    result.io_uring_available = true;
    result.io_uring_preferred = true;
    result.reason = "io_uring available and working";
    return result;
}
```

### 8.2 Runtime Initialization

```c
static async_runtime_t *g_runtime;

int init_indent_runtime() {
    io_backend_detection_t detection = detect_best_backend();
    
    log_info("I/O backend detection: %s", detection.reason);
    
    if (detection.io_uring_preferred) {
        log_info("Using io_uring backend");
        g_runtime = create_io_uring_runtime();
    } else {
        log_info("Using epoll backend");
        g_runtime = create_epoll_runtime();
    }
    
    if (!g_runtime) {
        return -1;
    }
    
    // Expose backend choice for debugging
    setenv("INDENT_ACTIVE_BACKEND", 
           detection.io_uring_preferred ? "iouring" : "epoll", 1);
    
    return 0;
}
```

### 8.3 User Documentation

Include in Indent documentation:

```markdown
## Async I/O Backend

Indent automatically detects the best async I/O backend for your environment:

- **io_uring**: Used on Linux 5.10+ when available (best performance)
- **epoll**: Fallback on older kernels or when io_uring is restricted

### Container Deployments

Most container runtimes (Docker 25+, containerd, Kubernetes) block io_uring 
by default for security reasons. Indent automatically falls back to epoll.

### Environment Variables

- `INDENT_IO_BACKEND=auto` (default): Automatically detect best backend
- `INDENT_IO_BACKEND=iouring`: Force io_uring (fails if unavailable)
- `INDENT_IO_BACKEND=epoll`: Force epoll fallback

### Enabling io_uring in Containers

If you need io_uring performance in containers, you must provide a custom 
seccomp profile. See [Enabling io_uring in Containers](./io_uring_containers.md).

**Warning**: Enabling io_uring in containers increases the kernel attack 
surface. Only enable for trusted workloads where performance is critical.
```

---

## 9. Files to Provide to Users

### 9.1 Seccomp Profile Template

Create `seccomp/io_uring_enabled.json` in Indent distribution:

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "archMap": [
    {"architecture": "SCMP_ARCH_X86_64", "subArchitectures": ["SCMP_ARCH_X86", "SCMP_ARCH_X32"]},
    {"architecture": "SCMP_ARCH_AARCH64", "subArchitectures": ["SCMP_ARCH_ARM"]}
  ],
  "syscalls": [
    {
      "comment": "io_uring syscalls - enable async I/O",
      "names": ["io_uring_setup", "io_uring_enter", "io_uring_register"],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "comment": "Base syscalls from Docker default profile",
      "names": [
        "accept", "accept4", "access", "adjtimex", "alarm", "bind", "brk",
        "capget", "capset", "chdir", "chmod", "chown", "chown32", "clock_adjtime",
        "clock_getres", "clock_gettime", "clock_nanosleep", "clone", "close",
        "connect", "copy_file_range", "creat", "dup", "dup2", "dup3",
        "epoll_create", "epoll_create1", "epoll_ctl", "epoll_pwait", "epoll_wait",
        "eventfd", "eventfd2", "execve", "execveat", "exit", "exit_group",
        "faccessat", "fadvise64", "fallocate", "fanotify_mark", "fchdir",
        "fchmod", "fchmodat", "fchown", "fchown32", "fchownat", "fcntl",
        "fdatasync", "fgetxattr", "flistxattr", "flock", "fork", "fremovexattr",
        "fsetxattr", "fstat", "fstatfs", "fsync", "ftruncate", "futex",
        "futimesat", "getcpu", "getcwd", "getdents", "getdents64", "getegid",
        "geteuid", "getgid", "getgroups", "getitimer", "getpeername", "getpgid",
        "getpgrp", "getpid", "getppid", "getpriority", "getrandom", "getresgid",
        "getresuid", "getrlimit", "get_robust_list", "getrusage", "getsid",
        "getsockname", "getsockopt", "get_thread_area", "gettid", "gettimeofday",
        "getuid", "getxattr", "inotify_add_watch", "inotify_init", "inotify_init1",
        "inotify_rm_watch", "io_cancel", "ioctl", "io_destroy", "io_getevents",
        "ioprio_get", "ioprio_set", "io_setup", "io_submit", "kill",
        "lchown", "lgetxattr", "link", "linkat", "listen", "listxattr",
        "llistxattr", "lremovexattr", "lseek", "lsetxattr", "lstat", "madvise",
        "memfd_create", "mincore", "mkdir", "mkdirat", "mknod", "mknodat",
        "mlock", "mlock2", "mlockall", "mmap", "mprotect", "mq_getsetattr",
        "mq_notify", "mq_open", "mq_timedreceive", "mq_timedsend", "mq_unlink",
        "mremap", "msgctl", "msgget", "msgrcv", "msgsnd", "msync", "munlock",
        "munlockall", "munmap", "nanosleep", "newfstatat", "open", "openat",
        "pause", "pipe", "pipe2", "poll", "ppoll", "prctl", "pread64",
        "preadv", "preadv2", "prlimit64", "pselect6", "pwrite64", "pwritev",
        "pwritev2", "read", "readahead", "readlink", "readlinkat", "readv",
        "recv", "recvfrom", "recvmmsg", "recvmsg", "remap_file_pages",
        "removexattr", "rename", "renameat", "renameat2", "restart_syscall",
        "rmdir", "rt_sigaction", "rt_sigpending", "rt_sigprocmask",
        "rt_sigqueueinfo", "rt_sigreturn", "rt_sigsuspend", "rt_sigtimedwait",
        "rt_tgsigqueueinfo", "sched_getaffinity", "sched_getattr", "sched_getparam",
        "sched_get_priority_max", "sched_get_priority_min", "sched_getscheduler",
        "sched_rr_get_interval", "sched_setaffinity", "sched_setattr",
        "sched_setparam", "sched_setscheduler", "sched_yield", "seccomp",
        "select", "semctl", "semget", "semop", "semtimedop", "send",
        "sendfile", "sendmmsg", "sendmsg", "sendto", "setfsgid", "setfsuid",
        "setgid", "setgroups", "setitimer", "setpgid", "setpriority",
        "setregid", "setresgid", "setresuid", "setreuid", "setrlimit",
        "set_robust_list", "setsid", "setsockopt", "set_thread_area",
        "set_tid_address", "setuid", "setxattr", "shmat", "shmctl", "shmdt",
        "shmget", "shutdown", "sigaltstack", "signalfd", "signalfd4",
        "socket", "socketpair", "splice", "stat", "statfs", "statx",
        "symlink", "symlinkat", "sync", "sync_file_range", "syncfs",
        "sysinfo", "tee", "tgkill", "time", "timer_create", "timer_delete",
        "timerfd_create", "timerfd_gettime", "timerfd_settime", "timer_getoverrun",
        "timer_gettime", "timer_settime", "times", "tkill", "truncate",
        "umask", "uname", "unlink", "unlinkat", "utime", "utimensat",
        "utimes", "vfork", "vmsplice", "wait4", "waitid", "waitpid",
        "write", "writev"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

### 9.2 Kubernetes Manifest Example

Create `examples/kubernetes/io_uring_enabled_pod.yaml`:

```yaml
# Example: Pod with io_uring enabled via custom seccomp profile
# 
# Prerequisites:
# 1. Copy io_uring_enabled.json to all nodes at:
#    /var/lib/kubelet/seccomp/profiles/io_uring_enabled.json
# 2. Ensure you're using a cluster that supports Localhost seccomp profiles
#    (GKE Standard, EKS, AKS - NOT GKE Autopilot)
#
# Security Warning: Only enable io_uring for trusted workloads where 
# performance is critical. This increases the kernel attack surface.
---
apiVersion: v1
kind: Pod
metadata:
  name: indent-app-with-io-uring
  labels:
    app: indent-app
  annotations:
    io-uring-enabled: "true"
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/io_uring_enabled.json
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
  containers:
  - name: indent-app
    image: your-registry/indent-app:latest
    env:
    - name: INDENT_IO_BACKEND
      value: "auto"  # Will auto-detect io_uring as available
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    resources:
      limits:
        memory: "512Mi"
        cpu: "500m"
      requests:
        memory: "256Mi"
        cpu: "250m"
```

### 9.3 Docker Compose Example

Create `examples/docker/docker-compose.io_uring.yml`:

```yaml
# Docker Compose with io_uring enabled
#
# Usage: docker-compose -f docker-compose.io_uring.yml up
#
# Note: Requires Docker on Linux. Does NOT work with Docker Desktop on 
# macOS/Windows due to VM-level io_uring restrictions.
version: '3.8'

services:
  indent-app:
    image: your-registry/indent-app:latest
    security_opt:
      - seccomp=./io_uring_enabled.json
    environment:
      - INDENT_IO_BACKEND=auto
```

---

## 10. Summary and Recommendations

### For Indent Runtime Implementation

1. **Default behavior**: Auto-detect with graceful fallback to epoll
2. **Detection**: Probe `io_uring_setup()` syscall at startup
3. **Environment override**: Support `INDENT_IO_BACKEND` for user control  
4. **Logging**: Always log which backend was selected and why
5. **Documentation**: Clearly document container restrictions and how to enable

### For Indent Users

1. **Accept the fallback**: epoll is adequate for most workloads
2. **Enable selectively**: Only enable io_uring where benchmarks show benefit
3. **Stay updated**: Use latest kernels if enabling io_uring
4. **Monitor**: Use security tools that can observe io_uring operations
5. **Test**: Verify application works correctly with both backends

### Platform Decision Matrix

| Environment | Recommendation |
|-------------|----------------|
| Bare metal Linux 5.10+ | Enable io_uring |
| Docker on Linux | Enable with custom seccomp if needed |
| Docker Desktop | Use epoll fallback |
| GKE Autopilot | Use epoll fallback |
| GKE Standard | Enable with custom seccomp if needed |
| AWS Fargate | Use epoll fallback |
| General Kubernetes | Use epoll fallback unless specifically configured |
| Development/testing | Auto-detect |
