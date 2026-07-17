# Building and Testing a Local Module

**Project:** Lunar Script System User Guide  
**Status:** Draft 0.1  
**Date:** 2026-07-17  
**Audience:** Lunar Linux users, maintainers, and module authors  
**Evidence basis:** LSS Evidence Foundation 0.1 — Checkpoints 1–6

## 1. Overview

A local Moonbase module is useful for:

- private software;
- experimental updates;
- unreleased patches;
- hardware-specific tools;
- local services;
- testing packaging changes;
- preparing upstream contributions.

A safe local-module workflow is:

```text
create
→ inspect
→ build in isolation
→ verify ownership
→ test runtime
→ remove
→ verify cleanup
→ reinstall
```

Do not begin with a critical package.

## 2. Use an isolated environment

Preferred environments:

- rootless Podman container;
- chroot;
- disposable VM;
- test machine;
- snapshot-capable system.

The test environment should allow:

```text
root access
Moonbase access
normal lin/lrm/lvu use
evidence export
easy rollback
```

## 3. Choose a simple first module

A good first module:

- builds quickly;
- has few or no dependencies;
- installs one small executable;
- has no service;
- has no plugin;
- has no complex `/etc` state;
- has no reverse dependents.

Avoid:

```text
glibc
gcc
bash
lunar
kernel
init system
network base
shared toolchain components
```

## 4. Local section

Keep local modules separate from upstream-managed sections when possible.

Example layout:

```text
/var/lib/lunar/moonbase/local/utils/hello-lunar
```

This provides:

- clear provenance;
- easier backup;
- fewer merge conflicts;
- safer Moonbase updates.

Follow the conventions of the active Moonbase.

## 5. Minimal module structure

A minimal module directory:

```text
hello-lunar/
├── DETAILS
└── BUILD
```

More complex modules may add:

```text
DEPENDS
CONFIGURE
OPTIONS
PRE_BUILD
POST_BUILD
POST_INSTALL
PRE_REMOVE
POST_REMOVE
patches
auxiliary files
```

Start with the minimum.

## 6. `DETAILS`

A minimal conceptual `DETAILS` contains:

```bash
MODULE=hello-lunar
VERSION=1.0
SOURCE=$MODULE-$VERSION.tar.gz
SOURCE_URL=https://example.org/releases/
SOURCE_VFY=sha256:...
WEB_SITE=https://example.org/
ENTERED=20260717
UPDATED=20260717
SHORT="Small example program"

cat << EOF
A small example program used to demonstrate a local Lunar module.
EOF
```

Use the actual syntax and verification conventions accepted by the active Moonbase.

## 7. Source verification

Never omit source verification casually.

Verify:

```text
source file name
version
download URL
checksum or signature
upstream identity
```

Calculate SHA-256:

```bash
sha256sum hello-lunar-1.0.tar.gz
```

Record the result according to Moonbase conventions.

## 8. `BUILD`

For a conventional project:

```bash
./configure --prefix=/usr &&
make &&
prepare_install &&
make install
```

For a simple Makefile project:

```bash
make &&
prepare_install &&
make install PREFIX=/usr
```

The critical boundary is:

```text
prepare_install
```

Call it before system installation.

## 9. Why `prepare_install` matters

Before `prepare_install`:

```text
build tree activity
```

After `prepare_install`:

```text
filesystem installation observed by installwatch
```

Installing to `/usr` before this boundary may produce incomplete ownership tracking.

## 10. Using default helpers

When appropriate:

```bash
default_build
```

or:

```bash
default_make
```

These reduce boilerplate.

Still inspect their behavior in the current LSS version.

Do not assume a helper matches every upstream build system.

## 11. Compiler flags

Use the Lunar build environment rather than hardcoding arbitrary optimization.

A module may adjust:

```bash
export CFLAGS+=" -fcommon"
```

only when a compatibility reason exists.

Document unusual flags.

For toolchain-specific software, select compiler family explicitly when necessary.

## 12. Dependencies

Add `DEPENDS` only for real relationships.

Separate:

```text
required
optional
build-time
runtime
provider choice
```

Do not add dependencies simply because they happen to be installed on the development system.

Test in a clean environment to discover hidden assumptions.

## 13. Optional dependencies

Optional dependencies should map clearly to functionality.

The user should understand:

```text
enable dependency
→ gain feature

disable dependency
→ omit feature
```

Pass explicit positive and negative build flags when possible.

Avoid accidental upstream auto-detection.

## 14. Configuration and options

Use `CONFIGURE` or `OPTIONS` when the user must choose behavior.

Good configuration questions are:

