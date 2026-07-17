# Logs, Manifests, and Troubleshooting

**Project:** Lunar Script System User Guide  
**Status:** Draft 0.1  
**Date:** 2026-07-17  
**Audience:** Lunar Linux users, maintainers, and system administrators  
**Evidence basis:** LSS Evidence Foundation 0.1 — Checkpoints 1–6

## 1. Overview

LSS preserves several kinds of operational evidence.

The most important are:

```text
compile log
install manifest
MD5 log
activity log
package-state database
dependency-state database
installation cache
console output
```

Each answers a different question.

A reliable troubleshooting process starts by identifying the exact question before choosing the evidence source.

## 2. The main evidence locations

```text
/var/log/lunar/compile
/var/log/lunar/install
/var/log/lunar/md5sum
/var/log/lunar/activity
/var/state/lunar/packages
/var/state/lunar/depends
/var/cache/lunar
```

Typical per-module files:

```text
/var/log/lunar/compile/module-version.xz
/var/log/lunar/install/module-version
/var/log/lunar/md5sum/module-version
```

These files are related, but not interchangeable.

## 3. Compile log

The compile log records build-time output.

It may contain:

- configure output;
- compiler commands;
- warnings;
- errors;
- linker commands;
- install output;
- patch failures;
- lifecycle-stage messages;
- toolchain selection.

Read it with:

```bash
xzless /var/log/lunar/compile/module-version.xz
```

Search for common failures:

```bash
xzgrep -n -i   -E 'error|failed|undefined|not found|cannot|fatal'   /var/log/lunar/compile/module-version.xz
```

Do not stop at the final error line.

Look for the first meaningful failure.

## 4. Lifecycle error boundaries

LSS associates failures with lifecycle stages such as:

```text
PRE_BUILD
BUILD
POST_BUILD
POST_INSTALL
```

These labels narrow the search.

### `PRE_BUILD`

Likely causes:

- failed patch;
- missing helper;
- bad source-tree assumption;
- environment preparation failure.

### `BUILD`

Likely causes:

- compiler error;
- configure failure;
- linker error;
- make failure;
- installation command failure.

### `POST_BUILD`

Likely causes:

- missing installed file;
- failed cleanup;
- failed symlink creation;
- permission adjustment failure.

### `POST_INSTALL`

Likely causes:

- cache/index refresh;
- service integration;
- registration step;
- post-transaction command.

A `POST_INSTALL` failure may occur after the core payload is already installed.

## 5. Console output

Console output is the first visible summary of the operation.

Preserve it when the operation matters:

```bash
lin module 2>&1 | tee install-console.log
```

For rebuild:

```bash
lin -c module 2>&1 | tee rebuild-console.log
```

For removal:

```bash
lrm module 2>&1 | tee remove-console.log
```

Console output is useful for chronology.

It is usually less complete than the compile log.

## 6. Install manifest

The install manifest records final owned paths:

```text
/var/log/lunar/install/module-version
```

Inspect:

```bash
cat /var/log/lunar/install/module-version
```

Count entries:

```bash
wc -l /var/log/lunar/install/module-version
```

Use it to answer:

```text
which paths does LSS attribute to this module?
```

Do not use it to answer:

```text
which files were ever touched during BUILD?
```

## 7. Raw events versus final ownership

Installwatch observes filesystem operations during installation.

The final manifest is created after interpretation and filtering.

A path may be:

```text
created
→ modified
→ deleted
```

during the same build.

Such a path may appear in raw installwatch evidence but not in the final manifest.

Validated example:

```text
/usr/lib/libxxhash.a
```

was installed and later unlinked.

It was absent from the final manifest.

## 8. Manifest classification

Classify each entry:

```bash
MANIFEST=/var/log/lunar/install/module-version

while IFS= read -r path; do
  if [ -L "$path" ]; then
    printf 'link\t%s\t%s\n' "$path" "$(readlink "$path")"
  elif [ -f "$path" ]; then
    printf 'file\t%s\n' "$path"
  elif [ -d "$path" ]; then
    printf 'directory\t%s\n' "$path"
  else
    printf 'missing\t%s\n' "$path"
  fi
done < "$MANIFEST"
```

This helps detect stale or damaged ownership state.

## 9. Missing manifest paths

A missing path may indicate:

- manual deletion;
- later module replacement;
- filesystem damage;
- stale manifest;
- a post-install action;
- another package taking ownership;
- incomplete cleanup.

Do not immediately recreate or delete state.

First determine whether the path is:

- expected;
- shared;
- replaced;
- protected;
- excluded;
- recreated elsewhere.

## 10. Comparing manifests

Before rebuild:

```bash
cp -a /var/log/lunar/install/module-version   install.before
```

After rebuild:

```bash
cp -a /var/log/lunar/install/module-version   install.after
```

Compare:

```bash
diff -u install.before install.after
```

No output means byte-for-byte identity.

A changed manifest may show:

- new feature;
- removed feature;
- different documentation;
- new plugin;
- changed path layout;
- packaging regression.

## 11. MD5 log

The MD5 log stores installed checksums:

```text
/var/log/lunar/md5sum/module-version
```

