# hurd-SMP

## Purpose

Fixes to the GNU Hurd servers and libraries for running Debian GNU/Hurd
x86_64 on several processors.

This is the Hurd part of [mach-SMP](https://github.com/MevPR/mach-SMP),
which lets GNU Mach run ordinary tasks on every CPU (boot option
`smp_pset=default`).  With real parallelism, races in the Hurd libraries
that a single CPU almost never hits become frequent: crashes and hangs
of the root file system server, mostly.  This repository holds the fixes.

Base: the GNU Hurd from git.savannah.gnu.org/git/hurd/hurd.git (branch
`upstream`).  The original Hurd README is unchanged in [`README`](README).

## Related repositories

The project is split into three repositories that belong together:
running Debian GNU/Hurd on several CPUs needs the fixes from all three.

| Repository | Component |
|---|---|
| [mach-SMP](https://github.com/MevPR/mach-SMP) | GNU Mach kernel (SMP fixes, `smp_pset=default`, tests); investigation notes, tools and test scripts for the whole project; the netdde and rumpkernel patches |
| **hurd-SMP** (this one) | Hurd libraries and servers (libports, libfshelp, libpager, pflocal, console-client) |
| [hurd-glibc](https://github.com/MevPR/hurd-glibc) | glibc's Hurd port (fork deadlock; `multiple_threads` for glibc ≤ 2.42) |
| [chromium-hurd](https://github.com/MevPR/chromium-hurd) | Chromium for the Hurd, used as the acceptance test: it opens and renders https://www.duckduckgo.com on 2 CPUs |

## Status

| | |
|---|---|
| **Fixes** | 6 patches on current upstream master (branch `main` = `smp-fixes` + this README) |
| **Tested** | on Debian GNU/Hurd amd64 (hurd 0.9.git20250801, rebuilt in a Hurd VM with these fixes; libports in the form adapted to that version), together with the gnumach and glibc fixes: a parallel build/sort/tar/find/fork stress test on 2 CPUs passes 6 times out of 6 |
| **Tested with Chromium** | on Debian GNU/Hurd unstable of November 2026 (hurd 0.9.git20261028, where all 6 patches apply unchanged): with these fixes installed, Chromium opens and renders https://www.duckduckgo.com on 2 CPUs, twice; with Debian's servers without them, the same load made ext2fs report "looked up a bogus port" and the system hang |
| **Upstream** | prepared for bug-hurd@gnu.org, **not yet sent** |

## What was fixed

- **libports: double deallocation of a port.**  Dropping the last hard
  reference and taking the hash table lock were separate steps, so two
  threads could both see a port become unreferenced and both deallocate
  it.  The last reference is now dropped with the hash table lock held.
  (`libports: drop the last reference with the hash table lock held`)
- **libports: use after free in payload lookups.**  A message dequeued
  before its port was destroyed could still carry the port as protected
  payload; the lookup then took a reference on freed memory.  The memory
  of a port is now freed only once all threads went through a quiescent
  state, and lookups refuse a port that is being destroyed.
  (`libports: do not free port_info under the feet of payload lookups`)
- **libfshelp: use after free of record locks.**  Releasing a lock freed it
  right after waking the threads blocked on it in `F_SETLKW`, which then
  touched freed memory (seen in ext2fs during `flock` on
  `/var/lib/wtmpdb/wtmp.db`).  Waiters are now counted, and the last one
  frees the lock.
  (`libfshelp: do not free a record lock under the feet of its waiters`)
- **libpager: control port used while the object terminates.**
  `_pager_lock_object` and `pager_change_attributes` sent requests on the
  memory object control port without the interlock while
  `memory_object_terminate` could destroy it and the name be reused
  (gnumach: "task ext2fs looked up a bogus port ... for 2095").
  Termination is now blocked around the request.
  (`libpager: do not use the control port while the object may terminate`)

- **pflocal: abort when a socket and its address die together.**  When
  the last user port of a bound socket and the port of its address went
  away at the same time (processes dying at shutdown or boot), both
  `sock_deref` and `addr_unbind` unbound the socket; `sock_deref` then got
  `EINVAL` and its `assert_backtrace` aborted pflocal, taking all Unix
  sockets of the system with it.  `sock_bind` also cleared the address'
  socket pointer without the address lock, so other timings double
  dereferenced socket and address.  Unbinding now locks the address
  before the socket, as `addr_unbind` does.  With the fix, 6 of 6 test
  boots ran without a pflocal abort, against 3 crashes in 4 boots before.
  (`pflocal: unbind a socket with its address locked`)

- **console-client: the Hurd console died when it started slowly.**  With
  `--daemonize`, the parent waits a fixed time for the daemon to report;
  when the daemon is done later, its report to the parent that already
  exited raises SIGPIPE and kills it.  Under QEMU without KVM the vga
  driver needs more than 20 s to parse the 5.7 MB `unifont.bdf`, so every
  boot ended without a console client.  The report is now sent with
  SIGPIPE ignored.  With the fix the client was running after 6 of 6
  boots, before after none.
  (`console-client: survive a parent that gave up waiting`)

The investigation behind each fix is in
[mach-SMP: smp-dev/notes/hang-fork-suspend.md](https://github.com/MevPR/mach-SMP/blob/main/smp-dev/notes/hang-fork-suspend.md).

## Branches

| Branch | Content |
|---|---|
| `upstream` | mirror of git.savannah.gnu.org/git/hurd/hurd.git master |
| `main` | the fix series + this README |
| `smp-fixes` | the fix series alone, as prepared for bug-hurd |
| `libports-final-deref`, `libfshelp-rlock-wait`, `libpager-memobjcntl-race`, `pflocal-unbind-race`, `console-late-daemon` | one topic each, on `upstream` |

Work by Magnus Reimann, done together with Claude (an AI model by
Anthropic).  The GNU Hurd is free software of the GNU project.
