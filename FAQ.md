_This document was generated with Opus 4.6 with the following prompt:_
> Please create a file named FAQ.md with a list of questions and their answers. I will provide you the list of questions and, analyzing the source code of the project, you will generate the answers. For each question write both question and answer in file FAQ.md. The questions are all about wslc, a new command line and SDK that has been released recently. The wslc source code is in the current repository (github.com/microsoft/WSL). Here is the list of answers:
>
> 1. Does wslc create new, isolated, VM every time `wslc run` is executed? Or are all the containers run in a shared VM?
> 2. Are wslc VMs more secure compared to wsl distributions? All wsl distriburions on the same host share the same VM and it's easy to access to the host filesystem and to the filesystem of other distributions. How do wslc VMs comapre?
> 3. wlsc has a system session subcommand. Are sessions the name that wslc uses for VMs?
> 4. Is is possible to use a custom Linux kernel for wslc? If yes can you point me to the instructions?
> 5. Can you provide a powershell command to manually create a VM similare to the one created by wslc?
> 6. When has wslc been added to the git repository? Can you provide a list of significative pull requests?
> 7. Can a client running on the host send a request to the containerd service running in the VM?
> 8. Containerd is used in wslc VM to manage the Linux containers. Considered that podman is a daemonless container runtime, wouldn't it be a better fit for the wslc use case? Why do you think Docker has been preferred?
> 9. Does the wslc Windows client share some source code with the Docker client for Windows? Are there any docker client subcommands missing in wslc? Are there some wslc subcommands that don't have a correspondant in docker?
> 10. How does wslc compares to Microsoft Execution Containers (MXC) SDK? Is there any evidence that wslc will become the default backend for MXC on Windows?
> 11. How does wslc compares to other microVMs? I am thinking about the features and the ecosystem described here https://emirb.github.io/blog/microvm-2026/. Can it be considered a VMM or does it belong to another another abstraction layer?
> 12. How does wslc compares to apple containers? To libkrun?
> 13. Where is the source code that specifies the packages that are installed in the wslc VM? For example containerd?
> 14. Is there any design document for wslc? Anything that explains the technical decisions that has been taken?
>
> This was the last question. When providing the answers please be clear but concise, and don't sound as an LLM. Double check your
> answers and provide evidences like links to documents or source code for every question.

# wslc FAQ

