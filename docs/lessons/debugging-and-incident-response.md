# Lessons: Debugging and Incident Response

The hardest incidents on this cluster were the ones where the logs said nothing
useful. These lessons are mostly about how to investigate when the evidence is thin,
and about the habits that held up (or did not) under pressure.

---

## 1. A fault that survives a power cycle is written down on disk

**What happened.** During the September control-plane outage, deus was power cycled
and was still unreachable afterwards. That fact was known on day one. The
investigation still spent days on thermals, storage and memory before looking at
persistent configuration. The cause turned out to be a Tailscale preference and a
firewall rule, both of which load at boot.

**The lesson.** Runtime state, temperature, memory corruption and transient routing
all clear on a reboot. If the problem comes back after one, start with "what
configuration is applied at boot".

## 2. Read the shape of the gap, not just the log lines

**What happened.** opus reset about 30 times over two days with uptimes anywhere from
1.5 to 82 minutes. That scatter has no pattern, which ruled out timers, cron jobs and
anything scheduled in one pass. Software failures tend to cluster around events. This
did not.

**The lesson.** When individual logs are empty, the distribution of failures can
still eliminate whole categories of cause.

## 3. Logs that end mid-sentence mean the machine stopped, not shut down

**What happened.** On deus, a clean shutdown always ends with
`systemd-journald: Journal stopped`. Three boots out of twenty ended on an ordinary
line with no shutdown sequence at all, which turned one lockup into a pattern of
three.

**What I do now.**

```bash
journalctl --list-boots
journalctl -b -1 -n 5          # last lines of the previous boot
```

Check the last line of each boot. No "Journal stopped" means a hard hang or power
loss.

## 4. `sar` survives what journald loses

**What happened.** journald had nothing from the last minutes before the deus hang.
sysstat's ten minute samples were on disk and showed the machine idle, with memory
flat and no load. That ruled out resource exhaustion immediately.

**What I do now.** sysstat is installed on every node.

## 5. A watchdog that stays silent is also evidence

**What happened.** The NMI watchdog was enabled on deus and did not fire during the
hang. It catches CPUs spinning with interrupts disabled. Its silence suggested the
CPU stopped executing rather than got stuck in a loop.

**The lesson.** Knowing what a tool would have reported narrows things down even when
it reports nothing.

## 6. No ECC means memory faults are invisible, not absent

**What happened.** `EDAC ie31200: No ECC support` appears on these boards. On opus,
rasdaemon reporting "clean" looked like good news until I remembered it has no way to
see errors on this hardware.

**What I do now.** Treat memory as an untested suspect on these nodes, and run
memtest86+ overnight when a hang has no other explanation.

## 7. A CPU spike on a control-plane node usually means it is failing, not busy

**What happened.** Grafana showed deus's CPU jumping just before its metrics
disappeared. It was first read as a possible cause and sent the investigation toward
runaway workloads. It was actually the isolated etcd member retrying elections and
peer probes in a tight loop after it had already lost contact.

**The lesson.** When a control-plane node spikes and then goes quiet, ask what the
process is retrying before asking what is consuming the machine.

## 8. Take the workload away and see if the fault stays

**What happened.** opus was drained, `k3s` was stopped and the machine was left idle.
It reset again two hours later.

**The lesson.** That one test produced no fix and no root cause, and it was still the
most valuable step of the investigation. It ruled out every software cause at once.

## 9. Change one thing at a time, especially at the fix

**What happened.** I was careful about single variables all through the opus
diagnosis, and that is what made the fix credible. By the time I added
`i915.enable_dc=0`, the idle test had already narrowed the cause to the hardware and
its drivers, and the only driver complaining in the logs was i915. The resets
stopped, and the evidence explained why.

**The lesson.** Discipline is easiest to drop right at the end, when you just want
the node back. That is exactly when it matters most for knowing what actually fixed
it.

## 10. Check the date and the freshness of what you are looking at

**What happened.** During the deus lockup, a burst of network teardowns looked like
the moment before the crash. They were from the day before. An impression that
Prometheus kept scraping deus through the outage was most likely a stale Grafana
panel.

**What I do now.** Confirm timestamps and time ranges before building a theory on a
log line or a graph.

## 11. Recurring low-severity noise deserves a second look when nothing else fits

**What happened.** The i915 graphics driver warnings on opus were in the logs from the
very first boot I read. I filed them as a known annoyance. They were the display
driver stalling on a headless box, and the root cause turned out to be that same
driver: an i915 display power setting fixed it.

## 12. Search for your exact hardware model

**What happened.** The opus fix came from other people running the same HP EliteDesk
as a homelab node. Back in Week 3, nebula's installer crash was fixed by `nomodeset`,
another problem specific to that hardware combination.

**The lesson.** Old business desktops have large communities running them 24/7.
Someone has usually hit your problem before.

## 13. Reboot to recover, but know that it destroys evidence

**What happened.** Both deus incidents needed a reboot to get service back. That was
the right call, and it also wiped the live state. Persistent journald is the only
reason the investigations could happen afterwards.

**What I do now.** If there is any way to look before rebooting, grab
`ip route show table all`, `ip rule`, `systemctl --failed` and the last journal lines
first. If there is not, make sure the logs will survive the reboot.

## 14. For pods, read the events before anything else

**The lesson from everyday debugging.** Most pod problems explain themselves if you
look in the right place:

```bash
kubectl describe pod <pod> -n <ns>     # Events section at the bottom
kubectl get events -n <ns> --sort-by=.lastTimestamp
kubectl logs <pod> -n <ns> --previous  # why the last container died
```
