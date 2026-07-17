# Inspecting Modules and System State

**Project:** Lunar Script System User Guide  
**Status:** Draft 0.1  
**Date:** 2026-07-17  
**Audience:** Lunar Linux users and system administrators  
**Evidence basis:** LSS Evidence Foundation 0.1 — Checkpoints 1–6

## 1. Overview

LSS keeps several complementary views of the system.

No single command or file tells the whole story.

The most useful inspection sources are:

```text
lvu
/var/state/lunar/packages
/var/state/lunar/depends
/var/log/lunar/install
/var/log/lunar/md5sum
/var/log/lunar/compile
/var/log/lunar/activity
/var/cache/lunar
Moonbase module files
```

Together they answer:

```text
what is installed?
which version?
which policy state?
what depends on it?
which files does it own?
which configuration files changed?
what happened during build?
what happened historically?
is a reusable cache available?
```

The correct operational habit is:

```text
inspect several views
→ compare them
→ interpret the result
```

## 2. `lvu` as the inspection interface

`lvu` is the main user-facing inspection tool.

It provides commands for:

- installed versions;
- Moonbase location;
- dependencies;
- reverse dependencies;
- module trees;
- module sections;
- sizes;
- module files;
- module history and metadata.

Useful starting commands:

```bash
lvu installed module
lvu where module
lvu depends module
lvu leert module
```

Example:

```bash
lvu installed xxhash
lvu where xxhash
lvu depends xxhash
lvu leert xxhash
```

These commands do not all read the same underlying state.

Their outputs should be interpreted according to the question being asked.

## 3. Checking whether a module is installed

Use:

```bash
lvu installed module
```

Example:

```bash
lvu installed foremost
```

Possible output:

```text
1.5.7
```

No useful output usually means the module is not currently installed.

For authoritative state, also inspect:

```bash
grep '^foremost:' /var/state/lunar/packages
```

Example record:

```text
foremost:20260717:installed:1.5.7:116KB
```

Use both when debugging inconsistent state.

## 4. Understanding `/var/state/lunar/packages`

The package-state database is:

```text
/var/state/lunar/packages
```

Observed record format:

```text
module:date:state-list:version:size
```

Example:

```text
xxhash:20260717:installed:0.8.3:520KB
```

Fields:

```text
module
→ module name

date
→ state or installation date in YYYYMMDD form

state-list
→ installed and policy states

version
→ installed version

size
→ recorded installed size
```

Known state terms include:

```text
installed
held
exiled
enforced
```

A record may represent both current installation and policy.

Therefore this file is not merely an installed-package list.

## 5. Querying package state safely

Exact module lookup:

```bash
grep '^module:' /var/state/lunar/packages
```

Example:

```bash
grep '^xxhash:' /var/state/lunar/packages
```

List installed modules:

```bash
awk -F: '$3 ~ /(^|\+)installed(\+|$)/ {print $1}'   /var/state/lunar/packages | sort
```

Show module, version, and size:

```bash
awk -F: '$3 ~ /(^|\+)installed(\+|$)/ {
  printf "%-30s %-20s %s\n", $1, $4, $5
}' /var/state/lunar/packages
```

Show held modules:

```bash
awk -F: '$3 ~ /(^|\+)held(\+|$)/ {print}'   /var/state/lunar/packages
```

Do not edit this database manually during normal administration.

Use LSS commands so related state remains consistent.

## 6. Date semantics

The date field is refreshed during installation or rebuild.

Example:

```text
before rebuild:
xxhash:20260605:installed:0.8.3:31KB

after rebuild:
xxhash:20260717:installed:0.8.3:520KB
```

The version remained unchanged, but the date and size changed.

Therefore:

```text
same version
≠ unchanged package-state record
```

A rebuild can refresh metadata without changing the upstream version.

## 7. Size semantics

The recorded size is derived from regular files in the final install manifest.

It may include:

- executables;
- libraries;
- headers;
- documentation;
- manual pages;
- Lunar install log;
- Lunar compile log;
- Lunar MD5 log.

It does not directly count directories and symbolic links.

To reproduce the current calculation:

```bash
MODULE=xxhash
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

Use `du -k` explicitly.

Interactive aliases such as:

```text
du='du -h'
```

can otherwise produce values like `100K`, which are not valid shell integers.

## 8. Locating a module in Moonbase

Use:

```bash
lvu where module
```

Example:

```bash
lvu where xxhash
```

Possible output:

```text
core/libs
```

Construct the full path:

```bash
MODULE=xxhash
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

printf '%s\n' "$MODULE_DIR"
```

This path contains the module definition used by LSS.

## 9. Inspecting module files

List module files:

```bash
find "$MODULE_DIR" -maxdepth 2 -type f -print | sort
```

Common files:

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
```

Read the main files:

```bash
sed -n '1,240p' "$MODULE_DIR/DETAILS"
sed -n '1,240p' "$MODULE_DIR/BUILD"
```

Inspect dependency declarations:

```bash
test -f "$MODULE_DIR/DEPENDS" &&
  sed -n '1,240p' "$MODULE_DIR/DEPENDS"
```

Inspect hooks:

```bash
for hook in PRE_BUILD POST_BUILD POST_INSTALL PRE_REMOVE POST_REMOVE; do
  if [ -f "$MODULE_DIR/$hook" ]; then
    echo "=== $hook ==="
    sed -n '1,240p' "$MODULE_DIR/$hook"
  fi
done
```

## 10. Why module files matter

The persistent state tells you what LSS believes now.

The module files tell you what LSS intends to do next.

For example:

```text
packages
→ current installed state

depends
→ current dependency relationships

BUILD
→ future build and installation actions

PRE_REMOVE / POST_REMOVE
→ future removal side effects
```

A safe decision often requires both current state and module intent.

## 11. Understanding `/var/state/lunar/depends`

The dependency-state database is:

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

Interpretation:

```text
module
→ ccache

dependency
→ xxhash

status
→ on

type
→ required

field5 and field6
→ additional declaration parameters
```

The final two fields may be empty.

Always preserve all six fields when analyzing or exporting records.

## 12. Inspecting direct dependency records

Show dependencies declared by a module:

```bash
grep '^module:' /var/state/lunar/depends
```

Example:

```bash
grep '^ccache:' /var/state/lunar/depends
```

Show modules that depend on a target:

```bash
grep -E '^[^:]+:target:' /var/state/lunar/depends
```

Example:

```bash
grep -E '^[^:]+:xxhash:' /var/state/lunar/depends
```

Show both directions:

```bash
MODULE=xxhash

grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends
```

This is useful before rebuild or removal.

## 13. `lvu depends`

Use:

```bash
lvu depends module
```

This shows installed modules that depend on the target.

Example:

```bash
lvu depends xxhash
```

Observed output included:

```text
ccache
```

This is a direct operational warning:

```text
removing xxhash
→ may affect ccache
```

Depending on the system, other modules may also appear.

## 14. `lvu leert`

Use:

```bash
lvu leert module
```

This shows the reverse dependency tree.

Example structure:

```text
xxhash: ccache dav1d kitty python-xxhash ...
^----ccache: ...
```

The tree helps identify indirect impact.

A leaf candidate may show only:

```text
foremost:
```

That means the tree has a root but no dependent branches.

Do not treat a dependency tree as the only safety test.

External scripts and operational use may not appear in LSS dependency state.

## 15. Leaf modules

Leaf modules are installed modules without recorded reverse dependents.

List them:

```bash
lvu leafs | sort
```

Leaf status is useful for:

- cleanup review;
- low-risk experiments;
- identifying optional software;
- removal candidates.

Leaf does not mean:

```text
unimportant
safe under every circumstance
unused outside LSS
```

Examples of technically leaf modules may still include:

- boot tools;
- network tools;
- package-management tools;
- administrative commands;
- shared libraries used outside declared Moonbase relationships.

Inspect role and module files before removal.

## 16. The install manifest

Per-module ownership manifests are stored under:

```text
/var/log/lunar/install
```

File name:

```text
module-version
```

Example:

```text
/var/log/lunar/install/foremost-1.5.7
```

Inspect:

```bash
cat /var/log/lunar/install/foremost-1.5.7
```

Count entries:

```bash
wc -l /var/log/lunar/install/foremost-1.5.7
```

The manifest describes final paths attributed to the module.

## 17. Classifying manifest entries

Use:

```bash
MANIFEST=/var/log/lunar/install/foremost-1.5.7

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

This distinguishes:

```text
regular files
symbolic links
directories
missing paths
```

A missing manifest path can indicate:

- manual deletion;
- later replacement;
- incomplete state;
- module behavior after installation;
- filesystem damage.

## 18. Ownership versus raw build activity

The install manifest is not a raw event log.

Installwatch may observe:

```text
create
chmod
rename
unlink
```

for a path during BUILD.

If the path no longer exists when final ownership is created, it may not appear in the manifest.

Validated example:

```text
/usr/lib/libxxhash.a
```

was installed and later unlinked during the same build.

It appeared in raw installwatch evidence but not in the final manifest.

Therefore:

```text
raw build activity
≠ final ownership
```

## 19. Finding possible shared ownership

Search all manifests for a path:

```bash
grep -R -F -x '/usr/bin/example'   /var/log/lunar/install 2>/dev/null
```

Example:

```bash
grep -R -F -x '/usr/lib/libexample.so'   /var/log/lunar/install 2>/dev/null
```

Multiple matches indicate potential overlapping ownership.

Overlapping ownership requires careful interpretation:

- one module may replace another;
- a path may be intentionally shared;
- state may be stale;
- manifests may reflect different installation eras.

Do not manually delete such paths without understanding the overlap.

## 20. MD5 logs

Per-module checksum logs are stored under:

```text
/var/log/lunar/md5sum
```

Example:

```text
/var/log/lunar/md5sum/foremost-1.5.7
```

Inspect:

```bash
cat /var/log/lunar/md5sum/foremost-1.5.7
```

These logs help LSS decide whether configuration files under `/etc` were locally modified.

They are also useful for:

- integrity checks;
- troubleshooting;
- comparing current state with installed state.

## 21. Checking a configuration file

Current checksum:

```bash
md5sum /etc/foremost.conf
```

Search the module's MD5 record:

```bash
grep '/etc/foremost.conf'   /var/log/lunar/md5sum/foremost-1.5.7
```

Interpretation:

```text
current checksum matches installed checksum
→ unchanged

