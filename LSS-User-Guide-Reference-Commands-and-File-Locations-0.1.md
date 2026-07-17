# Reference Commands and File Locations

**Project:** Lunar Script System User Guide  
**Status:** Draft 0.1  
**Date:** 2026-07-17  
**Audience:** Lunar Linux users, maintainers, and system administrators  
**Purpose:** Quick operational reference

## 1. Core commands

### Install a module

```bash
lin module
```

Example:

```bash
lin foremost
```

### Rebuild an installed module

```bash
lin -c module
```

Example:

```bash
lin -c xxhash
```

### Remove a module

```bash
lrm module
```

Example:

```bash
lrm foremost
```

### Inspect a module

```bash
lvu installed module
lvu version module
lvu where module
lvu depends module
lvu leert module
```

## 2. Essential command meanings

```text
lin module
→ install module

lin -c module
→ rebuild module through upgrade-style replacement

lrm module
→ remove module through its ownership manifest

lvu installed module
→ show installed version

lvu version module
→ show version available in active Moonbase

lvu where module
→ show Moonbase section

lvu depends module
→ show installed reverse dependents

lvu leert module
→ show reverse dependency tree
```

## 3. Locate a module in Moonbase

```bash
MODULE=name
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

printf '%s
' "$MODULE_DIR"
```

List files:

```bash
find "$MODULE_DIR" -maxdepth 2 -type f -print | sort
```

## 4. Common module files

```text
DETAILS
BUILD
DEPENDS
CONFIGURE
OPTIONS
PRE_BUILD
POST_BUILD
POST_INSTALL
PRE_REMOVE
POST_REMOVE
CONFLICTS
```

Inspect:

```bash
sed -n '1,240p' "$MODULE_DIR/DETAILS"
sed -n '1,240p' "$MODULE_DIR/BUILD"
```

## 5. Package-state database

Location:

```text
/var/state/lunar/packages
```

Observed format:

```text
module:date:state-list:version:size
```

Example:

```text
xxhash:20260717:installed:0.8.3:520KB
```

Lookup:

```bash
grep '^module:' /var/state/lunar/packages
```

List installed modules:

```bash
awk -F: '$3 ~ /(^|\+)installed(\+|$)/ {print $1}'   /var/state/lunar/packages | sort
```

## 6. Policy states

Known state terms:

```text
installed
held
exiled
enforced
```

Meaning:

```text
installed
→ currently recorded as present

held
→ retain current version

exiled
→ reject module presence

enforced
→ require module presence
```

Inspect special states:

```bash
awk -F: '$3 ~ /held|exiled|enforced/ {print}'   /var/state/lunar/packages
```

## 7. Dependency-state database

Location:

```text
/var/state/lunar/depends
```

Observed format:

```text
module:dependency:status:type:field5:field6
```

Example:

```text
ccache:xxhash:on:required::
```

Inspect forward relationships:

```bash
grep '^module:' /var/state/lunar/depends
```

Inspect reverse relationships:

```bash
grep -E '^[^:]+:module:' /var/state/lunar/depends
```

Inspect both:

```bash
MODULE=name

grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends
```

## 8. Leaf modules

List:

```bash
lvu leafs | sort
```

Leaf means:

```text
no recorded reverse dependents
```

It does not automatically mean safe or unused.

## 9. Install manifests

Location:

```text
/var/log/lunar/install
```

Per-module file:

```text
/var/log/lunar/install/module-version
```

Inspect:

```bash
cat /var/log/lunar/install/module-version
```

Count paths:

```bash
wc -l /var/log/lunar/install/module-version
```

## 10. Manifest path classification

```bash
MANIFEST=/var/log/lunar/install/module-version

while IFS= read -r path; do
  if [ -L "$path" ]; then
    printf 'link	%s	%s
' "$path" "$(readlink "$path")"
  elif [ -f "$path" ]; then
    printf 'file	%s
' "$path"
  elif [ -d "$path" ]; then
    printf 'directory	%s
' "$path"
  else
    printf 'missing	%s
' "$path"
  fi
done < "$MANIFEST"
```

