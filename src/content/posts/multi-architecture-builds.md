---
title: "Building Multi-Architecture Images at Scale"
slug: "building-multi-archicture-images-at-scale"
description: "Building Multi-Architecture Docker Images at Scale"
publicationDate: 2025-07-23
category: Platform, DevOps
public: true
author: "Rene Kabanda"
---

In 2025, building Docker images that run seamlessly across `amd64`, `arm64`, `ppc64le`, and even exotic platforms like IBM’s `s390x` is no longer a luxury—it’s a necessity. As enterprises adopt ARM (thanks to AWS Graviton and Apple Silicon) and edge devices proliferate, multi-arch Docker images underpin modern software delivery.

But how you build those images determines whether your CI pipelines scale—or crumble. Today, I’ll break down how big players like **Debian**, **Docker**, **GitHub**, and **GitLab** tackle multi-arch builds, why naive cross-building can hurt you, and what *best practices* you should be borrowing from them.

---

## The Challenge of Multi-Architecture Docker Images

As organisations scale and diversify their infrastructure, they increasingly need to support multiple CPU architectures—most commonly x86_64 (amd64) and ARM64 (aarch64). However, Docker images are, by default, architecture-specific: an image built for amd64 will not run on arm64, and vice versa.

This creates several challenges:

- **Fragmented image management**: You must build, tag, and distribute separate images for each architecture.

- **User confusion**: Consumers of your images must know which tag to use for their platform, or risk pulling an incompatible image.

- **Operational complexity**: CI/CD pipelines become more complicated, and mistakes can lead to “exec format error” or runtime failures in production.

- **Scalability bottlenecks**: As more architectures are added (e.g., for cost or performance reasons), the complexity and risk only increase.

The core problem:

How do you provide a seamless, reliable Docker image experience for all users, regardless of their underlying hardware, while keeping your build and release process maintainable and scalable?


## The Two Paths: Cross-Build vs Native Build

When building for multiple architectures, there are two broad approaches:

1. **Cross-build on x86 hosts using QEMU emulation**
2. **Build natively on hardware for each target architecture**

At first glance, QEMU + Docker Buildx sounds like magic: build `arm64` on your MacBook and push it. But for large projects and production pipelines, it’s often the wrong choice.



## The Numbers Don’t Lie: Native vs QEMU

| Task Type                     | Native Build Time | QEMU Emulated Build Time | Slowdown  |
| ----------------------------- | ----------------- | ------------------------ | --------- |
| Simple Go app (CGO disabled)  | 30 sec            | 2 min                    | **4×**    |
| Node.js image (npm install)   | 3 min             | 12 min                   | **4×**    |
| Rust crate build              | 5 min             | 40 min                   | **8×**    |
| Debian base image build       | 2 min             | 15–30 min                | **8–15×** |
| GitLab Omnibus (Ruby, Go, Pg) | 45 min            | 12 hours+                | **16×**   |

**On average, QEMU builds are 5–20× slower.**

For complex builds with heavy I/O or compilation, the slowdown can exceed **1,500%**.

### Why So Slow?

- QEMU translates every CPU instruction from ARM to x86.
- Syscalls are intercepted and emulated.
- Poor SMP support means multi-core workloads don’t scale well.
- Heavy I/O (e.g., `apt install`, `npm install`) exacerbates latency.

### Compatibility Issues

Besides speed, QEMU introduces subtle bugs:

- `uname -m` returns the host architecture.
- Certain binaries (e.g., `glibc`) behave differently under emulation.
- Toolchains may produce incorrect output due to syscall quirks.

