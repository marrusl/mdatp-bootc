# Lifecycle test

Working material, not product documentation. It is here so the lifecycle claims in the main README can be checked and extended by someone working on the recipe rather than only asserted.

`selftest/` covers one host, installed fresh and booted once. This harness covers what comes after: adopting the agent by switching an existing host onto the image, upgrading to a newer build, rolling back, and removing the agent by switching away. Those are the cases where a quietly wrong behavior would live, because content an image places in `/var` is applied when the host is installed and **not** on any subsequent deployment change. A host that arrives by `bootc switch` gets nothing in `/var` from the image; a host that upgrades keeps the `/var` it already had. Results are summarized in the main README under Verified.

## What it does

A bootc host changes deployments by rebooting, so a test that spans several deployments cannot be a single script run. `mdatp-lifecycle-test` is invoked once per boot by a systemd unit. Each run reports the state of the host to the console, advances a stage counter kept in `/var/lib/mdatp-lifecycle-test/stage`, performs the next `bootc` action, and reboots. The counter lives in `/var`, so it survives the deployment changes and the sequence continues across reboots and across separate qemu invocations against the same disk.

Everything goes to the serial console, so no SSH key and no login are needed.

| Stage | Host state when it reports | Action it then takes |
|---|---|---|
| 0 | Vanilla bootc base, no Defender, empty `/var` | `bootc switch` to the mdatp image |
| 1 | First boot after adopting the agent | Power off, end of run 1 |
| 2 | Same deployment, plain reboot | `bootc upgrade` to the second build |
| 3 | First boot on the upgraded deployment | `bootc rollback` |
| 4 | Back on the previous deployment | `bootc switch` to the vanilla base |
| 5 | Agent removed, its `/var` state left behind | Power off, end of run 2 |

Each report covers the booted image and digest, whether the agent is in the image, the service account, the writable state directories with their SELinux labels, the definitions directory with mtimes and checksums, the definition paths the agent is actually using from `wdavstate`, the seeding unit's condition result and journal, `mdatp.service` state and restart count, the log directory and whether the read-only `/opt` fallback was triggered, SELinux mode and loaded policy modules, AVC denials, `mdatp health`, and any failed units.

## What it needs

- The host toolchain in the main README under [Prerequisites](../README.md#prerequisites-on-the-build-and-test-host). On RHEL, `bcvk` comes from EPEL; see the note there.
- **A container registry the guest can reach**, which `selftest/` does not need. `bootc switch` and `bootc upgrade` resolve real image references, so the images have to be served from somewhere the VM can pull from. A throwaway local registry is enough, at an address that works from both host and guest: under qemu user-mode networking the guest reaches the host's LAN address, so `localhost` will not do.

## Running it

Set `REGISTRY` to your host's own LAN address and start a registry:

```bash
export REGISTRY=10.0.0.1:5000        # your host's LAN address, not localhost
podman run -d --name lifecycle-registry -p 5000:5000 docker.io/library/registry:2
```

Build the recipe image, then the two harness images, and push them:

```bash
cd ..
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

Boot the same disk again. This is run 2, stages 2 through 5:

```bash
qemu-system-x86_64 -machine q35,accel=kvm -cpu host -m 4096 -smp 4 \
  -nographic -serial mon:stdio -display none \
  -bios /usr/share/edk2/ovmf/OVMF_CODE.fd \
  -drive file=lifecycle.raw,format=raw,if=virtio \
  -netdev user,id=n0 -device virtio-net-pci,netdev=n0 | tee run2.log
```

Then read the reports out of the console logs:

```bash
sed -n '/LIFECYCLE-STAGE-1-BEGIN/,/LIFECYCLE-STAGE-1-END/p' run1.log
```

Tear the registry down when you are finished:

```bash
podman rm -f lifecycle-registry
```

## What to look at

In rough order of how much it would matter if it were wrong:

- **Stage 1, the log directory.** It must be `/var/log/microsoft/mdatp`, and `/opt/microsoft/mdatp/log` must not exist. This is the case where `/var` is least prepared, so it is where the installer's fallback into the read-only install tree is most likely to trigger.
- **Stages 3 and 4, the seeded file mtimes and checksums.** They must be unchanged from stage 1. If they move, the older definitions baked into the image have overwritten the newer ones in `/var`, which is the specific failure this design exists to avoid.
- **Stages 2 through 4, the seeding unit's `ConditionResult`.** It must be `no`, with the journal line saying the unit was skipped because of an unmet condition check.
- **Every stage, AVC denials and failed units.** Both must be zero.
- **Every stage with the agent present, health.** `health_issues` should contain `missing license` and nothing else, unless you supplied a tenant onboarding package.

## A note on scope

The harness does not cover re-adoption: switching away from the agent and later switching back onto it, so that the first boot meets a `/var` holding Defender state that may be months old. That is a different question from an empty `/var` or a current one, and it is not tested here. Adding it is one more entry in the driver's case statement.
