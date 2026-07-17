# Managing Dependencies and Optional Features

**Project:** Lunar Script System User Guide  
**Status:** Draft 0.1  
**Date:** 2026-07-17  
**Audience:** Lunar Linux users, maintainers, and system administrators  
**Evidence basis:** LSS Evidence Foundation 0.1 — Checkpoints 1–6

## 1. Overview

Dependency handling is one of the defining strengths of LSS.

LSS does not reduce every dependency to a fixed package list.

A module may express:

- required dependencies;
- optional dependencies;
- feature-dependent dependencies;
- alternative providers;
- enabled or disabled relationships;
- stored user choices;
- additional parameters associated with a dependency.

This allows the user to decide how much functionality a module should include.

The same module version can therefore produce different builds on different systems.

## 2. Why optional dependencies matter

Many package managers install a predefined dependency closure.

LSS allows the dependency graph to reflect the user's intended system.

Conceptually:

```text
module
→ required core
→ optional capabilities
→ user selection
→ resolved dependency graph
→ resulting build
```

This supports systems that are:

- smaller;
- more purpose-specific;
- easier to audit;
- less burdened by unused integrations;
- closer to the administrator's actual intent.

Optional dependencies are not merely a convenience.

They are part of system design.

## 3. Required dependencies

A required dependency is necessary for the selected module configuration.

Conceptually:

```text
module A
requires module B
→ B must be available before A can be built or used correctly
```

A stored dependency record may look like:

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
```

Required dependencies normally participate in ordering and installation planning.

## 4. Optional dependencies

An optional dependency enables additional functionality but is not necessary for the module's minimum usable form.

Examples of optional capabilities may include:

- graphical interfaces;
- database backends;
- compression formats;
- image formats;
- audio support;
- network protocols;
- scripting language bindings;
- documentation tools;
- hardware integration;
- authentication mechanisms.

Conceptually:

```text
optional dependency enabled
→ dependency is installed or activated
→ matching build feature is enabled

optional dependency disabled
→ dependency is omitted
→ matching build feature is disabled
```

The exact mechanism is defined by the module.

## 5. Optional does not mean unimportant

An optional dependency may still be operationally critical for a particular user.

Example:

```text
TLS support may be optional in the module definition
but essential for the administrator's intended use
```

Therefore:

```text
optional in Moonbase
≠ irrelevant in deployment
```

The administrator should evaluate functionality, not only dependency type.

## 6. Dependency declarations

Dependency declarations are commonly stored in:

```text
DEPENDS
```

Locate the module:

```bash
MODULE=name
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"
```

Inspect:

```bash
test -f "$MODULE_DIR/DEPENDS" &&
  sed -n '1,240p' "$MODULE_DIR/DEPENDS"
```

A module without a `DEPENDS` file may still rely on:

- bootstrap components;
- shell tools;
- implicit toolchain requirements;
- runtime commands not represented as formal dependencies.

## 7. Declaration versus resolved state

The module's `DEPENDS` file describes available or required relationships.

The current system's resolved dependency state is stored in:

```text
/var/state/lunar/depends
```

This distinction is fundamental:

```text
DEPENDS
→ what the module declares or permits

depends database
→ what the current system resolved and stored
```

Do not assume that reading `DEPENDS` alone reveals the active build choices.

## 8. Persistent dependency record format

Observed records use six colon-separated fields:

```text
module:dependency:status:type:field5:field6
```

Example:

```text
ccache:xxhash:on:required::
```

The first four fields are operationally clear:

```text
module
dependency
status
type
```

The final fields preserve additional declaration parameters and may be empty.

When parsing, preserve all fields.

Example:

```bash
awk -F: '{
  printf "module=%s dependency=%s status=%s type=%s extra1=%s extra2=%s\n",
         $1, $2, $3, $4, $5, $6
}' /var/state/lunar/depends
```

## 9. Enabled and disabled relationships

The status field may represent whether a dependency selection is active.

Conceptually:

```text
on
→ selected and active

off
→ known choice, currently disabled
```

A disabled optional dependency may remain represented because the user's choice itself is persistent state.

This is important:

```text
absence of a relationship
≠ explicitly disabled relationship
```

An explicit negative choice can be meaningful during reconfiguration.

## 10. Inspecting a module's current dependencies

Show records where the module is the depender:

```bash
MODULE=name

