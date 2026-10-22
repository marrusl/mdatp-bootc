# Microsoft Defender for Endpoint on a bootc image

A worked example of building Microsoft Defender for Endpoint on Linux (`mdatp`) into a bootable container image, and the verification steps that show it running.

## Status, read this first

This is **unofficial**. It is not a Red Hat supported configuration, it is not a Microsoft supported configuration, and it carries no commitment from either vendor. It is a reproducible example published so that people evaluating image mode with Defender in the picture have something concrete to start from and to argue with.

Everything here was built and booted on the **CentOS Stream 9 and CentOS Stream 10 bootc base images**, not on Red Hat Enterprise Linux. Stream was used because it needs no build-time subscription entitlement. The Microsoft package is the same one in both cases, installed from Microsoft's own RHEL 9 and RHEL 10 repositories, but a Stream result is not a RHEL result and should not be read as one. See [Not verified](#not-verified) below.

## What this demonstrates

Two Containerfiles, one for each major version:

| File | Base image | Microsoft repo |
|---|---|---|
| `Containerfile.el9` | `quay.io/centos-bootc/centos-bootc:stream9` | `packages.microsoft.com/config/rhel/9/prod.repo` |
| `Containerfile.el10` | `quay.io/centos-bootc/centos-bootc:stream10` | `packages.microsoft.com/config/rhel/10/prod.repo` |

They are identical apart from the base image, the repo URL, and the GPG key, since the el10 repo is signed with `microsoft-2025.asc` rather than `microsoft.asc`. Both repos served the same package build, `mdatp-101.26081.0010-1.x86_64`, at the time of writing.

On a host installed from either image and booted:

- `mdatp.service` is active, running `wdavdaemon` out of `/opt/microsoft/mdatp/sbin`.
- SELinux is in enforcing mode, both Defender policy modules are loaded, and there were no AVC denials across the boot.
- Real-time protection reports `real_time_protection_subsystem: "fanotify"` and `supplementary_events_subsystem: "ebpf"`. There is no kernel module involved.
- Antimalware definitions updated from Microsoft within seconds of boot and report `definitions_status: "up_to_date"`.
- `mdatp health` reports `health_issues: ["missing license"]` and nothing else, which is the expected state for a host that has not been onboarded to a tenant.

## Why a bootc build needs anything special

On a conventional host, `dnf install mdatp` runs the package's `%post` scriptlet against a live system: systemd is running, SELinux policy is loaded, and the scriptlet can start the daemon and confirm it came up. The scriptlet does a substantial amount of setup, not just file placement.

A container image build has none of that. There is no systemd to talk to, no SELinux enforcing, and no daemon to start. The scriptlet runs as far as it can and then exits 1:

```
Running scriptlet: mdatp-101.26081.0010-1.x86_64
warning: %post(mdatp-101.26081.0010-1.x86_64) scriptlet failed, exit status 1
```

`dnf` itself still returns 0 and the package payload is complete and correct on disk, so the build succeeds and nothing obviously looks wrong. What is missing is the work the scriptlet did not get to. These Containerfiles do that work explicitly.

Worth stating clearly, because it is the thing people assume and it is not true here: **relocating writable state out of `/opt` is not required.** Current Defender already keeps its writable state in `/var/opt/microsoft/mdatp`, its logs in `/var/log/microsoft/mdatp`, and its configuration in `/etc/opt/microsoft/mdatp`, with only static payload in `/opt/microsoft/mdatp`. That layout is already correct for a read-only root filesystem.

### A note on `/opt`

On a deployed bootc host, `/opt` is a real directory inside the read-only deployment. It is not a symlink to `/var/opt`. That symlink is the older rpm-ostree model, and while the CentOS Stream bootc base images still ship the tmpfiles dropin that creates `/var/opt`, content installed to `/opt` during the image build stays in `/opt` and is read-only at runtime:

```
$ touch /opt/microsoft/mdatp/WRITETEST
touch: cannot touch '/opt/microsoft/mdatp/WRITETEST': Read-only file system
```

