# Module Removal and Configuration Preservation

**Project:** Lunar Script System User Guide  
**Status:** Draft 0.1  
**Date:** 2026-07-17  
**Audience:** Lunar Linux users, maintainers, and system administrators  
**Evidence basis:** LSS Evidence Foundation 0.1 — Checkpoints 1–6

## 1. Overview

Removing a module is not simply deleting a binary.

LSS removal may involve:

- loading the module's ownership manifest;
- checking package and dependency state;
- running remove hooks;
- protecting special paths;
- preserving modified configuration;
- deleting owned payload;
- removing empty exclusive directories;
- deleting Lunar-generated module artifacts;
- updating persistent state.

The practical model is:

```text
ownership
→ policy
→ filesystem change
→ persistent state change
```

## 2. The removal command

Use:

```bash
lrm module
```

Example:

```bash
lrm foremost
```

Typical success output:

```text
Removed module: foremost
```

Run as root.

## 3. Before removal

Always inspect:

```bash
lvu installed module
lvu depends module
lvu leert module
lvu where module
```

Also inspect raw state:

```bash
grep '^module:' /var/state/lunar/packages

grep -E '^module:|^[^:]+:module:'   /var/state/lunar/depends
```

Do not remove a module only because it is small or appears as a leaf.

## 4. Reverse dependency review

Use:

```bash
lvu depends module
lvu leert module
```

A result such as:

```text
foremost:
```

shows the reverse-tree root with no dependent branches.

A result listing modules means removal may affect them.

Check direct records:

```bash
grep -E '^[^:]+:module:'   /var/state/lunar/depends
```

## 5. Leaf status

List leaf modules:

```bash
lvu leafs | sort
```

Leaf means:

```text
no recorded reverse dependents
```

It does not mean:

```text
unused
non-critical
safe in every deployment
```

A leaf may still be:

- a boot tool;
- a network utility;
- a package-management component;
- a directly used administrator command;
- required by local scripts.

## 6. Inspect remove hooks

Locate the module:

```bash
MODULE=name
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"
```

Inspect:

```bash
for hook in PRE_REMOVE POST_REMOVE; do
  if [ -f "$MODULE_DIR/$hook" ]; then
    echo "=== $hook ==="
    sed -n '1,240p' "$MODULE_DIR/$hook"
  fi
done
```

Hooks can affect state outside the install manifest.

## 7. Preserve removal evidence

Module-specific logs may be deleted during removal.

Before `lrm`:

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
BACKUP=/root/lss-remove/$MODULE

mkdir -p "$BACKUP"

cp -a "/var/log/lunar/install/${MODULE}-${VERSION}"   "$BACKUP/install-log"

cp -a "/var/log/lunar/md5sum/${MODULE}-${VERSION}"   "$BACKUP/md5sum-log"

cp -a "/var/log/lunar/compile/${MODULE}-${VERSION}".*   "$BACKUP/" 2>/dev/null || true

grep "^${MODULE}:" /var/state/lunar/packages   > "$BACKUP/package-record"

grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends   > "$BACKUP/depends-records" || true
```

## 8. Ownership manifest

The install manifest is:

```text
/var/log/lunar/install/module-version
```

It is the primary ownership source for removal.

Inspect:

```bash
cat /var/log/lunar/install/module-version
```

It may contain:

- files;
- symlinks;
- shared directories;
- module-specific directories;
- documentation;
- manual pages;
- Lunar logs.

## 9. Ownership-driven removal

The normal model is:

```text
manifest path
→ classify
→ apply policy
→ remove when permitted
```

LSS does not reconstruct ownership only from the current filesystem.

The saved manifest is essential.

## 10. Regular files

Owned regular payload is removed.

Validated examples:

```text
/usr/bin/foremost
/usr/share/doc/foremost/CHANGES
/usr/share/doc/foremost/README
/usr/share/man/man8/foremost.8.gz
```

After removal:

```bash
test -e /usr/bin/foremost &&
  echo present ||
  echo missing
```

## 11. Shared directories

Shared directories are not blindly removed.

Validated retained paths:

```text
/usr
/usr/share
/usr/share/doc
```

These remained because they still contained content used by the rest of the system.

## 12. Exclusive directories

A module-specific directory may be removed after its contents disappear.

Validated example:

```text
/usr/share/doc/foremost
```

The practical behavior is:

```text
directory becomes empty
+ safe to remove
→ remove

directory remains shared or non-empty
→ retain
```

## 13. Symlinks

Owned symbolic links are part of the manifest and may be removed.

Inspect before:

```bash
while IFS= read -r path; do
  [ -L "$path" ] &&
    printf '%s -> %s
