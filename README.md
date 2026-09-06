# amneziawg-tools-rpm

RPM packaging of [amneziawg-tools](https://github.com/amnezia-vpn/amneziawg-tools)
for EL9 and EL10, built in GitHub Actions with mock. Produces one package,
`amneziawg-tools`: `awg`, `awg-quick`, `awg-quick@.service`, man pages and bash
completion.

This is the upstream wireguard-tools tree with every command renamed — `awg`
for `wg`, `/etc/amnezia/amneziawg` for `/etc/wireguard` — so it installs
alongside `wireguard-tools` without conflicting.

The kernel module is packaged separately in
[amneziawg-dkms-rpm](https://github.com/blennuria/amneziawg-dkms-rpm); the two
upstreams have their own version streams, which is why they are not packaged
together.

## Install

```shell
sudo dnf install -y https://github.com/blennuria/amneziawg-tools-rpm/releases/latest/download/amneziawg-tools-3.1.20260812-1.el9.x86_64.rpm
```

`amneziawg-dkms` is a weak dependency, so a plain `dnf install` pulls the kernel
module too (that needs EPEL for `dkms`). Skip it with
`--setopt=install_weak_deps=False` on machines using a userspace implementation
via `WG_QUICK_USERSPACE_IMPLEMENTATION`.

### Usage

Same as `wg`/`wg-quick`, with the AmneziaWG obfuscation knobs (`Jc`, `Jmin`,
`Jmax`, `S1`–`S4`, `H1`–`H4`, `I1`–`I5`) in the `[Interface]` section:

```shell
sudo awg genkey | sudo tee /etc/amnezia/amneziawg/privatekey | awg pubkey
sudo $EDITOR /etc/amnezia/amneziawg/awg0.conf
sudo systemctl enable --now awg-quick@awg0
```

## Notes on the spec

- **LTO is off.** EL9's make is 4.3, which passes its jobserver as file
  descriptors and closes them for any recipe it does not recognise as a
  recursive make. The link step is such a recipe, and `-flto=auto` makes gcc
  spawn a make of its own inside it, which dies with `write jobserver: Bad file
  descriptor`. EL10 (make 4.4, fifo jobserver) is unaffected, but a ~100 KB CLI
  tool gains nothing from LTO, so it is dropped on both rather than branching.
- No systemd scriptlets: a template unit and a target need neither enabling nor
  a daemon-reload.
- `/etc/amnezia/amneziawg` is 0700 — private keys live there — and `%attr` pins
  it rather than trusting the umask the upstream Makefile installs under.

## Updating

Bump `Version:`, add a `%changelog` entry, push a tag. Nothing else references
the version — the workflow takes the source URL out of the spec.

Tags are `<version>-<release>` (e.g. `3.1.20260812-1`), and the build refuses to
publish when the tag and the spec disagree.

Upstream releases the kernel module more often than the tools — they share a tag
date only on joint release days — so the two packages are normally on different
versions. That is fine as long as they agree on the netlink API; if the module
grows an attribute these tools do not know, `awg` silently cannot configure it.
When bumping either side, check:

```shell
kmod=3.1.20260906; tools=3.1.20260812
attrs() { grep -oE 'WG(DEVICE|PEER)_A_[A-Z0-9_]+' - \
          | sed 's/^WGPEER_A_AWG$/WGPEER_A_ADVANCED_SECURITY/' | sort -u; }
diff <(curl -fsSL https://raw.githubusercontent.com/amnezia-vpn/amneziawg-linux-kernel-module/v$kmod/src/uapi/wireguard.h | attrs) \
     <(curl -fsSL https://raw.githubusercontent.com/amnezia-vpn/amneziawg-tools/v$tools/src/uapi/linux/linux/wireguard.h | attrs)
```

Empty output means the pair is interchangeable with a joint upstream release.
(The two repos spell attribute 11 differently — `WGPEER_A_ADVANCED_SECURITY`
versus `WGPEER_A_AWG` — for the same value, hence the `sed`.)

## Testing locally

The CI jobs are reproducible in a container — `podman`, `docker` or Apple's
`container`, with `--arch amd64` where the host is not x86_64:

```shell
container run --rm --arch amd64 -v "$PWD":/work quay.io/centos/centos:stream9 bash -c '
  dnf install -y --nogpgcheck rpm-build rpmdevtools gcc make systemd-rpm-macros
  rpmdev-setuptree && cp /work/*.spec ~/rpmbuild/SPECS/
  spectool -g -C ~/rpmbuild/SOURCES ~/rpmbuild/SPECS/amneziawg-tools.spec
  rpmbuild -ba ~/rpmbuild/SPECS/amneziawg-tools.spec
  dnf install -y ~/rpmbuild/RPMS/x86_64/amneziawg-tools-[0-9]*.rpm
  awg --version && awg genkey | awg pubkey'
```

## CI

`.github/workflows/build.yml`:

1. **build** — mock builds SRPM + RPM in a CentOS Stream 9/10 container.
2. **install-test** — installs the package on the matching EL and exercises
   `awg`, `awg-quick`, the config directory's permissions, the unit file and
   the man pages.
3. **release** — on a tag, publishes the RPMs to a GitHub release. Debug
   packages stay in the run artifacts.

Only x86_64 is built. For aarch64, add a matrix entry on an `ubuntu-24.04-arm`
runner with the `-aarch64` mock config.
