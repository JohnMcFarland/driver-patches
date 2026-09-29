# Dev update: RT3070 / rt2800usb monitor-mode channel drift

**Status:** Fix written and structurally validated against mainline. Applies cleanly
with `patch -p1`. Not yet compiled or tested on hardware. Handoff for build + on-air test.

**Component:** `drivers/net/wireless/ralink/rt2x00/` (Linux kernel), chipset RT3070
(`148f:3070`), module `rt2800usb`.

---

## 1. The bug

In monitor mode, after `iw dev wlan0 set channel 6`, the radio drifts to other
channels within seconds (observed: 3, 7, 9, 11, 12, 14). The targets track nearby
APs with active traffic. This breaks simultaneous `airodump-ng` capture and
`aireplay-ng` injection because the adapter is rarely on the requested channel.
Present across kernels 3.x through 7.1.

## 2. Already ruled out (do not re-investigate)

- **NetworkManager / wpa_supplicant** killed via `airmon-ng check kill`. Drift persists.
- **VCO / gain calibration in the periodic link tuner** (`rt2x00link_tuner`). Patched
  to skip both under `CONFIG_MONITORING`; drift persisted, including sub-second drift
  that fires before any tuner timer. Not the primary cause.
- **USB port type** (tested 2.0). No change.
- **`nohwcrypt=1`.** No change.

## 3. Root cause

`rt2800usb` does not implement channel contexts. mac80211 emulates them and pushes
every hardware retune through the driver's single `.config` callback:

```
rt2x00mac_config()  ->  rt2x00lib_config()  ->  rt2800_config_channel()
  rt2x00mac.c            rt2x00config.c           rt2800lib.c
```

That callback is the **only** runtime path that retunes the radio on this hardware.
So any off-channel work mac80211 schedules while a monitor interface is "locked" to a
fixed channel arrives here as a normal `IEEE80211_CONF_CHANGE_CHANNEL` request and
moves the radio. Triggers: software/background scan, channel-context reassignment
when another vif (e.g. a leftover managed `wlanX` left UP) wants a different channel,
and remain-on-channel work. The drift targets track nearby APs because those are the
channels the upper layers are chasing.

This also explains why the link-tuner/VCO patch did nothing: the drift comes through
the channel-config path, not the tuner, and precedes the tuner timers.

## 4. The fix (explicit)

**One file changed:** `drivers/net/wireless/ralink/rt2x00/rt2x00mac.c`. Two additions,
39 lines, no deletions. Marker `CAELION-FIX-RT3070-CHANDRIFT` in both comments and the
runtime debug message.

The driver cannot distinguish a user `iw set channel` from a mac80211 hop (identical
`.config` call), so the fix is an **opt-in module parameter** plus a guard clause.

### Fix 1 — new module parameter `monitor_channel`

Inserted after the includes, before the first function:

```c
/*
 * CAELION-FIX-RT3070-CHANDRIFT
 * Optional monitor-mode channel lock. When set to a non-zero 802.11 channel
 * number, the driver refuses runtime channel-change requests that would move
 * the radio off that channel while an interface is in monitor mode. This pins
 * a fixed-channel capture (e.g. airodump-ng/aireplay-ng on one channel)
 * against off-channel excursions requested from above the driver (software
 * scan, channel-context reassignment, remain-on-channel), all of which reach
 * this hardware as IEEE80211_CONF_CHANGE_CHANNEL and are the source of the
 * observed drift. Default 0 preserves historical behaviour; station, AP and
 * managed modes never set IEEE80211_CONF_MONITOR and are unaffected.
 */
static int monitor_channel;
module_param(monitor_channel, int, 0644);
MODULE_PARM_DESC(monitor_channel,
		 "Lock monitor-mode radio to this 802.11 channel (0 = disabled)");
```

- Type `int`, default `0` (feature off). Mode `0644` so it is writable at runtime via
  `/sys/module/rt2x00lib/parameters/monitor_channel`.
- It lands in `rt2x00lib.ko` because `rt2x00mac.o` is linked into that module.

### Fix 2 — guard clause in `rt2x00mac_config()`

Inserted immediately after the existing `DEVICE_STATE_PRESENT` early-return, before the
queue-stop / lock / `rt2x00lib_config()` sequence:

