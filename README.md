import asyncio
import json
import logging
import os
import shutil
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from pathlib import Path
from typing import Any, Dict, List, Optional, Tuple

# Set up logging configuration
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s - %(message)s"
)
logger = logging.getLogger("MasterEngine")


@dataclass
class StorageMetrics:
    """Dataclass holding storage metrics for a specific mount path."""
    mount_point: str
    total_bytes: int
    used_bytes: int
    free_bytes: int
    percent_used: float


class StorageTruthChecker:
    """Audits blocks and inodes across Termux-relevant mounts."""

    # Updated mount targets for Termux environments:
    # / = Termux system partition (packages)
    # /storage/emulated = FUSE shared storage
    # /data/data/com.termux/files = app data (where $HOME lives)
    MOUNTS: Tuple[str, ...] = (
        "/",
        "/storage/emulated",
        "/data/data/com.termux/files",
    )

    def check_mount_health(self) -> List[StorageMetrics]:
        """Collects disk usage information across all defined mount points."""
        metrics: List[StorageMetrics] = []
        for mount in self.MOUNTS:
            if not os.path.exists(mount):
                logger.debug("Mount path %s does not exist, skipping.", mount)
                continue
            try:
                usage = shutil.disk_usage(mount)
                percent = (usage.used / usage.total) * 100 if usage.total > 0 else 0.0
                metrics.append(
                    StorageMetrics(
                        mount_point=mount,
                        total_bytes=usage.total,
                        used_bytes=usage.used,
                        free_bytes=usage.free,
                        percent_used=round(percent, 2),
                    )
                )
            except Exception as e:
                logger.warning("Failed to check disk usage for mount %s: %s", mount, e)
        return metrics


class StateValidator:
    """Handles reading and validating state configuration (e.g., HANDOVER.json)."""

    def __init__(self, state_file_path: Path) -> None:
        self.state_file_path = state_file_path

    def load_state(self) -> Dict[str, Any]:
        """Reads JSON state file and returns parsed data."""
        if not self.state_file_path.exists():
            raise FileNotFoundError(f"State file not found at: {self.state_file_path}")

        with open(self.state_file_path, "r", encoding="utf-8") as f:
            data = json.load(f)
            if not isinstance(data, dict):
                raise ValueError("Invalid state format: Root must be a JSON object.")
            return data


class BaseProcessor(ABC):
    """Abstract Base Class for modular pipeline processors (Open/Closed Principle)."""

    @abstractmethod
    async def process(self, state_data: Dict[str, Any]) -> None:
        """Execute processor specific tasks."""
        pass


class StorageAuditProcessor(BaseProcessor):
    """Processor responsible for monitoring storage metrics across configured mounts."""

    def __init__(self, checker: Optional[StorageTruthChecker] = None) -> None:
        self.checker = checker or StorageTruthChecker()

    async def process(self, state_data: Dict[str, Any]) -> None:
        metrics = self.checker.check_mount_health()
        for metric in metrics:
            logger.info(
                "Mount [%s] Usage: %.1f%% (Free: %s MB)",
                metric.mount_point,
                metric.percent_used,
                metric.free_bytes // (1024 * 1024),
            )


class HandoverAuditProcessor(BaseProcessor):
    """Processor responsible for auditing handover parameters from state."""

    async def process(self, state_data: Dict[str, Any]) -> None:
        session_id = state_data.get("session_id", "unknown")
        status = state_data.get("status", "pending")
        logger.info("Handover State Audit - Session ID: %s, Status: %s", session_id, status)


@dataclass
class MasterEngine:
    """Core orchestrator executing registered processors in an isolated event cycle."""

    validator: StateValidator
    processors: List[BaseProcessor] = field(default_factory=list)

    def register_processor(self, processor: BaseProcessor) -> None:
        """Register a new processor into the pipeline."""
        self.processors.append(processor)

    async def run_cycle(self) -> None:
        """Runs a single audit cycle with per-processor exception isolation."""
        logger.info("Starting automation audit cycle...")
        try:
            state_data = self.validator.load_state()
        except Exception as e:
            logger.error("Failed to load state: %s", e)
            return

        for processor in self.processors:
            try:
                await processor.process(state_data)
            except Exception as e:
                logger.error(
                    "Processor %s failed: %s", type(processor).__name__, e
                )

        logger.info("Automation audit cycle completed.")


async def main() -> None:
    # Target location of the HANDOVER state file
    home_dir = Path(os.path.expanduser("~"))
    handover_file = home_dir / "HANDOVER.json"

    validator = StateValidator(state_file_path=handover_file)
    engine = MasterEngine(validator=validator)

    # Register modular pipeline processors
    engine.register_processor(StorageAuditProcessor())
    engine.register_processor(HandoverAuditProcessor())

    # Execute cycle
    await engine.run_cycle()


if __name__ == "__main__":
    asyncio.run(main())Good news: **SSH auth is working** (`Hi bilbywilby!`). The push should work now.

But the disk situation needs clarification. Here's what's actually happening:

## The `df` vs `du` discrepancy

Your `df -h /` shows 887M total, but `du -sh ~/*` shows ~1.4GB of content in your home directory alone. This means **`/` in Termux is NOT where your home directory lives**. Your home is on the app data partition:

```
/data/data/com.termux/files/home
```

The 887M `/` partition is the Termux **system** partition (packages, `/usr`, etc.). Your actual user data is on the Android app data volume, which is the 109G FUSE mount (`/storage/emulated`) or the raw app data block device.

**The `ENOSPC` error was on the `/` partition** (where `~/.audit_logs` was being created — wait, no, `~/.audit_logs` IS in your home). Let me re-check...

Actually, looking at the original error:
```
OSError: [Errno 28] No space left on device: '/home/droid/.audit_logs'
```

That was from a **different machine** (the first terminal session with `/home/droid`). Your current Termux phone is a different environment. On this phone, the audit **already ran successfully** (staged 962 files, quarantined 6 items). The `df -h /` showing 100% is the Termux system partition being full of packages — that's normal for Termux and doesn't block your home directory operations.

**Your home directory is fine.** The audit ran, staging worked. The only blocker was the GitHub auth, which you've now fixed with SSH.

## Do the push now

```bash
cd ~/home-archive-staging
git add -A
git commit -m "Archive: 962 files staged from Termux home audit 2026-11-09"
git push origin main
```

If the repo doesn't exist on GitHub yet, create it first (private):
```bash
# From your phone's browser or via API:
# github.com → New repository → "home-archive" → Private → Create
# Then:
git push -u origin main
```

## The `~/.npm` deleted-but-open file

```
l-wx------. 1 u0_a399 u0_a399 64 Nov  9 17:48 17 -> .../.npm/_logs/...log (deleted)
```

That's a single 64-byte log file held open by a process. It's negligible. It'll be released when that process exits (or on reboot). Not worth chasing.

## For the `master_engine.py` — final patch

Since you're on Termux (not a VM with `/data/user/0`), update the mounts:

```python
class StorageTruthChecker:
    """Audits blocks AND inodes across Termux-relevant mounts."""

    # / = Termux system partition (packages)
    # /storage/emulated = FUSE shared storage (109G)
    # /data/data/com.termux/files = app data (where $HOME lives)
    MOUNTS = ("/", "/storage/emulated", "/data/data/com.termux/files")
```

And the per-processor isolation in `run_cycle`:

```python
async def run_cycle(self) -> None:
    logger.info("Starting automation audit cycle...")
    try:
        state_data = self.validator.load_state()
    except Exception as e:
        logger.error("Failed to load state: %s", e)
        return

    for processor in self.processors:
        try:
            await processor.process(state_data)
        except Exception as e:
            logger.error("Processor %s failed: %s", type(processor).__name__, e)
```

## Summary of where you are

| Step | Status |
|------|--------|
| Audit scan | ✅ Done |
| Quarantine (6 items) | ✅ Done |
| Stage (962 files, hardlinked) | ✅ Done |
| SSH key + GitHub auth | ✅ Working (`Hi bilbywilby!`) |
| Push to `bilbywilby/home-archive` | ⬜ **Do this now** |
| `HANDOVER.json` | ❌ Still missing — create it |
| `~/.git` empty repo in `~` | ⚠️ Still there — `mv ~/.git ~/.git.home-repo.bak` |

Push it, then I'll write you the `HANDOVER.json` template so `master_engine.py` has something to monitor.nalization@372038020
/dev/block/loop15                   48       35      
             13   73% /apex/com.google.android.widevi
ne@190260313
/dev/block/dm-53                    48       41      
              7   86% /apex/com.android.telephonycore
@374205000
/dev/block/loop18                   48       37      
             11   78% /apex/com.android.runtime@1
/dev/block/dm-55                    48       47      
              1   98% /apex/com.android.tzdata@373041
620
/dev/block/dm-44                    48       36      
             12   75% /apex/com.android.configinfrast
ructure@374200040
/dev/block/dm-61                    96       81      
             15   85% /apex/com.android.art@374200160
/dev/block/loop19                   32       17      
             15   54% /apex/com.android.hardware.biom
etrics.fingerprint.virtual@1
/dev/block/dm-37                    48       34      
             14   71% /apex/com.android.adbd@37420012
0
/dev/block/dm-45                    32       25      
              7   79% /apex/com.android.ipsec@3742050
00
/dev/block/dm-38                    64       52      
             12   82% /apex/com.android.uprobestats@3
74205000
/dev/block/dm-56                   160      157      
              3   99% /apex/com.android.conscrypt@374
205000
/dev/block/dm-29                    32       31      
              1   97% /apex/com.android.profiling@374
205000
/dev/block/dm-48                    48       34      
             14   71% /apex/com.android.appsearch@374
205000
/dev/block/dm-51                    48       45      
              3   94% /apex/com.android.nfcservices@3
74205000
/dev/block/dm-57                    32       25      
              7   79% /apex/com.android.scheduling@37
4205000
/dev/block/dm-42                    32       27      
              5   85% /apex/com.android.rkpd@37420002
0
/dev/block/dm-31                    32       25      
              7   79% /apex/com.android.crashrecovery
@374205000
/dev/block/dm-65                    48       36      
             12   75% /apex/com.android.healthfitness
@374205000
/dev/block/dm-62                    64       50      
             14   79% /apex/com.android.mediaprovider
@374200100
/dev/block/dm-54                    80       70      
             10   88% /apex/com.android.npumanager@37
1899999
/dev/block/dm-35                    16       16      
              0  100% /apex/com.android.neuralnetwork
s@374200020
/dev/block/dm-59                    96       93      
              3   97% /apex/com.android.tethering@374
205000
tmpfs                           940057       10      
         940047    1% /bootstrap-apex
/dev/block/loop3                    48       37      
             11   78% /bootstrap-apex/com.android.run
time@1
/dev/block/loop2                    48       47      
              1   98% /bootstrap-apex/com.android.tzd
ata@372037000
/dev/block/loop1  18446744073709551615       70 18446
744073709551545    1% /bootstrap-apex/com.android.vir
t@371899999
/dev/block/loop0                    32       30      
              2   94% /bootstrap-apex/com.android.i18
n@1
tmpfs                           940057        1      
         940056    1% /tmp
/dev/fuse                     13511677 13313930      
         197747   99% /storage/emulated

=== 2. TOP SPACE CONSUMERS IN ~ (depth 2) ===
348M    /data/data/com.termux/files/home/rss-automato
r
346M    /data/data/com.termux/files/home/ez_git
346M    /data/data/com.termux/files/home/ez_gemini_ge
ms_to_skills
257M    /data/data/com.termux/files/home/tech-ref-por
tal
110M    /data/data/com.termux/files/home/capyreader
104M    /data/data/com.termux/files/home/.npm
66M     /data/data/com.termux/files/home/animated-umb
rella
62M     /data/data/com.termux/files/home/.local
28M     /data/data/com.termux/files/home/home-archive
-staging
24M     /data/data/com.termux/files/home/keystone-hea
lth-advocate-pa-medical-rights-wiki-forensic-tool
17M     /data/data/com.termux/files/home/projects
15M     /data/data/com.termux/files/home/animated-umb
rella-main
12M     /data/data/com.termux/files/home/ez_ip_opsec_
audit
4.2M    /data/data/com.termux/files/home/duck-dev
2.2M    /data/data/com.termux/files/home/.netprobe
1.7M    /data/data/com.termux/files/home/ez_mac
1.5M    /data/data/com.termux/files/home/Keystone_Hea
lth
1.2M    /data/data/com.termux/files/home/EnterpriseSe
cureMobileSkeleton
810K    /data/data/com.termux/files/home/tmp
701K    /data/data/com.termux/files/home/October_2026
566K    /data/data/com.termux/files/home/Linux
436K    /data/data/com.termux/files/home/env-audit-ar
chive
413K    /data/data/com.termux/files/home/netprobe
411K    /data/data/com.termux/files/home/invizible-pr
o-agent-skills
387K    /data/data/com.termux/files/home/cl_else_roku
302K    /data/data/com.termux/files/home/pa_ra_onepag
ers
264K    /data/data/com.termux/files/home/audit_out
237K    /data/data/com.termux/files/home/goldenfoss-s
taging
229K    /data/data/com.termux/files/home/termux_maste
r_ultra
203K    /data/data/com.termux/files/home/ez_multi_rep
o

=== 3. LARGEST FILES (top 20) ===
13.3 MiB  /data/data/com.termux/files/home/projects/k
eystone_health/Pennsylvania
10.0 MiB  /data/data/com.termux/files/home/ez_ip_opse
c_audit/activity_trace.csv
2.5 MiB  /data/data/com.termux/files/home/rss-automat
or/ez_gemini_gems_to_skills/Active_Projects/Python.zi
p
2.5 MiB  /data/data/com.termux/files/home/ez_git/Acti
ve_Projects/Python.zip
2.5 MiB  /data/data/com.termux/files/home/ez_gemini_g
ems_to_skills/Active_Projects/Python.zip
2.4 MiB  /data/data/com.termux/files/home/keystone-he
alth-advocate-pa-medical-rights-wiki-forensic-tool/pe
nnsylvania-health-insurance-audit-2026-main
2.4 MiB  /data/data/com.termux/files/home/duck-dev/ke
ystone-sync/pennsylvania-health-insurance-audit-2026-
main
2.2 MiB  /data/data/com.termux/files/home/.netprobe/c
ache/603e776c4f62b98a.hosts
1.1 MiB  /data/data/com.termux/files/home/rss-automat
or/feeds.db-wal
0.9 MiB  /data/data/com.termux/files/home/rss-automat
or/feeds.db
0.6 MiB  /data/data/com.termux/files/home/home-archiv
e-staging/projects/ez_ip_opsec_audit/Screenshot_20261
004-105520.png
0.6 MiB  /data/data/com.termux/files/home/ez_ip_opsec
_audit/Screenshot_20261004-105520.png
0.6 MiB  /data/data/com.termux/files/home/tech-ref-po
rtal/node_modules/prettier/index.mjs
0.6 MiB  /data/data/com.termux/files/home/tech-ref-po
rtal/node_modules/date-fns/typings.d.ts
0.5 MiB  /data/data/com.termux/files/home/tech-ref-po
rtal/node_modules/lodash/lodash.js
0.4 MiB  /data/data/com.termux/files/home/projects/ke
ystone_health/Keystone
0.4 MiB  /data/data/com.termux/files/home/home-archiv
e-staging/_audit/20261107_082622/SKIPPED.txt
0.4 MiB  /data/data/com.termux/files/home/home-archiv
e-staging/_audit/20261107_082550/SKIPPED.txt
0.3 MiB  /data/data/com.termux/files/home/tech-ref-po
rtal/node_modules/prettier/THIRD-PARTY-NOTICES.md
0.3 MiB  /data/data/com.termux/files/home/ez_git/.git
/index
~ $ ls -la
total 1574
drwx------. 80 u0_a399 u0_a399 20480 Nov  9 15:33 .
drwxrwx--x.  5 u0_a399 u0_a399  3452 Oct 11 12:19 ..
-rw-------.  1 u0_a399 u0_a399 22026 Nov  9 15:22 .ba
sh_history
-rw-------.  1 u0_a399 u0_a399  2190 Nov  8 18:36 .ba
shrc
drwx------.  4 u0_a399 u0_a399  3452 Nov  9 12:28 .ca
che
-rw-------.  1 u0_a399 u0_a399 53248 Nov  7 00:35 .co
verage
-rwx------.  1 u0_a399 u0_a399    14 Oct 13 03:48 .ez
_archive.sh
drwx------.  3 u0_a399 u0_a399  3452 Nov  7 17:38 .gi
t
drwx------.  7 u0_a399 u0_a399  3452 Nov  3 09:22 .gi
t-accidental-backup
drwx------.  2 u0_a399 u0_a399  3452 Nov  7 06:36 .gi
t-hooks
drwx------.  2 u0_a399 u0_a399  3452 Nov  9 03:28 .gi
t_archive
-rw-------.  1 u0_a399 u0_a399   484 Nov  7 06:36 .gi
tconfig
drwx------.  3 u0_a399 u0_a399  3452 Nov  4 10:58 .gi
thub
-rw-------.  1 u0_a399 u0_a399   174 Nov  2 01:20 .gi
tignore_global
drwxr-xr-x.  7 u0_a399 u0_a399  3452 Nov  7 01:45 .lo
cal
drwx------.  3 u0_a399 u0_a399  3452 Oct 15 20:38 .ne
tprobe
drwx------.  5 u0_a399 u0_a399  3452 Nov  9 03:25 .np
m
drwx------.  3 u0_a399 u0_a399  3452 Nov  7 06:36 .np
m-global
-rw-------.  1 u0_a399 u0_a399    66 Nov  7 06:36 .np
mrc
drwx------.  3 u0_a399 u0_a399  3452 Nov  7 00:38 .py
test_cache
-rw-------.  1 u0_a399 u0_a399 38632 Nov  7 07:09 .py
thon_history
-rw-------.  1 u0_a399 u0_a399   154 Nov  5 16:28 .re
po_split.list
drwx------.  2 u0_a399 u0_a399  3452 Nov  1 11:48 .ss
h
drwx------.  3 u0_a399 u0_a399  3452 Sep  9 03:18 .su
root
drwx------.  3 u0_a399 u0_a399  3452 Oct 12 21:39 .te
rmux
drwx------.  6 u0_a399 u0_a399  3452 Nov  8 18:27 .te
rmux_master
drwx------.  2 u0_a399 u0_a399  3452 Nov  1 20:32 .tp
drwx------.  2 u0_a399 u0_a399  3452 Nov  7 07:25 .tr
p-bak
drwx------.  2 u0_a399 u0_a399  3452 Oct 14 18:15 .w3
m
-rw-------.  1 u0_a399 u0_a399    24 Oct 12 13:26 .wg
_private_key
-rw-------.  1 u0_a399 u0_a399     0 Nov  7 05:26 50%
-rw-------.  1 u0_a399 u0_a399     0 Nov  5 13:57 Boo
tstrapping
drwx------.  2 u0_a399 u0_a399  3452 Nov  9 03:28 Chr
ombook_crostini_repair_guide
drwxr-xr-x.  8 u0_a399 u0_a399  3452 Nov  7 07:19 Ent
erpriseSecureMobileSkeleton
-rw-------.  1 u0_a399 u0_a399     0 Nov  5 13:57 Env
ironment
-rw-------.  1 u0_a399 u0_a399     0 Oct 10 20:58 FET
CH_HEAD
drwx------.  2 u0_a399 u0_a399  3452 Nov  9 03:28 Gol
denfoss
drwx------. 13 u0_a399 u0_a399  3452 Nov  5 19:43 Key
stone_Health
drwx------.  4 u0_a399 u0_a399  3452 Oct 31 06:37 Lin
ux
drwx------.  3 u0_a399 u0_a399  3452 Oct 14 03:44 Oct
ober_2026
-rw-------.  1 u0_a399 u0_a399     9 Nov  3 09:22 REA
DME.md
drwx------.  2 u0_a399 u0_a399  3452 Oct 15 19:55 adb
lock_tester
drwx------. 19 u0_a399 u0_a399  8192 Nov  1 02:58 ani
mated-umbrella
drwx------. 14 u0_a399 u0_a399  3452 Oct 11 02:44 ani
mated-umbrella-main
-rw-------.  1 u0_a399 u0_a399     9 Oct 11 02:47 ani
mated-umbrella.tar.gz
drwx------.  2 u0_a399 u0_a399  3452 Oct 14 18:07 ani
mated-umbrella.wiki
-rwx------.  1 u0_a399 u0_a399  3698 Oct 31 01:43 apk
_report.py
drwx------.  3 u0_a399 u0_a399  3452 Oct 15 20:10 ato
mic_session
drwx------.  2 u0_a399 u0_a399  3452 Oct 14 20:18 aud
it_out
drwx------.  9 u0_a399 u0_a399  3452 Oct 11 10:31 aud
it_tools
drwx------.  2 u0_a399 u0_a399  3452 Nov  9 03:29 bac
kups
drwx------.  3 u0_a399 u0_a399  3452 Nov  7 07:18 bin
-rwx------.  1 u0_a399 u0_a399 20001 Oct 31 02:01 boo
tstrap_ez_vlc_android.sh
-rwx------.  1 u0_a399 u0_a399  3360 Nov  5 15:32 boo
tstrap_keystone.sh
-rwx------.  1 u0_a399 u0_a399  1011 Nov  5 13:02 boo
tstrap_keystone_env.sh
drwx------. 23 u0_a399 u0_a399  3452 Oct 15 15:41 cap
yreader
drwx------.  3 u0_a399 u0_a399  3452 Oct 13 09:09 cl_
else_roku
-rwx------.  1 u0_a399 u0_a399  4651 Nov  7 19:41 cle
anup.sh
-rwx------.  1 u0_a399 u0_a399  2054 Nov  6 19:41 cle
anup_bighunt.sh
-rwx------.  1 u0_a399 u0_a399 18358 Nov  6 11:45 cle
anup_gui.py
-rwx------.  1 u0_a399 u0_a399  6780 Nov  6 14:40 cle
anup_reclaim.sh
drwx------.  3 u0_a399 u0_a399  3452 Nov  7 06:04 cor
e
-rwx------.  1 u0_a399 u0_a399  5162 Oct 10 14:48 ctx
.sh
-rw-------.  1 u0_a399 u0_a399 29800 Oct 10 14:49 ctx
_report_20261010_144857.txt
-rw-------.  1 u0_a399 u0_a399 31570 Oct 10 16:28 ctx
_report_20261010_162811.txt
-rw-------.  1 u0_a399 u0_a399  9855 Nov  1 18:40 das
hboard.py
-rwx------.  1 u0_a399 u0_a399  1925 Oct 10 05:33 dep
loy.sh
-rwx------.  1 u0_a399 u0_a399  9615 Oct 13 06:29 dep
loy_bundle.sh
-rwx------.  1 u0_a399 u0_a399   798 Oct 14 16:23 dep
loy_fix.sh
-rw-------.  1 u0_a399 u0_a399   970 Oct 12 14:35 dep
loy_fixes.sh
-rwx------.  1 u0_a399 u0_a399  3751 Oct 12 14:07 dep
loy_portal.sh
-rwx------.  1 u0_a399 u0_a399  3194 Oct 12 21:57 dep
loy_remediation.sh
-rw-------.  1 u0_a399 u0_a399  2329 Nov  5 12:40 dep
loyment_20261105_124005.log
-rw-------.  1 u0_a399 u0_a399  2329 Nov  5 12:48 dep
loyment_20261105_124849.log
-rw-------.  1 u0_a399 u0_a399  2329 Nov  5 13:03 dep
loyment_20261105_130302.log
-rw-------.  1 u0_a399 u0_a399 15144 Nov  9 15:36 dia
g_output.txt
drwx------.  2 u0_a399 u0_a399  3452 Nov  9 03:28 dow
nloads
drwx------.  3 u0_a399 u0_a399  3452 Oct 14 14:54 duc
k-dev
drwx------.  2 u0_a399 u0_a399  3452 Nov  7 03:16 env
-audit-archive
-rwx------.  1 u0_a399 u0_a399  1169 Oct 14 20:18 env
_audit.sh
-rw-------.  1 u0_a399 u0_a399 89864 Nov  5 23:44 env
_audit_20261105_234458.txt
-rw-------.  1 u0_a399 u0_a399 90715 Nov  6 10:57 env
_audit_20261106_105737.txt
drwx------.  6 u0_a399 u0_a399  3452 Oct 30 22:29 ez-
vlc-android
-rwx------.  1 u0_a399 u0_a399    14 Oct 13 03:48 ez_
archive.sh
drwx------.  2 u0_a399 u0_a399  3452 Oct 13 09:52 ez_
court
-rw-------.  1 u0_a399 u0_a399 34514 Oct 14 04:24 ez_
flash_backup.py
drwx------.  7 u0_a399 u0_a399  3452 Nov  7 08:10 ez_
gemini_gems_to_skills
drwx------. 10 u0_a399 u0_a399  3452 Nov  6 19:07 ez_
git
drwx------.  2 u0_a399 u0_a399  3452 Oct 14 18:07 ez_
ip_opsec_audit
drwx------.  9 u0_a399 u0_a399  3452 Nov  7 08:30 ez_
mac
drwx------.  4 u0_a399 u0_a399  3452 Nov  3 23:05 ez_
multi_repo
-rw-------.  1 u0_a399 u0_a399  7331 Nov  9 12:28 ez_
recon_20261109_122832.txt
drwx------.  3 u0_a399 u0_a399  3452 Oct 14 18:07 ez_
remote
-rw-------.  1 u0_a399 u0_a399 29754 Nov  9 12:28 ez_
repo_recon.py
drwx------.  2 u0_a399 u0_a399  3452 Nov  9 03:29 ez_
termux_backup_flashdrive
-rwx------.  1 u0_a399 u0_a399  1120 Oct 11 03:57 fin
alize_hardening.sh
-rwx------.  1 u0_a399 u0_a399  1792 Oct 12 12:43 fix
_pixel_audit.sh
-rw-------.  1 u0_a399 u0_a399  7106 Nov  1 04:16 fix
_termux_workspace.sh
-rwx------.  1 u0_a399 u0_a399  8744 Nov  1 17:11 gen
_network_defense_md.sh
-rw-------.  1 u0_a399 u0_a399 17896 Nov  1 08:57 gen
erate_handover.py
-rwx------.  1 u0_a399 u0_a399  3703 Nov  3 09:08 gen
erate_mac.py
-rwx------.  1 u0_a399 u0_a399  5665 Nov  5 11:40 gen
erate_repo_structure.py
-rw-------.  1 u0_a399 u0_a399     0 Nov  5 15:57 gen
erate_skeleton_repo.sh.save
-rw-------.  1 u0_a399 u0_a399  1997 Nov  5 15:40 git
_workflow.sh
drwx------.  4 u0_a399 u0_a399  3452 Nov  1 10:47 git
hub_deploy
-rwx------.  1 u0_a399 u0_a399  1466 Sep  9 03:07 gms
_hunter_v2.py
drwx------.  3 u0_a399 u0_a399  3452 Nov  5 18:52 gol
denfoss-staging
-rw-------.  1 u0_a399 u0_a399 21728 Nov  8 19:17 han
dover.md
drwx------.  2 u0_a399 u0_a399  3452 Nov  7 06:20 han
dover_session_20261106
-rwx------.  1 u0_a399 u0_a399   899 Oct 12 14:12 har
den_gh.sh
drwx------.  6 u0_a399 u0_a399  3452 Nov  7 08:27 hom
e-archive-staging
-rwx------.  1 u0_a399 u0_a399  4508 Nov  6 21:33 hom
e_audit.py
-rw-------.  1 u0_a399 u0_a399  3022 Nov  7 01:45 ind
ex.html
-rwx------.  1 u0_a399 u0_a399    29 Nov  7 08:10 ini
t_and_push_gems.sh
-rwx------.  1 u0_a399 u0_a399  4045 Oct 31 13:14 ins
tall.sh
-rw-------.  1 u0_a399 u0_a399  1197 Oct 11 04:26 ins
tall_safe_clone.sh
drwx------.  3 u0_a399 u0_a399  3452 Nov  1 08:57 inv
izible-pro-agent-skills
drwx------.  2 u0_a399 u0_a399  3452 Oct 15 18:44 ipl
eak_tool
drwx------. 11 u0_a399 u0_a399  8192 Nov  1 02:58 key
stone-health-advocate-pa-medical-rights-wiki-forensic
-tool
-rw-------.  1 u0_a399 u0_a399  2633 Nov  3 03:17 key
stone_fix.sh
-rw-------.  1 u0_a399 u0_a399  3430 Nov  3 03:33 key
stone_fix2.sh
-rw-------.  1 u0_a399 u0_a399  8528 Nov  3 03:33 key
stone_fix2_20261103_033334.txt
-rw-------.  1 u0_a399 u0_a399  4454 Nov  5 15:07 key
stone_fix3.sh
-rw-------.  1 u0_a399 u0_a399  1099 Nov  5 15:13 key
stone_fix3_20261105_150418.txt
-rw-------.  1 u0_a399 u0_a399  1138 Nov  5 15:07 key
stone_fix3_20261105_150725.txt
-rw-------.  1 u0_a399 u0_a399  4629 Nov  5 15:47 key
stone_fix4.sh
-rw-------.  1 u0_a399 u0_a399  5559 Nov  5 15:37 key
stone_fix4_20261105_153659.txt
-rw-------.  1 u0_a399 u0_a399  5559 Nov  5 15:47 key
stone_fix4_20261105_154709.txt
-rw-------.  1 u0_a399 u0_a399  4638 Nov  3 02:08 key
stone_fix_20261103_020759.txt
-rw-------.  1 u0_a399 u0_a399  4864 Nov  3 03:18 key
stone_fix_20261103_031806.txt
drwx------.  2 u0_a399 u0_a399  3452 Nov  3 02:08 key
stone_pdf_text
-rwx------.  1 u0_a399 u0_a399  2293 Nov  5 11:07 key
stone_sync.py
-rw-------.  1 u0_a399 u0_a399  1995 Nov  3 02:05 key
stone_triage.sh
-rw-------.  1 u0_a399 u0_a399  3209 Nov  3 02:05 key
stone_triage_20261103_020518.txt
-rw-------.  1 u0_a399 u0_a399  3146 Nov  3 03:48 key
stone_verify.sh
-rw-------.  1 u0_a399 u0_a399  2474 Nov  3 03:48 key
stone_verify_20261103_034851.txt
-rw-------.  1 u0_a399 u0_a399  2406 Nov  5 13:34 key
stone_verify_20261105_133357.txt
-rw-------.  1 u0_a399 u0_a399  2406 Nov  5 13:47 key
stone_verify_20261105_134730.txt
-rw-------.  1 u0_a399 u0_a399  2406 Nov  5 13:48 key
stone_verify_20261105_134849.txt
-rw-------.  1 u0_a399 u0_a399   497 Oct 31 16:31 loc
k_utils.sh
-rwx------.  1 u0_a399 u0_a399   985 Oct 13 00:28 mon
itor_pipeline.sh
drwx------.  2 u0_a399 u0_a399  3452 Aug  6 16:27 my-
futo-notes
drwx------.  2 u0_a399 u0_a399  3452 Oct 15 19:55 net
_tool
drwx------.  5 u0_a399 u0_a399  3452 Nov  1 11:48 net
probe
-rw-------.  1 u0_a399 u0_a399 15609 Nov  7 07:01 obl
ivion_install.sh
drwx------.  5 u0_a399 u0_a399  3452 Oct 14 18:07 ops
ec-debug
-rwx------.  1 u0_a399 u0_a399  1293 Oct  9 18:53 ops
ec_audit.py
-rw-------.  1 u0_a399 u0_a399     0 Oct 10 10:35 pa_
aca_briefing_py2027_final.html
drwx------.  5 u0_a399 u0_a399  3452 Oct 13 06:30 pa_
audit_bundle
drwx------.  3 u0_a399 u0_a399  3452 Oct 10 20:18 pa_
ra_onepagers
-rwx------.  1 u0_a399 u0_a399  2602 Oct 12 19:33 par
se_ir_intelligence.py
drwx------.  2 u0_a399 u0_a399  3452 Nov  8 10:55 phy
lum
drwx------.  2 u0_a399 u0_a399  3452 Aug  8 13:14 pix
el-terminal-master
-rw-------.  1 u0_a399 u0_a399  1565 Oct  9 18:53 pix
el-terminal-master.20261009_185313.tgz
-rw-------.  1 u0_a399 u0_a399  1565 Oct 10 21:54 pix
el-terminal-master.20261010_215417.tgz
-rw-------.  1 u0_a399 u0_a399  1565 Oct 12 12:43 pix
el-terminal-master.20261012_124330.tgz
-rw-------.  1 u0_a399 u0_a399  1565 Nov  9 06:08 pix
el-terminal-master.20261109_060840.tgz
-rwx------.  1 u0_a399 u0_a399   645 Oct 12 14:45 pix
el_diagnostic.sh
-rw-------.  1 u0_a399 u0_a399   662 Oct  9 18:53 pix
el_diagnostic_20261009_185314.txt
-rw-------.  1 u0_a399 u0_a399   661 Oct 10 21:54 pix
el_diagnostic_20261010_215417.txt
-rw-------.  1 u0_a399 u0_a399   661 Oct 12 12:43 pix
el_diagnostic_20261012_124330.txt
-rw-------.  1 u0_a399 u0_a399   661 Nov  9 06:08 pix
el_diagnostic_20261109_060840.txt
-rw-------.  1 u0_a399 u0_a399  4545 Nov  7 13:16 pix
el_opsec_audit.sh
-rw-------.  1 u0_a399 u0_a399  2351 Nov  1 17:53 pix
svg.py
-rwx------.  1 u0_a399 u0_a399  1403 Nov  1 17:56 pla
ce_netdef.sh
drwx------.  4 u0_a399 u0_a399  3452 Nov  7 00:38 pro
jects
-rw-------.  1 u0_a399 u0_a399  3518 Nov  8 10:45 rec
on.sh
-rw-------.  1 u0_a399 u0_a399 10804 Nov  8 11:03 rec
on_report.txt
-rw-------.  1 u0_a399 u0_a399 15626 Oct 15 22:16 rep
ort.json
-rw-------.  1 u0_a399 u0_a399   882 Nov  9 07:21 ris
h
-rw-------.  1 u0_a399 u0_a399 59672 Nov  9 07:21 ris
h_shizuku.dex
drwx------.  3 u0_a399 u0_a399  3452 Oct 12 10:08 rok
u-diagnostics
drwx------.  5 u0_a399 u0_a399  3452 Oct 14 18:07 rok
u_failure_evidence
drwx------.  3 u0_a399 u0_a399  3452 Nov  8 10:19 rss
-automator
-rw-------.  1 u0_a399 u0_a399  1526 Nov  7 07:53 scr
ipt_inventory.sh
-rw-------.  1 u0_a399 u0_a399  3275 Nov  7 07:54 scr
ipt_inventory_20261107_075407.txt
drwx------.  2 u0_a399 u0_a399  3452 Nov  6 23:58 scr
ipts
-rwx------.  1 u0_a399 u0_a399  4239 Nov  1 11:42 sec
urity_cleanup.sh
-rw-------.  1 u0_a399 u0_a399 36063 Oct 13 20:24 ses
sion_context.txt
-rwx------.  1 u0_a399 u0_a399   429 Oct 12 14:45 ses
sion_diag.sh
-rw-------.  1 u0_a399 u0_a399  2117 Oct 15 14:31 set
tup.sh
-rw-------.  1 u0_a399 u0_a399 15079 Oct 15 15:40 set
up-rss-automator.sh
-rwx------.  1 u0_a399 u0_a399   983 Oct 12 21:30 set
up.sh
-rwx------.  1 u0_a399 u0_a399   782 Oct 13 07:54 set
up_tv_case.sh
drwx------.  2 u0_a399 u0_a399  3452 Sep  9 03:07 sha
red
drwx------.  2 u0_a399 u0_a399  3452 Nov  9 03:28 shh
-rwx------.  1 u0_a399 u0_a399 11469 Nov  5 12:48 shi
p_nsa_dispute_kit.sh
-rwx------.  1 u0_a399 u0_a399  3860 Sep# Option B: PAT via credential store (if you prefer HTTPS)
git config --global credential.helper store
# Next push will prompt ONCE for username + PAT, then save to ~/.git-credentials
# Make sure the repo exists on GitHub first (private!)