This matters for any agent, not just Defender. If software writes under `/opt` at runtime, nothing rescues it automatically.

## The four workarounds

### 1. Create the `mdatp` service account at build time

This is the least obvious of the four and the most likely to cause trouble, so it gets the longest explanation.

The package's `%pre` scriptlet is supposed to create a system account named `mdatp` with `useradd`. In a bootc container build the account does not end up existing, and `rpm` reports no `%pre` failure. See [Open questions](#open-questions) for what is and is not understood about that.

The consequence is indirect. Part of Defender's install-time setup checks whether the `mdatp` user can traverse `/var`, `/var/log`, and `/var/log/microsoft`, using `runuser` to test as that user. With no such user, the check cannot succeed:

```
runuser: user mdatp does not exist or the user entry does not contain all the required fields
Exec permission missing for /var directory
Log directory permissions insufficient, switching to opt path
```

Defender then concludes that `/var` is unsuitable and relocates its log directory to `/opt/microsoft/mdatp/log`. On a conventional host that is merely unusual. On a bootc host `/opt` is read-only, so the agent has nowhere to write its logs, and the failure surfaces later and somewhere unrelated to its cause.

The fix is a `sysusers.d` entry applied at build time, which also re-asserts the account on every boot:

```dockerfile
RUN printf 'u mdatp 972 "Microsoft Defender for Endpoint" - /sbin/nologin\n' \
      > /usr/lib/sysusers.d/mdatp.conf && \
    systemd-sysusers && \
    id mdatp
```

The fixed UID is deliberate. Microsoft's prerequisites page notes that Defender otherwise creates the account with random UID and GID values and suggests pre-creating it if you want to control them, which for a reproducible image build you do.

### 2. Install the SELinux policy modules

The package ships two compiled policy modules at `/opt/microsoft/mdatp/conf/selinux_policies/out/`. The `%post` scriptlet installs them with `semodule -i`, but only after checking `sestatus`, which in a container build reports SELinux as not enabled, so the step is skipped entirely:

```
Installing MDATP SELinux policy
WARNING: SELinux not properly configured or not running. Skipping cleanup.
```

`semodule -i` does work during an image build, writing into the policy store under `/etc/selinux/targeted/`, so the modules can simply be installed explicitly:

```dockerfile
RUN semodule -i /opt/microsoft/mdatp/conf/selinux_policies/out/mdatp_service.pp && \
    semodule -i /opt/microsoft/mdatp/conf/selinux_policies/out/audisp_mdatp_rhel77.pp && \
    semodule -l | grep -q mdatp_service
```

It prints a harmless warning about a non-atomic rename across devices, which is a consequence of the container's overlay filesystem and not a failure.

### 3. Seed the antimalware definitions into `/var`

`%post` copies the shipped engine and signature files from `/opt/microsoft/mdatp/definitions` into `/var/opt/microsoft/mdatp/definitions.noindex/00000000-0000-0000-0000-000000000000/`, which is where the running daemon looks for them. Without that copy, the daemon starts with no engine.

This is done as a systemd unit rather than a build step, for a bootc-specific reason: content an image places in `/var` is applied when the host is installed and not on subsequent upgrades, so a build-time copy would go stale the first time the image shipped a newer engine. The unit is ordered before `mdatp.service`, is conditioned on the target being absent, and copies only if needed.

A companion `tmpfiles.d` entry recreates the writable directory skeleton on every boot:

```
d /var/opt/microsoft 0755 root root -
d /var/opt/microsoft/mdatp 0755 root root -
d /var/opt/microsoft/mdatp/wdavdiag 0755 root root -
d /var/opt/microsoft/mdatp/crash 0755 root root -
d /var/log/microsoft 0755 root root -
d /var/log/microsoft/mdatp 0775 root mdatp -
```

### 4. Create the CLI symlink and enable the units

`%post` writes `mdatp.service`, `mde_netfilter_v2.service`, and `mde_netfilter_v2.socket` into `/usr/lib/systemd/system` before it fails, so the unit files are present. What is missing is the enablement, which goes through systemd, and the `/usr/bin/mdatp` command-line symlink.

