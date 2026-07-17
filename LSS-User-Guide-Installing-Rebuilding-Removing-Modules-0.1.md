# Installing, Rebuilding, and Removing Modules

**Project:** Lunar Script System User Guide  
**Status:** Draft 0.1  
**Date:** 2026-07-17  
**Audience:** Lunar Linux users and system administrators  
**Evidence basis:** LSS Module Lifecycle 0.1 and runtime Checkpoints 1–6

## 1. Overview

LSS manages software through modules stored in Moonbase.

The three most common operations are:

```bash
lin module
lin -c module
lrm module
```

They mean:

```text
lin module
→ install a module

lin -c module
→ rebuild an installed module

lrm module
→ remove an installed module
```

These commands affect more than files on disk. They may also update:

```text
/var/state/lunar/packages
/var/state/lunar/depends
/var/log/lunar/install
/var/log/lunar/compile
/var/log/lunar/md5sum
/var/cache/lunar
/var/log/lunar/activity
```

Run them as root.

## 2. Before changing a module

Before installing, rebuilding, or removing a module, inspect its current state.

Useful commands:

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

These answer different questions:

```text
lvu installed
→ which version is installed?

lvu where
→ where is the module stored in Moonbase?

lvu depends
→ which installed modules depend on it?

lvu leert
→ what does its reverse dependency tree look like?
```

Do not remove a module only because it looks small or unfamiliar.

A module may be required indirectly by:

- the compiler toolchain;
- the boot process;
- networking;
- package management;
- another installed module;
- local system administration scripts.

## 3. Locating a module

Use:

```bash
lvu where module
```

Example:

```bash
lvu where foremost
```

Possible output:

```text
core/filesys
```

The full module directory is then typically:

```text
/var/lib/lunar/moonbase/core/filesys/foremost
```

Inspect its contents:

```bash
find /var/lib/lunar/moonbase/core/filesys/foremost   -maxdepth 2 -type f -print | sort
```

Common module files include:

```text
DETAILS
BUILD
DEPENDS
CONFIGURE
PRE_BUILD
POST_BUILD
POST_INSTALL
PRE_REMOVE
POST_REMOVE
```

Not every module uses every lifecycle file.

## 4. Inspecting the module definition

Read `DETAILS` to see:

- version;
- source URL;
- checksums;
- description;
- maintainer metadata.

Read `BUILD` to understand:

- compiler flags;
- patches;
- configure steps;
- installation commands;
- files removed after installation;
- custom paths.

Example:

```bash
sed -n '1,240p'   /var/lib/lunar/moonbase/core/filesys/foremost/BUILD
```

For potentially sensitive modules, also inspect:

```bash
for hook in PRE_REMOVE POST_REMOVE PRE_INSTALL POST_INSTALL; do
  test -f "$MODULE_DIR/$hook" && {
    echo "=== $hook ==="
    cat "$MODULE_DIR/$hook"
  }
done
```

This is especially important before removing services, shells, boot components, or package-management tools.

## 5. Installing a module

Install with:

```bash
lin module
```

Example:

```bash
lin foremost
```

A normal installation may perform:

```text
resolve configuration
→ resolve dependencies
→ download sources
→ compile
→ remove an older installation if needed
→ install files
→ create ownership manifest
→ create checksum log
→ create compile log
→ create cache archive
→ update package state
```

Typical console phases include:

```text
Downloading source file ...
Building ...
Preparing to install ...
Installing ...
installation completed
Creating /var/log/lunar/compile/...
Creating /var/log/lunar/install/...
Creating /var/log/lunar/md5sum/...
Creating /var/cache/lunar/...
```

Not every module prints exactly the same messages.

## 6. Verifying an installation

After `lin` finishes, check the package-state record:

```bash
grep '^module:' /var/state/lunar/packages
```

Example:

```bash
grep '^foremost:' /var/state/lunar/packages
```

A record has this general form:

```text
module:date:state:version:size
```

Example:

```text
foremost:20260717:installed:1.5.7:116KB
```

Check the installed version:

```bash
lvu installed foremost
```

Inspect the ownership manifest:

