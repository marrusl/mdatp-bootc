# Testing

How the claims in the main [README](README.md) were produced, and how to re-run or extend them. This is working material for an engineer touching the recipe. The results themselves are in the README under Verified and do not require reading any of this.

Two harnesses:

- **`selftest/`** checks one host, installed fresh and booted once.
- **`lifecycle-test/`** checks what happens across deployment changes: adopting the agent by switching an existing host onto the image, upgrading, rolling back, and removing it by switching away.

## Prerequisites

None of this applies to a host actually running Defender, which needs nothing beyond what the image carries. Building the image needs only `podman`; everything below is for testing it.

What the test requires, independent of tooling: a real `bootc install` to a disk image, booted as a VM under UEFI, with a read-only root and SELinux enforcing. A `podman run` of the image, or an ephemeral container-as-VM, gives you none of those and will not exercise anything in this recipe. Any virtualization platform that preserves those properties works.

| Tool | Version used | What it is for |
|---|---|---|
| `podman` | 5.8.7 | Building the images |
| `bcvk` | 0.21.0 | Installing a bootc image to a disk image, which is a real `bootc install` |
| `qemu-system-x86_64` with KVM | 10.2.2 | Booting the resulting disk |
| `edk2-ovmf` | 20260812 | UEFI firmware, `/usr/share/edk2/ovmf/OVMF_CODE.fd` |

`bcvk` is not a requirement. It wraps `bootc install to-disk` in one command that needs no root and no libvirt, which is why it is used here: a single non-interactive command means the verification can live in a build pipeline rather than being done by hand once. Substituting another virt platform is fine, but the scripts under `selftest/` and `lifecycle-test/` are written against `bcvk`, so it means replacing the install and boot steps rather than swapping a command name. Everything after the boot runs on the host and transfers unchanged.

Without `/dev/kvm`, falling back to emulation is slow enough that the 40 second settle in the harnesses stops being long enough.

`lifecycle-test/` has one prerequisite `selftest/` does not: a container registry the guest VM can reach, because `bootc switch` and `bootc upgrade` resolve real image references.

### Getting `bcvk`