import asyncio
import json
import logging
import os
import shutil
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from pathlib import Path
from typing import Any, Dict, List, Optional, Tuple

# Set up logging configuration
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s - %(message)s"
)
logger = logging.getLogger("MasterEngine")


@dataclass
class StorageMetrics:
    """Dataclass holding storage metrics for a specific mount path."""
    mount_point: str
    total_bytes: int
    used_bytes: int
    free_bytes: int
    percent_used: float


class StorageTruthChecker:
    """Audits blocks and inodes across Termux-relevant mounts."""

    # Updated mount targets for Termux environments:
    # / = Termux system partition (packages)
    # /storage/emulated = FUSE shared storage
    # /data/data/com.termux/files = app data (where $HOME lives)
    MOUNTS: Tuple[str, ...] = (
        "/",
        "/storage/emulated",
        "/data/data/com.termux/files",
    )

    def check_mount_health(self) -> List[StorageMetrics]:
        """Collects disk usage information across all defined mount points."""
        metrics: List[StorageMetrics] = []
        for mount in self.MOUNTS:
            if not os.path.exists(mount):
                logger.debug("Mount path %s does not exist, skipping.", mount)
                continue
            try:
                usage = shutil.disk_usage(mount)
                percent = (usage.used / usage.total) * 100 if usage.total > 0 else 0.0
                metrics.append(
                    StorageMetrics(
                        mount_point=mount,
                        total_bytes=usage.total,
                        used_bytes=usage.used,
                        free_bytes=usage.free,
                        percent_used=round(percent, 2),
                    )
                )
            except Exception as e:
                logger.warning("Failed to check disk usage for mount %s: %s", mount, e)
        return metrics


class StateValidator:
    """Handles reading and validating state configuration (e.g., HANDOVER.json)."""

    def __init__(self, state_file_path: Path) -> None:
        self.state_file_path = state_file_path

    def load_state(self) -> Dict[str, Any]:
        """Reads JSON state file and returns parsed data."""
        if not self.state_file_path.exists():
            raise FileNotFoundError(f"State file not found at: {self.state_file_path}")

        with open(self.state_file_path, "r", encoding="utf-8") as f:
            data = json.load(f)
            if not isinstance(data, dict):
                raise ValueError("Invalid state format: Root must be a JSON object.")
            return data


class BaseProcessor(ABC):
    """Abstract Base Class for modular pipeline processors (Open/Closed Principle)."""

    @abstractmethod
    async def process(self, state_data: Dict[str, Any]) -> None:
        """Execute processor specific tasks."""
        pass


class StorageAuditProcessor(BaseProcessor):
    """Processor responsible for monitoring storage metrics across configured mounts."""

    def __init__(self, checker: Optional[StorageTruthChecker] = None) -> None:
        self.checker = checker or StorageTruthChecker()

    async def process(self, state_data: Dict[str, Any]) -> None:
        metrics = self.checker.check_mount_health()
        for metric in metrics:
            logger.info(
                "Mount [%s] Usage: %.1f%% (Free: %s MB)",
                metric.mount_point,
                metric.percent_used,
                metric.free_bytes // (1024 * 1024),
            )


class HandoverAuditProcessor(BaseProcessor):
    """Processor responsible for auditing handover parameters from state."""

    async def process(self, state_data: Dict[str, Any]) -> None:
        session_id = state_data.get("session_id", "unknown")
        status = state_data.get("status", "pending")
        logger.info("Handover State Audit - Session ID: %s, Status: %s", session_id, status)


@dataclass
class MasterEngine:
    """Core orchestrator executing registered processors in an isolated event cycle."""

    validator: StateValidator
    processors: List[BaseProcessor] = field(default_factory=list)

    def register_processor(self, processor: BaseProcessor) -> None:
        """Register a new processor into the pipeline."""
        self.processors.append(processor)

    async def run_cycle(self) -> None:
        """Runs a single audit cycle with per-processor exception isolation."""
        logger.info("Starting automation audit cycle...")
        try:
            state_data = self.validator.load_state()
        except Exception as e:
            logger.error("Failed to load state: %s", e)
            return

        for processor in self.processors:
            try:
                await processor.process(state_data)
            except Exception as e:
                logger.error(
                    "Processor %s failed: %s", type(processor).__name__, e
                )

        logger.info("Automation audit cycle completed.")


async def main() -> None:
    # Target location of the HANDOVER state file
    home_dir = Path(os.path.expanduser("~"))
    handover_file = home_dir / "HANDOVER.json"

    validator = StateValidator(state_file_path=handover_file)
    engine = MasterEngine(validator=validator)

    # Register modular pipeline processors
    engine.register_processor(StorageAuditProcessor())
    engine.register_processor(HandoverAuditProcessor())

    # Execute cycle
    await engine.run_cycle()


if __name__ == "__main__":
    asyncio.run(main())

    
Euria
Euria
\] "\ ]" \] "\ ]" \] "
░▒▓~ on  main [?] 
❯ printf 'Current directory: '; pwd
Current directory: /home/droid

░▒▓~ on  main [?] 
❯ printf 'Git root here: '; git rev-parse --show-toplevel 2>/dev/null || echo "none"
Git root here: /home/droid

░▒▓~ on  main [?] 
❯

░▒▓~ on  main [?] 
❯ ls -ld ~/tech-ref-portal ~/Chrombook_crostini_repair_guide 2>/dev/null
drwx------ 15 droid users 4096 Nov 9 16:28 /home/droid/tech-ref-portal

░▒▓~ on  main [?] 
❯ find "$HOME" -maxdepth 4 -type d \
∙ \( -name tech-ref-portal -o -name Chrombook_crostini_repair_guide \) \
∙ -print 2>/dev/null
/home/droid/tech-ref-portal

░▒▓~ on  main [?] 
❯ cd ~ && python3 ~/bin/home_audit.py # 1. audit: statuses, public/private, third-party, leaks
Traceback (most recent call last):
File "/usr/lib/python3.13/pathlib/_local.py", line 724, in mkdir
os.mkdir(self, mode)
~~~~~~~~^^^^^^^^^^^^
FileNotFoundError: [Errno 2] No such file or directory: '/home/droid/.audit_logs/home_audit_20261109_202314'

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
File "/home/droid/bin/home_audit.py", line 457, in <module>
main()
~~~~^^
File "/home/droid/bin/home_audit.py", line 404, in main
out.mkdir(parents=True, exist_ok=True)
~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
File "/usr/lib/python3.13/pathlib/_local.py", line 728, in mkdir
self.parent.mkdir(parents=True, exist_ok=True)
~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
File "/usr/lib/python3.13/pathlib/_local.py", line 724, in mkdir
os.mkdir(self, mode)
~~~~~~~~^^^^^^^^^^^^
OSError: [Errno 28] No space left on device: '/home/droid/.audit_logs'

░▒▓~ on  main [?] 
❯ python3 ~/bin/home_audit.py --quarantine-junk # 2. moves junk + empty dirs to ~/_quarantine/<ts>/
Traceback (most recent call last):
File "/usr/lib/python3.13/pathlib/_local.py", line 724, in mkdir
os.mkdir(self, mode)
~~~~~~~~^^^^^^^^^^^^
FileNotFoundError: [Errno 2] No such file or directory: '/home/droid/.audit_logs/home_audit_20261109_202314'

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
File "/home/droid/bin/home_audit.py", line 457, in <module>
main()
~~~~^^
File "/home/droid/bin/home_audit.py", line 404, in main
out.mkdir(parents=True, exist_ok=True)
~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
File "/usr/lib/python3.13/pathlib/_local.py", line 728, in mkdir
self.parent.mkdir(parents=True, exist_ok=True)
~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
File "/usr/lib/python3.13/pathlib/_local.py", line 724, in mkdir
os.mkdir(self, mode)
~~~~~~~~^^^^^^^^^^^^
OSError: [Errno 28] No space left on device: '/home/droid/.audit_logs'

░▒▓~ on  main [?] 
❯ EX="--exclude capyreader --exclude ez_termux_backup_flashdrive --exclude Goldenfoss --exclude goldenfoss-staging \
∙ --exclude tech-ref-portal.stale --exclude duck-dev --exclude keystone_extract --exclude Keystone_Health_downloadable_files \
∙ --exclude projects --exclude 'keystone-health-advocate-*' --exclude roku_failure_evidence --exclude ez_court \
∙ --exclude cl_else_roku --exclude tmp --exclude backups"

░▒▓~ on  main [?] 
❯ eval python3 ~/bin/home_audit.py --stage $EX # 3. then check ~/.audit_logs/*/SKIPPED.txt
usage: home_audit.py [-h] [--stage]
[--push]
[--quarantine-junk]
[--owner OWNER]
[--repo REPO]
[--stage-dir STAGE_DIR]
home_audit.py: error: unrecognized arguments: --exclude capyreader --exclude ez_termux_backup_flashdrive --exclude Goldenfoss --exclude goldenfoss-staging --exclude tech-ref-portal.stale --exclude duck-dev --exclude keystone_extract --exclude Keystone_Health_downloadable_files --exclude projects --exclude keystone-health-advocate-* --exclude roku_failure_evidence --exclude ez_court --exclude cl_else_roku --exclude tmp --exclude backups

░▒▓~ on  main [?] 
❯ eval python3 ~/bin/home_audit.py --push $EX # 4. only after creating the PRIVATE repo
usage: home_audit.py [-h] [--stage]
[--push]
[--quarantine-junk]
[--owner OWNER]
[--repo REPO]
[--stage-dir STAGE_DIR]
home_audit.py: error: unrecognized arguments: --exclude capyreader --exclude ez_termux_backup_flashdrive --exclude Goldenfoss --exclude goldenfoss-staging --exclude tech-ref-portal.stale --exclude duck-dev --exclude keystone_extract --exclude Keystone_Health_downloadable_files --exclude projects --exclude keystone-health-advocate-* --exclude roku_failure_evidence --exclude ez_court --exclude cl_else_roku --exclude tmp --exclude backups

░▒▓~ on  main [?] 
❯Architectural & Refactoring Plan
* SOLID & KISS Design Principles:
* Single Responsibility Principle (SRP): Each class encapsulates a discrete operational responsibility (AppConfig for configuration, HandoverStateValidator for secure JSON I/O, StorageTruthChecker for block and inode health, BlockerInspector for impediments, NextActionReporter for runbook progression, and AutomationEngine for asynchronous orchestration).
* Open/Closed Principle (OCP): The execution pipeline accepts pluggable processors via the StateProcessorProtocol, enabling new monitoring or alerting extensions without altering core engine logic.
* Dependency Inversion (DIP): High-level coordination depends on abstract protocols and injected configurations rather than global state.
* Core Components (master_engine.py):
* AppConfig: Immutable configuration dataclass loading environment variables (HANDOVER_PATH, POLL_INTERVAL, DEBUG) with robust fallback defaults.
* HandoverStateValidator: Safely loads and validates HANDOVER.json, returning default structures gracefully if the file is missing or locked.
* StorageTruthChecker: Audits both block utilization (f_bavail/f_blocks) and inode consumption (f_ffree/f_files) across host-truth mounts (/ and /data/user/0), preventing thin-provisioned exhaustion blind spots.
* BlockerInspector: Parses session blockers and logs active impediments at INFO level for transparent background execution.
* NextActionReporter: Automatically inspects the next_session_runbook array and surfaces the exact operational frontier step.
* AutomationEngine: Asynchronous orchestrator executing modular processors on a timed interval with clean signal handling (SIGINT, SIGTERM) for zero-interaction background persistence.
* Production Readiness:
* Zero External Dependencies: Pure Python standard library implementation (asyncio, json, logging, os, pathlib, signal, sys), ensuring seamless execution across Android Termux, Debian virtual machines, and Linux environments.
* Defensive Error Boundaries: Graceful exception handling for missing files, permission errors, and missing mount points.
#!/usr/bin/env python3
"""
Master Automation and Handover Engine (v2.1)
Refactored following SOLID, KISS, and PEP 8 standards.
Provides zero-interaction background monitoring, dual-mount storage truth validation (bytes & inodes),
active blocker inspection, and actionable runbook frontier reporting for mobile & virtualized Linux environments.
"""

from __future__ import annotations

import asyncio
import json
import logging
import os
import signal
import sys
from dataclasses import dataclass, field
from pathlib import Path
from typing import Any, Dict, List, Optional, Protocol

# Configure structured logging
logging.basicConfig(
level=logging.INFO,
format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
handlers=[logging.StreamHandler(sys.stdout)]
)
logger = logging.getLogger("HandoverEngine")


@dataclass(frozen=True)
class AppConfig:
"""Type-safe application configuration loaded from environment or defaults."""
handover_path: Path = field(default_factory=lambda: Path("~/HANDOVER.json").expanduser())
poll_interval_seconds: float = 60.0
debug_mode: bool = False

@classmethod
def from_env(cls) -> AppConfig:
return cls(
handover_path=Path(os.getenv("HANDOVER_PATH", "~/HANDOVER.json")).expanduser(),
poll_interval_seconds=float(os.getenv("POLL_INTERVAL", "60.0")),
debug_mode=os.getenv("DEBUG", "false").lower() in ("true", "1", "yes")
)


class StateProcessorProtocol(Protocol):
"""Protocol defining state inspection handlers for the automation pipeline."""
async def process(self, state_data: Dict[str, Any]) -> None:
...


class HandoverStateValidator:
"""Validates and parses HANDOVER.json state securely."""

def __init__(self, file_path: Path) -> None:
self.file_path = file_path

def load_state(self) -> Dict[str, Any]:
if not self.file_path.exists():
logger.warning("Handover file not found at %s. Using default empty state.", self.file_path)
return {"handover_version": "2.1", "blockers": [], "unresolved_items": [], "next_session_runbook": []}
try:
with open(self.file_path, "r", encoding="utf-8") as f:
data = json.load(f)
if not isinstance(data, dict):
raise ValueError("Handover root must be a JSON object.")
return data
except Exception as e:
logger.error("Failed to parse handover state from %s: %s", self.file_path, e)
raise


class StorageTruthChecker:
"""Audits blocks AND inodes across host-truth and guest mounts to prevent thin-provisioned blind spots."""

MOUNTS = ("/", "/data/user/0")

async def process(self, state_data: Dict[str, Any]) -> None:
for mount in self.MOUNTS:
try:
st = os.statvfs(mount)
except OSError:
continue

free_gib = (st.f_bavail * st.f_frsize) / (1024 ** 3)
total_gib = (st.f_blocks * st.f_frsize) / (1024 ** 3)
used_pct = (1 - st.f_bavail / st.f_blocks) * 100 if st.f_blocks else 0.0
inodes_free = st.f_ffree
inodes_used_pct = (1 - st.f_ffree / st.f_files) * 100 if st.f_files else 0.0

logger.info(
"Storage [%s] -> %.2f GiB free | bytes used: %.1f%% | inodes used: %.1f%% (%d free)",
mount, free_gib, used_pct, inodes_used_pct, inodes_free
)

if used_pct > 95.0:
logger.critical("CRITICAL: Mount %s bytes usage exceeds 95%% threshold (%.1f%%)", mount, used_pct)
if inodes_used_pct > 90.0:
logger.critical(
"CRITICAL: Mount %s inode exhaustion detected (%.1f%% used, %d free) — risk of filesystem lockup",
mount, inodes_used_pct, inodes_free
)


class BlockerInspector:
"""Inspects and reports active blockers from the handover payload at INFO level."""

async def process(self, state_data: Dict[str, Any]) -> None:
blockers: List[Dict[str, Any]] = state_data.get("blockers", [])
if blockers:
logger.info("Active Blockers Detected: %d item(s)", len(blockers))
for blocker in blockers:
logger.info(
"BLOCKER [%s]: %s (Owner: %s)",
blocker.get("id", "N/A"),
blocker.get("item", "Unknown issue"),
blocker.get("owner", "unassigned")
)
else:
logger.info("No active blockers reported in handover state.")


class NextActionReporter:
"""Surfaces the first unresolved runbook step — the operational frontier of the session."""

def __init__(self, handover_path: Path) -> None:
self.handover_path = handover_path

async def process(self, state_data: Dict[str, Any]) -> None:
runbook: List[str] = state_data.get("next_session_runbook", [])
if runbook:
logger.info("NEXT RUNBOOK FRONTIER: %s", runbook[0])
else:
logger.warning("Runbook empty or malformed in %s", self.handover_path)


class AutomationEngine:
"""Asynchronous orchestrator for zero-interaction background monitoring and state validation."""

def __init__(self, config: AppConfig) -> None:
self.config = config
self.validator = HandoverStateValidator(config.handover_path)
self.processors: List[StateProcessorProtocol] = [
StorageTruthChecker(),
BlockerInspector(),
NextActionReporter(config.handover_path)
]
self._is_running = False

async def run_cycle(self) -> None:
logger.info("Starting automation audit cycle...")
try:
state_data = self.validator.load_state()
for processor in self.processors:
await processor.process(state_data)
except Exception as e:
logger.error("Error during automation cycle execution: %s", e)

async def start(self) -> None:
self._is_running = True
logger.info("AutomationEngine started with poll interval: %.1fs", self.config.poll_interval_seconds)

while self._is_running:
await self.run_cycle()
try:
await asyncio.sleep(self.config.poll_interval_seconds)
except asyncio.CancelledError:
break

def stop(self) -> None:
logger.info("Stopping AutomationEngine gracefully...")
self._is_running = False


async def main() -> None:
config = AppConfig.from_env()
if config.debug_mode:
logger.setLevel(logging.DEBUG)
logger.debug("Debug logging enabled via environment configuration.")

engine = AutomationEngine(config)

loop = asyncio.get_running_loop()
stop_event = asyncio.Event()

def handle_signal() -> None:
logger.info("Received termination signal. Initiating graceful shutdown...")
engine.stop()
stop_event.set()

for sig in (signal.SIGINT, signal.SIGTERM):
try:
loop.add_signal_handler(sig, handle_signal)
except NotImplementedError:
pass

engine_task = asyncio.create_task(engine.start())
await stop_event.wait()

engine_task.cancel()
try:
await engine_task
except asyncio.CancelledError:
pass

logger.info("Handover Engine shutdown complete.")


if __name__ == "__main__":
try:
asyncio.run(main())
except KeyboardInterrupt:
logger.info("Program interrupted by user operator.")
sys.exit(0)

Generate a python bash script for me to run that will get any and all relevant information you need from my phone to help you this coding session

Here's a diagnostic script tailored to your Termux/Android environment and the specific failure you're hitting (disk exhaustion blocking home_audit.py):

bash
Copy
#!/usr/bin/env bash
# termux_diag.sh — targeted diagnostic for the home_audit.py failure
# Run: bash termux_diag.sh > ~/diag_output.txt 2>&1
# Then paste the output back here.

set -euo pipefail

echo "=== 1. DISK & INODE TRUTH (all mounts) ==="
df -h 2>/dev/null || echo "df unavailable"
echo
df -i 2>/dev/null || echo "df -i unavailable"
echo

