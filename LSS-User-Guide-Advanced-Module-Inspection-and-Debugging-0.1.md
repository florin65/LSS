# Advanced Module Inspection and Debugging

**Project:** Lunar Script System User Guide  
**Status:** Draft 0.1  
**Date:** 2026-07-17  
**Audience:** Advanced Lunar Linux users, maintainers, and system administrators  
**Evidence basis:** LSS Evidence Foundation 0.1 — Checkpoints 1–6

## 1. Overview

Most LSS problems can be solved with ordinary inspection:

```text
console output
compile log
install manifest
MD5 log
packages
depends
Moonbase module files
```

Advanced debugging is needed when these views disagree or when the normal evidence is incomplete.

Typical cases include:

- package record exists but files are missing;
- manifest exists but package record is absent;
- dependency state looks stale;
- rebuild changes metadata unexpectedly;
- a file appears during build but not in the final manifest;
- removal preserves or deletes an unexpected path;
- cache behavior hides the actual build path;
- a hook produces side effects outside the manifest.

The purpose of advanced debugging is not to collect more data blindly.

It is to isolate one unclear transition and observe it precisely.

## 2. The advanced debugging model

A useful model is:

```text
declared intent
→ runtime events
→ interpreted ownership
→ persistent state
→ physical filesystem
→ runtime behavior
```

These layers may disagree.

Advanced inspection compares them directly.

## 3. Preserve first

Before changing anything:

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
BASE=/root/lss-advanced/$MODULE

mkdir -p "$BASE/before" "$BASE/after"
```

Preserve state:

```bash
cp -a /var/state/lunar/packages   "$BASE/before/packages"

cp -a /var/state/lunar/depends   "$BASE/before/depends"

grep "^${MODULE}:" /var/state/lunar/packages   > "$BASE/before/package-record" || true

grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends   > "$BASE/before/depends-records" || true
```

Preserve logs:

```bash
cp -a "/var/log/lunar/install/${MODULE}-${VERSION}"   "$BASE/before/install-log" 2>/dev/null || true

cp -a "/var/log/lunar/md5sum/${MODULE}-${VERSION}"   "$BASE/before/md5sum-log" 2>/dev/null || true

cp -a "/var/log/lunar/compile/${MODULE}-${VERSION}".*   "$BASE/before/" 2>/dev/null || true
```

Preserve module definition:

```bash
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

cp -a "$MODULE_DIR" "$BASE/before/module-definition"
```

## 4. Record the environment

Record the execution environment:

```bash
uname -a > "$BASE/before/uname"

env | sort > "$BASE/before/environment"

{
  command -v gcc && gcc --version
  command -v clang && clang --version
  command -v ld && ld --version
  command -v make && make --version
} > "$BASE/before/toolchain" 2>&1
```

Also record global LSS configuration:

```bash
cp -a /etc/lunar/config   "$BASE/before/lunar-config"
```

Environment drift can explain behavior that module and state files cannot.

## 5. Build a precise question

Good advanced-debugging questions are narrow.

Examples:

```text
Why is this file missing from the final manifest?

Why did installed size change while the manifest stayed identical?

Why did lrm preserve this /etc file?

Why did dependency state remain unchanged after rebuild?

Was this module compiled or restored from cache?

Which exact command removed this path?
```

Avoid broad questions such as:

```text
Why is LSS broken?
```

A precise question determines what evidence to capture.

## 6. Before/after filesystem state

Use the existing install manifest as the scope.

```bash
MANIFEST="$BASE/before/install-log"

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
done < "$MANIFEST"   > "$BASE/before/filesystem-state"
```

Repeat after the operation.

Compare:

```bash
diff -u   "$BASE/before/filesystem-state"   "$BASE/after/filesystem-state"
```

This shows physical transitions for known owned paths.

## 7. Before/after persistent state

After the operation:

```bash
cp -a /var/state/lunar/packages   "$BASE/after/packages"

cp -a /var/state/lunar/depends   "$BASE/after/depends"
```

Compare:

```bash
diff -u   "$BASE/before/packages"   "$BASE/after/packages"

diff -u   "$BASE/before/depends"   "$BASE/after/depends"
```

This is often more useful than summarizing records manually.

It reveals exact database mutations.

## 8. Before/after manifests

Preserve the new manifest:

```bash
VERSION_AFTER=$(lvu installed "$MODULE")

