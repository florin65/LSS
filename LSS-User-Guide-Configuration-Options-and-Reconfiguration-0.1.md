# Configuration, Options, and Reconfiguration

**Project:** Lunar Script System User Guide  
**Status:** Draft 0.1  
**Date:** 2026-07-17  
**Audience:** Lunar Linux users, maintainers, and system administrators  
**Evidence basis:** LSS Evidence Foundation 0.1 — Checkpoints 1–6

## 1. Overview

LSS allows a module to be configured according to the purpose of the system.

Configuration may control:

- optional features;
- dependency choices;
- providers;
- compiler flags;
- build modes;
- installed components;
- integration with other software;
- runtime capabilities.

The same module version may therefore produce different results on different Lunar systems.

This is expected and intentional.

## 2. Configuration as part of package identity

For a binary-only package manager, package identity is often reduced to:

```text
name
version
architecture
```

For LSS, a more complete identity is:

```text
module
+ version
+ Moonbase definition
+ selected dependencies
+ selected options
+ compiler and linker flags
+ toolchain
+ environment
```

Two installations of the same version can differ materially if their configuration differs.

## 3. Configuration sources

Module configuration may be expressed through:

```text
CONFIGURE
OPTIONS
DEPENDS
BUILD
global Lunar configuration
local Lunar configuration
environment variables
plugin decisions
```

Not every module uses all these mechanisms.

A simple module may only contain `DETAILS` and `BUILD`.

A complex module may combine several layers.

## 4. `CONFIGURE`

`CONFIGURE` contains module-specific configuration logic.

It may ask the user to:

- enable or disable functionality;
- select an implementation;
- choose an optional backend;
- enable language bindings;
- choose a service integration;
- select a build variant.

Inspect it with:

```bash
MODULE=name
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

test -f "$MODULE_DIR/CONFIGURE" &&
  sed -n '1,240p' "$MODULE_DIR/CONFIGURE"
```

Read the full logic, not only the visible question text.

The answer may trigger other state changes.

## 5. `OPTIONS`

`OPTIONS` defines build options in a reusable or structured form.

Options may map a user choice to:

```text
--enable-feature
--disable-feature
--with-library
--without-library
compiler definitions
environment variables
conditional build branches
```

Inspect:

```bash
test -f "$MODULE_DIR/OPTIONS" &&
  sed -n '1,240p' "$MODULE_DIR/OPTIONS"
```

The exact syntax depends on LSS conventions and module age.

## 6. `DEPENDS` as configuration

Optional dependencies are also configuration choices.

A dependency selection may determine whether a feature is built.

Conceptually:

```text
enable feature
→ enable dependency
→ add build switch
→ build extra functionality
```

or:

```text
disable feature
→ disable dependency
→ add negative build switch
→ omit functionality
```

Therefore `DEPENDS` is not only a graph declaration.

It can also be part of the module's configuration interface.

## 7. `BUILD` as final interpretation

Configuration decisions are ultimately consumed by `BUILD` and its helpers.

Typical patterns include:

```bash
OPTS+=" --enable-feature"
```

or:

```bash
if module_installed dependency; then
  OPTS+=" --with-dependency"
else
  OPTS+=" --without-dependency"
fi
```

The stored answer alone does not change installed software.

The module must be rebuilt so the build logic can apply the choice.

## 8. Global configuration

Global LSS behavior is controlled through files such as:

```text
/etc/lunar/config
```

Observed settings include:

```text
ARCHIVE
PRESERVE
REAP
```

Other global settings may affect:

- compiler flags;
- architecture;
- optimization;
- source locations;
- build directories;
- logging;
- cache behavior;
- color and sound;
- parallelism.

Inspect:

```bash
sed -n '1,260p' /etc/lunar/config
```

Do not change global settings casually before rebuilding a large part of the system.

## 9. Local configuration

LSS may load local configuration after its main configuration and function libraries.

Local configuration can override defaults or add site-specific behavior.

This is useful for:

- system-wide policy;
- local mirrors;
- build flags;
- organization-specific paths;
- hardware-specific choices.

Document local changes.

Otherwise later rebuild differences may be difficult to explain.

## 10. Environment variables

The build environment may influence a module even when no explicit option is selected.

Common variables:

```text
CC
CXX
CFLAGS
CXXFLAGS
LDFLAGS
MAKEFLAGS
PATH
PKG_CONFIG_PATH
```

Example:

```bash
CC=/usr/bin/clang
CXX=/usr/bin/clang++
```

A toolchain mismatch can produce failures that appear to be module bugs.