```c
	/*
	 * CAELION-FIX-RT3070-CHANDRIFT
	 * In monitor mode with the channel lock armed, drop a channel-change
	 * request that targets a channel other than the locked one, so the
	 * radio stays put. Everything else in this configuration change
	 * (power, retry limits, PS, monitor enable/disable, ...) is still
	 * applied by rt2x00lib_config(). The lock only acts while
	 * IEEE80211_CONF_MONITOR is set, so it cannot affect station, AP or
	 * managed-mode channel switching.
	 */
	if ((changed & IEEE80211_CONF_CHANGE_CHANNEL) &&
	    monitor_channel > 0 &&
	    (conf->flags & IEEE80211_CONF_MONITOR) &&
	    conf->chandef.chan &&
	    ieee80211_frequency_to_channel(conf->chandef.chan->center_freq) !=
		monitor_channel) {
		rt2x00_dbg(rt2x00dev,
			   "CAELION-FIX-RT3070-CHANDRIFT: refusing channel change to %d MHz, locked on channel %d\n",
			   conf->chandef.chan->center_freq, monitor_channel);
		changed &= ~IEEE80211_CONF_CHANGE_CHANNEL;
	}
```

What each condition does, in order:

1. `changed & IEEE80211_CONF_CHANGE_CHANNEL` — only act on channel-change requests.
2. `monitor_channel > 0` — feature is armed (default 0 = never fires).
3. `conf->flags & IEEE80211_CONF_MONITOR` — only while in monitor mode.
4. `conf->chandef.chan` — NULL guard.
5. `ieee80211_frequency_to_channel(center_freq) != monitor_channel` — the request
   targets a channel other than the locked one.

When all are true, it clears the channel-change bit from the local `changed` (passed by
value) and logs. `rt2x00lib_config(rt2x00dev, conf, changed)` then runs as normal but
with the channel bit off, so the RF is not reprogrammed. The initial `set channel N`
where `N == monitor_channel` is allowed (condition 5 is false), so the lock engages
without blocking the intended channel.

### Full diff

```diff
@@ -16,6 +16,23 @@
 #include "rt2x00.h"
 #include "rt2x00lib.h"
 
+/*
+ * CAELION-FIX-RT3070-CHANDRIFT
+ * Optional monitor-mode channel lock. When set to a non-zero 802.11 channel
+ * number, the driver refuses runtime channel-change requests that would move
+ * the radio off that channel while an interface is in monitor mode. This pins
+ * a fixed-channel capture (e.g. airodump-ng/aireplay-ng on one channel)
+ * against off-channel excursions requested from above the driver (software
+ * scan, channel-context reassignment, remain-on-channel), all of which reach
+ * this hardware as IEEE80211_CONF_CHANGE_CHANNEL and are the source of the
+ * observed drift. Default 0 preserves historical behaviour; station, AP and
+ * managed modes never set IEEE80211_CONF_MONITOR and are unaffected.
+ */
+static int monitor_channel;
+module_param(monitor_channel, int, 0644);
+MODULE_PARM_DESC(monitor_channel,
+		 "Lock monitor-mode radio to this 802.11 channel (0 = disabled)");
+
 static int rt2x00mac_tx_rts_cts(struct rt2x00_dev *rt2x00dev,
 				struct data_queue *queue,
 				struct sk_buff *frag_skb)
@@ -317,6 +334,28 @@
 		return 0;
 
 	/*
+	 * CAELION-FIX-RT3070-CHANDRIFT
+	 * In monitor mode with the channel lock armed, drop a channel-change
+	 * request that targets a channel other than the locked one, so the
+	 * radio stays put. Everything else in this configuration change
+	 * (power, retry limits, PS, monitor enable/disable, ...) is still
+	 * applied by rt2x00lib_config(). The lock only acts while
+	 * IEEE80211_CONF_MONITOR is set, so it cannot affect station, AP or
+	 * managed-mode channel switching.
+	 */
+	if ((changed & IEEE80211_CONF_CHANGE_CHANNEL) &&
+	    monitor_channel > 0 &&
+	    (conf->flags & IEEE80211_CONF_MONITOR) &&
+	    conf->chandef.chan &&
+	    ieee80211_frequency_to_channel(conf->chandef.chan->center_freq) !=
+		monitor_channel) {
+		rt2x00_dbg(rt2x00dev,
+			   "CAELION-FIX-RT3070-CHANDRIFT: refusing channel change to %d MHz, locked on channel %d\n",
+			   conf->chandef.chan->center_freq, monitor_channel);
+		changed &= ~IEEE80211_CONF_CHANGE_CHANNEL;
+	}
+
+	/*
 	 * Some configuration parameters (e.g. channel and antenna values) can
 	 * only be set when the radio is enabled, but do require the RX to
 	 * be off. During this period we should keep link tuning enabled,
```

## 5. Why it does not break other modes

- **Gated on `IEEE80211_CONF_MONITOR`.** Station, AP and managed modes never set that
  flag, so their channel switching (including CSA) is never touched.
- **Default 0 disables it.** Zero behaviour change for anyone who does not opt in.
- **Only the channel bit is stripped.** `CONFIG_MONITORING` is set/cleared inside
  `rt2x00lib_config()` from `IEEE80211_CONF_MONITOR` independently of the channel flag,
  and TX power / retry / PS are handled independently, so all of those still apply.
