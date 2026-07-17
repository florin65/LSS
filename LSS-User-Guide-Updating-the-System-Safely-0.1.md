# Updating the System Safely

**Project:** Lunar Script System User Guide  
**Status:** Draft 0.1  
**Date:** 2026-07-17  
**Audience:** Lunar Linux users and system administrators  
**Evidence basis:** LSS Evidence Foundation 0.1 — Checkpoints 1–6

## 1. Overview

Updating a source-based system is not a single file replacement.

An update may affect:

- module definitions;
- dependency choices;
- toolchain behavior;
- ABI compatibility;
- installed manifests;
- caches;
- local configuration;
- service integration;
- future rebuilds.

A safe update process is therefore:

```text
inspect
→ plan
→ preserve state
→ update in dependency order
→ verify
→ expand gradually
```

## 2. Types of update

Common update types include:

```text
single module version update
single module rebuild
dependency update
provider replacement
toolchain update
library ABI transition
Moonbase update
large system rebuild
kernel or boot component update
LSS self-update
```

Each has a different risk profile.

## 3. Small versus systemic updates

A small leaf utility update usually has a limited blast radius.

A systemic update may affect many modules.

High-impact examples:

```text
glibc
gcc
binutils
llvm
openssl
zlib
python
perl
pkgconf
lunar
kernel
```

Treat toolchain and core-library transitions as separate engineering operations.

## 4. Before updating Moonbase

Record the current state:

```bash
cp -a /var/state/lunar/packages   /root/packages.before-moonbase

cp -a /var/state/lunar/depends   /root/depends.before-moonbase
```

Record the active Moonbase revision if it is managed by Git:

```bash
git -C /var/lib/lunar/moonbase rev-parse HEAD   > /root/moonbase-revision.before
```

Preserve local modifications:

```bash
git -C /var/lib/lunar/moonbase status --short
git -C /var/lib/lunar/moonbase diff   > /root/moonbase-local-changes.patch
```

Do not update over unreviewed local changes.

## 5. Inspecting available updates

Compare installed and Moonbase versions.

For one module:

```bash
printf 'installed: %s
' "$(lvu installed module)"
printf 'moonbase:  %s
' "$(lvu version module)"
```

For several modules:

```bash
for module in module1 module2 module3; do
  printf '%-24s installed=%-16s moonbase=%s
'     "$module"     "$(lvu installed "$module" 2>/dev/null)"     "$(lvu version "$module" 2>/dev/null)"
done
```

A version difference indicates a candidate update, not automatic safety.

## 6. Read the module change

Before updating, inspect:

- `DETAILS`;
- `BUILD`;
- `DEPENDS`;
- `CONFIGURE`;
- `OPTIONS`;
- lifecycle hooks;
- patches.

Compare old and new Moonbase revisions when possible:

```bash
git -C /var/lib/lunar/moonbase diff OLD..NEW --   section/module
```

Look for:

```text
version change
source URL change
checksum change
new dependency
removed dependency
new build option
path layout change
service change
remove hook
toolchain requirement
```

## 7. Preserve the current module state

For a module:

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
BASE=/root/lss-update/$MODULE

mkdir -p "$BASE/before" "$BASE/after"

grep "^${MODULE}:" /var/state/lunar/packages   > "$BASE/before/package-record" || true

grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends   > "$BASE/before/depends-records" || true

cp -a "/var/log/lunar/install/${MODULE}-${VERSION}"   "$BASE/before/install-log" 2>/dev/null || true

cp -a "/var/log/lunar/md5sum/${MODULE}-${VERSION}"   "$BASE/before/md5sum-log" 2>/dev/null || true

cp -a "/var/log/lunar/compile/${MODULE}-${VERSION}".*   "$BASE/before/" 2>/dev/null || true
```

For critical modules, preserve the cache and relevant configuration too.

## 8. Dependency order

Updates should respect dependency direction.

Conceptually:

```text
dependency
→ update first

dependent
→ rebuild or update afterward
```

A library update may require rebuilding consumers even when their version does not change.

Use:

```bash
lvu depends library
lvu leert library
```

to estimate reverse impact.

## 9. Topological ordering

LSS uses dependency graph ordering for multi-module operations.

The general principle is:

```text
build prerequisites first
→ then modules that consume them
```

Avoid manual arbitrary order when several linked modules are involved.

## 10. Single-module update

A controlled single-module update:

```bash
lin -c module 2>&1 |
  tee /root/lss-update/module/rebuild.log