## 11. MD5 logs

Location:

```text
/var/log/lunar/md5sum
```

Per-module file:

```text
/var/log/lunar/md5sum/module-version
```

Inspect:

```bash
cat /var/log/lunar/md5sum/module-version
```

Compare a configuration file:

```bash
md5sum /etc/module.conf

grep '/etc/module.conf'   /var/log/lunar/md5sum/module-version
```

## 12. Compile logs

Location:

```text
/var/log/lunar/compile
```

Typical file:

```text
module-version.xz
```

Read:

```bash
xzless /var/log/lunar/compile/module-version.xz
```

Search errors:

```bash
xzgrep -n -i   -E 'error|failed|fatal|undefined|not found|cannot'   /var/log/lunar/compile/module-version.xz
```

## 13. Activity log

Location:

```text
/var/log/lunar/activity
```

Search module history:

```bash
grep 'module' /var/log/lunar/activity
```

Follow live:

```bash
tail -f /var/log/lunar/activity
```

## 14. Cache directory

Location:

```text
/var/cache/lunar
```

Find a module cache:

```bash
ls -lh /var/cache/lunar/module-*
```

Typical form:

```text
module-version-target-triplet.tar.xz
```

Inspect archive:

```bash
tar -tJf archive.tar.xz
```

Verify XZ integrity:

```bash
xz -t archive.tar.xz
```

## 15. Global configuration

Main file:

```text
/etc/lunar/config
```

Common settings:

```text
ARCHIVE
PRESERVE
REAP
```

Inspect:

```bash
grep -n -E 'ARCHIVE|PRESERVE|REAP'   /etc/lunar/config
```

## 16. Important LSS paths

```text
/sbin/lin
/sbin/lrm
/bin/lvu

/etc/lunar
/var/lib/lunar/functions
/var/lib/lunar/moonbase
/var/state/lunar/packages
/var/state/lunar/depends
/var/log/lunar/activity
/var/log/lunar/compile
/var/log/lunar/install
/var/log/lunar/md5sum
/var/cache/lunar
/usr/lib/installwatch.so
```

## 17. Environment inspection

```bash
uname -a
id

env | grep -E   '^(CC|CXX|CFLAGS|CXXFLAGS|LDFLAGS|MAKEFLAGS|PATH|PKG_CONFIG_PATH)='
```

Toolchain:

```bash
gcc --version
clang --version
ld --version
make --version
```

## 18. Installed versus Moonbase version

```bash
MODULE=name

printf 'installed: %s
'   "$(lvu installed "$MODULE" 2>/dev/null)"

printf 'moonbase:  %s
'   "$(lvu version "$MODULE" 2>/dev/null)"
```

## 19. Reverse dependency review

```bash
MODULE=name

lvu depends "$MODULE"
lvu leert "$MODULE"

grep -E "^[^:]+:${MODULE}:"   /var/state/lunar/depends
```

Run before removal or provider replacement.

## 20. Safe evidence capture

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
BASE=/root/lss-evidence/$MODULE

mkdir -p "$BASE/before" "$BASE/after"

cp -a /var/state/lunar/packages   "$BASE/before/packages"

cp -a /var/state/lunar/depends   "$BASE/before/depends"

grep "^${MODULE}:" /var/state/lunar/packages   > "$BASE/before/package-record" || true

grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends   > "$BASE/before/depends-records" || true

cp -a "/var/log/lunar/install/${MODULE}-${VERSION}"   "$BASE/before/install-log" 2>/dev/null || true

cp -a "/var/log/lunar/md5sum/${MODULE}-${VERSION}"   "$BASE/before/md5sum-log" 2>/dev/null || true

cp -a "/var/log/lunar/compile/${MODULE}-${VERSION}".*   "$BASE/before/" 2>/dev/null || true
```

## 21. Capture console output

Install:

```bash
lin "$MODULE" 2>&1 |
  tee "$BASE/install-console.log"