Upstream at [github.com/bootc-dev/bcvk](https://github.com/bootc-dev/bcvk), Apache-2.0 or MIT. The package is called `bcvk` everywhere.

On Fedora it is in the main repositories, no EPEL involved. Fedora 43, 44, and 45 carry 0.21.0, the version used here:

```bash
sudo dnf install bcvk
```

On RHEL it comes from EPEL, which is not enabled by default and which wants CodeReady Builder enabled first on a subscribed host. For RHEL 9:

```bash
sudo subscription-manager repos --enable codeready-builder-for-rhel-9-$(arch)-rpms
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
sudo dnf install bcvk
```

Substitute 10 in the first two commands for RHEL 10. Both EPEL 9 and EPEL 10 carry it, so there is no parity gap between the majors.

Two things to know first. EPEL is community-maintained and not covered by a Red Hat support agreement, which applies to the test host only. And both EPEL branches ship **0.10.0**, from January 2026, against Fedora's 0.21.0 from October. Every `bcvk` flag used in this repo, `--filesystem`, `--disk-size`, `--karg`, and `--target-transport`, is present in 0.10.0 with the same name and meaning, so the commands here work on EPEL as written. That was checked against the `v0.10.0` tag in the upstream source rather than inferred from the release notes.

## Single-host checks

Build the image as described in the README, then install it to a disk and boot it. This is a real `bootc install`, which is what makes the result meaningful.

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

On the booted host:

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

`healthy: false` with `missing license` as the only health issue is the expected unonboarded result. With a tenant onboarding file in place, the same command should show a non-empty `org_id`, `licensed: true`, and `healthy: true`. The full expected output is in the README under Verified.

### Automated

`selftest/` layers a oneshot unit onto either image that waits for the agent to settle, runs the checks above with output on the console, and powers the machine off, so the whole thing is one boot with no interactive login. The build context is `selftest/`, not the repo root, since that is where the script it copies lives.

```bash
podman build -f selftest/Containerfile.el9-selftest -t mdatp-bootc:el9-selftest selftest/
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

## Lifecycle checks

These cover the cases where a quietly wrong behavior would live, because content an image places in `/var` is applied when the host is installed and **not** on any subsequent deployment change. A host that arrives by `bootc switch` gets nothing in `/var` from the image; a host that upgrades keeps the `/var` it already had.

### How it works

A bootc host changes deployments by rebooting, so a test that spans several deployments cannot be a single script run. `lifecycle-test/mdatp-lifecycle-test` is invoked once per boot by a systemd unit. Each run reports the state of the host to the console, advances a stage counter kept in `/var/lib/mdatp-lifecycle-test/stage`, performs the next `bootc` action, and reboots. The counter lives in `/var`, so it survives the deployment changes and the sequence continues across reboots and across separate qemu invocations against the same disk. Everything goes to the serial console, so no SSH key and no login are needed.

| Stage | Host state when it reports | Action it then takes |
|---|---|---|
| 0 | Vanilla bootc base, no Defender, empty `/var` | `bootc switch` to the mdatp image |
| 1 | First boot after adopting the agent | Power off, end of run 1 |
| 2 | Same deployment, plain reboot | `bootc upgrade` to the second build |
| 3 | First boot on the upgraded deployment | `bootc rollback` |
| 4 | Back on the previous deployment | `bootc switch` to the vanilla base |
| 5 | Agent removed, its `/var` state left behind | Power off, end of run 2 |

Each report covers the booted image and digest, whether the agent is in the image, the service account, the writable state directories with their SELinux labels, the definitions directory with mtimes and checksums, the definition paths the agent is actually using from `wdavstate`, the seeding unit's condition result and journal, `mdatp.service` state and restart count, the log directory and whether the read-only `/opt` fallback was triggered, SELinux mode and loaded policy modules, AVC denials, `mdatp health`, and any failed units.

### Running it

Set `REGISTRY` to your host's own LAN address and start a registry. It must be reachable from both the host and the guest, which rules out `localhost`: under qemu user-mode networking the guest reaches the host's LAN address.

```bash
export REGISTRY=10.0.0.1:5000        # your host's LAN address, not localhost
podman run -d --name lifecycle-registry -p 5000:5000 docker.io/library/registry:2
```

Build the recipe image, then the two harness images, and push them:

```bash
podman build -f Containerfile.el9 -t mdatp-bootc:el9 .
cd lifecycle-test
podman build --build-arg REGISTRY=$REGISTRY -f Containerfile.testbase  -t $REGISTRY/testbase:el9 .
podman build --build-arg REGISTRY=$REGISTRY -f Containerfile.undertest -t $REGISTRY/mdatp:el9 .
podman push --tls-verify=false $REGISTRY/testbase:el9
podman push --tls-verify=false $REGISTRY/mdatp:el9
```

Install the vanilla base and boot it. This is run 1, stages 0 and 1:

```bash
bcvk to-disk --filesystem xfs --disk-size 25G --karg console=ttyS0 \
  $REGISTRY/testbase:el9 lifecycle.raw

qemu-system-x86_64 -machine q35,accel=kvm -cpu host -m 4096 -smp 4 \
  -nographic -serial mon:stdio -display none \
  -bios /usr/share/edk2/ovmf/OVMF_CODE.fd \
  -drive file=lifecycle.raw,format=raw,if=virtio \
  -netdev user,id=n0 -device virtio-net-pci,netdev=n0 | tee run1.log
```

The VM powers itself off at the end of stage 1. Push a second build over the same tag so `bootc upgrade` has somewhere to go:

```bash
podman build --build-arg REGISTRY=$REGISTRY --build-arg BUILD_MARKER=2 \
  -f Containerfile.undertest -t $REGISTRY/mdatp:el9 .
podman push --tls-verify=false $REGISTRY/mdatp:el9
```

Boot the same disk again for run 2, stages 2 through 5, then read the reports out of the console logs and tear the registry down:

```bash
qemu-system-x86_64 ... -drive file=lifecycle.raw,... | tee run2.log

sed -n '/LIFECYCLE-STAGE-1-BEGIN/,/LIFECYCLE-STAGE-1-END/p' run1.log
podman rm -f lifecycle-registry
```

### What to look at

In rough order of how much it would matter if it were wrong:

- **Stage 1, the log directory.** It must be `/var/log/microsoft/mdatp`, and `/opt/microsoft/mdatp/log` must not exist. This is the case where `/var` is least prepared, so it is where the installer's fallback into the read-only install tree is most likely to trigger.
- **Stages 3 and 4, the seeded file mtimes and checksums.** They must be unchanged from stage 1. If they move, the older definitions baked into the image have overwritten the newer ones in `/var`, which is the specific failure the design exists to avoid.
- **Stages 2 through 4, the seeding unit's `ConditionResult`.** It must be `no`, with the journal line saying the unit was skipped because of an unmet condition check.
- **Every stage, AVC denials and failed units.** Both must be zero.
- **Every stage with the agent present, health.** `health_issues` should contain `missing license` and nothing else, unless you supplied a tenant onboarding package.

### Not covered

Re-adoption: switching away from the agent and later switching back onto it, so that first boot meets a `/var` holding Defender state that may be months old. That is a different question from an empty `/var` or a current one. Adding it is one more entry in the driver's case statement.

Lifecycle testing has been run on el9 only, not el10.