This is why **Docker Official Images explicitly warn against relying on QEMU for production builds** [[1\]](https://github.com/docker-library/official-images).


## How the Big Players Build Multi-Arch Images

### Docker Official Images

Docker maintains a fleet of **native build servers**:

- `amd64` on x86\_64 servers
- `arm64` on ARM servers (AWS Graviton, Raspberry Pi farms)
- `ppc64le` and `s390x` on IBM Power and Z hardware

These native builds are then combined into a manifest list:

```bash
docker manifest create library/ubuntu:latest \
  ubuntu:amd64 ubuntu:arm64 ubuntu:s390x

docker manifest push library/ubuntu:latest
```

When you pull `ubuntu:latest`, Docker automatically resolves to the right architecture.

### Alpine Linux

Alpine’s CI builds each architecture **natively** and publishes separate images before combining them into multi-arch manifests [[2\]](https://github.com/alpinelinux/docker-alpine).

### GitLab

GitLab Omnibus images build natively using GitLab CI runners tagged for specific architectures (`arm64`, `amd64`). Native ARM runners on Graviton2 instances eliminate QEMU bottlenecks [[3\]](https://gitlab.com/gitlab-org/omnibus-gitlab/-/blob/master/.gitlab-ci.yml).

### GitHub

GitHub Container Registry (GHCR) workflows leverage **GitHub Actions runners on ARM and x86 hardware**. Their `build-push-action` supports native builds with matrix strategies:

```yaml
strategy:
  matrix:
    arch: [amd64, arm64]
jobs:
  build:
    runs-on: [self-hosted, linux, ${{ matrix.arch }}]
```

---
## Our Journey

We recently set out to update our base images across multiple versions, and quickly found ourselves wrestling with a host of unexpected challenges.

- **Random build failures:** QEMU emulation caused weird compiler errors and segfaults, especially with PHP extensions.

- **Painfully slow builds:** ARM64 images took up to 5× longer to build than amd64.

- **Resource hogs:** Emulated builds ate up CPU and memory, leading to OOM kills.

- **Debugging hell:** Errors were inconsistent and hard to reproduce.

- **Tag confusion:** Single-arch tags got overwritten, causing “exec format error” in downstream builds.

- **Pipeline mess:** Our scripts got bloated with workarounds for emulation quirks.


## The Solution: Native Builds + Manifest Creation

1. **Separate native builds:**
    - One CI job runs on an amd64 agent, another on an arm64 agent.
    - Each builds and pushes its arch-specific image (e.g., `php:8.2-amd64`, `php:8.2-arm64`).
2. **Manifest creation:**
    - A third job (can run on any agent) creates a multi-arch manifest (`php:8.2`) that references both images.
    - Consumers use `FROM ...:8.2` and Docker pulls the right image automatically.

### **Benefits**
- **Speed:** Native builds are 2–4× faster.
- **Reliability:** No more random QEMU segfaults or OOMs.
- **Scalability:** Easily add more architectures as needed.
- **Industry alignment:** Matches the approach used by Docker, Debian, and GitHub.

---



## ✅ Best Practices for Multi-Arch Builds

### 1️⃣ Use Native Builds Where Possible

- Leverage ARM cloud instances (AWS Graviton, GCP Tau T2A)
- Use hardware builders like Raspberry Pi clusters for edge architectures

### 2️⃣ Combine with Multi-Arch Manifests

Use `docker manifest` or `buildx` to combine native-built images:

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --push -t myorg/service:latest .
```

### 3️⃣ Automate with CI/CD

Design pipelines to:

- Build per-arch images in parallel
- Push images to registries
- Create and push manifest lists

---

## 🚩 Why Avoid QEMU for Release Pipelines

| Feature                    | Native Build   | QEMU Cross-Build   |
| -------------------------- | -------------- | ------------------ |
| Speed                      | 🚀 Fast        | 🐢 5–20× slower    |
| Compatibility              | ✅ Native       | ⚠ Subtle bugs      |
| Multi-threaded Compilation | ✅ Full support | ⚠ Poor performance |
| Infra Cost                 | 💸 Higher      | ✅ Lower            |

QEMU is perfect for **dev/test** workflows, but **never for production** if your project involves heavy compilation or I/O.

---

## The Scalability Factor

At scale, the difference between QEMU and native builds isn’t academic—it’s existential.

For a team pushing **100 images per day**:

- **Native build:** 100 builds × 10 min = \~17 hours total (parallelized)
- **QEMU build:** 100 builds × 2 hours = \~200 hours total 🔥

This 10× delta can clog your CI/CD, delay releases, and waste engineering hours debugging emulation quirks.

---

## 📌 TL;DR

✅ **For production-grade pipelines:** Build natively per-architecture and merge with multi-arch manifests.\
✅ **For local dev/testing:** QEMU + Buildx is fine.\
✅ **For large orgs:** Invest in native runners (or cloud ARM instances).

Big players like Docker, Debian, Alpine, and GitLab do this for a reason—**it’s the only way to scale multi-arch builds without drowning in pipeline latency and compatibility bugs**.



> "If it’s good enough for Docker, Debian, and GitHub, it’s good enough for us."



## 📚 References

1. [Docker Official Images Build Infrastructure](https://github.com/docker-library/official-images)
2. [Alpine Linux Docker Image CI](https://github.com/alpinelinux/docker-alpine)
3. [GitLab Omnibus Multi-Arch CI](https://gitlab.com/gitlab-org/omnibus-gitlab/-/blob/master/.gitlab-ci.yml)
4. [Docker Buildx Documentation](https://docs.docker.com/buildx/working-with-buildx/)
5. [QEMU User Documentation](https://wiki.qemu.org/Documentation)

---