cp -a "/var/log/lunar/install/${MODULE}-${VERSION_AFTER}"   "$BASE/after/install-log" 2>/dev/null || true
```

Compare:

```bash
diff -u   "$BASE/before/install-log"   "$BASE/after/install-log"
```

Possible results:

```text
no difference
→ same final ownership set

new paths
→ new feature or packaging change

removed paths
→ disabled feature or packaging change

reordered paths only
→ generation-order difference
```

Sort only when order itself is not under investigation.

## 9. Raw installwatch evidence

Installwatch records filesystem operations during the installation phase.

The raw stream may include:

- open;
- chmod;
- mkdir;
- link;
- symlink;
- rename;
- unlink.

It is normally temporary and destroyed after the lifecycle completes.

Capturing it requires controlled instrumentation.

## 10. When raw installwatch is useful

Use raw capture when:

```text
a file appears in build output but not in the manifest
a path is created and later removed
ownership filtering is unclear
rename or unlink order matters
a transient file changes build behavior
```

Do not capture raw installwatch for routine administration.

## 11. Safe raw-capture environment

Use:

- disposable container;
- chroot;
- test VM;
- snapshot-capable system;
- verified backup.

Before patching LSS:

```bash
cp -a /var/lib/lunar/functions/main.lunar   /root/main.lunar.before
```

Restore it immediately after the experiment.

## 12. Minimal instrumentation principle

Instrument only the needed boundary.

For example, copy `INSTALLWATCHFILE` immediately before its normal destruction.

Conceptually:

```bash
if [ -f "$INSTALLWATCHFILE" ]; then
  cp "$INSTALLWATCHFILE" /controlled/evidence/path
  temp_destroy "$INSTALLWATCHFILE"
fi
```

Do not alter parsing, lifecycle order, or return values.

The instrumentation should observe, not redesign behavior.

## 13. Raw stream interpretation

A raw sequence such as:

```text
open    /usr/lib/libexample.a
chmod   /usr/lib/libexample.a
unlink  /usr/lib/libexample.a
```

means:

```text
file created or opened during install
→ permissions set
→ file removed later
```

If the path is absent from the final manifest, that is consistent with final-state ownership.

## 14. Raw events are not ownership

Important distinction:

```text
raw installwatch event
→ something happened

final manifest entry
→ LSS claims final ownership
```

A successful event does not guarantee final ownership.

A missing event does not automatically prove no ownership if another path or mechanism was used.

Interpret raw events alongside BUILD and POST_BUILD.

## 15. Investigating installed-size changes

If package size changes unexpectedly:

1. compare old and new manifests;
2. reproduce current size calculation;
3. classify manifest entries;
4. check shell aliases;
5. inspect changed regular files;
6. inspect Lunar logs included in the manifest.

Reproduce:

```bash
SIZE=0

while IFS= read -r path; do
  if [ -f "$path" ]; then
    value=$(du -k -- "$path" | awk '{print $1}')
    printf '%8s KB  %s
' "$value" "$path"
    SIZE=$((SIZE + value))
  fi
done < install-log

echo "TOTAL=${SIZE}KB"
```

Use `du -k` explicitly.

## 16. Investigating missing paths

If a manifest path is missing:

```bash
grep -Fx '/path' "$BASE/before/install-log"
```

Search current manifests:

```bash
grep -R -F -x '/path'   /var/log/lunar/install 2>/dev/null
```

Inspect module source:

```bash
grep -R -n -F '/path' "$MODULE_DIR"
```

Check whether a service recreated or removed it.

## 17. Investigating overlapping ownership

If several manifests contain the same path:

```bash
grep -R -F -x '/path'   /var/log/lunar/install
```

Possible explanations:

- provider transition;
- intentional shared path;
- stale manifest;
- historical packaging error;
- replacement module;
- common generated file.

Do not remove or reassign manually before understanding the relationship.

## 18. Investigating `/etc` behavior

For an unexpected preserved or removed configuration:

```bash
md5sum /etc/example.conf
grep '/etc/example.conf' saved-md5sum-log
grep -n 'PRESERVE' /etc/lunar/config
```

Interpret:

```text
checksum same
→ unchanged at removal time