Before a sensitive rebuild, record:

```bash
env | sort > build-environment.txt
```

For a more focused view:

```bash
env | grep -E   '^(CC|CXX|CFLAGS|CXXFLAGS|LDFLAGS|MAKEFLAGS|PATH|PKG_CONFIG_PATH)='
```

## 11. Persistent choices

LSS preserves dependency and option choices so they can be reused.

This provides continuity across:

- rebuilds;
- upgrades;
- repeated installations;
- system-wide updates.

Persistent choice means:

```text
the system remembers administrator intent
```

It does not mean the choice can never be changed.

## 12. Explicit disabled choices

An optional relationship may be stored as disabled rather than omitted.

Conceptually:

```text
dependency absent from state
→ no recorded decision

dependency stored as off
→ explicit decision not to use it
```

This distinction matters during reconfiguration and automation.

A stored negative choice prevents the system from repeatedly asking or silently changing intent.

## 13. Inspecting active configuration

There is no single universal file that describes every module's complete active configuration.

Use several sources:

```text
module CONFIGURE
module OPTIONS
module DEPENDS
stored dependency records
package record
install manifest
compile log
global configuration
environment
```

Template:

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"

echo '=== PACKAGE STATE ==='
grep "^${MODULE}:" /var/state/lunar/packages || true

echo
echo '=== DEPENDENCY STATE ==='
grep "^${MODULE}:" /var/state/lunar/depends || true

echo
echo '=== CONFIGURE ==='
test -f "$MODULE_DIR/CONFIGURE" &&
  sed -n '1,240p' "$MODULE_DIR/CONFIGURE"

echo
echo '=== OPTIONS ==='
test -f "$MODULE_DIR/OPTIONS" &&
  sed -n '1,240p' "$MODULE_DIR/OPTIONS"

echo
echo '=== DEPENDS ==='
test -f "$MODULE_DIR/DEPENDS" &&
  sed -n '1,240p' "$MODULE_DIR/DEPENDS"
```

## 14. When reconfiguration is needed

Reconfigure when:

- enabling a previously disabled feature;
- disabling an active feature;
- changing an optional provider;
- changing a build backend;
- changing compiler or linker policy;
- changing a major global build setting;
- testing a different module variant;
- adapting the module to new hardware or services.

A simple source update may not require new answers if the intended configuration remains unchanged.

## 15. Reconfiguration requires rebuild

Changing configuration state does not rewrite existing binaries.

The correct lifecycle is:

```text
change choice
→ resolve dependency changes
→ rebuild module
→ create new manifest
→ update package state
→ test functionality
```

If dependent libraries are removed before the rebuild, the current installed binary may break.

## 16. Safe reconfiguration workflow

### Step 1: record current state

```bash
MODULE=name
VERSION=$(lvu installed "$MODULE")
BASE=/root/lss-reconfigure/$MODULE

mkdir -p "$BASE/before" "$BASE/after"

grep "^${MODULE}:" /var/state/lunar/packages   > "$BASE/before/package-record" || true

grep "^${MODULE}:" /var/state/lunar/depends   > "$BASE/before/depends" || true

cp -a "/var/log/lunar/install/${MODULE}-${VERSION}"   "$BASE/before/install-log" 2>/dev/null || true

cp -a "/var/log/lunar/md5sum/${MODULE}-${VERSION}"   "$BASE/before/md5sum-log" 2>/dev/null || true

env | sort > "$BASE/before/environment"
```

### Step 2: inspect the module

```bash
for file in CONFIGURE OPTIONS DEPENDS BUILD; do
  test -f "$MODULE_DIR/$file" || continue
  cp -a "$MODULE_DIR/$file" "$BASE/before/"
done
```

### Step 3: change the choice through LSS

Use the normal LSS configuration path.

Do not edit `/var/state/lunar/depends` manually.

### Step 4: rebuild

```bash
lin -c "$MODULE" 2>&1 |
  tee "$BASE/rebuild.log"
```

### Step 5: capture new state

```bash
VERSION_AFTER=$(lvu installed "$MODULE")

grep "^${MODULE}:" /var/state/lunar/packages   > "$BASE/after/package-record" || true

grep "^${MODULE}:" /var/state/lunar/depends   > "$BASE/after/depends" || true

