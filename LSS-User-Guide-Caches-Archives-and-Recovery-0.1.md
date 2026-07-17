# Caches, Archives, and Recovery

**Project:** Lunar Script System User Guide  
**Status:** Draft 0.1  
**Date:** 2026-07-17  
**Audience:** Lunar Linux users, maintainers, and system administrators  
**Evidence basis:** LSS Evidence Foundation 0.1 — Checkpoints 1–6

## 1. Overview

LSS can preserve reusable installation artifacts after a successful build.

These artifacts are stored under:

```text
/var/cache/lunar
```

They are distinct from:

```text
source archives
build directories
install manifests
compile logs
MD5 logs
package-state records
dependency-state records
```

Understanding these distinctions is essential for recovery, reproducibility, and troubleshooting.

## 2. Main artifact types

A normal module lifecycle may produce or use several artifact classes.

```text
source archive
→ upstream source package

build tree
→ unpacked and compiled working directory

installation cache
→ reusable installed payload

install manifest
→ final paths attributed to the module

compile log
→ build-time output

MD5 log
→ installed checksums

package record
→ current module state

dependency record
→ current relationship state
```

These artifacts may refer to the same module and version, but they serve different purposes.

## 3. Source archives

Source archives contain upstream source code.

They may be downloaded from:

- upstream project sites;
- GitHub or similar hosting;
- Lunar mirrors;
- configured local sources.

Example:

```text
xxHash-0.8.3.tar.gz
```

A source archive is not an installed package cache.

It still requires:

```text
unpack
→ configure
→ compile
→ install
```

unless another reusable artifact is available.

## 4. Build directories

The build directory contains the unpacked and transformed source tree.

It may include:

- generated files;
- object files;
- temporary binaries;
- patched source;
- configure results;
- build-system state.

It is working material, not authoritative installed state.

A build tree may contain files that never reach the system.

It may also lack files generated later during installation.

## 5. Installation caches

When archiving is enabled, LSS may create a cache such as:

```text
/var/cache/lunar/module-version-triplet.tar.xz
```

Observed example:

```text
/var/cache/lunar/xxhash-0.8.3-x86_64-pc-linux-gnu.tar.xz
```

The cache is intended to support installation or resurrection without repeating the full compile path.

## 6. `ARCHIVE`

The global setting:

```text
ARCHIVE=on
```

enables creation of installation archives after successful installation.

Inspect:

```bash
grep -n 'ARCHIVE' /etc/lunar/config
```

Possible value:

```text
ARCHIVE=${ARCHIVE:-on}
```

The effective value may also be influenced by environment or local configuration.

## 7. Inspecting caches

List all caches:

```bash
ls -lh /var/cache/lunar
```

Find one module:

```bash
ls -lh /var/cache/lunar/module-*
```

Example:

```bash
ls -lh /var/cache/lunar/xxhash-*
```

Count matching archives:

```bash
find /var/cache/lunar   -maxdepth 1   -type f   -name 'xxhash-*'   -print
```

## 8. Cache naming

A cache name commonly encodes:

```text
module
version
target triplet
compression
```

Example:

```text
xxhash-0.8.3-x86_64-pc-linux-gnu.tar.xz
```

This helps distinguish:

- version;
- target architecture;
- target system tuple;
- compression format.

It may not encode every meaningful build choice.

## 9. Cache identity is incomplete

Two builds can share:

```text
same module
same version
same target triplet
```

but differ in:

- optional dependencies;
- compiler flags;
- provider choice;
- configure options;
- toolchain version;
- patches;
- local environment.

Therefore:

```text
matching cache filename
≠ guaranteed semantic equivalence
```

A cache may be technically loadable but operationally wrong for the current intended configuration.

## 10. Cache resurrection

Source analysis shows that `lin_module` may attempt cache resurrection before the normal compile path.

Conceptually:

```text
module requested
→ check reusable cache
→ restore if acceptable
→ otherwise continue with source build
```

This can reduce build time significantly.