A list of frequently asked questions about **wslc** (WSL Containers), the container runtime built on the Windows Subsystem for Linux. Answers are based on the [microsoft/WSL](https://github.com/microsoft/WSL) source code at commit `ff898462` (June 2026).

---

## 1. Does wslc create a new, isolated VM every time `wslc run` is executed? Or are all containers run in a shared VM?

All containers within the same **session** share a single VM.

The session process (`wslcsession.exe`) owns a `IWSLCVirtualMachineFactory` that creates the VM on demand the first time a container is needed. Subsequent `wslc run` invocations that target the same session reuse the existing VM — the factory only creates a new one if the previous VM was idle-terminated or crashed. Inside that VM, containerd and dockerd manage individual containers using standard Linux namespaces/cgroups, not separate VMs.

The idle-terminate + lazy-restart behavior was added in commit [`fff03251`](https://github.com/microsoft/WSL/commit/fff03251) ("wslc: idle-terminate per-user container VMs and lazily restart"). The factory pattern is defined in [`src/windows/service/exe/HcsVirtualMachine.h:120-156`](https://github.com/microsoft/WSL/blob/master/src/windows/service/exe/HcsVirtualMachine.h#L120) (`WSLCVirtualMachineFactory`).

Different sessions get different VMs, so you can get per-project isolation by creating separate sessions via `wslc system session`.

---

## 2. Are wslc VMs more secure compared to WSL distributions?

Yes, substantially. Traditional WSL2 distributions share a single VM and kernel, and each distribution can access the host filesystem (via `/mnt/c`) and other distributions' filesystems. wslc sessions are isolated in several ways:

- **Per-session VMs**: Each session gets its own Hyper-V-backed VM created via the [Host Compute Service (HCS)](https://learn.microsoft.com/virtualization/api/hcs/overview). There is no shared kernel or filesystem between sessions.
- **Per-user process isolation**: Each session runs in a separate COM server process, with token-based access control ensuring only the owning user (or elevated callers) can interact with it ([`src/windows/service/exe/WSLCSessionManager.cpp:175-181`](https://github.com/microsoft/WSL/blob/master/src/windows/service/exe/WSLCSessionManager.cpp#L175)).
- **HvSocket security descriptors**: Each VM's Hyper-V sockets are ACL'd to the user's SID ([`src/windows/service/exe/HcsVirtualMachine.cpp:290-299`](https://github.com/microsoft/WSL/blob/master/src/windows/service/exe/HcsVirtualMachine.cpp#L290)).
- **Separate storage**: Each session has its own `storage.vhdx` for container data — there is no shared filesystem between sessions ([`src/windows/wslcsession/WSLCSession.cpp:41`](https://github.com/microsoft/WSL/blob/master/src/windows/wslcsession/WSLCSession.cpp#L41)).
- **No automatic host filesystem mount**: Unlike WSL distributions, wslc VMs don't mount `/mnt/c` by default. Windows folders are shared explicitly via VirtioFS or Plan9 when a bind mount is requested.

The root VHD (`system.vhd`) is attached read-only ([`src/windows/service/exe/HcsVirtualMachine.cpp:267`](https://github.com/microsoft/WSL/blob/master/src/windows/service/exe/HcsVirtualMachine.cpp#L267)), so the base OS image cannot be tampered with by containers.

---

## 3. wslc has a `system session` subcommand. Are sessions the name that wslc uses for VMs?

No. A session is not the same as a VM — it is a higher-level abstraction.

A **session** is a per-user COM server process (`wslcsession.exe`) that manages containers, images, volumes, and networks. A session *owns* a VM, but the two have different lifecycles: the VM can be idle-terminated and recreated without destroying the session, and the session persists configuration and storage independently of the VM.

The `system session` subcommands ([`src/windows/wslc/commands/SessionCommand.h:19-109`](https://github.com/microsoft/WSL/blob/master/src/windows/wslc/commands/SessionCommand.h#L19)) are:

| Command | Description |
|---|---|
| `system session list` | List existing sessions |
| `system session run` | Create and/or run a command in a session |
| `system session enter` | Enter an existing session's shell |
| `system session shell` | Open a shell in a session |
| `system session terminate` | Terminate a session (tears down VM and all containers) |

Session lifecycle is documented in [`src/windows/service/exe/WSLCSessionManager.h:13-31`](https://github.com/microsoft/WSL/blob/master/src/windows/service/exe/WSLCSessionManager.h#L13). Each session has a display name, a storage path, CPU/memory/networking settings, and can be persistent (held alive by the service) or non-persistent (lifetime tied to client COM references).

---

## 4. Is it possible to use a custom Linux kernel for wslc?

Yes, through debugging options in the session settings. The `WSLCSessionSettings` struct ([`src/windows/service/inc/WSLCCompat.idl:240-255`](https://github.com/microsoft/WSL/blob/master/src/windows/service/inc/WSLCCompat.idl#L240)) includes:

- `RootVhdOverride` — overrides the default `system.vhd` with a custom root VHD
- `RootVhdTypeOverride` — overrides the root VHD type

These are documented as "used for debugging purposes only" in the IDL.

For the kernel binary itself, the path is determined at build time via the `WSL_KERNEL_PATH` compile definition ([`src/windows/service/exe/HcsVirtualMachine.cpp:214-218`](https://github.com/microsoft/WSL/blob/master/src/windows/service/exe/HcsVirtualMachine.cpp#L214)). If not overridden at build time, it defaults to `C:\Program Files\WSL\tools\kernel`. Similarly, `WSL_SYSTEM_DISTRO_PATH` overrides the root VHD path.

On x86_64, the kernel is booted via Linux kernel direct boot ([`HcsVirtualMachine.cpp:220-226`](https://github.com/microsoft/WSL/blob/master/src/windows/service/exe/HcsVirtualMachine.cpp#L220)). On ARM64, UEFI boot is used instead ([`HcsVirtualMachine.cpp:227-237`](https://github.com/microsoft/WSL/blob/master/src/windows/service/exe/HcsVirtualMachine.cpp#L227)). The boot process is described in [`doc/docs/technical-documentation/boot-process.md`](https://github.com/microsoft/WSL/blob/master/doc/docs/technical-documentation/boot-process.md).

There is no `wslc` CLI flag to specify a custom kernel at runtime — this is a build-time or SDK-level option.

---

## 5. Can you provide a PowerShell command to manually create a VM similar to the one created by wslc?

Not directly. wslc creates VMs through the [Host Compute Service (HCS)](https://learn.microsoft.com/virtualization/api/hcs/overview) C API (`HcsCreateComputeSystem`), not through Hyper-V PowerShell cmdlets. HCS VMs are lightweight compute systems, not classic Hyper-V VMs — they don't show up in `Get-VM`.

The VM is created by building a JSON configuration and passing it to `HcsCreateComputeSystem()` ([`src/windows/service/exe/HcsVirtualMachine.cpp:314-319`](https://github.com/microsoft/WSL/blob/master/src/windows/service/exe/HcsVirtualMachine.cpp#L314)). The JSON schema is defined in [`src/windows/common/hcs_schema.h`](https://github.com/microsoft/WSL/blob/master/src/windows/common/hcs_schema.h). Key parameters:

- Schema version 2.7 on Windows 11+, 2.3 on Windows 10
- Memory: configurable (2MB granularity), with overcommit, deferred commit, and cold discard hints enabled
- CPU: configurable count, with perf PMU/LBR support on x86_64
- Root VHD: `system.vhd` attached read-only via SCSI
- Kernel: direct boot on x86_64, UEFI on ARM64
- HvSocket: ACL'd to the user's SID
- Networking: NAT, Consomme, or None

To replicate this from PowerShell you would need to P/Invoke the HCS APIs from `computecore.dll`. There is a .NET wrapper at [microsoft/hcsshim](https://github.com/microsoft/hcsshim), and the full JSON schema is in `hcs_schema.h`, but there's no turnkey PowerShell cmdlet for this.

---

## 6. When was wslc added to the git repository? Can you provide a list of significant pull requests?

The feature went through three phases:

**Phase 1 — Prototype ("WSL for Apps")**: July 2025
- [`b5769b4f`](https://github.com/microsoft/WSL/commit/b5769b4f) (2025-07-24) — "Merge User/oneblue/prototype lsw to a feature/wsl-for-apps" — first merge of the prototype
- [`c07fa29f`](https://github.com/microsoft/WSL/commit/c07fa29f) (2025-08-04) — "Implement WSL for apps TCP port forwarding"
- [`2a41fe20`](https://github.com/microsoft/WSL/commit/2a41fe20) (2025-12-02) — "Initial implementation for CreateContainer()"

**Phase 2 — wslc CLI creation and WSLA→WSLC rename**: Jan–Mar 2026
- [`e2f730e0`](https://github.com/microsoft/WSL/commit/e2f730e0) (2026-01-16) — "Initial creation of wslc.exe"
- [`318ade60`](https://github.com/microsoft/WSL/commit/318ade60) (2026-01-07) — "Switch from nerdctl to dockerd"
- [`8973aeb8`](https://github.com/microsoft/WSL/commit/8973aeb8) (2026-03-27) — "cleanup: rename wsla -> wslc"

**Phase 3 — Merge to master and active development**: Apr 2026–present
- [`358ab87d`](https://github.com/microsoft/WSL/commit/358ab87d) (2026-04-30) — **"Add WSLC (WSL Containers) feature (#40366)"** — the main merge to master, co-authored by ~10 developers
- [`58139c50`](https://github.com/microsoft/WSL/commit/58139c50) (2026-05-01) — "Implement GPU support"
- [`a8ff3108`](https://github.com/microsoft/WSL/commit/a8ff3108) (2026-05-19) — "Implement WinRT / C# projection for the WSLC SDK API surface"
- [`8cd5ef1f`](https://github.com/microsoft/WSL/commit/8cd5ef1f) (2026-05-18) — "Initial support for wslc network commands"
- [`e76cb8b0`](https://github.com/microsoft/WSL/commit/e76cb8b0) (2026-06-11) — "wslc: introduce IWSLCVirtualMachineFactory for session VM creation"
- [`4d3fa5c5`](https://github.com/microsoft/WSL/commit/4d3fa5c5) (2026-05-20) — "Add openvmm as vmm option for wslc vms"
- [`fff03251`](https://github.com/microsoft/WSL/commit/fff03251) (2026-06-23) — "wslc: idle-terminate per-user container VMs and lazily restart"

---

## 7. Can a client running on the host send a request to the containerd service running in the VM?

Not directly. The host communicates with the Docker daemon API (not containerd's gRPC API) over HTTP, relayed through Hyper-V sockets.

The connection flow works like this:

1. `wslcsession.exe` on the host creates a new HvSocket channel by sending a `WSLC_FORK` message to the init process inside the VM.
2. The forked channel connects to the Docker unix socket (`/var/run/docker.sock`) via a `WSLC_UNIX_CONNECT` message ([`src/shared/inc/lxinitshared.h:1836-1848`](https://github.com/microsoft/WSL/blob/master/src/shared/inc/lxinitshared.h#L1836)).
3. The `DockerHTTPClient` class ([`src/windows/wslcsession/DockerHTTPClient.h:100-130`](https://github.com/microsoft/WSL/blob/master/src/windows/wslcsession/DockerHTTPClient.h#L100)) sends HTTP requests over this socket using Boost.Beast/ASIO with the Hyper-V socket protocol.

The Docker API used is v1.44, matching dockerd v25.0.3 ([`src/windows/inc/docker_schema.h`](https://github.com/microsoft/WSL/blob/master/src/windows/inc/docker_schema.h)). There is no exposed endpoint for external clients to connect to containerd or dockerd inside the VM — all access goes through the wslc service layer.

---

## 8. Containerd is used in wslc VM to manage containers. Podman is a daemonless runtime — wouldn't it be a better fit? Why was Docker preferred?

wslc initially used nerdctl (the containerd CLI) directly, as seen in early commits like [`99d4c488`](https://github.com/microsoft/WSL/commit/99d4c488) (2025-12-10, "Use nerdctl events to track container state changes") and [`c7e80fbd`](https://github.com/microsoft/WSL/commit/c7e80fbd) (2025-12-11, "Pass --time to nerdctl stop to avoid timeouts"). The switch to dockerd happened in [`318ade60`](https://github.com/microsoft/WSL/commit/318ade60) (2026-01-07, "Switch from nerdctl to dockerd").

Dockerd was likely chosen over podman because:

1. **The wslc Windows host needs a stable API to talk to.** The `DockerHTTPClient` ([`src/windows/wslcsession/DockerHTTPClient.h`](https://github.com/microsoft/WSL/blob/master/src/windows/wslcsession/DockerHTTPClient.h)) communicates over HTTP using the Docker Engine API v1.44. This is a well-documented, stable REST API. Podman's API, while Docker-compatible, has subtle differences that would add a compatibility burden.

2. **Image build support.** `wslc image build` delegates to dockerd's build endpoint, which handles Dockerfile processing, layer caching, and multi-stage builds. Podman's build (via buildah) is capable but has different semantics in some edge cases.

3. **Event tracking.** The `DockerEventTracker` class relies on Docker's event stream for container state synchronization ([`src/windows/wslcsession/WSLCSession.cpp:404`](https://github.com/microsoft/WSL/blob/master/src/windows/wslcsession/WSLCSession.cpp#L404)).

That said, there is active exploration of podman as an alternative. A POC branch (`user/chemwolf6922/wlsc-on-podman-poc`) exists with commits from June 2026, including [`96f04ad0`](https://github.com/microsoft/WSL/commit/96f04ad0) ("Give wslc volume prune docker-compatible semantics via podman patch") and several test fixes for podman compatibility. This suggests the team is evaluating podman as a possible backend.

The architecture (containerd started first at [`WSLCSession.cpp:639-654`](https://github.com/microsoft/WSL/blob/master/src/windows/wslcsession/WSLCSession.cpp#L639), then dockerd on top at [`WSLCSession.cpp:656-667`](https://github.com/microsoft/WSL/blob/master/src/windows/wslcsession/WSLCSession.cpp#L656)) means the daemon overhead is bounded — there's exactly one containerd + one dockerd per session VM, not per container.

---

## 9. Does the wslc Windows client share source code with the Docker client? Are there Docker subcommands missing in wslc? Are there wslc subcommands without a Docker equivalent?

**No shared source code.** wslc is implemented entirely in C++ from scratch. It does not import Docker or Moby client libraries. The only Docker-related dependency is a JSON schema header ([`src/windows/inc/docker_schema.h`](https://github.com/microsoft/WSL/blob/master/src/windows/inc/docker_schema.h)) for serializing/deserializing Docker Engine API v1.44 responses.

**wslc commands** (from [`src/windows/wslc/commands/RootCommand.cpp:31-67`](https://github.com/microsoft/WSL/blob/master/src/windows/wslc/commands/RootCommand.cpp#L31)):

| Category | Commands |
|---|---|
| `container` | `attach`, `create`, `exec`, `export`, `inspect`, `kill`, `ls`/`ps`, `logs`, `prune`, `rm`, `run`, `start`, `stats`, `stop` |
| `image` | `build`, `import`, `inspect`, `ls`/`images`, `load`, `pull`, `push`, `prune`, `rm`/`rmi`, `save`, `tag` |
| `network` | `create`, `inspect`, `ls`, `prune`, `rm` |
| `volume` | `create`, `inspect`, `ls`, `prune`, `rm` |
| `registry` | `login`, `logout` |
| `system session` | `enter`, `list`, `run`, `shell`, `terminate` |
| `settings` | `reset` |
| root-level | `inspect`, `version` |

**Docker commands missing from wslc:**
- All Swarm mode commands (`service`, `stack`, `node`, `swarm`)
- `compose` (separate tool in Docker)
- `buildx` (advanced multi-platform builds)
- `plugin`, `config`, `secret`, `checkpoint`, `context`
- `search`, `info`, `system df`, `system prune`, `events`
- `cp` (copy files to/from containers)
- `diff`, `commit`, `rename`, `wait`, `pause`, `unpause`, `update`, `top`, `port`

**wslc commands without Docker equivalents:**
- `system session` (list, enter, run, shell, terminate) — managing the session/VM layer
- `settings` / `settings reset` — editing wslc user configuration

---

## 10. How does wslc compare to Microsoft Execution Containers (MXC)? Is there evidence that wslc will become the default backend for MXC on Windows?

MXC and wslc operate at different levels. [MXC](https://github.com/microsoft/mxc) is a policy-driven sandboxing SDK — it defines *what* an agent/process can access via JSON policy, and delegates the actual containment to a backend. wslc is one of those backends.

The MXC GitHub repository lists **WSLC** as one of its containment backends alongside ProcessContainer, Windows Sandbox, LXC, Bubblewrap, Seatbelt (macOS), MicroVM/NanVix, Hyperlight, and IsolationSession.

There is no reference to MXC anywhere in the WSL repository source code (`git log --grep="MXC"` and source-level grep both return nothing). This suggests the integration is on MXC's side — MXC calls into the wslc SDK (`wslcsdk.dll` / `Microsoft.WSL.Containers` NuGet) when the policy specifies a WSLC backend.

Whether wslc becomes the *default* backend for MXC on Windows is not determined by the WSL codebase. The MXC architecture supports multiple backends precisely so the right isolation level can be chosen per workload. For Linux container workloads on Windows, wslc is the natural choice; for lighter sandboxing needs, other backends may be preferred.

The wslc SDK is explicitly designed for programmatic integration. It ships as a NuGet package (`Microsoft.WSL.Containers`) with C/C++ and C#/WinRT projections ([`nuget/Microsoft.WSL.Containers/docs/README.MD`](https://github.com/microsoft/WSL/blob/master/nuget/Microsoft.WSL.Containers/docs/README.MD)), making it straightforward for MXC or any other framework to use.

---

## 11. How does wslc compare to other microVMs? Can it be considered a VMM or does it belong to another abstraction layer?

wslc is **not a VMM**. It belongs to a higher abstraction layer — it is a **container runtime** that *uses* a VMM (the Windows hypervisor via HCS) as its isolation backend.

Comparing to the taxonomy described in the [microVM blog post](https://emirb.github.io/blog/microvm-2026/):

| Layer | microVM ecosystem (Linux) | wslc (Windows) |
|---|---|---|
| **VMM** | Firecracker, Cloud Hypervisor, QEMU, libkrun | Windows Hypervisor Platform / HCS (with OpenVMM being explored — see commit [`4d3fa5c5`](https://github.com/microsoft/WSL/commit/4d3fa5c5)) |
| **Container runtime** | containerd + Kata, Podman + krun | **wslc** (containerd + dockerd inside Hyper-V VM) |
| **CLI / Orchestration** | Docker CLI, nerdctl, crictl | **wslc.exe** |

Key differences from Linux microVMs like Firecracker:

- **Not one-VM-per-container.** wslc shares a VM across containers in the same session (closer to Docker Desktop's model). Firecracker and Apple Containers create one VM per container.
- **Not KVM-based.** wslc uses Windows Hyper-V via HCS, not Linux KVM. The VMs are created by `HcsCreateComputeSystem()`, not by opening `/dev/kvm`.
- **Docker-compatible management.** wslc runs dockerd inside the VM and talks to it over the Docker Engine API, so it gets Docker image format, Dockerfile builds, and registry compatibility for free.
- **An OpenVMM exploration branch exists** ([`4d3fa5c5`](https://github.com/microsoft/WSL/commit/4d3fa5c5), "Add openvmm as vmm option for wslc vms"), which is Microsoft's Rust-based VMM (part of the [OpenHCL](https://github.com/microsoft/openvmm) project). This could bring wslc closer to the Firecracker/Cloud Hypervisor model.

wslc is most comparable to Docker Desktop for Windows (which also runs containers inside a Hyper-V VM), but with tighter OS integration, per-session isolation, an SDK for programmatic use, and no dependency on Docker Desktop licensing.

---

## 12. How does wslc compare to Apple Containers? To libkrun?

**vs. [Apple Containers](https://github.com/apple/container):**

| | wslc | Apple Containers |
|---|---|---|
| Platform | Windows (Hyper-V) | macOS (Virtualization.framework) |
| Architecture | Shared VM per session | One VM per container |
| Language | C++ | Swift |
| Container engine | containerd + dockerd inside VM | Custom, no Docker dependency |
| SDK | C/C++ and C#/WinRT ([NuGet](https://github.com/microsoft/WSL/blob/master/nuget/Microsoft.WSL.Containers/docs/README.MD)) | Swift framework |
| GPU passthrough | Yes (DXG, CDI) | Apple GPU via Virtualization.framework |
| OCI compatibility | Full (Docker API v1.44) | OCI images, no Docker API |
| Build integration | MSBuild and CMake targets | Xcode integration |
| Open source | Yes (MIT) | Yes (Apache 2.0) |

Apple Containers' one-VM-per-container model provides stronger isolation between containers at the cost of more overhead per container. wslc's shared-VM model is more memory-efficient when running many containers.

**vs. [libkrun](https://github.com/libkrun/libkrun):**

libkrun is an embeddable VMM library (KVM on Linux, Hypervisor.framework on macOS). It sits one layer below wslc — it creates and manages VMs, while wslc manages containers. libkrun could theoretically be used *as* the VMM backend for a container runtime, but wslc uses Hyper-V/HCS instead.

| | wslc | libkrun |
|---|---|---|
| What it is | Container runtime + SDK | Embeddable VMM library |
| Hypervisor | Hyper-V (HCS) | KVM / Hypervisor.framework |
| Platform | Windows | Linux, macOS |
| Container management | Yes (full lifecycle) | No (provides VM isolation only) |
| Sub-200ms boot | No (VM boots Linux kernel + containerd + dockerd) | Yes |
| Use case | Running Linux containers on Windows | Adding VM isolation to existing tools |

wslc and libkrun are not competitors — they solve different problems. libkrun would be a building block if someone wanted to build a container runtime like wslc on Linux/macOS.

---

## 13. Where is the source code that specifies the packages installed in the wslc VM?

The wslc VM root filesystem (`system.vhd`) is **not built from source in this repository**. It comes pre-built from the `Microsoft.WSLg` NuGet package (currently version 1.0.78, defined in [`packages.config`](https://github.com/microsoft/WSL/blob/master/packages.config)). The VHDs are at `packages/Microsoft.WSLg.1.0.78/build/native/bin/{x64,arm64}/system.vhd`.

At boot time, the VHD is attached read-only as a SCSI disk ([`src/windows/service/exe/HcsVirtualMachine.cpp:256-285`](https://github.com/microsoft/WSL/blob/master/src/windows/service/exe/HcsVirtualMachine.cpp#L256)). The path is either the MSI package path + `system.vhd`, or a build-time override via `WSL_SYSTEM_DISTRO_PATH` ([`CMakeLists.txt:497`](https://github.com/microsoft/WSL/blob/master/CMakeLists.txt#L497)).

What we can see from the source is what binaries the code expects to exist inside the VM:

- `/usr/bin/containerd` — started by [`WSLCSession.cpp:644`](https://github.com/microsoft/WSL/blob/master/src/windows/wslcsession/WSLCSession.cpp#L644)
- `/usr/bin/dockerd` — started by [`WSLCSession.cpp:658`](https://github.com/microsoft/WSL/blob/master/src/windows/wslcsession/WSLCSession.cpp#L658)
- `/bin/sh` — used for certificate installation scripts ([`WSLCSession.cpp:684`](https://github.com/microsoft/WSL/blob/master/src/windows/wslcsession/WSLCSession.cpp#L684))
- Kernel modules VHD at `tools/modules.vhd` ([`HcsVirtualMachine.cpp:239-243`](https://github.com/microsoft/WSL/blob/master/src/windows/service/exe/HcsVirtualMachine.cpp#L239))

The Docker daemon config is written at runtime to `/etc/docker/daemon.json` by `WSLCInit.cpp` ([`src/linux/init/WSLCInit.cpp:111-124`](https://github.com/microsoft/WSL/blob/master/src/linux/init/WSLCInit.cpp#L111)), and CDI GPU specs are written to `/etc/cdi/microsoft.com-wslc.json` ([`WSLCInit.cpp:71-109`](https://github.com/microsoft/WSL/blob/master/src/linux/init/WSLCInit.cpp#L71)).

For test distros, a package list is visible in [`tools/test/build-test-distro.ps1:50-57`](https://github.com/microsoft/WSL/blob/master/tools/test/build-test-distro.ps1#L50), but this is for testing, not the production system VHD.

---

## 14. Is there any design document for wslc?

There is limited design documentation in the repository:

- **Boot process**: [`doc/docs/technical-documentation/boot-process.md`](https://github.com/microsoft/WSL/blob/master/doc/docs/technical-documentation/boot-process.md) — describes the WSL2 VM boot sequence (shared with wslc), including HCS VM creation, kernel boot, and mini_init initialization.

- **SDK API design**: A design document for the `WslcGetCLISession` API existed at `doc/docs/technical-documentation/wslc-get-cli-session-api.md` (commit [`5020809e`](https://github.com/microsoft/WSL/commit/5020809e), 2026-04-08). It documented the app-owns-lifecycle model (`App → Library → Session → Container`), HRESULT return types, ref-counted handles, and process-scoped publish-once semantics. However, this file may have been removed or moved since.

- **IDL interfaces**: The COM interface definitions serve as de facto design documents:
  - [`src/windows/service/inc/wslc.idl`](https://github.com/microsoft/WSL/blob/master/src/windows/service/inc/wslc.idl) — internal service interfaces (ABI-breaking changes allowed since client & server ship together)
  - [`src/windows/service/inc/WSLCCompat.idl`](https://github.com/microsoft/WSL/blob/master/src/windows/service/inc/WSLCCompat.idl) — SDK-facing interfaces (must maintain backward compatibility, as noted in commit [`ff898462`](https://github.com/microsoft/WSL/commit/ff898462), "Scope ABI-stability guidance to SDK-facing WSLCCompat.idl")

- **OpenVMM POC**: An OpenVMM design document was briefly tracked and then removed ([`335c867d`](https://github.com/microsoft/WSL/commit/335c867d), "Stop tracking openvmm wslc POC doc").

- **The main merge PR** [`#40366`](https://github.com/microsoft/WSL/pull/40366) (commit `358ab87d`) contains the most comprehensive architectural overview in its commit message, describing all key components: wslc.exe CLI, wslcsession.exe per-user service, WSLC SDK, networking, storage, and GPU passthrough.

- **SDK README**: [`nuget/Microsoft.WSL.Containers/docs/README.MD`](https://github.com/microsoft/WSL/blob/master/nuget/Microsoft.WSL.Containers/docs/README.MD) — documents the C/C++, C#/WinRT, MSBuild, and CMake integration points.

There is no single comprehensive architecture document. The technical decisions are best understood by reading the IDL files, the commit history, and the main PR description.
