# System Recovery and State Repair

**Project:** Lunar Script System User Guide  
**Status:** Draft 0.1  
**Date:** 2026-07-17  
**Audience:** Advanced Lunar Linux users, maintainers, and system administrators  
**Evidence basis:** LSS Evidence Foundation 0.1 — Checkpoints 1–6

## 1. Overview

LSS maintains several layers of system state:

```text
filesystem payload
install manifest
MD5 log
package-state database
dependency-state database
Moonbase definition
cache
activity history
```

A healthy system keeps these layers aligned.

Recovery is needed when one or more layers disagree.

The first rule is:

```text
preserve evidence before repair
```

## 2. Common inconsistency classes

Typical cases:

```text
package record exists, payload missing
package record exists, manifest missing
manifest exists, package record missing
dependency record references missing module
payload exists, no ownership record
configuration survives without package ownership
cache exists, installed state absent
Moonbase definition changed, installed state old
```

Each case requires a different response.

## 3. Do not repair by reflex

Avoid immediate actions such as:

```text
editing packages by hand
editing depends by hand
deleting stale-looking logs
manually copying binaries into /usr
manually unpacking cache into /
```

These actions may hide the real inconsistency.

First determine which layer is authoritative for the question.

## 4. Build a recovery bundle

Before repair:

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
BASE=/root/lss-recovery/$MODULE

mkdir -p "$BASE"
```

Capture state:

```bash
cp -a /var/state/lunar/packages   "$BASE/packages.before"

cp -a /var/state/lunar/depends   "$BASE/depends.before"

grep "^${MODULE}:" /var/state/lunar/packages   > "$BASE/package-record.before" || true

grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends   > "$BASE/depends-records.before" || true
```

Capture logs:

```bash
cp -a /var/log/lunar/install/${MODULE}-*   "$BASE/" 2>/dev/null || true

cp -a /var/log/lunar/md5sum/${MODULE}-*   "$BASE/" 2>/dev/null || true

cp -a /var/log/lunar/compile/${MODULE}-*   "$BASE/" 2>/dev/null || true
```

Capture Moonbase definition:

```bash
SECTION=$(lvu where "$MODULE" 2>/dev/null)
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

test -d "$MODULE_DIR" &&
  cp -a "$MODULE_DIR" "$BASE/module-definition"
```

## 5. Record physical state

If a manifest exists:

```bash
MANIFEST=$(ls /var/log/lunar/install/${MODULE}-* 2>/dev/null | head -1)
```

Classify paths:

```bash
if [ -n "$MANIFEST" ]; then
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
  done < "$MANIFEST"     > "$BASE/filesystem-state.before"
fi
```

## 6. Package record exists, payload missing

Condition:

```text
packages
→ says installed

filesystem
→ important payload missing
```

Possible causes:

- manual deletion;
- filesystem damage;
- interrupted operation;
- restored state without payload.

Preferred recovery:

```text
preserve evidence
→ rebuild or reinstall module
→ regenerate payload
→ verify manifest
→ verify runtime
```

Do not delete the package record simply to remove the warning.

## 7. Package record exists, manifest missing

Condition:

```text
package state
→ present

ownership evidence
→ absent
```

Risk:

```text
lrm cannot reliably know what to remove
```

Preferred recovery:

```text
rebuild or reinstall the same configuration
→ regenerate manifest and MD5 log
→ compare payload
→ verify package state
```

Avoid removal before ownership is restored.

## 8. Manifest exists, package record missing

Condition:

```text
ownership log
→ present

package record
→ absent
```

Possible causes:

- interrupted removal;
- manual state edit;
- stale log;
- partial recovery.

Investigate:

```bash
grep "$MODULE" /var/log/lunar/activity
```

Check payload paths.

Do not assume the manifest still represents current ownership.

## 9. Payload exists, no manifest and no package record

Condition:

```text
files exist
→ no LSS ownership
```

Possible causes:

- manual install;
- preserved config;
- interrupted package lifecycle;
- external tool;
- orphaned files.

Identify provenance before assigning ownership.

Search all manifests:

```bash
grep -R -F -x '/path'   /var/log/lunar/install 2>/dev/null
```

## 10. Dependency record references missing module

Condition:

```text
depends
→ relationship exists

