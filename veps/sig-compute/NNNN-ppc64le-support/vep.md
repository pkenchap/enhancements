# VEP #NNNN: Restore and Support ppc64le Architecture in KubeVirt

## VEP Status Metadata

### Target releases

- This VEP targets alpha for version: 
- This VEP targets beta for version:
- This VEP targets GA for version:

### Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue created, which links to VEP dir in [kubevirt/enhancements] (not the initial VEP PR)
- [ ] (R) Alpha target version is explicitly mentioned and approved
- [ ] (R) Beta target version is explicitly mentioned and approved
- [ ] (R) GA target version is explicitly mentioned and approved

## Overview

This proposal outlines the technical requirements and implementation strategy to restore official ppc64le (PowerPC 64-bit Little Endian) architecture support in the upstream KubeVirt main branch. The focus is on fixing the broken Bazel build infrastructure, establishing proper cross-compilation toolchains, and ensuring native ppc64le builds can succeed in CI/CD pipelines.

## Motivation

KubeVirt is a critical component of the Kubernetes ecosystem, enabling virtualized workloads on container orchestration platforms. The ppc64le architecture is widely used in enterprise data centers and cloud providers, particularly in IBM Z and Power Systems environments. Restoring native ppc64le support is essential.

## Problem Statement

KubeVirt's build system currently lacks functional support for ppc64le architecture, despite prior support in earlier versions. The removal or degradation of ppc64le support has created multiple critical gaps:

1. **Missing Bazel Toolchain Support**
   - The Bazel workspace lacks native execution toolchains for ppc64le
   - Upstream Bazel execution tools (`bazeldnf`, `rules_oci` binaries like `regctl`, `zstd`, `coreutils`, `jq`) do not have registered ppc64le binaries
   - Attempting to build on native PowerPC hosts results in fatal `No matching toolchains found` errors
   - Bazel 6.5.0's strict toolchain resolution prevents fallback mechanisms

2. **Incomplete RPM Infrastructure**
   - The `rpm/BUILD.bazel` definitions for ppc64le were removed or never completed
   - `bazeldnf` sandbox trees for CentOS Stream 9/10 ppc64le repositories are missing
   - RPM dependency resolution for critical components (`launcherbase`, `handlerbase`, `passt_tree`, etc.) fails for ppc64le

3. **Cross-Compilation Gaps**
   - Cross-compiling from x86_64 CI nodes to ppc64le fails due to missing CGO cross-compilers
   - The workspace lacks `powerpc64le-linux-gnu-gcc` and associated C/C++ toolchain configurations
   - `virt-launcher` and other CGO-dependent components cannot be cross-compiled for ppc64le

4. **Enterprise Adoption Barriers**
   - Organizations running IBM Power Systems cannot adopt KubeVirt without functional ppc64le support
   - Power systems offer superior memory bandwidth and I/O performance for virtualized workloads
   - Lack of ppc64le support prevents KubeVirt from being truly architecture-agnostic

### Business Impact

- **Enterprise Market**: IBM Power Systems represent a significant enterprise infrastructure segment
- **Cloud Providers**: Service providers offering Power-based infrastructure need Kubernetes-native virtualization
- **Competitive Position**: Other virtualization platforms support ppc64le, creating a feature gap
- **Community Growth**: Power architecture community cannot contribute to or adopt KubeVirt

## Goals

1. **Add support for ppc64le**
   - Enable successful Bazel builds on native ppc64le hosts
   - Register ppc64le execution toolchains for all required Bazel tools
   - Generate complete RPM dependency trees for ppc64le

2. **Enable Cross-Compilation from x86_64**
   - Add CGO cross-compilation toolchains to kubevirt/builder container
   - Configure `cc_toolchain` for powerpc64le-linux-gnu targets
   - Enable x86_64 CI nodes to produce ppc64le artifacts

3. **Establish CI/CD Pipeline**
   - Integrate ppc64le builds into existing CI infrastructure
   - Publish multi-architecture container images including ppc64le
   - Validate ppc64le functionality through automated testing

4. **Maintain Upstream Standards**
   - Follow KubeVirt's existing multi-architecture patterns (amd64, arm64, s390x)
   - Ensure changes are maintainable and don't break existing architectures
   - Document build procedures and troubleshooting steps

## Non Goals

