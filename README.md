A tiny repo for maintaining `no-daemon.patch`, a patch against the official Nix
macOS binary installer. The patch allows the Darwin installer to take the
single-user `--no-daemon` path instead of forcing the daemon install path.

Shout out to [Ian Henry for the original idea](https://ianthehenry.com/posts/how-to-learn-nix/installing-nix-on-macos/)!

## What the patch does

`no-daemon.patch` modifies the upstream tarball's `install` script to:

- stop defaulting Darwin/macOS to `INSTALL_MODE=daemon`
- remove the Darwin/macOS error for `--no-daemon`

Nothing else in the install script is modified.

## Use

1. You must have a `/nix` directory configured before running `install`.

    - Make a new volume in Disk Utility called "Nix" (`Nix` is important for the `fstab` step later).
    * Create a mount point for the volume: `echo "nix" | sudo tee -a /etc/synthetic.conf`

      `man synthetic.conf` describes what this file is for:
      > synthetic.conf is intended to be used for creating mount points
      at / (e.g. for use as NFS mount points in enterprise
      deployments) and symbolic links (e.g. for creating a package
      manager root without modifying the system volume).
      synthetic.conf is read by apfs.util(8) during early system boot.

    * Mount the volume: `sudo bash -c 'echo "LABEL=Nix /nix apfs rw,nobrowse" >> /etc/fstab'`

      Note from `man fstab` on the first field:
      > For APFS volumes, this field should never be the block special
      device as it is not constant. Only the constructs `UUID` and
      `LABEL` should be used.

    * Reboot

2. Clone this repo, then run the commands below from the repo root:

```sh
version=2.34.6
url="https://releases.nixos.org/nix/nix-$version/nix-$version-aarch64-darwin.tar.xz"
tarball="nix-$version-aarch64-darwin.tar.xz"

curl -fL "$url" -o "$tarball"

expected="$(curl -fsSL "$url.sha256" | awk '{print $1}')"
actual="$(shasum -a 256 "$tarball" | awk '{print $1}')"
test "$expected" = "$actual"

tar -xf "$tarball"
patch -F 0 -d "nix-$version-aarch64-darwin" -p1 < no-daemon.patch

./"nix-$version-aarch64-darwin"/install --no-daemon
```

## Notes

- This intentionally follows an install mode that upstream Nix no longer
  supports on macOS. Use at your own risk.
- The manual commands above require `curl`, `tar`, `patch`, and `shasum`.
* Generate a new patch by running: `git diff --relative install > no-daemon.patch` from inside the release directory.