cp -a "/var/log/lunar/install/${MODULE}-${VERSION_AFTER}"   "$BASE/after/install-log" 2>/dev/null || true
```

### Step 6: compare

```bash
diff -u "$BASE/before/depends" "$BASE/after/depends"
diff -u "$BASE/before/install-log" "$BASE/after/install-log"
diff -u "$BASE/before/package-record" "$BASE/after/package-record"
```

### Step 7: test the feature

Do not stop at a successful build.

Test the capability that was enabled or disabled.

## 17. Enabling a feature

Safe sequence:

```text
inspect module
→ enable option
→ install new dependency if required
→ rebuild module
→ inspect new manifest
→ verify runtime feature
```

Possible results:

- new libraries linked;
- new executable installed;
- new plugin installed;
- new dependency records;
- larger package size;
- different configuration files.

## 18. Disabling a feature

Safe sequence:

```text
disable option
→ rebuild dependent module
→ verify feature is absent
→ verify dependency relationship changed
→ remove dependency only if no longer needed
```

Do not remove the dependency first.

The old binary may still link against it.

## 19. Switching providers

A provider change can affect more than one module.

Safe order:

```text
inspect current provider
→ inspect conflicts
→ enable replacement provider
→ rebuild affected modules
→ test runtime behavior
→ disable old provider relationships
→ remove old provider after verification
```

A provider may be selected through:

- optional dependency;
- configuration question;
- build option;
- conflict rule;
- plugin.

## 20. Configuration and manifests

A changed configuration may change the final ownership manifest.

Examples:

```text
enable documentation
→ new documentation paths

enable language bindings
→ new library and module paths

enable service integration
→ new service files

disable static libraries
→ fewer library files
```

Compare manifests before and after.

The manifest is the clearest evidence of final filesystem differences.

## 21. Configuration and package size

Package size may change after reconfiguration.

This may reflect:

- added files;
- removed files;
- different binary sizes;
- additional documentation;
- new plugins;
- new Lunar artifacts.

The version may remain the same.

Therefore:

```text
same version
+ new options
→ new package-state size
```

## 22. Configuration and dependencies

A configuration choice may produce:

```text
new required dependency
new optional dependency
on → off
off → on
provider replacement
removed dependency record
```

Inspect the raw state:

```bash
grep "^${MODULE}:" /var/state/lunar/depends
```

Also inspect reverse impact:

```bash
lvu depends dependency
lvu leert dependency
```

## 23. Configuration and reproducibility

To reproduce a configured module, preserve:

- module name;
- module version;
- active Moonbase revision;
- `DETAILS`;
- `BUILD`;
- `DEPENDS`;
- `CONFIGURE`;
- `OPTIONS`;
- dependency-state records;
- global configuration;
- local configuration;
- relevant environment variables;
- toolchain versions.

A minimal reproducibility record:

```bash
MODULE=name
SECTION=$(lvu where "$MODULE")
MODULE_DIR="/var/lib/lunar/moonbase/$SECTION/$MODULE"
OUT="/root/reproduce-$MODULE"

mkdir -p "$OUT"