grep "^${MODULE}:" /var/state/lunar/depends
```

Show modules that depend on it:

```bash
grep -E "^[^:]+:${MODULE}:" /var/state/lunar/depends
```

Show both directions:

```bash
grep -E "^${MODULE}:|^[^:]+:${MODULE}:"   /var/state/lunar/depends
```

This provides the raw stored relationships.

## 11. Using `lvu`

Useful commands:

```bash
lvu depends module
lvu leert module
```

`lvu depends` shows installed modules that depend on the target.

`lvu leert` displays the reverse dependency tree.

Example:

```bash
lvu depends xxhash
lvu leert xxhash
```

These commands help estimate the impact of rebuild or removal.

## 12. Forward and reverse views

Two questions must be kept separate:

```text
What does this module depend on?
→ forward dependency view

What depends on this module?
→ reverse dependency view
```

Forward view matters before installation and rebuild.

Reverse view matters before removal or replacement.

A module can have few forward dependencies but many reverse dependents.

## 13. Dependency ordering

LSS constructs dependency relationships and uses graph ordering for multi-module operations.

Conceptually:

```text
dependency graph
→ topological ordering
→ build dependencies before dependents
```

`tsort` is used in the dependency-ordering machinery.

A valid dependency graph must not contain unresolved cycles that prevent ordering.

## 14. Multi-module installation

For several requested modules, LSS separates work into stages:

```text
configure and resolve dependencies
→ download sources
→ build and install sequentially
```

This allows downloads to overlap while keeping installation state changes ordered.

Dependencies may enlarge the actual set of modules beyond the names entered by the user.

## 15. User choices and build results

Optional dependency choices may affect:

- configure arguments;
- compiler definitions;
- linked libraries;
- installed files;
- runtime commands;
- plugin availability;
- service integration;
- package size;
- future reverse dependencies.

Therefore:

```text
same module name
+ same version
+ different optional choices
→ potentially different software
```

This is expected behavior, not inconsistency.

## 16. Inspecting configuration files

Dependency choices may be implemented through:

```text
DEPENDS
CONFIGURE
OPTIONS
BUILD
```

Inspect all relevant files:

```bash
for file in DEPENDS CONFIGURE OPTIONS BUILD; do
  if [ -f "$MODULE_DIR/$file" ]; then
    echo "=== $file ==="
    sed -n '1,240p' "$MODULE_DIR/$file"
  fi
done
```

Search for feature flags:

```bash
grep -R -n -E   'optional_depends|required_depends|depends|enable|disable|with-|without-'   "$MODULE_DIR"
```

Exact helper names and conventions may vary.

## 17. Optional dependency pattern

A typical optional dependency conceptually maps:

```text
dependency selected
→ add dependency record
→ install dependency if needed
→ pass --enable-feature or --with-library

dependency rejected
→ store disabled state when applicable
→ pass --disable-feature or --without-library
```

The module may also use environment variables or custom shell branches.

Read the module rather than assuming a single universal syntax.

## 18. Reconfiguration

Changing optional features usually requires the module to be configured again and rebuilt.

Conceptually:

```text
old choices
→ reconfigure
→ dependency graph changes
→ rebuild
→ new manifest and package state
```

A dependency choice does not alter an already compiled binary merely by editing a state file.

The module must pass through its build lifecycle again.

## 19. Before changing optional features

Preserve current state:

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
BASE=/root/lss-option-change/$MODULE

mkdir -p "$BASE/before" "$BASE/after"

grep "^${MODULE}:" /var/state/lunar/packages   > "$BASE/before/package-record" || true

grep "^${MODULE}:" /var/state/lunar/depends   > "$BASE/before/forward-depends" || true

grep -E "^[^:]+:${MODULE}:" /var/state/lunar/depends   > "$BASE/before/reverse-depends" || true

cp -a "/var/log/lunar/install/${MODULE}-${VERSION}"   "$BASE/before/install-log" 2>/dev/null || true
```

Also record the module files:

```bash
cp -a "$MODULE_DIR" "$BASE/before/module-definition"
```

## 20. After reconfiguration

Capture:

```bash
grep "^${MODULE}:" /var/state/lunar/packages   > "$BASE/after/package-record" || true

grep "^${MODULE}:" /var/state/lunar/depends   > "$BASE/after/forward-depends" || true

grep -E "^[^:]+:${MODULE}:" /var/state/lunar/depends   > "$BASE/after/reverse-depends" || true

VERSION_AFTER=$(lvu installed "$MODULE")

cp -a "/var/log/lunar/install/${MODULE}-${VERSION_AFTER}"   "$BASE/after/install-log" 2>/dev/null || true
```