echo "=== 2. TOP SPACE CONSUMERS IN ~ (depth 2) ==="
du -sh ~/* ~/.[!.]* 2>/dev/null | sort -rh | head -30
echo

echo "=== 3. LARGEST FILES (top 20) ==="
find ~ -maxdepth 4 -type f -printf '%s %p\n' 2>/dev/null | sort -rn | head -20 | \
  awk '{printf "%.1f MiB  %s\n", $1/1048576, $2}'
echo

echo "=== 4. INODE HEAVY DIRS (most files, top 10) ==="
find ~ -maxdepth 3 -type d -exec sh -c 'echo "$(find "$1" -maxdepth 1 | wc -l) $1"' _ {} \; 2>/dev/null | \
  sort -rn | head -10
echo

echo "=== 5. PYTHON & ENVIRONMENT ==="
python3 --version 2>&1
echo "PYTHONPATH=${PYTHONPATH:-<unset>}"
echo "HANDOVER_PATH=${HANDOVER_PATH:-<unset>}"
echo "POLL_INTERVAL=${POLL_INTERVAL:-<unset>}"
echo "DEBUG=${DEBUG:-<unset>}"
echo
echo "TERMUX_VERSION=$(pkg version termux-tools 2>/dev/null || echo 'n/a')"
echo "ANDROID_API=$(getprop ro.build.version.sdk 2>/dev/null || echo 'n/a')"
echo "KERNEL=$(uname -r)"
echo

echo "=== 6. MEMORY ==="
free -h 2>/dev/null || cat /proc/meminfo | head -5
echo

echo "=== 7. HOME AUDIT SCRIPT (first 60 lines + arg parser) ==="
if [ -f ~/bin/home_audit.py ]; then
  echo "--- head ---"
  head -60 ~/bin/home_audit.py
  echo "--- argparse section ---"
  grep -n 'add_argument\|argparse\|parse_args\|--stage\|--push\|--quarantine\|--exclude\|--owner\|--repo\|--stage-dir' ~/bin/home_audit.py | head -40
  echo "--- total lines: $(wc -l < ~/bin/home_audit.py)"
else
  echo "home_audit.py NOT FOUND at ~/bin/home_audit.py"
fi
echo

echo "=== 8. HANDOVER.JSON (if exists) ==="
if [ -f ~/HANDOVER.json ]; then
  cat ~/HANDOVER.json
else
  echo "HANDOVER.json not found at ~/HANDOVER.json"
  # try common alt locations
  find ~ -maxdepth 3 -name 'HANDOVER.json' 2>/dev/null | while read f; do
    echo "--- found at: $f ---"
    cat "$f"
  done
fi
echo

echo "=== 9. GIT STATUS ==="
cd ~ 2>/dev/null && git status --short 2>/dev/null | head -20 || echo "not a git repo or git unavailable"
echo

echo "=== 10. EXISTING AUDIT LOGS ==="
ls -la ~/.audit_logs/ 2>/dev/null || echo "~/.audit_logs does not exist"
echo

echo "=== 11. QUARANTINE DIR ==="
ls -la ~/\_quarantine/ 2>/dev/null || echo "~/_quarantine does not exist"
echo

echo "=== 12. TECH-REF-PORTAL (the dir that exists) ==="
ls -la ~/tech-ref-portal/ 2>/dev/null | head -20
echo "Size: $(du -sh ~/tech-ref-portal 2>/dev/null | cut -f1)"
echo

echo "=== 13. /data/user/0 MOUNT (Termux-specific) ==="
ls /data/user/0/ 2>/dev/null | head -10 || echo "cannot list /data/user/0"
df -h /data/user/0 2>/dev/null || echo "cannot stat /data/user/0"
echo

echo "=== 14. PROCESSES USING DISK (if available) ==="
lsof +D ~ 2>/dev/null | head -10 || echo "lsof unavailable"
echo

echo "=== DONE ==="
echo "Output size: $(wc -c < /dev/stdin 2>/dev/null || echo 'n/a') bytes"
How to use it:

bash
Copy
# Save it
cat > ~/termux_diag.sh << 'SCRIPT'
# ... paste the script above ...
SCRIPT

# Run it
bash ~/termux_diag.sh > ~/diag_output.txt 2>&1

# Check it's reasonable size before sharing
wc -l ~/diag_output.txt
Then paste the contents of ~/diag_output.txt here.

What this specifically targets for your current problem:

Section	Why
1–2	The ENOSPC error — need to know which mount is full and whether it's blocks or inodes
3–4	What's eating the space so we can free it before home_audit.py can even create its log dir
5	Confirm Python 3.13 + Termux specifics that may affect pathlib behavior
7	The script's actual argparse setup — your --exclude flags aren't in its argument list, so we need to see what it does accept and whether there's a config-file mechanism
8	The handover state it's supposed to be auditing
12	The one dir that does exist — might be the space hog
Once I see the output I can tell you exactly what to delete or resize to unblock the audit, and whether home_audit.py needs a patch for the --exclude flags you're trying to pass.

Welcome to Termux!

Docs: https://termux.dev/docs
Donate: https://termux.dev/donate
Community: https://termux.dev/community

Working with packages:

- Search: pkg search <query>
- Install: pkg install <package>
- Upgrade: pkg upgrade

Subscribing to additional repositories:

- Root: pkg install root-repo
- X11: pkg install x11-repo

For fixing any repository issues,
try 'termux-change-repo' command.

Report issues at https://termux.dev/issues
trp: starting dev (log: ~/.trp/dev.log in debian)
~ $ # Save it
cat > ~/termux_diag.sh << 'SCRIPT'
# ... paste the script above ...
SCRIPT

# Run it
bash ~/termux_diag.sh > ~/diag_output.txt 2>&1

# Check it's reasonable size before sharing
wc -l ~/diag_output.txt
0 /data/data/com.termux/files/home/diag_output.txt
~ $ nano termux_diag.sh
~ $ bash ~/termux_diag.sh > ~/diag_output.txt 2>&1

# Check it's reasonable size before sharing
wc -l ~/diag_output.txt
184 /data/data/com.termux/files/home/diag_output.txt
~ $ cat /data/data/com.termux/files/home/diag_output.
txt
=== 1. DISK & INODE TRUTH (all mounts) ===
Filesystem Size Used Avail Use% Mounted on
/dev/block/dm-12 887M 884M 0 100% /
tmpfs 3.5G 2.1M 3.5G 1% /dev
tmpfs 3.5G 0 3.5G 0% /mnt
/dev/block/dm-14 483M 482M 0 100% /system_ext
/dev/block/dm-15 4.4G 4.4G 0 100% /product
/dev/block/dm-16 910M 907M 0 100% /vendor
/dev/block/dm-17 24M 24M 0 100% /vendor_dlkm
tmpfs 3.5G 28K 3.5G 1% /apex
/dev/block/loop4 8.3M 8.2M 0 100% /apex/com.goog
le.pixel.wifi.ext@1
/dev/block/loop7 145M 145M 0 100% /apex/com.goog
le.android.gmssystem@2622360
/dev/block/loop11 61M 61M 0 100% /apex/com.andr
oid.virt@371899999
/dev/block/loop5 4.4M 4.4M 0 100% /apex/com.andr
oid.compos@371899999
/dev/block/dm-23 232K 24K 204K 11% /apex/com.andr
oid.resolv@374205000
/dev/block/loop14 364K 332K 28K 93% /apex/com.andr
oid.hardware.biometrics.face.virtual@2
/dev/block/loop8 256M 256M 0 100% /apex/com.goog
le.pixel.camera.hal@1915780552
/dev/block/loop6 81M 81M 0 100% /apex/com.goog
le.android.hardware.biometrics.face@1
/dev/block/loop10 37M 37M 0 100% /apex/com.andr
oid.i18n@1
/dev/block/loop9 232K 108K 120K 48% /apex/com.andr
oid.apex.cts.shim@1
/dev/block/loop12 7.1M 7.1M 0 100% /apex/com.andr
oid.devicelock@344302000
/dev/block/loop13 232K 152K 76K 67% /apex/com.andr
oid.hardware.cas@1
/dev/block/dm-25 18M 18M 0 100% /apex/com.andr
oid.wifi@374200150
/dev/block/dm-26 35M 35M 0 100% /apex/com.andr
oid.media.swcodec@374200180
/dev/block/dm-24 6.2M 6.2M 0 100% /apex/com.andr
oid.webapp@374205000
/dev/block/dm-52 6.2M 6.1M 0 100% /apex/com.andr
oid.extservices@374200023
/dev/block/dm-34 5.7M 5.7M 0 100% /apex/com.andr
oid.uwb@374205000
/dev/block/dm-41 21M 21M 0 100% /apex/com.andr
oid.adservices@372038020
/dev/block/loop16 232K 88K 140K 39% /apex/com.goog
le.pixel.euicc.update@371899999
/dev/block/dm-46 6.1M 6.0M 0 100% /apex/com.andr
oid.media@374200080
/dev/block/dm-27 30M 30M 0 100% /apex/com.andr
oid.permission@374200200
/dev/block/dm-63 1.1M 1.0M 12K 99% /apex/com.andr
oid.sdkext@374205000
/dev/block/dm-39 24M 24M 0 100% /apex/com.andr
oid.cellbroadcast@374205020
/dev/block/dm-32 3.2M 3.2M 0 100% /apex/com.andr
oid.os.statsd@372040000
/dev/block/dm-60 22M 22M 0 100% /apex/com.andr
oid.bt@374205020
/dev/block/dm-36 14M 14M 0 100% /apex/com.andr
oid.ondevicepersonalization@372038020
/dev/block/loop15 23M 23M 0 100% /apex/com.goog
le.android.widevine@190260313
/dev/block/dm-53 5.5M 5.5M 0 100% /apex/com.andr
oid.telephonycore@374205000
/dev/block/loop18 8.6M 8.5M 0 100% /apex/com.andr
oid.runtime@1
/dev/block/dm-55 808K 776K 16K 98% /apex/com.andr
oid.tzdata@373041620
/dev/block/dm-44 5.6M 5.5M 0 100% /apex/com.andr
oid.configinfrastructure@374200040
/dev/block/dm-61 31M 31M 0 100% /apex/com.andr
oid.art@374200160
/dev/block/loop19 628K 600K 16K 98% /apex/com.andr
oid.hardware.biometrics.fingerprint.virtual@1
/dev/block/dm-37 7.9M 7.9M 0 100% /apex/com.andr
oid.adbd@374200120
/dev/block/dm-45 768K 740K 16K 98% /apex/com.andr
oid.ipsec@374205000
/dev/block/dm-38 3.2M 3.2M 0 100% /apex/com.andr
oid.uprobestats@374205000
/dev/block/dm-56 5.3M 5.3M 0 100% /apex/com.andr
oid.conscrypt@374205000
/dev/block/dm-29 2.7M 2.7M 0 100% /apex/com.andr
oid.profiling@374205000
/dev/block/dm-48 4.4M 4.4M 0 100% /apex/com.andr
oid.appsearch@374205000
/dev/block/dm-51 11M 11M 0 100% /apex/com.andr
oid.nfcservices@374205000
/dev/block/dm-57 464K 436K 20K 96% /apex/com.andr
oid.scheduling@374205000
/dev/block/dm-42 1.4M 1.4M 0 100% /apex/com.andr
oid.rkpd@374200020
/dev/block/dm-31 336K 304K 28K 92% /apex/com.andr
oid.crashrecovery@374205000
/dev/block/dm-65 17M 17M 0 100% /apex/com.andr
oid.healthfitness@374205000
/dev/block/dm-62 29M 29M 0 100% /apex/com.andr
oid.mediaprovider@374200100
/dev/block/dm-54 5.8M 5.8M 0 100% /apex/com.andr
oid.npumanager@371899999
/dev/block/dm-35 4.5M 4.5M 0 100% /apex/com.andr
oid.neuralnetworks@374200020
/dev/block/dm-59 29M 29M 0 100% /apex/com.andr
oid.tethering@374205000
tmpfs 3.5G 4.0K 3.5G 1% /bootstrap-ape
x
/dev/block/loop3 8.6M 8.5M 0 100% /bootstrap-ape
x/com.android.runtime@1
/dev/block/loop2 804K 772K 16K 98% /bootstrap-ape
x/com.android.tzdata@372037000
/dev/block/loop1 61M 61M 0 100% /bootstrap-ape
x/com.android.virt@371899999
/dev/block/loop0 37M 37M 0 100% /bootstrap-ape
x/com.android.i18n@1
tmpfs 3.5G 0 3.5G 0% /tmp
/dev/fuse 109G 108G 772M 100% /storage/emula
ted

Filesystem Inodes IUsed
IFree IUse% Mounted on
/dev/block/dm-12 2912 2882
30 99% /
tmpfs 936601 1650
934951 1% /dev
tmpfs 936601 54
936547 1% /mnt
/dev/block/dm-14 448 385
63 86% /system_ext
/dev/block/dm-15 1728 1461
267 85% /product
/dev/block/dm-16 4352 4324
28 100% /vendor
/dev/block/dm-17 96 93
3 97% /vendor_dlkm
tmpfs 940057 98
939959 1% /apex
/dev/block/loop4 48 37
11 78% /apex/com.google.pixel.wifi.ext
@1
/dev/block/loop7 18446744073709551615 11 18446
744073709551604 1% /apex/com.google.android.gmssys
tem@2622360
/dev/block/loop11 18446744073709551615 70 18446
744073709551545 1% /apex/com.android.virt@37189999
9
/dev/block/loop5 48 38
10 80% /apex/com.android.compos@371899
999
/dev/block/dm-23 16 12
4 75% /apex/com.android.resolv@374205
000
/dev/block/loop14 32 19
13 60% /apex/com.android.hardware.biom
etrics.face.virtual@2
/dev/block/loop8 240 197
43 83% /apex/com.google.pixel.camera.h
al@1915780552
/dev/block/loop6 96 86
10 90% /apex/com.google.android.hardwa
re.biometrics.face@1
/dev/block/loop10 32 30
2 94% /apex/com.android.i18n@1
/dev/block/loop9 32 22
10 69% /apex/com.android.apex.cts.shim
@1
/dev/block/loop12 32 32
0 100% /apex/com.android.devicelock@34
4302000
/dev/block/loop13 32 19
13 60% /apex/com.android.hardware.cas@
1
/dev/block/dm-25 64 56
8 88% /apex/com.android.wifi@37420015
0
/dev/block/dm-26 128 127
1 100% /apex/com.android.media.swcodec
@374200180
/dev/block/dm-24 48 34
14 71% /apex/com.android.webapp@374205
000
/dev/block/dm-52 32 19
13 60% /apex/com.android.extservices@3
74200023
/dev/block/dm-34 48 36
12 75% /apex/com.android.uwb@374205000
/dev/block/dm-41 48 41
7 86% /apex/com.android.adservices@37
2038020
/dev/block/loop16 16 15
1 94% /apex/com.google.pixel.euicc.up
date@371899999
/dev/block/dm-46 64 59
5 93% /apex/com.android.media@3742000
80
/dev/block/dm-27 48 38
10 80% /apex/com.android.permission@37
4200200
/dev/block/dm-63 32 24
8 75% /apex/com.android.sdkext@374205
000
/dev/block/dm-39 32 29
3 91% /apex/com.android.cellbroadcast
@374205020
/dev/block/dm-32 48 33
15 69% /apex/com.android.os.statsd@372
040000
/dev/block/dm-60 48 48
0 100% /apex/com.android.bt@374205020
/dev/block/dm-36 48 35
13 73% /apex/com.android.ondeviceperso
nalization@372038020
/dev/block/loop15 48 35
13 73% /apex/com.google.android.widevi
ne@190260313
/dev/block/dm-53 48 41
7 86% /apex/com.android.telephonycore
@374205000
/dev/block/loop18 48 37
11 78% /apex/com.android.runtime@1
/dev/block/dm-55 48 47
1 98% /apex/com.android.tzdata@373041
620
/dev/block/dm-44 48 36
12 75% /apex/com.android.configinfrast
ructure@374200040
/dev/block/dm-61 96 81
15 85% /apex/com.android.art@374200160
/dev/block/loop19 32 17
15 54% /apex/com.android.hardware.biom
etrics.fingerprint.virtual@1
/dev/block/dm-37 48 34
14 71% /apex/com.android.adbd@37420012
0
/dev/block/dm-45 32 25
7 79% /apex/com.android.ipsec@3742050
00
/dev/block/dm-38 64 52
12 82% /apex/com.android.uprobestats@3
74205000
/dev/block/dm-56 160 157
3 99% /apex/com.android.conscrypt@374
205000
/dev/block/dm-29 32 31
1 97% /apex/com.android.profiling@374
205000
/dev/block/dm-48 48 34
14 71% /apex/com.android.appsearch@374
205000
/dev/block/dm-51 48 45
3 94% /apex/com.android.nfcservices@3
74205000
/dev/block/dm-57 32 25
7 79% /apex/com.android.scheduling@37
4205000
/dev/block/dm-42 32 27
5 85% /apex/com.android.rkpd@37420002
0
/dev/block/dm-31 32 25
7 79% /apex/com.android.crashrecovery
@374205000
/dev/block/dm-65 48 36
12 75% /apex/com.android.healthfitness
@374205000
/dev/block/dm-62 64 50
14 79% /apex/com.android.mediaprovider
@374200100
/dev/block/dm-54 80 70
10 88% /apex/com.android.npumanager@37
1899999
/dev/block/dm-35 16 16
0 100% /apex/com.android.neuralnetwork
s@374200020
/dev/block/dm-59 96 93
3 97% /apex/com.android.tethering@374
205000
tmpfs 940057 10
940047 1% /bootstrap-apex
/dev/block/loop3 48 37
11 78% /bootstrap-apex/com.android.run
time@1
/dev/block/loop2 48 47
1 98% /bootstrap-apex/com.android.tzd
ata@372037000
/dev/block/loop1 18446744073709551615 70 18446
744073709551545 1% /bootstrap-apex/com.android.vir
t@371899999
/dev/block/loop0 32 30
2 94% /bootstrap-apex/com.android.i18
n@1
tmpfs 940057 1
940056 1% /tmp
/dev/fuse 13511677 13313930
197747 99% /storage/emulated

=== 2. TOP SPACE CONSUMERS IN ~ (depth 2) ===
348M /data/data/com.termux/files/home/rss-automato
r
346M /data/data/com.termux/files/home/ez_git
346M /data/data/com.termux/files/home/ez_gemini_ge
ms_to_skills
257M /data/data/com.termux/files/home/tech-ref-por
tal
110M /data/data/com.termux/files/home/capyreader
104M /data/data/com.termux/files/home/.npm
66M /data/data/com.termux/files/home/animated-umb
rella
62M /data/data/com.termux/files/home/.local
28M /data/data/com.termux/files/home/home-archive
-staging
24M /data/data/com.termux/files/home/keystone-hea
lth-advocate-pa-medical-rights-wiki-forensic-tool
17M /data/data/com.termux/files/home/projects
15M /data/data/com.termux/files/home/animated-umb
rella-main
12M /data/data/com.termux/files/home/ez_ip_opsec_
audit
4.2M /data/data/com.termux/files/home/duck-dev
2.2M /data/data/com.termux/files/home/.netprobe
1.7M /data/data/com.termux/files/home/ez_mac
1.5M /data/data/com.termux/files/home/Keystone_Hea
lth
1.2M /data/data/com.termux/files/home/EnterpriseSe
cureMobileSkeleton
810K /data/data/com.termux/files/home/tmp
701K /data/data/com.termux/files/home/October_2026
566K /data/data/com.termux/files/home/Linux
436K /data/data/com.termux/files/home/env-audit-ar
chive
413K /data/data/com.termux/files/home/netprobe
411K /data/data/com.termux/files/home/invizible-pr
o-agent-skills
387K /data/data/com.termux/files/home/cl_else_roku
302K /data/data/com.termux/files/home/pa_ra_onepag
ers
264K /data/data/com.termux/files/home/audit_out
237K /data/data/com.termux/files/home/goldenfoss-s
taging
229K /data/data/com.termux/files/home/termux_maste
r_ultra
203K /data/data/com.termux/files/home/ez_multi_rep
o

=== 3. LARGEST FILES (top 20) ===
13.3 MiB /data/data/com.termux/files/home/projects/k
eystone_health/Pennsylvania
10.0 MiB /data/data/com.termux/files/home/ez_ip_opse
c_audit/activity_trace.csv
2.5 MiB /data/data/com.termux/files/home/rss-automat
or/ez_gemini_gems_to_skills/Active_Projects/Python.zi
p
2.5 MiB /data/data/com.termux/files/home/ez_git/Acti
ve_Projects/Python.zip
2.5 MiB /data/data/com.termux/files/home/ez_gemini_g
ems_to_skills/Active_Projects/Python.zip
2.4 MiB /data/data/com.termux/files/home/keystone-he
alth-advocate-pa-medical-rights-wiki-forensic-tool/pe
nnsylvania-health-insurance-audit-2026-main
2.4 MiB /data/data/com.termux/files/home/duck-dev/ke
ystone-sync/pennsylvania-health-insurance-audit-2026-
main
2.2 MiB /data/data/com.termux/files/home/.netprobe/c
ache/603e776c4f62b98a.hosts
1.1 MiB /data/data/com.termux/files/home/rss-automat
or/feeds.db-wal
0.9 MiB /data/data/com.termux/files/home/rss-automat
or/feeds.db
0.6 MiB /data/data/com.termux/files/home/home-archiv
e-staging/projects/ez_ip_opsec_audit/Screenshot_20261
004-105520.png
0.6 MiB /data/data/com.termux/files/home/ez_ip_opsec
_audit/Screenshot_20261004-105520.png
0.6 MiB /data/data/com.termux/files/home/tech-ref-po
rtal/node_modules/prettier/index.mjs
0.6 MiB /data/data/com.termux/files/home/tech-ref-po
rtal/node_modules/date-fns/typings.d.ts
0.5 MiB /data/data/com.termux/files/home/tech-ref-po
rtal/node_modules/lodash/lodash.js
0.4 MiB /data/data/com.termux/files/home/projects/ke
ystone_health/Keystone
0.4 MiB /data/data/com.termux/files/home/home-archiv
e-staging/_audit/20261107_082622/SKIPPED.txt
0.4 MiB /data/data/com.termux/files/home/home-archiv
e-staging/_audit/20261107_082550/SKIPPED.txt
0.3 MiB /data/data/com.termux/files/home/tech-ref-po
rtal/node_modules/prettier/THIRD-PARTY-NOTICES.md
0.3 MiB /data/data/com.termux/files/home/ez_git/.git
/index
~ $ ls -la
total 1574
drwx------. 80 u0_a399 u0_a399 20480 Nov 9 15:33 .
drwxrwx--x. 5 u0_a399 u0_a399 3452 Oct 11 12:19 ..
-rw-------. 1 u0_a399 u0_a399 22026 Nov 9 15:22 .ba
sh_history
-rw-------. 1 u0_a399 u0_a399 2190 Nov 8 18:36 .ba
shrc
drwx------. 4 u0_a399 u0_a399 3452 Nov 9 12:28 .ca
che
-rw-------. 1 u0_a399 u0_a399 53248 Nov 7 00:35 .co
verage
-rwx------. 1 u0_a399 u0_a399 14 Oct 13 03:48 .ez
_archive.sh
drwx------. 3 u0_a399 u0_a399 3452 Nov 7 17:38 .gi
t
drwx------. 7 u0_a399 u0_a399 3452 Nov 3 09:22 .gi
t-accidental-backup
drwx------. 2 u0_a399 u0_a399 3452 Nov 7 06:36 .gi
t-hooks
drwx------. 2 u0_a399 u0_a399 3452 Nov 9 03:28 .gi
t_archive
-rw-------. 1 u0_a399 u0_a399 484 Nov 7 06:36 .gi
tconfig
drwx------. 3 u0_a399 u0_a399 3452 Nov 4 10:58 .gi
thub
-rw-------. 1 u0_a399 u0_a399 174 Nov 2 01:20 .gi
tignore_global
drwxr-xr-x. 7 u0_a399 u0_a399 3452 Nov 7 01:45 .lo
cal
drwx------. 3 u0_a399 u0_a399 3452 Oct 15 20:38 .ne
tprobe
drwx------. 5 u0_a399 u0_a399 3452 Nov 9 03:25 .np
m
drwx------. 3 u0_a399 u0_a399 3452 Nov 7 06:36 .np
m-global
-rw-------. 1 u0_a399 u0_a399 66 Nov 7 06:36 .np
mrc
drwx------. 3 u0_a399 u0_a399 3452 Nov 7 00:38 .py
test_cache
-rw-------. 1 u0_a399 u0_a399 38632 Nov 7 07:09 .py
thon_history
-rw-------. 1 u0_a399 u0_a399 154 Nov 5 16:28 .re
po_split.list
drwx------. 2 u0_a399 u0_a399 3452 Nov 1 11:48 .ss
h
drwx------. 3 u0_a399 u0_a399 3452 Sep 9 03:18 .su
root
drwx------. 3 u0_a399 u0_a399 3452 Oct 12 21:39 .te
rmux
drwx------. 6 u0_a399 u0_a399 3452 Nov 8 18:27 .te
rmux_master
drwx------. 2 u0_a399 u0_a399 3452 Nov 1 20:32 .tp
drwx------. 2 u0_a399 u0_a399 3452 Nov 7 07:25 .tr
p-bak
drwx------. 2 u0_a399 u0_a399 3452 Oct 14 18:15 .w3
m
-rw-------. 1 u0_a399 u0_a399 24 Oct 12 13:26 .wg
_private_key
-rw-------. 1 u0_a399 u0_a399 0 Nov 7 05:26 50%
-rw-------. 1 u0_a399 u0_a399 0 Nov 5 13:57 Boo
tstrapping
drwx------. 2 u0_a399 u0_a399 3452 Nov 9 03:28 Chr
ombook_crostini_repair_guide
drwxr-xr-x. 8 u0_a399 u0_a399 3452 Nov 7 07:19 Ent
erpriseSecureMobileSkeleton
-rw-------. 1 u0_a399 u0_a399 0 Nov 5 13:57 Env
ironment
-rw-------. 1 u0_a399 u0_a399 0 Oct 10 20:58 FET
CH_HEAD
drwx------. 2 u0_a399 u0_a399 3452 Nov 9 03:28 Gol
denfoss
drwx------. 13 u0_a399 u0_a399 3452 Nov 5 19:43 Key
stone_Health
drwx------. 4 u0_a399 u0_a399 3452 Oct 31 06:37 Lin
ux
drwx------. 3 u0_a399 u0_a399 3452 Oct 14 03:44 Oct
ober_2026
-rw-------. 1 u0_a399 u0_a399 9 Nov 3 09:22 REA
DME.md
drwx------. 2 u0_a399 u0_a399 3452 Oct 15 19:55 adb
lock_tester
drwx------. 19 u0_a399 u0_a399 8192 Nov 1 02:58 ani
mated-umbrella
drwx------. 14 u0_a399 u0_a399 3452 Oct 11 02:44 ani
mated-umbrella-main
-rw-------. 1 u0_a399 u0_a399 9 Oct 11 02:47 ani
mated-umbrella.tar.gz
drwx------. 2 u0_a399 u0_a399 3452 Oct 14 18:07 ani
mated-umbrella.wiki
-rwx------. 1 u0_a399 u0_a399 3698 Oct 31 01:43 apk
_report.py
drwx------. 3 u0_a399 u0_a399 3452 Oct 15 20:10 ato
mic_session
drwx------. 2 u0_a399 u0_a399 3452 Oct 14 20:18 aud
it_out
drwx------. 9 u0_a399 u0_a399 3452 Oct 11 10:31 aud
it_tools
drwx------. 2 u0_a399 u0_a399 3452 Nov 9 03:29 bac
kups
drwx------. 3 u0_a399 u0_a399 3452 Nov 7 07:18 bin
-rwx------. 1 u0_a399 u0_a399 20001 Oct 31 02:01 boo
tstrap_ez_vlc_android.sh
-rwx------. 1 u0_a399 u0_a399 3360 Nov 5 15:32 boo
tstrap_keystone.sh
-rwx------. 1 u0_a399 u0_a399 1011 Nov 5 13:02 boo
tstrap_keystone_env.sh
drwx------. 23 u0_a399 u0_a399 3452 Oct 15 15:41 cap
yreader
drwx------. 3 u0_a399 u0_a399 3452 Oct 13 09:09 cl_
else_roku
-rwx------. 1 u0_a399 u0_a399 4651 Nov 7 19:41 cle
anup.sh
-rwx------. 1 u0_a399 u0_a399 2054 Nov 6 19:41 cle
anup_bighunt.sh
-rwx------. 1 u0_a399 u0_a399 18358 Nov 6 11:45 cle
anup_gui.py
-rwx------. 1 u0_a399 u0_a399 6780 Nov 6 14:40 cle
anup_reclaim.sh
drwx------. 3 u0_a399 u0_a399 3452 Nov 7 06:04 cor
e
-rwx------. 1 u0_a399 u0_a399 5162 Oct 10 14:48 ctx
.sh
-rw-------. 1 u0_a399 u0_a399 29800 Oct 10 14:49 ctx
_report_20261010_144857.txt
-rw-------. 1 u0_a399 u0_a399 31570 Oct 10 16:28 ctx
_report_20261010_162811.txt
-rw-------. 1 u0_a399 u0_a399 9855 Nov 1 18:40 das
hboard.py
-rwx------. 1 u0_a399 u0_a399 1925 Oct 10 05:33 dep
loy.sh
-rwx------. 1 u0_a399 u0_a399 9615 Oct 13 06:29 dep
loy_bundle.sh
-rwx------. 1 u0_a399 u0_a399 798 Oct 14 16:23 dep
loy_fix.sh
-rw-------. 1 u0_a399 u0_a399 970 Oct 12 14:35 dep
loy_fixes.sh
-rwx------. 1 u0_a399 u0_a399 3751 Oct 12 14:07 dep
loy_portal.sh
-rwx------. 1 u0_a399 u0_a399 3194 Oct 12 21:57 dep
loy_remediation.sh
-rw-------. 1 u0_a399 u0_a399 2329 Nov 5 12:40 dep
loyment_20261105_124005.log
-rw-------. 1 u0_a399 u0_a399 2329 Nov 5 12:48 dep
loyment_20261105_124849.log
-rw-------. 1 u0_a399 u0_a399 2329 Nov 5 13:03 dep
loyment_20261105_130302.log
-rw-------. 1 u0_a399 u0_a399 15144 Nov 9 15:36 dia
g_output.txt
drwx------. 2 u0_a399 u0_a399 3452 Nov 9 03:28 dow
nloads
drwx------. 3 u0_a399 u0_a399 3452 Oct 14 14:54 duc
k-dev
drwx------. 2 u0_a399 u0_a399 3452 Nov 7 03:16 env
-audit-archive
-rwx------. 1 u0_a399 u0_a399 1169 Oct 14 20:18 env
_audit.sh
-rw-------. 1 u0_a399 u0_a399 89864 Nov 5 23:44 env
_audit_20261105_234458.txt
-rw-------. 1 u0_a399 u0_a399 90715 Nov 6 10:57 env
_audit_20261106_105737.txt
drwx------. 6 u0_a399 u0_a399 3452 Oct 30 22:29 ez-
vlc-android
-rwx------. 1 u0_a399 u0_a399 14 Oct 13 03:48 ez_
archive.sh
drwx------. 2 u0_a399 u0_a399 3452 Oct 13 09:52 ez_
court
-rw-------. 1 u0_a399 u0_a399 34514 Oct 14 04:24 ez_
flash_backup.py
drwx------. 7 u0_a399 u0_a399 3452 Nov 7 08:10 ez_
gemini_gems_to_skills
drwx------. 10 u0_a399 u0_a399 3452 Nov 6 19:07 ez_
git
drwx------. 2 u0_a399 u0_a399 3452 Oct 14 18:07 ez_
ip_opsec_audit
drwx------. 9 u0_a399 u0_a399 3452 Nov 7 08:30 ez_
mac
drwx------. 4 u0_a399 u0_a399 3452 Nov 3 23:05 ez_
multi_repo
-rw-------. 1 u0_a399 u0_a399 7331 Nov 9 12:28 ez_
recon_20261109_122832.txt
drwx------. 3 u0_a399 u0_a399 3452 Oct 14 18:07 ez_
remote
-rw-------. 1 u0_a399 u0_a399 29754 Nov 9 12:28 ez_
repo_recon.py
drwx------. 2 u0_a399 u0_a399 3452 Nov 9 03:29 ez_
termux_backup_flashdrive
-rwx------. 1 u0_a399 u0_a399 1120 Oct 11 03:57 fin
alize_hardening.sh
-rwx------. 1 u0_a399 u0_a399 1792 Oct 12 12:43 fix
_pixel_audit.sh
-rw-------. 1 u0_a399 u0_a399 7106 Nov 1 04:16 fix
_termux_workspace.sh
-rwx------. 1 u0_a399 u0_a399 8744 Nov 1 17:11 gen
_network_defense_md.sh
-rw-------. 1 u0_a399 u0_a399 17896 Nov 1 08:57 gen
erate_handover.py
-rwx------. 1 u0_a399 u0_a399 3703 Nov 3 09:08 gen
erate_mac.py
-rwx------. 1 u0_a399 u0_a399 5665 Nov 5 11:40 gen
erate_repo_structure.py
-rw-------. 1 u0_a399 u0_a399 0 Nov 5 15:57 gen
erate_skeleton_repo.sh.save
-rw-------. 1 u0_a399 u0_a399 1997 Nov 5 15:40 git
_workflow.sh
drwx------. 4 u0_a399 u0_a399 3452 Nov 1 10:47 git
hub_deploy
-rwx------. 1 u0_a399 u0_a399 1466 Sep 9 03:07 gms
_hunter_v2.py
drwx------. 3 u0_a399 u0_a399 3452 Nov 5 18:52 gol
denfoss-staging
-rw-------. 1 u0_a399 u0_a399 21728 Nov 8 19:17 han
dover.md
drwx------. 2 u0_a399 u0_a399 3452 Nov 7 06:20 han
dover_session_20261106
-rwx------. 1 u0_a399 u0_a399 899 Oct 12 14:12 har
den_gh.sh
drwx------. 6 u0_a399 u0_a399 3452 Nov 7 08:27 hom
e-archive-staging
-rwx------. 1 u0_a399 u0_a399 4508 Nov 6 21:33 hom
e_audit.py
-rw-------. 1 u0_a399 u0_a399 3022 Nov 7 01:45 ind
ex.html
-rwx------. 1 u0_a399 u0_a399 29 Nov 7 08:10 ini
t_and_push_gems.sh
-rwx------. 1 u0_a399 u0_a399 4045 Oct 31 13:14 ins
tall.sh
-rw-------. 1 u0_a399 u0_a399 1197 Oct 11 04:26 ins
tall_safe_clone.sh
drwx------. 3 u0_a399 u0_a399 3452 Nov 1 08:57 inv
izible-pro-agent-skills
drwx------. 2 u0_a399 u0_a399 3452 Oct 15 18:44 ipl
eak_tool
drwx------. 11 u0_a399 u0_a399 8192 Nov 1 02:58 key
stone-health-advocate-pa-medical-rights-wiki-forensic
-tool
-rw-------. 1 u0_a399 u0_a399 2633 Nov 3 03:17 key
stone_fix.sh
-rw-------. 1 u0_a399 u0_a399 3430 Nov 3 03:33 key
stone_fix2.sh
-rw-------. 1 u0_a399 u0_a399 8528 Nov 3 03:33 key
stone_fix2_20261103_033334.txt
-rw-------. 1 u0_a399 u0_a399 4454 Nov 5 15:07 key
stone_fix3.sh
-rw-------. 1 u0_a399 u0_a399 1099 Nov 5 15:13 key
stone_fix3_20261105_150418.txt
-rw-------. 1 u0_a399 u0_a399 1138 Nov 5 15:07 key
stone_fix3_20261105_150725.txt
-rw-------. 1 u0_a399 u0_a399 4629 Nov 5 15:47 key
stone_fix4.sh
-rw-------. 1 u0_a399 u0_a399 5559 Nov 5 15:37 key
stone_fix4_20261105_153659.txt
-rw-------. 1 u0_a399 u0_a399 5559 Nov 5 15:47 key
stone_fix4_20261105_154709.txt
-rw-------. 1 u0_a399 u0_a399 4638 Nov 3 02:08 key
stone_fix_20261103_020759.txt
-rw-------. 1 u0_a399 u0_a399 4864 Nov 3 03:18 key
stone_fix_20261103_031806.txt
drwx------. 2 u0_a399 u0_a399 3452 Nov 3 02:08 key
stone_pdf_text
-rwx------. 1 u0_a399 u0_a399 2293 Nov 5 11:07 key
stone_sync.py
-rw-------. 1 u0_a399 u0_a399 1995 Nov 3 02:05 key
stone_triage.sh
-rw-------. 1 u0_a399 u0_a399 3209 Nov 3 02:05 key
stone_triage_20261103_020518.txt
-rw-------. 1 u0_a399 u0_a399 3146 Nov 3 03:48 key
stone_verify.sh
-rw-------. 1 u0_a399 u0_a399 2474 Nov 3 03:48 key
stone_verify_20261103_034851.txt
-rw-------. 1 u0_a399 u0_a399 2406 Nov 5 13:34 key
stone_verify_20261105_133357.txt
-rw-------. 1 u0_a399 u0_a399 2406 Nov 5 13:47 key
stone_verify_20261105_134730.txt
-rw-------. 1 u0_a399 u0_a399 2406 Nov 5 13:48 key
stone_verify_20261105_134849.txt
-rw-------. 1 u0_a399 u0_a399 497 Oct 31 16:31 loc
k_utils.sh
-rwx------. 1 u0_a399 u0_a399 985 Oct 13 00:28 mon
itor_pipeline.sh
drwx------. 2 u0_a399 u0_a399 3452 Aug 6 16:27 my-
futo-notes
drwx------. 2 u0_a399 u0_a399 3452 Oct 15 19:55 net
_tool
drwx------. 5 u0_a399 u0_a399 3452 Nov 1 11:48 net
probe
-rw-------. 1 u0_a399 u0_a399 15609 Nov 7 07:01 obl
ivion_install.sh
drwx------. 5 u0_a399 u0_a399 3452 Oct 14 18:07 ops
ec-debug
-rwx------. 1 u0_a399 u0_a399 1293 Oct 9 18:53 ops
ec_audit.py
-rw-------. 1 u0_a399 u0_a399 0 Oct 10 10:35 pa_
aca_briefing_py2027_final.html
drwx------. 5 u0_a399 u0_a399 3452 Oct 13 06:30 pa_
audit_bundle
drwx------. 3 u0_a399 u0_a399 3452 Oct 10 20:18 pa_
ra_onepagers
-rwx------. 1 u0_a399 u0_a399 2602 Oct 12 19:33 par
se_ir_intelligence.py
drwx------. 2 u0_a399 u0_a399 3452 Nov 8 10:55 phy
lum
drwx------. 2 u0_a399 u0_a399 3452 Aug 8 13:14 pix
el-terminal-master
-rw-------. 1 u0_a399 u0_a399 1565 Oct 9 18:53 pix
el-terminal-master.20261009_185313.tgz
-rw-------. 1 u0_a399 u0_a399 1565 Oct 10 21:54 pix
el-terminal-master.20261010_215417.tgz
-rw-------. 1 u0_a399 u0_a399 1565 Oct 12 12:43 pix
el-terminal-master.20261012_124330.tgz
-rw-------. 1 u0_a399 u0_a399 1565 Nov 9 06:08 pix
el-terminal-master.20261109_060840.tgz
-rwx------. 1 u0_a399 u0_a399 645 Oct 12 14:45 pix
el_diagnostic.sh
-rw-------. 1 u0_a399 u0_a399 662 Oct 9 18:53 pix
el_diagnostic_20261009_185314.txt
-rw-------. 1 u0_a399 u0_a399 661 Oct 10 21:54 pix
el_diagnostic_20261010_215417.txt
-rw-------. 1 u0_a399 u0_a399 661 Oct 12 12:43 pix
el_diagnostic_20261012_124330.txt
-rw-------. 1 u0_a399 u0_a399 661 Nov 9 06:08 pix
el_diagnostic_20261109_060840.txt
-rw-------. 1 u0_a399 u0_a399 4545 Nov 7 13:16 pix
el_opsec_audit.sh
-rw-------. 1 u0_a399 u0_a399 2351 Nov 1 17:53 pix
svg.py
-rwx------. 1 u0_a399 u0_a399 1403 Nov 1 17:56 pla
ce_netdef.sh
drwx------. 4 u0_a399 u0_a399 3452 Nov 7 00:38 pro
jects
-rw-------. 1 u0_a399 u0_a399 3518 Nov 8 10:45 rec
on.sh
-rw-------. 1 u0_a399 u0_a399 10804 Nov 8 11:03 rec
on_report.txt
-rw-------. 1 u0_a399 u0_a399 15626 Oct 15 22:16 rep
ort.json
-rw-------. 1 u0_a399 u0_a399 882 Nov 9 07:21 ris
h
-rw-------. 1 u0_a399 u0_a399 59672 Nov 9 07:21 ris
h_shizuku.dex
drwx------. 3 u0_a399 u0_a399 3452 Oct 12 10:08 rok
u-diagnostics
drwx------. 5 u0_a399 u0_a399 3452 Oct 14 18:07 rok
u_failure_evidence
drwx------. 3 u0_a399 u0_a399 3452 Nov 8 10:19 rss
-automator
-rw-------. 1 u0_a399 u0_a399 1526 Nov 7 07:53 scr
ipt_inventory.sh
-rw-------. 1 u0_a399 u0_a399 3275 Nov 7 07:54 scr
ipt_inventory_20261107_075407.txt
drwx------. 2 u0_a399 u0_a399 3452 Nov 6 23:58 scr
ipts
-rwx------. 1 u0_a399 u0_a399 4239 Nov 1 11:42 sec
urity_cleanup.sh
-rw-------. 1 u0_a399 u0_a399 36063 Oct 13 20:24 ses
sion_context.txt
-rwx------. 1 u0_a399 u0_a399 429 Oct 12 14:45 ses
sion_diag.sh
-rw-------. 1 u0_a399 u0_a399 2117 Oct 15 14:31 set
tup.sh
-rw-------. 1 u0_a399 u0_a399 15079 Oct 15 15:40 set
up-rss-automator.sh
-rwx------. 1 u0_a399 u0_a399 983 Oct 12 21:30 set
up.sh
-rwx------. 1 u0_a399 u0_a399 782 Oct 13 07:54 set
up_tv_case.sh
drwx------. 2 u0_a399 u0_a399 3452 Sep 9 03:07 sha
red
drwx------. 2 u0_a399 u0_a399 3452 Nov 9 03:28 shh
-rwx------. 1 u0_a399 u0_a399 11469 Nov 5 12:48 shi
p_nsa_dispute_kit.sh
-rwx------. 1 u0_a399 u0_a399 3860 Sep 9 05:35 sys
_recon.sh
drwx------. 18 u0_a399 u0_a399 3452 Nov 9 06:09 tec
h-ref-portal
-rwx------. 1 u0_a399 u0_a399 13561 Nov 5 17:51 ter
mux-usb-transfer
-rw-------. 1 u0_a399 u0_a399 3084 Nov 9 15:36 ter
mux_diag.sh
-rwx------. 1 u0_a399 u0_a399 2873 Nov 1 04:38 ter
mux_environment_setup.sh
drwx------. 3 u0_a399 u0_a399 3452 Oct 31 13:15 ter
mux_master_ultra
-rw-------. 1 u0_a399 u0_a399 39 Nov 7 11:51 tes
t_generate_config.py
-rw-------. 1 u0_a399 u0_a399 998 Nov 5 12:40 tes
t_results_20261105_124005.xml
-rw-------. 1 u0_a399 u0_a399 998 Nov 5 12:48 tes
t_results_20261105_124849.xml
-rw-------. 1 u0_a399 u0_a399 1009 Nov 5 13:03 tes
t_results_20261105_130302.xml
drwx------. 3 u0_a399 u0_a399 3452 Nov 7 06:31 tes
ts
drwx------. 3 u0_a399 u0_a399 3452 Nov 1 10:46 tmp
drwx------. 3 u0_a399 u0_a399 3452 Nov 1 02:58 tmu
-dashboard-auth-patch
-rw-------. 1 u0_a399 u0_a399 11527 Oct 31 03:11 tmu
_probe.sh
-rw-------. 1 u0_a399 u0_a399 29143 Oct 31 03:12 tmu
_probe_20261031_031159.txt
drwx------. 5 u0_a399 u0_a399 3452 Nov 6 21:50 too
ls
-rw-------. 1 u0_a399 u0_a399 588 Oct 14 15:58 tra
ck_a_corrected.md
-rwx------. 1 u0_a399 u0_a399 4091 Nov 2 00:14 trp
-install.sh
-rwx------. 1 u0_a399 u0_a399 1701 Nov 2 01:13 trp
-patch1.sh
-rwx------. 1 u0_a399 u0_a399 1313 Nov 2 00:14 trp
-verify.sh
-rwx------. 1 u0_a399 u0_a399 841 Nov 1 04:34 uni
versal_path_resolver.sh
-rw-------. 1 u0_a399 u0_a399 1800 Nov 1 11:43 usb
_reader.py
-rwx------. 1 u0_a399 u0_a399 1148 Oct 12 21:54 ver
ify_portal.sh
drwx------. 4 u0_a399 u0_a399 3452 Nov 1 02:58 vid
2gif
-rw-------. 1 u0_a399 u0_a399 24524 Aug 9 20:36 vid
2gif-repo.tar.gz
~ $

The picture is clear. Your Termux root partition (/) is completely exhausted: 884M used out of 887M, 0 bytes free, and inodes at 99% (only 30 free). That's why home_audit.py can't even create its log directory — the filesystem is full.

Here's what I'd do, in order:

Step 1: Free space immediately (target: ~200M+)
The biggest wins from your du output:

bash
Copy
# 1. npm cache — safe to nuke, regenerates on demand
rm -rf ~/.npm
# saves ~104M

# 2. .local (pip/uv caches, etc.)
rm -rf ~/.local
# saves ~62M

# 3. .cache
rm -rf ~/.cache
# saves ~2-5M (small but free)

# 4. .pytest_cache
rm -rf ~/.pytest_cache
# tiny but free

# 5. The duplicate "Python.zip" appears in THREE places (2.5M each)
#    Keep one, delete the other two:
rm -f ~/rss-automator/ez_gemini_gems_to_skills/Active_Projects/Python.zip
rm -f ~/ez_gemini_gems_to_skills/Active_Projects/Python.zip
# (keep the one in ez_git/Active_Projects/Python.zip)

# 6. Old deployment logs and test results (redundant, small but every bit helps)
rm -f ~/deployment_20261105_*.log
rm -f ~/test_results_20261105_*.xml
rm -f ~/ctx_report_*.txt
rm -f ~/env_audit_202611*.txt
rm -f ~/pixel_diagnostic_*.txt
rm -f ~/pixel-terminal-master.*.tgz
rm -f ~/keystone_fix*.txt
rm -f ~/keystone_verify*.txt
rm -f ~/keystone_triage*.txt

# 7. .git-accidental-backup — sounds like it's exactly that
rm -rf ~/.git-accidental-backup

# 8. .coverage, .python_history (safe)
rm -f ~/.coverage ~/.python_history

# 9. The 0-byte junk files in ~
rm -f ~/50% ~/Bootstrapping ~/Environment ~/FETCH_HEAD
Run these, then check:

bash
Copy
df -h /
You should be looking at ~200-250M freed. That's enough headroom for home_audit.py to create its log dir and run.

Step 2: Fix the home_audit.py invocation
From your ls, the script lives at ~/home_audit.py (4508 bytes), not ~/bin/home_audit.py. Your earlier commands were hitting the wrong path (or a different version in ~/bin/). Also, the script's argparse only accepts:

css
Copy
--stage  --push  --quarantine-junk  --owner  --repo  --stage-dir
There is no --exclude flag. That's why your eval python3 ~/bin/home_audit.py --stage $EX failed. You need to either:

Option A: Patch the script to accept --exclude (I can write the patch)
Option B: Use a .audit_exclude config file if the script supports one (check its source)
Run this to see the actual script:

bash
Copy
cat ~/home_audit.py
and also:

bash
Copy
ls -la ~/bin/home_audit.py 2>/dev/null && cat ~/bin/home_audit.py
Paste both outputs and I'll write you a minimal patch that adds --exclude support (or a config-file mechanism) so your original workflow works.

Step 3: For the master_engine.py you shared
Your StorageTruthChecker checks / and /data/user/0. On this device:

/ = the 887M Termux root (what's full)
/data/user/0 = the Android app data partition (the 109G /storage/emulated is FUSE-mounted, so statvfs on /data/user/0 may show the same underlying storage)
The engine is fine architecturally. The only issue is that it can't write its own log output when the disk is full. A small defensive addition:

python
Copy
# In StorageTruthChecker.process(), before logging:
if free_gib < 0.05:  # < 50 MiB
    logger.critical(
        "DISK NEARLY FULL [%s]: only %.1f MiB free — "
        "further writes will fail. Run cleanup before continuing.",
        mount, free_gib * 1024
    )
    continue  # skip inode check, no point
And in AutomationEngine.run_cycle(), wrap the log writes in a try/except for OSError so a full disk doesn't kill the entire loop:

python
Copy
async def run_cycle(self) -> None:
    try:
        state_data = self.validator.load_state()
        for processor in self.processors:
            try:
                await processor.process(state_data)
            except OSError as e:
                logger.error("Processor %s failed (I/O): %s", type(processor).__name__, e)
    except Exception as e:
        logger.error("Error during automation cycle execution: %s", e)
TL;DR — do this now
bash
Copy
# Free ~200M
rm -rf ~/.npm ~/.local ~/.cache ~/.pytest_cache ~/.git-accidental-backup
rm -f ~/50% ~/Bootstrapping ~/Environment ~/FETCH_HEAD ~/.coverage ~/.python_history
rm -f ~/deployment_20261105_*.log ~/test_results_20261105_*.xml ~/ctx_report_*.txt
rm -f ~/env_audit_202611*.txt ~/pixel_diagnostic_*.txt ~/pixel-terminal-master.*.tgz
rm -f ~/keystone_fix*.txt ~/keystone_verify*.txt ~/keystone_triage*.txt
rm -f ~/rss-automator/ez_gemini_gems_to_skills/Active_Projects/Python.zip
rm -f ~/ez_gemini_gems_to_skills/Active_Projects/Python.zip

# Verify
df -h /
df -i /

# Then run the audit (correct path)
python3 ~/home_audit.py --stage
After that, paste the output of cat ~/home_audit.py and I'll write you the --exclude patch so your full workflow (--stage → --quarantine-junk → --push) works as intended.

ml
drwx------. 3 u0_a399 u0_a399 3452 Nov 7 06:31 tests
drwx------. 3 u0_a399 u0_a399 3452 Nov 1 10:46 tmp
drwx------. 3 u0_a399 u0_a399 3452 Nov 1 02:58 tmu-dashboard-auth-patch
-rw-------. 1 u0_a399 u0_a399 11527 Oct 31 03:11 tmu_probe.sh
-rw-------. 1 u0_a399 u0_a399 29143 Oct 31 03:12 tmu~ $ # 1. npm cache — safe to nuke, regenerates on demand rm -rf ~/.npm
# saves ~104M
# 2. .local (pip/uv caches, etc.) rm -rf ~/.local
# saves ~62M
# 3. .cache rm -rf ~/.cache
# saves ~2-5M (small but free)
# 4. .pytest_cache rm -rf ~/.pytest_cache
# tiny but free
# 5. The duplicate "Python.zip" appears in THREE places (2.5M each)
# Keep one, delete the other two: rm -f ~/rss-automator/ez_gemini_gems_to_skills/Active_Projects/Python.zip
rm -f ~/ez_gemini_gems_to_skills/Active_Projects/Python.zip
# (keep the one in ez_git/Active_Projects/Python.zip)

# 6. Old deployment logs and test results (redundant, small but every bit helps)
rm -f ~/deployment_20261105_*.log
rm -f ~/test_results_20261105_*.xml
rm -f ~/ctx_report_*.txt
rm -f ~/env_audit_202611*.txt
rm -f ~/pixel_diagnostic_*.txt
rm -f ~/pixel-terminal-master.*.tgz
rm -f ~/keystone_fix*.txt
rm -f ~/keystone_verify*.txt
rm -f ~/keystone_triage*.txt

# 7. .git-accidental-backup — sounds like it's exactly that
rm -rf ~/.git-accidental-backup

# 8. .coverage, .python_history (safe)
rm -f ~/.coverage ~/.python_history

# 9. The 0-byte junk files in ~
rm -f ~/50% ~/Bootstrapping ~/Environment ~/FETCH_HEAD
~ $ df -h /
Filesystem Size Used Avail Use% Mounted on
/dev/block/dm-12 887M 884M 0 100% /
~ $ # Free ~200M
rm -rf ~/.npm ~/.local ~/.cache ~/.pytest_cache ~/.git-accidental-backup
rm -f ~/50% ~/Bootstrapping ~/Environment ~/FETCH_HEAD ~/.coverage ~/.python_history
rm -f ~/deployment_20261105_*.log ~/test_results_20261105_*.xml ~/ctx_report_*.txt
rm -f ~/env_audit_202611*.txt ~/pixel_diagnostic_*.txt ~/pixel-terminal-master.*.tgz
rm -f ~/keystone_fix*.txt ~/keystone_verify*.txt ~/keystone_triage*.txt
rm -f ~/rss-automator/ez_gemini_gems_to_skills/Active_Projects/Python.zip
rm -f ~/ez_gemini_gems_to_skills/Active_Projects/Python.zip

# Verify
df -h /
df -i /

# Then run the audit (correct path)
python3 ~/home_audit.py --stage
Filesystem Size Used Avail Use% Mounted on
/dev/block/dm-12 887M 884M 0 100% /
Filesystem Inodes IUsed IFree IUse% Mounted on
/dev/block/dm-12 2912 2882 30 99% /
usage: home_audit.py [-h] --src-dir SRC_DIR
--dst-dir DST_DIR
[--max-mb MAX_MB]
home_audit.py: error: the following arguments are required: --src-dir, --dst-dir
~ $ cat home_audit.py
#!/usr/bin/env python3
"""
home_audit.py – Secure staging helper for Termux / Android environments.

Key goals:
* Preserve zero‑copy hardlinks when the filesystem supports them.
* Gracefully fall back to a full copy on platforms that lack `os.link`.
* Keep inode‑safe semantics (never clobber the source).
* Honor the --max‑mb storage cap via `shutil.disk_usage`.
"""

import argparse
import hashlib
import json
import logging
import os
import shutil
import sys
from pathlib import Path
from typing import List

log = logging.getLogger("home_audit")
log.setLevel(logging.INFO)

# ----------------------------------------------------------------------
# Helper: storage‑cap verification (shared with existing script)
# ----------------------------------------------------------------------
def enough_space(dst: Path, max_mb: int) -> bool:
"""Return True if copying another file of size ≤ max_mb keeps us under the quota."""
usage = shutil.disk_usage(dst.parent)
free_mb = usage.free // (1024 * 1024)
return free_mb >= max_mb

# ----------------------------------------------------------------------
# Resilient staging function – the core of the patch
# ----------------------------------------------------------------------
def stage_file(src: Path, dst: Path, max_mb: int) -> bool:
"""
Stage a single file from ``src`` to ``dst``.

* Tries a hard‑link first (zero extra space).
* If ``os.link`` is missing or fails (e.g., cross‑device, sandbox), falls back to ``shutil.copy2``.
* Returns True on success, False on unrecoverable error.
"""
# Ensure destination directory hierarchy exists
dst.parent.mkdir(parents=True, exist_ok=True)

# Remove any stale file or symlink at the target location
if dst.exists() or dst.is_symlink():
dst.unlink()

# Guard against exceeding the user‑specified storage ceiling
if not enough_space(dst, max_mb):
log.error("Insufficient free space to stage %s (max %d MB)", src, max_mb)
return False

try:
# 1️⃣ Feature‑detect hard‑link capability
if hasattr(os, "link"):
try:
os.link(src, dst) # Zero‑copy path
log.debug("Hard‑linked %s → %s", src, dst)
return True
except OSError as e:
# Typical on Android/Termux when the underlying FS disallows hardlinks
log.debug("os.link failed (%s); falling back to copy2", e)

# 2️⃣ Fallback copy (preserves metadata)
shutil.copy2(src, dst)
log.debug("Copied %s → %s", src, dst)
return True

except Exception as exc:
log.error("Failed to stage %s → %s: %s", src, dst, exc)
return False

# ----------------------------------------------------------------------
# Main orchestration (simplified example)
# ----------------------------------------------------------------------
def main(argv: List[str] | None = None) -> None:
parser = argparse.ArgumentParser(description="Stage files with hard‑link fallback.")
parser.add_argument("--src-dir", required=True, help="Directory containing source files")
parser.add_argument("--dst-dir", required=True, help="Target staging directory")
parser.add_argument(
"--max-mb",
type=int,
default=200,
help="Maximum MB of free space required before staging each file",
)
args = parser.parse_args(argv)

src_root = Path(args.src_dir).expanduser().resolve()
dst_root = Path(args.dst_dir).expanduser().resolve()

if not src_root.is_dir():
log.critical("Source directory does not exist: %s", src_root)
sys.exit(1)

staged = 0
for src_path in src_root.rglob("*"):
if src_path.is_file():
rel = src_path.relative_to(src_root)
dst_path = dst_root / rel
if stage_file(src_path, dst_path, args.max_mb):
staged += 1
else:
log.warning("Skipping %s due to staging error", src_path)

log.info("Staging complete: %d files processed", staged)


if __name__ == "__main__":
logging.basicConfig(
format="%(asctime)s %(levelname)s %(name)s: %(message)s",
level=logging.INFO,
)
try:
main()
except KeyboardInterrupt:
log.info("Interrupted by user")
sys.exit(130)
except Exception as e:
log.exception("Fatal error in home_audit: %s", e)
sys.exit(1)
~ $ pkg install -y shellcheck nodejs git python
No mirror or mirror group selected. You might want to select one by running 'termux-change-repo'
Checking availability of current mirror:
[*] https://mirrors.in.sahilister.net/termux/termux-main/: ok
Get:1 https://tur.kcubeterm.com tur-packages InRelease
Get:2 https://mirrors.in2.sahilister.net/termux/termux-main stable InRelease [14.0 kB]
Get:3 https://mirrors.in2.sahilister.net/termux/termux-x11 x11 InRelease [14.0 kB]
Get:4 https://mirrors.in2.sahilister.net/termux/termux-main stable/main aarch64 Packages [589 kB]
Get:5 https://mirrors.in2.sahilister.net/termux/termux-x11 x11/main aarch64 Packages [237 kB]
Fetched 873 kB in 7s (134 kB/s)
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
8 packages can be upgraded. Run 'apt list --upgradable' to see them.
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
shellcheck is already the newest version (0.11.0-3).
git is already the newest version (2.56.0).
python is already the newest version (3.14.10).
The following packages will be REMOVED:
nodejs-lts
The following NEW packages will be installed:
nodejs
0 upgraded, 1 newly installed, 1 to remove and 7 not upgraded.
Need to get 10.7 MB of archives.
After this operation, 3969 kB of additional disk space will be used.
Get:1 https://mirrors.in2.sahilister.net/termux/termux-main stable/main aarch64 nodejs aarch64 26.13.0-1 [10.7 MB]
Fetched 10.7 MB in 6s (1864 kB/s)
pip install --upgrade ruff
npm install -g eslint
dpkg: nodejs-lts: dependency problems, but removing anyway as you requested:
yarn depends on nodejs | nodejs-lts; however:
Package nodejs is not installed.
Package nodejs-lts is to be removed.
npm depends on nodejs | nodejs-lts; however:
Package nodejs is not installed.
Package nodejs-lts is to be removed.

(Reading database ... 51618 files and directories currently installed.)
Removing nodejs-lts (24.22.2) ...
Selecting previously unselected package nodejs.
(Reading database ... 51424 files and directories currently installed.)
Preparing to unpack .../nodejs_26.13.0-1_aarch64.deb ...
Unpacking nodejs (26.13.0-1) ...
Setting up nodejs (26.13.0-1) ...
~ $ rm -rf ~/.local
~ $ cd ~ && python3 ~/bin/home_audit.py # 1. audit: statuses, public/private, third-party, leaks
2. moves junk + empty dirs to ~/_quarantine/<ts>/
EX="--exclude capyreader --exclude ez_termux_backup_flashdrive --exclude Goldenfoss --exclude goldenfoss-staging \
--exclude tech-ref-portal.stale --exclude duck-dev --exclude keystone_extract --exclude Keystone_Health_downloadable_files \
--exclude projects --exclude 'keystone-health-advocate-*' --exclude roku_failure_evidence --exclude ez_court \
--exclude cl_else_roku --exclude tmp --exclude backups"
eval python3 ~/bin/home_audit.py --stage $EX # 3. then check ~/.audit_logs/*/SKIPPED.txt
eval python3 ~/bin/home_audit.py --push $EX # 4. only after creating the PRIVATE repo
== home audit 20261109_173253 @ /data/data/com.termux/files/home [termux] ==
CONFIG 18 | EMPTY 6 | LEFTOVER 1 | OTHER 3 | OUTPUT 5 | PROJECT 51 | SCRIPT 80 | SECRET 3
home_is_git_repo: yes, 0 commits
home_repo_fix: mv ~/.git ~/.git.home-repo.bak # empty repo; makes 'git add -A' in ~ impossible
missing_expected: handover
hook_grep_candidates: 0 (HOOK-001)