packages
→ dependency absent
```

Possible causes:

- interrupted removal;
- manual state edit;
- failed dependency installation;
- stale configuration.

Preferred action:

```text
inspect dependent module
→ inspect DEPENDS/CONFIGURE/OPTIONS
→ reconfigure or rebuild dependent
→ verify new dependency state
```

Do not simply delete the dependency record unless no supported path exists.

## 11. Installed dependency with no reverse dependents

Condition:

```text
module installed
→ no current reverse relationships
```

Possible interpretations:

- direct user-installed tool;
- orphaned dependency;
- provider retained intentionally;
- stale dependency state.

Use:

```bash
lvu leafs
lvu depends module
lvu leert module
```

Review role before removal.

## 12. Preserved orphaned configuration

Condition:

```text
/etc file remains
package removed
```

This may be correct behavior under:

```text
PRESERVE=on
```

Check saved MD5 evidence.

Actions:

- keep;
- archive;
- compare with new default;
- remove deliberately.

Do not force ownership onto a new module without review.

## 13. Cache exists, package absent

This is normal.

```text
cache
→ reusable payload

package state
→ current installation
```

A cache alone does not mean installed.

Use LSS to reinstall.

Avoid manual extraction into `/`.

## 14. Package installed, cache absent

Also normal.

The module can be installed without a reusable cache.

Possible reasons:

- `ARCHIVE=off`;
- cache deleted;
- cache creation failed;
- older installation.

Ownership and removal still depend on manifest and state.

## 15. Stale package size

A recorded size may differ from current calculation.

Reproduce:

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

If manifest and payload are healthy, a rebuild may refresh size metadata.

## 16. Stale Moonbase versus installed state

Moonbase may change while installed state remains old.

This is not necessarily corruption.

Compare:

```bash
lvu installed module
lvu version module
```

Inspect Moonbase history.

Rebuild only after reviewing changed dependencies and configuration.

## 17. Interrupted rebuild

Possible states:

```text
old package removed
partial new payload
no final manifest
stale package record
old cache available
```

Recovery sequence:

```text
stop further updates
→ preserve all evidence
→ inspect activity and compile log
→ inspect package record
→ inspect old cache
→ reinstall known-good version/configuration
→ verify
```

## 18. Interrupted removal

Possible states:

```text
package record removed
some payload remains
logs partially removed
configuration preserved
```

Use saved or surviving manifest.

If no manifest remains, compare against cache, backup, or module reinstall in a test environment.

## 19. Repair through rebuild

A rebuild is often the safest repair when:

- module definition still exists;
- source is available;
- configuration is known;
- toolchain works;
- payload is incomplete;
- manifest is missing or stale.

Use:

```bash
lin -c module
```

Preserve console output.

## 20. Repair through reinstall

A reinstall is appropriate when:

- module is no longer recorded as installed;
- payload is absent or incomplete;
- cache or source is trusted;
- clean ownership must be recreated.

Use:

```bash
lin module
```

Then verify package state, manifest, MD5 log, and runtime.

## 21. Repair through cache

Use cache when:

- source build is unavailable;
- trusted archive exists;
- configuration matches;
- target triplet matches;
- provenance is known.

Restore through LSS.

Do not unpack manually unless performing expert emergency recovery.

## 22. Repair through rollback

Rollback sources:

```text
trusted old cache
filesystem snapshot
container backup
system backup
old Moonbase revision
old source archive
```

A complete rollback restores:

```text
payload
manifest
MD5 log
package state
dependency state
configuration
```

## 23. Manual database repair

Manual repair is last resort.

Before:

```bash
cp -a /var/state/lunar/packages   /root/packages.pre-manual-repair

cp -a /var/state/lunar/depends   /root/depends.pre-manual-repair
```

Document:

- exact inconsistency;
- why supported operations failed;
- exact line changed;
- expected result;
- validation performed.

Make the smallest possible edit.

## 24. Manual manifest reconstruction

Also last resort.

Use only when:

- no rebuild possible;
- no cache;
- no backup;
- payload must be removed or recovered;
- ownership can be established from strong evidence.

Possible evidence:

- package cache content;
- upstream install list;
- filesystem timestamps;
- activity logs;
- old backups;
- another identical system.

A guessed manifest is dangerous.

## 25. State consistency matrix

### Healthy installed module

```text
package record present
manifest present
MD5 log present
payload present
dependency state coherent
```

### Healthy removed module

```text
package record absent
manifest absent
payload absent
preserved local config possible
```

### Suspicious state

```text
record present, manifest absent
record absent, manifest present
active dependency, missing module
manifest paths missing
payload present, no ownership
```

## 26. Verify after repair

Package state:

```bash
grep '^module:' /var/state/lunar/packages
lvu installed module
```

Ownership:

```bash
cat /var/log/lunar/install/module-version
```

Dependency state:

```bash
grep -E '^module:|^[^:]+:module:'   /var/state/lunar/depends
```

Runtime:

```bash
command -v program
ldd /usr/bin/program
program --version
```

Configuration:

```bash
md5sum /etc/module.conf
```

## 27. System-wide state audit

List package records:

```bash
wc -l /var/state/lunar/packages
```

List manifests:

```bash
find /var/log/lunar/install   -maxdepth 1   -type f   -printf '%f
' |
  sort