Compare:

```bash
diff -u   "$BASE/before/forward-depends"   "$BASE/after/forward-depends"

diff -u   "$BASE/before/install-log"   "$BASE/after/install-log"

diff -u   "$BASE/before/package-record"   "$BASE/after/package-record"
```

## 21. What to expect after changing an option

Possible changes:

```text
new dependency record
removed dependency record
on → off
off → on
new linked library
new executable or plugin
removed executable or plugin
different package size
different install manifest
same version but different build
```

A version number alone cannot describe the full configuration.

## 22. Removing an optional dependency

Do not immediately run:

```bash
lrm optional-library
```

First determine whether dependent modules were built with it enabled.

Inspect:

```bash
lvu depends optional-library
lvu leert optional-library
```

Then inspect the dependent modules' stored records:

```bash
grep -E "^[^:]+:optional-library:"   /var/state/lunar/depends
```

If the relationship is active, first reconfigure or rebuild the dependent module without that feature.

Safe conceptual order:

```text
disable feature in dependent module
→ rebuild dependent module
→ verify relationship is removed or off
→ remove optional dependency
```

## 23. Orphaned dependencies

A dependency may remain installed after no module depends on it.

This can happen after:

- disabling an optional feature;
- removing a dependent module;
- changing providers;
- rebuilding with fewer features.

Use:

```bash
lvu leafs
```

to identify modules without reverse dependents.

But leaf status does not automatically mean unnecessary.

The module may be directly useful to the administrator.

## 24. Optional dependency cleanup

A careful cleanup process:

```text
identify leaf
→ inspect package role
→ inspect Moonbase definition
→ search external usage
→ preserve logs
→ remove only after review
```

Useful checks:

```bash
MODULE=name

lvu depends "$MODULE"
lvu leert "$MODULE"

grep -R -n -w "$MODULE"   /etc /usr/local /root 2>/dev/null | head -100
```

The search is only a clue, not definitive proof of use.

## 25. Alternative providers

Some capabilities may be satisfied by alternative modules.

Conceptually:

```text
feature requires capability X
→ provider A or provider B
```

The selected provider affects:

- dependency state;
- installed files;
- runtime behavior;
- conflict handling;
- future rebuilds.

Before switching providers:

1. inspect conflicts;
2. inspect reverse dependents;
3. preserve current state;
4. configure the replacement;
5. rebuild affected modules;
6. verify runtime behavior;
7. remove the old provider only after validation.

## 26. Dependency conflicts

Conflicts may be declared separately from dependencies.

Inspect:

```bash
test -f "$MODULE_DIR/CONFLICTS" &&
  sed -n '1,240p' "$MODULE_DIR/CONFLICTS"
```

A dependency choice may activate a conflict indirectly.

Do not force both providers into the filesystem unless the module design explicitly supports coexistence.

## 27. Hidden or implicit dependencies

Not all operational requirements are necessarily recorded as formal dependency relationships.

Possible implicit requirements:

- shell commands assumed by BUILD;
- compiler toolchain;
- kernel features;
- hardware;
- service manager;
- external configuration;
- local scripts;
- manually installed software.

This is why dependency state is necessary but not sufficient for risk analysis.

## 28. Build-time versus runtime dependencies

A dependency may be needed only to build the module.

Another may be required at runtime.

Some may be needed for both.

This distinction matters for cleanup.

Removing a build-only dependency after installation may be safe in some cases, but only if:

- no future rebuild is expected without reinstalling it;
- no runtime linkage exists;
- no other module needs it;
- the stored policy allows it.

Do not infer dependency phase only from package size or name.

Inspect the declaration and build commands.

## 29. Checking runtime linkage

For an executable:

```bash
ldd /usr/bin/program
```

For a shared library:

```bash
readelf -d /usr/lib/libexample.so | grep NEEDED
```

For pkg-config metadata:

```bash
pkgconf --print-requires package
pkgconf --print-requires-private package
```

These commands provide runtime or link metadata, not LSS policy state.

Compare them with `/var/state/lunar/depends`.

## 30. Dependency state and truth

The dependency database records LSS's resolved relationship state.

