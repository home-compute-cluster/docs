# Postmortem: opus reset loop (Sep 2026)

**Node:** opus, HP EliteDesk 800 G4 Desktop Mini, i5-8500, 192.168.1.252
**Role:** K3s server (control plane), etcd member, worker
**Status:** resolved. Root cause: i915 display power-state transitions on a headless box
**Detected:** ~4 Sep 2026 | **Last reset:** 6 Sep 2026 | **Fixed:** 6 Sep 2026, no resets since

---

## Summary

opus was hard resetting roughly every 30 minutes for about two days. No crash, no panic, no clean shutdown. The machine just stopped mid-execution and came back 60 to 90 seconds later on a full POST cycle.

The cause was the Intel integrated graphics driver, i915. On this platform, the display engine moving between its low-power DC5/DC6 states can take the whole machine down with it, and on a headless node with nothing plugged into the display outputs the driver was visibly struggling from the first boot I looked at. Adding `i915.enable_dc=0` to the kernel command line disables those display power states. opus has not reset since.

## Impact

- opus was in and out of the cluster ~30 times over the observed window. It is an etcd member, so every rejoin churned the leader.
- No data loss, no service outage. Three-member etcd held quorum throughout because deus and sol stayed up.
- Prometheus lost scrape coverage for opus in the gaps, which is how I noticed in the first place.
- While opus was drained and stopped for testing, etcd ran at 2 of 3 with quorum 2. Zero fault tolerance for about half a day. Nothing happened, but that was luck.

## Detection

I did not get an alert. I noticed gaps in the Prometheus graphs, every 30 minutes to an hour, and only then went looking. That is a detection gap, not a diagnosis. Covered in follow-ups.

## Timeline

| When | What |
|---|---|
| ~4 Sep | Resets begin. Not noticed. |
| 6 Sep, morning | Gaps spotted in Prometheus. Started digging. |
| 6 Sep | Confirmed 165 boots against 6 clean shutdowns. Every reset unclean. |
| 6 Sep | Ruled out cluster automation, watchdog, thermal, scheduled jobs. |
| 6 Sep | Drained opus, stopped k3s, ran idle. Reset again after 2 hours. |
| 6 Sep | Added `i915.enable_dc=0` to the kernel command line. |
| 6 to 8 Sep | Stable. k3s back, uncordoned, rejoined etcd cleanly. |
| Since | No further resets. |

## What the evidence actually said

Mean uptime was about 29 minutes. Range was 1.5 to 82 minutes across 30 consecutive boots. That scatter is basically memoryless, which killed the timer and scheduled-job theories immediately. Software failures cluster around events. This did not cluster around anything.

Ruled out, and how:

- **Cluster automation.** Checked `homelab-cicd-config`. kured appears only in `for:` comments on PrometheusRules, never as a deployment. Nothing in-cluster reboots nodes.
- **Watchdog.** No watchdog device, no `wdt` modules loaded, `RuntimeWatchdogSec` commented off.
- **Thermal.** 46C package against a 100C crit. Nothing thermal in the kernel log. The `90D` code I had logged earlier was noise.
- **Kernel panic.** `panic=0` is the default, so a panic would hang rather than reboot. pstore was mounted and empty.
- **RTC state.** Clock correct and synced at the time I checked, no wake alarm armed.

The tail of the last boot before a reset was completely ordinary. etcd snapshot reconciliation, chronyd source selection, veth churn, an iSCSI attach. Then nothing. No shutdown sequence, no error, no partial write. That is the signature of the platform resetting underneath the CPU, not of the OS deciding to stop.

The one thing that was *not* ordinary was the i915 driver. Every boot logged workqueue warnings from it, `output_poll_execute` and `i915_hpd_poll_init_work` reported as hogging the CPU. That is the display driver stalling while it probes outputs on a machine with no monitor attached. I had seen these from the very first boot and filed them as a known annoyance.

## The test that mattered

Draining opus and stopping k3s entirely, then leaving it idle. It reset again after 2 hours.

That killed software as a cause outright. No etcd, no Longhorn, no container churn, and it still went down. What was left running on an idle headless box was the kernel and its drivers, and the only driver complaining was i915.

The 2 hours against a 29 minute mean fits that picture too: an idle machine sits in its low-power states longer and makes fewer transitions in and out of them.

## Root cause

Display C-states (DC5/DC6) gate power wells in the PCH. A bad transition into or out of them can trigger a platform reset with nothing on the CPU side to log it and nothing in pstore. That matches what I saw exactly: no panic, no shutdown sequence, a log that simply stops.

The driver was already showing it was unhappy on this headless setup through the workqueue warnings. Other people running the same EliteDesk model as a homelab node reported the same reset pattern and the same fix.

## What I changed

Added `i915.enable_dc=0` to `GRUB_CMDLINE_LINUX_DEFAULT` and rebooted. That keeps the display engine out of the DC5/DC6 states. The cost is a few watts at most on a machine that never drives a display anyway.

The hotplug polling warnings did not go away afterwards, and should not have been expected to. They come from output polling, which `enable_dc` does not touch. Silencing them is a separate, cosmetic change (follow-up below).

## What I actually learned

**Not every fault leaves evidence.** I went into this expecting the logs to tell me something. They told me the machine was working normally right up until it was not. That absence was itself the most informative thing in the whole investigation, because it eliminated every failure mode that would have left a trace. Reading the shape of the gap mattered more than reading the contents of the log.

**The strongest signal was in the distribution, not the messages.** 30 uptimes with a memoryless spread ruled out more theories in one pass than any individual log line did.

**Ruling things out is progress even when nothing gets fixed.** The idle test produced no fix on its own, and it was the single most valuable step. It cut the search space down to the hardware and the drivers.

**I dismissed the actual lead as noise.** The i915 workqueue warnings were in the logs from the very first boot I looked at. I filed them as a known annoyance and moved on. Recurring low-severity noise on the same subsystem deserves a second look when nothing else fits.

**No ECC means memory faults are invisible.** `EDAC ie31200: No ECC support`. rasdaemon reporting clean on this board is not evidence of anything. I nearly treated it as a clean bill of health. Something to weigh when buying nodes.

**Old hardware has a community.** The fix came from other people running the same model in homelabs, not from anything I derived. That is genuinely luck. A less common box and I would still be bisecting.

## Follow-ups

- [ ] **Alert on unexpected reboots.** `changes(node_boot_time_seconds[1h]) > 0` with a suitable `for:`. Alertmanager now notifies a Discord channel, but this specific rule is not written yet. I 
- [ ] **`drm_kms_helper.poll=0`** to silence the polling warnings. Cosmetic, do it separately so it stays clean.
- [x] **Watch opus for a full week** before treating this as closed. It stayed up.
- [x] **Think about etcd margin during long node outages.** Losing one of three is fine. Having one out for maintenance leaves no margin at all, and I ran that way for half a day without planning to.
- [x] **Unrelated:** `adaptsg-dev` `source-checkout` init container has been in CrashLoopBackOff throughout. Nothing to do with this, still needs fixing. Finally done.