```bash
cat /var/log/lunar/install/foremost-1.5.7
```

Check the installed command:

```bash
command -v foremost
```

For a library, use the appropriate file or linker inspection instead.

Examples:

```bash
ls -l /usr/lib/libxxhash.so*
pkgconf --modversion libxxhash
```

## 7. Understanding the install manifest

The install manifest is stored at:

```text
/var/log/lunar/install/module-version
```

It records the final paths attributed to the module.

It may contain:

- files;
- symbolic links;
- directories;
- documentation;
- manual pages;
- Lunar-generated logs.

It does not necessarily contain every path touched during the build.

A file can be:

```text
created during BUILD
→ observed by installwatch
→ removed before finalization
→ absent from final manifest
```

Therefore the manifest describes final ownership, not raw build history.

## 8. Installed size

The size stored in `/var/state/lunar/packages` is calculated from regular files listed in the final manifest.

It may include:

- executables;
- libraries;
- headers;
- documentation;
- manual pages;
- Lunar install log;
- Lunar compile log;
- Lunar MD5 log.

It does not directly count directories or symbolic links.

Therefore the recorded size should be understood as:

```text
regular filesystem content attributed to the module
```

not strictly:

```text
upstream application payload
```

A rebuild may recalculate this value even when the version and manifest remain unchanged.

## 9. Rebuilding an installed module

Use:

```bash
lin -c module
```

Example:

```bash
lin -c xxhash
```

A rebuild compiles and installs the module again.

For an already installed module, LSS uses an upgrade-style transition:

```text
build replacement
→ prepare installation
→ remove old ownership using upgrade semantics
→ install replacement
→ regenerate logs and manifest
→ update package state
```

Use rebuild when:

- testing changed compiler flags;
- validating a new toolchain;
- repairing a damaged installation;
- applying an updated module recipe;
- regenerating metadata;
- testing reproducibility.

## 10. Before a rebuild

Record the current state when the result matters.

Example:

```bash
MODULE=xxhash
VERSION=$(lvu installed "$MODULE")
BASE=/root/lss-check/$MODULE

mkdir -p "$BASE"

grep "^${MODULE}:" /var/state/lunar/packages   > "$BASE/package.before"

cp -a "/var/log/lunar/install/${MODULE}-${VERSION}"   "$BASE/install.before"

cp -a "/var/log/lunar/md5sum/${MODULE}-${VERSION}"   "$BASE/md5sum.before"
```

Then rebuild:

```bash
lin -c "$MODULE" 2>&1 | tee "$BASE/rebuild.log"
```

Compare afterward:

```bash
grep "^${MODULE}:" /var/state/lunar/packages   > "$BASE/package.after"

cp -a "/var/log/lunar/install/${MODULE}-${VERSION}"   "$BASE/install.after"

diff -u "$BASE/install.before" "$BASE/install.after"
diff -u "$BASE/package.before" "$BASE/package.after"
```

No manifest difference means the final owned path set is unchanged.

The package record may still show a new date or recalculated size.

## 11. Rebuild safety

Before rebuilding a critical module, consider:

- available disk space;
- build time;
- source availability;
- whether the installed system depends on the module during the build;
- whether a cache exists;
- whether recovery media or a container snapshot is available.

Critical examples include:

```text
glibc
gcc
bash
coreutils
lunar
openssl
kernel
filesystem-related libraries
```

Prefer a disposable container or chroot for experiments.

## 12. Removing a module

Remove with:

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

Normal removal is driven by the module's install manifest.

LSS uses that manifest to identify owned paths.

## 13. Before removing a module

Check reverse dependencies:

```bash
lvu depends module
lvu leert module
```

A safe standalone candidate may show no dependents.

Example:

```text
foremost:
```

This represents the root of the reverse tree with no dependent branches.

Also inspect dependency records:

```bash
grep -E '^module:|^[^:]+:module:'   /var/state/lunar/depends
```

Example:

```bash
grep -E '^foremost:|^[^:]+:foremost:'   /var/state/lunar/depends
```

No output means no matching stored relationships were found.

Do not treat that alone as proof that removal is safe. Also consider operational use outside LSS.