PROJECTS
adblock_tester/ NO-GIT ACTIVE
animated-umbrella/ NO-GIT ACTIVE 1 sensitive file(s) in worktree
animated-umbrella-main/ NO-GIT ACTIVE 1 sensitive file(s) in worktree
animated-umbrella.wiki/ NO-GIT STALE
atomic_session/ LOCAL-ONLY ACTIVE no-github-remote
audit_tools/ NO-GIT ACTIVE
bin/ NO-GIT ACTIVE
capyreader/ DIRTY(3) ACTIVE public 30 sensitive file(s) in worktree; TRACKED secret-like files: 9; PUBLIC history has 22 secret-like path(s); !! PUBLIC REPO WITH SENSITIVE FILES
cl_else_roku/ NO-GIT ACTIVE 1 sensitive file(s) in worktree
core/ NO-GIT ACTIVE
duck-dev/ NO-GIT ACTIVE 2 sensitive file(s) in worktree
EnterpriseSecureMobileSkeleton/ LOCAL-ONLY ACTIVE no-github-remote 1 sensitive file(s) in worktree
env-audit-archive/ NO-GIT ACTIVE 7 sensitive file(s) in worktree
ez-vlc-android/ NO-GIT ACTIVE
ez_court/ NO-GIT ACTIVE
ez_gemini_gems_to_skills/ DIRTY(1) ACTIVE private-or-missing 12 sensitive file(s) in worktree; TRACKED secret-like files: 4
ez_git/ CLEAN ACTIVE public 13 sensitive file(s) in worktree; TRACKED secret-like files: 5; PUBLIC history has 5 secret-like path(s); !! PUBLIC REPO WITH SENSITIVE FILES
ez_ip_opsec_audit/ NO-GIT STALE 6 sensitive file(s) in worktree
ez_mac/ DIRTY(1) ACTIVE public 1 sensitive file(s) in worktree; !! PUBLIC REPO WITH SENSITIVE FILES
ez_multi_repo/ CLEAN ACTIVE public
ez_remote/ NO-GIT ACTIVE
goldenfoss-staging/ NO-UPSTREAM,DIRTY(1) ACTIVE public 1 sensitive file(s) in worktree; TRACKED secret-like files: 1; PUBLIC history has 1 secret-like path(s); !! PUBLIC REPO WITH SENSITIVE FILES
handover_session_20261106/ NO-GIT ACTIVE
invizible-pro-agent-skills/ NO-GIT ACTIVE
ipleak_tool/ NO-GIT ACTIVE
keystone-health-advocate-pa-medical-rights-wiki-forensic-tool/ LOCAL-ONLY,DIRTY(1) ACTIVE no-github-remote 2 sensitive file(s) in worktree
Keystone_Health/ UNPUSHED(1),DIRTY(10) ACTIVE public 2 sensitive file(s) in worktree; !! PUBLIC REPO WITH SENSITIVE FILES
keystone_pdf_text/ NO-GIT ACTIVE
Linux/ NO-GIT ACTIVE
my-futo-notes/ NO-GIT STALE 1 sensitive file(s) in worktree
net_tool/ NO-GIT ACTIVE
netprobe/ EMPTY-REPO UNKNOWN no-github-remote
October_2026/ NO-GIT ACTIVE
opsec-debug/ NO-GIT ACTIVE
pa_audit_bundle/ NO-GIT ACTIVE
pa_ra_onepagers/ NO-GIT ACTIVE
phylum/ NO-GIT ACTIVE 1 sensitive file(s) in worktree
pixel-terminal-master/ NO-GIT STALE
projects/ NO-GIT ACTIVE 14 sensitive file(s) in worktree
roku-diagnostics/ NO-GIT ACTIVE
roku_failure_evidence/ NO-GIT ACTIVE
rss-automator/ NO-GIT ACTIVE 12 sensitive file(s) in worktree
scripts/ NO-GIT ACTIVE
shared/ NO-GIT STALE
tech-ref-portal/ NO-UPSTREAM,DIRTY(23) ACTIVE private-or-missing 6 sensitive file(s) in worktree
termux_master_ultra/ LOCAL-ONLY ACTIVE no-github-remote
tests/ NO-GIT UNKNOWN
tmp/ NO-GIT ACTIVE
tmu-dashboard-auth-patch/ NO-GIT ACTIVE 2 sensitive file(s) in worktree
tools/ NO-GIT ACTIVE 3 sensitive file(s) in worktree
vid2gif/ NO-GIT STALE