checksum different
→ locally modified

PRESERVE=on
→ modified file should remain
```

Also inspect `PROTECTED` policy and remove hooks.

## 19. Investigating dependency-state drift

Capture raw records:

```bash
grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends
```

Inspect declaration:

```bash
sed -n '1,240p' "$MODULE_DIR/DEPENDS"
```

Inspect configuration:

```bash
test -f "$MODULE_DIR/CONFIGURE" &&
  sed -n '1,240p' "$MODULE_DIR/CONFIGURE"

test -f "$MODULE_DIR/OPTIONS" &&
  sed -n '1,240p' "$MODULE_DIR/OPTIONS"
```

Possible drift:

```text
stored choice from old module definition
new declaration not yet applied
manual database change
interrupted reconfiguration
provider renamed
optional dependency default changed
```

## 20. Investigating cache use

Preserve console output:

```bash
lin -c "$MODULE" 2>&1 |
  tee "$BASE/rebuild-console.log"
```

Inspect:

```bash
ls -l /var/cache/lunar/${MODULE}-*
ls -l /var/log/lunar/compile/${MODULE}-*
grep "$MODULE" /var/log/lunar/activity
```

A missing fresh compile path may indicate cache resurrection.

Move the cache aside for a controlled fresh build.

## 21. Investigating hooks

Hooks can affect state outside the manifest.

Inspect:

```bash
for hook in PRE_BUILD POST_BUILD POST_INSTALL PRE_REMOVE POST_REMOVE; do
  test -f "$MODULE_DIR/$hook" || continue
  echo "=== $hook ==="
  sed -n '1,240p' "$MODULE_DIR/$hook"
done
```

Search external paths and commands.

A hook may:

- stop services;
- rebuild indexes;
- remove generated data;
- create users or groups;
- write outside normal ownership paths.

## 22. Controlled single-variable experiments

Change only one factor:

```text
compiler
dependency choice
cache availability
Moonbase revision
PRESERVE setting
module patch
```

Bad experiment:

```text
new compiler
+ new Moonbase
+ new options
+ removed cache
```

A result from several simultaneous changes is difficult to interpret.

## 23. Rebuild versus reinstall versus removal

Choose the correct transition.

```text
rebuild
→ inspect upgrade-style replacement

remove
→ inspect ownership cleanup

install after removal
→ inspect fresh installation

cache restoration
→ inspect resurrection path
```

Do not mix transitions in one evidence bundle without clear phase boundaries.

## 24. Evidence naming

Use explicit names:

```text
packages.before
packages.after
depends.before
depends.after
install-log.before
install-log.after
console.log
installwatch.raw
filesystem-state.before
filesystem-state.after
```

Avoid vague names such as:

```text
test1
new
final2
```

Clear names reduce interpretation errors.

## 25. Evidence immutability

Once captured, do not overwrite raw evidence.

Use new files for interpretation:

```text
raw/
derived/
notes/
```

Example:

```text
raw/installwatch.raw
raw/packages.before
derived/installwatch-summary.txt
notes/conclusions.md
```

Raw evidence should remain unchanged.

## 26. Timestamp discipline

Record exact date and time:

```bash
date --iso-8601=seconds   > "$BASE/timestamp"
```

Record timezone if evidence may be compared across systems.

LSS state dates use compact forms such as:

```text
YYYYMMDD
```

Console and filesystem timestamps may use different time formats.

## 27. Checksums for evidence

For a completed evidence directory:

```bash
find "$BASE" -type f -print0 |
  sort -z |
  xargs -0 sha256sum   > "$BASE/SHA256SUMS"