```

Afterward:

```bash
lvu installed module
grep '^module:' /var/state/lunar/packages
```

Compare manifests and dependency state.

Test the actual program or library.

## 11. Updating a shared library

A shared library update may preserve the same SONAME or change it.

Check before:

```bash
readelf -d /usr/lib/libexample.so | grep SONAME
```

Check consumers:

```bash
lvu depends library
lvu leert library
```

After update:

```bash
ldconfig
ldd /usr/bin/consumer | grep 'not found'
```

A successful library build does not prove all consumers remain functional.

## 12. ABI transitions

An ABI transition may require rebuilding reverse dependents.

Warning signs:

```text
SONAME changed
major version changed
symbols removed
compiler runtime changed
C++ standard library changed
language runtime changed
```

Safe sequence:

```text
update library
→ verify installed library
→ rebuild direct consumers
→ test
→ rebuild indirect consumers as needed
```

## 13. Toolchain updates

Toolchain transitions deserve special handling.

Components may include:

```text
binutils
gcc
glibc
llvm
clang
lld
compiler-rt
libc++
make
pkgconf
```

A toolchain update can affect every later build.

Record before:

```bash
gcc --version
clang --version
ld --version
make --version
```

Record relevant variables:

```bash
env | grep -E   '^(CC|CXX|CFLAGS|CXXFLAGS|LDFLAGS|MAKEFLAGS)='
```

## 14. Validate toolchain on a small module

Before rebuilding core components, test a small known module.

Good validation characteristics:

```text
small
quick to build
simple C or C++
clear manifest
easy runtime test
```

Verify:

- compilation;
- linking;
- installwatch;
- manifest;
- runtime.

Only then expand.

## 15. Compiler-family consistency

Do not mix compiler-family assumptions carelessly.

Example problem:

```text
BUILD expects Clang
→ environment uses GCC
→ Clang-specific flags fail
```

or the reverse.

For LLVM-family modules, explicit selection may be required:

```bash
CC=/usr/bin/clang
CXX=/usr/bin/clang++
```

The correct choice belongs in module or toolchain policy, not ad hoc repeated fixes.

## 16. Rebuilding reverse dependents

After a library or toolchain change, identify consumers:

```bash
lvu depends target
lvu leert target
```

Start with direct consumers.

Rebuild in waves:

```text
wave 1
→ direct dependents

wave 2
→ higher-level consumers

wave 3
→ applications and optional integrations
```

Verify after each wave.

## 17. Optional dependency changes during update

A Moonbase update may introduce new optional dependencies.

Do not automatically enable all of them.

Review:

```text
what feature does it add?
is it needed?
does it enlarge the dependency graph?
does it introduce a service or provider?
does it change attack surface?
```

Preserve administrator intent.

## 18. Required dependency changes

A new required dependency changes the minimum module form.

Before accepting:

1. inspect the dependency;
2. inspect its own dependencies;
3. inspect conflicts;
4. estimate build and runtime impact;
5. record the decision.

A required dependency is part of the updated module contract.

## 19. Configuration drift during updates

A newer module recipe may interpret old choices differently.

Possible drift:

```text
old option removed
new default introduced
provider renamed
optional dependency becomes required
configure auto-detection changes
```

After update, compare:

```bash
grep "^${MODULE}:" /var/state/lunar/depends
diff old-install-log new-install-log
```

Test intended features.

## 20. Preserve `/etc`

Before updating a service or daemon:

```bash
cp -a /etc/module.conf   /root/module.conf.before-update
```

Afterward:

```bash
diff -u   /root/module.conf.before-update   /etc/module.conf
```

Also inspect MD5 logs.

A rebuild may preserve, replace, merge, or leave configuration unchanged depending on module and upstream behavior.

## 21. Services

After updating a service:

```text
verify binary
→ verify configuration
→ restart or reload
→ inspect logs
→ test client behavior
```

Check service-manager state using the system's actual init framework.

Installation success is not service validation.

## 22. Kernel and boot updates

Kernel, initramfs, bootloader, and firmware updates are high risk.

Before:

- ensure a known-good kernel remains available;
- preserve boot configuration;
- preserve initramfs;
- verify filesystem space;
- verify recovery media;
- avoid deleting the previous working entry immediately.

After:

- verify generated files;
- verify bootloader entries;
- reboot only after inspection;
- retain fallback.

## 23. LSS self-update

Updating `lunar` or core LSS components can affect all future operations.

Before:

```bash
cp -a /var/lib/lunar/functions   /root/lunar-functions.before

cp -a /sbin/lin /sbin/lrm /bin/lvu   /root/lunar-programs.before/

cp -a /etc/lunar   /root/lunar-config.before
```

Use a container or recovery environment when testing major LSS changes.

## 24. Multi-module updates

For a planned set:

```text
group by dependency layer
→ update low-level components first
→ rebuild consumers
→ test after each layer
```

Do not combine unrelated high-risk transitions into one run.

Bad pattern:

```text
toolchain update
+ libc update
+ init update
+ package manager update
```

all at once.

This makes failure attribution difficult.

## 25. Update waves

A practical update strategy:

### Wave 1: low-risk leaves

Small utilities and isolated modules.

### Wave 2: common libraries

Shared libraries with manageable reverse dependency sets.

### Wave 3: applications

User-facing programs.

### Wave 4: services

Daemons and network-facing components.

### Wave 5: core and toolchain

Only after earlier validation.

## 26. Checkpoints between waves

After each wave, record:

```bash
cp -a /var/state/lunar/packages   packages.wave-N