It does not prove:

- that the dependency is physically present;
- that linking succeeded;
- that runtime use is functional;
- that no undeclared dependency exists;
- that the declaration is still current.

Cross-check when accuracy matters:

```text
depends database
→ intended/resolved relationship

packages database
→ installed module state

filesystem/linker inspection
→ physical implementation

runtime test
→ actual functionality
```

## 31. Detecting inconsistencies

Potential problems:

```text
active dependency record but dependency not installed
installed dependency but no reverse relationships
dependent binary linked against missing library
disabled record but build still contains feature
Moonbase declaration changed but stored choice is old
```

Checks:

```bash
DEP=xxhash

grep -E "^[^:]+:${DEP}:on:" /var/state/lunar/depends

grep "^${DEP}:" /var/state/lunar/packages ||
  echo "dependency package record missing"
```

For a program:

```bash
ldd /usr/bin/program | grep 'not found'
```

Preserve evidence before repairing state.

## 32. Optional features and reproducibility

To reproduce a build, recording only:

```text
module name
version
```

is insufficient.

Also preserve:

- dependency selections;
- configuration answers;
- options;
- compiler flags;
- provider choices;
- active Moonbase revision;
- toolchain versions.

A more complete build identity is:

```text
module
+ version
+ Moonbase definition
+ selected dependencies
+ options
+ toolchain
+ environment
```

## 33. A dependency review template

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

echo '=== PACKAGE STATE ==='
grep "^${MODULE}:" /var/state/lunar/packages || true

echo
echo '=== DECLARATION ==='
test -f "$MODULE_DIR/DEPENDS" &&
  sed -n '1,240p' "$MODULE_DIR/DEPENDS"

echo
echo '=== STORED FORWARD RELATIONSHIPS ==='
grep "^${MODULE}:" /var/state/lunar/depends || true

echo
echo '=== REVERSE RELATIONSHIPS ==='
grep -E "^[^:]+:${MODULE}:"   /var/state/lunar/depends || true

echo
echo '=== REVERSE DEPENDENCY VIEW ==='
lvu depends "$MODULE"
lvu leert "$MODULE"

echo
echo '=== OPTIONS AND CONFIGURATION ==='
for file in CONFIGURE OPTIONS; do
  test -f "$MODULE_DIR/$file" || continue
  echo "--- $file"
  sed -n '1,240p' "$MODULE_DIR/$file"
done
```

## 34. Safe optional-feature change pattern

```text
1. Inspect declaration.
2. Inspect stored dependency state.
3. Record current manifest and package state.
4. Change the option through LSS.
5. Rebuild the module.
6. Compare dependency records.
7. Compare final manifest.
8. Test the affected feature.
9. Remove newly unused dependencies only after review.
```

This avoids breaking a working module by removing a dependency too early.

## 35. Common mistakes

### Mistake 1: installing every optional dependency

This defeats one of LSS's main design advantages.

### Mistake 2: disabling features only to reduce package count

A small dependency may provide essential functionality.

### Mistake 3: removing a dependency before rebuilding the dependent module

The existing binary may still require it.

### Mistake 4: treating `DEPENDS` as the current system state

It is a declaration, not the complete resolved state.

### Mistake 5: treating `/var/state/lunar/depends` as physical proof

It is persistent LSS state, not a runtime test.

### Mistake 6: comparing builds only by version

Optional choices can change the result substantially.

### Mistake 7: assuming leaf means orphan

A leaf may be directly installed for user use.

## 36. Design value of optional dependencies

Optional dependencies let Lunar systems remain intentionally different.

A workstation may enable:

```text
graphical interfaces
audio
printing
media formats
desktop integration
```

A server may omit them and enable:

```text
TLS
database support
network services
monitoring
```

A rescue environment may choose only:

```text
filesystem tools
network diagnostics
recovery utilities
```

The module remains the same logical module, but its resolved form reflects the machine's purpose.

## 37. Summary

LSS dependency management is not only package ordering.

It is a configuration mechanism.

The operational model is:

```text
module declaration
→ required and optional relationships
→ user choice
→ stored dependency state
→ ordered build
→ feature-specific installed result
```

The central rule is:

```text
enable what the system needs
→ omit what it does not need
→ preserve the choices
→ rebuild before removing dependencies
```

Optional dependencies are one of the mechanisms through which Lunar Linux remains both source-based and administrator-directed.
