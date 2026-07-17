# Policy States: Held, Exiled, and Enforced Modules

**Project:** Lunar Script System User Guide  
**Status:** Draft 0.1  
**Date:** 2026-07-17  
**Audience:** Lunar Linux users and system administrators  
**Evidence basis:** LSS Evidence Foundation 0.1 — Checkpoints 1–6

## 1. Overview

LSS stores more than installation state.

The package-state database can also preserve policy decisions.

Known state terms include:

```text
installed
held
exiled
enforced
```

These states describe different administrative intentions.

The central distinction is:

```text
installation state
→ what is present

policy state
→ what LSS should or should not do
```

## 2. Package-state database

The database is:

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

The third field may represent one or more states.

Do not treat it as a simple yes/no installed flag.

## 3. Why policy states exist

A source-based rolling system needs more than automatic progression.

Administrators may need to:

- keep a known-good version;
- prevent an unwanted module from returning;
- require a module to remain present;
- protect a transition in progress;
- preserve local system policy.

Policy states make that intent persistent.

## 4. `installed`

`installed` means the module is recorded as present.

It normally corresponds to:

- package record;
- installed version;
- size;
- install manifest;
- MD5 log;
- payload on disk.

However, persistent state and filesystem reality can drift.

Always verify critical cases.

## 5. `held`

A held module is intentionally prevented from normal update progression.

Conceptually:

```text
module remains installed
→ current version is retained
→ normal update should not replace it
```

Typical reasons:

- known regression in newer version;
- compatibility requirement;
- local patch;
- ABI freeze;
- production stability;
- pending validation.

## 6. When to hold a module

Good reasons:

```text
new version breaks required functionality
toolchain transition incomplete
dependent software not ready
local patch exists only for current version
critical service needs validation first
```

Bad reason:

```text
avoid understanding the update indefinitely
```

A hold should be documented and reviewed.

## 7. Risks of long-term holds

A held module may accumulate:

- security exposure;
- dependency incompatibility;
- ABI divergence;
- stale configuration;
- unsupported patches;
- upgrade complexity.

Record:

```text
why held
since when
which version
which dependents
what condition ends the hold
```

## 8. Inspecting held modules

Search:

```bash
awk -F: '$3 ~ /(^|\+)held(\+|$)/ {print}'   /var/state/lunar/packages
```

For one module:

```bash
grep '^module:' /var/state/lunar/packages
```

Inspect reverse dependents before changing policy:

```bash
lvu depends module
lvu leert module
```

## 9. `exiled`

An exiled module is intentionally excluded from the system's accepted package universe.

Conceptually:

```text
module should not be installed
→ dependency resolution should respect that policy
```

Possible reasons:

- incompatible software;
- unwanted provider;
- security policy;
- replaced implementation;
- licensing or operational policy;
- broken module.

## 10. Exiled versus uninstalled

```text
uninstalled
→ not currently present

exiled
→ explicitly forbidden or rejected
```

This distinction matters.

An uninstalled dependency may be installed later.

An exiled dependency represents a negative administrative decision.

## 11. Exiled versus disabled optional dependency

A disabled optional dependency means:

```text
this module does not use that feature
```

An exiled module means:

```text
this module should not be accepted on the system
```

The scope differs.

Optional disablement belongs to one module relationship.

Exile is broader system policy.

## 12. When to exile a module

Possible uses:

- prevent a conflicting provider;
- prevent an obsolete implementation;
- enforce local security policy;
- block accidental installation;
- preserve a deliberate alternative stack.

Review conflicts and dependents first.

## 13. Risks of exile

Exiling a module may make other modules impossible to install.

Potential results:

- dependency resolution failure;
- feature loss;
- provider conflict;
- incomplete system update.

Before exile:

```bash
lvu depends module
lvu leert module
grep -E '^[^:]+:module:' /var/state/lunar/depends
```

## 14. `enforced`

An enforced module is required by policy to remain part of the system.

Conceptually:

```text
module must be present
→ system policy should restore or retain it
```

Possible uses:

- essential administration tool;
- mandatory security component;
- required system service;
- organizational baseline;
- critical provider.

## 15. Enforced versus required dependency

A required dependency belongs to another module's dependency graph.

An enforced module belongs to system policy.

```text
required dependency
→ needed by module A

enforced module
→ required by administrator or system policy
```

A module can be enforced even when no package depends on it.

## 16. Enforced versus installed

```text
installed
→ current presence

enforced
→ persistent requirement for presence
```

An installed module can be removed.

An enforced module should not be casually removed because policy expects it to remain.

## 17. Combined state lists

The state field may encode multiple terms.

Conceptually:

```text
installed+held
installed+enforced
```

Do not assume exact combination syntax without inspecting real records.

Parse the state field as a list, not a single fixed value.

## 18. Inspecting state combinations

```bash
awk -F: '{
  printf "%-30s %-24s %-16s %s\n",
         $1, $3, $4, $5
}' /var/state/lunar/packages
```

Search a term:

```bash
awk -F: '$3 ~ /held/ {print}'   /var/state/lunar/packages
```

Preserve the full original record.

## 19. Policy state and update behavior

Before update:

```bash
grep '^module:' /var/state/lunar/packages
```

If held:

```text
do not assume normal update will proceed
```

If enforced:

```text
do not assume removal is administratively valid
```

If exiled:

```text
do not assume dependency installation is allowed
```

## 20. Policy state and rebuild