current checksum differs
→ locally modified or otherwise changed
```

With `PRESERVE=on`, a modified `/etc` file is preserved during normal removal.

## 22. Compile logs

Compile logs are stored under:

```text
/var/log/lunar/compile
```

Typical file:

```text
module-version.xz
```

Read with:

```bash
xzless /var/log/lunar/compile/module-version.xz
```

Search errors:

```bash
xzgrep -n -i   -E 'error|failed|undefined|not found'   /var/log/lunar/compile/module-version.xz
```

Compile logs may contain:

- compiler commands;
- configure output;
- linker errors;
- patch failures;
- install output;
- lifecycle stage messages.

They are often the first source to inspect after a failed `lin`.

## 23. Activity log

The broader system activity log is:

```text
/var/log/lunar/activity
```

Search by module:

```bash
grep 'foremost' /var/log/lunar/activity
```

Follow live activity:

```bash
tail -f /var/log/lunar/activity
```

The activity log provides historical transitions such as:

- installation;
- rebuild;
- removal;
- failure;
- version changes.

Unlike module-specific logs, it normally remains after the module is removed.

## 24. Cache inspection

Caches are stored under:

```text
/var/cache/lunar
```

Find a module cache:

```bash
ls -l /var/cache/lunar/module-*
```

Example:

```bash
ls -l /var/cache/lunar/xxhash-*
```

A cache file may include:

```text
module
version
target triplet
compression suffix
```

The cache is distinct from the install manifest.

The manifest describes ownership.

The cache contains reusable installation content.

## 25. Comparing state before and after an operation

Create a small evidence directory:

```bash
MODULE=foremost
BASE=/root/lss-state-check/$MODULE

mkdir -p "$BASE/before" "$BASE/after"
```

Before:

```bash
cp -a /var/state/lunar/packages "$BASE/before/packages"
cp -a /var/state/lunar/depends "$BASE/before/depends"

grep "^${MODULE}:" /var/state/lunar/packages   > "$BASE/before/package-record" || true

grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends   > "$BASE/before/depends-records" || true
```

After:

```bash
cp -a /var/state/lunar/packages "$BASE/after/packages"
cp -a /var/state/lunar/depends "$BASE/after/depends"
```

Compare:

```bash
diff -u "$BASE/before/packages" "$BASE/after/packages"
diff -u "$BASE/before/depends" "$BASE/after/depends"
```

This reveals exact persistent-state transitions.

## 26. Comparing manifests

Before rebuild:

```bash
cp -a "/var/log/lunar/install/${MODULE}-${VERSION}"   "$BASE/install.before"
```

After rebuild:

```bash
cp -a "/var/log/lunar/install/${MODULE}-${VERSION}"   "$BASE/install.after"
```

Compare:

```bash
diff -u "$BASE/install.before" "$BASE/install.after"
```

No output means the manifests are byte-for-byte identical.

This does not necessarily mean package metadata is unchanged.

## 27. Inspecting filesystem state from a manifest

Capture:

```bash
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
done < "$MANIFEST" > filesystem-state.txt
```

Repeat after an operation and compare:

```bash
diff -u filesystem-state.before filesystem-state.after
```

This is especially useful for removal testing.

## 28. Detecting stale metadata

Possible signs:

```text
recorded size differs greatly from current calculation
manifest contains missing files
installed version differs from module metadata
package record exists but manifest is missing
manifest exists but package record is absent
dependency record references a missing module
```

Example checks:

```bash
MODULE=xxhash
VERSION=$(lvu installed "$MODULE")

test -f "/var/log/lunar/install/${MODULE}-${VERSION}" ||
  echo "missing install manifest"

grep "^${MODULE}:" /var/state/lunar/packages ||
  echo "missing package record"
