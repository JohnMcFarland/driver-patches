# driver-patches

Out-of-tree patches and fixes for Linux kernel device drivers. Each
subdirectory is a self-contained fix: the `.patch` plus a README with the root
cause, build steps, and usage.

## Patches

| Patch | Driver | Issue |
|-------|--------|-------|
| [rt2x00-rt3070-monitor-channel-drift](rt2x00-rt3070-monitor-channel-drift/) | rt2x00 / rt2800usb — Ralink RT3070 (`148f:3070`) | In monitor mode the radio will not hold a fixed channel; it drifts to nearby APs' channels within seconds, breaking simultaneous `airodump-ng` capture and `aireplay-ng` injection. Adds an opt-in `monitor_channel` module parameter that pins the channel. |

## Building a patch

Each patch applies to a matching kernel source tree and builds the affected
driver out-of-tree against your installed kernel:

```bash
cd /path/to/linux-source          # matching uname -r, or the mainline tree
patch -p1 < <patch-dir>/<name>.patch
make -C /lib/modules/$(uname -r)/build M=$(pwd)/<driver/source/dir> modules
```

See each subdirectory's README for the exact commands, module load order, and
usage.

## License

These patches modify Linux kernel source and are offered under the same license
as the code they modify (GPL-2.0-or-later for the rt2x00 driver). They are
provided as-is, without warranty, and are workarounds rather than upstream-merged
fixes. Test before relying on them.