Without the symlink the daemon runs fine but there is no way to query it:

```
$ mdatp health
bash: mdatp: command not found
```

Microsoft's installation troubleshooting page documents this exact symlink as the fix:

```dockerfile
RUN ln -sf /opt/microsoft/mdatp/sbin/wdavdaemonclient /usr/bin/mdatp

RUN systemctl enable mdatp.service && \
    systemctl enable mdatp-seed-definitions.service && \
    systemctl enable mde_netfilter_v2.socket
```

### A fifth item, not strictly a workaround

A drop-in orders `mdatp.service` after `cloud-init.service` and `systemd-hostnamed.service`. Microsoft's golden image guidance explains why: Defender reads the hostname when the service starts and keeps reporting the device under that name until the service is restarted, and on a machine created from an image the final hostname is often set at first boot by a provisioning service running concurrently.

## Onboarding

**Onboarding requires your own tenant's package and is not included here.** Nothing in this repository is tenant-specific, and nothing here will connect a host to a Defender tenant.

Step 6 of each Containerfile is a placeholder. To onboard, download the onboarding package from the Microsoft Defender portal under Settings, Endpoints, Device management, Onboarding, selecting Linux Server and a configuration-management deployment method. Extract `mdatp_onboard.json` from it and replace the placeholder `RUN` with:

```dockerfile
COPY mdatp_onboard.json /etc/opt/microsoft/mdatp/mdatp_onboard.json
```

Two consequences worth being deliberate about. The onboarding file is tenant-specific, so an image carrying it is tenant-specific too, and should be stored and distributed accordingly. And Microsoft's golden image guidance is explicit that the file must be placed while the Defender service is **not running**, because a running service onboards immediately and bakes device identity state into the image, which then clones across every machine built from it. A container build satisfies that by construction: nothing starts during a build, so nothing enrolls.

Microsoft documents device identity as deriving from the hardware UUID reported by `dmidecode`, which cloud platforms supply uniquely per instance and which on-premises hypervisors must be configured not to copy when cloning.

## Build and boot test

What follows is what was actually run, on a Fedora 44 host with `podman`, `bcvk`, and `qemu`.

### Build

```bash
podman build -f Containerfile.el9 -t mdatp-bootc:el9 .
podman build -f Containerfile.el10 -t mdatp-bootc:el10 .
```

The `%post` warning described above appears during the build and is expected.

### Install to a disk image and boot it

This is a real `bootc install`, which is what makes the test meaningful. Running the container image directly under `podman` or as an ephemeral VM will not give you a read-only root or enforcing SELinux, and will not tell you what you need to know.

```bash
bcvk to-disk --filesystem xfs --disk-size 20G --karg console=ttyS0 \
  --target-transport containers-storage localhost/mdatp-bootc:el9 el9.raw

qemu-system-x86_64 -machine q35,accel=kvm -cpu host -m 4096 -smp 4 \
  -nographic -serial mon:stdio -display none \
  -bios /usr/share/edk2/ovmf/OVMF_CODE.fd \
  -drive file=el9.raw,format=raw,if=virtio \
  -netdev user,id=n0 -device virtio-net-pci,netdev=n0
```

The `console=ttyS0` kernel argument is needed for the boot to appear on the serial console.

### Checks on the booted host

```bash
sestatus | grep "Current mode"            # expect: enforcing
semodule -l | grep mdatp                  # expect: audisp_mdatp_rhel77, mdatp_service
systemctl is-active mdatp                 # expect: active
id mdatp                                  # expect: the account exists
touch /opt/microsoft/mdatp/WRITETEST      # expect: Read-only file system
ls /var/log/microsoft/mdatp               # expect: logs here, not under /opt
ausearch -m avc -ts boot | grep -c denied # expect: 0
mdatp health
```

An unlicensed `mdatp health` passing looks like this, which is verbatim what both images produced:

```
ATTENTION: No license found. Contact your administrator for help.
healthy                       : false
health_issues                 : ["missing license"]
licensed                      : false
app_version                   : "101.26081.0010"
engine_version                : "1.1.26060.7001"
real_time_protection_available: true
real_time_protection_subsystem: "fanotify"
supplementary_events_subsystem: "ebpf"
definitions_version           : "1.451.328.0"
definitions_status            : "up_to_date"
conflicting_applications      : []
```

`healthy: false` with `missing license` as the only health issue is the expected unonboarded result. With a tenant onboarding file in place, the same command should show a non-empty `org_id`, `licensed: true`, and `healthy: true`.

### Automated selftest

The `selftest/` directory contains a harness that automates the checks above. It layers a oneshot unit onto either image that waits for the agent to settle, runs the checks with output on the console, and powers the machine off, so the whole verification is one boot with no interactive login.

```bash
podman build -f selftest/Containerfile.el9-selftest -t mdatp-bootc:el9-selftest .
bcvk to-disk --filesystem xfs --disk-size 20G --karg console=ttyS0 \
  --target-transport containers-storage localhost/mdatp-bootc:el9-selftest el9.raw
qemu-system-x86_64 -machine q35,accel=kvm -cpu host -m 4096 -smp 4 \
  -nographic -serial mon:stdio -display none \
  -bios /usr/share/edk2/ovmf/OVMF_CODE.fd \
  -drive file=el9.raw,format=raw,if=virtio \
  -netdev user,id=n0 -device virtio-net-pci,netdev=n0 \
  | sed -n '/MDATP-SELFTEST-BEGIN/,/MDATP-SELFTEST-END/p'
```

The selftest images are for verification only. Do not deploy them.

## Verified

Observed on a host installed with `bootc install` and booted, on both CentOS Stream 9 and CentOS Stream 10:

- The agent builds, installs, and runs. `systemctl is-active mdatp` returns `active`.
- SELinux enforcing, both policy modules loaded, zero AVC denials since boot.
- The root filesystem is read-only and `/opt/microsoft/mdatp` is not writable.
- Logs are written to `/var/log/microsoft/mdatp`, not relocated into `/opt`.
- Definitions are present in `/var` and updated from Microsoft over the network after boot.
- `mdatp health` runs and reports the expected unlicensed state.
- Both the el9 and el10 Microsoft repositories served the same package build.

## Not verified

- **Anything on RHEL.** Only CentOS Stream was built. The kernel versions differ, which matters for the eBPF sensor because Microsoft documents per-distribution minimum kernels, and the `selinux-policy` base version differs, which matters because the Defender policy modules are compiled against a policy version. Neither is likely to change the outcome, but a RHEL rebuild is the right thing to do before relying on this.
- **Anything requiring a license or a tenant.** Onboarding, active-mode real-time protection, the EICAR antivirus test, the EDR detection test, and telemetry reaching the Defender portal. The test hosts came up `passive_mode_enabled: true` by default in the absence of a license.
- **Cloning.** One image booted as two instances should appear as two devices in the portal. This follows from Microsoft's documented hardware-UUID behavior and from nothing enrolling at build time, but it was reasoned rather than observed.
- **Upgrade and rollback.** `bootc upgrade` to a rebuilt image and `bootc rollback` back from it were not tested. Note that `/var` does not roll back, so an older agent would inherit state written by a newer one.
- **Secure Boot and sealed images.** The test installs used XFS and the ordinary ostree backend with composefs enabled. Nothing in the package suggests a conflict, since there is no kernel module and eBPF loading by a root daemon is not restricted by kernel lockdown, but this was not tested with Secure Boot enabled.
- **The hostname ordering drop-in.** Correct in form, but the test VMs had no `cloud-init`, so the race it guards against was never exercised.

## Open questions

### Why `%pre` does not create the service account

This is the one genuinely unexplained result, and it is recorded here rather than papered over because it is probably the most useful detail in this repository for anyone maintaining the package.

The `%pre` scriptlet calls its `CreateUser` function inside a `set -e` block. That function runs:

```bash
useradd --system --no-create-home --user-group --shell /sbin/nologin mdatp
```

Run by hand in the same `centos-bootc:stream9` base image, that command succeeds and creates `mdatp` with a home directory of `/var/home/mdatp`. After `dnf -y install mdatp` in a build of that same image, `/etc/passwd` contains only `root` and the account is absent, yet `rpm` reports no `%pre` failure, only the `%post` one. Something causes `CreateUser` either not to run or to return success without creating the account, and that was not isolated.

Creating the account explicitly, as step 1 of these Containerfiles does, makes the question moot for anyone using this recipe. It does not explain the behavior.

### Microsoft support stance

Whether a bootc-built host counts as a supported Defender configuration is an open question, and it is not one that can be answered from outside Microsoft. Two pieces of published material bear on it:

- [mdatp-xplat issue #240](https://github.com/microsoft/mdatp-xplat/issues/240), "Bug: Install fails under Atomic Fedora (Silverblue, Kinoite, etc)", filed 2026-07-03. A Microsoft contributor replied on 2026-09-18: "Atomic Fedora variants such as Silverblue/Kinoite/rpm-ostree are not currently supported. Microsoft Defender for Endpoint supports standard Fedora releases listed in the public prerequisites, but not the immutable rpm-ostree install model used by these variants. Closing this as unsupported." The reported failure is in `rpm-ostree install mdatp`, which layers a package onto an already-deployed host and runs scriptlets in a sandbox against a read-only `/var`. That is a different operation from installing the package into a container image during a build, which is what this repository does and where `/var` is writable.
- The [prerequisites page](https://learn.microsoft.com/en-us/defender-endpoint/mde-linux-prerequisites) states that Defender "can be installed and may function on customized operating systems that meet minimal kernel requirements and are derived from known, standard, vendor-provided Linux distributions that Microsoft supports... However, these customized environments aren't part of Microsoft's validated or maintained support baseline. As a result, they're treated as custom OS configurations from a support perspective. Customers are expected to validate Defender for Endpoint within these custom environments and, if needed, reproduce issues on a supported, standard (unmodified) Linux distribution."

Where an image mode host built from a vendor bootc base image, carrying the stock package from Microsoft's own repository, falls between those two statements is a question for Microsoft. It is noted here as open.

## References

Microsoft documentation, which is the source for every claim in this README about the product's behavior that was not directly observed:

- [Prerequisites for Defender for Endpoint on Linux](https://learn.microsoft.com/en-us/defender-endpoint/mde-linux-prerequisites)
- [Deploy Microsoft Defender for Endpoint on Linux manually](https://learn.microsoft.com/en-us/defender-endpoint/linux-install-manually)
- [Deploy using golden images](https://learn.microsoft.com/en-us/defender-endpoint/linux-deploy-defender-for-endpoint-using-golden-images)
- [Enabling deployment to a custom location](https://learn.microsoft.com/en-us/defender-endpoint/linux-custom-location-installation)
- [Defender for Endpoint on Linux resources](https://learn.microsoft.com/en-us/defender-endpoint/linux-resources)
- [Deploy updates](https://learn.microsoft.com/en-us/defender-endpoint/linux-updates)
- [Use eBPF-based sensor](https://learn.microsoft.com/en-us/defender-endpoint/linux-support-ebpf)
- [Troubleshoot installation issues](https://learn.microsoft.com/en-us/defender-endpoint/linux-support-install)
- [Configure security settings](https://learn.microsoft.com/en-us/defender-endpoint/linux-preferences)

bootc and image mode:

- [bootc documentation](https://bootc-dev.github.io/bootc/)
- [Using image mode for RHEL to build, deploy, and manage operating systems](https://docs.redhat.com/)
- [bcvk](https://github.com/bootc-dev/bcvk), used for the install-to-disk step

Microsoft, Microsoft Defender, and Defender for Endpoint are trademarks of Microsoft Corporation. Red Hat and Red Hat Enterprise Linux are trademarks of Red Hat, Inc. This repository is not affiliated with or endorsed by either.