Inspect:

```bash
cat /var/log/lunar/md5sum/module-version
```

It is especially important for `/etc`.

LSS uses it to distinguish:

```text
unchanged configuration
modified configuration
```

## 12. Verifying a configuration file

Current checksum:

```bash
md5sum /etc/example.conf
```

Stored checksum:

```bash
grep '/etc/example.conf'   /var/log/lunar/md5sum/module-version
```

Interpretation:

```text
same checksum
→ unchanged

different checksum
→ locally modified or otherwise altered
```

With `PRESERVE=on`, a modified configuration is preserved during normal removal.

## 13. Activity log

The activity log is:

```text
/var/log/lunar/activity
```

Search by module:

```bash
grep 'module' /var/log/lunar/activity
```

Follow live:

```bash
tail -f /var/log/lunar/activity
```

It provides broader historical context:

- install;
- rebuild;
- removal;
- failure;
- version transition;
- lifecycle result.

Unlike module-specific logs, it usually survives `lrm`.

## 14. Package-state database

The package-state database is:

```text
/var/state/lunar/packages
```

Inspect one module:

```bash
grep '^module:' /var/state/lunar/packages
```

Record format:

```text
module:date:state-list:version:size
```

Use it to answer:

```text
is the module recorded as installed?
which version?
which policy state?
what size is recorded?
```

It is not a raw filesystem inventory.

## 15. Dependency-state database

The dependency-state database is:

```text
/var/state/lunar/depends
```

Inspect both directions:

```bash
MODULE=name

grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends
```

Use it to answer:

```text
which relationships are stored?
which are active?
which type?
```

It does not prove runtime functionality.

## 16. Cache evidence

Caches are stored under:

```text
/var/cache/lunar
```

Inspect:

```bash
ls -l /var/cache/lunar/module-*
```

A cache can explain why a module was restored without a normal compile path.

When troubleshooting unexpected results, determine whether LSS:

```text
compiled from source
or
resurrected from cache
```

## 17. A complete evidence capture

Before a risky operation:

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

After the operation, capture the same state again.

## 18. Diagnosing a failed build

A practical sequence:

```text
1. Read console output.
2. Identify lifecycle phase.
3. Open compile log.
4. Find first real error.
5. Inspect BUILD and hooks.
6. Inspect toolchain variables.
7. Inspect dependencies.
8. Verify source and checksum.
9. Reproduce only after forming a hypothesis.
```

Avoid blind repeated rebuilds.

They can overwrite useful evidence.

## 19. Compiler errors

Typical signs:

```text
undeclared identifier
unknown option
unsupported flag
missing header
type mismatch
syntax error
```

Check:

```bash
echo "$CC"
echo "$CXX"
echo "$CFLAGS"
echo "$CXXFLAGS"
```

Search module:

```bash
grep -R -n -E 'CC=|CXX=|CFLAGS|CXXFLAGS'   "$MODULE_DIR"
```

A GCC/Clang mismatch can produce misleading failures.

## 20. Linker errors

Typical signs:

```text
undefined reference
cannot find -l...
DSO missing from command line
symbol not found
```

Check:

```bash
echo "$LDFLAGS"
ldd /path/to/binary
readelf -d /path/to/binary | grep NEEDED
```

Inspect dependency state and pkg-config metadata.

## 21. Missing dependency

Typical signs:

```text
header not found
library not found
pkg-config package missing
command not found
```

Check:

```bash
grep '^module:' /var/state/lunar/depends
grep '^dependency:' /var/state/lunar/packages
command -v required-command
pkgconf --exists package
```

A stored dependency record may exist even when the dependency is physically missing.

## 22. Configure failure

Search:

```bash
xzgrep -n -E   'checking|not found|cannot|failed|error'   /var/log/lunar/compile/module-version.xz
```

Inspect build flags.

A feature may be auto-detected differently after a dependency or toolchain change.

## 23. Patch failure

Typical signs:

```text
Hunk FAILED
Reversed patch
file not found
malformed patch
```

Inspect:

- `PRE_BUILD`;
- patch files;
- source version;
- source-tree layout.

A patch may be stale after a version bump.

## 24. Installation failure

Typical signs:

```text
permission denied
no such file or directory
destination missing
install target failed
```

Check whether `prepare_install` was called before installation.

Inspect:

- path rewrites;
- DESTDIR use;
- permissions;
- filesystem availability;
- installwatch environment.

## 25. Post-build failure

A POST_BUILD failure may occur after files were installed.

Inspect:

- final manifest presence;
- raw console chronology;
- installed files;
- POST_BUILD script;
- package-state record.

Do not assume the system is unchanged.

## 26. Post-install failure

A POST_INSTALL failure may leave the module installed but incompletely integrated.

Check:

```bash
grep '^module:' /var/state/lunar/packages
cat /var/log/lunar/install/module-version
```

Then inspect:

- service state;
- cache refresh;
- registration step;
- plugin discovery;
- printed notices.

## 27. Diagnosing unexpected removal

If a file remains:

```text
modified /etc file
PROTECTED path
EXCLUDED path
shared ownership
recreated by service
not present in old manifest
```

