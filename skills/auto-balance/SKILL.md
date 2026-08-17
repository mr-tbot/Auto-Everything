---
name: auto-balance
description: "Use at the start of a session on a machine running several editors, agents or builds at once — when builds fight for CPU/RAM, an OOM kill lands, devices are contested (Android phones, ESP32 or dev boards on serial, iOS devices on a remote Mac), or the user mentions resource limits, parallel sessions, subagent count, account limits, or says \"/auto-balance\"; or before starting a long build or flashing a board."
---

# /auto-balance

Make this session a good citizen on a machine that is running several others. Cap what it consumes,
take turns on hardware, and size its own agent count so the account never hits a limit.

**First run does setup and writes the environment description down. Every later run re-balances
against what is actually running now.**

## Step 1 — First-Run Setup (once per repo)

Ask, then persist to `.agent/balance.json` (gitignored — it may name hosts):

1. **The machine**: cores, RAM, and whether builds routinely OOM today.
2. **Typical concurrency**: how many editor windows / agent sessions run at once.
3. **Attached hardware**: Android devices, dev boards, ESP32s, anything on serial.
4. **Remote hosts**: a Mac for iOS work, a build server. **Ask for connection details** — an SSH
   `Host` alias, a Tailscale name, the user to connect as. Store the *alias*, never a key or password;
   the key lives in `~/.ssh/` and is referenced by config.
5. **Claude plan and session habits**: which plan, and how many sessions typically run at once.

Record what you measured alongside what you were told:

```bash
nproc; grep -E '^(MemTotal|MemAvailable|SwapTotal)' /proc/meminfo
cat /sys/fs/cgroup/user.slice/user-$(id -u).slice/user@$(id -u).service/cgroup.controllers
oomctl        # works unprivileged: SwapUsedLimit, DefaultMemoryPressureLimit, duration
```

`oomctl` is step zero on every run — it tells you *what will kill you and at what threshold* before you
start anything.

## Step 2 — Know What Actually Works Here

Resource controls fail silently more often than they fail loudly. Establish the truth per machine:

**cgroup delegation.** A user slice typically delegates `cpu memory pids` and **not `io`**. Consequence:

```bash
systemd-run --user --scope -p CPUQuota=50%  ...    # works
systemd-run --user --scope -p MemoryMax=2G  ...    # works
systemd-run --user --scope -p IOWeight=50   ...    # EXIT 0, NO ERROR, NO EFFECT
```

Detect the silent discard by comparing intent against reality — **`IOWeight=[not set]` after passing
`-p IOWeight=` is the signature**:

```bash
systemctl --user show <unit> -p CPUQuotaPerSecUSec -p MemoryHigh -p IOWeight
```

Do **not** infer from a missing `cpu.max` file: systemd enables controllers on demand, so absent means
either "discarded" or "you never asked", and the file cannot tell you which.

**`nice` is one-way.** You can lower your own priority; raising it back is `Permission denied`. Set it
on the child at spawn, never on the session shell, or the session stays deprioritized for its life.
And in scripts use `renice --priority N` (always absolute) or `--relative N` — **never `-n`**, whose
meaning flips based on `POSIXLY_CORRECT`, which some test harnesses export.

**Threads do not inherit.** `renice -p` moves only the main thread and `renice -g` means process
*group*, not threads. `taskset` has `-a/--all-tasks`; `ionice` has `-P/--pgid`; `renice` has neither:

```bash
for t in /proc/$PID/task/*; do renice --priority 19 -p "$(basename "$t")"; done
```