cp -a "$MODULE_DIR" "$OUT/module"
grep "^${MODULE}:" /var/state/lunar/packages   > "$OUT/package-record"
grep "^${MODULE}:" /var/state/lunar/depends   > "$OUT/dependency-records"
cp -a /etc/lunar/config "$OUT/lunar-config"
env | sort > "$OUT/environment"
```

## 24. Reconfiguration and caches

An existing installation cache may reflect old configuration choices.

A cache built with one feature set should not automatically be assumed equivalent to a newly requested configuration.

Before using or trusting a cache, consider:

```text
module version
architecture
build options
dependency selection
toolchain
environment
```

Cache identity may not encode every meaningful build decision.

When configuration changes materially, a fresh build is safer.

## 25. Reconfiguration and `/etc`

A rebuild may interact with existing configuration files.

Possible outcomes depend on:

- installation behavior;
- MD5 state;
- module logic;
- preservation policy;
- upstream install rules.

Before rebuilding a service or daemon:

```bash
cp -a /etc/module.conf /root/module.conf.before
```

Compare afterward:

```bash
diff -u /root/module.conf.before /etc/module.conf
```

Do not assume a rebuild leaves local configuration untouched.

## 26. Reconfiguration and services

A newly enabled feature may require:

- service restart;
- service enablement;
- socket activation;
- new user or group;
- changed permissions;
- regenerated cache or index.

Inspect `POST_INSTALL`, service files, and module documentation.

LSS installation success does not prove the running service is using the new feature.

## 27. Reconfiguration and plugins

A configuration change may install or remove plugins.

This can affect:

- LSS itself;
- desktop environments;
- interpreters;
- applications;
- service frameworks.

When plugin paths change:

1. compare manifests;
2. identify host application;
3. reload or restart when required;
4. verify plugin discovery;
5. inspect runtime logs.

## 28. Detecting active build features

Possible methods:

```text
program --version
program --help
pkgconf metadata
linked libraries
installed plugins
configuration report
runtime feature test
```

Examples:

```bash
program --version
program --help | grep -i feature
ldd /usr/bin/program
pkgconf --print-requires package
```

Use the method appropriate to the software.

## 29. Troubleshooting ignored choices

If a selected option appears to have no effect:

1. inspect `CONFIGURE` and `OPTIONS`;
2. inspect stored dependency state;
3. inspect compile log for configure switches;
4. verify the module was rebuilt;
5. verify a cache did not restore the old build;
6. inspect the final manifest;
7. test runtime behavior;
8. inspect whether the upstream build system auto-detected a different result.

Search compile log:

```bash
xzgrep -n -E   'enable|disable|with-|without-|checking for|found|not found'   /var/log/lunar/compile/module-version.xz
```

## 30. Troubleshooting unexpected auto-detection

Some upstream build systems enable features automatically when a library is present.

This can override the administrator's expectation if the module does not pass an explicit negative flag.

Example pattern:

```text
library installed
→ configure auto-detects it
→ feature enabled
```

Even if no active LSS dependency record was expected.

Inspect:

- configure output;
- `BUILD`;
- `OPTIONS`;
- linked libraries;
- final manifest.

Explicit `--without-feature` or `--disable-feature` is more reproducible than relying on absence.

## 31. Troubleshooting stale choices

Possible symptoms:

- module keeps using an old provider;
- disabled dependency returns after rebuild;
- configuration question is not asked;
- package manifest does not match current intent.

Preserve state, then inspect:

```bash
grep "^${MODULE}:" /var/state/lunar/depends
grep -R -n -F "$MODULE" /etc/lunar /var/state/lunar
```

Do not delete state records blindly.

First determine which LSS command is responsible for resetting or reconfiguring the module.

## 32. Configuration drift

Configuration drift occurs when:

```text
stored intent
≠ current module declaration
≠ installed build
≠ runtime behavior
```

Causes may include:

- Moonbase update;
- manual file changes;
- interrupted rebuild;
- provider replacement;
- stale cache;
- toolchain change;
- undeclared auto-detection.

A drift review should compare all four layers.

## 33. Configuration layers

A useful model:

```text
Layer 1: administrator intent
→ selected features and providers

Layer 2: persistent LSS choices
→ dependency and option state

Layer 3: build interpretation
→ flags and commands used

Layer 4: installed ownership
→ manifest and package record

Layer 5: runtime reality
→ actual program behavior
```

Reliable administration verifies that these layers agree.

## 34. Safe global configuration changes

A change to global compiler or optimization settings may affect many modules.

Before changing:

1. record current `/etc/lunar/config`;
2. record toolchain versions;
3. select a small representative module;
4. rebuild in a container or chroot;
5. inspect logs and runtime;
6. expand gradually;
7. avoid full-system rebuild until the setting is validated.

This reduces the blast radius.

## 35. Common mistakes

### Mistake 1: editing persistent state files manually

This can desynchronize choices and module behavior.

### Mistake 2: changing an option without rebuilding

The installed software remains unchanged.

### Mistake 3: removing an optional dependency before rebuilding

The current binary may still need it.

### Mistake 4: using package version as complete configuration identity

It omits important build choices.

### Mistake 5: trusting a cache after a major configuration change

The cache may represent old choices.

### Mistake 6: assuming build success proves feature success

Runtime validation is still necessary.

### Mistake 7: changing global flags and rebuilding everything immediately

Validate on a small module first.

## 36. Reconfiguration checklist

Before:

```text
inspect CONFIGURE, OPTIONS, DEPENDS, BUILD
record package and dependency state
save manifest and configuration
record environment and toolchain
```

During:

```text
change choice through LSS
allow dependencies to resolve
watch configure and build output
```

After:

```text
compare dependency state
compare manifest
compare package record
verify configuration files
test runtime feature
remove unused dependency only after review
```

## 37. Summary

Configuration in LSS is an operational contract between the administrator and the module.

The lifecycle is:

```text
administrator choice
→ persistent option and dependency state
→ build interpretation
→ final installed result
→ runtime verification
```

The central rule is:

```text
change intent through LSS
→ rebuild
→ compare state
→ test reality
```

This preserves one of Lunar Linux's strongest qualities: software is built for the actual purpose of the system rather than for a universal predefined feature set.
