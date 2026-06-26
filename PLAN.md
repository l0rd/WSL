# Plan: Podman Backend for wslc

## Assessment: How Hard Is This?

**Short answer: Moderate.** The existing podman POC branch (`user/chemwolf6922/wlsc-on-podman-poc`) has already done the heaviest lifting. The branch touches 21 files with ~764 additions and ~415 deletions across 12 commits, and it passes E2E tests with adaptations. What remains is mostly polish, edge cases, and build/rootfs integration.

The core difficulty is not replacing dockerd with podman — podman's Docker-compatible API makes that almost a drop-in swap. The difficulty is in the **differences between the two APIs** at the edges: different event names, different error codes, different image reference formats, null vs empty arrays in JSON responses, and missing fields like `ConsoleSize`. The POC branch has already cataloged and fixed most of these.

---

## The Podman POC Branch: Current State

**Branch:** `origin/user/chemwolf6922/wlsc-on-podman-poc` (by chemwolf6922)
**Commits ahead of master:** 12

### What's Done

The branch replaces the dockerd + containerd dual-process model with a single `podman system service --time=0` process. Here's what was changed:

| Area | Change | Files |
|------|--------|-------|
| **Runtime swap** | `StartContainerd()` + `StartDockerd()` → `StartPodmanSystemService()` | `WSLCSession.cpp`, `WSLCSession.h` |
| **Socket path** | `/var/run/docker.sock` → `/run/podman/podman.sock` | `DockerHTTPClient.cpp:584` |
| **Storage path** | `/var/lib/docker` → `/var/lib/containers` | `WSLCSession.cpp:31` |
| **Readiness detection** | `"API listen on /var/run/docker.sock"` → `"API service listening on"` | `WSLCSession.cpp:33` |
| **Event names** | Docker's `"destroy"` → Podman's `"remove"` for both container and volume events | `DockerEventTracker.cpp` |
| **Null-safe JSON** | Custom `from_json` for `Image` struct to handle podman's `"RepoTags": null` | `docker_schema.h` |
| **Error format** | `errorDetail: {message: ...}` → `error: "..."` (flat string) | `docker_schema.h`, `WSLCSession.cpp` |
| **ConsoleSize removed** | Podman doesn't support `ConsoleSize` in exec create/start | `docker_schema.h` |
| **BuildKit removed** | Podman doesn't use BuildKit progress format | `docker_schema.h` |
| **Image ref stripping** | `docker.io/library/foo:tag` → `foo:tag`, `localhost/foo:tag` → `foo:tag` | `WSLCSession.cpp` (new functions) |
| **Streaming uploads** | New `SendStreamRequest()` for image load/import with chunked body | `DockerHTTPClient.cpp/h` |
| **Error code mapping** | Podman returns 500 for some client errors where Docker returns 4xx | `InspectTasks.cpp` |
| **Volume management** | New `WSLCVolumes` class for event-driven volume tracking | `WSLCVolumes.cpp/h` |
| **tmpfs mount** | Podman requires `/dev/shm` for its shm-based lock manager | `WSLCSession.cpp:323` |
| **E2E tests** | Updated 11 test files for podman semantics | `test/windows/wslc/e2e/*.cpp` |

### Commit Progression

1. `038770e9` — "bare minimum changes" (core runtime swap)
2. `d70d69b5` — "resolve schema incompatibility" (JSON null handling)
3. `3fc73dbd` — "Fixed load issue" (streaming uploads)
4. `27b91563` — "fix build issue"
5. `9cfea507` — "fixed e2e test container create volume issue"
6. `14374773` — "fix dns e2e issue"
7. `8ae6cd55` — "improve e2e test case for DNS"
8. `e8d63605` — "fix inspect issue"
9. `6418e3b0` — "fix image save issue"
10. `e688701f` — "fix image prefix issue" (registry prefix stripping)
11. `230617fe` — "fix error handling and e2e test"
12. `50c5204a` — "fix volume creation issue" (null RepoTags/RepoDigests)

### What's Not Done (Gaps in the POC)