- stable;
- meaningful;
- reproducible;
- directly connected to build output.

Avoid exposing every upstream switch without operational value.

## 15. Patches

Place patches in the module directory.

Apply them in `PRE_BUILD` or `BUILD` according to conventions.

A patch should have:

```text
purpose
upstream status
affected version
author/source
removal condition
```

Test that it still applies after a version update.

## 16. Shell quality

Moonbase modules are executable shell.

Use:

- clear `&&` chains;
- quoted variables where appropriate;
- explicit failure behavior;
- minimal global side effects;
- standard helpers;
- readable comments for unusual logic.

Validate syntax:

```bash
bash -n BUILD
bash -n PRE_BUILD
```

## 17. Pre-build inspection

Before running `lin`:

```bash
MODULE=hello-lunar
SECTION=local/utils
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

find "$MODULE_DIR" -maxdepth 2 -type f -print | sort

sed -n '1,240p' "$MODULE_DIR/DETAILS"
sed -n '1,240p' "$MODULE_DIR/BUILD"
```

Inspect all hooks and dependencies.

## 18. Record baseline

```bash
BASE=/root/lss-local-module/$MODULE

mkdir -p "$BASE/before" "$BASE/after-install" "$BASE/after-remove"

cp -a /var/state/lunar/packages   "$BASE/before/packages"

cp -a /var/state/lunar/depends   "$BASE/before/depends"

cp -a "$MODULE_DIR"   "$BASE/before/module-definition"

env | sort > "$BASE/before/environment"
```

Record Moonbase revision and local diff.

## 19. Build and install

Run:

```bash
lin "$MODULE" 2>&1 |
  tee "$BASE/install-console.log"
```

Do not run unrelated package operations during the test.

## 20. Verify package state

```bash
lvu installed "$MODULE"

grep "^${MODULE}:"   /var/state/lunar/packages
```

Expected record:

```text
module:date:installed:version:size
```

Verify dependency records:

```bash
grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends || true
```

## 21. Verify generated artifacts

```bash
VERSION=$(lvu installed "$MODULE")

ls -l   "/var/log/lunar/install/${MODULE}-${VERSION}"   "/var/log/lunar/md5sum/${MODULE}-${VERSION}"

ls -l   /var/log/lunar/compile/${MODULE}-${VERSION}.*   2>/dev/null || true
```

If `ARCHIVE=on`:

```bash
ls -l /var/cache/lunar/${MODULE}-${VERSION}-*
```

## 22. Inspect the manifest

```bash
cat "/var/log/lunar/install/${MODULE}-${VERSION}"
```

Check that it contains:

- intended files;
- intended symlinks;
- expected directories;
- no build-tree paths;
- no accidental host files;
- no unwanted static libraries;
- no unrelated configuration.

## 23. Classify paths

```bash
MANIFEST="/var/log/lunar/install/${MODULE}-${VERSION}"

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

A missing final path requires investigation.

## 24. Verify runtime

For a command:

```bash
command -v hello-lunar
hello-lunar --version
hello-lunar
```

For a library:

```bash
readelf -d /usr/lib/libhello.so
pkgconf --modversion hello
```

For a service:

- inspect service file;
- start in test environment;
- inspect process and logs;
- run a functional request.

## 25. Verify linking

```bash
ldd /usr/bin/hello-lunar
```

Look for:

```text
not found
```

Compare runtime linkage with declared dependencies.

## 26. Verify installed size

```bash
SIZE=0

while IFS= read -r path; do
  if [ -f "$path" ]; then
    value=$(du -k -- "$path" | awk '{print $1}')
    SIZE=$((SIZE + value))
  fi
done < "$MANIFEST"

echo "${SIZE}KB"
```

Compare with `/var/state/lunar/packages`.

## 27. Rebuild test

Run:

```bash
cp -a "$MANIFEST"   "$BASE/after-install/install.before-rebuild"

lin -c "$MODULE" 2>&1 |
  tee "$BASE/rebuild-console.log"

cp -a "$MANIFEST"   "$BASE/after-install/install.after-rebuild"
```

Compare:

```bash
diff -u   "$BASE/after-install/install.before-rebuild"   "$BASE/after-install/install.after-rebuild"
```

A simple deterministic module should normally produce stable ownership.

## 28. Removal test

Before removal, preserve logs:

```bash
cp -a "$MANIFEST"   "$BASE/after-install/install-log"

cp -a "/var/log/lunar/md5sum/${MODULE}-${VERSION}"   "$BASE/after-install/md5sum-log"
```

Run:

```bash
lrm "$MODULE" 2>&1 |
  tee "$BASE/remove-console.log"