## 14. Preserve evidence before removal

Module-specific logs are normally removed with the module.

Copy them first when you need them:

```bash
MODULE=foremost
VERSION=$(lvu installed "$MODULE")
BACKUP=/root/lss-remove-backup/$MODULE

mkdir -p "$BACKUP"

cp -a "/var/log/lunar/install/${MODULE}-${VERSION}"   "$BACKUP/install-log"

cp -a "/var/log/lunar/md5sum/${MODULE}-${VERSION}"   "$BACKUP/md5sum-log"

cp -a "/var/log/lunar/compile/${MODULE}-${VERSION}".*   "$BACKUP/" 2>/dev/null || true

grep "^${MODULE}:" /var/state/lunar/packages   > "$BACKUP/package-record"
```

This matters because `lrm` may delete:

```text
install manifest
compile log
MD5 log
```

## 15. What removal deletes

For a normal standalone module, LSS may remove:

- executable files;
- libraries;
- headers;
- documentation;
- manual pages;
- module-specific directories that become empty;
- module-specific Lunar logs;
- the module record from `/var/state/lunar/packages`;
- dependency records when applicable.

Validated example:

```text
/usr/bin/foremost
/usr/share/doc/foremost/CHANGES
/usr/share/doc/foremost/README
/usr/share/man/man8/foremost.8.gz
```

were removed.

## 16. What removal does not blindly delete

Shared directories remain when still required.

Validated examples:

```text
/usr
/usr/share
/usr/share/doc
```

remained after removing `foremost`.

An exclusive directory may disappear when empty:

```text
/usr/share/doc/foremost
```

Therefore the presence of a directory in the install manifest does not mean LSS will remove the entire directory tree regardless of other content.

## 17. Configuration files and `PRESERVE`

LSS treats files under `/etc` specially.

With:

```text
PRESERVE=on
```

the validated behavior is:

```text
configuration unchanged since installation
→ remove it with the module

configuration modified locally
→ preserve it
```

This protects local administrative changes without leaving every unmodified configuration behind.

## 18. Checking whether a configuration was modified

Use the module's MD5 log:

```text
/var/log/lunar/md5sum/module-version
```

For a simple manual check:

```bash
md5sum /etc/example.conf
grep '/etc/example.conf'   /var/log/lunar/md5sum/module-version
```

The exact format of the MD5 log should be inspected before scripting around it.

The decisive question is:

```text
does the current file match the checksum recorded at installation?
```

## 19. Preserved orphaned configuration

When a modified configuration survives `lrm`:

```text
the file remains
the package record disappears
the install and MD5 logs disappear
```

The file is now orphaned local state.

LSS no longer records an installed module as its owner.

Before reinstalling, decide whether to:

- keep it and let the new installation interact with it;
- back it up;
- compare it with the new default;
- remove it deliberately.

## 20. Reinstalling after removal

Use:

```bash
lin module
```

After reinstalling, verify:

```bash
grep '^module:' /var/state/lunar/packages
lvu installed module
```

For a configuration file:

```bash
md5sum /etc/module.conf
```

When testing preservation behavior, remove or archive the orphaned modified file first if the goal is to restore the pristine default.

Example:

```bash
cp -a /etc/foremost.conf   /root/foremost.conf.saved

rm -f /etc/foremost.conf
lin foremost
```

## 21. Held, exiled, and enforced states

The package-state database supports more than `installed`.

Known state terms include:

```text
held
exiled
enforced
```

These are policy states and can affect normal operations.

Before changing a module, inspect its complete record:

```bash
grep '^module:' /var/state/lunar/packages
```

Do not assume that every installed-looking module is freely upgradeable or removable.

The exact operational handling of all policy-state combinations belongs in a separate chapter.

## 22. Caches

With archiving enabled, installation may create a cache such as:

```text
/var/cache/lunar/module-version-triplet.tar.xz
```

Check with:

```bash
ls -l /var/cache/lunar/module-*
```

A cache is not the same as an install manifest.

It is a reusable installation artifact.

The relevant objects are distinct:

```text
source archive
build tree
installation cache
install manifest
compile log
MD5 log
package-state record
```

