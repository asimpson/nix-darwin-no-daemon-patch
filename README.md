# nix-darwin-no-daemon-patch

A tiny repo for maintaining `no-daemon.patch`, a patch against the official Nix
macOS binary installer. The patch allows the Darwin installer to take the
single-user `--no-daemon` path instead of forcing the daemon install path.

Current known-good target:

```text
Nix:    2.34.6
System: aarch64-darwin
URL:    https://releases.nixos.org/nix/nix-2.34.6/nix-2.34.6-aarch64-darwin.tar.xz
SHA256: 596ae5555acfb497723934b98b66b7d638eb8f7d856975466f2c1217ce94f8a1
```

## What the patch does

`no-daemon.patch` modifies the upstream tarball's `install` script to:

- stop defaulting Darwin/macOS to `INSTALL_MODE=daemon`
- remove the Darwin/macOS error for `--no-daemon`

Nothing else in the Nix distribution is modified.

## Use

Clone this repo, then run the commands below from the repo root:

```sh
version=2.34.6
system=aarch64-darwin
url="https://releases.nixos.org/nix/nix-$version/nix-$version-$system.tar.xz"
tarball="nix-$version-$system.tar.xz"

curl -fL "$url" -o "$tarball"

expected="$(curl -fsSL "$url.sha256" | awk '{print $1}')"
actual="$(shasum -a 256 "$tarball" | awk '{print $1}')"
test "$expected" = "$actual"

tar -xf "$tarball"
patch -F 0 -d "nix-$version-$system" -p1 < no-daemon.patch

./"nix-$version-$system"/install --no-daemon
```

For Intel macOS, use `system=x86_64-darwin` and the matching upstream tarball.

## Check whether the patch applies

```sh
version=2.34.6
system=aarch64-darwin
url="https://releases.nixos.org/nix/nix-$version/nix-$version-$system.tar.xz"

curl -fL "$url" -o nix.tar.xz
mkdir -p check
tar -xf nix.tar.xz -C check
patch --dry-run -F 0 -d "check/nix-$version-$system" -p1 < no-daemon.patch
```

## GitHub Actions

`.github/workflows/check-patch.yml` runs daily and checks that `no-daemon.patch`
applies with zero fuzz to the latest published upstream Nix macOS tarball.

To avoid downloading and checking the same release every day, the workflow stores
a tiny success marker in the GitHub Actions cache. The cache key includes:

- the Nix version
- the Nix system, currently `aarch64-darwin`
- the upstream tarball SHA-256
- the SHA-256 of `no-daemon.patch`

If the exact latest tarball and patch are unchanged, the workflow skips the
tarball download and patch check. A new upstream release, tarball checksum change,
or patch change gets checked again. Failed checks are not cached, so they keep
failing daily until the patch is refreshed.

When the patch no longer applies, the workflow fails and opens a GitHub issue in
this repo. The issue stays open as the reminder to refresh `no-daemon.patch`. Once
the check passes again, the workflow comments on and closes that issue.

## Notes

- This intentionally follows an install mode that upstream Nix no longer
  supports on macOS. Use at your own risk.
- The manual commands above require `curl`, `tar`, `patch`, and `shasum`.
* Generate a new patch by running: `git diff --relative install > no-daemon.patch` from inside the release directory.