```

Do not repair persistent state by hand before preserving evidence.

First determine how the inconsistency arose.

## 29. Cross-checking installed and Moonbase versions

Installed version:

```bash
lvu installed module
```

Moonbase version:

```bash
lvu version module
```

Example:

```bash
printf 'installed: %s\n' "$(lvu installed xxhash)"
printf 'moonbase:  %s\n' "$(lvu version xxhash)"
```

Possible interpretations:

```text
same
→ current according to active Moonbase

different
→ upgrade available, local divergence, or stale Moonbase
```

Use the active Moonbase as the reference, not an unrelated repository copy.

## 30. Inspecting sections

Show a module's section:

```bash
lvu where module
```

List a section:

```bash
lvu section section-name
```

Use `lvu where` for a module name.

Do not use:

```bash
lvu section module-name
```

unless the module name is also a valid section name.

A section error does not imply the module is missing.

## 31. Inspecting multiple modules

For a compact installed-state report:

```bash
for module in xxhash foremost ccache; do
  printf '\n=== %s ===\n' "$module"
  printf 'installed: %s\n' "$(lvu installed "$module" 2>/dev/null)"
  printf 'section:   %s\n' "$(lvu where "$module" 2>/dev/null)"
  grep "^${module}:" /var/state/lunar/packages || true
  grep -E "^${module}:|^[^:]+:${module}:"     /var/state/lunar/depends || true
done
```

This is useful before a toolchain or dependency transition.

## 32. A module inspection template

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

echo '=== PACKAGE STATE ==='
grep "^${MODULE}:" /var/state/lunar/packages || true

echo
echo '=== INSTALLED VERSION ==='
printf '%s\n' "$VERSION"

echo
echo '=== MOONBASE LOCATION ==='
printf '%s\n' "$MODULE_DIR"

echo
echo '=== REVERSE DEPENDENTS ==='
lvu depends "$MODULE"
lvu leert "$MODULE"

echo
echo '=== DEPENDENCY RECORDS ==='
grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends || true

echo
echo '=== MODULE FILES ==='
find "$MODULE_DIR" -maxdepth 2 -type f -print | sort

echo
echo '=== INSTALL MANIFEST ==='
if [ -n "$VERSION" ] &&
   [ -f "/var/log/lunar/install/${MODULE}-${VERSION}" ]; then
  cat "/var/log/lunar/install/${MODULE}-${VERSION}"
fi
```

This template provides a reliable first-pass inspection.

## 33. Interpreting absence

No output can mean different things.

Examples:

```text
lvu depends module
→ no reverse dependents

grep packages
→ no current package record

grep depends
→ no matching dependency records

missing manifest
→ uninstalled, stale state, or damaged logs

missing module directory
→ inactive Moonbase, removed module, or wrong section
```

Always identify which source produced no output.

Do not collapse all absence into “not installed.”

## 34. Source of truth by question

Use the right source for the right question.

```text
Is the module recorded as installed?
→ /var/state/lunar/packages

Which version is installed?
→ lvu installed + packages record

Where is the module definition?
→ lvu where

What does the module declare?
→ Moonbase files

What currently depends on it?
→ lvu depends, lvu leert, depends database

Which paths does it own?
→ install manifest

Was an /etc file modified?
→ current checksum + MD5 log

Why did compilation fail?
→ compile log

What happened historically?
→ activity log

Can installation be resurrected?
→ cache directory
```

## 35. Common mistakes

### Mistake 1: treating `packages` as only an installed list

It also contains policy state.

### Mistake 2: treating the install manifest as raw build history

It describes final ownership.

### Mistake 3: using `du` on manifest directories

This recursively counts unrelated filesystem content.

### Mistake 4: trusting leaf status alone

Operational dependencies may exist outside LSS records.

### Mistake 5: removing before copying logs

`lrm` may delete the evidence.

### Mistake 6: assuming no command output means failure

Some inspection commands legitimately return nothing.

### Mistake 7: editing state files manually

This may desynchronize LSS state.

## 36. Minimal daily inspection set

For routine administration:

```bash
lvu installed module
lvu where module
lvu depends module
grep '^module:' /var/state/lunar/packages
cat /var/log/lunar/install/module-version
```

For risky changes, add:

```bash
lvu leert module
grep dependency records
inspect BUILD and hooks
copy install and MD5 logs
snapshot packages and depends
```

## 37. Summary

LSS exposes system truth through several coordinated artifacts.

The core inspection model is:

```text
Moonbase
→ intended module behavior

packages
→ current installation and policy state

depends
→ current relationship state

install manifest
→ final owned paths

MD5 log
→ installed checksums

compile log
→ build evidence

activity log
→ historical transitions

cache
→ reusable installation artifact
```

Reliable administration comes from comparing these views rather than trusting one in isolation.

The practical rule is:

```text
ask one precise question
→ choose the correct state source
→ cross-check when the change is risky
