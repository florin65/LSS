# Working with Moonbase

**Project:** Lunar Script System User Guide  
**Status:** Draft 0.1  
**Date:** 2026-07-17  
**Audience:** Lunar Linux users, maintainers, and system administrators  
**Evidence basis:** LSS Evidence Foundation 0.1 — Checkpoints 1–6

## 1. Overview

Moonbase is the active collection of module definitions used by LSS.

It tells LSS:

- which modules exist;
- where their sources are;
- which versions are available;
- how software is built;
- which dependencies are required or optional;
- which hooks and policies apply.

Moonbase is not the installed system.

It is the source of module intent.

The installed system is represented separately through:

```text
/var/state/lunar/packages
/var/state/lunar/depends
/var/log/lunar/install
/var/log/lunar/md5sum
```

## 2. Active Moonbase location

A common active path is:

```text
/var/lib/lunar/moonbase
```

Inspect:

```bash
ls -la /var/lib/lunar/moonbase
```

If managed by Git:

```bash
git -C /var/lib/lunar/moonbase status
git -C /var/lib/lunar/moonbase rev-parse HEAD
```

The active Moonbase used by the current system is more relevant than an unrelated repository clone.

## 3. Module sections

Modules are organized into sections.

Examples:

```text
core/libs
core/filesys
core/utils
```

A module name remains the normal operational identifier:

```bash
lin foremost
lrm foremost
lvu where foremost
```

The section is used to locate its definition.

## 4. Locating a module

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

## 5. Listing sections and modules

To inspect a section:

```bash
lvu section section-name
```

Use `lvu where` for a module name.

Do not confuse:

```bash
lvu section xxhash
```

with:

```bash
lvu where xxhash
```

The first expects a section.

The second locates a module.

## 6. Moonbase as executable policy

A Moonbase module is not static metadata.

Its files contain executable shell logic.

Examples:

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

Therefore Moonbase acts as:

```text
software catalog
+
build policy
+
dependency policy
+
lifecycle policy
```

## 7. Active Moonbase versus upstream repository

The active Moonbase may differ from upstream.

Possible reasons:

- local modifications;
- uncommitted patches;
- delayed update;
- pinned revision;
- private modules;
- experimental changes;
- branch differences.

Always inspect the active copy before diagnosing current behavior.

Upstream is useful for comparison, not automatic proof of what the local system executed.

## 8. Recording Moonbase revision

Before a significant update:

```bash
git -C /var/lib/lunar/moonbase rev-parse HEAD   > /root/moonbase-revision.before
```

Record branch:

```bash
git -C /var/lib/lunar/moonbase branch --show-current   > /root/moonbase-branch.before
```

Record status:

```bash
git -C /var/lib/lunar/moonbase status --short   > /root/moonbase-status.before
```

This makes later build results traceable.

## 9. Detecting local changes

Use:

```bash
git -C /var/lib/lunar/moonbase status --short
```

Inspect diff:

```bash
git -C /var/lib/lunar/moonbase diff
```

Save patch:

```bash
git -C /var/lib/lunar/moonbase diff   > /root/moonbase-local-changes.patch
```

Do not update Moonbase until local changes are understood and preserved.

## 10. Why local changes matter

A one-line change can alter:

- compiler selection;
- dependency graph;
- source URL;
- checksum;
- installed paths;
- service behavior;
- remove hooks;
- configuration defaults.

A build may therefore differ even when module name and version are unchanged.

## 11. Comparing a module across revisions

For one module:

```bash
git -C /var/lib/lunar/moonbase diff OLD..NEW --   core/libs/xxhash
```

Inspect history:

```bash
git -C /var/lib/lunar/moonbase log --   core/libs/xxhash
```

Show a previous version:

```bash
git -C /var/lib/lunar/moonbase show   REVISION:core/libs/xxhash/BUILD
```

This helps identify packaging changes separately from upstream changes.

## 12. Updating Moonbase safely

A safe process:

```text
record revision
→ preserve local changes
→ update Moonbase
→ inspect changed modules
→ review dependency changes
→ update selected modules gradually
```

Do not treat Moonbase update and full-system rebuild as the same operation.

Updating definitions does not automatically rebuild installed software.

## 13. Reviewing changed module definitions

