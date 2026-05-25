# copyfail_mitigation module

The `copyfail_mitigation` module allows operators to mitigate the [Copy Fail vulnerability](https://nvd.nist.gov/vuln/detail/CVE-2026-31431) across many cluster nodes in one operation.

> **Note:** This module supports Ubuntu 22.04 and 24.04 host operating systems.

> **Note:** This module is implemented and validated against the specific Ansible versions provided by MOSK for Ubuntu 22.04 and Ubuntu 24.04 in Cluster release 20.1.0: **Ansible Core 2.16.3** and **Ansible Collection 8.3.0**.
>
> To verify the Ansible version in a specific Cluster release, refer to the
> **Release artifacts > Management cluster artifacts > System and MCR artifacts**
> section of the required management Cluster release in the
> [MOSK documentation: Release notes](https://docs.mirantis.com/mosk/latest/release-notes.html).

# Version 1.0.0 (latest)

The `copyfail_mitigation` module is designed to mitigate the following vulnerability: [Local privilege escalation (Linux kernel "Copy-Fail", CVE-2026-31431) affects Mirantis OpenStack for Kubernetes (MOSK) cluster nodes when AF_ALG/AEAD is exposed](https://github.com/Mirantis/security/blob/main/advisories/0015.md).

The module accepts the following input parameters:

- `mitigate`: Boolean. Optional (defaults to `false`). Set to `true` to mitigate Copy-Fail on the host.
- `revert`: Boolean. Optional (defaults to `false`). Set to `true` to revert the mitigation and restore original kernel settings.

The `mitigate` and `revert` parameters cannot be set to `true` simultaneously.

## Mitigation workflow

When `mitigate` is set to `true`, the module performs the following actions:

1. Creates `/etc/modprobe.d/blacklist-algif_aead.conf` to block loading of the `algif_aead` kernel module.
2. Verifies whether `algif_aead` is currently loaded.

If the kernel module is loaded on the host, the `copyfail_mitigation` module also performs the following actions:

1. Attempts to unload `algif_aead` from the live kernel memory space.
2. Flushes the kernel page cache after a successful unload to purge any volatile in-memory exploitation.
3. Creates a reboot request.
4. Creates a local system marker file to ensure that `revert` can restore the original configuration if needed.

## Rebooting the hosts
After applying the mitigation, you must perform a reboot of affected hosts as soon as possible.
Plan a maintenance window and use the official Mirantis procedure on how to [Perform a graceful reboot of a cluster](https://docs.mirantis.com/mosk/latest/ops/general-operations/graceful-reboot.html).

## Reversion workflow

When `revert` is set to `true`, the `copyfail_mitigation` module safely rolls back changes:

1. Removes `/etc/modprobe.d/blacklist-algif_aead.conf` to allow loading of the `algif_aead` kernel module.
2. Inspects the local system marker file to verify whether the module was active prior to the initial mitigation.
   And if so, reloads the `algif_aead` module into the active kernel and tidies up tracking paths.

# Configuration examples

Example of a `HostOSConfiguration` custom resource for the `copyfail_mitigation` module version 1.0.0:

```yaml
apiVersion: kaas.mirantis.com/v1alpha1
kind: HostOSConfiguration
metadata:
  name: copyfail-mitigation
  namespace: default
spec:
  configs:
  - module: copyfail_mitigation
    moduleVersion: 1.0.0
    values:
      mitigate: true
      revert: false
  machineSelector:
    matchLabels:
      day2-copyfail-mitigation-label: 'true'
```

---