# ppc64le Architecture Support VEP

## Overview

This directory contains the Virtualization Enhancement Proposal (VEP) for restoring and officially supporting the ppc64le (IBM PowerPC 64-bit Little Endian) architecture in KubeVirt.

## Files

- [`vep.md`](./vep.md) - The complete VEP document

## Key Technical Focus Areas

### 1. Bazel Build Infrastructure
- **Problem**: Missing native and cross-compilation toolchain support for ppc64le
- **Solution**: Register ppc64le toolchains for bazeldnf, rules_oci, and aspect_bazel_lib

### 2. RPM Dependency Management
- **Problem**: Missing RPM trees for CentOS Stream 9/10 ppc64le
- **Solution**: Update rpm-deps.sh and repo configurations to generate all 11 component trees

### 3. Cross-Compilation Support
- **Problem**: Cannot cross-compile CGO components from x86_64 to ppc64le
- **Solution**: Add powerpc64le-linux-gnu-gcc to kubevirt/builder and configure cc_toolchain

### 4. CI/CD Integration
- **Problem**: No automated builds or testing for ppc64le
- **Solution**: Add ppc64le to GitHub Actions matrix with self-hosted runners

## Implementation Phases

### Alpha
- Restore RPM infrastructure
- Document manual build procedures
- Provide Docker/Podman alternative for downstream

### Beta
- Complete Bazel toolchain registration
- Implement cross-compilation support
- Integrate CI/CD pipeline

### GA
- Full automation and testing
- Production validation
- Remove feature gate

## Upstream Dependencies

This VEP tracks the following upstream dependencies:

1. **rmohr/bazeldnf** - PR #48 / Issue #45
   - Waiting for official ppc64le binary releases
   - Currently using workaround with manual binary injection

2. **rules_oci** - Toolchain registration
   - Need ppc64le support for regctl and related tools

3. **aspect_bazel_lib** - Utility toolchains
   - Need ppc64le binaries for zstd, coreutils, jq

## Alternative Build Strategy

Until Bazel infrastructure is complete, downstream builds can use standard Docker/Podman multi-stage builds:

```dockerfile
FROM registry.access.redhat.com/ubi9/ubi:latest AS builder
RUN dnf install -y golang gcc git make
WORKDIR /workspace
COPY . .
RUN make build ARCH=ppc64le

FROM registry.access.redhat.com/ubi9/ubi-minimal:latest
COPY --from=builder /workspace/bin/* /usr/bin/
ENTRYPOINT ["/usr/bin/virt-launcher"]
```

## Next Steps

1. **Submit VEP for Review**
   - Create enhancement issue in kubevirt/enhancements
   - Link to this VEP directory
   - Request SIG-compute review

2. **Engage with Upstream Projects**
   - Follow up on rmohr/bazeldnf PR #48
   - Request ppc64le support in rules_oci
   - Request ppc64le binaries in aspect_bazel_lib

3. **Begin Alpha Implementation**
   - Update hack/rpm-deps.sh
   - Generate RPM trees for ppc64le
   - Document build procedures

4. **Secure Hardware Access**
   - Identify Power9/Power10 systems for testing
   - Set up self-hosted GitHub Actions runner
   - Configure CI/CD pipeline

## Contributing

This VEP follows the standard KubeVirt enhancement process:

1. VEP must be approved by SIG-compute
2. Implementation PRs reference this VEP
3. Feature gate guards alpha/beta functionality
4. Graduation criteria must be met for each phase

## References

- [KubeVirt Enhancement Process](https://github.com/kubevirt/enhancements/blob/main/README.md)
- [IBM Z Secure Execution VEP](../19-IBM-Z-secure-execution/secure-execution.md) - Similar architecture support
- [Heterogeneous Cluster Support](../../sig-storage/dic-on-heterogeneous-cluster/dic-on-heterogeneous-cluster.md) - Multi-arch patterns
- [TDX/SEV-SNP VEP](../80-TDX-SEV-SNP/tdx-sev-snp-proposal.md) - Feature gate patterns

## Contact

- **SIG**: sig-compute
- **Slack**: #kubevirt-dev
- **Mailing List**: kubevirt-dev@googlegroups.com