After update:

```bash
git -C /var/lib/lunar/moonbase diff   REVISION_BEFORE..HEAD
```

List changed files:

```bash
git -C /var/lib/lunar/moonbase diff   --name-only REVISION_BEFORE..HEAD
```

Focus on:

```text
DETAILS
BUILD
DEPENDS
CONFIGURE
OPTIONS
hooks
patches
```

## 14. Module version versus packaging revision

A module can change without changing upstream version.

Examples:

- patch added;
- dependency corrected;
- compiler flag changed;
- install path fixed;
- checksum method changed;
- hook adjusted.

Therefore:

```text
same VERSION
≠ identical module behavior
```

Moonbase revision is part of reproducibility.

## 15. Local modules

A local module may be used for:

- private software;
- experimental updates;
- hardware-specific tools;
- unreleased patches;
- local services;
- testing new packaging.

Keep local modules clearly separated from upstream-managed content when possible.

This reduces accidental overwrite and makes ownership clear.

## 16. Local section strategy

A practical approach is a dedicated local section.

Conceptually:

```text
local/tools
local/services
local/libs
```

Benefits:

- easier review;
- easier backup;
- reduced merge conflict;
- clearer provenance;
- safer upstream updates.

Follow the actual Moonbase and LSS conventions in use on the system.

## 17. Testing a local module

Use a disposable environment.

Workflow:

```text
create module
→ inspect shell logic
→ build in container
→ inspect console output
→ inspect manifest
→ inspect packages and depends
→ test runtime
→ remove
→ verify cleanup
```

Do not begin with a critical production host.

## 18. Source verification

`DETAILS` may define:

- source archive;
- source URL;
- verification data;
- upstream site.

Inspect before trusting a new or modified module.

Check that:

```text
source URL matches intended upstream
version matches archive
verification data matches source
description matches software
```

A source URL change deserves review.

## 19. Patches

Patches are part of Moonbase behavior.

Inspect:

```bash
find "$MODULE_DIR" -type f   \( -name '*.patch' -o -name '*.diff' \)   -print
```

Search usage:

```bash
grep -R -n -E 'patch|sedit' "$MODULE_DIR"
```

A patch file present but unused does not affect the build.

A patch applied conditionally may affect only some configurations.

## 20. Dependency review after Moonbase update

Compare `DEPENDS` changes.

Potential impacts:

```text
new required dependency
new optional feature
removed dependency
provider replacement
default on/off change
```

Inspect current stored state:

```bash
grep "^${MODULE}:" /var/state/lunar/depends
```

The declaration may have changed while the installed state still reflects the old configuration.

## 21. Configuration review after update

A changed `CONFIGURE` or `OPTIONS` file may:

- add a new question;
- remove an old option;
- change defaults;
- rename a feature;
- alter provider selection.

Before rebuild, determine whether old stored choices still map correctly.

## 22. Build review after update

A changed `BUILD` may alter:

- toolchain;
- configure flags;
- install prefix;
- static/shared library policy;
- documentation;
- cleanup;
- generated files.

Compare old and new `BUILD` directly.

## 23. Hook review after update

New or changed hooks deserve special attention.

Examples:

```text
PRE_BUILD
POST_BUILD
POST_INSTALL
PRE_REMOVE
POST_REMOVE
```

A new hook can introduce side effects beyond ordinary ownership handling.

## 24. Moonbase and installed state can diverge

Common divergence:

```text
Moonbase version newer than installed
installed module built from old recipe
local Moonbase modified after installation
dependency declaration changed
current manifest reflects previous recipe
```

This is normal until the module is rebuilt or updated.

Do not expect the installed system to change immediately when Moonbase changes.

## 25. Inspecting installed versus current module definition

Record:

```bash
MODULE=name
VERSION_INSTALLED=$(lvu installed "$MODULE")
VERSION_MOONBASE=$(lvu version "$MODULE")
SECTION=$(lvu where "$MODULE")
```

Show:

```bash
printf 'installed=%s\n' "$VERSION_INSTALLED"
printf 'moonbase=%s\n' "$VERSION_MOONBASE"
printf 'section=%s\n' "$SECTION"
```

Then inspect history to determine whether the recipe changed.

