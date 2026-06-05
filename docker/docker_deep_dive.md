# Docker Deep Dive

Docker is a platform for developing, shipping, and running applications in **containers** — lightweight, isolated environments that package code with all its dependencies.

## Core Concepts

**Containers vs VMs**

Virtual machines virtualize hardware and run a full OS, while containers virtualize the OS kernel and share it with the host. This makes containers far more lightweight — they start in milliseconds, use less memory, and have minimal overhead.

```
VM:          Container:
┌─────────┐  ┌─────────┐
│   App   │  │   App   │
├─────────┤  ├─────────┤
│ Bins/Lib│  │ Bins/Lib│
├─────────┤  └────┬────┘
│ Guest OS│       │
├─────────┤  ┌────┴────┐
│Hypervisor│ │  Docker │
├─────────┤  ├─────────┤
│ Host OS │  │ Host OS │
└─────────┘  └─────────┘
```

**Images**

Immutable templates built from a `Dockerfile`. Images are layered — each instruction creates a new layer, and layers are cached and shared between images. This is why you want to order your Dockerfile from least-frequently-changed to most-frequently-changed instructions.

**Union Filesystem**

Docker uses a copy-on-write filesystem (like OverlayFS). Container layers are read-only; when a container modifies a file, it's copied to a thin writable layer on top. This enables instant container creation and efficient storage.

## Namespaces and Cgroups

These are the Linux kernel features that make containers possible:

**Namespaces** provide isolation:
- `pid` — process isolation (container sees only its processes)
- `net` — network stack isolation (own interfaces, routing tables)
- `mnt` — filesystem mount points
- `uts` — hostname and domain name
- `ipc` — inter-process communication
- `user` — user/group ID mapping

**Cgroups** (control groups) limit resources:
- CPU shares and pinning
- Memory limits (hard and soft)
- I/O bandwidth
- Device access

```bash
# See a container's cgroup limits
docker inspect --format '{{.HostConfig.Memory}}' <container>
```

## Networking Deep Dive

Docker provides several network drivers:

**bridge** (default) — containers on the same bridge can communicate via IP. Docker creates a virtual bridge (`docker0`) and assigns containers IPs from a private subnet.

**host** — container shares the host's network namespace directly. No network isolation, but no NAT overhead.

**none** — no networking at all.

**overlay** — multi-host networking for Swarm/Kubernetes. Uses VXLAN to encapsulate packets.

```bash
# Create a custom bridge network
docker network create --driver bridge --subnet 172.20.0.0/16 mynet

# Inspect network details
docker network inspect bridge
```

**DNS resolution**: Containers on user-defined networks can resolve each other by container name. The embedded DNS server runs at `127.0.0.11`.

## Storage Deep Dive

**Volumes** — managed by Docker, stored in `/var/lib/docker/volumes/`. Best for persistent data.

**Bind mounts** — map host paths directly into containers. Good for development.

**tmpfs** — in-memory only, never written to disk.

```bash
# Named volume
docker run -v mydata:/app/data nginx

# Bind mount
docker run -v /host/path:/container/path nginx

# tmpfs
docker run --tmpfs /app/cache nginx
```

**Storage drivers** determine how layers are stored: `overlay2` (modern default), `devicemapper`, `btrfs`, `zfs`. Overlay2 is fastest for most workloads.

## Image Building Optimization

```dockerfile
# Bad: cache invalidated on every code change
FROM node:20
COPY . /app
RUN npm install

# Good: dependencies cached separately
FROM node:20
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
```

**Multi-stage builds** reduce final image size dramatically:

```dockerfile
# Build stage
FROM gcc:13 AS builder
WORKDIR /src
COPY . .
RUN g++ -O3 -static main.cpp -o app

# Runtime stage
FROM alpine:3.19
COPY --from=builder /src/app /usr/local/bin/
CMD ["app"]
```

This gives you a final image of just a few MB instead of hundreds.

## Security Considerations

**Root in container = root on host** (by default). Mitigations:
- Run as non-root user: `USER 1000` in Dockerfile
- Use user namespaces to remap UIDs
- Drop capabilities: `--cap-drop=ALL --cap-add=NET_BIND_SERVICE`
- Read-only root filesystem: `--read-only`
- Seccomp profiles to limit syscalls
- AppArmor/SELinux policies

```bash
# Security-hardened run
docker run --read-only \
           --cap-drop=ALL \
           --security-opt=no-new-privileges \
           --user 1000:1000 \
           myapp
```

## Performance Tuning

**Resource limits** prevent runaway containers:

```bash
docker run --memory=512m --memory-swap=512m \  # hard memory limit
           --cpus=1.5 \                         # 1.5 CPU cores
           --pids-limit=100 \                   # prevent fork bombs
           myapp
```

**Build caching with BuildKit**:

```bash
DOCKER_BUILDKIT=1 docker build --cache-from myimage:latest -t myimage:new .
```

---

Any specific area you want me to go deeper on? Given your compiler work, you might be interested in optimizing Docker builds for C++ projects with complex dependencies, or container security hardening.

# Calculating CPU Usage in Docker

Docker CPU metrics come from **cgroups**, and calculating actual percentage usage requires understanding what the kernel reports.

## Raw Metrics Location

For a running container, cgroup stats are exposed at:

```bash
# cgroup v2 (modern systems)
/sys/fs/cgroup/docker/<container_id>/cpu.stat

# cgroup v1 (older systems)
/sys/fs/cgroup/cpu/docker/<container_id>/cpuacct.usage
```