1. **Build system (`system.vhd`)**: The POC doesn't address how podman gets installed in the VM rootfs. The current `system.vhd` ships containerd + dockerd. A podman-based rootfs needs `podman`, `conmon`, `crun` (or `runc`), `fuse-overlayfs`, `slirp4netns`, `containers-common`, etc.

2. **Image build**: `wslc image build` currently shells out to `docker build` inside the VM ([`WSLCSession.cpp:725`](src/windows/wslcsession/WSLCSession.cpp#L725)). The POC doesn't show how this works with podman — presumably `podman build` (backed by buildah) would replace it, but the build output format differs from Docker's BuildKit.

3. **GPU/CDI integration**: `WSLCInit.cpp` writes Docker-specific daemon config (`/etc/docker/daemon.json`) with `{"features": {"cdi": true}}`. Podman handles CDI differently — it reads CDI specs from `/etc/cdi/` natively without a daemon config file. The CDI spec file itself (`/etc/cdi/microsoft.com-wslc.json`) should still work.

4. **Trusted root certificates**: The current code writes certs to `/etc/ssl/certs/wsl-windows-roots.pem` because "dockerd and containerd read certificates found in `/etc/ssl/certs/`" ([`WSLCSession.cpp:679-681`](src/windows/wslcsession/WSLCSession.cpp#L679)). Podman uses the same system certificate store, so this should work as-is, but it hasn't been tested.

5. **Networking edge cases**: Container-to-container networking via `--network container:<id>` mode and custom bridge networks may have subtle differences in podman's netavark vs Docker's libnetwork.

6. **SDK tests**: `test/windows/WslcSdkTests.cpp` and `test/windows/WslcSdkWinRTTests.cpp` are not updated in the POC.

7. **Config file**: The POC still references `c_containerdSocket` (unused dead code at `WSLCSession.cpp:32`).

---

## Detailed Plan

### Phase 0: Setup and Branch Preparation

**Goal:** Get a working dev environment with the POC branch merged onto current master.

1. Create a new branch from master.
2. Cherry-pick or merge the 12 commits from `user/chemwolf6922/wlsc-on-podman-poc`.
3. Resolve merge conflicts (the POC branch is based on a slightly older master — the `WSLCVolumes` class and several E2E tests have since been updated on master).
4. Build and verify compilation succeeds.

**Risk:** The POC branch was created around June 18, 2026. Master has had ~30 commits since then. Most are additive (new CLI flags, new SDK APIs), but some touch the same files (e.g., `WSLCSession.cpp`, `DockerHTTPClient.cpp`). Expect moderate merge conflicts.

### Phase 1: Build a Podman-based `system.vhd`

**Goal:** Produce a root VHD containing podman instead of containerd + dockerd.

This is the biggest open question. The current `system.vhd` comes pre-built from the `Microsoft.WSLg` NuGet package (version 1.0.78, see [`packages.config`](packages.config)). The build process for that VHD is not in this repository.

**Options:**

**(a) Modify the existing rootfs:** Mount the existing `system.vhd`, install podman packages, remove dockerd/containerd, save as a new VHD. Quick for prototyping but brittle.

**(b) Use `RootVhdOverride`:** Build a custom VHD externally (e.g., using debootstrap or a container build), install podman + dependencies, and pass it via the `RootVhdOverride` session setting. The [`WSLCCompat.idl:253`](src/windows/service/inc/WSLCCompat.idl#L253) and [`HcsVirtualMachine.cpp:247-249`](src/windows/service/exe/HcsVirtualMachine.cpp#L247) already support this.

**(c) Use `WSL_SYSTEM_DISTRO_PATH` build flag:** Set this CMake variable to point to a custom VHD at build time ([`CMakeLists.txt:497`](CMakeLists.txt#L497)). This is the cleanest approach for a dev build.

**Required packages in the VHD:**
- `podman` (container runtime + Docker-compat API)
- `crun` or `runc` (OCI runtime)
- `conmon` (container monitor)
- `fuse-overlayfs` (rootless storage driver, may not be needed for root)
- `containers-common` (default configs, registries.conf)
- `slirp4netns` (rootless networking, may not be needed since wslc handles networking)
- `uidmap` (for newuidmap/newgidmap if needed)

**Deliverable:** A script or Dockerfile that produces a `system-podman.vhd` and instructions to use it with `RootVhdOverride` or `WSL_SYSTEM_DISTRO_PATH`.

### Phase 2: Fix the Runtime Swap (Merge POC + Fix Gaps)

**Goal:** Clean up the POC changes and address the known gaps.

**2a. Merge POC core changes:**
- `WSLCSession.cpp`: Replace `StartContainerd()` + `StartDockerd()` with `StartPodmanSystemService()`
- `WSLCSession.h`: Update member variables (`m_podmanSystemServiceProcess` instead of `m_containerdProcess` + `m_dockerdProcess`)
- `DockerHTTPClient.cpp`: Socket path change, `SendStreamRequest()` addition
- `DockerEventTracker.cpp`: Event name mapping (`"destroy"` → `"remove"`)
- `docker_schema.h`: Null-safe `Image`, flat error string, no BuildKit, no ConsoleSize

**2b. Fix image build flow:**
- The current `BuildImage()` in `WSLCSession.cpp` runs `docker build` inside the VM. This needs to become `podman build`.
- Podman build output is line-based, not BuildKit-formatted — the POC already removed BuildKit structs, but the build output parsing in `WSLCSession.cpp` needs verification.
- Check: does `wslc image build` pass the Dockerfile context correctly? The mechanism (tarball to stdin) should be the same.

**2c. Fix GPU/CDI config:**
- Remove or guard the `WriteDockerDaemonConfig()` call in [`WSLCInit.cpp:111-124`](src/linux/init/WSLCInit.cpp#L111). Podman doesn't use `/etc/docker/daemon.json`.
- Verify that the CDI spec at `/etc/cdi/microsoft.com-wslc.json` is picked up by podman. Podman supports CDI natively since v4.1, reading from `/etc/cdi/` and `/var/run/cdi/`.

**2d. Clean up dead code:**
- Remove the unused `c_containerdSocket` constant.
- Consider renaming `DockerHTTPClient` → something neutral if scope allows, or leave it (the Docker-compat API is still "Docker" semantically).

**2e. Podman-specific config:**
- Write `/etc/containers/registries.conf` if needed (for unqualified search registries like `docker.io`).
- Write `/etc/containers/storage.conf` if the default overlay driver needs tweaking.
- Write `/etc/containers/containers.conf` for any runtime defaults.

### Phase 3: Fix Edge Cases and Compatibility

**Goal:** Address the API differences the POC worked around, more robustly.

**3a. Image reference normalization:**
The POC adds `StripPodmanRegistryPrefix()` to convert podman's fully-qualified names back to docker-style short names. This works but is applied in several places. Consider:
- Moving it into `DockerHTTPClient` as a response post-processor so all callers get normalized refs.
- Or keeping it in `WSLCSession.cpp` but extracting to a shared utility.

**3b. Error code mapping:**
Podman returns HTTP 500 for some client errors. The POC handles this in `InspectTasks.cpp` with `IsPodmanNameValidationError()`. Consider a more general approach:
- A wrapper in `DockerHTTPClient` that re-maps known podman 500 patterns to appropriate HRESULTs.
- Or a podman-aware `DockerHTTPException` that checks the error body.

**3c. Container stats format:**
Verify that podman's `/containers/{id}/stats` response matches Docker's `ContainerStats` struct. Podman's stats may have minor differences (field naming, units).

**3d. Prune semantics:**
Commit [`96f04ad0`](https://github.com/microsoft/WSL/commit/96f04ad0) ("Give wslc volume prune docker-compatible semantics via podman patch") suggests podman's prune behavior differs. Verify all prune commands (container, image, volume, network) return the expected format.

**3e. Network driver:**
Podman uses netavark (not Docker's libnetwork) for networking. Verify:
- Bridge network creation/deletion
- Container DNS resolution within custom networks
- Port mapping behavior
- `--network container:<id>` mode

### Phase 4: Update Tests

**Goal:** All E2E and SDK tests pass with the podman backend.

**4a. Apply the POC's test changes** (already adapted 11 E2E test files).

**4b. Update SDK tests:**
- `test/windows/WslcSdkTests.cpp`
- `test/windows/WslcSdkWinRTTests.cpp`

**4c. Add podman-specific test coverage:**
- Test `StripPodmanRegistryPrefix()` with various registry patterns.
- Test image operations with dangling images (null RepoTags).
- Test error handling for podman's 500-for-client-errors pattern.
- Test volume prune semantics.

### Phase 5: Make It Switchable (Optional but Recommended)

**Goal:** Allow choosing between Docker and Podman backends without rebuilding.

This would make the POC more useful by not requiring a hard fork. Approach:

**5a. Compile-time flag:**
- Add a CMake option `WSLC_USE_PODMAN` (default OFF).
- Guard the daemon startup, socket path, and event mapping behind `#ifdef WSLC_USE_PODMAN`.
- Requires two builds but no runtime overhead.

**5b. Runtime configuration:**
- Add a setting to `WSLCSessionSettings` (e.g., `ContainerEngine` enum: Docker/Podman).
- The session process reads this and starts the appropriate daemon.
- More flexible but adds complexity to every code path that differs.

**5c. Separate `system.vhd` per backend:**
- Ship `system-docker.vhd` and `system-podman.vhd`.
- The session selects the right one based on the engine setting.
- Or use `RootVhdOverride` for the podman VHD during development.

For a POC, option (a) is sufficient. For upstreaming, option (b) is the right long-term approach.

---

## Summary

| Phase | Effort | Description |
|-------|--------|-------------|
| 0. Setup | 1 day | Branch prep, merge POC, resolve conflicts |
| 1. Rootfs | 2-3 days | Build a podman-based system.vhd |
| 2. Runtime swap | 2-3 days | Merge POC, fix build/GPU/config gaps |
| 3. Edge cases | 2-3 days | Error mapping, stats, prune, networking |
| 4. Tests | 1-2 days | Update SDK tests, add podman-specific coverage |
| 5. Switchable (optional) | 2-3 days | Compile-time or runtime engine selection |
| **Total** | **~8-12 days** (without Phase 5) | |

The POC branch has done roughly 60% of the work. What's left is build integration (the VHD), edge case handling, and testing. The core architectural decision — using podman's Docker-compatible API over a unix socket — is sound and validated by the POC.

---

## Key Files to Modify

| File | What Changes |
|------|-------------|
| `src/windows/wslcsession/WSLCSession.cpp` | Runtime swap, image ref stripping, cert handling |
| `src/windows/wslcsession/WSLCSession.h` | Process member variables |
| `src/windows/wslcsession/DockerHTTPClient.cpp` | Socket path, `SendStreamRequest()` |
| `src/windows/wslcsession/DockerHTTPClient.h` | Method signatures for streaming |
| `src/windows/wslcsession/DockerEventTracker.cpp` | Event name mapping |
| `src/windows/inc/docker_schema.h` | Null-safe Image, flat errors, no BuildKit/ConsoleSize |
| `src/windows/wslc/tasks/InspectTasks.cpp` | Error code remapping |
| `src/linux/init/WSLCInit.cpp` | Remove/guard Docker daemon config |
| `src/windows/wslcsession/WSLCVolumes.cpp/h` | Volume event tracking |
| `test/windows/wslc/e2e/*.cpp` | E2E test adaptations (11 files) |

## References

- POC branch: `origin/user/chemwolf6922/wlsc-on-podman-poc`
- Main WSLC merge PR: [`#40366`](https://github.com/microsoft/WSL/pull/40366) (commit `358ab87d`)
- Podman Docker-compat API docs: https://docs.podman.io/en/latest/_static/api.html
- Docker Engine API v1.44: https://docs.docker.com/engine/api/v1.44/
- CDI in Podman: https://github.com/cdi-dev/cdi
- Session settings IDL: [`src/windows/service/inc/WSLCCompat.idl`](src/windows/service/inc/WSLCCompat.idl)
- VM creation: [`src/windows/service/exe/HcsVirtualMachine.cpp`](src/windows/service/exe/HcsVirtualMachine.cpp)