```

Rebuild:

```bash
lin -c "$MODULE" 2>&1 |
  tee "$BASE/rebuild-console.log"
```

Remove:

```bash
lrm "$MODULE" 2>&1 |
  tee "$BASE/remove-console.log"
```

## 22. Compare state before and after

```bash
cp -a /var/state/lunar/packages   "$BASE/after/packages"

cp -a /var/state/lunar/depends   "$BASE/after/depends"

diff -u   "$BASE/before/packages"   "$BASE/after/packages"

diff -u   "$BASE/before/depends"   "$BASE/after/depends"
```

## 23. Compare manifests

```bash
diff -u install.before install.after
```

No output means byte-for-byte identity.

## 24. Reproduce installed size

```bash
SIZE=0

while IFS= read -r path; do
  if [ -f "$path" ]; then
    value=$(du -k -- "$path" | awk '{print $1}')
    SIZE=$((SIZE + value))
  fi
done < /var/log/lunar/install/module-version

echo "${SIZE}KB"
```

Use `du -k` explicitly.

Do not recursively measure manifest directories.

## 25. Shared ownership search

```bash
grep -R -F -x '/path'   /var/log/lunar/install 2>/dev/null
```

Multiple results require review.

## 26. Missing path check

```bash
while IFS= read -r path; do
  [ -e "$path" ] || [ -L "$path" ] ||
    echo "missing: $path"
done < /var/log/lunar/install/module-version
```

## 27. Binary and linkage checks

```bash
command -v program
file /usr/bin/program
ldd /usr/bin/program
```

Missing libraries:

```bash
ldd /usr/bin/program | grep 'not found'
```

Library metadata:

```bash
readelf -d /usr/lib/libexample.so
```

## 28. Pkg-config checks

```bash
pkgconf --exists package
pkgconf --modversion package
pkgconf --print-requires package
pkgconf --print-requires-private package
```

## 29. Moonbase Git inspection

```bash
git -C /var/lib/lunar/moonbase status --short
git -C /var/lib/lunar/moonbase branch --show-current
git -C /var/lib/lunar/moonbase rev-parse HEAD
git -C /var/lib/lunar/moonbase diff
```

Save local changes:

```bash
git -C /var/lib/lunar/moonbase diff   > /root/moonbase-local-changes.patch
```

## 30. Compare module revisions

```bash
git -C /var/lib/lunar/moonbase diff OLD..NEW --   section/module
```

History:

```bash
git -C /var/lib/lunar/moonbase log --   section/module
```

## 31. Plugin inspection

Find plugins:

```bash
find /var/lib/lunar   -type f   -name '*.plugin'   -print
```

Search Moonbase plugin installers:

```bash
grep -R -n -E 'plugin\.d|\.plugin'   /var/lib/lunar/moonbase
```

Syntax check:

```bash
bash -n /path/to/plugin.plugin
```

Return semantics:

```text
0
→ handled successfully; stop

1
→ handled unsuccessfully; stop

2
→ continue
```

## 32. Remove-hook inspection

```bash
for hook in PRE_REMOVE POST_REMOVE; do
  if [ -f "$MODULE_DIR/$hook" ]; then
    echo "=== $hook ==="
    sed -n '1,240p' "$MODULE_DIR/$hook"
  fi
done
```

## 33. Build-hook inspection

```bash
for hook in PRE_BUILD POST_BUILD POST_INSTALL; do
  if [ -f "$MODULE_DIR/$hook" ]; then
    echo "=== $hook ==="
    sed -n '1,240p' "$MODULE_DIR/$hook"
  fi
done
```

## 34. Optional-feature inspection

```bash
for file in DEPENDS CONFIGURE OPTIONS BUILD; do
  if [ -f "$MODULE_DIR/$file" ]; then
    echo "=== $file ==="
    sed -n '1,240p' "$MODULE_DIR/$file"
  fi