Or via Docker's API at `/containers/<id>/stats`.

## The Calculation

CPU usage percentage is calculated from **two snapshots** taken at different times:

```
CPU % = (Δ container_cpu_usage / Δ system_cpu_usage) × num_cpus × 100
```

Where:
- `Δ container_cpu_usage` = change in container's CPU time (nanoseconds)
- `Δ system_cpu_usage` = change in total system CPU time (nanoseconds)
- `num_cpus` = number of CPUs available to the container

## Getting Stats via Docker API

```bash
# Stream stats as JSON
docker stats --no-stream --format "{{json .}}" <container>

# Raw stats from API
curl --unix-socket /var/run/docker.sock http://localhost/containers/<id>/stats?stream=false
```

The API returns:

```json
{
  "cpu_stats": {
    "cpu_usage": {
      "total_usage": 123456789000,      // nanoseconds
      "percpu_usage": [456789000, ...], // per-core breakdown
      "usage_in_kernelmode": 100000000,
      "usage_in_usermode": 23456789000
    },
    "system_cpu_usage": 9876543210000000,
    "online_cpus": 8
  },
  "precpu_stats": { /* previous sample, same structure */ }
}
```

## C++ Implementation

```cpp
#include <cstdint>
#include <optional>

struct CpuSample {
    uint64_t container_usage;  // nanoseconds
    uint64_t system_usage;     // nanoseconds (or jiffies × 10^7 on cgroup v1)
    uint32_t online_cpus;
};

// Returns CPU percentage (0.0 - 100.0 × num_cpus)
std::optional<double> calculate_cpu_percent(
    const CpuSample& current,
    const CpuSample& previous
) {
    auto cpu_delta = current.container_usage - previous.container_usage;
    auto system_delta = current.system_usage - previous.system_usage;
    
    if (system_delta == 0 || current.online_cpus == 0) {
        return std::nullopt;
    }
    
    double percent = (static_cast<double>(cpu_delta) / 
                      static_cast<double>(system_delta)) * 
                     current.online_cpus * 100.0;
    
    return percent;
}
```

## Reading from cgroup v2 Directly

```cpp
#include <fstream>
#include <string>
#include <unordered_map>
#include <filesystem>

namespace fs = std::filesystem;

// Parse cpu.stat file (cgroup v2)
// Format: "usage_usec 123456\nuser_usec 100000\nsystem_usec 23456\n..."
std::unordered_map<std::string, uint64_t> parse_cpu_stat(const fs::path& cgroup_path) {
    std::unordered_map<std::string, uint64_t> stats;
    std::ifstream file(cgroup_path / "cpu.stat");
    
    std::string key;
    uint64_t value;
    while (file >> key >> value) {
        stats[key] = value;
    }
    return stats;
}

// Get container CPU usage in nanoseconds
uint64_t get_container_cpu_ns(const fs::path& cgroup_path) {
    auto stats = parse_cpu_stat(cgroup_path);
    // usage_usec is in microseconds, convert to nanoseconds
    return stats["usage_usec"] * 1000;
}

// Get system-wide CPU time from /proc/stat
uint64_t get_system_cpu_ns() {
    std::ifstream file("/proc/stat");
    std::string cpu;
    uint64_t user, nice, system, idle, iowait, irq, softirq, steal;
    
    file >> cpu >> user >> nice >> system >> idle >> iowait >> irq >> softirq >> steal;
    
    // Values are in jiffies (typically 10ms each)
    // Convert to nanoseconds: jiffies × (1e9 / HZ), where HZ is usually 100
    uint64_t total_jiffies = user + nice + system + idle + iowait + irq + softirq + steal;
    return total_jiffies * 10'000'000;  // assuming HZ=100
}
```

## cgroup v1 vs v2 Differences

| Aspect | cgroup v1 | cgroup v2 |
|--------|-----------|-----------|
| CPU usage file | `cpuacct.usage` (ns) | `cpu.stat` → `usage_usec` (μs) |
| Per-CPU stats | `cpuacct.usage_percpu` | Not directly exposed |
| Throttling stats | `cpu.stat` | `cpu.stat` → `nr_throttled`, `throttled_usec` |
| Path | `/sys/fs/cgroup/cpu/docker/<id>/` | `/sys/fs/cgroup/docker/<id>/` |

## Checking for CPU Throttling

Throttling occurs when a container exceeds its CPU quota:

```bash
# cgroup v2
cat /sys/fs/cgroup/docker/<id>/cpu.stat
# nr_periods 1000
# nr_throttled 50      # 5% of periods were throttled
# throttled_usec 123456
```

```cpp
struct ThrottlingStats {
    uint64_t periods;
    uint64_t throttled_periods;
    uint64_t throttled_time_us;
    
    double throttle_percent() const {
        return periods > 0 ? (throttled_periods * 100.0 / periods) : 0.0;
    }
};
```

## Quick CLI Methods

```bash
# Real-time stats
docker stats

# One-shot with custom format
docker stats --no-stream --format \
  "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"

# Using ctop (like htop for containers)
ctop

# Raw calculation with two samples
ID=$(docker ps -q | head -1)
cat /sys/fs/cgroup/docker/$ID/cpu.stat
sleep 1
cat /sys/fs/cgroup/docker/$ID/cpu.stat
# Then compute delta manually
```

---

Want me to expand on throttling detection, implementing a full stats collector, or optimizing CPU limits for your build containers?