# Lessons: Storage and Backups

Storage is where mistakes stop being cheap. These lessons come from installing
Longhorn in Weeks 7 to 10, wiring up CNPG backups in Weeks 10 and 11, and adding
Longhorn monitoring in Week 15.

---

## 1. Longhorn cannot use a ZFS dataset

**What happened.** The plan was to give Longhorn space on nebula's ZFS `tank` pool.
Longhorn needs a dedicated disk formatted ext4 or XFS and mounted at a stable path,
so that was never going to work.

**The lesson.** Check what the storage layer needs from the layer below it before
buying or partitioning anything.

**What I do now.** Longhorn gets its own disks: a dedicated NVMe on nebula and an
ext4 partition on legion's WD Red SSD. On each node the default
`/var/lib/longhorn` path on the OS disk has scheduling disabled, so replicas only
land on the dedicated disks.

## 2. You cannot shrink a ZFS pool to make room

**What happened.** The fallback idea was to take space back from the ZFS mirror.
ZFS does not support shrinking a vdev in place, so that meant either breaking the
mirror or a full backup, destroy and restore.

**The lesson.** Some storage decisions are one-way. Plan capacity with that in mind.

## 3. The replica count has to match the disks you actually have

**What happened.** In Week 15, while adding Longhorn monitoring, the volumes showed up
as degraded and unschedulable. After a brief panic it turned out Longhorn wanted
three replicas and there are only two nodes with Longhorn disks (nebula and legion).

**The lesson.** Longhorn's default is three replicas. With two storage nodes, a
volume asking for three can never be healthy, and the dashboard will say so.

**What I do now.** Two replicas is the target, and it has to be set in both places
Longhorn reads it from: the default replica count setting and the StorageClass. Two is also the floor: one replica gives no protection
against losing a node. The honest downside of two targets is zero placement slack,
so losing nebula or legion leaves every volume degraded until it comes back.

## 4. `multipathd` breaks Longhorn, and the check looks for the daemon, not the config

**What happened.** Longhorn's environment check flagged `multipathd` on every node.
deus already had the right blacklist in `/etc/multipath.conf`, but the warning
stayed.

**What I do now.** Disable it outright on nodes with directly attached storage
(`sudo systemctl disable --now multipathd`). Multipath is for SAN setups and does
nothing useful here.

## 5. Used disks remember their previous life

**What happened.** The free NVMe drive for nebula showed no partitions in `lsblk`,
but `mkfs.ext4` found an old GPT table on it.

**What I do now.** The sequence for a new Longhorn disk is: check SMART
(`smartctl -a`), `wipefs -a`, `mkfs.ext4 -L <label>`, create the mount point, get the
UUID with `blkid`, add an fstab entry by UUID with `nofail`, then `mount -a`. The
`nofail` matters: without it, a dead disk stops the node from booting.

## 6. Ubuntu Server does not use the whole disk by default

**What happened.** Every node had about 100GB for the OS despite larger disks. The
installer creates an LVM volume group from the whole disk but only allocates part of
it to the root volume.

**What I do now.** Extend it online, no reboot needed:

```bash
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

It belongs in node provisioning, so new nodes do not repeat the problem.

## 7. Archiving working does not mean restore works

**What happened.** After the Week 11 chase, WAL segments finally flowed from the
portfolio database to Garage. That proves the credentials, the endpoint and the
bucket. It says nothing about whether I can rebuild the database from what is in the
bucket.

**The lesson.** A backup is only proven by a restore.

**What I do now.** Before real data lands in a database, run a base backup and a
canary restore into a throwaway `Cluster`. This drill is still outstanding and it is
listed as such rather than assumed done.

## 8. S3-compatible does not mean identical to S3

**What happened.** Once authentication worked, Garage returned `400 Bad Request` on
`HeadBucket`. barman-cloud signs requests for region `us-east-1`, and Garage's
default region is `garage`.

**What I do now.** Set `s3_region = "us-east-1"` in `garage.toml` so it matches what
AWS client libraries assume. The same class of mismatch is worth checking for any
S3-compatible store.

**Related.** The whole reason for Garage was that MinIO's community edition removed
access key management from its web console in 2025. Check the licence and feature
set of self-hosted tools before building on them, not after.

## 9. Check CRD fields against the version you actually run

**What happened.** The Longhorn `BackupTarget` failed to sync with
`.spec.pollIntervalInSeconds: field not declared in schema`. The `v1beta2` API uses
`pollInterval`.

**What I do now.** When copying an example, confirm the field names with
`kubectl explain backuptarget.spec` against the installed CRD rather than trusting
a blog post or an older version's docs.

## 10. Short resource names can collide once you add operators

**What happened.** After Longhorn was installed, `kubectl get backup` returned
Longhorn backups instead of CNPG backups.

**What I do now.** Use the full name when more than one operator defines a kind:
`kubectl get backups.postgresql.cnpg.io` or `kubectl get backups.longhorn.io`.

## 11. Keep backups on a different failure domain from the data

**What happened.** nebula is the node with the most storage, so it became the backup
path for most deployments: CNPG backups go to Garage on `tank`, and the Longhorn
backup target is on nebula as well.

**The lesson.** That protects against losing a Longhorn volume or a database pod. It
does not protect against losing nebula itself, or the house.

**What I do now.** Treat an off-site copy of the most important backups as an open
gap, and write it down as one.