done
```

## 35. Configuration preservation checks

Inspect effective setting:

```bash
grep -n 'PRESERVE' /etc/lunar/config
```

Check current config checksum:

```bash
md5sum /etc/module.conf
```

Compare with saved MD5 log.

Expected with `PRESERVE=on`:

```text
unchanged /etc file
→ removed

modified /etc file
→ preserved
```

## 36. Cache backup

```bash
mkdir -p /root/lunar-cache-backup

cp -a /var/cache/lunar/module-version-*   /root/lunar-cache-backup/

sha256sum /root/lunar-cache-backup/module-version-*   > /root/lunar-cache-backup/SHA256SUMS
```

## 37. Evidence export from container

```bash
podman cp   lunar-dev:/root/lss-evidence/.   ~/LSS-Evidence/
```

Verify:

```bash
find ~/LSS-Evidence -type f | sort
```

## 38. Quick install verification

```bash
lvu installed module
grep '^module:' /var/state/lunar/packages
test -f /var/log/lunar/install/module-version
```

## 39. Quick rebuild verification

```bash
lvu installed module
grep '^module:' /var/state/lunar/packages
diff -u install.before install.after
```

## 40. Quick removal verification

```bash
grep '^module:' /var/state/lunar/packages || true

test -e /known/payload/path &&
  echo present ||
  echo missing
```

## 41. Quick dependency safety check

```bash
MODULE=name

lvu depends "$MODULE"
lvu leert "$MODULE"

grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends
```

## 42. Quick troubleshooting sequence

```text
1. Read console output.
2. Identify lifecycle phase.
3. Open compile log.
4. Find first meaningful error.
5. Inspect module BUILD and hooks.
6. Inspect toolchain environment.
7. Inspect dependency state.
8. Compare before and after.
9. Test runtime.
```

## 43. Quick recovery sequence

```text
preserve evidence
→ identify inconsistent layers
→ rebuild or reinstall through LSS
→ regenerate manifest and state
→ verify filesystem
→ verify runtime
```

## 44. Critical distinctions

```text
Moonbase
→ intended module behavior

installwatch
→ raw observed filesystem events

install manifest
→ final ownership

MD5 log
→ installed checksums

packages
→ installation and policy state

depends
→ relationship state

cache
→ reusable installation payload

activity log
→ historical transitions

runtime
→ actual behavior
```

## 45. Operational rules

```text
inspect before changing
preserve logs before removal
rebuild before removing an optional dependency
do not edit state databases casually
do not treat leaf as automatically safe
do not treat cache as complete truth
do not treat same version as same build
verify runtime after successful installation
```

## 46. Minimal administrator checklist

Before install or rebuild:

```text
inspect module
inspect dependencies
record current state
verify source and toolchain
```

Before removal:

```text
inspect reverse dependents
inspect hooks
save manifest and MD5 log
check /etc state
```

After any change:

```text
verify packages
verify depends
verify manifest
verify filesystem
verify runtime
```

## 47. User Guide chapter map

```text
1. Installing, Rebuilding, and Removing Modules
2. Inspecting Modules and System State
3. Understanding Moonbase Modules
4. Managing Dependencies and Optional Features
5. Configuration, Options, and Reconfiguration
6. Logs, Manifests, and Troubleshooting
7. Caches, Archives, and Recovery
8. Updating the System Safely
9. Working with Moonbase
10. Advanced Module Inspection and Debugging
11. Module Removal and Configuration Preservation
12. Plugins and Extensibility
13. Policy States: Held, Exiled, and Enforced Modules
14. Building and Testing a Local Module
15. System Recovery and State Repair
16. Reference Commands and File Locations
```

## 48. Final summary

LSS administration follows one consistent discipline:

```text
understand intent
→ inspect state
→ preserve evidence
→ perform one controlled operation
→ verify ownership
→ verify persistent state
→ verify runtime
```

This reference is the compact operational map for that discipline.