```

Look for obvious mismatches between installed records and manifests.

Automated audit should report, not auto-repair.

## 28. Check installed modules without manifests

Conceptual audit:

```bash
awk -F: '$3 ~ /installed/ {print $1 ":" $4}'   /var/state/lunar/packages |
while IFS=: read -r module version; do
  test -f "/var/log/lunar/install/${module}-${version}" ||
    echo "missing manifest: ${module}-${version}"
done
```

Review results manually.

Some historical exceptions may exist.

## 29. Check manifests without records

```bash
for manifest in /var/log/lunar/install/*; do
  name=$(basename "$manifest")
  module=${name%-*}

  grep "^${module}:" /var/state/lunar/packages >/dev/null ||
    echo "manifest without obvious record: $name"
done
```

Version parsing by filename can be ambiguous when names contain dashes.

Treat output as a lead, not proof.

## 30. Check missing manifest paths

```bash
while IFS= read -r path; do
  [ -e "$path" ] || [ -L "$path" ] ||
    echo "missing: $path"
done < manifest
```

Some paths may be intentionally absent after later transitions.

Investigate context.

## 31. Check dependency references

```bash
awk -F: '{print $1; print $2}'   /var/state/lunar/depends |
sort -u |
while read -r module; do
  grep "^${module}:" /var/state/lunar/packages >/dev/null ||
    echo "dependency-state module absent: $module"
done
```

This may report exiled or special-state modules.

Interpret policy state.

## 32. Repair and policy states

Do not lose:

```text
held
exiled
enforced
```

during recovery.

Restoring only `installed` may change administrator intent.

Preserve full records.

## 33. Repair and local Moonbase changes

If the installed module came from a local patch:

```text
rebuild from upstream current module
→ may not reproduce the installed result
```

Preserve active module definition and local diff before recovery.

## 34. Repair and optional features

A reinstall with different optional choices may restore a working module but change functionality.

Preserve dependency records and configuration answers.

Recovery should restore intended behavior, not merely a successful binary.

## 35. Repair and toolchain drift

A module rebuilt with a new compiler may differ from the damaged installation.

Record:

```text
CC
CXX
CFLAGS
CXXFLAGS
LDFLAGS
toolchain versions
```

Use a trusted cache if exact rebuild is impossible.

## 36. Recovery checkpoints

After each major repair step:

```bash
cp -a /var/state/lunar/packages   "$BASE/packages.checkpoint-N"

cp -a /var/state/lunar/depends   "$BASE/depends.checkpoint-N"
```

Record what changed.

Do not perform several major repairs without intermediate checkpoints.

## 37. Recovery report

A useful report includes:

```text
initial inconsistency
evidence preserved
probable cause
chosen repair path
commands executed
state diffs
runtime verification
remaining uncertainty
```

This turns recovery into reusable knowledge.

## 38. Common mistakes

### Mistake 1: deleting state to match missing files

This hides damage.

### Mistake 2: restoring files without ownership

Future removal and updates remain broken.

### Mistake 3: restoring ownership without payload

State remains false.

### Mistake 4: ignoring dependency state

Consumers may still be inconsistent.

### Mistake 5: losing policy states

Administrator intent changes silently.

### Mistake 6: rebuilding with different options unknowingly

Functionality may drift.

### Mistake 7: repairing before preserving evidence

Cause becomes harder to understand.

## 39. Recovery decision model

```text
identify inconsistent layers
→ preserve all relevant evidence
→ choose supported repair path
→ restore alignment
→ verify persistent state
→ verify filesystem
→ verify runtime
→ document
```

## 40. Summary

LSS recovery is the restoration of agreement between several state layers.

```text
Moonbase
→ intended module behavior

packages and depends
→ persistent administrative state

manifest and MD5
→ ownership and installed checksums

filesystem
→ physical payload

runtime
→ actual functionality
```

The central rule is:

```text
repair alignment
→ not only the visible symptom