- **Cross-Architecture VM Emulation**: Running x86_64 VMs on ppc64le hosts (or vice versa) through QEMU emulation
- **Big Endian Support**: ppc64 (big endian) architecture is explicitly excluded
- **Power-Specific Optimizations**: Performance tuning specific to Power processors (future enhancement)
- **Immediate GA Status**: Initial implementation targets alpha/beta graduation path
- **Bazel Workspace Overhaul**: Complete redesign of build system (addressed separately if needed)

## Definition of Users

- **CI/CD Engineers**: Maintaining KubeVirt build infrastructure and release pipelines
- **Platform Engineers**: Deploying KubeVirt on IBM Power Systems infrastructure
- **Downstream Distributors**: Building KubeVirt packages for ppc64le distributions
- **Enterprise IT Teams**: Operating Power-based virtualization infrastructure
- **Open Source Contributors**: Developing and testing KubeVirt on Power hardware

## User Stories

- As a CI/CD engineer, I need the Bazel build to succeed on ppc64le hosts so that I can produce official ppc64le container images
- As a platform engineer, I want to deploy KubeVirt on my kubernetes cluster on Power10/Power11 using official upstream images
- As a downstream distributor, I need clear documentation on building KubeVirt for ppc64le so I can package it for my distribution
- As an enterprise IT administrator, I want to run KubeVirt on my existing Power infrastructure without maintaining custom forks
- As a contributor, I want to develop and test KubeVirt features on my Power workstation

## Repos