It can also complicate troubleshooting if the administrator expects a fresh compile.

## 11. Detecting cache use

Possible clues:

- operation completes unusually quickly;
- no normal compiler output appears;
- console mentions recovery or resurrection;
- package state changes without a new compile log;
- cache timestamp precedes installation time;
- installed payload matches an older configuration.

Preserve console output:

```bash
lin module 2>&1 | tee install-console.log
```

Inspect activity:

```bash
grep 'module' /var/log/lunar/activity
```

Inspect compile logs:

```bash
ls -l /var/log/lunar/compile/module-*
```

## 12. Cache versus install manifest

The cache contains reusable installation content.

The install manifest records final ownership.

The relationship is:

```text
cache
→ material that can be restored

manifest
→ paths LSS attributes after installation
```

A cache without a valid manifest is not sufficient for correct removal.

A manifest without a cache is still sufficient for ownership and removal, but not for fast restoration.

## 13. Cache versus package state

A cache can exist even when the module is not currently installed.

A module can be installed even when no cache exists.

Therefore:

```text
cache present
≠ installed

installed
≠ cache present
```

Check current installation through:

```bash
grep '^module:' /var/state/lunar/packages
```

Check cache separately:

```bash
ls -l /var/cache/lunar/module-*
```

## 14. Cache versus compile log

A cache may have been created from an earlier build.

The current installation may have been restored from it.

The compile log may therefore describe:

- the original build;
- a previous installation;
- no current compile at all.

Do not assume the compile-log timestamp always matches the latest installation transaction.

Compare timestamps and activity history.

## 15. Rebuild when a fresh compile is required

A fresh compile is preferable when:

- compiler flags changed;
- toolchain changed;
- optional dependencies changed;
- Moonbase recipe changed;
- patches changed;
- provider changed;
- reproducibility is being tested;
- old cache provenance is uncertain.

In such cases, do not rely blindly on cache restoration.

Preserve or move the existing cache first.

Example:

```bash
mkdir -p /root/lunar-cache-saved

mv /var/cache/lunar/module-version-*   /root/lunar-cache-saved/
```

Then rebuild.

Use a disposable environment for controlled experiments.

## 16. Preserving caches

Copy a module cache:

```bash
mkdir -p /root/lunar-cache-backup

cp -a /var/cache/lunar/module-version-*   /root/lunar-cache-backup/
```

Record checksum:

```bash
sha256sum /root/lunar-cache-backup/module-version-*   > /root/lunar-cache-backup/SHA256SUMS
```

Record metadata:

```bash
stat /root/lunar-cache-backup/module-version-*   > /root/lunar-cache-backup/stat.txt
```

This helps distinguish cache corruption from build problems.

## 17. Inspecting cache contents

For an XZ-compressed tar archive:

```bash
tar -tJf /var/cache/lunar/module-version-triplet.tar.xz
```

For gzip:

```bash
tar -tzf archive.tar.gz
```

For bzip2:

```bash
tar -tjf archive.tar.bz2
```

List before extracting.

Do not extract directly into `/`.

Use a temporary directory:

```bash
mkdir -p /tmp/lunar-cache-inspect

tar -xJf archive.tar.xz   -C /tmp/lunar-cache-inspect
```

## 18. Comparing cache and manifest

Create sorted lists:

```bash
tar -tJf archive.tar.xz |
  sed 's#^\./##' |
  sort > cache.paths

sed 's#^/##' install-manifest |
  sort > manifest.paths
```

Compare:

```bash
diff -u manifest.paths cache.paths
```

Interpret carefully.

The archive may encode paths differently or omit metadata files created outside the cached payload.

## 19. Cache integrity

Verify the archive structure:

```bash
xz -t archive.tar.xz
```

Then verify tar readability:

```bash
tar -tJf archive.tar.xz >/dev/null
```

Checksum:

```bash
sha256sum archive.tar.xz
```

A readable archive can still be semantically stale.

Integrity and suitability are different questions.