## 23. Logs

Useful locations:

```text
/var/log/lunar/activity
/var/log/lunar/compile
/var/log/lunar/install
/var/log/lunar/md5sum
```

Use the activity log for broader history:

```bash
grep 'module' /var/log/lunar/activity
```

Use the compile log for build failures:

```bash
xzless /var/log/lunar/compile/module-version.xz
```

Use the install manifest to inspect ownership:

```bash
cat /var/log/lunar/install/module-version
```

Use the MD5 log for `/etc` decisions and integrity comparison.

## 24. Troubleshooting a failed installation

When `lin` fails:

1. identify the reported phase;
2. inspect the compile log;
3. inspect the module's BUILD and lifecycle hooks;
4. verify compiler and linker selection;
5. verify source download and checksum;
6. verify available disk space;
7. check dependency state;
8. avoid repeated blind rebuilds.

Typical phase labels include:

```text
PRE_BUILD
BUILD
POST_BUILD
POST_INSTALL
```

Open the compile log:

```bash
xzless /var/log/lunar/compile/module-version.xz
```

If the log was not finalized, inspect the path reported by `lin` or the temporary build area.

## 25. Troubleshooting an unexpected removal result

If a file remains after `lrm`, determine whether it is:

- a modified `/etc` file preserved by policy;
- a `PROTECTED` path;
- an `EXCLUDED` path that was never owned;
- a shared file or directory;
- recreated by another service;
- owned by another installed module;
- outside the old install manifest.

Check:

```bash
test -e /path && ls -ld /path
grep -Fx '/path' /root/saved-install-log
grep -R -F '/path' /var/log/lunar/install 2>/dev/null
```

The last command may identify another module manifest containing the same path.

## 26. A safe operational pattern

For an unfamiliar module:

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"
```

Inspect:

```bash
grep "^${MODULE}:" /var/state/lunar/packages
lvu depends "$MODULE"
lvu leert "$MODULE"
find "$MODULE_DIR" -maxdepth 2 -type f -print | sort
cat "/var/log/lunar/install/${MODULE}-${VERSION}"
```

Preserve evidence:

```bash
mkdir -p "/root/module-backup/$MODULE"

cp -a "/var/log/lunar/install/${MODULE}-${VERSION}"   "/root/module-backup/$MODULE/"

cp -a "/var/log/lunar/md5sum/${MODULE}-${VERSION}"   "/root/module-backup/$MODULE/"
```

Then perform only the intended operation:

```bash
lin "$MODULE"
```

or:

```bash
lin -c "$MODULE"
```

or:

```bash
lrm "$MODULE"
```

Verify immediately afterward.

## 27. Minimal verification checklist

### After install

```bash
lvu installed module
grep '^module:' /var/state/lunar/packages
test -f /var/log/lunar/install/module-version
```

### After rebuild

```bash
lvu installed module
diff old-install-log new-install-log
grep '^module:' /var/state/lunar/packages
```

### After removal

```bash
grep '^module:' /var/state/lunar/packages || true
test -e /known/payload/path && echo present || echo missing
```

### For configuration preservation

```bash
test -f /etc/module.conf && echo preserved || echo removed
```

## 28. Safety rules

1. Check reverse dependencies before removal.
2. Avoid testing on critical modules.
3. Preserve logs before `lrm`.
4. Use a container, chroot, snapshot, or backup for experiments.
5. Inspect module hooks.
6. Treat `/etc` files as administrative state.
7. Do not infer ownership only from filesystem presence.
8. Do not infer final ownership from raw install events.
9. Verify package and dependency databases after state-changing operations.
10. Restore instrumentation and test modifications when finished.

## 29. Summary

The normal operational model is:

```text
install
→ build and observe
→ record final ownership
→ register persistent state

rebuild
→ replace old owned state
→ regenerate ownership and metadata

remove
→ follow the ownership manifest
→ preserve modified configuration according to policy
→ remove package state and module artifacts
```

LSS combines flexible shell-based module recipes with filesystem observation and persistent ownership records.

For the user, the essential discipline is:

```text
inspect
→ preserve evidence
→ perform one operation
→ verify state
```