Check:

```bash
grep -Fx '/path' saved-install-log
grep -R -F -x '/path' /var/log/lunar/install
```

If the file disappeared unexpectedly, inspect:

- old manifest;
- protection policy;
- remove hooks;
- shared ownership;
- preserved evidence.

## 28. Diagnosing preserved configuration

If `/etc/file` remains after `lrm`:

```bash
md5sum /etc/file
grep '/etc/file' saved-md5sum-log
```

A checksum difference with `PRESERVE=on` explains preservation.

The file is then local orphaned state.

## 29. Diagnosing stale package state

Possible symptoms:

```text
package record exists, manifest missing
manifest exists, package record missing
recorded version differs from file names
recorded size is implausible
```

Capture evidence first.

Then compare:

```bash
grep '^module:' /var/state/lunar/packages
ls -l /var/log/lunar/install/module-*
ls -l /var/log/lunar/md5sum/module-*
ls -l /var/log/lunar/compile/module-*
```

Do not hand-edit databases before understanding the inconsistency.

## 30. Diagnosing stale dependency state

Possible symptoms:

```text
dependency record references missing module
reverse tree disagrees with actual installed state
disabled choice appears active
old provider remains selected
```

Check:

```bash
grep -E '^module:|^[^:]+:module:'   /var/state/lunar/depends

grep '^dependency:' /var/state/lunar/packages
```

Inspect current `DEPENDS`, `CONFIGURE`, and `OPTIONS`.

## 31. Reproducing installed size

Use only regular files:

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
SIZE=0

while IFS= read -r path; do
  if [ -f "$path" ]; then
    value=$(du -k -- "$path" | awk '{print $1}')
    SIZE=$((SIZE + value))
  fi
done < "/var/log/lunar/install/${MODULE}-${VERSION}"

echo "${SIZE}KB"
```

Do not run recursive `du` over manifest directories.

That counts unrelated content under `/usr`, `/usr/lib`, and other shared trees.

## 32. Raw installwatch capture

The raw installwatch file is temporary and normally destroyed.

Capturing it requires controlled instrumentation.

This is an advanced debugging technique.

Use it only in:

- a container;
- a chroot;
- a disposable test system;
- a backed-up environment.

The raw stream can reveal:

- transient files;
- unlink operations;
- renames;
- paths filtered from final ownership.

Restore the original LSS code after the experiment.

## 33. Comparing before and after state

Capture:

```bash
cp -a /var/state/lunar/packages packages.before
cp -a /var/state/lunar/depends depends.before
```

After operation:

```bash
cp -a /var/state/lunar/packages packages.after
cp -a /var/state/lunar/depends depends.after
```

Compare:

```bash
diff -u packages.before packages.after
diff -u depends.before depends.after
```

This provides exact persistent-state transitions.

## 34. Troubleshooting by question

### Did compilation fail?

```text
compile log
console output
BUILD and PRE_BUILD
toolchain environment
```

### Did installation ownership differ?

```text
install manifest
raw installwatch if captured
POST_BUILD
EXCLUDED and PROTECTED
```

### Did package state change?

```text
/var/state/lunar/packages
activity log
```

### Did dependency state change?

```text
/var/state/lunar/depends
DEPENDS
CONFIGURE
OPTIONS
```

### Did configuration survive removal?

```text
MD5 log
current checksum
PRESERVE setting
```

### Was cache used?

```text
console output
cache directory
activity history
```

## 35. Common diagnostic mistakes

### Mistake 1: reading only the last error line

The first meaningful failure is usually more useful.

### Mistake 2: treating console output as complete evidence

The compile log may contain more detail.

### Mistake 3: treating manifest as raw history

It records final ownership.

### Mistake 4: running recursive `du` on directory entries

This produces misleading totals.

### Mistake 5: removing a module before saving logs

`lrm` may destroy them.

### Mistake 6: editing state databases immediately

This destroys evidence and may worsen inconsistency.

### Mistake 7: rebuilding repeatedly without changing the hypothesis

This adds noise rather than knowledge.

## 36. A disciplined troubleshooting loop

```text
observe
→ preserve evidence
→ identify lifecycle phase
→ form one hypothesis
→ make one controlled change
→ rerun
→ compare
→ accept or reject hypothesis
```

This is faster and safer than random experimentation.

## 37. Minimal troubleshooting bundle

For support or review, preserve:

```text
console output
compile log
install manifest
MD5 log
package record
dependency records
module DETAILS
module BUILD
module hooks
relevant configuration
toolchain environment
```

Archive:

```bash
tar -cJf module-debug-bundle.tar.xz debug-directory/
```

Remove secrets before sharing.

## 38. Summary

LSS troubleshooting depends on separating several forms of evidence:

```text
compile log
→ what happened during build

install manifest
→ what LSS finally owns

MD5 log
→ installed checksums

packages
→ current module and policy state

depends
→ current relationship state

activity
→ historical transitions

cache
→ reusable installation artifact
```

The central rule is:

```text
preserve first
→ identify the exact layer
→ diagnose from the correct evidence
→ change one thing at a time