```

Store the checksum file outside or regenerate carefully to avoid self-reference.

Checksums help verify that raw evidence was not altered.

## 28. Exporting from a container

Example:

```bash
podman cp   lunar-dev:/root/lss-evidence/.   ~/LSS-Evidence/
```

Verify:

```bash
find ~/LSS-Evidence -type f | sort
```

Do not destroy the test container before confirming the export.

## 29. Restoring instrumentation

After testing:

```bash
cp -a /root/main.lunar.before   /var/lib/lunar/functions/main.lunar
```

Verify the exact source fragment.

Do not assume restoration succeeded because the copy command returned no error.

## 30. Restoring test modules

After a removal experiment:

```text
remove orphaned test configuration if appropriate
→ reinstall module
→ verify package record
→ verify canonical configuration
→ verify runtime
```

Return the environment to a known clean state.

## 31. Advanced inconsistency matrix

### Package record present, manifest present, payload missing

Likely:

- manual deletion;
- filesystem damage;
- partial replacement.

Preferred action:

```text
preserve
→ rebuild/reinstall
→ verify
```

### Package record present, manifest missing

Likely:

- state/log inconsistency;
- manual log deletion.

Preferred action:

```text
preserve
→ rebuild same configuration
→ regenerate manifest
```

### Package record absent, manifest present

Likely:

- interrupted removal;
- manual database edit;
- stale logs.

Preferred action:

```text
preserve
→ inspect activity
→ determine real payload ownership
```

### Depends record references missing module

Likely:

- stale dependency state;
- interrupted reconfiguration;
- manual removal.

Preferred action:

```text
reconfigure/rebuild dependent module
```

## 32. Physical reality checks

Check binary:

```bash
command -v program
file /usr/bin/program
```

Check linkage:

```bash
ldd /usr/bin/program
```

Check library:

```bash
readelf -d /usr/lib/libexample.so
```

Check service:

```text
service state
process state
listening socket
functional request
```

Persistent state is not a substitute for runtime validation.

## 33. Advanced troubleshooting report

A strong report contains:

```text
problem statement
expected behavior
actual behavior
environment
module version
Moonbase revision
configuration
dependency state
package state
console output
compile log
manifest
MD5 log
raw evidence when needed
before/after diffs
conclusion
remaining uncertainty
```

This makes review efficient.

## 34. Avoiding false conclusions

Common false conclusions:

```text
file absent from manifest
→ file was never created

no depends record
→ no operational dependency

same version
→ same build

successful lin
→ runtime feature works

preserved config
→ package still owns it

cache archive valid
→ cache matches intended configuration
```

Each requires a separate verification step.

## 35. When not to instrument

Do not instrument when:

- normal logs already answer the question;
- the system is production-critical;
- no rollback exists;
- the experiment has no precise hypothesis;
- the code path is poorly understood;
- evidence cannot be exported safely.

Advanced instrumentation should reduce uncertainty, not create more risk.

## 36. Escalation ladder

Use the least invasive method first.

```text
1. lvu and state files
2. compile/install/MD5/activity logs
3. Moonbase source inspection
4. before/after snapshots
5. cache isolation
6. container reproduction
7. raw installwatch capture
8. source-level instrumentation
```

Stop when the question is answered.

## 37. Methodological discipline

A reliable advanced investigation follows:

```text
observe reality
→ preserve raw evidence
→ identify disagreement
→ formulate one hypothesis
→ instrument one boundary
→ execute one transition
→ compare
→ restore
→ document
```

## 38. Common mistakes

### Mistake 1: instrumenting production first

Use a disposable environment.

### Mistake 2: changing several variables

The result becomes ambiguous.

### Mistake 3: overwriting raw evidence

Interpretation should be separate.

### Mistake 4: forgetting to restore LSS code

Temporary instrumentation can become permanent accidental behavior.

### Mistake 5: trusting only state files

Check physical and runtime reality.

### Mistake 6: trusting only filesystem state

Ownership and policy may differ.

### Mistake 7: collecting everything without a question

More data is not automatically more knowledge.

## 39. Advanced-debugging checklist

Before:

```text
define one question
choose disposable environment
backup LSS code
preserve packages, depends, logs, module source
record environment and toolchain
```

During:

```text
change one variable
capture console
capture raw evidence if required
avoid unrelated operations
```

After:

```text
capture state again
diff before and after
verify runtime
restore instrumentation
restore module state
export evidence
write conclusion
```

## 40. Summary

Advanced LSS debugging is the controlled comparison of intent, events, ownership, state, and reality.

The full model is:

```text
Moonbase
→ declared intent

installwatch and logs
→ observed execution

manifest
→ interpreted ownership

packages and depends
→ persistent state

filesystem and runtime
→ actual result
```

The central rule is:

```text
instrument only the boundary that remains unclear
→ preserve evidence
→ restore the system