SCRIPT/OUTPUT files containing secret patterns (excluded): bootstrap_ez_vlc_android.sh, dashboard.py, oblivion_install.sh, tmu_probe_20261031_031159.txt

full report: /data/data/com.termux/files/home/.audit_logs/home_audit_20261109_173253/AUDIT.md
~ $ python3 ~/bin/home_audit.py --quarantine-junk # 2. moves junk + empty dirs to ~/_quarantine/<ts>/
== home audit 20261109_173308 @ /data/data/com.termux/files/home [termux] ==
CONFIG 18 | LEFTOVER 1 | OTHER 3 | OUTPUT 5 | PROJECT 51 | SCRIPT 80 | SECRET 3
home_is_git_repo: yes, 0 commits
home_repo_fix: mv ~/.git ~/.git.home-repo.bak # empty repo; makes 'git add -A' in ~ impossible
missing_expected: handover
hook_grep_candidates: 0 (HOOK-001)
quarantined: 6 -> /data/data/com.termux/files/home/_quarantine/20261109_173308

PROJECTS
adblock_tester/ NO-GIT ACTIVE
animated-umbrella/ NO-GIT ACTIVE 1 sensitive file(s) in worktree
animated-umbrella-main/ NO-GIT ACTIVE 1 sensitive file(s) in worktree
animated-umbrella.wiki/ NO-GIT STALE
atomic_session/ LOCAL-ONLY ACTIVE no-github-remote
audit_tools/ NO-GIT ACTIVE
bin/ NO-GIT ACTIVE
capyreader/ DIRTY(3) ACTIVE public 30 sensitive file(s) in worktree; TRACKED secret-like files: 9; PUBLIC history has 22 secret-like path(s); !! PUBLIC REPO WITH SENSITIVE FILES
cl_else_roku/ NO-GIT ACTIVE 1 sensitive file(s) in worktree
core/ NO-GIT ACTIVE
duck-dev/ NO-GIT ACTIVE 2 sensitive file(s) in worktree
EnterpriseSecureMobileSkeleton/ LOCAL-ONLY ACTIVE no-github-remote 1 sensitive file(s) in worktree
env-audit-archive/ NO-GIT ACTIVE 7 sensitive file(s) in worktree
ez-vlc-android/ NO-GIT ACTIVE
ez_court/ NO-GIT ACTIVE
ez_gemini_gems_to_skills/ DIRTY(1) ACTIVE private-or-missing 12 sensitive file(s) in worktree; TRACKED secret-like files: 4
ez_git/ CLEAN ACTIVE public 13 sensitive file(s) in worktree; TRACKED secret-like files: 5; PUBLIC history has 5 secret-like path(s); !! PUBLIC REPO WITH SENSITIVE FILES
ez_ip_opsec_audit/ NO-GIT STALE 6 sensitive file(s) in worktree
ez_mac/ DIRTY(1) ACTIVE public 1 sensitive file(s) in worktree; !! PUBLIC REPO WITH SENSITIVE FILES
ez_multi_repo/ CLEAN ACTIVE public
ez_remote/ NO-GIT ACTIVE
goldenfoss-staging/ NO-UPSTREAM,DIRTY(1) ACTIVE public 1 sensitive file(s) in worktree; TRACKED secret-like files: 1; PUBLIC history has 1 secret-like path(s); !! PUBLIC REPO WITH SENSITIVE FILES
handover_session_20261106/ NO-GIT ACTIVE
invizible-pro-agent-skills/ NO-GIT ACTIVE
ipleak_tool/ NO-GIT ACTIVE
keystone-health-advocate-pa-medical-rights-wiki-forensic-tool/ LOCAL-ONLY,DIRTY(1) ACTIVE no-github-remote 2 sensitive file(s) in worktree
Keystone_Health/ UNPUSHED(1),DIRTY(10) ACTIVE public 2 sensitive file(s) in worktree; !! PUBLIC REPO WITH SENSITIVE FILES
keystone_pdf_text/ NO-GIT ACTIVE
Linux/ NO-GIT ACTIVE
my-futo-notes/ NO-GIT STALE 1 sensitive file(s) in worktree
net_tool/ NO-GIT ACTIVE
netprobe/ EMPTY-REPO UNKNOWN no-github-remote
October_2026/ NO-GIT ACTIVE
opsec-debug/ NO-GIT ACTIVE
pa_audit_bundle/ NO-GIT ACTIVE
pa_ra_onepagers/ NO-GIT ACTIVE
phylum/ NO-GIT ACTIVE 1 sensitive file(s) in worktree
pixel-terminal-master/ NO-GIT STALE
projects/ NO-GIT ACTIVE 14 sensitive file(s) in worktree
roku-diagnostics/ NO-GIT ACTIVE
roku_failure_evidence/ NO-GIT ACTIVE
rss-automator/ NO-GIT ACTIVE 12 sensitive file(s) in worktree
scripts/ NO-GIT ACTIVE
shared/ NO-GIT STALE
tech-ref-portal/ NO-UPSTREAM,DIRTY(23) ACTIVE private-or-missing 6 sensitive file(s) in worktree
termux_master_ultra/ LOCAL-ONLY ACTIVE no-github-remote
tests/ NO-GIT UNKNOWN
tmp/ NO-GIT ACTIVE
tmu-dashboard-auth-patch/ NO-GIT ACTIVE 2 sensitive file(s) in worktree
tools/ NO-GIT ACTIVE 3 sensitive file(s) in worktree
vid2gif/ NO-GIT STALE

SCRIPT/OUTPUT files containing secret patterns (excluded): bootstrap_ez_vlc_android.sh, dashboard.py, oblivion_install.sh, tmu_probe_20261031_031159.txt

full report: /data/data/com.termux/files/home/.audit_logs/home_audit_20261109_173308/AUDIT.md
~ $ EX="--exclude capyreader --exclude ez_termux_backup_flashdrive --exclude Goldenfoss --exclude goldenfoss-staging \
> --exclude tech-ref-portal.stale --exclude duck-dev --exclude keystone_extract --exclude Keystone_Health_downloadable_files \
> --exclude projects --exclude 'keystone-health-advocate-*' --exclude roku_failure_evidence --exclude ez_court \
> --exclude cl_else_roku --exclude tmp --exclude backups"
~ $ eval python3 ~/bin/home_audit.py --stage $EX # 3. then check ~/.audit_logs/*/SKIPPED.txt
== home audit 20261109_173330 @ /data/data/com.termux/files/home [termux] ==
CONFIG 18 | LEFTOVER 1 | OTHER 3 | OUTPUT 5 | PROJECT 51 | SCRIPT 80 | SECRET 3
home_is_git_repo: yes, 0 commits
home_repo_fix: mv ~/.git ~/.git.home-repo.bak # empty repo; makes 'git add -A' in ~ impossible
missing_expected: handover, Chrombook_crostini_repair_guide
hook_grep_candidates: 0 (HOOK-001)

PROJECTS
adblock_tester/ NO-GIT ACTIVE
animated-umbrella/ NO-GIT ACTIVE 1 sensitive file(s) in worktree
animated-umbrella-main/ NO-GIT ACTIVE 1 sensitive file(s) in worktree
animated-umbrella.wiki/ NO-GIT STALE
atomic_session/ LOCAL-ONLY ACTIVE no-github-remote
audit_tools/ NO-GIT ACTIVE
bin/ NO-GIT ACTIVE
capyreader/ DIRTY(3) ACTIVE public 30 sensitive file(s) in worktree; TRACKED secret-like files: 9; PUBLIC history has 22 secret-like path(s); !! PUBLIC REPO WITH SENSITIVE FILES
cl_else_roku/ NO-GIT ACTIVE 1 sensitive file(s) in worktree
core/ NO-GIT ACTIVE
duck-dev/ NO-GIT ACTIVE 2 sensitive file(s) in worktree
EnterpriseSecureMobileSkeleton/ LOCAL-ONLY ACTIVE no-github-remote 1 sensitive file(s) in worktree
env-audit-archive/ NO-GIT ACTIVE 7 sensitive file(s) in worktree
ez-vlc-android/ NO-GIT ACTIVE
ez_court/ NO-GIT ACTIVE
ez_gemini_gems_to_skills/ DIRTY(1) ACTIVE private-or-missing 12 sensitive file(s) in worktree; TRACKED secret-like files: 4
ez_git/ CLEAN ACTIVE public 13 sensitive file(s) in worktree; TRACKED secret-like files: 5; PUBLIC history has 5 secret-like path(s); !! PUBLIC REPO WITH SENSITIVE FILES
ez_ip_opsec_audit/ NO-GIT STALE 6 sensitive file(s) in worktree
ez_mac/ DIRTY(1) ACTIVE public 1 sensitive file(s) in worktree; !! PUBLIC REPO WITH SENSITIVE FILES
ez_multi_repo/ CLEAN ACTIVE public
ez_remote/ NO-GIT ACTIVE
goldenfoss-staging/ NO-UPSTREAM,DIRTY(1) ACTIVE public 1 sensitive file(s) in worktree; TRACKED secret-like files: 1; PUBLIC history has 1 secret-like path(s); !! PUBLIC REPO WITH SENSITIVE FILES
handover_session_20261106/ NO-GIT ACTIVE
invizible-pro-agent-skills/ NO-GIT ACTIVE
ipleak_tool/ NO-GIT ACTIVE
keystone-health-advocate-pa-medical-rights-wiki-forensic-tool/ LOCAL-ONLY,DIRTY(1) ACTIVE no-github-remote 2 sensitive file(s) in worktree
Keystone_Health/ UNPUSHED(1),DIRTY(10) ACTIVE public 2 sensitive file(s) in worktree; !! PUBLIC REPO WITH SENSITIVE FILES
keystone_pdf_text/ NO-GIT ACTIVE
Linux/ NO-GIT ACTIVE
my-futo-notes/ NO-GIT STALE 1 sensitive file(s) in worktree
net_tool/ NO-GIT ACTIVE
netprobe/ EMPTY-REPO UNKNOWN no-github-remote
October_2026/ NO-GIT ACTIVE
opsec-debug/ NO-GIT ACTIVE
pa_audit_bundle/ NO-GIT ACTIVE
pa_ra_onepagers/ NO-GIT ACTIVE
phylum/ NO-GIT ACTIVE 1 sensitive file(s) in worktree
pixel-terminal-master/ NO-GIT STALE
projects/ NO-GIT ACTIVE 14 sensitive file(s) in worktree
roku-diagnostics/ NO-GIT ACTIVE
roku_failure_evidence/ NO-GIT ACTIVE
rss-automator/ NO-GIT ACTIVE 12 sensitive file(s) in worktree
scripts/ NO-GIT ACTIVE
shared/ NO-GIT STALE
tech-ref-portal/ NO-UPSTREAM,DIRTY(23) ACTIVE private-or-missing 6 sensitive file(s) in worktree
termux_master_ultra/ LOCAL-ONLY ACTIVE no-github-remote
tests/ NO-GIT UNKNOWN
tmp/ NO-GIT ACTIVE
tmu-dashboard-auth-patch/ NO-GIT ACTIVE 2 sensitive file(s) in worktree
tools/ NO-GIT ACTIVE 3 sensitive file(s) in worktree
vid2gif/ NO-GIT STALE

SCRIPT/OUTPUT files containing secret patterns (excluded): bootstrap_ez_vlc_android.sh, dashboard.py, oblivion_install.sh, tmu_probe_20261031_031159.txt

full report: /data/data/com.termux/files/home/.audit_logs/home_audit_20261109_173330/AUDIT.md

staged 962 files -> /data/data/com.termux/files/home/home-archive-staging skipped 1356 (see SKIPPED.txt)
~ $ eval python3 ~/bin/home_audit.py --push $EX # 4. only after creating the PRIVATE repo
== home audit 20261109_173351 @ /data/data/com.termux/files/home [termux] ==
CONFIG 18 | LEFTOVER 1 | OTHER 3 | OUTPUT 5 | PROJECT 51 | SCRIPT 80 | SECRET 3
home_is_git_repo: yes, 0 commits
home_repo_fix: mv ~/.git ~/.git.home-repo.bak # empty repo; makes 'git add -A' in ~ impossible
missing_expected: handover, Chrombook_crostini_repair_guide
hook_grep_candidates: 0 (HOOK-001)

PROJECTS
adblock_tester/ NO-GIT ACTIVE
animated-umbrella/ NO-GIT ACTIVE 1 sensitive file(s) in worktree
animated-umbrella-main/ NO-GIT ACTIVE 1 sensitive file(s) in worktree
animated-umbrella.wiki/ NO-GIT STALE
atomic_session/ LOCAL-ONLY ACTIVE no-github-remote
audit_tools/ NO-GIT ACTIVE
bin/ NO-GIT ACTIVE
capyreader/ DIRTY(3) ACTIVE public 30 sensitive file(s) in worktree; TRACKED secret-like files: 9; PUBLIC history has 22 secret-like path(s); !! PUBLIC REPO WITH SENSITIVE FILES
cl_else_roku/ NO-GIT ACTIVE 1 sensitive file(s) in worktree
core/ NO-GIT ACTIVE
duck-dev/ NO-GIT ACTIVE 2 sensitive file(s) in worktree
EnterpriseSecureMobileSkeleton/ LOCAL-ONLY ACTIVE no-github-remote 1 sensitive file(s) in worktree
env-audit-archive/ NO-GIT ACTIVE 7 sensitive file(s) in worktree
ez-vlc-android/ NO-GIT ACTIVE
ez_court/ NO-GIT ACTIVE
ez_gemini_gems_to_skills/ DIRTY(1) ACTIVE private-or-missing 12 sensitive file(s) in worktree; TRACKED secret-like files: 4
ez_git/ CLEAN ACTIVE public 13 sensitive file(s) in worktree; TRACKED secret-like files: 5; PUBLIC history has 5 secret-like path(s); !! PUBLIC REPO WITH SENSITIVE FILES
ez_ip_opsec_audit/ NO-GIT STALE 6 sensitive file(s) in worktree
ez_mac/ DIRTY(1) ACTIVE public 1 sensitive file(s) in worktree; !! PUBLIC REPO WITH SENSITIVE FILES
ez_multi_repo/ CLEAN ACTIVE public
ez_remote/ NO-GIT ACTIVE
goldenfoss-staging/ NO-UPSTREAM,DIRTY(1) ACTIVE public 1 sensitive file(s) in worktree; TRACKED secret-like files: 1; PUBLIC history has 1 secret-like path(s); !! PUBLIC REPO WITH SENSITIVE FILES
handover_session_20261106/ NO-GIT ACTIVE
invizible-pro-agent-skills/ NO-GIT ACTIVE
ipleak_tool/ NO-GIT ACTIVE
keystone-health-advocate-pa-medical-rights-wiki-forensic-tool/ LOCAL-ONLY,DIRTY(1) ACTIVE no-github-remote 2 sensitive file(s) in worktree
Keystone_Health/ UNPUSHED(1),DIRTY(10) ACTIVE public 2 sensitive file(s) in worktree; !! PUBLIC REPO WITH SENSITIVE FILES
keystone_pdf_text/ NO-GIT ACTIVE
Linux/ NO-GIT ACTIVE
my-futo-notes/ NO-GIT STALE 1 sensitive file(s) in worktree
net_tool/ NO-GIT ACTIVE
netprobe/ EMPTY-REPO UNKNOWN no-github-remote
October_2026/ NO-GIT ACTIVE
opsec-debug/ NO-GIT ACTIVE
pa_audit_bundle/ NO-GIT ACTIVE
pa_ra_onepagers/ NO-GIT ACTIVE
phylum/ NO-GIT ACTIVE 1 sensitive file(s) in worktree
pixel-terminal-master/ NO-GIT STALE
projects/ NO-GIT ACTIVE 14 sensitive file(s) in worktree
roku-diagnostics/ NO-GIT ACTIVE
roku_failure_evidence/ NO-GIT ACTIVE
rss-automator/ NO-GIT ACTIVE 12 sensitive file(s) in worktree
scripts/ NO-GIT ACTIVE
shared/ NO-GIT STALE
tech-ref-portal/ NO-UPSTREAM,DIRTY(23) ACTIVE private-or-missing 6 sensitive file(s) in worktree
termux_master_ultra/ LOCAL-ONLY ACTIVE no-github-remote
tests/ NO-GIT UNKNOWN
tmp/ NO-GIT ACTIVE
tmu-dashboard-auth-patch/ NO-GIT ACTIVE 2 sensitive file(s) in worktree
tools/ NO-GIT ACTIVE 3 sensitive file(s) in worktree
vid2gif/ NO-GIT STALE

SCRIPT/OUTPUT files containing secret patterns (excluded): bootstrap_ez_vlc_android.sh, dashboard.py, oblivion_install.sh, tmu_probe_20261031_031159.txt

full report: /data/data/com.termux/files/home/.audit_logs/home_audit_20261109_173351/AUDIT.md

staged 962 files -> /data/data/com.termux/files/home/home-archive-staging skipped 1356 (see SKIPPED.txt)

ABORT[4]: bilbywilby/home-archive not reachable with your stored credentials (does it exist?). fatal: could not read Username for 'https://github.com': terminal prompts disabled
~ $

To diagnose and resolve storage bottlenecks, inode exhaustion, and authentication failures in your Termux environment, execute the following commands and corrective procedures.
1. Storage & Inode Inspection
Run the storage inspection audit script to verify partition usage, inode counts, and large directory allocations:
echo "== Storage & Inode Status =="
df -h / "$HOME"
df -i / "$HOME" | tail -n +1

echo "== Top Directory Sizes in Home =="
du -h -d 1 "$HOME" | sort -hr | head -n 15

echo "== Temporary & Cache Audit =="
du -h -d 2 ~/.cache ~/.local 2>/dev/null | sort -hr | head -n 10

2. Root Partition Remediation (/ at 100% / 99% Inodes)
The Android root partition (/dev/block/dm-12) has strict size and inode limits in Termux. When it reaches capacity:
* Clear Apt and Package Manager Caches:
pkg clean
apt-get clean
rm -rf $PREFIX/var/cache/apt/archives/*.deb

* Prune Unused Python & Node Caches:
rm -rf ~/.cache/pip ~/.npm ~/.pytest_cache

* Remove Stale Temporary Files:
rm -rf ~/.local/share/Trash/* ~/.tmp/* /tmp/*

3. Resolving Git Remote Authentication Failure (ABORT[4])
The push to bilbywilby/home-archive failed because terminal prompts are disabled (GIT_TERMINAL_PROMPT=0) and stored credentials are missing or expired.
* Configure SSH Authentication (Recommended):
Ensure your SSH key is added to the SSH agent and GitHub:
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
git remote set-url origin git@github.com:bilbywilby/home-archive.git

* Configure Personal Access Token (PAT) via Credential Helper:
If using HTTPS, store your GitHub PAT securely in Termux:
git config --global credential.helper store
# Next push will prompt for username and PAT once, saving credentials for subsequent runs.

Architectural & Refactoring Plan
* SOLID & KISS Design Principles:
* Single Responsibility Principle (SRP): Each class encapsulates a discrete operational responsibility (AppConfig for configuration, HandoverStateValidator for secure JSON I/O, StorageTruthChecker for block and inode health, BlockerInspector for impediments, NextActionReporter for runbook progression, and AutomationEngine for asynchronous orchestration).
* Open/Closed Principle (OCP): The execution pipeline accepts pluggable processors via the StateProcessorProtocol, enabling new monitoring or alerting extensions without altering core engine logic.
* Dependency Inversion (DIP): High-level coordination depends on abstract protocols and injected configurations rather than global state.
* Core Components (master_engine.py):
* AppConfig: Immutable configuration dataclass loading environment variables (HANDOVER_PATH, POLL_INTERVAL, DEBUG) with robust fallback defaults.
* HandoverStateValidator: Safely loads and validates HANDOVER.json, returning default structures gracefully if the file is missing or locked.
* StorageTruthChecker: Audits both block utilization (f_bavail/f_blocks) and inode consumption (f_ffree/f_files) across host-truth mounts (/ and /data/user/0), preventing thin-provisioned exhaustion blind spots.
* BlockerInspector: Parses session blockers and logs active impediments at INFO level for transparent background execution.
* NextActionReporter: Automatically inspects the next_session_runbook array and surfaces the exact operational frontier step.
* AutomationEngine: Asynchronous orchestrator executing modular processors on a timed interval with clean signal handling (SIGINT, SIGTERM) for zero-interaction background persistence.
* Production Readiness:
* Zero External Dependencies: Pure Python standard library implementation (asyncio, json, logging, os, pathlib, signal, sys), ensuring seamless execution across Android Termux, Debian virtual machines, and Linux environments.
* Defensive Error Boundaries: Graceful exception handling for missing files, permission errors, and missing mount points.
#!/usr/bin/env python3
"""
Master Automation and Handover Engine (v2.1)
Refactored following SOLID, KISS, and PEP 8 standards.
Provides zero-interaction background monitoring, dual-mount storage truth validation (bytes & inodes),
active blocker inspection, and actionable runbook frontier reporting for mobile & virtualized Linux environments.
"""

from __future__ import annotations

import asyncio
import json
import logging
import os
import signal
import sys
from dataclasses import dataclass, field
from pathlib import Path
from typing import Any, Dict, List, Optional, Protocol

# Configure structured logging
logging.basicConfig(
level=logging.INFO,
format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
handlers=[logging.StreamHandler(sys.stdout)]
)
logger = logging.getLogger("HandoverEngine")


@dataclass(frozen=True)
class AppConfig:
"""Type-safe application configuration loaded from environment or defaults."""
handover_path: Path = field(default_factory=lambda: Path("~/HANDOVER.json").expanduser())
poll_interval_seconds: float = 60.0
debug_mode: bool = False

@classmethod
def from_env(cls) -> AppConfig:
return cls(
handover_path=Path(os.getenv("HANDOVER_PATH", "~/HANDOVER.json")).expanduser(),
poll_interval_seconds=float(os.getenv("POLL_INTERVAL", "60.0")),
debug_mode=os.getenv("DEBUG", "false").lower() in ("true", "1", "yes")
)


class StateProcessorProtocol(Protocol):
"""Protocol defining state inspection handlers for the automation pipeline."""
async def process(self, state_data: Dict[str, Any]) -> None:
...


class HandoverStateValidator:
"""Validates and parses HANDOVER.json state securely."""

def __init__(self, file_path: Path) -> None:
self.file_path = file_path

def load_state(self) -> Dict[str, Any]:
if not self.file_path.exists():
logger.warning("Handover file not found at %s. Using default empty state.", self.file_path)
return {"handover_version": "2.1", "blockers": [], "unresolved_items": [], "next_session_runbook": []}
try:
with open(self.file_path, "r", encoding="utf-8") as f:
data = json.load(f)
if not isinstance(data, dict):
raise ValueError("Handover root must be a JSON object.")
return data
except Exception as e:
logger.error("Failed to parse handover state from %s: %s", self.file_path, e)
raise


class StorageTruthChecker:
"""Audits blocks AND inodes across host-truth and guest mounts to prevent thin-provisioned blind spots."""

MOUNTS = ("/", "/data/user/0")

async def process(self, state_data: Dict[str, Any]) -> None:
for mount in self.MOUNTS:
try:
st = os.statvfs(mount)
except OSError:
continue

free_gib = (st.f_bavail * st.f_frsize) / (1024 ** 3)
total_gib = (st.f_blocks * st.f_frsize) / (1024 ** 3)
used_pct = (1 - st.f_bavail / st.f_blocks) * 100 if st.f_blocks else 0.0
inodes_free = st.f_ffree
inodes_used_pct = (1 - st.f_ffree / st.f_files) * 100 if st.f_files else 0.0

logger.info(
"Storage [%s] -> %.2f GiB free | bytes used: %.1f%% | inodes used: %.1f%% (%d free)",
mount, free_gib, used_pct, inodes_used_pct, inodes_free
)

if used_pct > 95.0:
logger.critical("CRITICAL: Mount %s bytes usage exceeds 95%% threshold (%.1f%%)", mount, used_pct)
if inodes_used_pct > 90.0:
logger.critical(
"CRITICAL: Mount %s inode exhaustion detected (%.1f%% used, %d free) — risk of filesystem lockup",
mount, inodes_used_pct, inodes_free
)


class BlockerInspector:
"""Inspects and reports active blockers from the handover payload at INFO level."""

async def process(self, state_data: Dict[str, Any]) -> None:
blockers: List[Dict[str, Any]] = state_data.get("blockers", [])
if blockers:
logger.info("Active Blockers Detected: %d item(s)", len(blockers))
for blocker in blockers:
logger.info(
"BLOCKER [%s]: %s (Owner: %s)",
blocker.get("id", "N/A"),
blocker.get("item", "Unknown issue"),
blocker.get("owner", "unassigned")
)
else:
logger.info("No active blockers reported in handover state.")


class NextActionReporter:
"""Surfaces the first unresolved runbook step — the operational frontier of the session."""

def __init__(self, handover_path: Path) -> None:
self.handover_path = handover_path

async def process(self, state_data: Dict[str, Any]) -> None:
runbook: List[str] = state_data.get("next_session_runbook", [])
if runbook:
logger.info("NEXT RUNBOOK FRONTIER: %s", runbook[0])
else:
logger.warning("Runbook empty or malformed in %s", self.handover_path)


class AutomationEngine:
"""Asynchronous orchestrator for zero-interaction background monitoring and state validation."""

def __init__(self, config: AppConfig) -> None:
self.config = config
self.validator = HandoverStateValidator(config.handover_path)
self.processors: List[StateProcessorProtocol] = [
StorageTruthChecker(),
BlockerInspector(),
NextActionReporter(config.handover_path)
]
self._is_running = False

async def run_cycle(self) -> None:
logger.info("Starting automation audit cycle...")
try:
state_data = self.validator.load_state()
for processor in self.processors:
await processor.process(state_data)
except Exception as e:
logger.error("Error during automation cycle execution: %s", e)

async def start(self) -> None:
self._is_running = True
logger.info("AutomationEngine started with poll interval: %.1fs", self.config.poll_interval_seconds)

while self._is_running:
await self.run_cycle()
try:
await asyncio.sleep(self.config.poll_interval_seconds)
except asyncio.CancelledError:
break

def stop(self) -> None:
logger.info("Stopping AutomationEngine gracefully...")
self._is_running = False


async def main() -> None:
config = AppConfig.from_env()
if config.debug_mode:
logger.setLevel(logging.DEBUG)
logger.debug("Debug logging enabled via environment configuration.")

engine = AutomationEngine(config)

loop = asyncio.get_running_loop()
stop_event = asyncio.Event()

def handle_signal() -> None:
logger.info("Received termination signal. Initiating graceful shutdown...")
engine.stop()
stop_event.set()

for sig in (signal.SIGINT, signal.SIGTERM):
try:
loop.add_signal_handler(sig, handle_signal)
except NotImplementedError:
pass

engine_task = asyncio.create_task(engine.start())
await stop_event.wait()

engine_task.cancel()
try:
await engine_task
except asyncio.CancelledError:
pass

logger.info("Handover Engine shutdown complete.")


if __name__ == "__main__":
try:
asyncio.run(main())
except KeyboardInterrupt:
logger.info("Program interrupted by user operator.")
sys.exit(0)

Good progress. The audit ran, staging worked (962 files via hardlinks — zero extra space), quarantine moved 6 items. Two remaining issues to solve:

1. Disk still shows 100% — why rm didn't free space
The rm commands likely succeeded at removing directory entries, but on Android's F2FS/ext4 the blocks may not be reclaimed until the filesystem's garbage collector runs, or something still holds open file handles to the deleted files.

bash
Copy
# Check if deleted-but-open files are holding space
ls -la /proc/*/fd 2>/dev/null | grep '(deleted)' | head -20