```

## 29. Verify removal

Package record:

```bash
grep "^${MODULE}:" /var/state/lunar/packages ||
  echo "package record removed"
```

Verify payload paths from saved manifest.

Shared directories should remain.

Module-specific empty directories should disappear.

## 30. Reinstall test

```bash
lin "$MODULE" 2>&1 |
  tee "$BASE/reinstall-console.log"
```

Verify:

```bash
lvu installed "$MODULE"
grep "^${MODULE}:" /var/state/lunar/packages
```

Run the functional test again.

## 31. Configuration preservation test

Only for a non-critical test configuration.

Sequence:

```text
install
→ preserve original checksum
→ modify config
→ lrm
→ verify modified config remains
→ remove or archive orphan
→ reinstall
→ verify clean default
```

Do not perform this test on production configuration.

## 32. Dependency test

For a module with dependencies:

```text
install dependency
→ install module
→ inspect depends state
→ inspect reverse tree
→ disable optional feature
→ rebuild
→ inspect relationship change
```

Use one optional relationship at a time.

## 33. Cache test

When safe:

```text
build normally
→ verify cache
→ remove module
→ reinstall with cache available
→ observe whether cache is used
→ verify manifest and runtime
```

Then isolate cache and compare with fresh build.

## 34. Failure-path test

A mature module should also be tested under controlled failure.

Examples:

- unavailable source;
- wrong checksum;
- missing required dependency;
- intentionally failing patch;
- invalid compiler.

Verify:

- clear failure phase;
- useful compile log;
- no false package record;
- no unmanaged partial payload;
- recoverable next run.

## 35. Hidden dependency detection

Test in a minimal container.

A module that builds only on the developer's full system may depend on undeclared tools or libraries.

Search errors for:

```text
command not found
header missing
library missing
pkg-config missing
```

Add real dependencies rather than relying on ambient software.

## 36. Install path review

Look for unwanted paths:

```text
/usr/local
/opt unexpected
home directory
build directory
temporary directory
host-specific path
```

Normalize paths in the module where appropriate.

## 37. `/etc` review

Configuration should be:

- intentional;
- documented;
- tracked by MD5;
- compatible with `PRESERVE`;
- free of generated secrets;
- not overwritten blindly.

Do not ship machine-specific credentials.

## 38. Service integration

For a service module, include the files appropriate to the target init system.

Test:

```text
install
→ service definition present
→ start
→ stop
→ restart
→ remove
→ service definition gone
→ preserved config behavior correct
```

Do not start services unexpectedly during package build unless policy explicitly requires it.

## 39. Local module evidence bundle

Preserve:

```text
module definition
source checksum
Moonbase revision
environment
toolchain
install console
compile log
manifest
MD5 log
package record
dependency records
runtime test
remove console
before/after diffs
```

Archive:

```bash
tar -cJf "${MODULE}-test-evidence.tar.xz"   "$BASE"
```

## 40. Preparing for upstream contribution

Before proposing a module:

- remove local-only assumptions;
- use canonical source;
- verify license;
- verify checksums;
- minimize dependencies;
- make optional features explicit;
- test clean install;
- test rebuild;
- test removal;
- document unusual choices;
- follow Moonbase style.

## 41. Review questions

Ask:

```text
Does the module describe real dependencies?
Does prepare_install occur before system installation?
Is final ownership clean?
Are optional features explicit?
Does removal clean payload safely?
Are modified configs preserved correctly?
Can the module rebuild reproducibly?
Does it work in a minimal environment?
```

## 42. Common mistakes

### Mistake 1: testing only successful compile

Install, runtime, removal, and reinstall also matter.

### Mistake 2: building on a full host

Hidden dependencies remain undetected.

### Mistake 3: installing before `prepare_install`

Ownership may be incomplete.

### Mistake 4: hardcoding local paths

The module becomes non-portable.

### Mistake 5: enabling every optional feature

This weakens administrator choice.

### Mistake 6: omitting removal test

Manifest errors remain hidden.

### Mistake 7: editing state databases manually

Use normal LSS operations.

## 43. Minimal acceptance criteria

A local module is ready for wider testing when:

```text
source verifies
build succeeds
manifest is correct
package state is correct
dependencies are correct
runtime works
rebuild is stable
removal is clean
reinstall works
no local test modifications remain
```

## 44. Summary

Building a local module is a complete lifecycle exercise.

```text
definition
→ dependency resolution
→ build
→ observed installation
→ ownership
→ runtime
→ removal
→ restoration
```

The central rule is:

```text
test the module as LSS will live with it
→ not only as a source tree that compiles