cp -a /var/state/lunar/depends   depends.wave-N
```

Run basic tests.

Do not continue if state is inconsistent.

## 27. Disk space

Source builds require space for:

```text
source archive
unpacked tree
object files
installed payload
logs
cache archive
old version during transition
```

Check:

```bash
df -h
du -sh /var/cache/lunar
du -sh /var/log/lunar
```

A full filesystem during installation can leave partial state.

## 28. Source availability

Before a critical update, verify source access.

Potential problems:

```text
upstream URL removed
tag renamed
mirror unavailable
checksum changed
network failure
```

Preserve source archives for core modules.

## 29. Cache use during updates

A cache may accelerate recovery.

But after configuration, Moonbase, or toolchain changes, it may restore an old result.

For a controlled fresh update:

```text
preserve old cache
→ prevent accidental reuse
→ rebuild
→ create new cache
→ compare
```

## 30. Failed update response

If an update fails:

```text
stop expansion
→ preserve evidence
→ identify failed phase
→ inspect current package state
→ determine whether old payload remains
→ restore from trusted cache or rebuild
```

Do not immediately update other modules.

## 31. Partial installation detection

Check:

```bash
grep '^module:' /var/state/lunar/packages
ls -l /var/log/lunar/install/module-*
ls -l /var/log/lunar/md5sum/module-*
command -v program
```

Possible states:

```text
old record + old payload
new record + new payload
no record + partial payload
record present + missing manifest
```

Each requires a different recovery action.

## 32. Rollback

A rollback may use:

- trusted old cache;
- old source and Moonbase recipe;
- filesystem snapshot;
- container backup;
- system backup;
- previous kernel or boot entry.

Rollback should restore:

```text
payload
ownership manifest
MD5 log
package state
dependency state
configuration
```

Restoring only the binary is incomplete.

## 33. Verifying after update

### State

```bash
lvu installed module
grep '^module:' /var/state/lunar/packages
```

### Ownership

```bash
cat /var/log/lunar/install/module-version
```

### Dependencies

```bash
grep -E '^module:|^[^:]+:module:'   /var/state/lunar/depends
```

### Linking

```bash
ldd /usr/bin/program | grep 'not found'
```

### Configuration

```bash
diff -u config.before /etc/module.conf
```

### Runtime

Run an actual feature or service test.

## 34. System-wide consistency checks

After a larger update:

```bash
find /var/log/lunar/install   -maxdepth 1   -type f   | sort
```

Look for obvious missing state.

Check broken links:

```bash
find /usr -xtype l -print 2>/dev/null
```

Check missing shared libraries in important binaries.

Avoid indiscriminate full-filesystem scans during normal operation unless needed.

## 35. Activity review

Review recent transitions:

```bash
tail -200 /var/log/lunar/activity
```

Search failures:

```bash
grep -i 'failed' /var/log/lunar/activity | tail -50
```

Compare with console and compile logs.

## 36. Update documentation

For significant updates, record:

```text
date
modules
old versions
new versions
Moonbase revision
dependency changes
configuration changes
toolchain
tests performed
problems
recovery actions
```

This becomes valuable evidence for future maintenance.

## 37. Safe update checklist

Before:

```text
review Moonbase change
inspect dependencies
save state and logs
verify disk space
verify source availability
preserve configuration
prepare rollback
```

During:

```text
update in dependency order
change one major layer at a time
preserve console output
stop on unexplained failure
```

After:

```text
verify packages and depends
compare manifests
check linking
check configuration
test runtime
create checkpoint
```

## 38. Common mistakes

### Mistake 1: updating everything at once

Failure attribution becomes difficult.

### Mistake 2: ignoring reverse dependents

Library updates can break consumers.

### Mistake 3: trusting version numbers alone

Configuration and ABI may have changed.

### Mistake 4: deleting the old cache immediately

It may be the best rollback path.

### Mistake 5: rebuilding core modules before validating the toolchain

Test a small module first.

### Mistake 6: assuming build success means system success

Runtime and service verification remain necessary.

### Mistake 7: updating over local Moonbase changes

Preserve and review them first.

## 39. Recommended update model

```text
observe current system
→ preserve state
→ review module changes
→ update dependencies first
→ rebuild consumers
→ verify each wave
→ preserve rollback until confidence is high
```

## 40. Summary

Safe system updating in LSS is controlled graph evolution.

The administrator is not only replacing versions.

They are managing:

```text
source
dependencies
configuration
ABI
ownership
persistent state
runtime behavior
recovery paths
```

The central rule is:

```text
small validated steps
→ explicit checkpoints
→ gradual expansion