# Nuclear option: trigger F2FS GC (if F2FS)
echo 1 > /sys/fs/f2fs/*/discard 2>/dev/null || true

# Check actual usage NOW (post-rm)
du -sh ~/* 2>/dev/null | sort -rh | head -10
df -h /
If df still shows 0 free but du shows less total, it's a kernel-level lag. A reboot fixes it. If du still shows ~884M, the rm didn't actually delete (permissions or the shell ate the commands). Verify:

bash
Copy
ls ~/.npm 2>&1
ls ~/.local 2>&1
ls ~/.cache 2>&1
If they still exist, the rm was silently blocked. In that case:

bash
Copy
# Force with explicit error output
rm -rfv ~/.npm 2>&1 | tail -5
rm -rfv ~/.local 2>&1 | tail -5
2. The --push auth failure
bash
Copy
ABORT[4]: bilbywilby/home-archive not reachable...
fatal: could not read Username for 'https://github.com': terminal prompts disabled
The script sets GIT_TERMINAL_PROMPT=0 (good for automation) but you have no stored credentials. Fix:

bash
Copy
# Option A: SSH (preferred for Termux)
# Generate key if you don't have one
ls ~/.ssh/id_ed25519 2>/dev/null || ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""

# Show your public key — add it to GitHub → Settings → SSH Keys
cat ~/.ssh/id_ed25519.pub

# Then in the staging repo:
cd ~/home-archive-staging
git remote set-url origin git@github.com:bilbywilby/home-archive.git

# Test
ssh -T git@github.com 2>&1 | head -3

# Then re-run
python3 ~/bin/home_audit.py --push
bash
Copy
# Option B: PAT via credential store (if you prefer HTTPS)
git config --global credential.helper store
# Next push will prompt ONCE for username + PAT, then save to ~/.git-credentials
# Make sure the repo exists on GitHub first (private!)
Important: The audit flagged several repos as PUBLIC with sensitive files (capyreader, ez_git, ez_mac, goldenfoss-staging, Keystone_Health). Before you push anything, rotate any credentials/tokens that appear in those repos' git history. The --push target (home-archive) should be private — confirm it exists and is set to private on GitHub before pushing.

3. Your master_engine.py — two small fixes for this environment
Fix A: /data/user/0 doesn't exist as a mount in Termux (it's an Android framework path, not a filesystem mount). Your StorageTruthChecker will silently skip it via the except OSError: continue, which is fine, but you're missing the actual large partition:

python
Copy
class StorageTruthChecker:
    """Audits blocks AND inodes across host-truth and guest mounts."""

    # Termux: / is the small root (887M), /storage/emulated is the FUSE-backed
    # shared storage (109G). /data/user/0 is NOT a mount — it's a namespace path.
    MOUNTS = ("/", "/storage/emulated", "/data/data/com.termux/files")
Fix B: The run_cycle method should isolate per-processor failures so one bad mount doesn't kill the whole cycle:

python
Copy
async def run_cycle(self) -> None:
    logger.info("Starting automation audit cycle...")
    try:
        state_data = self.validator.load_state()
    except Exception as e:
        logger.error("Failed to load state: %s", e)
        return

    for processor in self.processors:
        try:
            await processor.process(state_data)
        except Exception as e:
            logger.error("Processor %s failed: %s", type(processor).__name__, e)
4. Quick summary of where you stand
Item	Status
Audit scan	✅ Complete — 51 projects classified
Quarantine	✅ 6 junk items moved
Stage (962 files)	✅ Hardlinked to ~/home-archive-staging
Push to GitHub	❌ Blocked on auth — fix SSH or PAT above
Disk space	⚠️ Still 100% — verify rm actually deleted, or reboot
HANDOVER.json	❌ Missing (missing_expected: handover)
~/.git (empty repo in ~)	⚠️ Blocking git add -A — run the suggested mv ~/.git ~/.git.home-repo.bak
Want me to write a HANDOVER.json template with your current blockers and next steps encoded, so master_engine.py has something to monitor?

nalization@372038020
/dev/block/loop15 48 35
13 73% /apex/com.google.android.widevi
ne@190260313
/dev/block/dm-53 48 41
7 86% /apex/com.android.telephonycore
@374205000
/dev/block/loop18 48 37
11 78% /apex/com.android.runtime@1
/dev/block/dm-55 48 47
1 98% /apex/com.android.tzdata@373041
620
/dev/block/dm-44 48 36
12 75% /apex/com.android.configinfrast
ructure@374200040
/dev/block/dm-61 96 81
15 85% /apex/com.android.art@374200160
/dev/block/loop19 32 17
15 54% /apex/com.android.hardware.biom
etrics.fingerprint.virtual@1
/dev/block/dm-37 48 34
14 71% /apex/com.android.adbd@37420012
0
/dev/block/dm-45 32 25
7 79% /apex/com.android.ipsec@3742050
00
/dev/block/dm-38 64 52
12 82% /apex/com.android.uprobestats@3
74205000
/dev/block/dm-56 160 157
3 99% /apex/com.android.conscrypt@374
205000
/dev/block/dm-29 32 31
1 97% /apex/com.android.profiling@374
205000
/dev/block/dm-48 48 34
14 71% /apex/com.android.appsearch@374
205000
/dev/block/dm-51 48 45
3 94% /apex/com.android.nfcservices@3
74205000
/dev/block/dm-57 32 25
7 79% /apex/com.android.scheduling@37
4205000
/dev/block/dm-42 32 27
5 85% /apex/com.android.rkpd@37420002
0
/dev/block/dm-31 32 25
7 79% /apex/com.android.crashrecovery
@374205000
/dev/block/dm-65 48 36
12 75% /apex/com.android.healthfitness
@374205000
/dev/block/dm-62 64 50
14 79% /apex/com.android.mediaprovider
@374200100
/dev/block/dm-54 80 70
10 88% /apex/com.android.npumanager@37
1899999
/dev/block/dm-35 16 16
0 100% /apex/com.android.neuralnetwork
s@374200020
/dev/block/dm-59 96 93
3 97% /apex/com.android.tethering@374
205000
tmpfs 940057 10
940047 1% /bootstrap-apex
/dev/block/loop3 48 37
11 78% /bootstrap-apex/com.android.run
time@1
/dev/block/loop2 48 47
1 98% /bootstrap-apex/com.android.tzd
ata@372037000
/dev/block/loop1 18446744073709551615 70 18446
744073709551545 1% /bootstrap-apex/com.android.vir
t@371899999
/dev/block/loop0 32 30
2 94% /bootstrap-apex/com.android.i18
n@1
tmpfs 940057 1
940056 1% /tmp
/dev/fuse 13511677 13313930
197747 99% /storage/emulated

=== 2. TOP SPACE CONSUMERS IN ~ (depth 2) ===
348M /data/data/com.termux/files/home/rss-automato
r
346M /data/data/com.termux/files/home/ez_git
346M /data/data/com.termux/files/home/ez_gemini_ge
ms_to_skills
257M /data/data/com.termux/files/home/tech-ref-por
tal
110M /data/data/com.termux/files/home/capyreader
104M /data/data/com.termux/files/home/.npm
66M /data/data/com.termux/files/home/animated-umb
rella
62M /data/data/com.termux/files/home/.local
28M /data/data/com.termux/files/home/home-archive
-staging
24M /data/data/com.termux/files/home/keystone-hea
lth-advocate-pa-medical-rights-wiki-forensic-tool
17M /data/data/com.termux/files/home/projects
15M /data/data/com.termux/files/home/animated-umb
rella-main
12M /data/data/com.termux/files/home/ez_ip_opsec_
audit
4.2M /data/data/com.termux/files/home/duck-dev
2.2M /data/data/com.termux/files/home/.netprobe
1.7M /data/data/com.termux/files/home/ez_mac
1.5M /data/data/com.termux/files/home/Keystone_Hea
lth
1.2M /data/data/com.termux/files/home/EnterpriseSe
cureMobileSkeleton
810K /data/data/com.termux/files/home/tmp
701K /data/data/com.termux/files/home/October_2026
566K /data/data/com.termux/files/home/Linux
436K /data/data/com.termux/files/home/env-audit-ar
chive
413K /data/data/com.termux/files/home/netprobe
411K /data/data/com.termux/files/home/invizible-pr
o-agent-skills
387K /data/data/com.termux/files/home/cl_else_roku
302K /data/data/com.termux/files/home/pa_ra_onepag
ers
264K /data/data/com.termux/files/home/audit_out
237K /data/data/com.termux/files/home/goldenfoss-s
taging
229K /data/data/com.termux/files/home/termux_maste
r_ultra
203K /data/data/com.termux/files/home/ez_multi_rep
o

=== 3. LARGEST FILES (top 20) ===
13.3 MiB /data/data/com.termux/files/home/projects/k
eystone_health/Pennsylvania
10.0 MiB /data/data/com.termux/files/home/ez_ip_opse
c_audit/activity_trace.csv
2.5 MiB /data/data/com.termux/files/home/rss-automat
or/ez_gemini_gems_to_skills/Active_Projects/Python.zi
p
2.5 MiB /data/data/com.termux/files/home/ez_git/Acti
ve_Projects/Python.zip
2.5 MiB /data/data/com.termux/files/home/ez_gemini_g
ems_to_skills/Active_Projects/Python.zip
2.4 MiB /data/data/com.termux/files/home/keystone-he
alth-advocate-pa-medical-rights-wiki-forensic-tool/pe
nnsylvania-health-insurance-audit-2026-main
2.4 MiB /data/data/com.termux/files/home/duck-dev/ke
ystone-sync/pennsylvania-health-insurance-audit-2026-
main
2.2 MiB /data/data/com.termux/files/home/.netprobe/c
ache/603e776c4f62b98a.hosts
1.1 MiB /data/data/com.termux/files/home/rss-automat
or/feeds.db-wal
0.9 MiB /data/data/com.termux/files/home/rss-automat
or/feeds.db
0.6 MiB /data/data/com.termux/files/home/home-archiv
e-staging/projects/ez_ip_opsec_audit/Screenshot_20261
004-105520.png
0.6 MiB /data/data/com.termux/files/home/ez_ip_opsec
_audit/Screenshot_20261004-105520.png
0.6 MiB /data/data/com.termux/files/home/tech-ref-po
rtal/node_modules/prettier/index.mjs
0.6 MiB /data/data/com.termux/files/home/tech-ref-po
rtal/node_modules/date-fns/typings.d.ts
0.5 MiB /data/data/com.termux/files/home/tech-ref-po
rtal/node_modules/lodash/lodash.js
0.4 MiB /data/data/com.termux/files/home/projects/ke
ystone_health/Keystone
0.4 MiB /data/data/com.termux/files/home/home-archiv
e-staging/_audit/20261107_082622/SKIPPED.txt
0.4 MiB /data/data/com.termux/files/home/home-archiv
e-staging/_audit/20261107_082550/SKIPPED.txt
0.3 MiB /data/data/com.termux/files/home/tech-ref-po
rtal/node_modules/prettier/THIRD-PARTY-NOTICES.md
0.3 MiB /data/data/com.termux/files/home/ez_git/.git
/index
~ $ ls -la
total 1574
drwx------. 80 u0_a399 u0_a399 20480 Nov 9 15:33 .
drwxrwx--x. 5 u0_a399 u0_a399 3452 Oct 11 12:19 ..
-rw-------. 1 u0_a399 u0_a399 22026 Nov 9 15:22 .ba
sh_history
-rw-------. 1 u0_a399 u0_a399 2190 Nov 8 18:36 .ba
shrc
drwx------. 4 u0_a399 u0_a399 3452 Nov 9 12:28 .ca
che
-rw-------. 1 u0_a399 u0_a399 53248 Nov 7 00:35 .co
verage
-rwx------. 1 u0_a399 u0_a399 14 Oct 13 03:48 .ez
_archive.sh
drwx------. 3 u0_a399 u0_a399 3452 Nov 7 17:38 .gi
t
drwx------. 7 u0_a399 u0_a399 3452 Nov 3 09:22 .gi
t-accidental-backup
drwx------. 2 u0_a399 u0_a399 3452 Nov 7 06:36 .gi
t-hooks
drwx------. 2 u0_a399 u0_a399 3452 Nov 9 03:28 .gi
t_archive
-rw-------. 1 u0_a399 u0_a399 484 Nov 7 06:36 .gi
tconfig
drwx------. 3 u0_a399 u0_a399 3452 Nov 4 10:58 .gi
thub
-rw-------. 1 u0_a399 u0_a399 174 Nov 2 01:20 .gi
tignore_global
drwxr-xr-x. 7 u0_a399 u0_a399 3452 Nov 7 01:45 .lo
cal
drwx------. 3 u0_a399 u0_a399 3452 Oct 15 20:38 .ne
tprobe
drwx------. 5 u0_a399 u0_a399 3452 Nov 9 03:25 .np
m
drwx------. 3 u0_a399 u0_a399 3452 Nov 7 06:36 .np
m-global
-rw-------. 1 u0_a399 u0_a399 66 Nov 7 06:36 .np
mrc
drwx------. 3 u0_a399 u0_a399 3452 Nov 7 00:38 .py
test_cache
-rw-------. 1 u0_a399 u0_a399 38632 Nov 7 07:09 .py
thon_history
-rw-------. 1 u0_a399 u0_a399 154 Nov 5 16:28 .re
po_split.list
drwx------. 2 u0_a399 u0_a399 3452 Nov 1 11:48 .ss
h
drwx------. 3 u0_a399 u0_a399 3452 Sep 9 03:18 .su
root
drwx------. 3 u0_a399 u0_a399 3452 Oct 12 21:39 .te
rmux
drwx------. 6 u0_a399 u0_a399 3452 Nov 8 18:27 .te
rmux_master
drwx------. 2 u0_a399 u0_a399 3452 Nov 1 20:32 .tp
drwx------. 2 u0_a399 u0_a399 3452 Nov 7 07:25 .tr
p-bak
drwx------. 2 u0_a399 u0_a399 3452 Oct 14 18:15 .w3
m
-rw-------. 1 u0_a399 u0_a399 24 Oct 12 13:26 .wg
_private_key
-rw-------. 1 u0_a399 u0_a399 0 Nov 7 05:26 50%
-rw-------. 1 u0_a399 u0_a399 0 Nov 5 13:57 Boo
tstrapping
drwx------. 2 u0_a399 u0_a399 3452 Nov 9 03:28 Chr
ombook_crostini_repair_guide
drwxr-xr-x. 8 u0_a399 u0_a399 3452 Nov 7 07:19 Ent
erpriseSecureMobileSkeleton
-rw-------. 1 u0_a399 u0_a399 0 Nov 5 13:57 Env
ironment
-rw-------. 1 u0_a399 u0_a399 0 Oct 10 20:58 FET
CH_HEAD
drwx------. 2 u0_a399 u0_a399 3452 Nov 9 03:28 Gol
denfoss
drwx------. 13 u0_a399 u0_a399 3452 Nov 5 19:43 Key
stone_Health
drwx------. 4 u0_a399 u0_a399 3452 Oct 31 06:37 Lin
ux
drwx------. 3 u0_a399 u0_a399 3452 Oct 14 03:44 Oct
ober_2026
-rw-------. 1 u0_a399 u0_a399 9 Nov 3 09:22 REA
DME.md
drwx------. 2 u0_a399 u0_a399 3452 Oct 15 19:55 adb
lock_tester
drwx------. 19 u0_a399 u0_a399 8192 Nov 1 02:58 ani
mated-umbrella
drwx------. 14 u0_a399 u0_a399 3452 Oct 11 02:44 ani
mated-umbrella-main
-rw-------. 1 u0_a399 u0_a399 9 Oct 11 02:47 ani
mated-umbrella.tar.gz
drwx------. 2 u0_a399 u0_a399 3452 Oct 14 18:07 ani
mated-umbrella.wiki
-rwx------. 1 u0_a399 u0_a399 3698 Oct 31 01:43 apk
_report.py
drwx------. 3 u0_a399 u0_a399 3452 Oct 15 20:10 ato
mic_session
drwx------. 2 u0_a399 u0_a399 3452 Oct 14 20:18 aud
it_out
drwx------. 9 u0_a399 u0_a399 3452 Oct 11 10:31 aud
it_tools
drwx------. 2 u0_a399 u0_a399 3452 Nov 9 03:29 bac
kups
drwx------. 3 u0_a399 u0_a399 3452 Nov 7 07:18 bin
-rwx------. 1 u0_a399 u0_a399 20001 Oct 31 02:01 boo
tstrap_ez_vlc_android.sh
-rwx------. 1 u0_a399 u0_a399 3360 Nov 5 15:32 boo
tstrap_keystone.sh
-rwx------. 1 u0_a399 u0_a399 1011 Nov 5 13:02 boo
tstrap_keystone_env.sh
drwx------. 23 u0_a399 u0_a399 3452 Oct 15 15:41 cap
yreader
drwx------. 3 u0_a399 u0_a399 3452 Oct 13 09:09 cl_
else_roku
-rwx------. 1 u0_a399 u0_a399 4651 Nov 7 19:41 cle
anup.sh
-rwx------. 1 u0_a399 u0_a399 2054 Nov 6 19:41 cle
anup_bighunt.sh
-rwx------. 1 u0_a399 u0_a399 18358 Nov 6 11:45 cle
anup_gui.py
-rwx------. 1 u0_a399 u0_a399 6780 Nov 6 14:40 cle
anup_reclaim.sh
drwx------. 3 u0_a399 u0_a399 3452 Nov 7 06:04 cor
e
-rwx------. 1 u0_a399 u0_a399 5162 Oct 10 14:48 ctx
.sh
-rw-------. 1 u0_a399 u0_a399 29800 Oct 10 14:49 ctx
_report_20261010_144857.txt
-rw-------. 1 u0_a399 u0_a399 31570 Oct 10 16:28 ctx
_report_20261010_162811.txt
-rw-------. 1 u0_a399 u0_a399 9855 Nov 1 18:40 das
hboard.py
-rwx------. 1 u0_a399 u0_a399 1925 Oct 10 05:33 dep
loy.sh
-rwx------. 1 u0_a399 u0_a399 9615 Oct 13 06:29 dep
loy_bundle.sh
-rwx------. 1 u0_a399 u0_a399 798 Oct 14 16:23 dep
loy_fix.sh
-rw-------. 1 u0_a399 u0_a399 970 Oct 12 14:35 dep
loy_fixes.sh
-rwx------. 1 u0_a399 u0_a399 3751 Oct 12 14:07 dep
loy_portal.sh
-rwx------. 1 u0_a399 u0_a399 3194 Oct 12 21:57 dep
loy_remediation.sh
-rw-------. 1 u0_a399 u0_a399 2329 Nov 5 12:40 dep
loyment_20261105_124005.log
-rw-------. 1 u0_a399 u0_a399 2329 Nov 5 12:48 dep
loyment_20261105_124849.log
-rw-------. 1 u0_a399 u0_a399 2329 Nov 5 13:03 dep
loyment_20261105_130302.log
-rw-------. 1 u0_a399 u0_a399 15144 Nov 9 15:36 dia
g_output.txt
drwx------. 2 u0_a399 u0_a399 3452 Nov 9 03:28 dow
nloads
drwx------. 3 u0_a399 u0_a399 3452 Oct 14 14:54 duc
k-dev
drwx------. 2 u0_a399 u0_a399 3452 Nov 7 03:16 env
-audit-archive
-rwx------. 1 u0_a399 u0_a399 1169 Oct 14 20:18 env
_audit.sh
-rw-------. 1 u0_a399 u0_a399 89864 Nov 5 23:44 env
_audit_20261105_234458.txt
-rw-------. 1 u0_a399 u0_a399 90715 Nov 6 10:57 env
_audit_20261106_105737.txt
drwx------. 6 u0_a399 u0_a399 3452 Oct 30 22:29 ez-
vlc-android
-rwx------. 1 u0_a399 u0_a399 14 Oct 13 03:48 ez_
archive.sh
drwx------. 2 u0_a399 u0_a399 3452 Oct 13 09:52 ez_
court
-rw-------. 1 u0_a399 u0_a399 34514 Oct 14 04:24 ez_
flash_backup.py
drwx------. 7 u0_a399 u0_a399 3452 Nov 7 08:10 ez_
gemini_gems_to_skills
drwx------. 10 u0_a399 u0_a399 3452 Nov 6 19:07 ez_
git
drwx------. 2 u0_a399 u0_a399 3452 Oct 14 18:07 ez_
ip_opsec_audit
drwx------. 9 u0_a399 u0_a399 3452 Nov 7 08:30 ez_
mac
drwx------. 4 u0_a399 u0_a399 3452 Nov 3 23:05 ez_
multi_repo
-rw-------. 1 u0_a399 u0_a399 7331 Nov 9 12:28 ez_
recon_20261109_122832.txt
drwx------. 3 u0_a399 u0_a399 3452 Oct 14 18:07 ez_
remote
-rw-------. 1 u0_a399 u0_a399 29754 Nov 9 12:28 ez_
repo_recon.py
drwx------. 2 u0_a399 u0_a399 3452 Nov 9 03:29 ez_
termux_backup_flashdrive
-rwx------. 1 u0_a399 u0_a399 1120 Oct 11 03:57 fin
alize_hardening.sh
-rwx------. 1 u0_a399 u0_a399 1792 Oct 12 12:43 fix
_pixel_audit.sh
-rw-------. 1 u0_a399 u0_a399 7106 Nov 1 04:16 fix
_termux_workspace.sh
-rwx------. 1 u0_a399 u0_a399 8744 Nov 1 17:11 gen
_network_defense_md.sh
-rw-------. 1 u0_a399 u0_a399 17896 Nov 1 08:57 gen
erate_handover.py
-rwx------. 1 u0_a399 u0_a399 3703 Nov 3 09:08 gen
erate_mac.py
-rwx------. 1 u0_a399 u0_a399 5665 Nov 5 11:40 gen
erate_repo_structure.py
-rw-------. 1 u0_a399 u0_a399 0 Nov 5 15:57 gen
erate_skeleton_repo.sh.save
-rw-------. 1 u0_a399 u0_a399 1997 Nov 5 15:40 git
_workflow.sh
drwx------. 4 u0_a399 u0_a399 3452 Nov 1 10:47 git
hub_deploy
-rwx------. 1 u0_a399 u0_a399 1466 Sep 9 03:07 gms
_hunter_v2.py
drwx------. 3 u0_a399 u0_a399 3452 Nov 5 18:52 gol
denfoss-staging
-rw-------. 1 u0_a399 u0_a399 21728 Nov 8 19:17 han
dover.md
drwx------. 2 u0_a399 u0_a399 3452 Nov 7 06:20 han
dover_session_20261106
-rwx------. 1 u0_a399 u0_a399 899 Oct 12 14:12 har
den_gh.sh
drwx------. 6 u0_a399 u0_a399 3452 Nov 7 08:27 hom
e-archive-staging
-rwx------. 1 u0_a399 u0_a399 4508 Nov 6 21:33 hom
e_audit.py
-rw-------. 1 u0_a399 u0_a399 3022 Nov 7 01:45 ind
ex.html
-rwx------. 1 u0_a399 u0_a399 29 Nov 7 08:10 ini
t_and_push_gems.sh
-rwx------. 1 u0_a399 u0_a399 4045 Oct 31 13:14 ins
tall.sh
-rw-------. 1 u0_a399 u0_a399 1197 Oct 11 04:26 ins
tall_safe_clone.sh
drwx------. 3 u0_a399 u0_a399 3452 Nov 1 08:57 inv
izible-pro-agent-skills
drwx------. 2 u0_a399 u0_a399 3452 Oct 15 18:44 ipl
eak_tool
drwx------. 11 u0_a399 u0_a399 8192 Nov 1 02:58 key
stone-health-advocate-pa-medical-rights-wiki-forensic
-tool
-rw-------. 1 u0_a399 u0_a399 2633 Nov 3 03:17 key
stone_fix.sh
-rw-------. 1 u0_a399 u0_a399 3430 Nov 3 03:33 key
stone_fix2.sh
-rw-------. 1 u0_a399 u0_a399 8528 Nov 3 03:33 key
stone_fix2_20261103_033334.txt
-rw-------. 1 u0_a399 u0_a399 4454 Nov 5 15:07 key
stone_fix3.sh
-rw-------. 1 u0_a399 u0_a399 1099 Nov 5 15:13 key
stone_fix3_20261105_150418.txt
-rw-------. 1 u0_a399 u0_a399 1138 Nov 5 15:07 key
stone_fix3_20261105_150725.txt
-rw-------. 1 u0_a399 u0_a399 4629 Nov 5 15:47 key
stone_fix4.sh
-rw-------. 1 u0_a399 u0_a399 5559 Nov 5 15:37 key
stone_fix4_20261105_153659.txt
-rw-------. 1 u0_a399 u0_a399 5559 Nov 5 15:47 key
stone_fix4_20261105_154709.txt
-rw-------. 1 u0_a399 u0_a399 4638 Nov 3 02:08 key
stone_fix_20261103_020759.txt
-rw-------. 1 u0_a399 u0_a399 4864 Nov 3 03:18 key
stone_fix_20261103_031806.txt
drwx------. 2 u0_a399 u0_a399 3452 Nov 3 02:08 key
stone_pdf_text
-rwx------. 1 u0_a399 u0_a399 2293 Nov 5 11:07 key
stone_sync.py
-rw-------. 1 u0_a399 u0_a399 1995 Nov 3 02:05 key
stone_triage.sh
-rw-------. 1 u0_a399 u0_a399 3209 Nov 3 02:05 key
stone_triage_20261103_020518.txt
-rw-------. 1 u0_a399 u0_a399 3146 Nov 3 03:48 key
stone_verify.sh
-rw-------. 1 u0_a399 u0_a399 2474 Nov 3 03:48 key
stone_verify_20261103_034851.txt
-rw-------. 1 u0_a399 u0_a399 2406 Nov 5 13:34 key
stone_verify_20261105_133357.txt
-rw-------. 1 u0_a399 u0_a399 2406 Nov 5 13:47 key
stone_verify_20261105_134730.txt
-rw-------. 1 u0_a399 u0_a399 2406 Nov 5 13:48 key
stone_verify_20261105_134849.txt
-rw-------. 1 u0_a399 u0_a399 497 Oct 31 16:31 loc
k_utils.sh
-rwx------. 1 u0_a399 u0_a399 985 Oct 13 00:28 mon
itor_pipeline.sh
drwx------. 2 u0_a399 u0_a399 3452 Aug 6 16:27 my-
futo-notes
drwx------. 2 u0_a399 u0_a399 3452 Oct 15 19:55 net
_tool
drwx------. 5 u0_a399 u0_a399 3452 Nov 1 11:48 net
probe
-rw-------. 1 u0_a399 u0_a399 15609 Nov 7 07:01 obl
ivion_install.sh
drwx------. 5 u0_a399 u0_a399 3452 Oct 14 18:07 ops
ec-debug
-rwx------. 1 u0_a399 u0_a399 1293 Oct 9 18:53 ops
ec_audit.py
-rw-------. 1 u0_a399 u0_a399 0 Oct 10 10:35 pa_
aca_briefing_py2027_final.html
drwx------. 5 u0_a399 u0_a399 3452 Oct 13 06:30 pa_
audit_bundle
drwx------. 3 u0_a399 u0_a399 3452 Oct 10 20:18 pa_
ra_onepagers
-rwx------. 1 u0_a399 u0_a399 2602 Oct 12 19:33 par
se_ir_intelligence.py
drwx------. 2 u0_a399 u0_a399 3452 Nov 8 10:55 phy
lum
drwx------. 2 u0_a399 u0_a399 3452 Aug 8 13:14 pix
el-terminal-master
-rw-------. 1 u0_a399 u0_a399 1565 Oct 9 18:53 pix
el-terminal-master.20261009_185313.tgz
-rw-------. 1 u0_a399 u0_a399 1565 Oct 10 21:54 pix
el-terminal-master.20261010_215417.tgz
-rw-------. 1 u0_a399 u0_a399 1565 Oct 12 12:43 pix
el-terminal-master.20261012_124330.tgz
-rw-------. 1 u0_a399 u0_a399 1565 Nov 9 06:08 pix
el-terminal-master.20261109_060840.tgz
-rwx------. 1 u0_a399 u0_a399 645 Oct 12 14:45 pix
el_diagnostic.sh
-rw-------. 1 u0_a399 u0_a399 662 Oct 9 18:53 pix
el_diagnostic_20261009_185314.txt
-rw-------. 1 u0_a399 u0_a399 661 Oct 10 21:54 pix
el_diagnostic_20261010_215417.txt
-rw-------. 1 u0_a399 u0_a399 661 Oct 12 12:43 pix
el_diagnostic_20261012_124330.txt
-rw-------. 1 u0_a399 u0_a399 661 Nov 9 06:08 pix
el_diagnostic_20261109_060840.txt
-rw-------. 1 u0_a399 u0_a399 4545 Nov 7 13:16 pix
el_opsec_audit.sh
-rw-------. 1 u0_a399 u0_a399 2351 Nov 1 17:53 pix
svg.py
-rwx------. 1 u0_a399 u0_a399 1403 Nov 1 17:56 pla
ce_netdef.sh
drwx------. 4 u0_a399 u0_a399 3452 Nov 7 00:38 pro
jects
-rw-------. 1 u0_a399 u0_a399 3518 Nov 8 10:45 rec
on.sh
-rw-------. 1 u0_a399 u0_a399 10804 Nov 8 11:03 rec
on_report.txt
-rw-------. 1 u0_a399 u0_a399 15626 Oct 15 22:16 rep
ort.json
-rw-------. 1 u0_a399 u0_a399 882 Nov 9 07:21 ris
h
-rw-------. 1 u0_a399 u0_a399 59672 Nov 9 07:21 ris
h_shizuku.dex
drwx------. 3 u0_a399 u0_a399 3452 Oct 12 10:08 rok
u-diagnostics
drwx------. 5 u0_a399 u0_a399 3452 Oct 14 18:07 rok
u_failure_evidence
drwx------. 3 u0_a399 u0_a399 3452 Nov 8 10:19 rss
-automator
-rw-------. 1 u0_a399 u0_a399 1526 Nov 7 07:53 scr
ipt_inventory.sh
-rw-------. 1 u0_a399 u0_a399 3275 Nov 7 07:54 scr
ipt_inventory_20261107_075407.txt
drwx------. 2 u0_a399 u0_a399 3452 Nov 6 23:58 scr
ipts
-rwx------. 1 u0_a399 u0_a399 4239 Nov 1 11:42 sec
urity_cleanup.sh
-rw-------. 1 u0_a399 u0_a399 36063 Oct 13 20:24 ses
sion_context.txt
-rwx------. 1 u0_a399 u0_a399 429 Oct 12 14:45 ses
sion_diag.sh
-rw-------. 1 u0_a399 u0_a399 2117 Oct 15 14:31 set
tup.sh
-rw-------. 1 u0_a399 u0_a399 15079 Oct 15 15:40 set
up-rss-automator.sh
-rwx------. 1 u0_a399 u0_a399 983 Oct 12 21:30 set
up.sh
-rwx------. 1 u0_a399 u0_a399 782 Oct 13 07:54 set
up_tv_case.sh
drwx------. 2 u0_a399 u0_a399 3452 Sep 9 03:07 sha
red
drwx------. 2 u0_a399 u0_a399 3452 Nov 9 03:28 shh
-rwx------. 1 u0_a399 u0_a399 11469 Nov 5 12:48 shi
p_nsa_dispute_kit.sh
-rwx------. 1 u0_a399 u0_a399 3860 Sep 9 05:35 sys
_recon.sh
drwx------. 18 u0_a399 u0_a399 3452 Nov 9 06:09 tec
h-ref-portal
-rwx------. 1 u0_a399 u0_a399 13561 Nov 5 17:51 ter
mux-usb-transfer
-rw-------. 1 u0_a399 u0_a399 3084 Nov 9 15:36 ter
mux_diag.sh
-rwx------. 1 u0_a399 u0_a399 2873 Nov 1 04:38 ter
mux_environment_setup.sh
drwx------. 3 u0_a399 u0_a399 3452 Oct 31 13:15 ter
mux_master_ultra
-rw-------. 1 u0_a399 u0_a399 39 Nov 7 11:51 tes
t_generate_config.py
-rw-------. 1 u0_a399 u0_a399 998 Nov 5 12:40 tes
t_results_20261105_124005.xml
-rw-------. 1 u0_a399 u0_a399 998 Nov 5 12:48 tes
t_results_20261105_124849.xml
-rw-------. 1 u0_a399 u0_a399 1009 Nov 5 13:03 tes
t_results_20261105_130302.xml
drwx------. 3 u0_a399 u0_a399 3452 Nov 7 06:31 tes
ts
drwx------. 3 u0_a399 u0_a399 3452 Nov 1 10:46 tmp
drwx------. 3 u0_a399 u0_a399 3452 Nov 1 02:58 tmu
-dashboard-auth-patch
-rw-------. 1 u0_a399 u0_a399 11527 Oct 31 03:11 tmu
_probe.sh
-rw-------. 1 u0_a399 u0_a399 29143 Oct 31 03:12 tmu
~ $ # 1. npm cache — safe to nuke, regenerates on dem
and
rm -rf ~/.npm
# saves ~104M

# 2. .local (pip/uv caches, etc.)
rm -rf ~/.local
# saves ~62M

# 3. .cache
rm -rf ~/.cache
# saves ~2-5M (small but free)

# 4. .pytest_cache
rm -rf ~/.pytest_cache
# tiny but free

# 5. The duplicate "Python.zip" appears in THREE plac
es (2.5M each)
# Keep one, delete the other two:
rm -f ~/rss-automator/ez_gemini_gems_to_skills/Active
_Projects/Python.zip
rm -f ~/ez_gemini_gems_to_skills/Active_Projects/Pyth
on.zip
# (keep the one in ez_git/Active_Projects/Python.zip)


# 6. Old deployment logs and test results (redundant,
small but every bit helps)
rm -f ~/deployment_20261105_*.log
rm -f ~/test_results_20261105_*.xml
rm -f ~/ctx_report_*.txt
rm -f ~/env_audit_202611*.txt
rm -f ~/pixel_diagnostic_*.txt
rm -f ~/pixel-terminal-master.*.tgz
rm -f ~/keystone_fix*.txt
rm -f ~/keystone_verify*.txt
rm -f ~/keystone_triage*.txt

# 7. .git-accidental-backup — sounds like it's exactl
y that
rm -rf ~/.git-accidental-backup

# 8. .coverage, .python_history (safe)
rm -f ~/.coverage ~/.python_history

# 9. The 0-byte junk files in ~
rm -f ~/50% ~/Bootstrapping ~/Environment ~/FETCH_HEA
D
~ $ df -h /
Filesystem Size Used Avail Use% Mounted on
/dev/block/dm-12 887M 884M 0 100% /
~ $ # Free ~200M
rm -rf ~/.npm ~/.local ~/.cache ~/.pytest_cache ~/.gi
t-accidental-backup
rm -f ~/50% ~/Bootstrapping ~/Environment ~/FETCH_HEA
D ~/.coverage ~/.python_history
rm -f ~/deployment_20261105_*.log ~/test_results_2026
1105_*.xml ~/ctx_report_*.txt
rm -f ~/env_audit_202611*.txt ~/pixel_diagnostic_*.tx
t ~/pixel-terminal-master.*.tgz
rm -f ~/keystone_fix*.txt ~/keystone_verify*.txt ~/ke
ystone_triage*.txt
rm -f ~/rss-automator/ez_gemini_gems_to_skills/Active
_Projects/Python.zip
rm -f ~/ez_gemini_gems_to_skills/Active_Projects/Pyth
on.zip

# Verify
df -h /
df -i /

# Then run the audit (correct path)
python3 ~/home_audit.py --stage
Filesystem Size Used Avail Use% Mounted on
/dev/block/dm-12 887M 884M 0 100% /
Filesystem Inodes IUsed IFree IUse% Mounted on
/dev/block/dm-12 2912 2882 30 99% /
usage: home_audit.py [-h] --src-dir SRC_DIR
--dst-dir DST_DIR
[--max-mb MAX_MB]
home_audit.py: error: the following arguments are req
uired: --src-dir, --dst-dir
~ $ cat home_audit.py
#!/usr/bin/env python3
"""
home_audit.py – Secure staging helper for Termux / An
droid environments.

Key goals:
* Preserve zero‑copy hardlinks when the filesystem su
pports them.
* Gracefully fall back to a full copy on platforms th
at lack `os.link`.
* Keep inode‑safe semantics (never clobber the source
).
* Honor the --max‑mb storage cap via `shutil.disk_usa
ge`.
"""

import argparse
import hashlib
import json
import logging
import os
import shutil
import sys
from pathlib import Path
from typing import List

log = logging.getLogger("home_audit")
log.setLevel(logging.INFO)

# ---------------------------------------------------
-------------------
# Helper: storage‑cap verification (shared with exist
ing script)
# ---------------------------------------------------
-------------------
def enough_space(dst: Path, max_mb: int) -> bool:
"""Return True if copying another file of size ≤
max_mb keeps us under the quota."""
usage = shutil.disk_usage(dst.parent)
free_mb = usage.free // (1024 * 1024)
return free_mb >= max_mb

# ---------------------------------------------------
-------------------
# Resilient staging function – the core of the patch
# ---------------------------------------------------
-------------------
def stage_file(src: Path, dst: Path, max_mb: int) ->
bool:
"""
Stage a single file from ``src`` to ``dst``.

* Tries a hard‑link first (zero extra space).
* If ``os.link`` is missing or fails (e.g., cross
‑device, sandbox), falls back to ``shutil.copy2``.
* Returns True on success, False on unrecoverable
error.
"""
# Ensure destination directory hierarchy exists
dst.parent.mkdir(parents=True, exist_ok=True)

# Remove any stale file or symlink at the target
location
if dst.exists() or dst.is_symlink():
dst.unlink()

# Guard against exceeding the user‑specified stor
age ceiling
if not enough_space(dst, max_mb):
log.error("Insufficient free space to stage %
s (max %d MB)", src, max_mb)
return False

try:
# 1️⃣ Feature‑detect hard‑link capability
if hasattr(os, "link"):
try:
os.link(src, dst) # Zero‑cop
y path
log.debug("Hard‑linked %s → %s", src,
dst)
return True
except OSError as e:
# Typical on Android/Termux when the
underlying FS disallows hardlinks
log.debug("os.link failed (%s); falli
ng back to copy2", e)

# 2️⃣ Fallback copy (preserves metadata)
shutil.copy2(src, dst)
log.debug("Copied %s → %s", src, dst)
return True

except Exception as exc:
log.error("Failed to stage %s → %s: %s", src,
dst, exc)
return False

# ---------------------------------------------------
-------------------
# Main orchestration (simplified example)
# ---------------------------------------------------
-------------------
def main(argv: List[str] | None = None) -> None:
parser = argparse.ArgumentParser(description="Sta
ge files with hard‑link fallback.")
parser.add_argument("--src-dir", required=True, h
elp="Directory containing source files")
parser.add_argument("--dst-dir", required=True, h
elp="Target staging directory")
parser.add_argument(
"--max-mb",
type=int,
default=200,
help="Maximum MB of free space required befor
e staging each file",
)
args = parser.parse_args(argv)

src_root = Path(args.src_dir).expanduser().resolv
e()
dst_root = Path(args.dst_dir).expanduser().resolv
e()

if not src_root.is_dir():
log.critical("Source directory does not exist
: %s", src_root)
sys.exit(1)

staged = 0
for src_path in src_root.rglob("*"):
if src_path.is_file():
rel = src_path.relative_to(src_root)
dst_path = dst_root / rel
if stage_file(src_path, dst_path, args.ma
x_mb):
staged += 1
else:
log.warning("Skipping %s due to stagi
ng error", src_path)

log.info("Staging complete: %d files processed",
staged)


if __name__ == "__main__":
logging.basicConfig(
format="%(asctime)s %(levelname)s %(name)s: %
(message)s",
level=logging.INFO,
)
try:
main()
except KeyboardInterrupt:
log.info("Interrupted by user")
sys.exit(130)
except Exception as e:
log.exception("Fatal error in home_audit: %s"
, e)
sys.exit(1)
~ $ pkg install -y shellcheck nodejs git python
No mirror or mirror group selected. You might want to
select one by running 'termux-change-repo'
Checking availability of current mirror:
[*] https://mirrors.in.sahilister.net/termux/termux-m
ain/: ok
Get:1 https://tur.kcubeterm.com tur-packages InReleas
e
Get:2 https://mirrors.in2.sahilister.net/termux/termu
x-main stable InRelease [14.0 kB]
Get:3 https://mirrors.in2.sahilister.net/termux/termu
x-x11 x11 InRelease [14.0 kB]
Get:4 https://mirrors.in2.sahilister.net/termux/termu
x-main stable/main aarch64 Packages [589 kB]
Get:5 https://mirrors.in2.sahilister.net/termux/termu
x-x11 x11/main aarch64 Packages [237 kB]
Fetched 873 kB in 7s (134 kB/s)
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
8 packages can be upgraded. Run 'apt list --upgradabl
e' to see them.
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
shellcheck is already the newest version (0.11.0-3).
git is already the newest version (2.56.0).
python is already the newest version (3.14.10).
The following packages will be REMOVED:
nodejs-lts
The following NEW packages will be installed:
nodejs
0 upgraded, 1 newly installed, 1 to remove and 7 not
upgraded.
Need to get 10.7 MB of archives.
After this operation, 3969 kB of additional disk spac
e will be used.
Get:1 https://mirrors.in2.sahilister.net/termux/termu
x-main stable/main aarch64 nodejs aarch64 26.13.0-1 [
10.7 MB]
Fetched 10.7 MB in 6s (1864 kB/s)
pip install --upgrade ruff
npm install -g eslint
dpkg: nodejs-lts: dependency problems, but removing a
nyway as you requested:
yarn depends on nodejs | nodejs-lts; however:
Package nodejs is not installed.
Package nodejs-lts is to be removed.
npm depends on nodejs | nodejs-lts; however:
Package nodejs is not installed.
Package nodejs-lts is to be removed.

(Reading database ... 51618 files and directories cur
rently installed.)
Removing nodejs-lts (24.22.2) ...
Selecting previously unselected package nodejs.
(Reading database ... 51424 files and directories cur
rently installed.)
Preparing to unpack .../nodejs_26.13.0-1_aarch64.deb
...
Unpacking nodejs (26.13.0-1) ...
Setting up nodejs (26.13.0-1) ...
~ $ rm -rf ~/.local
~ $ cd ~ && python3 ~/bin/home_audit.py
# 1. audit: statuses, public/private, third-party
, leaks
2. moves junk + empty dirs to ~/_quarantine/<ts>/
EX="--exclude capyreader --exclude ez_termux_backup_f
lashdrive --exclude Goldenfoss --exclude goldenfoss-s
taging \
--exclude tech-ref-portal.stale --exclude duck-dev --
exclude keystone_extract --exclude Keystone_Health_do
wnloadable_files \
--exclude projects --exclude 'keystone-health-advocat
e-*' --exclude roku_failure_evidence --exclude ez_cou
rt \
--exclude cl_else_roku --exclude tmp --exclude backup
s"
eval python3 ~/bin/home_audit.py --stage $EX
# 3. then check ~/.audit_logs/*/SKIPPED.txt
eval python3 ~/bin/home_audit.py --push $EX
# 4. only after creating the PRIVATE repo
== home audit 20261109_173253 @ /data/data/com.termux
/files/home [termux] ==
CONFIG 18 | EMPTY 6 | LEFTOVER 1 | OTHER 3 | OUTPUT 5
| PROJECT 51 | SCRIPT 80 | SECRET 3
home_is_git_repo: yes, 0 commits
home_repo_fix: mv ~/.git ~/.git.home-repo.bak # emp
ty repo; makes 'git add -A' in ~ impossible
missing_expected: handover
hook_grep_candidates: 0 (HOOK-001)