' "$path" "$(readlink "$path")"
done < install-log
```

A symlink may point to a file owned by the same module or by another module.

Ownership of the link and ownership of the target are separate.

## 14. Lunar-generated artifacts

Validated module-owned artifacts include:

```text
/var/log/lunar/install/module-version
/var/log/lunar/compile/module-version.xz
/var/log/lunar/md5sum/module-version
```

These may be deleted during `lrm`.

Therefore:

```text
preserve evidence before removal
```

is mandatory for forensic or documentation work.

## 15. Package-state update

Normal removal deletes the module record from:

```text
/var/state/lunar/packages
```

Validated transition:

```diff
-foremost:20260605:installed:1.5.7:8KB
```

No replacement `removed` record was created.

## 16. Dependency-state update

If the module has no dependency records, `/var/state/lunar/depends` may remain unchanged.

This was validated with `foremost`.

Modules participating in relationships may cause state changes.

Capture the full before/after diff:

```bash
cp -a /var/state/lunar/depends depends.before

lrm module

cp -a /var/state/lunar/depends depends.after

diff -u depends.before depends.after
```

## 17. Configuration under `/etc`

LSS treats `/etc` specially.

The MD5 log is used to compare current configuration with the installed version.

The validated normal behavior with `PRESERVE=on` is:

```text
unchanged configuration
→ remove

locally modified configuration
→ preserve
```

## 18. Unchanged configuration

In the first `foremost` removal test:

```text
/etc/foremost.conf
```

was unchanged.

Result:

```text
removed
```

Therefore:

```text
PRESERVE=on
≠ preserve every file under /etc
```

## 19. Modified configuration

After reinstalling, a controlled marker was appended to:

```text
/etc/foremost.conf
```

Then `lrm foremost` was run again.

Result:

```text
configuration remained
payload disappeared
package record disappeared
module logs disappeared
```

This validates local-change preservation.

## 20. Orphaned configuration

After package removal, a preserved configuration is:

```text
present on disk
not owned by an installed module
not backed by the removed module's current manifest
```

This is local orphaned state.

It may still be valuable.

It should not be mistaken for an installed package-owned file.

## 21. Handling orphaned configuration

Possible actions:

### Keep

Useful when reinstalling later or preserving local policy.

### Back up

```bash
cp -a /etc/module.conf   /root/module.conf.saved
```

### Compare with a new default

Reinstall in a controlled environment and compare.

### Remove deliberately

Only after confirming it is no longer needed.

## 22. `PRESERVE`

Inspect:

```bash
grep -n 'PRESERVE' /etc/lunar/config
```

Observed default-style setting:

```text
PRESERVE=${PRESERVE:-on}
```

The effective value may be influenced by environment or local configuration.

## 23. `PRESERVE=off`

Source analysis indicates a different branch for modified `/etc` files when preservation is disabled.

Expected behavior involves archival and removal.

This branch was not yet runtime-validated in the evidence series.

Treat it as source-derived until tested.

## 24. `PROTECTED`

`PROTECTED` paths may remain in ownership state but be blocked from deletion.

Conceptually:

```text
owned or tracked
+ protected policy
→ do not delete
```

Use protection for paths whose deletion would be unsafe.

Do not confuse protection with preservation of local modification.

## 25. `EXCLUDED`

`EXCLUDED` paths are filtered out of final ownership.

Conceptually:

```text
path touched during install
+ excluded policy
→ not claimed in final manifest
```

Because removal is manifest-driven, excluded paths are not normally removed through module ownership.

## 26. `PROTECTED` versus `EXCLUDED`

```text
PROTECTED
→ ownership may remain visible
→ deletion blocked

