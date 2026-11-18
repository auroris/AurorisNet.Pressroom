# Deployment examples

The files in this folder run Pressroom as a container. Each release includes the folder, and
[Containers](../docs/container.md) gives the procedure for each file.

| File | Use |
|---|---|
| `compose.yaml` | One Linux container, published on `127.0.0.1:8080`, with a read-only root file system and temporary state. Set `PRESSROOM_VERSION` and create `pressroom.settings.json` beside the file before you start it. |
| `compose.host-isolated.yaml` | An override for `compose.yaml` that attaches the container to the `pressroom-egress-blocked` network. Apply the iptables rules in [Containers](../docs/container.md#optional-linux-outbound-isolation) first. |
| `kubernetes.yaml` | A Deployment with one replica, memory-backed state, a settings Secret, and a NetworkPolicy that allows TCP 8080 in and nothing out. The cluster's network plugin must enforce NetworkPolicy. |
| `pressroom-seccomp.json` | The seccomp profile for the Linux container. |
| `check.settings.json` | Anonymous settings for an offline `--check` of an image. **Do not deploy it as a production configuration.** |
| `Start-WindowsContainer.ps1` | Starts the Windows container. `-BlockOutbound` first runs `Install-WindowsContainerPolicy.ps1`. |
| `Install-WindowsContainerPolicy.ps1` | Creates the `pressroom-egress-blocked` Windows NAT network and its HNS rules. Needs an administrator session. |
| `LICENSE.moby` | The Apache-2.0 license of the profile that `pressroom-seccomp.json` derives from. |

## The seccomp profile

`pressroom-seccomp.json` derives from Docker's default profile at
[moby/profiles 4b8214f](https://github.com/moby/profiles/blob/4b8214fc4971e06172938a1f1554ccff6647ee80/seccomp/default.json)
(Apache-2.0). It adds one rule that allows these system calls without a host capability:

- `clone`, `clone3`, `unshare` and `setns`, which Chromium's sandbox uses to create user
  namespaces, as [Playwright describes](https://playwright.dev/docs/docker);
- `chroot`, which Chromium calls inside its own user namespace. Docker's default rule allows
  `chroot` only with the `CAP_SYS_CHROOT` host capability, which `cap_drop: [ALL]` removes;
- `pidfd_open` and `pidfd_send_signal`, which Pressroom uses to signal its own browser processes
  without reaching a process that reused the same ID.

The profile's default action still denies, and the rest of Docker's restrictions stay in place.
Use the profile with the non-root container and all host capabilities dropped, as the Compose
file does.

See also [Linux](../docs/linux.md).