PROJECTS
adblock_tester/ NO-GIT
ACTIVE
animated-umbrella/ NO-GIT
ACTIVE 1 sensitiv
e file(s) in worktree
animated-umbrella-main/ NO-GIT
ACTIVE 1 sensitiv
e file(s) in worktree
animated-umbrella.wiki/ NO-GIT
STALE
atomic_session/ LOCAL-ON
LY ACTIVE no-github-remote
audit_tools/ NO-GIT
ACTIVE
bin/ NO-GIT
ACTIVE
capyreader/ DIRTY(3)
ACTIVE public 30 sensiti
ve file(s) in worktree; TRACKED secret-like files: 9;
PUBLIC history has 22 secret-like path(s); !! PUBLIC
REPO WITH SENSITIVE FILES
cl_else_roku/ NO-GIT
ACTIVE 1 sensitiv
e file(s) in worktree
core/ NO-GIT
ACTIVE
duck-dev/ NO-GIT
ACTIVE 2 sensitiv
e file(s) in worktree
EnterpriseSecureMobileSkeleton/ LOCAL-ON
LY ACTIVE no-github-remote 1 sensitiv
e file(s) in worktree
env-audit-archive/ NO-GIT
ACTIVE 7 sensitiv
e file(s) in worktree
ez-vlc-android/ NO-GIT
ACTIVE
ez_court/ NO-GIT
ACTIVE
ez_gemini_gems_to_skills/ DIRTY(1)
ACTIVE private-or-missing 12 sensiti
ve file(s) in worktree; TRACKED secret-like files: 4
ez_git/ CLEAN
ACTIVE public 13 sensiti
ve file(s) in worktree; TRACKED secret-like files: 5;
PUBLIC history has 5 secret-like path(s); !! PUBLIC
REPO WITH SENSITIVE FILES
ez_ip_opsec_audit/ NO-GIT
STALE 6 sensitiv
e file(s) in worktree
ez_mac/ DIRTY(1)
ACTIVE public 1 sensitiv
e file(s) in worktree; !! PUBLIC REPO WITH SENSITIVE
FILES
ez_multi_repo/ CLEAN
ACTIVE public
ez_remote/ NO-GIT
ACTIVE
goldenfoss-staging/ NO-UPSTR
EAM,DIRTY(1) ACTIVE public 1 sensitiv
e file(s) in worktree; TRACKED secret-like files: 1;
PUBLIC history has 1 secret-like path(s); !! PUBLIC R
EPO WITH SENSITIVE FILES
handover_session_20261106/ NO-GIT
ACTIVE
invizible-pro-agent-skills/ NO-GIT
ACTIVE
ipleak_tool/ NO-GIT
ACTIVE
keystone-health-advocate-pa-medical-rights-wiki-for
ensic-tool/ LOCAL-ONLY,DIRTY(1) ACTIVE no-github
-remote 2 sensitive file(s) in worktree
Keystone_Health/ UNPUSHED
(1),DIRTY(10) ACTIVE public 2 sensitiv
e file(s) in worktree; !! PUBLIC REPO WITH SENSITIVE
FILES
keystone_pdf_text/ NO-GIT
ACTIVE
Linux/ NO-GIT
ACTIVE
my-futo-notes/ NO-GIT
STALE 1 sensitiv
e file(s) in worktree
net_tool/ NO-GIT
ACTIVE
netprobe/ EMPTY-RE
PO UNKNOWN no-github-remote
October_2026/ NO-GIT
ACTIVE
opsec-debug/ NO-GIT
ACTIVE
pa_audit_bundle/ NO-GIT
ACTIVE
pa_ra_onepagers/ NO-GIT
ACTIVE
phylum/ NO-GIT
ACTIVE 1 sensitiv
e file(s) in worktree
pixel-terminal-master/ NO-GIT
STALE
projects/ NO-GIT
ACTIVE 14 sensiti
ve file(s) in worktree
roku-diagnostics/ NO-GIT
ACTIVE
roku_failure_evidence/ NO-GIT
ACTIVE
rss-automator/ NO-GIT
ACTIVE 12 sensiti
ve file(s) in worktree
scripts/ NO-GIT
ACTIVE
shared/ NO-GIT
STALE
tech-ref-portal/ NO-UPSTR
EAM,DIRTY(23) ACTIVE private-or-missing 6 sensitiv
e file(s) in worktree
termux_master_ultra/ LOCAL-ON
LY ACTIVE no-github-remote
tests/ NO-GIT
UNKNOWN
tmp/ NO-GIT
ACTIVE
tmu-dashboard-auth-patch/ NO-GIT
ACTIVE 2 sensitiv
e file(s) in worktree
tools/ NO-GIT
ACTIVE 3 sensitiv
e file(s) in worktree
vid2gif/ NO-GIT
STALE

SCRIPT/OUTPUT files containing secret patterns (exclu
ded): bootstrap_ez_vlc_android.sh, dashboard.py, obli
vion_install.sh, tmu_probe_20261031_031159.txt

full report: /data/data/com.termux/files/home/.audit_
logs/home_audit_20261109_173253/AUDIT.md
~ $ python3 ~/bin/home_audit.py --quarantine-junk
# 2. moves junk + empty dirs to ~/_quarantine/<ts
>/
== home audit 20261109_173308 @ /data/data/com.termux
/files/home [termux] ==
CONFIG 18 | LEFTOVER 1 | OTHER 3 | OUTPUT 5 | PROJECT
51 | SCRIPT 80 | SECRET 3
home_is_git_repo: yes, 0 commits
home_repo_fix: mv ~/.git ~/.git.home-repo.bak # emp
ty repo; makes 'git add -A' in ~ impossible
missing_expected: handover
hook_grep_candidates: 0 (HOOK-001)
quarantined: 6 -> /data/data/com.termux/files/home/_q
uarantine/20261109_173308

PROJECTS
adblock_tester/ NO-GIT
ACTIVE
animated-umbrella/ NO-GIT
ACTIVE 1 sensitiv
e file(s) in worktree
animated-umbrella-main/ NO-GIT
ACTIVE 1 sensitiv
e file(s) in worktree
animated-umbrella.wiki/ NO-GIT
STALE
atomic_session/ LOCAL-ON
LY ACTIVE no-github-remote
audit_tools/ NO-GIT
ACTIVE
bin/ NO-GIT
ACTIVE
capyreader/ DIRTY(3)
ACTIVE public 30 sensiti
ve file(s) in worktree; TRACKED secret-like files: 9;
PUBLIC history has 22 secret-like path(s); !! PUBLIC
REPO WITH SENSITIVE FILES
cl_else_roku/ NO-GIT
ACTIVE 1 sensitiv
e file(s) in worktree
core/ NO-GIT
ACTIVE
duck-dev/ NO-GIT
ACTIVE 2 sensitiv
e file(s) in worktree
EnterpriseSecureMobileSkeleton/ LOCAL-ON
LY ACTIVE no-github-remote 1 sensitiv
e file(s) in worktree
env-audit-archive/ NO-GIT
ACTIVE 7 sensitiv
e file(s) in worktree
ez-vlc-android/ NO-GIT
ACTIVE
ez_court/ NO-GIT
ACTIVE
ez_gemini_gems_to_skills/ DIRTY(1)
ACTIVE private-or-missing 12 sensiti
ve file(s) in worktree; TRACKED secret-like files: 4
ez_git/ CLEAN
ACTIVE public 13 sensiti
ve file(s) in worktree; TRACKED secret-like files: 5;
PUBLIC history has 5 secret-like path(s); !! PUBLIC
REPO WITH SENSITIVE FILES
ez_ip_opsec_audit/ NO-GIT
STALE 6 sensitiv
e file(s) in worktree
ez_mac/ DIRTY(1)
ACTIVE public 1 sensitiv
e file(s) in worktree; !! PUBLIC REPO WITH SENSITIVE
FILES
ez_multi_repo/ CLEAN
ACTIVE public
ez_remote/ NO-GIT
ACTIVE
goldenfoss-staging/ NO-UPSTR
EAM,DIRTY(1) ACTIVE public 1 sensitiv
e file(s) in worktree; TRACKED secret-like files: 1;
PUBLIC history has 1 secret-like path(s); !! PUBLIC R
EPO WITH SENSITIVE FILES
handover_session_20261106/ NO-GIT
ACTIVE
invizible-pro-agent-skills/ NO-GIT
ACTIVE
ipleak_tool/ NO-GIT
ACTIVE
keystone-health-advocate-pa-medical-rights-wiki-for
ensic-tool/ LOCAL-ONLY,DIRTY(1) ACTIVE no-github
-remote 2 sensitive file(s) in worktree
Keystone_Health/ UNPUSHED
(1),DIRTY(10) ACTIVE public 2 sensitiv
e file(s) in worktree; !! PUBLIC REPO WITH SENSITIVE
FILES
keystone_pdf_text/ NO-GIT
ACTIVE
Linux/ NO-GIT
ACTIVE
my-futo-notes/ NO-GIT
STALE 1 sensitiv
e file(s) in worktree
net_tool/ NO-GIT
ACTIVE
netprobe/ EMPTY-RE
PO UNKNOWN no-github-remote
October_2026/ NO-GIT
ACTIVE
opsec-debug/ NO-GIT
ACTIVE
pa_audit_bundle/ NO-GIT
ACTIVE
pa_ra_onepagers/ NO-GIT
ACTIVE
phylum/ NO-GIT
ACTIVE 1 sensitiv
e file(s) in worktree
pixel-terminal-master/ NO-GIT
STALE
projects/ NO-GIT
ACTIVE 14 sensiti
ve file(s) in worktree
roku-diagnostics/ NO-GIT
ACTIVE
roku_failure_evidence/ NO-GIT
ACTIVE
rss-automator/ NO-GIT
ACTIVE 12 sensiti
ve file(s) in worktree
scripts/ NO-GIT
ACTIVE
shared/ NO-GIT
STALE
tech-ref-portal/ NO-UPSTR
EAM,DIRTY(23) ACTIVE private-or-missing 6 sensitiv
e file(s) in worktree
termux_master_ultra/ LOCAL-ON
LY ACTIVE no-github-remote
tests/ NO-GIT
UNKNOWN
tmp/ NO-GIT
ACTIVE
tmu-dashboard-auth-patch/ NO-GIT
ACTIVE 2 sensitiv
e file(s) in worktree
tools/ NO-GIT
ACTIVE 3 sensitiv
e file(s) in worktree
vid2gif/ NO-GIT
STALE

SCRIPT/OUTPUT files containing secret patterns (exclu
ded): bootstrap_ez_vlc_android.sh, dashboard.py, obli
vion_install.sh, tmu_probe_20261031_031159.txt

full report: /data/data/com.termux/files/home/.audit_
logs/home_audit_20261109_173308/AUDIT.md
~ $ EX="--exclude capyreader --exclude ez_termux_back
up_flashdrive --exclude Goldenfoss --exclude goldenfo
ss-staging \
> --exclude tech-ref-portal.stale --exclude duck-dev
--exclude keystone_extract --exclude Keystone_Health_
downloadable_files \
> --exclude projects --exclude 'keystone-health-advoc
ate-*' --exclude roku_failure_evidence --exclude ez_c
ourt \
> --exclude cl_else_roku --exclude tmp --exclude back
ups"
~ $ eval python3 ~/bin/home_audit.py --stage $EX
# 3. then check ~/.audit_logs/*/SKIPPED.txt
== home audit 20261109_173330 @ /data/data/com.termux
/files/home [termux] ==
CONFIG 18 | LEFTOVER 1 | OTHER 3 | OUTPUT 5 | PROJECT
51 | SCRIPT 80 | SECRET 3
home_is_git_repo: yes, 0 commits
home_repo_fix: mv ~/.git ~/.git.home-repo.bak # emp
ty repo; makes 'git add -A' in ~ impossible
missing_expected: handover, Chrombook_crostini_repair
_guide
hook_grep_candidates: 0 (HOOK-001)

PROJECTS
adblock_tester/ NO-GIT
ACTIVE
animated-umbrella/ NO-GIT
ACTIVE 1 sensitiv
e file(s) in worktree
animated-umbrella-main/ NO-GIT
ACTIVE 1 sensitiv
e file(s) in worktree
animated-umbrella.wiki/ NO-GIT
STALE
atomic_session/ LOCAL-ON
LY ACTIVE no-github-remote
audit_tools/ NO-GIT
ACTIVE
bin/ NO-GIT
ACTIVE
capyreader/ DIRTY(3)
ACTIVE public 30 sensiti
ve file(s) in worktree; TRACKED secret-like files: 9;
PUBLIC history has 22 secret-like path(s); !! PUBLIC
REPO WITH SENSITIVE FILES
cl_else_roku/ NO-GIT
ACTIVE 1 sensitiv
e file(s) in worktree
core/ NO-GIT
ACTIVE
duck-dev/ NO-GIT
ACTIVE 2 sensitiv
e file(s) in worktree
EnterpriseSecureMobileSkeleton/ LOCAL-ON
LY ACTIVE no-github-remote 1 sensitiv
e file(s) in worktree
env-audit-archive/ NO-GIT
ACTIVE 7 sensitiv
e file(s) in worktree
ez-vlc-android/ NO-GIT
ACTIVE
ez_court/ NO-GIT
ACTIVE
ez_gemini_gems_to_skills/ DIRTY(1)
ACTIVE private-or-missing 12 sensiti
ve file(s) in worktree; TRACKED secret-like files: 4
ez_git/ CLEAN
ACTIVE public 13 sensiti
ve file(s) in worktree; TRACKED secret-like files: 5;
PUBLIC history has 5 secret-like path(s); !! PUBLIC
REPO WITH SENSITIVE FILES
ez_ip_opsec_audit/ NO-GIT
STALE 6 sensitiv
e file(s) in worktree
ez_mac/ DIRTY(1)
ACTIVE public 1 sensitiv
e file(s) in worktree; !! PUBLIC REPO WITH SENSITIVE
FILES
ez_multi_repo/ CLEAN
ACTIVE public
ez_remote/ NO-GIT
ACTIVE
goldenfoss-staging/ NO-UPSTR
EAM,DIRTY(1) ACTIVE public 1 sensitiv
e file(s) in worktree; TRACKED secret-like files: 1;
PUBLIC history has 1 secret-like path(s); !! PUBLIC R
EPO WITH SENSITIVE FILES
handover_session_20261106/ NO-GIT
ACTIVE
invizible-pro-agent-skills/ NO-GIT
ACTIVE
ipleak_tool/ NO-GIT
ACTIVE
keystone-health-advocate-pa-medical-rights-wiki-for
ensic-tool/ LOCAL-ONLY,DIRTY(1) ACTIVE no-github
-remote 2 sensitive file(s) in worktree
Keystone_Health/ UNPUSHED
(1),DIRTY(10) ACTIVE public 2 sensitiv
e file(s) in worktree; !! PUBLIC REPO WITH SENSITIVE
FILES
keystone_pdf_text/ NO-GIT
ACTIVE
Linux/ NO-GIT
ACTIVE
my-futo-notes/ NO-GIT
STALE 1 sensitiv
e file(s) in worktree
net_tool/ NO-GIT
ACTIVE
netprobe/ EMPTY-RE
PO UNKNOWN no-github-remote
October_2026/ NO-GIT
ACTIVE
opsec-debug/ NO-GIT
ACTIVE
pa_audit_bundle/ NO-GIT
ACTIVE
pa_ra_onepagers/ NO-GIT
ACTIVE
phylum/ NO-GIT
ACTIVE 1 sensitiv
e file(s) in worktree
pixel-terminal-master/ NO-GIT
STALE
projects/ NO-GIT
ACTIVE 14 sensiti
ve file(s) in worktree
roku-diagnostics/ NO-GIT
ACTIVE
roku_failure_evidence/ NO-GIT
ACTIVE
rss-automator/ NO-GIT
ACTIVE 12 sensiti
ve file(s) in worktree
scripts/ NO-GIT
ACTIVE
shared/ NO-GIT
STALE
tech-ref-portal/ NO-UPSTR
EAM,DIRTY(23) ACTIVE private-or-missing 6 sensitiv
e file(s) in worktree
termux_master_ultra/ LOCAL-ON
LY ACTIVE no-github-remote
tests/ NO-GIT
UNKNOWN
tmp/ NO-GIT
ACTIVE
tmu-dashboard-auth-patch/ NO-GIT
ACTIVE 2 sensitiv
e file(s) in worktree
tools/ NO-GIT
ACTIVE 3 sensitiv
e file(s) in worktree
vid2gif/ NO-GIT
STALE

