# rt2x00 monitor-mode channel lock — RT3070 / rt2800usb channel drift

Fixes the long-standing bug where an RT3070 (`148f:3070`, `rt2800usb`) in monitor
mode does not stay on a channel set with `iw dev <if> set channel N`, drifting to
the channels of nearby busy APs within seconds and breaking concurrent
`airodump-ng` / `aireplay-ng`.

Patch: `rt2x00-monitor-channel-lock.patch`
Marker: `CAELION-FIX-RT3070-CHANDRIFT`

## Root cause

`rt2800usb` does not implement channel contexts, so mac80211 **emulates** them and
funnels every hardware retune through the driver's single `.config` callback:

```
rt2x00mac_config()  ->  rt2x00lib_config()  ->  rt2800_config_channel()
   (rt2x00mac.c)          (rt2x00config.c)         (rt2800lib.c)
```

That callback is the *only* runtime path that retunes the radio for this chip.
So whenever mac80211 schedules off-channel work while a monitor interface is
"locked" to a fixed channel, it reaches the driver as an ordinary
`IEEE80211_CONF_CHANGE_CHANNEL` request and the radio moves. Triggers include:

- **software / background scan** hopping channels,
- **channel-context reassignment** when another vif (e.g. a leftover managed
  `wlanX` left `UP`) wants a different channel,
- **remain-on-channel** work.

The drift targets track nearby APs' channels because those are the channels the
upper layers are chasing. This is consistent with the earlier finding that the
VCO / link-tuner patch did **not** help: sub-second drift fires before any tuner
timer, because it comes through the channel-config path, not the tuner.

## The fix

The driver cannot distinguish an explicit user `set channel` from a
mac80211-initiated hop — both are the same `.config` call. So the patch adds an
opt-in module parameter, `monitor_channel`. When it holds a non-zero 802.11
channel number **and** the interface is in monitor mode, `rt2x00mac_config()`
strips `IEEE80211_CONF_CHANGE_CHANNEL` from any request whose target channel
differs from the locked one. The radio stays put. All other configuration in
that call (TX power, retry limits, PS, monitor enable/disable) is still applied.

Why it is safe:

- Gated on `IEEE80211_CONF_MONITOR`; station / AP / managed modes never set it,
  so their channel switching (including CSA) is untouched.
- Default `0` disables the feature entirely — no behaviour change for anyone.
- Link tuning and VCO calibration are not modified.
- Channel matching uses `ieee80211_frequency_to_channel(center_freq)`, correct on
  both 2.4 GHz and 5 GHz (`chan->hw_value` is only an index into `spec.channels[]`,
  not the channel number, so it is not used for matching).

Cosmetic limitation: `rt2x00lib_config()` still records `conf->chandef` into
`curr_freq`/`curr_band` unconditionally, so a refused change can briefly make the
per-packet radiotap channel annotation report the channel mac80211 *thinks* it
moved to. The radio physically stays on the locked channel for capture and
injection.

## Build (out-of-tree)

```bash
# 1. Get kernel source matching your running kernel (or the mainline tree).
cd /path/to/linux-source

# 2. Apply the patch.
patch -p1 < rt2x00-monitor-channel-lock.patch

# 3. Build just the rt2x00 driver directory against your installed kernel build.
make -C /lib/modules/$(uname -r)/build \
     M=$(pwd)/drivers/net/wireless/ralink/rt2x00 modules
```

The `monitor_channel` parameter lives in `rt2x00lib.ko` (that is the module
`rt2x00mac.o` is linked into). Load the rebuilt modules, e.g.:

```bash
sudo rmmod rt2800usb rt2800lib rt2x00usb rt2x00lib 2>/dev/null
sudo insmod ./drivers/net/wireless/ralink/rt2x00/rt2x00lib.ko
sudo insmod ./drivers/net/wireless/ralink/rt2x00/rt2x00usb.ko
sudo insmod ./drivers/net/wireless/ralink/rt2x00/rt2800lib.ko
sudo insmod ./drivers/net/wireless/ralink/rt2x00/rt2800usb.ko
```

(or copy them into `/lib/modules/$(uname -r)/...`, run `depmod -a`, and reload).

## Use

```bash
sudo airmon-ng check kill
echo 6 | sudo tee /sys/module/rt2x00lib/parameters/monitor_channel
sudo ip link set wlan0 down
sudo iw dev wlan0 set monitor control
sudo ip link set wlan0 up
sudo iw dev wlan0 set channel 6          # sticks on 6 now
# move the lock: set the parameter to N, then re-issue "iw ... set channel N"
# disable the lock: echo 0 > /sys/module/rt2x00lib/parameters/monitor_channel
```

## Quick diagnostic worth running first

The most common concrete trigger is a **second interface left up**. Modern
`airmon-ng` deletes the managed interface, but not every setup does. Check:

```bash
iw dev            # expect ONLY your monitor interface; delete any stray managed vif
```

If a managed `wlanX` is still present alongside the monitor interface, removing it
(`sudo iw dev wlanX del`) may stop the drift on its own; the channel lock is the
robust catch-all regardless of the trigger.

## Notes

- Not compile-tested in the authoring environment; validated structurally against
  current mainline sources (signatures, flags, and the frequency helper were read
  from the tree; the patch applies cleanly with `patch -p1 --dry-run`).
- No `Signed-off-by:` is included — add your own if submitting upstream (the DCO
  sign-off must be yours). Note upstream would likely prefer a different shape than
  a module parameter; this is tuned for a reliable out-of-tree workaround.