EXCLUDED
→ ownership not claimed
→ removal does not target it through the module manifest
```

These solve different problems.

## 27. Configuration versus protected path

A modified `/etc` file is preserved because checksum policy says local state changed.

A protected path remains because deletion policy forbids removal.

The observable result may be similar:

```text
file remains
```

But the reason is different.

Inspect both checksum and policy.

## 28. Remove hooks and configuration

A hook may modify or remove configuration independently of normal checksum policy.

Before removing a service or complex module, inspect:

```text
PRE_REMOVE
POST_REMOVE
```

Do not assume `/etc` behavior is controlled only by `PRESERVE`.

## 29. Upgrade removal

During rebuild, LSS may invoke:

```text
lrm --upgrade
```

Upgrade removal differs from ordinary user-requested removal.

Its purpose is to replace old ownership while preparing the new installation.

Do not use runtime results from normal `lrm` to assume every upgrade detail is identical.

## 30. Removal after failed installation

A failed install may leave:

- partial payload;
- incomplete manifest;
- stale package record;
- no package record;
- temporary installwatch state.

Before running `lrm`, inspect what ownership evidence exists.

If the manifest is incomplete, normal removal may not clean all partial files.

## 31. Missing manifest

If:

```text
package record exists
manifest missing
```

normal removal is unsafe because ownership evidence is incomplete.

Preferred recovery:

```text
preserve state
→ rebuild or reinstall same configuration
→ regenerate manifest
→ verify
→ remove if still desired
```

## 32. Manifest without package record

If:

```text
manifest exists
package record absent
```

possible causes include:

- interrupted removal;
- manual state edit;
- stale logs.

Preserve evidence.

Inspect activity log and filesystem before deciding whether the manifest is stale or payload remains.

## 33. Shared ownership

Search all manifests:

```bash
grep -R -F -x '/path'   /var/log/lunar/install 2>/dev/null
```

Multiple matches indicate possible overlapping ownership.

Do not remove the path manually until the relationship is understood.

## 34. Files recreated after removal

A file may reappear because:

- a service recreates it;
- a login script regenerates it;
- another module owns it;
- a cache/index refresh recreates it;
- a hook runs after removal.

Compare timestamps and inspect hooks.

## 35. Safe removal test

For a low-risk module:

```text
select leaf
→ inspect role
→ inspect hooks
→ preserve logs and state
→ capture filesystem state
→ run lrm
→ compare state
→ verify payload
→ reinstall
→ verify clean restoration
```

This is the preferred method for studying removal behavior.

## 36. Filesystem-state capture

Before:

```bash
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
done < install-log > filesystem.before
```

Repeat after and compare:

```bash
diff -u filesystem.before filesystem.after
```

## 37. Reinstall after testing

After preserving evidence:

```bash
lin module
```

Verify:

```bash
grep '^module:' /var/state/lunar/packages
lvu installed module
```

For configuration:

```bash
md5sum /etc/module.conf
```

Remove test markers or orphaned files before reinstalling when the goal is a pristine default.

## 38. Removal and services

Before removing a daemon:

- stop it;
- inspect active sockets;
- preserve configuration;
- inspect service-manager units;
- inspect hooks;
- verify dependent services.

After removal:

- confirm process is gone;
- confirm service entry behavior;
- confirm ports are closed;
- confirm configuration preservation.

## 39. Removal and plugins

A plugin-bearing module may extend LSS or another application.

Before removal:

- identify plugin paths;
- identify host process;
- inspect reload behavior;
- stop active operations;
- preserve plugin configuration.

A plugin file can disappear while the host process still has old code loaded.

Restart or reload as appropriate.

## 40. Removal and critical modules

Do not casually remove:

```text
lunar
glibc
gcc
bash
coreutils
filesystem components
init system
network base
boot components
shared toolchain libraries
```

Even if dependency state appears sparse.

Use a container or recovery environment.

## 41. Removal checklist

Before:

```text
confirm installed version
check reverse dependencies
inspect package record
inspect depends records
inspect remove hooks
save install and MD5 logs
save configuration
prepare rollback
```

During:

```text
capture console output
avoid unrelated operations
stop if unexpected warnings appear
```

After:

```text
verify package record is gone
verify dependency-state changes
verify payload
inspect preserved configuration
inspect shared directories
review activity log
```

## 42. Common mistakes

### Mistake 1: removing a dependency before rebuilding consumers

The current binaries may still need it.

### Mistake 2: assuming all `/etc` files remain

Unchanged configuration is normally removed.

### Mistake 3: assuming preserved configuration is still package-owned

It becomes orphaned local state.

### Mistake 4: deleting shared directories manually

They may contain files from other modules.

### Mistake 5: forgetting logs disappear

Preserve them first.

### Mistake 6: trusting leaf status alone

Operational use may exist outside LSS records.

### Mistake 7: ignoring remove hooks

They may have external side effects.

## 43. Removal model

```text
module record
→ ownership manifest
→ path classification
→ checksum and protection policy
→ payload removal
→ empty-directory cleanup
→ Lunar artifact removal
→ packages and depends update
→ preserved local state remains
```

## 44. Summary

LSS removal is a policy-controlled ownership transition.

The essential distinctions are:

```text
owned payload
→ normally removed

shared directory
→ retained when still needed

unchanged configuration
→ removed

modified configuration + PRESERVE=on
→ retained as local orphaned state

PROTECTED
→ tracked but deletion blocked

EXCLUDED
→ not claimed as ownership
```

The central rule is:

```text
inspect ownership and policy before removal
→ preserve evidence
→ verify both filesystem and persistent state afterward