A rebuild may or may not preserve policy terms depending on the operation.

Before:

```bash
grep '^module:' /var/state/lunar/packages   > package.before
```

After:

```bash
grep '^module:' /var/state/lunar/packages   > package.after

diff -u package.before package.after
```

Verify both version and state-list.

## 21. Policy state and removal

Before `lrm`, inspect:

```bash
grep '^module:' /var/state/lunar/packages
```

Do not remove an enforced module without first changing the policy intentionally.

A held module may still be removable, but the hold explains administrative intent.

An exiled module may already be absent.

## 22. Policy state and dependency resolution

Possible interactions:

```text
dependency needed
+ dependency exiled
→ resolution conflict

module enforced
+ dependency missing
→ restoration or failure path

module held
+ dependent requires newer ABI
→ update conflict
```

Policy and dependency state must be considered together.

## 23. Policy review before system update

List policy states:

```bash
awk -F: '$3 != "installed" {print}'   /var/state/lunar/packages
```

Review each record before a large update.

Create a policy report:

```bash
awk -F: '{
  if ($3 ~ /held|exiled|enforced/)
    printf "%s:%s:%s:%s:%s\n", $1, $2, $3, $4, $5
}' /var/state/lunar/packages   > /root/lunar-policy-report
```

## 24. Documenting policy decisions

For each policy state, record:

```text
module
state
reason
date
owner
expected review date
exit condition
related modules
```

Persistent state without rationale becomes technical debt.

## 25. Temporary holds

A temporary hold should have a clear exit condition.

Examples:

```text
remove hold after LLVM family rebuild passes
remove hold after service regression is fixed
remove hold after dependent module supports new ABI
```

Avoid indefinite anonymous holds.

## 26. Provider policy

Policy states are useful for alternative providers.

Example:

```text
provider A enforced
provider B exiled
```

This makes system intent explicit.

Before switching:

```text
remove old policy
→ configure replacement
→ rebuild dependents
→ verify runtime
→ apply new policy
```

## 27. Policy drift

Policy drift occurs when:

```text
state says held
but module was manually replaced

state says enforced
but payload is missing

state says exiled
but module is present

state says installed
but manifest is missing
```

Detect through cross-checking.

## 28. Policy consistency checks

For enforced modules:

```bash
awk -F: '$3 ~ /enforced/ {print $1}'   /var/state/lunar/packages |
while read -r module; do
  grep "^${module}:" /var/state/lunar/packages >/dev/null ||
    echo "missing record: $module"
done
```

Also verify manifests and payload.

For exiled modules, inspect whether they are unexpectedly installed.

## 29. Do not edit the database manually

Manual editing may:

- break state-list syntax;
- desynchronize related state;
- hide policy history;
- create unsupported combinations.

Use supported LSS commands.

If a repair is unavoidable:

```text
preserve original database
→ document reason
→ make minimal change
→ verify all affected state
```

## 30. Policy state backup

Before policy changes:

```bash
cp -a /var/state/lunar/packages   /root/packages.before-policy-change
```

Capture relevant dependency state:

```bash
cp -a /var/state/lunar/depends   /root/depends.before-policy-change
```

Afterward, diff both.

## 31. Policy and recovery

A recovery should restore not only payload but policy.

Example:

```text
restore module files
but lose held state
→ system may update unexpectedly later
```

Back up:

```text
packages
depends
Moonbase revision
local policy notes
```

## 32. Policy and automation

Automation should respect administrator policy.

A safe automated updater must:

- skip held modules;
- reject exiled modules;
- preserve enforced modules;
- report conflicts;
- avoid silently changing policy.

Automation should not reinterpret intent without evidence.

## 33. Common mistakes

### Mistake 1: treating held as installed

Held adds update policy.

### Mistake 2: treating exiled as merely absent

It is an explicit rejection.

### Mistake 3: treating enforced as a dependency

It is system-level policy.

### Mistake 4: forgetting policy during rollback

Payload and state must both be restored.

### Mistake 5: leaving holds undocumented

Future administrators cannot evaluate them.

### Mistake 6: forcing dependency resolution around exile

Resolve the policy conflict first.

## 34. Policy review template

```bash
echo '=== SPECIAL POLICY STATES ==='

awk -F: '
  $3 ~ /held|exiled|enforced/ {
    printf "module=%s date=%s states=%s version=%s size=%s\n",
           $1, $2, $3, $4, $5
  }
' /var/state/lunar/packages
```

For each module:

```bash
MODULE=name

grep "^${MODULE}:" /var/state/lunar/packages
lvu depends "$MODULE"
lvu leert "$MODULE"
grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends
```

## 35. Safe policy-change model

```text
inspect current state
→ identify reason
→ inspect dependencies
→ preserve databases
→ change through LSS
→ verify record
→ test intended behavior
→ document decision
```

## 36. Policy hierarchy

A useful conceptual model:

```text
administrator intent
→ held / exiled / enforced

module intent
→ required / optional dependencies

installed result
→ package record and manifest

runtime reality
→ actual files and behavior
```

Conflicts between layers must be resolved explicitly.

## 37. Summary

LSS policy states preserve administrator intent across normal package operations.

```text
held
→ keep current version

exiled
→ reject module presence

enforced
→ require module presence
```

These states are distinct from ordinary installation and dependency state.

The central rule is:

```text
inspect policy before update or removal
→ change it deliberately
→ preserve the rationale