- **Link tuner / VCO calibration untouched** and continue normally outside monitor mode.

## 6. Build (out-of-tree)

```bash
cd /path/to/linux-source            # matching uname -r, or the mainline tree
patch -p1 < rt2x00-monitor-channel-lock.patch
make -C /lib/modules/$(uname -r)/build \
     M=$(pwd)/drivers/net/wireless/ralink/rt2x00 modules
```

Load rebuilt modules (parameter lives in `rt2x00lib.ko`):

```bash
sudo rmmod rt2800usb rt2800lib rt2x00usb rt2x00lib 2>/dev/null
sudo insmod ./drivers/net/wireless/ralink/rt2x00/rt2x00lib.ko
sudo insmod ./drivers/net/wireless/ralink/rt2x00/rt2x00usb.ko
sudo insmod ./drivers/net/wireless/ralink/rt2x00/rt2800lib.ko
sudo insmod ./drivers/net/wireless/ralink/rt2x00/rt2800usb.ko
```

(or copy into `/lib/modules/$(uname -r)/...`, `depmod -a`, reload).

## 7. Use

```bash
sudo airmon-ng check kill
echo 6 | sudo tee /sys/module/rt2x00lib/parameters/monitor_channel
sudo ip link set wlan0 down
sudo iw dev wlan0 set monitor control
sudo ip link set wlan0 up
sudo iw dev wlan0 set channel 6      # stays on 6
# move the lock: set the parameter to N, then re-issue "iw ... set channel N"
# disable: echo 0 > /sys/module/rt2x00lib/parameters/monitor_channel
```

## 8. Verification done in this session

Read from mainline (torvalds/master via raw.githubusercontent.com):

- `rt2x00mac_config()` signature is now `(struct ieee80211_hw *hw, int radio_idx, u32 changed)`.
  The patch does **not** touch the signature line, so it applies on both mainline and
  the 7.1.5 form `(hw, u32 changed)`; the guard body only uses `changed`, `conf`,
  `rt2x00dev`, all present in both.
- `rt2x00lib_config()` gates all channel RF programming behind
  `IEEE80211_CONF_CHANGE_CHANNEL`, and sets/clears `CONFIG_MONITORING` from
  `IEEE80211_CONF_MONITOR` independently. Confirms stripping the channel bit is safe.
- `chan->hw_value` is an index into `spec.channels[]`, not the 802.11 channel number;
  hence matching via `ieee80211_frequency_to_channel(center_freq)` (correct on 2.4 and
  5 GHz).
- `ieee80211_frequency_to_channel(int freq)` is a `static inline` in
  `include/net/cfg80211.h`, reachable by the driver.
- `IEEE80211_CONF_MONITOR = 1<<0`, `IEEE80211_CONF_CHANGE_CHANNEL = BIT(6)`,
  `ieee80211_conf.chandef` is a `struct cfg80211_chan_def`.
- Brace/paren balance identical to pristine; `patch -p1 --dry-run` and a real apply
  succeed cleanly.

## 9. Known limitation (cosmetic)

`rt2x00lib_config()` still copies `conf->chandef` into `rt2x00dev->curr_freq` /
`curr_band` unconditionally. While a change is being refused, the per-packet radiotap
channel annotation can briefly report the channel mac80211 believes it moved to rather
than the pinned channel. The radio physically stays on the locked channel for capture
and injection. Optional follow-up: also suppress that bookkeeping when the change is
refused (would enlarge the patch beyond the current guard-only change).

## 10. Next steps for whoever picks this up

1. **Compile** with the section 6 command on the target kernel (7.1.5) and fix any
   build error. Nothing is compile-tested yet.
2. **On-air test:** arm `monitor_channel=6`, enter monitor mode, set channel 6, run
   `airodump-ng` + `aireplay-ng`, confirm the radio holds channel 6 over minutes.
   Watch `dmesg` for the `CAELION-FIX-RT3070-CHANDRIFT: refusing channel change...`
   lines to see which channels/frequencies the upper layers were pushing.
3. **Check for a stray interface first:** `iw dev`. If a managed `wlanX` is still up
   next to the monitor interface, deleting it (`sudo iw dev wlanX del`) may stop the
   drift by itself. That is the most likely concrete trigger; the lock is the robust
   catch-all regardless.
4. **If upstreaming:** a module parameter is right for an out-of-tree workaround, but
   maintainers would likely prefer a different shape (per-phy config, or fixing the
   upper-layer trigger). Add your own `Signed-off-by:` (DCO). I did not add one.

Files: `rt2x00-monitor-channel-lock.patch`, `README.md`.
