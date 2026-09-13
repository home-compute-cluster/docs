# Postmortem: opus reset loop (Sep 2026)

**Node:** opus, HP EliteDesk 800 G4 Desktop Mini, i5-8500, 192.168.1.252
**Role:** K3s server (control plane), etcd member, worker
**Status:** resolved, root cause unconfirmed
**Detected:** ~4 Sep 2026 | **Last reset:** 6 Sep 2026 | **Stable since:** 2 days

---

## Summary

opus was hard resetting roughly every 30 minutes for about two days. No crash, no panic, no clean shutdown. The machine just stopped mid-execution and came back 60 to 90 seconds later on a full POST cycle.

It is now stable after I replaced the CMOS battery and added `i915.enable_dc=0` to the kernel command line. I changed both at once and did not run the revert test, so I cannot say which one fixed it. There is also a third possibility I did not plan for, covered below.

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
| 6 Sep | Replaced CMOS battery, added `i915.enable_dc=0`. |
| 6 to 8 Sep | Stable. k3s back, uncordoned, rejoined etcd cleanly. |

## What the evidence actually said

Mean uptime was about 29 minutes. Range was 1.5 to 82 minutes across 30 consecutive boots. That scatter is basically memoryless, which killed the timer and scheduled-job theories immediately. Software failures cluster around events. This did not cluster around anything.

Ruled out, and how:

- **Cluster automation.** Checked `homelab-cicd-config`. kured appears only in `for:` comments on PrometheusRules, never as a deployment. Nothing in-cluster reboots nodes.
- **Watchdog.** No watchdog device, no `wdt` modules loaded, `RuntimeWatchdogSec` commented off.
- **Thermal.** 46C package against a 100C crit. Nothing thermal in the kernel log. The `90D` code I had logged earlier was noise.
- **Kernel panic.** `panic=0` is the default, so a panic would hang rather than reboot. pstore was mounted and empty.
- **Memory errors.** rasdaemon reported clean. This means nothing on this board, see below.
- **RTC state.** Clock correct and synced at the time I checked, no wake alarm armed.

The tail of the last boot before a reset was completely ordinary. etcd snapshot reconciliation, chronyd source selection, veth churn, an iSCSI attach. Then nothing. No shutdown sequence, no error, no partial write. That is the signature of the CPU losing its reset line or the rails browning out.

## The test that mattered

Draining opus and stopping k3s entirely, then leaving it idle. It reset again after 2 hours.

That killed software as a cause outright. No kernel workload, no etcd, no Longhorn, no container churn, and it still went down. Everything after that point was a hardware hunt.

The 2 hours against a 29 minute mean was a mild signal that load raises the failure rate, but the fault clearly exists without load.

## What I changed

Two things, at the same time, which broke the one-variable-at-a-time discipline I had been holding up to that point:

1. Replaced the CR2032.
2. Added `i915.enable_dc=0` to `GRUB_CMDLINE_LINUX_DEFAULT`. Found this from other people running the same box as a homelab node.

There is a third variable I did not intend to introduce. Swapping the cell on this chassis means lifting the drive cage, which flexes the board and disturbs the SODIMM and NVMe seating. Given the `2E1` intermittent SODIMM detection code sitting in the BIOS event log from before any of this, an accidental reseat is a real candidate and it is not one I chose or controlled.

So: three candidates, one outcome, no attribution.

## Attribution, honestly

`i915.enable_dc=0` is the leading candidate, but only by elimination.

The mechanism is at least coherent. Display C-states gate power wells in the PCH. A bad DC5/DC6 transition can trigger a platform reset with nothing on the CPU side to log it and nothing in pstore. That matches what I saw exactly.

But my corroboration test failed. I expected the `output_poll_execute` and `i915_hpd_poll_init_work` "hogged CPU" warnings to drop off if the display path was the culprit. They did not. They are still climbing.

That turned out to be a bad test rather than a bad result. Those warnings are hotplug polling, the driver probing outputs on a headless box. `enable_dc` controls display power wells. Different parts of the driver. Disabling DC states was never going to change polling behaviour. So the counter climbing neither confirms nor denies anything, and I built an expectation on a mechanism I had not checked.

I am also less sure than I was that the coin cell was irrelevant. The reasoning at the start was that a dying cell only matters when mains power is removed, which is true for RTC and settings retention. But the RTC well sits inside the PCH, and a marginal cell dragging that rail is a rare but real path to a platform reset. I dismissed it more firmly than the evidence justified.

I am not running the revert test. It would cost days of instability and etcd churn to buy knowledge I do not need right now. That is a deliberate trade, not an oversight.

## What I actually learned

**Not every fault leaves evidence.** I went into this expecting the logs to tell me something. They told me the machine was working normally right up until it was not. That absence was itself the most informative thing in the whole investigation, because it eliminated every failure mode that would have left a trace. Reading the shape of the gap mattered more than reading the contents of the log.

**The strongest signal was in the distribution, not the messages.** 30 uptimes with a memoryless spread ruled out more theories in one pass than any individual log line did.

**Ruling things out is progress even when nothing gets fixed.** The idle test produced no fix and no root cause, and it was the single most valuable step. It cut the search space in half.

**I dismissed the actual lead as noise.** The i915 workqueue warnings were in the logs from the very first boot I looked at. I filed them as a known annoyance and moved on. Recurring low-severity noise on the same subsystem deserves a second look when nothing else fits.

**No ECC means memory faults are invisible.** `EDAC ie31200: No ECC support`. rasdaemon reporting clean on this board is not evidence of anything. I nearly treated it as a clean bill of health. Something to weigh when buying nodes.

**Old hardware has a community.** The fix came from other people running the same model in homelabs, not from anything I derived. That is genuinely luck. A less common box and I would still be bisecting.

**Discipline held right up until it did not.** I was careful about single variables through the whole diagnosis and then changed three things at once at the fix. Understandable when you just want the node back, but it is why this postmortem has no root cause section.

## Follow-ups

- [ ] **Alert on unexpected reboots.** `changes(node_boot_time_seconds[1h]) > 0` with a suitable `for:`. I found this by eye. I should not have.
- [ ] **Overnight memtest86+.** Still not done. With no ECC, this is the only window into memory faults, and `2E1` is unexplained.
- [ ] **Photograph the BIOS event log properly.** Checked it, nothing new, but I did not capture it.
- [ ] **BIOS 02.24.00 to 02.27.00.** Deferred deliberately during diagnosis because it bundles microcode and ACM changes and clears the event log. Reasonable to do now that things are stable.
- [ ] **`drm_kms_helper.poll=0`** to silence the polling warnings. Cosmetic, do it separately so it stays clean.
- [ ] **Watch opus for a full week** before treating this as closed.
- [ ] **Think about etcd margin during long node outages.** Losing one of three is fine. Having one out for maintenance leaves no margin at all, and I ran that way for half a day without planning to.
- [ ] **Unrelated:** `adaptsg-dev` `source-checkout` init container has been in CrashLoopBackOff throughout. Nothing to do with this, still needs fixing.

## Open question

Whether `i915.enable_dc=0` is the fix or a coincidence. If opus stays up through the week I will write it down as the working answer with the caveat attached. If it resets again, the reseat theory moves to the front and the next step is a single-stick bisect followed by a power brick swap, with 12+ hours of observation per variable.