**`ionice` may be inert.** Check the scheduler — if every device reads `none mq-deadline`, ioprio
barely bites (bfq is what honours it, and it is often an unloaded module). **Never**
`echo bfq > /sys/block/*/queue/scheduler`: that is a global, root-only mutation affecting every user.
Same forbidden class: `/proc/sys/vm/*`, cpufreq governors, and `renice -u <user>` (which hits that
user's editor too).

## Step 3 — Decide Whether To Start At All

Use PSI, not load average — it catches contention that load average shows as zero:

```bash
cat /proc/pressure/{cpu,io,memory}     # some avg10 is the "right now" number
grep MemAvailable /proc/meminfo
```

**A snapshot gate is TOCTOU by construction** — two sessions sampling an idle box in the same second
both start. The gate must be held under a lock (Step 4).

For event-driven backoff, register a PSI trigger — with two constraints that break naive code:

- **The trigger string must end in `\n`.** Without it you get `EINVAL`: the kernel copies
  `min(nbytes,32)` and NUL-terminates by clobbering the last byte, eating the final digit of your
  window.
- **Unprivileged windows must be multiples of 2s**, and triggers only work on a cgroup **you own** —
  writing to `user@$UID.service/memory.pressure` is `EACCES` (root-owned). Create your own cgroup or
  use a transient scope.

## Step 4 — Take Turns

`flock` is the correct primitive because **the kernel releases it when the fd closes** — clean exit,
segfault, or SIGKILL alike. No stale-lock problem by construction.

```bash
flock -x -w 1800 /run/user/$UID/agent-build.lock -c 'make -j6'
# payload that spawns children: flock -o (close fd before exec) or the in-script form:
exec 9>/run/user/$UID/agent-build.lock; flock -n 9 || exit 1
```

**Never build a PID-file lock.** Every throttling tool takes a raw PID, which invites one, and PID
reuse means you throttle a stranger's process. If you must track a PID, validate it with the start
time in field 22 of `/proc/PID/stat` plus the boot id — and read `/proc/sys/kernel/pid_max`, because a
32768 value is a fast-wrap environment.

`systemd-run --user --scope --unit=NAME` **fails if that scope already exists** — free mutual
exclusion. Pair with `--collect` so a failed unit does not block the next run by name.

## Step 5 — Run The Build Politely

```bash
systemd-run --user --scope --collect --unit=agent-build-$$ \
  -p CPUQuota=600% -p MemoryHigh=8G -p MemoryMax=12G \
  nice -n 10 ionice -c 3 make -j6
```

Cap **every** layer, because nested parallelism multiplies — `make -j8` each spawning `cargo -j8` is
64 compilers. Set the per-tool knob as well as the cgroup: `make -j`/`-l`, `ninja -j`,
`CMAKE_BUILD_PARALLEL_LEVEL`, `CARGO_BUILD_JOBS`, `org.gradle.workers.max` **and**
`org.gradle.jvmargs` **and** the Kotlin daemon's own heap (it is a separate JVM), `xcodebuild -jobs`,
`bazel --jobs`, `MSBuild /m`, `pytest -n`.

**Account for daemons you did not start.** Gradle, Kotlin, Bazel, sccache and TypeScript watchers hold
memory between builds and ignore your `-j`. Inspect and stop them before sizing anything.

**Scoping does not protect a build from systemd-oomd — it makes it the preferred victim.** oomd kills
*leaf* descendants chosen by reclaim activity, and the build is the leaf reclaiming hardest. The real
reason to scope is that an unscoped build inherits the *terminal's* scope, so the kill takes your
terminal with it. Use `ManagedOOMPreference=avoid` on things you want spared (honoured only when the
cgroup owners match), and remember memory accounting must be on or the cgroup is never monitored.

**Prefer freezing to killing.** When another session needs the machine:

```bash
echo 1 > $CG/cgroup.freeze     # suspend the whole tree, no work lost
echo 0 > $CG/cgroup.freeze     # resume
echo 1 > $CG/cgroup.kill       # race-free whole-tree kill, leaks no orphans
```

Size the next run honestly from `memory.peak`, `pids.peak` and `memory.events` (its `high`/`max`
counters say whether your limit actually throttled) rather than guessing. On finishing, hand memory
back with `memory.reclaim` instead of leaving page cache for the next session to fight.

## Step 6 — Arbitrate Devices

**One claim per physical device, held for the duration, released on exit.** Same `flock` lease
discipline, keyed on a stable device identity.

**Android.** Run exactly one adb server machine-wide, and **pin one adb binary first** — a distro
`/usr/bin/adb` ahead of the SDK's `platform-tools` on `PATH` produces two servers that kill each
other. Claim by serial (`adb -s <serial>` / `$ANDROID_SERIAL`), never by index. Check device *state*
before use — `unauthorized`, `offline` and `recovery` are not `device`. Expect a disappearing-device
window around `adb reboot`, `adb root`, an install, and any `tcpip` switch: wait for it back rather
than failing. Never `adb kill-server` while another session may be mid-install.

**Emulators are a policy decision, not a default.** An emulator or simulator is one of the heaviest
processes on the box and will push a loaded machine into swap or an OOM kill that lands on another
session's build — record the project's policy in `.agent/balance.json` at setup and honour it. Where
they are permitted, pin each to its own AVD and use `-port <even 5554..5584>` (adb is console+1);
adb only scans up to 5585.

**Serial boards.** Take the lease **before** invoking `esptool` — esptool's own exclusive open detects
a collision but does not prevent the damage, and any "is the port free?" probe that opens the tty
**resets the board**. Use `fuser`/`lsof` to check occupancy without opening.

Identify boards by a stable path, and verify the assumption: `/dev/serial/by-id/` is unusable when two
identical adapters report the same serial (two CP2102s both at `0001` is common), and a board may have
no `by-id` entry at all. `/dev/serial/by-path/` or a per-board udev rule keyed on `KERNELS`/`ID_PATH`
is the reliable identity. **Audit for multiple udev rules claiming the same SYMLINK** — that is a
coin-flip over which board a name resolves to.

On Linux, ModemManager will grab a serial device and can corrupt a flash; suppress it per-device by
udev rule rather than disabling it globally.

**iOS on a remote Mac.** Connect over the stored SSH alias with `ControlMaster`/`ControlPersist` for
reuse. Enumerate with `xcrun devicectl list devices` (or `xcrun simctl list` for simulators), select by
UDID, and hold the same lease on the remote host. Xcode's own device manager conflicts with concurrent
CLI use — one claimant at a time.

**Release contract:** always release on exit, including on failure. A crashed session holding a device
lease blocks everyone; `flock` handles this for you, which is why it is the primitive.

## Step 7 — Size This Session's Agents

Set concurrency from the account and the machine, whichever is tighter:

- Divide by **actual concurrent sessions**, not by cores alone. Four sessions on a 16-core box is 4
  effective cores each, and the memory division matters more than the CPU one.
- Respect the plan. Ask the user their plan and typical session count; treat published limits as
  changeable and **check rather than hardcode** — the observable symptom of hitting one is a refusal
  or a throttle, not a crash.
- Fewer, longer-running agents beat many short ones for both limits and coherence.

State the chosen number and the reason. If the machine is already at pressure, the correct answer is
sometimes **one**, or to wait.

## Step 8 — Re-Balance On Every Later Run

Read `.agent/balance.json`, then measure reality: how many editor windows and agent sessions are live
now, what is building, what devices are claimed and by whom, and what the current pressure is. Adjust
caps and agent count, report what changed, and update the file if the environment did.

## Rationalization Table

| Excuse | Reality |
|---|---|
| "I set IOWeight, I/O is capped" | Check `systemctl --user show -p IOWeight`. `[not set]` means it was discarded |
| "cpu.max is missing, my limit failed" | Or you never set one. Controllers are enabled on demand |
| "`nice`'d it, so it's polite" | Only the main thread, and only if the tool is CPU-bound |
| "`ionice -c 3` handles the disk" | Not on `none`/`mq-deadline`. Check the scheduler |
| "I'll switch the scheduler to bfq" | Global root mutation affecting every user. Never |
| "Scoping protects the build from OOM" | It makes it the preferred victim. It protects your *terminal* |
| "The box looks idle, I'll start" | Two sessions just made that same measurement. Take the lock |
| "I'll track the build by PID" | PID reuse means throttling a stranger. Use flock |
| "Another session needs the box, I'll kill mine" | Freeze it. `cgroup.freeze` loses no work |
| "adb sees the device, it's mine" | Claim it. Two sessions driving one device is how a flash corrupts |
| "by-id is a stable name" | Not for two identical adapters sharing a serial. Use by-path or a udev rule |
| "I'll just check if the serial port is free" | Opening the tty resets the board. Use `fuser`/`lsof` |
| "More agents is faster" | Not past the memory wall, and not past the account limit |

## Red Flags — Stop

- Claiming a limit is in force without reading it back from `systemctl show` or the cgroup file
- A resource gate that is a bare snapshot with no lock around it
- Any PID-file-based lock
- Touching a global knob: I/O scheduler, `/proc/sys/vm/*`, cpufreq, `renice -u`
- Killing another session's build when freezing would have done
- Driving a device without a lease, or probing a serial port by opening it
- Running `adb kill-server` on a shared machine
- Choosing an agent count without asking the plan or measuring concurrent sessions
- Leaving a lease or a manual cgroup behind on exit