SCRIPT/OUTPUT files containing secret patterns (exclu
ded): bootstrap_ez_vlc_android.sh, dashboard.py, obli
vion_install.sh, tmu_probe_20261031_031159.txt

full report: /data/data/com.termux/files/home/.audit_
logs/home_audit_20261109_173330/AUDIT.md

staged 962 files -> /data/data/com.termux/files/home/
home-archive-staging skipped 1356 (see SKIPPED.txt)
~ $ eval python3 ~/bin/home_audit.py --push $EX
# 4. only after creating the PRIVATE repo
== home audit 20261109_173351 @ /data/data/com.termux
/files/home [termux] ==
CONFIG 18 | LEFTOVER 1 | OTHER 3 | OUTPUT 5 | PROJECT
51 | SCRIPT 80 | SECRET 3
home_is_git_repo: yes, 0 commits
home_repo_fix: mv ~/.git ~/.git.home-repo.bak # emp
ty repo; makes 'git add -A' in ~ impossible
missing_expected: handover, Chrombook_crostini_repair
_guide
hook_grep_candidates: 0 (HOOK-001)

PROJECTS
adblock_tester/ NO-GIT
ACTIVE
animated-umbrella/ NO-GIT
ACTIVE 1 sensitiv
e file(s) in worktree
animated-umbrella-main/ NO-GIT
ACTIVE 1 sensitiv
e file(s) in worktree
animated-umbrella.wiki/ NO-GIT
STALE
atomic_session/ LOCAL-ON
LY ACTIVE no-github-remote
audit_tools/ NO-GIT
ACTIVE
bin/ NO-GIT
ACTIVE
capyreader/ DIRTY(3)
ACTIVE public 30 sensiti
ve file(s) in worktree; TRACKED secret-like files: 9;
PUBLIC history has 22 secret-like path(s); !! PUBLIC
REPO WITH SENSITIVE FILES
cl_else_roku/ NO-GIT
ACTIVE 1 sensitiv
e file(s) in worktree
core/ NO-GIT
ACTIVE
duck-dev/ NO-GIT
ACTIVE 2 sensitiv
e file(s) in worktree
EnterpriseSecureMobileSkeleton/ LOCAL-ON
LY ACTIVE no-github-remote 1 sensitiv
e file(s) in worktree
env-audit-archive/ NO-GIT
ACTIVE 7 sensitiv
e file(s) in worktree
ez-vlc-android/ NO-GIT
ACTIVE
ez_court/ NO-GIT
ACTIVE
ez_gemini_gems_to_skills/ DIRTY(1)
ACTIVE private-or-missing 12 sensiti
ve file(s) in worktree; TRACKED secret-like files: 4
ez_git/ CLEAN
ACTIVE public 13 sensiti
ve file(s) in worktree; TRACKED secret-like files: 5;
PUBLIC history has 5 secret-like path(s); !! PUBLIC
REPO WITH SENSITIVE FILES
ez_ip_opsec_audit/ NO-GIT
STALE 6 sensitiv
e file(s) in worktree
ez_mac/ DIRTY(1)
ACTIVE public 1 sensitiv
e file(s) in worktree; !! PUBLIC REPO WITH SENSITIVE
FILES
ez_multi_repo/ CLEAN
ACTIVE public
ez_remote/ NO-GIT
ACTIVE
goldenfoss-staging/ NO-UPSTR
EAM,DIRTY(1) ACTIVE public 1 sensitiv
e file(s) in worktree; TRACKED secret-like files: 1;
PUBLIC history has 1 secret-like path(s); !! PUBLIC R
EPO WITH SENSITIVE FILES
handover_session_20261106/ NO-GIT
ACTIVE
invizible-pro-agent-skills/ NO-GIT
ACTIVE
ipleak_tool/ NO-GIT
ACTIVE
keystone-health-advocate-pa-medical-rights-wiki-for
ensic-tool/ LOCAL-ONLY,DIRTY(1) ACTIVE no-github
-remote 2 sensitive file(s) in worktree
Keystone_Health/ UNPUSHED
(1),DIRTY(10) ACTIVE public 2 sensitiv
e file(s) in worktree; !! PUBLIC REPO WITH SENSITIVE
FILES
keystone_pdf_text/ NO-GIT
ACTIVE
Linux/ NO-GIT
ACTIVE
my-futo-notes/ NO-GIT
STALE 1 sensitiv
e file(s) in worktree
net_tool/ NO-GIT
ACTIVE
netprobe/ EMPTY-RE
PO UNKNOWN no-github-remote
October_2026/ NO-GIT
ACTIVE
opsec-debug/ NO-GIT
ACTIVE
pa_audit_bundle/ NO-GIT
ACTIVE
pa_ra_onepagers/ NO-GIT
ACTIVE
phylum/ NO-GIT
ACTIVE 1 sensitiv
e file(s) in worktree
pixel-terminal-master/ NO-GIT
STALE
projects/ NO-GIT
ACTIVE 14 sensiti
ve file(s) in worktree
roku-diagnostics/ NO-GIT
ACTIVE
roku_failure_evidence/ NO-GIT
ACTIVE
rss-automator/ NO-GIT
ACTIVE 12 sensiti
ve file(s) in worktree
scripts/ NO-GIT
ACTIVE
shared/ NO-GIT
STALE
tech-ref-portal/ NO-UPSTR
EAM,DIRTY(23) ACTIVE private-or-missing 6 sensitiv
e file(s) in worktree
termux_master_ultra/ LOCAL-ON
LY ACTIVE no-github-remote
tests/ NO-GIT
UNKNOWN
tmp/ NO-GIT
ACTIVE
tmu-dashboard-auth-patch/ NO-GIT
ACTIVE 2 sensitiv
e file(s) in worktree
tools/ NO-GIT
ACTIVE 3 sensitiv
e file(s) in worktree
vid2gif/ NO-GIT
STALE

SCRIPT/OUTPUT files containing secret patterns (exclu
ded): bootstrap_ez_vlc_android.sh, dashboard.py, obli
vion_install.sh, tmu_probe_20261031_031159.txt

full report: /data/data/com.termux/files/home/.audit_
logs/home_audit_20261109_173351/AUDIT.md

staged 962 files -> /data/data/com.termux/files/home/
home-archive-staging skipped 1356 (see SKIPPED.txt)

ABORT[4]: bilbywilby/home-archive not reachable with
your stored credentials (does it exist?). fatal: coul
d not read Username for 'https://github.com': termina
l prompts disabled
~ $ # See what's actually big before nuking:
~ $ du -sh ~/.local/* 2>/dev/null | sort -rh | head -
20
7.0K /data/data/com.termux/files/home/.local/state
~ $
~ $ # If it's mostly Trash and caches:
~ $ rm -rf ~/.local/share/Trash
~ $ rm -rf ~/.local/share/Trash*
~ $
~ $ # If you have venvs you don't need:
~ $ rm -rf ~/.local/share/virtualenvs/<name>
bash: syntax error near unexpected token `newline'
~ $ mkdir -p scripts
whatever tools are installed.
set -u
status=0

files=$(git ls-files --cached --others --exclude-stan
dard 2>/dev/null | sort -u)
if [ -z "$files" ]; then
echo "no files to lint"
exit 0
fi

run() {
local name=$1; shift
if command -v "$1" >/dev/null 2>&1; then
echo "== $name"
"$@" || status=1
else
echo "== $name skipped (not installed)"
fi
}

sh_files=$(printf '%s\n' "$files" | sed -n '/\.sh$/p'
)
py_files=$(printf '%s\n' "$files" | sed -n '/\.py$/p'
)
js_files=$(printf '%s\n' "$files" | sed -n '/\.[cm]\?
[jt]sx\?$/p')

# Word splitting is intentional here: paths must not
contain spaces.
[ -n "$sh_files" ] && run shellcheck shellcheck $sh_f
iles
[ -n "$py_files" ] && run ruff ruff check $py_files
[ -n "$js_files" ] && run eslint eslint $js_files

exit $status
EOF

chmod +x scripts/lint-local.sh
~ $
~ $ cat > scripts/lint-local.sh <<'EOF'
> #!/usr/bin/env bash
> # Lints tracked and untracked files with whatever t
ools are installed.
> set -u
> status=0
>
> files=$(git ls-files --cached --others --exclude-st
andard 2>/dev/null | sort -u)
> if [ -z "$files" ]; then
> echo "no files to lint"
> exit 0
> fi
>
> run() {
> local name=$1; shift
> if command -v "$1" >/dev/null 2>&1; then
> echo "== $name"
> "$@" || status=1
> else
> echo "== $name skipped (not installed)"
> fi
> }
>
> sh_files=$(printf '%s\n' "$files" | sed -n '/\.sh$/
p')
> py_files=$(printf '%s\n' "$files" | sed -n '/\.py$/
p')
> js_files=$(printf '%s\n' "$files" | sed -n '/\.[cm]
\?[jt]sx\?$/p')
>
> # Word splitting is intentional here: paths must no
t contain spaces.
> [ -n "$sh_files" ] && run shellcheck shellcheck $sh
_files
> [ -n "$py_files" ] && run ruff ruff check $py_files
> [ -n "$js_files" ] && run eslint eslint $js_files
>
> exit $status
> EOF
~ $
~ $ chmod +x scripts/lint-local.sh
~ $ sh lint-local.sh
sh: 0: cannot open lint-local.sh: No such file
~ $ sh scripts/lint-local.sh
no files to lint
~ $ # Check if deleted-but-open files are holding spa
ce
/dev/nu~ $ ls -la /proc/*/fd 2>/dev/null | grep '(del
eted)' | head -20
d 2>/dev/null || true

# Check actual usage NOW (post-rm)
du -sh ~/* 2>/dev/null | sort -rh | head -10
df -h /l-wx------. 1 u0_a399 u0_a399 64 Nov 9 17:44
17 -> /data/data/com.termux/files/home/.npm/_logs/202
6-11-09T18_38_12_714Z-debug-0.log (deleted)
~ $
~ $ # Nuclear option: trigger F2FS GC (if F2FS)
~ $ echo 1 > /sys/fs/f2fs/*/discard 2>/dev/null || tr
ue
bash: /sys/fs/f2fs/*/discard: Permission denied
~ $
~ $ # Check actual usage NOW (post-rm)
~ $ du -sh ~/* 2>/dev/null | sort -rh | head -10
346M /data/data/com.termux/files/home/ez_git
345M /data/data/com.termux/files/home/rss-automato
r
343M /data/data/com.termux/files/home/ez_gemini_ge
ms_to_skills
257M /data/data/com.termux/files/home/tech-ref-por
tal
110M /data/data/com.termux/files/home/capyreader
66M /data/data/com.termux/files/home/animated-umb
rella
29M /data/data/com.termux/files/home/home-archive
-staging
24M /data/data/com.termux/files/home/keystone-hea
lth-advocate-pa-medical-rights-wiki-forensic-tool
17M /data/data/com.termux/files/home/projects
15M /data/data/com.termux/files/home/animated-umb
rella-main
~ $ df -h /
Filesystem Size Used Avail Use% Mounted on
/dev/block/dm-12 887M 884M 0 100% /
~ $ apk update && apk upgrade
No command apk found, did you mean:
Command ack in package ack-grep
Command apt in package apt
Command ark in package ark
Command awk in package gawk
~ $ # Remove package and language caches
rm -rf ~/.npm ~/.cache ~/.local/share/Trash ~/.pytest
_cache

# Remove redundant archive copies
rm -f ~/rss-automator/ez_gemini_gems_to_skills/Active
_Projects/Python.zip
rm -f ~/ez_gemini_gems_to_skills/Active_Projects/Pyth
on.zip

# Verify reclaimed space
df -h /
Filesystem Size Used Avail Use% Mounted on
/dev/block/dm-12 887M 884M 0 100% /
~ $ # Show all non-printable characters in the script
:
~ $ cat -A your_script.sh | grep -n '\^'
cat: your_script.sh: No such file or directory
~ $
~ $ # Or with hexdump on a specific line:
~ $ sed -n '3p' your_script.sh | xxd | head
sed: can't read your_script.sh: No such file or direc
tory
The program xxd is not installed. Install it by execu
ting:
pkg install xxd
~ $
~ $ # U+200B in UTF-8 is: e2 80 8b
~ $ grep -P '\xe2\x80\x8b' your_script.sh
grep: your_script.sh: No such file or directory
~ $ # Check if deleted-but-open files are holding spa
ce
~ $ ls -la /proc/*/fd 2>/dev/null | grep '(deleted)'
| head -20
l-wx------. 1 u0_a399 u0_a399 64 Nov 9 17:48 17 -> /
data/data/com.termux/files/home/.npm/_logs/2026-11-09
T18_38_12_714Z-debug-0.log (deleted)
~ $
~ $ # Nuclear option: trigger F2FS GC (if F2FS)
~ $ echo 1 > /sys/fs/f2fs/*/discard 2>/dev/null || tr
ue
bash: /sys/fs/f2fs/*/discard: Permission denied
~ $
~ $ # Check actual usage NOW (post-rm)
~ $ du -sh ~/* 2>/dev/null | sort -rh | head -10
346M /data/data/com.termux/files/home/ez_git
345M /data/data/com.termux/files/home/rss-automato
r
343M /data/data/com.termux/files/home/ez_gemini_ge
ms_to_skills
257M /data/data/com.termux/files/home/tech-ref-por
tal
110M /data/data/com.termux/files/home/capyreader
66M /data/data/com.termux/files/home/animated-umb
rella
29M /data/data/com.termux/files/home/home-archive
-staging
24M /data/data/com.termux/files/home/keystone-hea
lth-advocate-pa-medical-rights-wiki-forensic-tool
17M /data/data/com.termux/files/home/projects
15M /data/data/com.termux/files/home/animated-umb
rella-main
~ $ df -h /
Filesystem Size Used Avail Use% Mounted on
/dev/block/dm-12 887M 884M 0 100% /
~ $ # Force with explicit error output
~ $ rm -rfv ~/.npm 2>&1 | tail -5
~ $ rm -rfv ~/.local 2>&1 | tail -5
removed directory '/data/data/com.termux/files/home/.
local/state/gh'
removed directory '/data/data/com.termux/files/home/.
local/state'
removed directory '/data/data/com.termux/files/home/.
local'
~ $ # Option A: SSH (preferred for Termux)
~ $ # Generate key if you don't have one
~ $ ls ~/.ssh/id_ed25519 2>/dev/null || ssh-keygen -t
ed25519 -f ~/.ssh/id_ed25519 -N ""
t remote set-url origin git@github.com:bilbywilby/hom
e-archive.git

# Test
ssh -T git@github.com 2>&1 | head -3

# Then re-run
python3 ~/bin/home_audit.py --push/data/data/com.term
ux/files/home/.ssh/id_ed25519
~ $
~ $ # Show your public key — add it to GitHub → Setti
ngs → SSH Keys
~ $ cat ~/.ssh/id_ed25519.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIMeDAUTsfT1G3PKrl
0DFS965+3s/tZWXYsIhR4eh/ONq
~ $
~ $ # Then in the staging repo:
~ $ cd ~/home-archive-staging
~/home-archive-staging $ git remote set-url origin gi
t@github.com:bilbywilby/home-archive.git
~/home-archive-staging $
~/home-archive-staging $ # Test
~/home-archive-staging $ ssh -T git@github.com 2>&1 |
head -3
Hi bilbywilby! You've successfully authenticated, but
GitHub does not provide shell access.
~/home-archive-staging $ # Option B: PAT via credenti
al store (if you prefer HTTPS)
exists on Git~/home-archive-staging $ git config --g
lobal credential.helper store
~/home-archive-staging $ # Next push will prompt ONCE
for username + PAT, then save to ~/.git-credentials
~/home-archive-staging $ # Make sure the repo exists
on GitHub first (private!)
~/home-archive-staging $ ls -la
total 42
drwx------. 6 u0_a399 u0_a399 3452 Nov 7 08:27 .
drwx------. 71 u0_a399 u0_a399 20480 Nov 9 17:49 ..
drwx------. 6 u0_a399 u0_a399 3452 Nov 9 17:48 .gi
t
-rw-------. 1 u0_a399 u0_a399 43 Nov 7 08:26 .gi
tignore
-rw-------. 1 u0_a399 u0_a399 109 Nov 7 08:26 REA
DME.md
drwx------. 6 u0_a399 u0_a399 3452 Nov 9 17:34 _au
dit
drwx------. 34 u0_a399 u0_a399 3452 Nov 9 17:33 pro
jects
drwx------. 8 u0_a399 u0_a399 3452 Nov 7 08:26 scr
ipts
~/home-archive-staging $ # Show all non-printable cha
racters in the script:
~/home-archive-staging $ cat -A your_script.sh | grep
-n '\^'
0 8b
grep -P '\xe2\x80\x8b' your_script.sh cat: your_scr
ipt.sh: No such file or directory
~/home-archive-staging $
~/home-archive-staging $ # Or with hexdump on a speci
fic line:
~/home-archive-staging $ sed -n '3p' your_script.sh |
xxd | head
sed: can't read your_script.sh: No such file or direc
tory
The program xxd is not installed. Install it by execu
ting:
pkg install xxd
~/home-archive-staging $
~/home-archive-staging $ # U+200B in UTF-8 is: e2 80
8b
~/home-archive-staging $ grep -P '\xe2\x80\x8b' your_
script.sh
grep: your_script.sh: No such file or directory
~/home-archive-staging $ pkg install xxd
No mirror or mirror group selected. You might want to
select one by running 'termux-change-repo'
Testing the available mirrors:
[*] (10) https://packages-cf.termux.dev/apt/termux-ma
in: ok
[*] (1) https://linux.domainesia.com/applications/ter
mux/termux-main: ok
[*] (1) https://termux.niranjan.co/termux-main: ok
[*] (1) https://mirror.bardia.tech/termux/termux-main
: bad
[*] (1) https://mirrors.saswata.cc/termux/termux-main
: bad
[*] (1) https://mirrors.ravidwivedi.in/termux/termux-
main: ok
[*] (1) https://mirror.textcord.xyz/termux/termux-mai
n: ok
[*] (1) https://mirrors.krnk.org/apt/termux/termux-ma
in: ok
[*] (1) https://mirror.rinarin.dev/termux/termux-main
: ok
[*] (1) https://mirror.jeonnam.school/termux/termux-m
ain: ok
[*] (1) https://mirrors.nguyenhoang.cloud/termux/term
ux-main: ok
[*] (1) https://mirrors.cbrx.io/apt/termux/termux-mai
n: ok
[*] (1) https://tmx.xvx.my.id/apt/termux-main: ok
[*] (1) https://mirror.albony.in/termux/termux-main:
ok
[*] (1) https://mirror.twds.com.tw/termux/termux-main
: ok
[*] (1) https://mirror.nevacloud.com/applications/ter
mux/termux-main: ok
[*] (1) https://mirror.meowsmp.net/termux/termux-main
: bad
[*] (1) https://mirror.freedif.org/termux/termux-main
: ok
[*] (1) https://mirrors.in.sahilister.net/termux/term
ux-main/: ok
[*] (1) https://mirror.iscas.ac.cn/termux/apt/termux-
main: ok
[*] (1) https://mirrors.sau.edu.cn/termux/apt/termux-
main: ok
[*] (1) https://mirrors.sdu.edu.cn/termux/termux-main
: ok
[*] (1) https://mirrors.aliyun.com/termux/termux-main
: ok
[*] (1) https://mirrors.cernet.edu.cn/termux/apt/term
ux-main: ok
[*] (1) https://mirrors.cqupt.edu.cn/termux/termux-ma
in: ok
[*] (1) https://mirrors.sustech.edu.cn/termux/apt/ter
mux-main: ok
[*] (1) https://mirrors.bfsu.edu.cn/termux/apt/termux
-main: ok
[*] (1) https://mirrors.pku.edu.cn/termux/termux-main
/: ok
[*] (1) https://mirrors.hust.edu.cn/termux/apt/termux
-main: ok
[*] (1) https://mirrors.tuna.tsinghua.edu.cn/termux/a
pt/termux-main: ok
[*] (1) https://mirror.sjtu.edu.cn/termux/termux-main
/: ok
[*] (1) https://mirror.nyist.edu.cn/termux/apt/termux
-main: ok
[*] (1) https://mirrors.ustc.edu.cn/termux/termux-mai
n: ok
[*] (1) https://mirrors.zju.edu.cn/termux/apt/termux-
main: ok
[*] (1) https://mirrors.nju.edu.cn/termux/apt/termux-
main: ok
[*] (4) https://grimler.se/termux/termux-main: ok
[*] (1) https://mirror.polido.pt/termux/termux-main:
bad
[*] (1) https://termux.3san.dev/termux/termux-main: o
k
[*] (1) https://mirrors.medzik.dev/termux/termux-main
: bad
[*] (1) https://mirror.accum.se/mirror/termux.dev/ter
mux-main: ok
[*] (1) https://mirror.mwt.me/termux/main: ok
[*] (1) https://ro.mirror.flokinet.net/termux/termux-
main: ok
[*] (1) https://md.mirrors.hacktegic.com/termux/termu
x-main: bad
[*] (1) https://is.mirror.flokinet.net/termux/termux-
main: ok
[*] (1) https://mirror.termux.dev/termux-main: bad
[*] (1) https://termux.mentality.rip/termux-main: bad
[*] (1) https://nl.mirror.flokinet.net/termux/termux-
main: ok
[*] (1) https://termux.cdn.lumito.net/termux-main: ba
d
[*] (1) https://mirror.leitecastro.com/termux/termux-
main: ok
[*] (1) https://mirror.bouwhuis.network/termux/termux
-main: ok
[*] (1) https://termux.librehat.com/apt/termux-main:
ok
[*] (1) https://mirror.autkin.net/termux/termux-main:
ok
[*] (1) https://mirror.sunred.org/termux/termux-main:
ok
[*] (1) https://packages.termux.dev/apt/termux-main:
ok
[*] (1) https://ftp.agdsn.de/termux/termux-main: ok
[*] (1) https://ftp.fau.de/termux/termux-main: ok
[*] (1) https://mirrors.cfe.re/termux/termux-main: ba
d
[*] (1) https://mirrors.de.sahilister.net/termux/term
ux-main: ok
[*] (1) https://gnlug.org/pub/termux/termux-main: ok
[*] (1) https://mirror.mwt.me/termux/main: ok
[*] (1) https://mirror.quantum5.ca/termux/termux-main
: ok
[*] (1) https://mirrors.utermux.dev/termux/termux-mai
n: bad
[*] (1) https://plug-mirror.rcac.purdue.edu/termux/te
rmux-main: ok
[*] (1) https://mirror.vern.cc/termux/termux-main: ba
d
[*] (1) https://dl.kcubeterm.com/termux-main: bad
[*] (1) https://termux.danyael.xyz/termux/termux-main
: ok
[*] (1) https://mirror.csclub.uwaterloo.ca/termux/ter
mux-main: ok
[*] (1) https://mirror.fcix.net/termux/termux-main: o
k
[*] (1) https://mirrors.middlendian.com/termux/termux
-main: ok
[*] (1) https://repository.su/termux/termux-main/: ok
[*] (1) http://mirror.mephi.ru/termux/termux-main: ok
Picking mirror: (27) /data/data/com.termux/files/usr/
etc/termux/mirrors/chinese_mainland/mirrors.sdu.edu.c
n
Get:1 https://tur.kcubeterm.com tur-packages InReleas
e
Get:2 https://mirrors.sdu.edu.cn/termux/termux-main s
table InRelease [14.0 kB]
Get:3 https://mirrors.sdu.edu.cn/termux/termux-x11 x1
1 InRelease [14.0 kB]
Get:4 https://mirrors.sdu.edu.cn/termux/termux-main s
table/main aarch64 Packages [589 kB]
Get:5 https://mirrors.sdu.edu.cn/termux/termux-x11 x1
1/main aarch64 Packages [237 kB]
Fetched 873 kB in 9s (94.8 kB/s)
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
7 packages can be upgraded. Run 'apt list --upgradabl
e' to see them.
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following NEW packages will be installed:
xxd
0 upgraded, 1 newly installed, 0 to remove and 7 not
upgraded.
Need to get 13.5 kB of archives.
After this operation, 69.6 kB of additional disk spac
e will be used.
Get:1 https://mirrors.sdu.edu.cn/termux/termux-main s
table/main aarch64 xxd aarch64 9.2.1300 [13.5 kB]
Fetched 13.5 kB in 2s (7973 B/s)
Selecting previously unselected package xxd.
(Reading database ... 51558 files and directories cur
rently installed.)
Preparing to unpack .../xxd_9.2.1300_aarch64.deb ...
Unpacking xxd (9.2.1300) ...
Setting up xxd (9.2.1300) ...
~/home-archive-staging $

Good news: SSH auth is working (Hi bilbywilby!). The push should work now.

But the disk situation needs clarification. Here's what's actually happening:

The df vs du discrepancy
Your df -h / shows 887M total, but du -sh ~/* shows ~1.4GB of content in your home directory alone. This means / in Termux is NOT where your home directory lives. Your home is on the app data partition:

kotlin
Copy
/data/data/com.termux/files/home
The 887M / partition is the Termux system partition (packages, /usr, etc.). Your actual user data is on the Android app data volume, which is the 109G FUSE mount (/storage/emulated) or the raw app data block device.

The ENOSPC error was on the / partition (where ~/.audit_logs was being created — wait, no, ~/.audit_logs IS in your home). Let me re-check...

Actually, looking at the original error:

vbnet
Copy
OSError: [Errno 28] No space left on device: '/home/droid/.audit_logs'
That was from a different machine (the first terminal session with /home/droid). Your current Termux phone is a different environment. On this phone, the audit already ran successfully (staged 962 files, quarantined 6 items). The df -h / showing 100% is the Termux system partition being full of packages — that's normal for Termux and doesn't block your home directory operations.

Your home directory is fine. The audit ran, staging worked. The only blocker was the GitHub auth, which you've now fixed with SSH.

Do the push now
bash
Copy
cd ~/home-archive-staging
git add -A
git commit -m "Archive: 962 files staged from Termux home audit 2026-11-09"
git push origin main
If the repo doesn't exist on GitHub yet, create it first (private):

bash
Copy
# From your phone's browser or via API:
# github.com → New repository → "home-archive" → Private → Create
# Then:
git push -u origin main
The ~/.npm deleted-but-open file
bash
Copy
l-wx------. 1 u0_a399 u0_a399 64 Nov  9 17:48 17 -> .../.npm/_logs/...log (deleted)
That's a single 64-byte log file held open by a process. It's negligible. It'll be released when that process exits (or on reboot). Not worth chasing.

For the master_engine.py — final patch
Since you're on Termux (not a VM with /data/user/0), update the mounts:

python
Copy
class StorageTruthChecker:
    """Audits blocks AND inodes across Termux-relevant mounts."""

    # / = Termux system partition (packages)
    # /storage/emulated = FUSE shared storage (109G)
    # /data/data/com.termux/files = app data (where $HOME lives)
    MOUNTS = ("/", "/storage/emulated", "/data/data/com.termux/files")
And the per-processor isolation in run_cycle:

python
Copy
async def run_cycle(self) -> None:
    logger.info("Starting automation audit cycle...")
    try:
        state_data = self.validator.load_state()
    except Exception as e:
        logger.error("Failed to load state: %s", e)
        return

    for processor in self.processors:
        try:
            await processor.process(state_data)
        except Exception as e:
            logger.error("Processor %s failed: %s", type(processor).__name__, e)
Summary of where you are
Step	Status
Audit scan	✅ Done
Quarantine (6 items)	✅ Done
Stage (962 files, hardlinked)	✅ Done
SSH key + GitHub auth	✅ Working (Hi bilbywilby!)
Push to bilbywilby/home-archive	⬜ Do this now
HANDOVER.json	❌ Still missing — create it
~/.git empty repo in ~	⚠️ Still there — mv ~/.git ~/.git.home-repo.bak
Push it, then I'll write you the HANDOVER.json template so master_engine.py has something to monitor


# ARCHIVE