## 20. Cache recovery risks

Potential risks:

- wrong optional features;
- stale configuration;
- old toolchain output;
- outdated module recipe;
- incompatible ABI;
- missing post-install side effects;
- old service integration;
- mismatched ownership metadata.

A cache can restore files correctly while still producing the wrong operational system.

## 21. Recovery after accidental removal

Possible recovery sources:

```text
installation cache
source archive
Moonbase module
saved manifest
saved MD5 log
filesystem backup
container snapshot
system backup
```

Preferred recovery order depends on confidence.

If a trusted cache exists:

```text
restore through LSS
→ regenerate state
→ verify ownership
→ verify runtime
```

Avoid manually unpacking a cache into `/` unless performing expert recovery.

Manual extraction may bypass:

- package-state registration;
- dependency-state updates;
- manifest generation;
- configuration policy;
- post-install actions.

## 22. Recovery after failed rebuild

A failed rebuild may leave:

- old package removed;
- partial new payload;
- incomplete manifest;
- stale package record;
- missing logs;
- valid old cache.

Recovery sequence:

```text
preserve current evidence
→ inspect package state
→ inspect manifest
→ inspect cache
→ determine whether old payload remains
→ restore through LSS when possible
→ verify state
```

Do not immediately delete temporary files.

They may explain the failure.

## 23. Recovery after missing manifest

If the package record exists but the install manifest is missing:

```text
installed state
→ present

ownership evidence
→ missing
```

This is dangerous because normal removal cannot reliably know what belongs to the module.

Possible recovery:

1. preserve package and dependency state;
2. inspect caches;
3. rebuild or reinstall the same module configuration;
4. regenerate manifest;
5. compare resulting files;
6. only then consider removal.

Do not invent a manifest manually unless no better recovery path exists.

## 24. Recovery after stale package record

If the package record exists but payload is missing:

```text
packages database
→ says installed

filesystem
→ disagrees
```

Possible causes:

- manual deletion;
- interrupted operation;
- filesystem damage;
- restored state database without payload.

Preferred recovery:

```text
reinstall or rebuild module
→ regenerate payload and logs
→ verify package state
```

Manual database deletion hides the inconsistency without repairing the system.

## 25. Recovery after stale dependency state

If dependency records reference missing modules:

1. preserve `/var/state/lunar/depends`;
2. inspect affected module declarations;
3. inspect package records;
4. rebuild or reconfigure dependent modules;
5. verify new dependency state;
6. remove obsolete records only through supported LSS operations.

The goal is to restore alignment, not merely silence an error.

## 26. Recovery from a container snapshot

For experiments, a container snapshot or backup is often the safest recovery mechanism.

Recommended pattern:

```text
backup container
→ perform experiment
→ preserve evidence
→ restore if necessary
```

This protects the real system while still allowing full lifecycle testing.

## 27. Recovery from system backup

For critical systems, preserve:

```text
/etc
/var/state/lunar
/var/log/lunar
/var/cache/lunar
Moonbase
local configuration
important user data
```

A state-only backup is insufficient if installed payload is lost.

A filesystem-only backup is insufficient if LSS state is lost.

Recovery requires both:

```text
files
+
state
```

## 28. `REAP`

The global setting:

```text
REAP=on
```

controls cleanup behavior.

Inspect:

```bash
grep -n 'REAP' /etc/lunar/config
```

Cleanup may remove temporary build material after success.

This reduces disk use but also removes evidence useful for deep debugging.

For a difficult build problem, consider preserving the build tree before cleanup.

## 29. Preserving a build tree

When a build fails, identify its temporary source/build directory.

Copy it before another run or cleanup:

```bash
cp -a /path/to/build-tree   /root/lss-debug/module-build-tree
```

Also preserve:

```text
configure logs
generated Makefiles
object files
patched sources
command output
```

This can reveal differences not visible in the compressed compile log.

## 30. Archive cleanup

Caches accumulate over time.

Review:

```bash
du -sh /var/cache/lunar
find /var/cache/lunar -maxdepth 1 -type f | wc -l
```

List by size:

```bash
du -h /var/cache/lunar/* | sort -h
```

List by age:

```bash
find /var/cache/lunar   -maxdepth 1   -type f   -printf '%TY-%Tm-%Td %TH:%TM %p\n' |
  sort
```

Do not delete caches solely because they are old.

They may be the fastest recovery path for a difficult package.

## 31. Safe cache cleanup

A careful process:

```text
identify obsolete versions
→ verify current installed version
→ preserve critical caches
→ remove only known-unneeded archives
→ keep recovery path for core modules
```

For one module:

```bash
MODULE=name
CURRENT=$(lvu installed "$MODULE")

ls -l /var/cache/lunar/${MODULE}-*
```

Retain the current trusted cache when possible.

## 32. Core-module caches

Caches for critical modules deserve special value.

Examples:

```text
glibc
gcc
bash
coreutils
lunar
openssl
kernel-related modules
```

A trusted cache can reduce recovery time if a source rebuild becomes impossible.

Preserve checksums and provenance.

## 33. Cache provenance

A useful cache record should include:

```text
module
version
creation date
Moonbase revision
toolchain versions
optional dependency choices
compiler flags
target triplet
checksum
```

The archive filename alone is not enough.

Create sidecar metadata:

```bash
cat > archive.metadata <<EOF
module=...
version=...
moonbase_revision=...
cc=...
cxx=...
cflags=...
created=...
EOF
```

## 34. Reproducible cache sets

For a known system baseline, preserve:

```text
selected caches
packages database
depends database
Moonbase revision
Lunar configuration
checksums
```

This creates a stronger recovery set than arbitrary cache accumulation.

It can support:

- container recreation;
- system restoration;
- regression comparison;
- offline recovery.

## 35. Offline recovery

A useful offline recovery set may contain:

```text
source archives
installation caches
Moonbase
LSS tools
state backups
checksums
documentation
```

Verify the set before it is needed.

A backup that has never been tested is only a hypothesis.

## 36. Recovery validation

After restoration:

```bash
grep '^module:' /var/state/lunar/packages
lvu installed module
cat /var/log/lunar/install/module-version
```

Verify payload:

```bash
command -v program
ldd /usr/bin/program
```

Verify configuration:

```bash
md5sum /etc/module.conf
```

Verify runtime behavior:

```text
start program
run feature test
check service
check plugin discovery
```

Recovery is complete only when state and runtime agree.

## 37. Cache troubleshooting checklist

If cache restoration gives an unexpected result:

```text
Was the cache actually used?
Does the version match?
Does the target triplet match?
Were options different?
Did Moonbase change?
Did the toolchain change?
Were post-install actions run?
Does the manifest match?
Does runtime behavior match intent?
```

## 38. Common mistakes

### Mistake 1: treating a cache as an installed package record

It is only reusable payload.

### Mistake 2: trusting filename identity

It may omit important build choices.

### Mistake 3: manually unpacking into `/`

This bypasses LSS state management.

### Mistake 4: deleting all old caches

This removes valuable recovery paths.

### Mistake 5: keeping caches without provenance

Their suitability becomes difficult to assess.

### Mistake 6: repairing only state or only files

Recovery requires alignment between both.

### Mistake 7: assuming archive integrity means configuration correctness

A valid archive can still be stale.

## 39. Safe recovery model

```text
preserve evidence
→ identify the damaged layer
→ select trusted recovery artifact
→ restore through LSS
→ regenerate state
→ verify ownership
→ verify runtime
```

## 40. Summary

LSS caches and archives support fast restoration, but they are not substitutes for state.

The full recovery model is:

```text
source
→ rebuild capability

cache
→ reusable installed payload

manifest
→ ownership

MD5 log
→ installed checksums

packages
→ current module state

depends
→ relationship state

backup
→ broader restoration
```

The central rule is:

```text
treat cache as evidence and reusable material
→ not as complete system truth