- [kubevirt/kubevirt](https://github.com/kubevirt/kubevirt) - Core implementation and build system
- [kubevirt/project-infra](https://github.com/kubevirt/project-infra) - CI/CD infrastructure
- [rmohr/bazeldnf](https://github.com/rmohr/bazeldnf) - RPM dependency management tool
- [kubevirt/kubevirt.github.io](https://github.com/kubevirt/kubevirt.github.io) - Documentation

## Design

### Proof of Concept Validation

A proof-of-concept implementation has successfully demonstrated that ppc64le support is technically feasible:

1. **Go Toolchain Success**
   - Fixed Bazel 6.5.0 strict toolchain resolution by defining `ppc64le_strict` constraint in `user.bazelrc`
   - Proved that KubeVirt's Go codebase compiles flawlessly for PowerPC architecture
   - No code changes required in core KubeVirt components

2. **RPM Infrastructure Restoration**
   - Authored custom script to execute `bazeldnf` CLI for ppc64le
   - Successfully generated all 11 deterministic RPM trees:
     - `launcherbase`, `handlerbase`, `passt_tree`
     - `libvirt`, `qemu`, `seabios`, `edk2`, `swtpm`
     - Additional dependency trees for complete runtime environment
   - Updated `rpm/repo-cs10.yaml` to point to modern CentOS Stream 10 HTTPS mirrors
   - Integrated PowerPC Virt SIG repository (`kvm-power`) for Power-specific packages

3. **Toolchain Workarounds**
   - Proved native compilation possible by injecting custom `bazeldnf_toolchain`
   - Manually downloaded ppc64le binary and registered in `tools/bazeldnf/BUILD.bazel`
   - Demonstrated path forward for official toolchain registration

### Proposed Architecture

#### Phase 1: RPM Infrastructure Restoration

**Update RPM Generation Scripts**

Modify `hack/rpm-deps.sh` and related scripts to include ppc64le in architecture generation loops:

```bash
# Current: ARCHITECTURES="amd64 arm64 s390x"
# Proposed: ARCHITECTURES="amd64 arm64 s390x ppc64le"

for arch in ${ARCHITECTURES}; do
    generate_rpm_tree "${arch}"
done
```

**Update Repository Configuration**

Ensure `rpm/repo-cs10.yaml` includes ppc64le-specific repositories:

```yaml
repositories:
  - name: baseos-ppc64le
    baseurl: https://mirror.stream.centos.org/SIGs/10-stream/virt/ppc64le/kvm-power/
  - name: appstream-ppc64le
    baseurl: https://mirror.stream.centos.org/10-stream/AppStream/ppc64le/os/
  - name: crb-ppc64le
    baseurl: https://mirror.stream.centos.org/10-stream/CRB/ppc64le/os/
```

**Generate Complete RPM Trees**

Execute `bazeldnf` for all required component trees:

```bash
# Generate launcherbase RPM tree for ppc64le
bazel run //rpm:bazeldnf -- rpmtree \
    --arch ppc64le \
    --repofile rpm/repo-cs10.yaml \
    --name launcherbase \
    --output rpm/BUILD.bazel

# Repeat for all 11 component trees
```

#### Phase 2: Bazel Toolchain Registration

**Update bazeldnf Dependency**

Migrate from outdated fork to official repository with ppc64le support:

```python
# WORKSPACE file update
http_archive(
    name = "bazeldnf",
    sha256 = "...",
    urls = [
        "https://github.com/rmohr/bazeldnf/releases/download/v0.x.x/bazeldnf-v0.x.x.tar.gz",
    ],
)

# Register ppc64le toolchain (pending upstream PR #48 / Issue #45)
register_toolchains(
    "@bazeldnf//toolchains:ppc64le_toolchain",
)
```

**Register rules_oci Toolchains**

Add ppc64le binaries for container image operations:

```python
# WORKSPACE or MODULE.bazel
oci_register_toolchains(
    name = "oci",
    regctl_version = "v0.x.x",
    platforms = [
        "@platforms//cpu:x86_64",
        "@platforms//cpu:aarch64", 
        "@platforms//cpu:s390x",
        "@platforms//cpu:ppc64le",  # Add ppc64le
    ],
)
```

**Register aspect_bazel_lib Toolchains**

Ensure utilities (zstd, coreutils, jq) support ppc64le:

```python
# WORKSPACE
http_archive(
    name = "aspect_bazel_lib",
    # ... version details
)

# Register ppc64le variants
register_toolchains(
    "@aspect_bazel_lib//lib:zstd_ppc64le_toolchain",
    "@aspect_bazel_lib//lib:coreutils_ppc64le_toolchain",
    "@aspect_bazel_lib//lib:jq_ppc64le_toolchain",
)
```

#### Phase 3: Cross-Compilation Support

**Extend kubevirt/builder Container**

Add CGO cross-compilation toolchain to builder image:

```dockerfile
# In kubevirt/builder Dockerfile
RUN dnf install -y \
    gcc-powerpc64le-linux-gnu \
    g++-powerpc64le-linux-gnu \
    binutils-powerpc64le-linux-gnu

# Set up sysroot for cross-compilation
RUN mkdir -p /usr/powerpc64le-linux-gnu/sysroot
```

**Register cc_toolchain for ppc64le**

Create `toolchain/cc_toolchain_config_ppc64le.bzl`:

```python
def _ppc64le_cc_toolchain_config_impl(ctx):
    return cc_common.create_cc_toolchain_config_info(
        ctx = ctx,
        toolchain_identifier = "ppc64le-linux-gnu",
        host_system_name = "x86_64-unknown-linux-gnu",
        target_system_name = "powerpc64le-unknown-linux-gnu",
        target_cpu = "ppc64le",
        target_libc = "glibc",
        compiler = "gcc",
        abi_version = "unknown",
        abi_libc_version = "unknown",
        tool_paths = [
            tool_path(name = "gcc", path = "/usr/bin/powerpc64le-linux-gnu-gcc"),
            tool_path(name = "ld", path = "/usr/bin/powerpc64le-linux-gnu-ld"),
            tool_path(name = "ar", path = "/usr/bin/powerpc64le-linux-gnu-ar"),
            tool_path(name = "cpp", path = "/usr/bin/powerpc64le-linux-gnu-cpp"),
            tool_path(name = "gcov", path = "/usr/bin/powerpc64le-linux-gnu-gcov"),
            tool_path(name = "nm", path = "/usr/bin/powerpc64le-linux-gnu-nm"),
            tool_path(name = "objdump", path = "/usr/bin/powerpc64le-linux-gnu-objdump"),
            tool_path(name = "strip", path = "/usr/bin/powerpc64le-linux-gnu-strip"),
        ],
        cxx_builtin_include_directories = [
            "/usr/powerpc64le-linux-gnu/include",
            "/usr/lib/gcc-cross/powerpc64le-linux-gnu/11/include",
        ],
    )

ppc64le_cc_toolchain_config = rule(
    implementation = _ppc64le_cc_toolchain_config_impl,
    attrs = {},
    provides = [CcToolchainConfigInfo],
)
```

**Update BUILD.bazel for Cross-Compilation**

```python
# toolchain/BUILD.bazel
cc_toolchain(
    name = "ppc64le_toolchain",
    all_files = ":empty",
    compiler_files = ":empty",
    dwp_files = ":empty",
    linker_files = ":empty",
    objcopy_files = ":empty",
    strip_files = ":empty",
    supports_param_files = 1,
    toolchain_config = ":ppc64le_toolchain_config",
    toolchain_identifier = "ppc64le-linux-gnu",
)

toolchain(
    name = "cc_toolchain_ppc64le",
    exec_compatible_with = [
        "@platforms//cpu:x86_64",
        "@platforms//os:linux",
    ],
    target_compatible_with = [
        "@platforms//cpu:ppc64le",
        "@platforms//os:linux",
    ],
    toolchain = ":ppc64le_toolchain",
    toolchain_type = "@bazel_tools//tools/cpp:toolchain_type",
)
```

#### Phase 4: CI/CD Integration

**GitHub Actions Workflow**

Add ppc64le to build matrix:

```yaml
name: Build Multi-Arch
on: [push, pull_request]

jobs:
  build:
    strategy:
      matrix:
        arch: [amd64, arm64, s390x, ppc64le]
        include:
          - arch: ppc64le
            runner: [self-hosted, linux, ppc64le]
    runs-on: ${{ matrix.runner }}
    steps:
      - uses: actions/checkout@v3
      - name: Build for ${{ matrix.arch }}
        run: |
          make bazel-build-images ARCH=${{ matrix.arch }}
```

**Multi-Architecture Manifest Generation**

Update image publishing to include ppc64le:

```bash
# Create and push multi-arch manifest
docker manifest create \
  quay.io/kubevirt/virt-launcher:latest \
  quay.io/kubevirt/virt-launcher:latest-amd64 \
  quay.io/kubevirt/virt-launcher:latest-arm64 \
  quay.io/kubevirt/virt-launcher:latest-s390x \
  quay.io/kubevirt/virt-launcher:latest-ppc64le

docker manifest push quay.io/kubevirt/virt-launcher:latest
```

### Runtime Configuration

**QEMU Machine Type Detection**

Update `pkg/virt-launcher/virtwrap/api/defaults.go`:

```go
func getDefaultMachineType(arch string) string {
    switch arch {
    case "amd64", "x86_64":
        return "q35"
    case "arm64", "aarch64":
        return "virt"
    case "s390x":
        return "s390-ccw-virtio"
    case "ppc64le":
        return "pseries"  // Power Systems machine type
    default:
        return "q35"
    }
}
```

**Firmware Configuration**

Configure SLOF (Slimline Open Firmware) for ppc64le:

```go
func getFirmwarePath(arch string) string {
    switch arch {
    case "ppc64le":
        return "/usr/share/qemu/slof.bin"
    case "amd64", "x86_64":
        return "/usr/share/OVMF/OVMF_CODE.fd"
    // ... other architectures
    }
}
```

## API Examples

### Basic ppc64le VM

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: fedora-ppc64le-vm
spec:
  domain:
    cpu:
      cores: 4
    machine:
      type: pseries
    resources:
      requests:
        memory: 4Gi
    devices:
      disks:
      - name: containerdisk
        disk:
          bus: virtio
      - name: cloudinitdisk
        disk:
          bus: virtio
  nodeSelector:
    kubernetes.io/arch: ppc64le
  volumes:
  - name: containerdisk
    containerDisk:
      image: quay.io/kubevirt/fedora-cloud-container-disk:ppc64le
  - name: cloudinitdisk
    cloudInitNoCloud:
      userData: |
        #cloud-config
        password: fedora
        chpasswd: { expire: False }
```

### Multi-Architecture Deployment

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: multi-arch-workload
spec:
  running: true
  template:
    spec:
      domain:
        resources:
          requests:
            memory: 2Gi
        devices:
          disks:
          - name: datavolumedisk
            disk: {}
      # Architecture-specific scheduling handled automatically
      volumes:
      - name: datavolumedisk
        dataVolume:
          name: os-image-ppc64le
```

## Alternatives

### Alternative 1: Downstream-Only Support

**Description**: Document that ppc64le builds must use standard Docker/Podman multi-stage builds instead of Bazel until toolchain infrastructure is complete.

**Implementation**:
```dockerfile
# Multi-stage Dockerfile for ppc64le
FROM registry.access.redhat.com/ubi9/ubi:latest AS builder
RUN dnf install -y golang gcc git make

WORKDIR /workspace
COPY . .
RUN make build ARCH=ppc64le

FROM registry.access.redhat.com/ubi9/ubi-minimal:latest
COPY --from=builder /workspace/bin/* /usr/bin/
ENTRYPOINT ["/usr/bin/virt-launcher"]
```

**Pros**:
- Immediate workaround for downstream distributors
- Uses native ppc64le host's gcc and dnf package manager
- No dependency on Bazel toolchain fixes

**Cons**:
- Diverges from upstream build methodology
- Requires maintaining separate build pipelines
- Cannot leverage Bazel's hermetic build guarantees
- Increases maintenance burden for downstream maintainers

**Decision**: Use as interim solution while upstream Bazel infrastructure is being fixed. Document clearly in build instructions.

### Alternative 2: Bazel Workspace Overhaul

**Description**: Complete redesign of KubeVirt's build system to use more modern Bazel patterns and toolchain registration.

**Pros**:
- Could solve multiple architecture support issues simultaneously
- Opportunity to modernize build infrastructure
- Better long-term maintainability

**Cons**:
- Massive scope, affects all architectures
- High risk of breaking existing builds
- Requires extensive testing across all platforms
- Delays ppc64le support significantly

**Decision**: Defer to separate enhancement proposal. Focus this VEP on minimal changes to restore ppc64le support.

### Alternative 3: Remove Bazel Dependency

**Description**: Migrate KubeVirt build system away from Bazel entirely to standard Go tooling and Makefiles.

**Pros**:
- Simpler build system
- Better Go ecosystem integration
- Easier for contributors to understand

**Cons**:
- Enormous migration effort
- Loss of Bazel's hermetic build benefits
- Would affect entire project, not just ppc64le
- Not aligned with current project direction

**Decision**: Rejected. Out of scope for this VEP.

## Scalability

### Build System Scalability

- **Parallel Builds**: Bazel's caching and parallelization work identically for ppc64le
- **CI Resource Usage**: ppc64le builds consume similar resources to other architectures
- **Artifact Storage**: Multi-arch manifests efficiently share layers across architectures

### Runtime Scalability

- **VM Density**: Power systems support high VM density due to superior memory bandwidth
- **Network Performance**: virtio-net scales well on Power architecture
- **Storage I/O**: Power systems' high I/O capabilities benefit virtualized workloads

### Multi-Architecture Cluster Scalability

- Heterogeneous clusters with mixed architectures supported
- Scheduling constraints ensure VMs run on appropriate architecture nodes
- No additional control plane overhead for ppc64le nodes

## Update/Rollback Compatibility

### Update Compatibility

**Bazel Workspace Changes**:
- All toolchain registrations are additive
- Existing x86_64, arm64, s390x builds unaffected
- New `ppc64le_strict` constraint only applies when building for ppc64le

**RPM Infrastructure**:
- New ppc64le RPM trees added alongside existing architecture trees
- No changes to existing architecture RPM definitions
- Repository configuration extended, not replaced

**Container Images**:
- Multi-arch manifests maintain backward compatibility
- Existing single-arch image references continue to work
- New ppc64le images added to manifest lists

### Rollback Compatibility

**Feature Gate Protection**:
```go
// Feature gate for ppc64le support
const SupportPPC64LE = "SupportPPC64LE"

// In feature gate initialization
featureGates[SupportPPC64LE] = &FeatureGate{
    Default:    false,
    PreRelease: Alpha,
}
```

**Build System Rollback**:
- Toolchain registrations can be commented out without affecting other architectures
- RPM tree generation scripts maintain backward compatibility
- CI/CD pipelines can exclude ppc64le from build matrix

**Runtime Rollback**:
- ppc64le-specific code paths guarded by feature gate
- Disabling feature gate prevents ppc64le VM creation
- No impact on existing VMs on other architectures

### Migration Path

**Phase 1 (Alpha)**: 
- Manual builds on ppc64le hosts using documented procedures
- Downstream distributors can build using Docker/Podman alternative
- Limited CI integration

**Phase 2 (Beta)**:
- Automated CI builds for ppc64le
- Official multi-arch images published
- Feature gate enabled by default in development builds

**Phase 3 (GA)**:
- Full CI/CD integration
- Feature gate removed
- ppc64le treated as first-class architecture

## Functional Testing Approach

### Unit Testing

**Toolchain Detection Tests**:
```go
func TestArchitectureDetection(t *testing.T) {
    tests := []struct {
        arch     string
        expected string
    }{
        {"ppc64le", "pseries"},
        {"amd64", "q35"},
        {"arm64", "virt"},
    }
    
    for _, tt := range tests {
        t.Run(tt.arch, func(t *testing.T) {
            machineType := getDefaultMachineType(tt.arch)
            assert.Equal(t, tt.expected, machineType)
        })
    }
}
```

**Build System Tests**:
- Verify RPM tree generation for ppc64le
- Validate Bazel toolchain resolution
- Test cross-compilation from x86_64 to ppc64le

### Integration Testing

**Native ppc64le Build Tests**:
```bash
# On ppc64le host
bazel build //cmd/virt-launcher:virt-launcher
bazel test //pkg/...
```

**Cross-Compilation Tests**:
```bash
# On x86_64 host
bazel build --platforms=@io_bazel_rules_go//go/toolchain:linux_ppc64le \
    //cmd/virt-launcher:virt-launcher
```

**Container Image Tests**:
```bash
# Verify multi-arch manifest
docker manifest inspect quay.io/kubevirt/virt-launcher:latest | \
    jq '.manifests[] | select(.platform.architecture == "ppc64le")'
```

### End-to-End Testing

**VM Lifecycle Tests**:
- Create ppc64le VM on Power node
- Verify VM boots successfully
- Test VM migration between ppc64le nodes
- Validate storage and network functionality

**CI/CD Pipeline Tests**:
- Automated build on ppc64le runner
- Image publishing to registry
- Multi-arch manifest creation
- Deployment validation

### Hardware Requirements

**Development Testing**:
- Access to Power9 or Power10 systems
- Minimum 16GB RAM, 8 cores
- CentOS Stream 9/10 or RHEL 9 installation

**CI/CD Infrastructure**:
- Self-hosted GitHub Actions runner on ppc64le
- Alternative: IBM Cloud PowerVS instances
- Community hardware donations (OSU Open Source Lab)

## Implementation History

*This section will be updated as implementation progresses*

- **TBD**: VEP created and submitted for review
- **TBD**: Proof of concept validation completed
- **TBD**: Phase 1 (RPM infrastructure) implementation
- **TBD**: Phase 2 (Bazel toolchains) implementation
- **TBD**: Phase 3 (Cross-compilation) implementation
- **TBD**: Phase 4 (CI/CD integration) implementation

## Graduation Requirements

### Alpha

- [ ] Feature gate `SupportPPC64LE` implemented and disabled by default
- [ ] RPM infrastructure restored for ppc64le (all 11 component trees)
- [ ] `hack/rpm-deps.sh` updated to include ppc64le in generation loops
- [ ] `rpm/repo-cs10.yaml` configured with ppc64le repositories
- [ ] Documentation for manual ppc64le builds using native hosts
- [ ] Documentation for downstream Docker/Podman build alternative
- [ ] Basic unit tests for ppc64le-specific code paths
- [ ] Proof of concept demonstrates successful native ppc64le build

### Beta

- [ ] Bazel toolchain registration complete for ppc64le:
  - [ ] `bazeldnf` with ppc64le support (tracking [rmohr/bazeldnf PR #48](https://github.com/rmohr/bazeldnf/pull/48))
  - [ ] `rules_oci` toolchains registered for ppc64le
  - [ ] `aspect_bazel_lib` utilities (zstd, coreutils, jq) for ppc64le
- [ ] Cross-compilation support implemented:
  - [ ] `powerpc64le-linux-gnu-gcc` added to kubevirt/builder
  - [ ] `cc_toolchain` configured for ppc64le cross-compilation
  - [ ] CGO components successfully cross-compile from x86_64
- [ ] CI/CD pipeline integration:
  - [ ] GitHub Actions workflow includes ppc64le builds
  - [ ] Self-hosted ppc64le runner configured
  - [ ] Multi-arch container images published with ppc64le
- [ ] Runtime configuration complete:
  - [ ] QEMU machine type detection for pseries
  - [ ] SLOF firmware configuration
  - [ ] Node labeling and scheduling for ppc64le
- [ ] Integration tests passing on ppc64le hardware
- [ ] Feature gate enabled by default in development builds

### GA

- [ ] Full CI/CD automation for ppc64le builds
- [ ] End-to-end tests passing consistently on ppc64le
- [ ] Live migration validated between ppc64le nodes
- [ ] Storage and networking feature parity with other architectures
- [ ] Performance benchmarking completed and documented
- [ ] Production deployments validated in enterprise environments
- [ ] Comprehensive documentation published:
  - [ ] Build instructions for ppc64le
  - [ ] Deployment guide for Power systems
  - [ ] Troubleshooting guide
  - [ ] Architecture-specific considerations
- [ ] Community feedback incorporated and issues resolved
- [ ] Feature gate removed, ppc64le support always enabled
- [ ] Long-term maintenance plan established