# Postmortems

This section is for real cluster failures: the ones where something broke badly
enough that I had to step in, debug it and change how the cluster is run. Weekly
diary entries mention these incidents in passing. The postmortems are where the
full timeline, the evidence and the fixes live.

## What goes in a postmortem

A failure earns a write-up when at least one of these is true:

- A service or the whole cluster became unreachable.
- The control plane lost fault tolerance (for example etcd running on 2 of 3
  members).
- Recovery needed hands on the hardware rather than a `kubectl` command.
- The cause was not obvious and the investigation taught something worth keeping.

Each postmortem follows roughly the same shape: summary, impact, timeline, what the
evidence showed, root cause (or an honest note that the root cause is unconfirmed),
what went wrong in the response, action items and lessons. They are blameless in
the usual sense. The point is to fix the system, not to feel bad about the person
who configured it, even when that person is me.

## Incidents

| Date | Incident | Severity | Duration | Status |
|------|----------|----------|----------|--------|
| 2 Sep 2026 | [deus silent lockup and total tailnet plane outage](2026-09-02-deus-lockup.md) | High | About 21 minutes, physical reboot needed | Root cause narrowed to hardware or firmware |
| 4 to 6 Sep 2026 | [opus reset loop](2026-09-06-opus-reset-loop.md) | Medium | About two days of resets every 30 minutes or so | Resolved, root cause unconfirmed |
| 8 to 12 Sep 2026 | [deus control-plane outage](2026-09-08-deus-control-plane-outage.md) | SEV-1 | About 3 days 15 hours | Root cause confirmed, follow-ups open |

## Themes so far

Reading the three together, a few patterns stand out:

- **deus carries too many roles.** It is a control-plane node, an etcd member, a
  worker and the only Tailscale subnet router. The last role is what turned two
  separate single-node faults into a total loss of remote access.
- **Nothing alerted.** Every incident was found by a person noticing a gap in
  Grafana or a `kubectl` timeout. The monitoring stack also sits behind the same
  tailnet path it would need to report on.
- **Hand-applied config drifts.** Tailscale flags and `ufw` state were set by hand
  per node, and both caused faults that nothing could have detected in advance.
- **No ECC and no out-of-band access.** Two of the three incidents ended with me
  physically at the machine, and memory faults on these boards cannot be ruled
  out from logs alone.