## 26. Pinning Moonbase revision

For a stable environment, pinning a known revision may be useful.

Benefits:

- reproducible module definitions;
- controlled updates;
- easier rollback;
- clearer evidence.

Costs:

- delayed fixes;
- delayed security updates;
- manual update planning.

A pinned revision must be documented.

## 27. Branches

Different branches may represent:

- stable;
- testing;
- development;
- local experimentation.

Check:

```bash
git -C /var/lib/lunar/moonbase branch --show-current
```

Do not switch branches on a production system without reviewing differences.

## 28. Updating with local modifications

Safe options:

```text
commit local changes
stash local changes
export patch
use dedicated local branch
move private modules to local section
```

Avoid:

```text
pull and hope
```

A merge conflict inside `BUILD` or `DEPENDS` can produce subtle breakage.

## 29. Moonbase backup

Backup:

```bash
tar -cJf /root/moonbase-backup.tar.xz   /var/lib/lunar/moonbase
```

Or preserve Git metadata and working tree through filesystem backup.

For a compact reproducibility record, save:

```text
revision
branch
status
local diff
selected module directories
```

## 30. Searching Moonbase

Search module names:

```bash
find /var/lib/lunar/moonbase   -mindepth 2   -maxdepth 3   -type d   -name 'pattern'
```

Search content:

```bash
grep -R -n -E 'prepare_install|optional_depends|CC='   /var/lib/lunar/moonbase
```

Use narrow searches to avoid excessive noise.

## 31. Finding modules using a dependency

Search persistent state:

```bash
grep -E '^[^:]+:dependency:'   /var/state/lunar/depends
```

Search declarations:

```bash
grep -R -n -F 'dependency'   /var/lib/lunar/moonbase/*/*/*/DEPENDS
```

These answer different questions:

```text
persistent state
→ currently resolved relationships

Moonbase search
→ declared possibilities
```

## 32. Finding modules with hooks

```bash
find /var/lib/lunar/moonbase   -type f   \( -name PRE_REMOVE -o -name POST_REMOVE      -o -name PRE_BUILD -o -name POST_BUILD      -o -name POST_INSTALL \)   -print
```

Useful for architectural review and risk analysis.

## 33. Finding modules that install plugins

Search:

```bash
grep -R -n -E 'plugin\.d|\.plugin'   /var/lib/lunar/moonbase
```

Review such modules carefully before removal or update.

## 34. Canonical source versus working fork

For project research, distinguish:

```text
canonical upstream repository
working fork
local active Moonbase
```

The canonical repository explains upstream intent.

The working fork explains local development.

The active Moonbase explains current system behavior.

Never collapse these into one source.

## 35. Reproducibility record

For a significant build, preserve:

```bash
OUT=/root/lss-build-record

mkdir -p "$OUT"

git -C /var/lib/lunar/moonbase rev-parse HEAD   > "$OUT/moonbase-revision"

git -C /var/lib/lunar/moonbase status --short   > "$OUT/moonbase-status"

git -C /var/lib/lunar/moonbase diff   > "$OUT/moonbase-diff"

cp -a "$MODULE_DIR" "$OUT/module-definition"
```

Add package, dependency, environment, and toolchain records.

## 36. Common mistakes

### Mistake 1: diagnosing from upstream only

The active local Moonbase may differ.

### Mistake 2: updating over local modifications

This risks lost or merged behavior.

### Mistake 3: assuming Moonbase update changes installed software

A rebuild is still required.

### Mistake 4: comparing only VERSION

Packaging logic can change independently.

### Mistake 5: mixing local modules into upstream sections without documentation

This complicates updates and provenance.

### Mistake 6: treating a fork as canonical

Use the correct source for the question.

### Mistake 7: switching branches without reviewing module differences

This can change the entire build policy.

## 37. Safe Moonbase workflow

```text
inspect active revision
→ preserve local changes
→ update
→ review diffs
→ classify changed modules
→ rebuild selectively
→ verify state and runtime
```

## 38. Summary

Moonbase is the active executable definition of the software universe known to LSS.

It describes:

```text
identity
source
dependencies
configuration
build
installation
removal
integration
```

The central rule is:

```text
Moonbase defines intent
→ installed state records result
→ runtime verifies reality
